# Server-Side Request Forgery (SSRF)

Tricking the **server** into making HTTP requests *for you* — to internal services, cloud metadata endpoints, or a server you control. The attacker doesn't reach these directly; they abuse a feature that fetches URLs and redirect it, inheriting the trust internal systems place in the application server.

> This is the **TryHackMe "SSRF" room**. It covers what SSRF is, the four input patterns that carry it, how to spot and confirm it (including blind), how to beat deny-list/allow-list/open-redirect defences, and a practical on the Acme IT Support site.
>
> 🔗 Sibling labs in this folder: [SQLi](../SQLi/SQL-Injection-Lab.md), [XSS](../XSS/XSS.md). SQLi attacks the **database**, XSS attacks **other users' browsers**, SSRF attacks the **internal network from the server's position**.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What SSRF is and why it works](#what-ssrf-is-and-why-it-works)
- [Regular vs blind SSRF](#regular-vs-blind-ssrf)
- [Impact](#impact)
- [The four input vectors](#the-four-input-vectors)
- [Identifying SSRF](#identifying-ssrf)
- [Confirming blind SSRF](#confirming-blind-ssrf)
- [Bypassing defences](#bypassing-defences)
- [Practical: Acme IT Support avatar SSRF](#practical-acme-it-support-avatar-ssrf)
- [Mitigation](#mitigation)
- [Key takeaways](#key-takeaways)

---

## In plain English

Some app features take a URL (or part of one) and the **server** goes and fetches it — think URL previews, webhooks, "import from URL", PDF generators, avatar loaders. SSRF is when you change *where* it fetches from. Because the request now comes **from the server**, it lands on things the outside world can't reach: internal admin panels, databases, and the cloud "metadata" service that hands out credentials.

The core trick: **internal systems trust the server's IP, so if you steer the server's requests, you borrow that trust.**

| Concept | One-line hook |
| --- | --- |
| **Regular SSRF** | The fetched page comes back in the response — you can read it. |
| **Blind SSRF** | You get a canned "success" — confirm it by making the server call *your* listener. |
| **Cloud metadata** | `169.254.169.254` — reach it and you may pull temporary cloud credentials. |
| **Deny list** | Blocks known-bad hosts → beaten with alternate IP encodings / DNS tricks. |
| **Allow list** | Only permits trusted prefixes → beaten with `@`, subdomains, or an open redirect. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>SSRF</strong> = Server-Side Request Forgery — making the server issue a request of your choosing. <strong>Cloud metadata endpoint</strong> = a link-local address (<code>169.254.169.254</code>) that VMs query for their own config and short-lived IAM credentials. <strong>Loopback</strong> = <code>127.0.0.1</code>/<code>localhost</code>, the machine talking to itself. <strong>Deny list / allow list</strong> = block-known-bad vs permit-only-known-good filtering. <strong>Open redirect</strong> = an endpoint that forwards you to a URL from a parameter. <strong>Path normalisation</strong> = the server resolving <code>x/../private</code> down to <code>/private</code>.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not rote memorisation. Remember the one idea — <em>find a parameter the server fetches, then point it somewhere it shouldn't go</em> — plus the metadata address and the two filter-bypass families. Exact payloads follow from the vector you're looking at.
</div>

---

## What SSRF is and why it works

SSRF lets an attacker make the **server-side application** send HTTP requests to a destination they choose — an internal service, a cloud metadata endpoint, or an external server they control. You do it by manipulating a parameter the app uses to *build* a server-side request.

It works because of misplaced trust: backend services, databases, and cloud infrastructure often accept requests **without extra authentication** as long as they come from a trusted internal IP — they assume anything from inside is legitimate. Control where the server sends its requests and you **inherit that trust**.

---

## Regular vs blind SSRF

The distinction changes how you exploit it:

| Type | Response visible? | Description |
| --- | --- | --- |
| **Regular SSRF** | **Yes** | The back-end response is returned in the app's front-end response — you read the output directly (e.g. force a fetch of an internal admin page and its HTML shows up in the reply). |
| **Blind SSRF** | **No** | The server makes the request but doesn't return the body — often a fixed "success" regardless of outcome. Still exploitable via indirect signals. |

Blind SSRF is confirmed by pointing the request at **a server you control** (e.g. Burp Collaborator) and watching for the callback. Differences in **response time** or **error messages** between reachable and unreachable hosts also leak information about internal infrastructure.

---

## Impact

Impact depends on what internal services are reachable from the app server:

| Impact | Description |
| --- | --- |
| **Access to internal endpoints** | Admin panels, config interfaces, monitoring dashboards not exposed to the internet become reachable — IP-based access controls are bypassed because the request originates from the server. |
| **Sensitive data exposure** | Backend databases, private APIs, and internal tooling that trust the server's network position may return customer data, records, or secrets. |
| **Internal network recon** | Sending requests to different IPs/ports maps internal hosts and services via variations in response time, status codes, and error messages. |
| **Cloud metadata theft** | AWS/GCP/Azure expose instance metadata at **`169.254.169.254`** — reach it and you can retrieve temporary credentials, IAM role details, and instance config. |
| **Credential / token leakage** | Tokens and secrets passed between internal services can be intercepted, especially where back-end comms run over unencrypted HTTP. |

---

## The four input vectors

SSRF rarely appears as a neat full URL; user input feeds the server request in different shapes. Recognising the pattern is the first step.

### 1. Full URL in a parameter

Most direct form — the app takes a complete URL and fetches it. Common in URL previews, webhooks, PDF generators. Example stock-checker:

```text
https://website.thm/item/2?server=api
```

builds `https://server.website.thm/api/item?id=2`. Redirect it:

| Input | Resulting server-side request |
| --- | --- |
| `server=api` | `https://server.website.thm/api/item?id=2` |
| `server=server.website.thm/flag?id=9&x=` | `https://server.website.thm/flag?id=9&x=/api/item?id=2` |

The trailing **`&x=`** turns whatever the app appends (`/api/item?id=2`) into an ignored parameter, neutralising it.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Try It Yourself answer:</strong> to fetch <code>/flag?id=9</code> on <code>server.website.thm</code>, set <code>server=server.website.thm/flag?id=9&amp;x=</code> — the <code>&amp;x=</code> swallows the app's appended path so only your URL is requested.
</div>

### 2. Partial URL (hostname or path only)

The app accepts just a hostname and builds the rest server-side. Developers assume this limits the attack surface, but you can still supply a host you control:

```text
https://website.thm/stock?server=api.internal      →  https://api.internal/stock/item
https://website.thm/stock?server=attacker.com      →  request goes to attacker.com
```

If the response is reflected, you exfiltrate internal data; if blind, you at least confirm the callback.

### 3. Path traversal in the URL

When you control only a **path segment**, `../` sequences reach endpoints outside the intended directory:

```text
https://website.thm/stock?url=/item/123/details
url=/../admin   →  https://website.thm/admin
```

Same traversal idea as file-inclusion bugs, applied to URL paths instead of filesystem paths.

### 4. Hidden form fields

Not everything is in the address bar. Some vectors live in the HTML source and only surface via View Source or request interception. Classic: a profile avatar path stored in a hidden field:

```html
<input type="hidden" name="avatar" value="/images/avatars/default.png">
```

If the server fetches whatever path this holds, edit it (DevTools or Burp) to point at an internal resource. **Test form fields, API requests, and any parameter that feeds a server-side request — not just visible URLs.**

---

## Identifying SSRF

Strong signals an app may be vulnerable:

- **Full URL in a parameter** — a complete URL in the query string is almost certainly used for a server-side request.
- **Hidden form fields** — reveal via page source / proxy; their values may control resource fetching.
- **Partial URL (hostname only)** — app accepts a host and builds the rest.
- **Path only** — only the path is user-controlled; app prepends scheme + host.

Features that frequently carry SSRF:

| Feature | Why it's relevant |
| --- | --- |
| Webhook configuration | App requests a user-supplied URL to verify the endpoint. |
| PDF / report generation | Server fetches content from a supplied URL to render it. |
| URL preview / unfurling | App retrieves metadata (title, thumbnail) from a user link. |
| File import by URL | Server downloads a file from a remote location you specify. |
| Integration settings | Third-party service URLs are stored and queried by the server. |

A full URL in a query param is easy to test; a partial path segment may need lots of trial and error. **Recognise the pattern first, then experiment.**

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Room question — which URL is most likely vulnerable?</strong><br>
<code>https://website.thm/fetch-file.php?fname=242533.pdf&srv=filestorage.cloud.thm&port=8001</code> — it takes a <strong>server host + port</strong> (<code>srv</code>/<code>port</code>) that the app clearly uses to make an outbound fetch, exactly the SSRF shape. The others pass only IDs/prices/categories, which don't drive a server request.
</div>

---

## Confirming blind SSRF

When the server makes the request but doesn't reflect the body:

| Method | How it works |
| --- | --- |
| **External HTTP logger** (e.g. requestbin.com) | Supply the logger's URL as the payload; watch its dashboard for incoming requests from the target. |
| **Burp Collaborator** | Unique domain logging HTTP **and DNS** callbacks — useful when HTTP is blocked but DNS still resolves. |
| **Self-hosted listener** (`python3 -m http.server`) | Run a simple HTTP server and monitor for the target's connection. |
| **Timing analysis** | Compare response times for internal hosts that exist vs don't — consistent differences mean the server is resolving/connecting to your address. |
| **Error-based inference** | Different errors for reachable vs unreachable hosts leak internal-network info even when the body is hidden. |

Confirming the server makes outbound requests based on your input is enough to establish the vulnerability.

---

## Bypassing defences

Aware developers add validation. Three families — and how each falls.

### Deny lists (block known-bad)

Blocks specific addresses/patterns (localhost, `127.0.0.1`, metadata) but permits everything else — inherently fragile because `127.0.0.1` has many alternate representations:

| Representation | Value |
| --- | --- |
| Standard | `127.0.0.1` |
| Decimal | `2130706433` |
| Octal | `017700000001` |
| Shorthand | `127.1`, `0`, `0.0.0.0` |
| Wildcard | `127.*.*.*` |
| IPv6 | `[::1]` |
| DNS-based | `127.0.0.1.nip.io` |

**DNS-based is especially effective:** services like `nip.io` make subdomains that resolve to any IP — `127.0.0.1.nip.io` resolves to `127.0.0.1`, but a string-based deny list just sees a normal domain. Same move defeats metadata blocking: register a domain whose DNS record points at `169.254.169.254`; the deny list checks the *hostname string*, finds no match, and the server resolves it straight to the metadata service.

### Allow lists (permit only known-good)

Denies by default unless the destination matches an approved entry (e.g. must start with `https://website.thm`). Stronger, but weak *string matching* still bypasses:

| Technique | Example | Why it works |
| --- | --- | --- |
| Subdomain matching | `https://website.thm.attackers-domain.thm` | String starts with the expected prefix, but the real host is attacker-controlled. |
| URL credential abuse | `https://website.thm@attacker.com/` | Some libraries treat the part before `@` as credentials and after as host — allow list sees `website.thm`, request goes to `attacker.com`. |

Root cause in both: the app validates the URL **string** with pattern matching instead of **parsing** it into components.

### Open redirects (chain the target's own feature)

When deny/allow bypasses fail, abuse an open redirect on the trusted domain — an endpoint that forwards to a URL in a parameter (common for click tracking):

```text
https://website.thm/link?url=https://tryhackme.com          # normal use
https://website.thm/link?url=http://169.254.169.254/latest/meta-data/   # chained
```

The allow list is satisfied (URL begins with the trusted domain), but the server follows it into the redirect, which forwards to the metadata service. **Two individually-harmless behaviours combined** — a reminder that controls must account for feature *interactions*, not just each feature alone.

---

## Practical: Acme IT Support avatar SSRF

Combines a **hidden-form-field vector** with a **deny-list bypass via directory traversal**.

**Scenario — two discovered endpoints:**

| Endpoint | Behaviour |
| --- | --- |
| `/private` | "Contents cannot be viewed from your IP" — restricted to requests originating from the server itself. |
| `/customers/new-account-page` | Newer account page with a profile-avatar selector — the attack vector. |

**Step 1 — locate the avatar feature.** Create a customer account, sign in, go to `/customers/new-account-page`, and View Source. Each avatar is a radio button whose `value` holds an image **path**; the server fetches whatever path this field contains.

**Step 2 — see how the server handles it.** Select an avatar → Update Avatar. In source, the avatar now renders as a **data URI** with the image **base64-encoded** in `src`. Meaning: the server fetches the path, reads the response, and encodes it into the page — so if we redirect it to `/private`, that page's contents will appear as base64.

**Step 3 — try direct access.** Inspect a radio button, change `value` to `private`, select it, Update Avatar → error: *the path cannot start with `/private`*. There's a **deny list** blocking paths that begin with `/private`.

**Step 4 — bypass with traversal.** The deny list string-matches the **raw** input, but the web server normalises the path *after*. Set the value to:

```text
x/../private
```

| Stage | Path | Explanation |
| --- | --- | --- |
| Input validation | `x/../private` | Doesn't begin with `/private`, so the deny list passes it. |
| Path normalisation | `/private` | Server enters dir `x`, `../` moves up one level → arrives at `/private`. |

The validator and the server interpret the path at **different stages** (check-before-normalise), so the traversal slips through. Select the modified button → Update Avatar → request succeeds.

**Step 5 — decode the flag.** The avatar `<img>` now holds base64 of `/private`. Copy it and decode:

```bash
echo "PASTE_BASE64_STRING_HERE" | base64 -d
```

The decoded output contains the flag.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>The reusable lesson:</strong> whenever a filter checks input at one stage and the system acts on it at another (validate-then-normalise, validate-then-redirect, validate-then-resolve-DNS), the gap between those stages is the bypass. This is the same class of bug as the allow-list <code>@</code> trick and the open-redirect chain above.
</div>

---

## Mitigation

- **Parse, don't pattern-match.** Validate URLs by parsing them into scheme/host/port and checking the **resolved host**, never by string prefix/substring matching.
- **Prefer allow lists over deny lists** — permit only the specific hosts/ports the feature needs; deny lists always miss an encoding (decimal/octal/IPv6/DNS).
- **Validate *after* normalisation and DNS resolution** — resolve the hostname, confirm the final IP isn't private/loopback/link-local (`127.0.0.0/8`, `169.254.0.0/16`, RFC1918), and re-check on every redirect hop (or disable redirects).
- **Block the cloud metadata endpoint** at the network layer and enforce **IMDSv2** (session-token metadata) on AWS.
- **Segment the network** so the app server can't reach sensitive internal services it doesn't need; require authentication on internal services rather than trusting source IP.
- **Restrict schemes/ports** to `https` (and only needed ports); reject `file://`, `gopher://`, etc.

---

## Key takeaways

- **SSRF = make the server request something for you**, borrowing the trust internal systems grant its IP. The prize is usually internal-only endpoints or `169.254.169.254` cloud credentials.
- **Four vectors:** full URL, partial URL (host), path-only (traversal), and hidden form fields — always test parameters that aren't in the address bar too.
- **Regular vs blind:** if the fetched body comes back you read it directly; if not, confirm with a callback (Collaborator / requestbin / `python3 -m http.server`), timing, or error differences.
- **Deny lists fall to alternate IP encodings and DNS tricks** (`2130706433`, `[::1]`, `127.0.0.1.nip.io`, a domain pointing at `169.254.169.254`).
- **Allow lists fall to string-matching flaws** — `website.thm@attacker.com`, `website.thm.attacker.thm`, or chaining a trusted-domain **open redirect** into the metadata service.
- **Validate-then-act gaps are the master pattern:** the Acme lab's `x/../private` beats a deny list that checks *before* the server normalises the path — the same idea as the `@` and open-redirect bypasses.
- **Defence is parse-and-resolve validation + allow lists + post-redirect re-checks + network segmentation + blocking metadata**, not string filters.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>SQLi vs XSS vs SSRF in one line:</strong> all three inject attacker input into a trusted context — SQLi into a <em>database query</em>, XSS into a <em>page other users load</em>, SSRF into a <em>request the server makes</em>. SSRF's twist is that the victim is the server's own network position, so the fix is validating the <em>destination</em>, the way SQLi/XSS fix the <em>data-vs-code</em> boundary.
</div>
