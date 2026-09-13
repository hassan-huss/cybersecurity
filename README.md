# 🛡️ Cybersecurity Learning Notes

<p align="center">
  <img alt="Track" src="https://img.shields.io/badge/track-TryHackMe%20Jr.%20Penetration%20Tester-red?style=flat-square&logo=hackthebox&logoColor=white">
  <img alt="Type" src="https://img.shields.io/badge/type-personal%20study%20notes-blue?style=flat-square">
  <img alt="Status" src="https://img.shields.io/badge/status-in%20progress-brightgreen?style=flat-square">
</p>

<p align="center"><em>A personal knowledge base for penetration testing — condensed, room-by-room notes as I work through TryHackMe's Jr. Penetration Tester path.</em></p>

---

### Table of contents

- [What this repo is](#-what-this-repo-is)
- [Structure](#-structure)
- [How a note is written](#-how-a-note-is-written)
- [Progress](#-progress)

---

## 📌 What this repo is

This is **not** a copy-paste of TryHackMe's room text. Every file here is a **condensed, self-written summary** — the goal is a lookup sheet I can come back to months later and immediately remember *why* a technique works, not just *that* it exists.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Guiding principle:</strong> understand the mechanism well enough to explain it in plain English, then keep the exact commands/payloads as a reference — never memorize syntax for its own sake.
</div>

## 🗂️ Structure

Notes are organized by **certification path → module → room**, one Markdown file per room:

```
Jr-Penetration-Tester/
├── Network-Reconnaissance/
├── nmap/
└── Web-Application-Security-Fundamentals/
```

| Module | What it covers |
| --- | --- |
| **Network Reconnaissance** | Active vs. passive recon methodology |
| **nmap** | Host discovery, basic/advanced port scanning, post-scan analysis |
| **Web Application Security Fundamentals** | Content discovery, fingerprinting modern web stacks (MERN, Next.js, Django, LAMP) and exploiting a real CVE in each |

More modules and rooms are added as the path progresses.

## ✍️ How a note is written

Every room file follows the same visual format so notes stay easy to scan:

| Element | Purpose |
| --- | --- |
| **Table of contents** | Jump straight to the section needed |
| **Tables** | Fingerprinting signals, comparisons, confidence levels |
| **Code blocks** | Exact, runnable commands/payloads |
| 🟦 Info / 🟧 Warning callouts | Gotchas, caveats, version-specific behavior |
| **"In plain English" recap** | Jargon-free summary + terms explained, for quick review without an SE background |

## 📈 Progress

**Current room:** *Web Application Security Fundamentals → Modern Web Stacks*
Fingerprint a stack from passive HTTP signals, then exploit one real CVE per stack.

| Stack | CVE / bug | Status |
| --- | --- | --- |
| MERN (Express) | Prototype pollution via unfiltered object merge | ✅ Done |
| Next.js | CVE-2025-29927 (middleware auth bypass) | ✅ Done |
| Django | CVE-2021-35042 (`order_by()` SQL injection) | ✅ Done |
| LAMP (Apache 2.4.49) | CVE-2021-41773 (path traversal → `mod_cgi` RCE) | ✅ Done |

---

<p align="center"><sub>Personal study repo — not affiliated with TryHackMe or the vendors/frameworks mentioned.</sub></p>
