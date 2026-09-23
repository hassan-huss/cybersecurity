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

Notes are organized by **platform → certification path/topic → module → room/lab**, one Markdown file per room or lab:

```
TryHackMe/
├── Jr-Penetration-Tester/
│   ├── Network-Reconnaissance/
│   ├── nmap/
│   └── Web-Application-Security-Fundamentals/
└── Vulnerabilities/
    ├── SQLi/
    └── XSS.md
Web-Security-Academy/
└── SQL-Injection/
```

| Platform | Path/Topic | What it covers |
| --- | --- | --- |
| **TryHackMe** | Jr. Penetration Tester → Network Reconnaissance | Active vs. passive recon methodology |
| **TryHackMe** | Jr. Penetration Tester → nmap | Host discovery, basic/advanced port scanning, post-scan analysis |
| **TryHackMe** | Jr. Penetration Tester → Web Application Security Fundamentals | Content discovery, fingerprinting modern web stacks (MERN, Next.js, Django, LAMP) and exploiting a real CVE in each, plus attacking web servers directly — Apache/Nginx/Python/Node and the IIS attack chain |
| **TryHackMe** | Vulnerabilities | Standalone, vulnerability-focused labs outside a single path — SQLi (login bypass, UNION, blind, second-order), XSS (reflected, stored, DOM, blind + filter bypass) |
| **PortSwigger** | Web Security Academy → SQL Injection | In progress |

More platforms, paths, and rooms/labs are added as I progress.

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

Recent rooms under **Web Application Security Fundamentals**:

| Room | Focus | Status |
| --- | --- | --- |
| Modern Web Stacks | Fingerprint a stack from HTTP signals, exploit one CVE each (MERN, Next.js, Django, LAMP) + Nikto | ✅ Done |
| Web Server Attacks - I | Attack the server software itself — Apache, Nginx, Python HTTP server, Node/Express | ✅ Done |
| Web Server Attacks - II | The IIS / Windows Server attack chain: fingerprinting → tilde enumeration → WebDAV shell upload → ASPX shells → misconfigurations | ✅ Done |

Standalone labs under **Vulnerabilities**:

| Room | Focus | Status |
| --- | --- | --- |
| SQL Injection Lab | Hands-on SQLi against a Flask/SQLite app: login bypass, `UNION` dumps, boolean-based blind, and two second-order injections + sqlmap tamper scripts | ✅ Done |
| Cross-Site Scripting (XSS) | Reflected, stored, DOM-based and blind XSS; escaping different injection contexts, filter bypasses, cookie exfiltration, and mitigations | ✅ Done |

---

<p align="center"><sub>Personal study repo — not affiliated with TryHackMe or the vendors/frameworks mentioned.</sub></p>
