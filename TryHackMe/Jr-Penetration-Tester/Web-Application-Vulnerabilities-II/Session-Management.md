# Session Management

How a web app remembers *who you are* between requests after you log in — and everything that can go wrong from the moment a session is handed out to the moment it should die. Break any stage and an attacker can guess, steal, reuse or outlive your session: **session hijacking**.

> This is the **TryHackMe "Session Management" room** (Jr. Penetration Tester → Web Application Vulnerabilities II). It covers the session lifecycle, the IAAA model, cookies vs tokens, the vulnerabilities at each lifecycle stage, and a practical that maps the lifecycle of a student/lecturer web app.
>
> 🔗 Same module (**Web Application Vulnerabilities II**): [Broken Authentication](./Broken-Authentication.md) (the attack side of these session flaws), [File Inclusion](./File-Inclusion.md). Related standalone labs: [IDOR](../../Vulnerabilities/IDOR/IDOR.md) (a horizontal authorisation bypass is exactly an IDOR), [XSS](../../Vulnerabilities/XSS/XSS.md) (the classic way to *steal* a session cookie — and why `HttpOnly` exists).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Why sessions exist](#why-sessions-exist)
- [The session management lifecycle](#the-session-management-lifecycle)
- [The IAAA model](#the-iaaa-model)
- [Cookies vs tokens](#cookies-vs-tokens)
  - [Cookie-based sessions](#cookie-based-sessions)
  - [Token-based sessions](#token-based-sessions)
  - [Side by side](#side-by-side)
- [What goes wrong at each stage](#what-goes-wrong-at-each-stage)
  - [Creation](#creation)
  - [Tracking](#tracking)
  - [Expiry](#expiry)
  - [Termination](#termination)
- [Practical: mapping the lifecycle](#practical-mapping-the-lifecycle)
- [Defences](#defences)
- [Key takeaways](#key-takeaways)

---

## In plain English

You don't type your password on every click. Instead, after you log in, the site gives your browser a **ticket** (the session value) and your browser shows that ticket with every request. Whoever holds the ticket *is* you, as far as the server is concerned. So session security is really just: **is the ticket hard to fake, hard to steal, checked properly, and does it stop working when it should?**

| Stage | What should happen | The bug in one line |
| --- | --- | --- |
| **Creation** | You log in → get a fresh, random, unforgeable ticket | Ticket is guessable, forgeable, or the *same* one you had before login |
| **Tracking** | Every request: "whose ticket is this, and are they allowed?" | Server checks you're logged in but not whether you may do *this* |
| **Expiry** | Ticket stops working after a sensible time | Ticket lives for days/weeks |
| **Termination** | Logout kills the ticket **on the server** | Logout only deletes it from your browser — a stolen copy still works |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>Session</strong> = the server's memory of a logged-in user, identified by a value you send each request. <strong>Stateless (HTTP)</strong> = each request stands alone; the protocol itself forgets you between requests. <strong>Session hijacking</strong> = using someone else's session value to act as them. <strong>Cookie</strong> = a small value the browser stores and sends back automatically. <strong>Token</strong> = a session value your page's JavaScript stores and attaches manually. <strong>JWT</strong> = JSON Web Token, a common self-contained token format. <strong>LocalStorage</strong> = browser storage that JavaScript can read/write. <strong>CSRF</strong> = Cross-Site Request Forgery, tricking your browser into sending a request (with your cookies) that you didn't intend. <strong>Server-side</strong> = on the web server; <strong>client-side</strong> = in your browser, where the user (or attacker) controls it.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not rote memorisation. When testing, walk the four stages in order and ask the question in the table above for each one — that's the whole methodology.
</div>

---

## Why sessions exist

HTTP is **stateless** — the server doesn't inherently know two requests came from the same person. Sending the username + password on every request would be terrible, so after authentication the app issues a **session value** that you send instead. The server uses it to:

- **keep your state** (who you are, what's in your cart),
- **track your actions** (for logging/accountability),
- **decide what you're allowed to do** (authorisation).

**Session management** = generating, tracking, expiring and destroying these values securely.

---

## The session management lifecycle

```text
  Creation ──► Tracking ──► Expiry
     ▲            │            │
     │            ▼            ▼
     └──── (log in again) ◄── Termination (logout)
```

| Phase | What happens |
| --- | --- |
| **Creation** | Session value issued. Often *before* login too (apps tracking anonymous visitors) — but the one that matters is issued after authentication. How it's generated, sent and stored is critical. |
| **Tracking** | Value sent with every request → server looks it up server-side to learn *who* and *what permissions*. |
| **Expiry** | Because HTTP is stateless, the server can't tell you closed the tab. So every session needs a **lifetime**; an expired value must be rejected and the user sent back to login. |
| **Termination** | User clicks **logout** → session must be killed even if its lifetime hasn't run out. Failure here = attacker keeps persistent access. |

---

## The IAAA model

Authentication and authorisation get confused constantly; IAAA separates the four jobs:

| Step | Question | Web example | Session role |
| --- | --- | --- | --- |
| **Identification** | Who do you *claim* to be? | Typing a username / email | — |
| **Authentication** | *Prove* it | Supplying the matching password | Success → **session creation** |
| **Authorisation** | Are you *allowed* to do this? | Viewing vs modifying data | **Session tracking** looks up the session's permissions on every request |
| **Accountability** | Record what you did | Logs of every action per session | Log each request **with its session** so incidents can be reconstructed |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>One-liner to keep:</strong> authentication decides whether you <em>get</em> a session; authorisation decides what that session may <em>do</em>; accountability records what it <em>did</em>.
</div>

---

## Cookies vs tokens

### Cookie-based sessions

The "old-school" approach. The server sends a `Set-Cookie` response header; the browser stores it and **automatically** attaches it to matching requests.

```http
Set-Cookie: session=12345; Secure; HttpOnly; SameSite=Lax; Expires=Wed, 01 Oct 2026 12:00:00 GMT
```

| Attribute | Tells the browser… | Protects against |
| --- | --- | --- |
| `Secure` | Only send over valid HTTPS (not HTTP, not with cert errors) | Sniffing the cookie on the network |
| `HttpOnly` | JavaScript may **not** read this cookie | Stealing it via XSS (`document.cookie`) |
| `Expires` / `Max-Age` | When to delete it | Long-lived stolen cookies (client-side only!) |
| `SameSite` | Whether to send it on cross-site requests | CSRF |

The key trait: **the browser decides** when to send the cookie (based on domain + attributes) — no JavaScript involved.

### Token-based sessions

The newer approach. After login, the app returns a token **in the response body**; client-side JavaScript stores it (usually in **LocalStorage**) and must **manually** attach it to each request, typically:

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The most common format is the **JWT**. Because no browser feature governs this, it's "the wild west" — standards exist but nothing forces apps to follow them.

### Side by side

| | Cookies | Tokens |
| --- | --- | --- |
| **Sending** | Browser attaches automatically | JavaScript must attach as a header |
| **Built-in protection** | Attributes (`Secure`, `HttpOnly`, `SameSite`) | None automatic — must be guarded against disclosure (e.g. XSS can read LocalStorage) |
| **CSRF** | Vulnerable (browser auto-sends them) | Resistant — not auto-sent, and other domains can't read your LocalStorage |
| **Decentralised / multi-domain apps** | Awkward — cookies are locked to a domain | Good fit — can carry everything needed to verify themselves |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>The trade-off is one coin:</strong> the <em>automatic</em> sending that makes cookies vulnerable to CSRF is the same feature that lets <code>HttpOnly</code> hide them from JavaScript. Tokens flip both: immune to CSRF, but any XSS can read them straight out of LocalStorage.
</div>

---

## What goes wrong at each stage

### Creation

*Where the most vulnerabilities creep in.*

| Vulnerability | What it looks like | Why it's bad |
| --- | --- | --- |
| **Weak session values** | Custom scheme, e.g. `session = base64(username)` | Reverse-engineer the scheme → generate anyone's session. Frameworks mostly killed this, but AI-generated code is bringing it back. |
| **Controllable session values** | JWT whose signature isn't verified, or is signed with a weak/guessable secret | Attacker forges their own token with any identity/role (covered in a later JWT room). |
| **Session fixation** | App issues a session *before* login and **keeps the same value after** login | Attacker learns/plants your pre-login value, waits for you to log in, then uses it — now authenticated as you. Fix: **rotate the session on login**. |
| **Insecure session transmission** | SSO: auth server hands session material to the app server *via your browser redirect*; redirect URL is attacker-controllable | Your session gets delivered to the attacker's URL. Happened in Oracle's real SSO product. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Quick test for weak values:</strong> log in a few times (and as a couple of users) and compare session values. Decode anything that looks like base64 (<code>echo 'value' | base64 -d</code>). If you can see a username, ID, timestamp or counter, the value is predictable.
</div>

### Tracking

*The second-biggest source of bugs.*

**Authorisation bypass** — the session is tracked, but its rights aren't checked properly.

| Type | Meaning | Example |
| --- | --- | --- |
| **Vertical** | Do something reserved for a **more privileged** role | Student hits a lecturer-only endpoint |
| **Horizontal** | Do an action you *are* allowed, but on **someone else's** data | Student views *another* student's grades — this is [IDOR](../../Vulnerabilities/IDOR/IDOR.md) |

Vertical is usually easy to defend (function decorators, path-based access rules). Horizontal needs real code: take the **user from the session**, compare to the **data requested**, decide.

**Insufficient logging** — if actions can't be traced to a session and a session to a user, incident investigations have gaps. Log **accepted *and* rejected** actions: a hijacked session's actions look perfectly legitimate, so rejected-only logs miss the attack entirely.

### Expiry

One vulnerability: **excessive session lifetime**. A session is like a movie ticket — valid for tonight's showing, not every showing forever. Match lifetime to the app's risk:

- Banking app → **short** lifetime.
- Webmail → longer is acceptable, **but** the session should be bound to where it's used; if the location suddenly changes (a hijack signal), terminate it.

### Termination

Key issue: **logout doesn't invalidate the session server-side** (it only deletes the browser's copy). Then if an attacker has stolen the session, the victim has **no way to kick them out**.

- Tokens with the lifetime baked in (JWTs) are hard to revoke → keep a server-side **blocklist** of logged-out tokens.
- Good practice: a page listing **all active sessions** with the ability to terminate each.
- On **password reset**, terminate **all** sessions so the user fully regains control.

---

## Practical: mapping the lifecycle

Target: a university-style app with **Student** (open sign-up) and **Lecturer** (needs a verification code) accounts. Tools: browser DevTools (Network + Storage tabs) or Burp.

**Step-by-step enumeration**

1. **Visit unauthenticated** → no cookies/tokens set → anonymous visitors aren't tracked.
2. **Sign up as a student** (no brute force needed to start the lifecycle) → log in with **Network** tab open.
3. **Login response** shows a `Set-Cookie` with **`HttpOnly`** → cookie-based sessions, and JavaScript can't read the cookie.
4. **Browse the dashboard** → the cookie rides along on every request. Each response **re-sends the same `Set-Cookie`** with a pushed-out expiry → the lifetime keeps extending (a persistent cookie).
5. **Delete the cookie (Storage tab) and replay the modules request** → it *still succeeds*, but returns less data (modules, but no enrolled-student counts or tests) → some endpoints work **without any session**; worth mapping exactly what.
6. **Log out** → cookie removed client-side. Re-login, swap the **old** cookie back in, refresh → **500 Internal Server Error**. The old session *is* rejected server-side, but an invalid session should give a clean 401/redirect, not a crash — a sign of fragile handling.
7. **Check Local Storage** → lots of data stored there: `userRole`, and a `user` object with `id`, `username`, `role`.
8. **Tamper**: `userRole` `student` → `lecturer` and refresh → **extra tabs appear** but no extra data. Changing `role` `2` → `3` → same result. `id` and `username` are still untested.

→ Conclusion: sessions are a **hybrid** — a cookie *plus* client-side token data that decides what the UI shows.

**Lifecycle map**

| Phase | Observation |
| --- | --- |
| Creation | Hybrid cookie + token session management |
| Creation | New session value on every login (good — no fixation) |
| Creation | Session value appears sufficiently random |
| Tracking | Unauthenticated actions aren't tracked |
| Tracking | Cookie sent on every request |
| Tracking | **Several requests succeed with no cookie at all** |
| Tracking | **Client-side token values loaded at login dictate what's visible** |
| Expiry | Client-side and server-side expiry times don't match |
| Expiry | **Expiry time is extremely long** |
| Termination | Users can log out |
| Termination | Sessions are killed server-side, but reusing an old one causes a 500 error |

**Reportable already:** Excessive Session Lifetimes.
**Needs further investigation:** access control on **every API endpoint**, and application logging for accountability.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Exploitation leads (the room's challenge: get lecturer data as a student):</strong> the map points at two weak spots. (1) The UI trusts <strong>client-side</strong> values in Local Storage — the untested <code>id</code> / <code>username</code> fields are the next thing to tamper with. (2) Some <strong>API endpoints answer without a session</strong> — so find the lecturer-only API calls (log them in the Network tab after flipping <code>userRole</code> to <code>lecturer</code>) and replay them directly, with and without your student cookie. If the server returns lecturer data, that's a <strong>vertical authorisation bypass</strong>: the server never checked the session's role.
</div>

---

## Defences

- **Store session values securely** — cookies with `Secure` + `HttpOnly` + `SameSite`; tokens kept out of reach of XSS.
- **Unguessable or signed values** — cryptographically random IDs, or a signature (verified server-side!) so they can't be tampered with.
- **Rotate the session on login** — kills session fixation.
- **Authorise every request server-side from the session** — role *and* ownership; never trust client-side role/ID values.
- **Sensible expiry**, enforced **server-side**, matched to the app's risk.
- **Logout = delete client-side *and* invalidate server-side**; blocklist revoked tokens; kill all sessions on password reset.
- **Log accepted and rejected actions** with the session attached.

---

## Key takeaways

- **HTTP is stateless → sessions** carry your identity; whoever holds the value *is* you.
- **Four lifecycle stages to test:** creation, tracking, expiry, termination. Most bugs are in **creation**, then **tracking**.
- **IAAA:** identification (claim) → authentication (prove) → authorisation (allowed?) → accountability (logged).
- **Cookies** are auto-sent and protected by attributes (`Secure`, `HttpOnly`, `SameSite`, `Expires`) but face CSRF; **tokens** (JWT in `Authorization: Bearer`) resist CSRF but have no built-in protection.
- **Creation bugs:** weak values (e.g. base64 username), forgeable tokens (unverified JWT), **session fixation** (no rotation on login), insecure SSO redirects.
- **Tracking bugs:** vertical (higher role) vs horizontal (other user's data = IDOR) authorisation bypasses; logs that miss accepted actions.
- **Expiry:** too long is the bug. **Termination:** logout that doesn't kill the session server-side.
- **In testing:** map the lifecycle in DevTools first — delete cookies, replay old ones, tamper Local Storage — then attack the gaps. Anything the **client** decides (role, id) is attacker-controlled.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How this ties to the other notes:</strong> <a href="../../Vulnerabilities/XSS/XSS.md">XSS</a> is how sessions get <em>stolen</em>; <a href="../../Vulnerabilities/IDOR/IDOR.md">IDOR</a> is the horizontal case of <em>tracking</em> failing. Session management is the layer both of them are really attacking.
</div>
