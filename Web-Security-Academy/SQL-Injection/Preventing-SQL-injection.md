# How to prevent SQL injection

The primary defence is **parameterized queries** (a.k.a. prepared statements) instead of building queries by **string concatenation**. The parameter placeholder keeps user input as *data*, so it can never change the *structure* of the query.

> Part of the [SQL Injection](README.md) path. This is the defensive counterpart to every exploitation note in this folder.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [The vulnerable pattern: string concatenation](#the-vulnerable-pattern-string-concatenation)
- [The fix: parameterized queries](#the-fix-parameterized-queries)
- [Where parameters don't apply](#where-parameters-dont-apply)
- [Rules for parameters to actually work](#rules-for-parameters-to-actually-work)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Parameterized queries** | Send the query and the data to the database *separately*, with a `?` placeholder where the data goes | The database already knows the query's shape before it sees your input, so input can only be a value — never new SQL. This is **the** fix. |
| **Where it doesn't apply** | Placeholders only work for **data** (WHERE values, INSERT/UPDATE values), not for table/column names or `ORDER BY` | For those, use an **allowlist** of permitted values instead. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet. The one rule that prevents most SQLi: <em>never concatenate untrusted input into a query string — bind it as a parameter.</em>
</div>

---

## The vulnerable pattern: string concatenation

Here the user input is glued **directly** into the query text. The input can contain quotes and SQL that break out of the data and rewrite the query:

```java
String query = "SELECT * FROM products WHERE category = '" + input + "'";
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(query);
```

Because `input` becomes part of the query *string*, a value like `Gifts' OR 1=1--` changes what the query *does*, not just what it filters on.

---

## The fix: parameterized queries

Rewrite so the input can't interfere with the query **structure**. The `?` is a placeholder; `setString` binds the input as a pure value:

```java
PreparedStatement statement =
    connection.prepareStatement("SELECT * FROM products WHERE category = ?");
statement.setString(1, input);
ResultSet resultSet = statement.executeQuery();
```

Now the database parses the query with the placeholder in place *first*, then plugs in `input` as data. Quotes, `OR 1=1`, `--` — all of it is treated as a literal category value, so there's nothing to inject.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why "prepared statement" = safe:</strong> the query's grammar is locked in before your data arrives. Parameter binding can add a value but can never add a keyword, quote-break, or comment that the parser will act on.
</div>

---

## Where parameters don't apply

Parameterized queries cover any spot where untrusted input appears **as data** — including `WHERE` clauses and the values in `INSERT` / `UPDATE` statements.

They **cannot** be used for other parts of the query, such as:

- **Table or column names**
- The **`ORDER BY`** clause

Functionality that must place untrusted data into those positions needs a different approach:

- **Allowlist** the permitted input values (e.g. map a user-supplied sort key to a fixed set of real column names).
- **Use different logic** to deliver the required behaviour without putting raw input into the query at all.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Common blind spot:</strong> a "sort by" dropdown that feeds a column name straight into <code>ORDER BY</code> can't be protected by a placeholder. Validate it against a hard-coded allowlist of column names instead.
</div>

---

## Rules for parameters to actually work

A parameterized query only protects you if it's used correctly:

- The query string **must be a hard-coded constant**. It must **never** contain variable data from *any* origin.
- **Don't** decide case-by-case that some piece of data is "trusted" and concatenate it while parameterising the rest. It's easy to be wrong about where data really came from, and a later code change can taint data you'd assumed was safe.
- Apply parameters **everywhere** untrusted data enters a query as a value — consistency is the point. One concatenated "safe" exception is all it takes.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Defence in depth (beyond the primary fix):</strong> least-privilege database accounts, avoiding verbose DB errors in responses, and input validation all reduce impact — but they're backups. Parameterized queries are the fix that actually closes the vulnerability.
</div>
