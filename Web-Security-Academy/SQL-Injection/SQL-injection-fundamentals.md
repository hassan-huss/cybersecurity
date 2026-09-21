# SQL injection fundamentals

**SQL injection (SQLi)** is a web vulnerability that lets an attacker interfere with the queries an application sends to its database — reading, modifying, or deleting data they shouldn't, and sometimes compromising the server behind it.

> Part of the [SQL Injection](README.md) path. The starting point — *what* SQLi is, *why* it matters, and *where* it shows up — before the exploitation techniques in the rest of this folder. Source: [PortSwigger Web Security Academy — SQL injection](https://portswigger.net/web-security/sql-injection).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What is SQL injection?](#what-is-sql-injection)
- [Impact of a successful attack](#impact-of-a-successful-attack)
- [Where injection occurs in a query](#where-injection-occurs-in-a-query)
- [Example 1 — retrieving hidden data](#example-1--retrieving-hidden-data)
- [Example 2 — subverting application logic (login bypass)](#example-2--subverting-application-logic-login-bypass)
- [Other kinds of SQLi](#other-kinds-of-sqli)
- [Labs](#labs)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **SQL injection** | You send input that breaks out of the *data* part of a query and becomes part of the *query itself* | If user input is glued into a SQL string, a well-placed `'` and some SQL rewrites what the query does. |
| **Impact** | You can read secrets, change/delete data, bypass logins, sometimes reach the server | It's high severity — a single injectable field can expose the whole database. |
| **Where** | Anywhere input reaches a query: `WHERE`, `INSERT`/`UPDATE` values, table/column names, `ORDER BY` | Test *every* input, not just the obvious search box. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not something to memorise. Hold the core idea — <em>input becomes query</em> — and come back for the example payloads.
</div>

---

## What is SQL injection?

> "SQL injection (SQLi) is a web security vulnerability that allows an attacker to interfere with the queries that an application makes to its database."

It generally lets an attacker view data they're not normally able to retrieve — other users' data, or anything the application can access. Attackers can also **modify or delete** that data, and in some cases **escalate** to compromising the underlying server or launching a denial-of-service attack.

---

## Impact of a successful attack

A successful SQLi attack can result in:

- **Unauthorised access to sensitive data** — passwords, credit card details, personal user information.
- **Data tampering** — modifying or deleting records, causing persistent changes to application content or behaviour.
- **Deeper compromise** — in some cases a **persistent backdoor**, giving long-term, potentially undetected access.
- **Business fallout** — many high-profile breaches were SQLi, resulting in reputational damage and regulatory fines.

---

## Where injection occurs in a query

The vulnerable pattern is user input concatenated into a query. That input can land in several parts of a SQL statement:

- The **`WHERE`** clause of a `SELECT`
- **Values** in an `INSERT` or `UPDATE` statement
- **Table or column names**
- The **`ORDER BY`** clause

Most obvious cases are in a `SELECT`'s `WHERE` clause, but any of the above can be injectable.

---

## Example 1 — retrieving hidden data

A shopping app shows products in a category via:

```
https://insecure-website.com/products?category=Gifts
```

```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1
```

The `AND released = 1` hides unreleased products. Inject a comment to drop that restriction:

```
https://insecure-website.com/products?category=Gifts'--
```

```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```

The `--` comments out the rest of the query, so `AND released = 1` never applies — **unreleased products are now returned**. Going further, `category=Gifts'+OR+1=1--` returns *every* product, released or not:

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--'
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Careful with <code>OR 1=1</code>:</strong> on a <code>SELECT</code> it just returns extra rows, but if the same trick hits an <code>UPDATE</code> or <code>DELETE</code>, <code>OR 1=1</code> can affect <em>every</em> row. Understand the surrounding statement before firing it.
</div>

---

## Example 2 — subverting application logic (login bypass)

A login runs:

```sql
SELECT * FROM users WHERE username = 'wiener' AND password = 'bluecheese'
```

If the app logs you in when this returns a user, submit username `administrator'--` with **any/blank** password:

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```

The `--` comments out the password check entirely, so you log in as `administrator` **without knowing the password**.

---

## Other kinds of SQLi

Beyond these first-order `WHERE`-clause examples, this folder covers:

- **[UNION attacks](SQL-injection-UNION-attacks.md)** — read data from other tables when results are reflected.
- **[Examining the database](Examining-the-database.md)** — enumerate DB version, tables, columns.
- **[Blind SQL injection](Blind-SQL-injection.md)** — when neither results nor errors are returned.
- **[Different contexts](SQL-injection-in-different-contexts.md)** — JSON/XML input, WAF bypass.
- **[Second-order](Second-order-SQL-injection.md)** — stored input that fires on a later request.

---

## Labs

> Both **APPRENTICE** — the classic entry-point labs for these two examples.

| Lab | Focus |
| --- | --- |
| SQL injection vulnerability in WHERE clause allowing retrieval of hidden data | `category=Gifts'--` / `OR 1=1--` to reveal hidden products |
| SQL injection vulnerability allowing login bypass | Username `administrator'--` to skip the password check |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Next step:</strong> once you can spot where input hits a query, learn to <em>find</em> injectable inputs systematically — see <a href="Detecting-SQL-injection.md">Detecting SQL injection</a>.
</div>
