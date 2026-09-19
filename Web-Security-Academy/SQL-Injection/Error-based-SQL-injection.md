# Error-based SQL injection

**Error-based** SQL injection is any case where you use database **error messages** to extract or infer data — including in blind contexts where the query results themselves are never shown.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What error-based SQLi covers](#what-error-based-sqli-covers)
- [Two flavours](#two-flavours)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Error-based SQLi** | Let the database's error messages do the talking | When the page won't show query **results**, it might still show **errors** — and an error you deliberately trigger can act as a yes/no signal, or even carry the data itself. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> these are a lookup sheet, not something to memorise. Remember the one-line hook above; the two techniques are detailed in the linked pages.
</div>

---

## What error-based SQLi covers

To exploit it, you use error messages to either **extract** sensitive data directly or **infer** it one condition at a time. What's possible depends on:

- the **configuration** of the database (how verbose its errors are, whether they reach you), and
- the **types of errors** you can trigger through your injection.

---

## Two flavours

| Flavour | How it works | Detailed in |
| --- | --- | --- |
| **Conditional errors** | Force an error **only when an injected condition is true**, then watch whether the response changes. Exploited like [conditional responses](Exploiting-blind-SQLi-conditional-responses.md) — but the signal is an error, not a message. | [Exploiting blind SQLi via conditional errors](Exploiting-blind-SQLi-conditional-errors.md) |
| **Verbose error messages** | Trigger an error whose text **contains the query's data**, turning a blind vulnerability into a visible one. | [Extracting data via verbose SQL error messages](Extracting-data-via-verbose-SQL-error-messages.md) |
