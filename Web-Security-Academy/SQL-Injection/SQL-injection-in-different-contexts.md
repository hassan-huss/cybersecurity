# SQL injection in different contexts

SQL injection isn't tied to the query string — **any** input the app feeds into a SQL query is a candidate, including **JSON** and **XML** request bodies. These alternative formats also give you room to **obfuscate** payloads and slip past keyword-based filters (WAFs).

> Part of the [SQL Injection](README.md) path. Builds on [UNION attacks](Retrieving-multiple-values-in-a-single-column.md) — same exploitation, different input channel plus a filter-bypass twist.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Injection lives in the input, not the query string](#injection-lives-in-the-input-not-the-query-string)
- [Bypassing WAF keyword filters with encoding](#bypassing-waf-keyword-filters-with-encoding)
- [XML entity encoding to hide a keyword](#xml-entity-encoding-to-hide-a-keyword)
- [Lab: SQL injection with filter bypass via XML encoding](#lab-sql-injection-with-filter-bypass-via-xml-encoding)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Different contexts** | The injection point can be any field the app puts into a query — JSON, XML, a header, a cookie | Don't only test the URL. If the request body is JSON or XML, inject there too. |
| **Filter bypass by encoding** | A WAF blocks payloads by looking for words like `SELECT` or `UNION` in the raw request | If the app **decodes** your input before running the query (e.g. XML entities), encode the letters of the banned word. The filter sees gibberish; the database sees `SELECT`. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not something to memorise. Remember the two hooks above; come back here for the exact encoding trick.
</div>

---

## Injection lives in the input, not the query string

Every lab so far injected through the **query string** (`?category=...`). But the vulnerable pattern is *input → concatenated into SQL*, and that input can arrive in any format the application parses:

- **JSON** bodies (`{"productId": "123"}`)
- **XML** bodies (a `<stockCheck>` document)
- Cookies, headers, or any other controllable field

The exploitation is identical once you're in the query — a UNION attack is still a UNION attack. What changes is **where you put the payload** and, often, **what defences sit in front of it**.

---

## Bypassing WAF keyword filters with encoding

A **Web Application Firewall (WAF)** or similar filter often blocks injection by scanning the raw request for common SQL keywords — `SELECT`, `UNION`, `FROM`, etc. These are frequently **weak implementations**: they match the literal text in the request and nothing more.

The bypass: if the application **decodes or unescapes** your input *before* building the query, you can encode/escape characters inside the prohibited keyword. The filter inspects the request and sees an encoded blob with no keyword in it; the server decodes it back to a valid keyword before the database ever sees it.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why this works:</strong> the filter and the database look at the input at <em>different stages</em>. The filter checks the request as sent; the database runs it after the app has decoded it. Any transform the app applies in between is a gap you can hide a keyword in.
</div>

---

## XML entity encoding to hide a keyword

XML lets you write any character as a **numeric character reference** — an escape sequence the parser decodes back to the real character. For example `&#x53;` is the hex XML entity for the letter `S`.

So `SELECT` can be written `&#x53;ELECT`. To a keyword filter that's not the word `SELECT`; to the XML parser (and then the database) it is:

```xml
<stockCheck>
    <productId>123</productId>
    <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>
</stockCheck>
```

This is decoded **server-side** before being passed into the SQL query, so the injected `SELECT` runs normally.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>You can encode as much or as little as you need.</strong> If a single keyword is blocked, encoding one letter of it is enough to dodge a naive filter. Against stricter filters, encode more characters (or the whole payload). Burp's <strong>Hackvertor</strong> extension can auto-apply XML-entity encoding to a selected region.
</div>

---

## Lab: SQL injection with filter bypass via XML encoding

> **PRACTITIONER** — SQL injection with filter bypass via XML encoding. **Status: solved.**

**Goal:** the **stock check** feature is SQL-injectable and returns query results in the response, so a **UNION** attack works — but a filter blocks obvious SQL keywords. Retrieve the `administrator` user's credentials from the `users` table and log in.

**Approach:**

1. In Burp, find the **Check stock** request — it sends an **XML** body containing `<productId>` and `<storeId>`.
2. Confirm the `storeId` is injectable (e.g. a maths expression like `1 UNION SELECT NULL` gets blocked → the filter is active).
3. Defeat the filter by **XML-entity-encoding** the SQL keywords in your payload. Select the injected value in Burp Repeater and use the **Hackvertor** extension → **Encode → dec_entities/hex_entities** to encode it automatically. First find the column count, then pull the data:

   ```xml
   <storeId>1 UNION SELECT username || '~' || password FROM users</storeId>
   ```

   …with the keywords/characters entity-encoded, e.g. `&#x55;NION &#x53;ELECT ...`.
4. Send it; the response lists `administrator` and its password (joined with `~`).
5. Log in as `administrator` with the recovered password — lab solved.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Two vulns stacked:</strong> a classic UNION-based SQL injection <em>plus</em> a weak WAF that only pattern-matches raw text. The XML encoding is purely to get the payload past the filter; once decoded, it's an ordinary UNION attack you already know.
</div>
