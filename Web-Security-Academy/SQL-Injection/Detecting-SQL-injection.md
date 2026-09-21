# Detecting SQL injection

Finding SQLi is a matter of **systematically testing every entry point** with a small set of probes and watching for any change in behaviour — an error, a different response, a delay, or an out-of-band callback.

> Part of the [SQL Injection](README.md) path. Follows [SQL injection fundamentals](SQL-injection-fundamentals.md); the probes here feed directly into the exploitation techniques (UNION, blind, OAST). Source: [PortSwigger Web Security Academy — SQL injection](https://portswigger.net/web-security/sql-injection).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [The manual detection checklist](#the-manual-detection-checklist)
- [Test every entry point](#test-every-entry-point)
- [What each probe tells you](#what-each-probe-tells-you)
- [Automated detection](#automated-detection)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Detection** | Poke each input with SQL-meaningful characters and look for *any* difference | You don't need to exploit it to detect it — a broken quote, a flipped boolean, or a 10-second delay is proof enough. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet. Run the checklist below against every parameter; any anomaly = candidate for the exploitation techniques in this folder.
</div>

---

## The manual detection checklist

At each entry point, submit these in turn and compare responses to a clean baseline:

1. **Single quote `'`** — submit it and look for **errors** or other anomalies (a broken quote often breaks the query's syntax).
2. **SQL-specific syntax** — submit a value that evaluates to the **base (original) value** and one that evaluates to a **different** value, and look for systematic differences (e.g. an arithmetic expression that resolves to the original ID).
3. **Boolean conditions** — inject `OR 1=1` and `OR 1=2`, and watch for differences between the true and false cases.
4. **Time-delay payloads** — inject payloads that trigger a **time delay** inside the query, and detect differences in how long the response takes.
5. **OAST payloads** — inject payloads designed to trigger an **out-of-band network interaction** (e.g. a DNS lookup to Burp Collaborator), and monitor for any resulting interactions.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>The point is <em>difference</em>, not a specific output.</strong> The app might show the DB error, a generic error, or nothing at all — as long as the true vs false (or delay vs no-delay) cases behave differently, the input is injectable.
</div>

---

## Test every entry point

A large proportion of SQLi is missed because only the obvious inputs get tested. Probe **all** of these:

- **URL query-string** parameters
- **POST body** / form fields
- **HTTP headers** — notably the **`Cookie`** header and `User-Agent`/`Referer` where apps log or process them
- Any other value the application processes — JSON/XML fields, etc. (see [different contexts](SQL-injection-in-different-contexts.md))

Test each parameter **separately**, keeping the others at their normal values, so you can attribute any change to the exact input you're probing.

---

## What each probe tells you

| Probe | Signal | Which technique it opens up |
| --- | --- | --- |
| `'` | Error / broken response | Confirms input reaches the query; may reveal [verbose errors](Extracting-data-via-verbose-SQL-error-messages.md) |
| `OR 1=1` vs `OR 1=2` | Different response for true vs false | [Conditional responses](Exploiting-blind-SQLi-conditional-responses.md) |
| `CASE ... 1/0` style | Response differs only when an error fires | [Conditional errors](Exploiting-blind-SQLi-conditional-errors.md) |
| Time-delay payload | Response is slow only when condition true | [Time delays](Exploiting-blind-SQLi-time-delays.md) |
| OAST payload | DNS/HTTP callback received | [Out-of-band (OAST)](Exploiting-blind-SQLi-out-of-band-OAST.md) |

The escalation ladder: try visible results (errors/UNION) first; if the response gives nothing away, fall back through conditional responses → conditional errors → time delays → OAST, in that order.

---

## Automated detection

The vast majority of SQLi can be found quickly and reliably with **Burp Scanner** (Burp Suite's web vulnerability scanner), which fires these probe classes automatically and flags anomalies. Manual testing remains essential for confirming, understanding, and exploiting what the scanner surfaces — and for inputs a scanner might not reach.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Payload references:</strong> exact per-database syntax for the boolean, error, time-delay, and OAST probes is in the <a href="SQL-injection-cheat-sheet.md">SQL injection cheat sheet</a>.
</div>
