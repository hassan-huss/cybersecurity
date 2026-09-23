# Insecure Direct Object Reference (IDOR)

Changing a reference the app trusts — a number in a URL, an encoded blob in a cookie, an `id` in a background API call — to reach an object that isn't yours. The server looks up whatever you asked for and hands it back **without checking you're allowed to have it**. Simple to do, severe in impact.

> This is the **TryHackMe "IDOR" room**. It covers what IDOR is (and its OWASP naming), the forms an object reference takes (plaintext, encoded, hashed, unpredictable), where the vectors hide, and a practical against the Acme IT Support customer API.
>
> 🔗 Sibling labs in this folder: [SQLi](../SQLi/SQL-Injection-Lab.md), [XSS](../XSS/XSS.md), [SSRF](../SSRF/SSRF.md). Those inject *code/requests*; IDOR injects **nothing** — it's a pure **missing authorisation** bug.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What IDOR is](#what-idor-is)
- [Authentication vs authorisation](#authentication-vs-authorisation)
- [Forms an object reference takes](#forms-an-object-reference-takes)
  - [Plaintext IDs](#plaintext-ids)
  - [Encoded references](#encoded-references)
  - [Hashed references](#hashed-references)
  - [Unpredictable references](#unpredictable-references)
- [Where IDOR vectors hide](#where-idor-vectors-hide)
- [Practical: Acme customer API](#practical-acme-customer-api)
- [Mitigation](#mitigation)
- [Key takeaways](#key-takeaways)

---

## In plain English

Apps give every thing — your profile, your invoice, your ticket — an internal **reference** (usually a number). When the app lets *you* supply that reference and then fetches the matching object **without asking "is this yours?"**, you can just point at someone else's. Read their data, or on a write endpoint, change their email/password/resources.

The whole bug in one sentence: **the app checks *who you are* (login) but not *what you're allowed to touch* (permission).**

| Form of the ID | How you exploit it |
| --- | --- |
| **Plaintext** (`?user_id=1305`) | Change the number. |
| **Encoded** (`MTIz`) | Decode → edit → re-encode → resend. |
| **Hashed** (`202cb96…`) | Guess the scheme (hash small integers / crack it), reproduce valid hashes. |
| **Unpredictable** (a UUID) | Can't enumerate — use the **two-account** test: leak A's ID, request it as B. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>IDOR</strong> = Insecure Direct Object Reference. <strong>Object reference</strong> = the identifier the app uses to locate a record (id, invoice number, file path). <strong>Access control</strong> = the rules for who may do what. <strong>Authentication</strong> = proving who you are; <strong>authorisation</strong> = deciding what you're allowed to do. <strong>Broken Access Control</strong> = OWASP Top 10 #1, the category IDOR lives in. <strong>BOLA</strong> = Broken Object Level Authorisation, the same bug's name in the OWASP API Security Top 10. <strong>Enumeration</strong> = walking through IDs (1, 2, 3…) to harvest objects.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not rote memorisation. The single idea to keep: <em>encoding and hashing an ID only make it harder to guess — they never add an authorisation check</em>. The fix is always server-side ownership verification, no matter how the ID looks.
</div>

---

## What IDOR is

Web apps use identifiers to tell objects apart — a profile, an invoice, a ticket, a document each has a reference (a number or string) the app uses internally to find it. When the app lets the **user supply that reference** and then retrieves the object **without checking the user is permitted to access it**, that's an **Insecure Direct Object Reference**.

- It's an **access control** vulnerability, sitting in **Broken Access Control** — **#1 in the OWASP Top 10**.
- The same flaw is **Broken Object Level Authorisation (BOLA)** in the **OWASP API Security Top 10**. Different name, identical root cause: *the server doesn't verify the authenticated user may interact with the specific object requested.*

What makes IDOR notable is the gap between **how easy** it is (often just change a number) and **how bad** it can be (mass data disclosure → full account takeover). No injection, no session hijacking, no special tooling required.

---

## Authentication vs authorisation

Take the vulnerable profile page:

```text
http://online-service.thm/profile?user_id=1305     # your record → your details
http://online-service.thm/profile?user_id=1000     # someone else's record → their details
```

If changing `1305` → `1000` returns a different user's profile, it's IDOR. Why it happens:

- **Authentication works** — the app knows who you are; you have a valid session.
- **Authorisation is missing** — there's no server-side check asking *"does this session own record 1000?"* / *"is this user allowed to view it?"* The server assumes any authenticated user may access any object they reference.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Read vs write — why severity varies:</strong> the same missing check on a <em>write</em> endpoint is far worse. If <code>user_id</code> also drives "update email" or "reset password", changing it lets you modify <em>another</em> user's account — turning data disclosure into <strong>account takeover</strong>. Always test whether the vulnerable parameter reaches write actions, not just reads.
</div>

---

## Forms an object reference takes

### Plaintext IDs

The simplest case — the identifier sits directly in a parameter and is used to query the database with no auth check. Just change the value (`user_id=1305` → `1000`). Sequential integers also let you **enumerate** every record.

### Encoded references

Developers often **encode** IDs before putting them in query strings, POST data, or cookies — usually **base64** (uses `a-z A-Z 0-9` and `=` padding; strings look longer than the value and often end in `=`). Examples:

```text
123            →  MTIz
{"user_id": 5} →  eyJ1c2VyX2lkIjogNX0=
```

Four-step exploit:

1. **Decode** — `echo 'value' | base64 -d` (or base64decode.org).
2. **Modify** — change the ID (e.g. `5` → `1`).
3. **Re-encode** — `echo 'value' | base64` (or base64encode.org).
4. **Substitute** the new string back into the request and submit.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Encoding is not security.</strong> base64 is a reversible transformation, not encryption — anyone with the string can decode it. An app relying on encoded IDs without server-side authorisation is <em>exactly</em> as vulnerable as one using plaintext IDs.
</div>

### Hashed references

Some apps **hash** the ID → a fixed-length hex string that *looks* obscured. But if the input is predictable (e.g. a sequential integer), you can reproduce the hashing and forge valid references.

```text
MD5(123) = 202cb962ac59075b964b07152d234b70
```

- Suspect sequential integers? Hash `1, 2, 3, …` yourself and compare to the hashes seen in requests. A match confirms the scheme and makes **every object reachable**.
- Hashing is **one-way** (no math reverses MD5), but for short/predictable inputs that barely matters: **CrackStation** and similar keep billions of precomputed hash→value pairs, so looking up a hashed small integer is instant.

**Identify the algorithm by hash length** (then reproduce/crack):

| Algorithm | Hex length |
| --- | --- |
| MD5 | 32 |
| SHA-1 | 40 |
| SHA-256 | 64 |

Tools: `hash-identifier` / `hashid` (Kali) automate the guess. Hashing an identifier **does not** replace an authorisation check — once the scheme is understood, forging references is as easy as plaintext.

### Unpredictable references

Random strings / **UUIDs** (e.g. `d3b07384-d9a0-4e9b-8b3c-2f1a6c7e4a90`) can't be incremented or hashed into other users' IDs — that removes **enumeration**, but **not the IDOR risk**. If the server still doesn't verify ownership, the bug remains; the attacker just needs a valid ID from another channel.

**The two-account technique:**

1. Create **Account A** and **Account B**.
2. Log into **A**, record its resource identifiers (profile IDs, order refs, document paths…).
3. Log into **B**, substitute **A's** identifiers into **B's** requests.
4. If B receives A's data → the endpoint lacks authorisation checks.

This separates two questions: *(1) can an attacker obtain a valid ID for someone else's object?* and *(2) will the app serve it without checking permission?* **The app is only secure if the answer to #2 is "no" — regardless of how hard #1 is.**

In practice, "unguessable" IDs leak anyway — via shared URLs, API responses that reference other users' resources, HTML source, JavaScript files, notification emails, or exported CSV/reports. Once you have a valid ID from any of these, exploitation is identical to the plaintext case.

---

## Where IDOR vectors hide

Testing only the address bar misses a large share of IDOR flaws. Look everywhere the app processes a user-supplied reference:

- **Background (AJAX) requests** — pages load data via async calls that never show in the address bar but are fully visible in DevTools → **Network** and in Burp. E.g. "Your Account" quietly calls `/api/v1/customer?id=15`; if `id` isn't tied to the session, it's exploitable like any URL param.
- **JavaScript files** — often reference API endpoints and parameter names not exposed in the UI. Reviewing them (by hand or tooling) reveals endpoints the developer didn't mean for you to hit directly.
- **Parameter mining** — some endpoints honour parameters the front end never sends (leftover debug/test params). `/user/details` might normally use only your session cookie, but appending `?user_id=123` may make it return another user's record. Fuzzing for such hidden params uncovers otherwise-invisible IDOR.

**Common locations to test:** query-string parameters, POST body data, cookie values, HTTP request headers, REST API **path segments** (`/api/users/123/orders`), and background AJAX requests. **Principle: intercept and inspect every request the browser makes — not just what's in the address bar.**

---

## Practical: Acme customer API

Goal: find an API endpoint vulnerable to IDOR and read other users' data.

**1. Create an account.** Under **Customers**, sign up (any details) and log in. Go to **Your Account** — it lets you change username, email, and password, pre-filled with your registration details.

**2. Find the vulnerable endpoint.** Open DevTools (**F12**) → **Network** tab → refresh. Among the requests is:

```text
/api/v1/customer?id={user_id}
```

Click it — the response is a JSON object with your **user ID, username, and email**. The data returned is decided **entirely by the `id` query parameter** → a direct object reference to your record.

**3. Exploit.** Right-click the request → **Edit and Resend** (Firefox), or replay it with Burp / `curl`, changing `id`:

```bash
curl "https://LAB_WEB_URL.p.thmlabs.com/api/v1/customer?id=1"
```

Set `id=1` and submit — a **different** user's details come back → no authorisation check (the server trusts the `id` without verifying the session owns it). Repeat with `id=3` to pull that user's record (answers the room's second question).

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why the Network tab mattered here:</strong> the <code>id</code> parameter is in a background API call, not the page URL — this is exactly the "background requests" vector. The habit of refreshing with DevTools open and reading every request is what surfaces it.
</div>

---

## Mitigation

- **Enforce authorisation server-side on every object access** — for each request, verify the authenticated session **owns or is permitted** the specific object before returning or modifying it. This is the only real fix.
- **Don't rely on obscurity** — encoding (base64) and hashing (MD5/SHA) only slow guessing; they add no access control. Never treat "hard to guess" as "protected".
- **Use unpredictable identifiers (UUIDs) *as defence-in-depth*, not the control** — they stop enumeration but must still be paired with ownership checks.
- **Scope queries to the current user** — prefer deriving the record from the **session** (e.g. "the logged-in user's profile") over accepting a client-supplied ID; when an ID is needed, filter by `WHERE id = ? AND owner = <session_user>`.
- **Apply checks to writes too** — update/delete/reset endpoints need the same ownership check as reads; a missing one there means account takeover.
- **Centralise access control** — enforce it in a common layer/middleware so no endpoint is forgotten, and test with the two-account technique.

---

## Key takeaways

- **IDOR = missing authorisation on a user-supplied object reference.** Auth (who you are) works; authz (what you may touch) is absent. It's OWASP **Broken Access Control #1** / API **BOLA**.
- **Encoding and hashing are not security.** base64 is reversible; hashed small integers are guessable/crackable (CrackStation). Identify a hash by length: 32=MD5, 40=SHA-1, 64=SHA-256.
- **Unpredictable IDs (UUIDs) only stop enumeration**, not IDOR — use the **two-account technique** (leak A's ID, request it as B) to test, since "unguessable" IDs leak through URLs, APIs, JS, emails, and exports.
- **Look beyond the address bar:** background AJAX calls (DevTools → Network), JavaScript files, and **parameter mining** (hidden/leftover params like `?user_id=`); test query strings, POST bodies, cookies, headers, and REST path segments.
- **Write endpoints are the high-severity case** — the same missing check on "change email"/"reset password" is account takeover, not just disclosure.
- **The fix is always server-side ownership verification**, ideally centralised, with IDs scoped to the session.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Where IDOR sits among the others:</strong> SQLi/XSS/SSRF are <em>injection</em> bugs — you smuggle code or a request into a trusted context. IDOR injects nothing; it exploits a check the app <em>never wrote</em>. That's why no payload cleverness helps the defender — only an explicit "does this user own this object?" on the server does.
</div>
