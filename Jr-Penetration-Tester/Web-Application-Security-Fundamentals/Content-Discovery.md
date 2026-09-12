# Content Discovery

**Content discovery** — locating the pages, directories, files, and subdomains of a web application that are not linked from the homepage, so they can be fed into later stages of a penetration test.

---

**Table of contents**

- [Overview](#overview)
- [Manual discovery](#manual-discovery)
- [OSINT](#osint)
- [Automated discovery](#automated-discovery)
- [Method recap](#method-recap)
- [Workflow](#workflow)

---

## Overview

Content discovery is one of the most important phases of web application reconnaissance. The techniques in this room work together:

- **Manual checks** surface quick wins.
- **OSINT** finds information the target has already shared publicly.
- **Automated tools** cover the breadth that neither approach can do alone.

No single method is sufficient on its own — each one sees a different slice of the application.

---

## Manual discovery

Reading what the application volunteers about itself.

| Technique | What it reveals |
| --- | --- |
| `robots.txt` | Paths the owner asked crawlers to avoid — often the interesting ones. |
| `sitemap.xml` | A listing of pages the owner wants indexed, including ones not linked in the UI. |
| Favicon fingerprinting | Matching the favicon hash against known frameworks to identify the stack. |
| HTTP headers | Server, language, and framework details leaked in the response. |
| Framework stack | Default paths, admin panels, and docs that ship with the identified framework. |

---

## OSINT

Information the target has already published somewhere else.

| Source | What it provides |
| --- | --- |
| Google dorking | Search operators that surface indexed pages, file types, and subdomains. |
| Wappalyzer | Passive fingerprinting of CMS, web server, JS frameworks, and CDNs. |
| Wayback Machine | Historical snapshots — endpoints that existed before and may still respond. |
| GitHub | Repositories, commits, and configs that expose paths, keys, or internal structure. |
| S3 buckets | Publicly readable cloud storage tied to the target. |

---

## Automated discovery

Brute-forcing paths from a wordlist to cover ground manual checks cannot.

| Gobuster mode | Purpose |
| --- | --- |
| `dir` | Discover directories and files on a web server. |
| `dns` | Discover subdomains of a domain. |
| `vhost` | Discover virtual hosts served from the same IP. |

```bash
gobuster dir  -u http://MACHINE_IP -w /usr/share/wordlists/dirb/common.txt
gobuster dns  -d example.com       -w /usr/share/wordlists/subdomains.txt
gobuster vhost -u http://MACHINE_IP -w /usr/share/wordlists/subdomains.txt
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> Automated discovery is loud. It generates large volumes of requests and 404s that are trivial to spot in logs and may trip rate limiting or a WAF. Only run it against authorized targets.
</div>

---

## Method recap

| Method | Techniques |
| --- | --- |
| **Manual** | `robots.txt`, `sitemap.xml`, favicon fingerprinting, HTTP headers, framework stack |
| **OSINT** | Google dorking, Wappalyzer, Wayback Machine, GitHub, S3 buckets |
| **Automated** | Gobuster `dir`, `dns`, and `vhost` modes |

---

## Workflow

A good content discovery workflow runs **all three methods** against a target before moving to exploitation.

1. **Manual** — check what the app tells you for free.
2. **OSINT** — check what the target has already published elsewhere.
3. **Automated** — brute-force the rest of the namespace.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> The directories and files you find here feed directly into later stages of a penetration test. Record every hit — an endpoint that looks uninteresting now often becomes the entry point once you know more about the stack.
</div>
