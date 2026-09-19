# Blind SQL injection

**Blind** SQL injection is SQL injection where the application is genuinely vulnerable, but its **responses give nothing back** — no query results, no database errors — so you can't just read the data off the page.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What is blind SQL injection?](#what-is-blind-sql-injection)
- [Why UNION attacks don't work here](#why-union-attacks-dont-work-here)
- [The techniques that do work](#the-techniques-that-do-work)

---

## In plain English

| Topic | The idea | Takeaway |
| --- | --- | --- |
| **Blind SQLi** | The injection works, but the page never shows the query's output or errors | You can't **read** the answer, so you have to **infer** it — make the database behave differently for true vs. false and watch that behaviour, one bit at a time. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> these are a lookup sheet, not something to memorise. Remember the one-line hook above; the detailed techniques live in the linked pages.
</div>

---

## What is blind SQL injection?

Blind SQL injection happens when an application **is** vulnerable to SQL injection, but its HTTP responses **do not contain**:

- the results of the injected query, **nor**
- the details of any database errors.

The vulnerability is real and still exploitable — you just can't see the data directly. You have to extract it indirectly.

---

## Why UNION attacks don't work here

Techniques like **UNION attacks** rely on seeing the injected query's results **inside the response**. In a blind scenario there's nothing reflected, so those techniques come up empty. You can still reach unauthorised data, but you need **different** methods.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>"Blind" doesn't mean safe:</strong> the exact same data (passwords included) is still reachable — it just comes out one true/false answer at a time instead of all at once. It's slower, not weaker.
</div>

---

## The techniques that do work

Blind SQL injection is exploited by making the database's **observable behaviour** depend on an injected condition, then testing conditions one by one:

| Technique | What you observe | Covered in |
| --- | --- | --- |
| **Conditional responses** | A visible page difference (e.g. a "Welcome back" message appears or not) | [Exploiting blind SQLi via conditional responses](Exploiting-blind-SQLi-conditional-responses.md) |
| Conditional errors | Whether the query throws a database error or not | *(later topic)* |
| Time delays | How long the response takes when you inject a conditional `SLEEP`/`WAITFOR` | *(later topic)* |
| Out-of-band (OAST) | A DNS/HTTP callback triggered by the query | *(later topic)* |

The common thread: turn the invisible query result into **something you can measure** — a page change, an error, a delay, or a network callback.
