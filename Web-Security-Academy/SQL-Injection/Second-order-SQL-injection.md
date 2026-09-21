# Second-order SQL injection

**Second-order (a.k.a. stored) SQL injection**: the app safely stores your input, then later reads it back out and drops it **unsafely** into a SQL query on a *different* request — so the payload fires long after it was submitted.

> Part of the [SQL Injection](README.md) path. Contrast with **first-order** injection, where the payload executes on the same request that submits it (every other note in this folder).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [First-order vs second-order](#first-order-vs-second-order)
- [Why developers miss it](#why-developers-miss-it)
- [How to find it](#how-to-find-it)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Second-order SQLi** | Your payload is stored now, but doesn't run until some later action reads it back into a query | The dangerous input and the vulnerable query are in **two different requests**. Storing is safe; the bug is in the *re-use*, where the developer wrongly trusts the stored value. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet. The one thing to hold onto: <em>stored ≠ safe</em>. Data that was inserted safely can still be injected when it's later pulled out and concatenated.
</div>

---

## First-order vs second-order

| | First-order | Second-order (stored) |
| --- | --- | --- |
| **When it runs** | Immediately, on the request that submits the input | Later, on a **different** request that reads the stored value |
| **Where the flaw is** | The query that directly handles the HTTP input | The query that re-uses the previously stored data |
| **Example** | `?category=Gifts'--` returns extra rows in the same response | You register a username `admin'--`; the bug fires later when *another* feature (e.g. a password change) builds a query from your stored username |

**First-order:** the application takes input from an HTTP request and unsafely concatenates it into a query *then and there*.

**Second-order:** the application takes input from an HTTP request and **stores it** (usually in the database) — safely, with no vulnerability at the point of storage. Later, handling a *different* request, it **retrieves the stored data** and incorporates it into a SQL query in an unsafe way.

---

## Why developers miss it

Second-order injection tends to appear precisely where developers **are** aware of SQL injection — so they carefully parameterise the *initial* insert. The trap is what happens next: when the data is later read back, it's deemed **already safe** *because it came from the database*, so it's handled with plain string concatenation.

The value is treated as trusted simply because of where it now lives — but its real origin was still user input. The safe storage lulls the developer into an unsafe re-use.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Trust follows origin, not location.</strong> Data doesn't become clean by sitting in a database. If it originally came from a user, it must be parameterised <em>every</em> time it's used in a query — including reads of stored values.
</div>

---

## How to find it

Because the injection and its effect are decoupled, testing is harder than first-order:

- **Submit payloads into stored fields** (usernames, profile fields, comments, anything persisted) — even if the immediate response looks clean.
- **Then exercise other features** that might re-use that stored value in a query, and watch for errors, timing, or behaviour changes there.
- Watch for **delayed / out-of-context** errors: an error triggered by an action unrelated to where you submitted the input is a strong tell.
- Combine with **blind techniques** ([conditional responses](Exploiting-blind-SQLi-conditional-responses.md), [time delays](Exploiting-blind-SQLi-time-delays.md), [OAST](Exploiting-blind-SQLi-out-of-band-OAST.md)) once you've located a re-use point, since the feedback often isn't in the response that stored the payload.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Mindset shift:</strong> stop thinking "does <em>this</em> request reflect my payload?" and start thinking "where might this value be <em>read back</em> later?" Map input → storage → every downstream feature that touches it.
</div>
