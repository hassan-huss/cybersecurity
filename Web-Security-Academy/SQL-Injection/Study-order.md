# SQL injection — study order

The notes in this folder, listed in the **order they were studied**, following the PortSwigger Web Security Academy SQL injection learning path. Read top to bottom to retrace the progression from basic in-band attacks → blind techniques → advanced contexts → prevention.

> This is a reading guide only. The full topic index (alphabetical-ish) lives in [README.md](README.md).

---

## 1 · Fundamentals

Start here — what SQLi is, its impact, where it occurs, and how to find it.

| # | Topic | Notes |
| --- | --- | --- |
| 1 | SQL injection fundamentals (what/impact/where + hidden data & login bypass) | [SQL-injection-fundamentals.md](SQL-injection-fundamentals.md) |
| 2 | Detecting SQL injection | [Detecting-SQL-injection.md](Detecting-SQL-injection.md) |

---

## 2 · Examining the database

Groundwork — fingerprint the DB and enumerate its structure before attacking.

| # | Topic | Notes |
| --- | --- | --- |
| 3 | Examining the database (type/version + listing tables & columns) | [Examining-the-database.md](Examining-the-database.md) |

---

## 3 · UNION attacks (in-band exfiltration)

When query results are reflected in the response.

| # | Topic | Notes |
| --- | --- | --- |
| 4 | SQL injection UNION attacks (column count → text column → retrieve data) | [SQL-injection-UNION-attacks.md](SQL-injection-UNION-attacks.md) |
| 5 | Retrieving multiple values within a single column | [Retrieving-multiple-values-in-a-single-column.md](Retrieving-multiple-values-in-a-single-column.md) |

---

## 4 · Blind SQL injection

When results and errors are **not** returned — infer data indirectly.

| # | Topic | Notes |
| --- | --- | --- |
| 6 | Blind SQL injection (overview) | [Blind-SQL-injection.md](Blind-SQL-injection.md) |
| 7 | Exploiting blind SQLi via conditional responses | [Exploiting-blind-SQLi-conditional-responses.md](Exploiting-blind-SQLi-conditional-responses.md) |
| 8 | Error-based SQL injection (overview) | [Error-based-SQL-injection.md](Error-based-SQL-injection.md) |
| 9 | Exploiting blind SQLi via conditional errors | [Exploiting-blind-SQLi-conditional-errors.md](Exploiting-blind-SQLi-conditional-errors.md) |
| 10 | Extracting data via verbose SQL error messages | [Extracting-data-via-verbose-SQL-error-messages.md](Extracting-data-via-verbose-SQL-error-messages.md) |
| 11 | Exploiting blind SQLi via time delays | [Exploiting-blind-SQLi-time-delays.md](Exploiting-blind-SQLi-time-delays.md) |
| 12 | Exploiting blind SQLi via out-of-band (OAST) | [Exploiting-blind-SQLi-out-of-band-OAST.md](Exploiting-blind-SQLi-out-of-band-OAST.md) |

---

## 5 · Advanced contexts

Injection points and patterns beyond the query string.

| # | Topic | Notes |
| --- | --- | --- |
| 13 | SQL injection in different contexts (JSON/XML input, WAF bypass) | [SQL-injection-in-different-contexts.md](SQL-injection-in-different-contexts.md) |
| 14 | Second-order (stored) SQL injection | [Second-order-SQL-injection.md](Second-order-SQL-injection.md) |

---

## 6 · Defence & reference

| # | Topic | Notes |
| --- | --- | --- |
| 15 | How to prevent SQL injection | [Preventing-SQL-injection.md](Preventing-SQL-injection.md) |
| 16 | SQL injection cheat sheet (per-database syntax) | [SQL-injection-cheat-sheet.md](SQL-injection-cheat-sheet.md) |

---

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use this list:</strong> it mirrors the PortSwigger learning-path order, so following it in sequence keeps each note building on the last (you need <em>examining the database</em> before <em>UNION attacks</em>, and the blind techniques escalate as each previous one stops working). The <a href="SQL-injection-cheat-sheet.md">cheat sheet</a> is a reference to keep open alongside all of them.
</div>
