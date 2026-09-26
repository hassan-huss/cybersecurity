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
│   ├── Web-Application-Security-Fundamentals/
│   ├── Web-Application-Vulnerabilities-II/
│   └── Vulnerability-Knowledge/
└── Vulnerabilities/
    ├── IDOR/
    ├── SQLi/
    ├── SSRF/
    └── XSS/
Web-Security-Academy/
└── SQL-Injection/
```

| Platform | Path/Topic | What it covers |
| --- | --- | --- |
| **TryHackMe** | Jr. Penetration Tester → Network Reconnaissance | Active vs. passive recon methodology |
| **TryHackMe** | Jr. Penetration Tester → nmap | Host discovery, basic/advanced port scanning, post-scan analysis |
| **TryHackMe** | Jr. Penetration Tester → Web Application Security Fundamentals | Content discovery, fingerprinting modern web stacks (MERN, Next.js, Django, LAMP) and exploiting a real CVE in each, plus attacking web servers directly — Apache/Nginx/Python/Node and the IIS attack chain |
| **TryHackMe** | Jr. Penetration Tester → Web Application Vulnerabilities II | Session management (lifecycle, IAAA, cookies vs tokens), broken authentication (username enumeration, brute force, password-reset logic flaws, cookie tampering), and file inclusion (path traversal, LFI, RFI) |
| **TryHackMe** | Jr. Penetration Tester → Vulnerability Knowledge | Researching vulnerabilities in public databases (CVE, NVD, Exploit-DB), scanning at scale, and manually validating whether a CVE actually applies to a target |
| **TryHackMe** | Vulnerabilities | Standalone, vulnerability-focused labs outside a single path — SQLi (login bypass, UNION, blind, second-order), XSS (reflected, stored, DOM, blind + filter bypass), SSRF (vectors, defence bypasses, cloud metadata), IDOR (broken access control / BOLA) |
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

Rooms under **Web Application Vulnerabilities II**:

| Room | Focus | Status |
| --- | --- | --- |
| Session Management | Session lifecycle (creation → tracking → expiry → termination), IAAA, cookies vs tokens, fixation/authorisation-bypass/expiry/logout flaws, and mapping a live app's lifecycle in DevTools | ✅ Done |
| Broken Authentication | Username enumeration + brute force with ffuf, a password-reset parameter-pollution logic flaw, and plain/hashed/base64 cookie manipulation — plus mitigations | ✅ Done |
| File Inclusion | Path traversal (reading files outside the web root), LFI → RCE, filter bypasses (null byte, `....//`, forced-prefix), RFI via `allow_url_fopen`, and testing every input channel (URL/cookie/POST) | ✅ Done |
| Command Injection | Injecting OS commands via shell operators (`;` `&&` `\|`), verbose vs blind detection (time delay / write-to-file), Linux & Windows payloads, and remediation (server-side allowlisting, least privilege, why filters get bypassed) | ✅ Done |
| API Pentesting | RESTful fundamentals (methods, 401 vs 403, JWTs), then the OWASP API Top 10 headliners — BOLA/IDOR, broken authentication, excessive data exposure, mass assignment, rate limiting — and chaining them from customer to admin | ✅ Done |
| Support (challenge) | ⬜ Not started |

Rooms under **Vulnerability Knowledge**:

| Room | Focus | Status |
| --- | --- | --- |
| Understanding Vulnerability Databases | What vuln databases are and why they exist; the CVE/CVSS/CPE/CWE/CNA building blocks; severity vs risk; and the division of labour between the CVE List (MITRE), NVD (NIST), and Exploit-DB | ✅ Done |
| Vulnerability Scanning Tools | Nmap (host discovery → `-sV` → `-A` → NSE scripts like `ftp-anon`), Nikto web-server checks (version leak, missing headers, `phpinfo`, directory indexing, RFI lead), OpenVAS/Greenbone targets → tasks → reports, and scanning best practices | ✅ Done |
| Basic Vulnerability Identification Techniques | ⬜ Not started |
| NoScope: Finding RCE (CVE-2026-35482) | ⬜ Not started |
| n8n: CVE-2025-68613 | ⬜ Not started |
| AD: BadSuccessor | ⬜ Not started |
| Fragnesia (CVE-2026-46300) | ⬜ Not started |
| Nginx Rift (CVE-2026-42945) | ⬜ Not started |

Standalone labs under **Vulnerabilities**:

| Room | Focus | Status |
| --- | --- | --- |
| SQL Injection Lab | Hands-on SQLi against a Flask/SQLite app: login bypass, `UNION` dumps, boolean-based blind, and two second-order injections + sqlmap tamper scripts | ✅ Done |
| Cross-Site Scripting (XSS) | Reflected, stored, DOM-based and blind XSS; escaping different injection contexts, filter bypasses, cookie exfiltration, and mitigations | ✅ Done |
| Server-Side Request Forgery (SSRF) | The four input vectors, identifying/confirming blind SSRF, deny-list/allow-list/open-redirect bypasses, cloud metadata theft, and the Acme avatar traversal lab | ✅ Done |
| Insecure Direct Object Reference (IDOR) | Broken access control / BOLA; plaintext, encoded, hashed and unpredictable object references, the two-account technique, where vectors hide, and the Acme customer-API lab | ✅ Done |

---

<p align="center"><sub>Personal study repo — not affiliated with TryHackMe or the vendors/frameworks mentioned.</sub></p>
