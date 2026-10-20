<h1 align="center">🛡️ Cybersecurity Learning Notes</h1>

<p align="center">
  <em>A personal, battle-tested knowledge base for offensive security —<br>
  condensed, room-by-room notes written to be understood, not memorized.</em>
</p>

<p align="center">
  <img alt="Notes" src="https://img.shields.io/badge/notes-75%2B%20rooms%20%26%20labs-2b8cf0?style=for-the-badge&logo=bookstack&logoColor=white">
  <img alt="Focus" src="https://img.shields.io/badge/focus-web%20app%20pentesting-8b5cf6?style=for-the-badge&logo=hackaday&logoColor=white">
  <img alt="Status" src="https://img.shields.io/badge/status-actively%20growing-22c55e?style=for-the-badge">
</p>

<p align="center">
  <img alt="TryHackMe" src="https://img.shields.io/badge/TryHackMe-Jr.%20Pen%20Tester%20%E2%9C%93-c11111?style=flat-square&logo=tryhackme&logoColor=white">
  <img alt="Web App Pentesting" src="https://img.shields.io/badge/TryHackMe-Web%20App%20Pentesting-c11111?style=flat-square&logo=tryhackme&logoColor=white">
  <img alt="PortSwigger" src="https://img.shields.io/badge/PortSwigger-Web%20Security%20Academy-ff6633?style=flat-square&logo=portswigger&logoColor=white">
</p>

<p align="center">
  <a href="#-what-this-repo-is">What this is</a> ·
  <a href="#-structure">Structure</a> ·
  <a href="#-how-a-note-is-written">Note format</a> ·
  <a href="#-progress">Progress</a>
</p>

---

## 📌 What this repo is

This is **not** a copy-paste of room text. Every file is a **condensed, self-written summary** — the goal is a lookup sheet I can come back to months later and instantly remember *why* a technique works, not just *that* it exists.

> [!NOTE]
> **Guiding principle:** understand the mechanism well enough to explain it in plain English, then keep the exact commands and payloads as a reference — never memorize syntax for its own sake.

Every note covers the same ground: **what** the vulnerability is → **why** it happens (root cause) → **how** it's exploited → **how** to fix it, closed out with a jargon-free *"in plain English"* recap.

---

## 🗂️ Structure

Notes are organized by **platform → path/topic → module → room/lab**, one Markdown file per room or lab:

```
TryHackMe/
├── Jr-Penetration-Tester/
│   ├── Network-Reconnaissance/
│   ├── nmap/
│   ├── Web-Application-Security-Fundamentals/
│   ├── Web-Application-Vulnerabilities-II/
│   ├── Vulnerability-Knowledge/
│   ├── Password-Attacks/
│   ├── Metasploit-and-Exploitation/
│   ├── Specialized-Domains/
│   ├── Python-Scripting-Basics/
│   └── Privilege-Escalation/
├── Web-Application-Pentesting/
│   ├── Authentication/
│   ├── Injection-Attacks/
│   ├── Advanced-Server-Side-Attacks/
│   ├── Advanced-Client-Side-Attacks/
│   └── HTTP-Request-Smuggling/
└── Vulnerabilities/
    ├── IDOR/
    ├── SQLi/
    ├── SSRF/
    └── XSS/
Web-Security-Academy/
└── SQL-Injection/
```

<details>
<summary><strong>📚 Full platform &amp; path coverage</strong> — what each area contains</summary>

<br>

| Platform | Path / Topic | What it covers |
| --- | --- | --- |
| **TryHackMe** | Jr. Penetration Tester → Network Reconnaissance | Active vs. passive recon methodology |
| **TryHackMe** | Jr. Penetration Tester → nmap | Host discovery, basic/advanced port scanning, post-scan analysis |
| **TryHackMe** | Jr. Penetration Tester → Web Application Security Fundamentals | Content discovery, fingerprinting modern web stacks (MERN, Next.js, Django, LAMP) and exploiting a real CVE in each, plus attacking web servers directly — Apache/Nginx/Python/Node and the IIS attack chain |
| **TryHackMe** | Jr. Penetration Tester → Web Application Vulnerabilities II | Session management (lifecycle, IAAA, cookies vs tokens), broken authentication (username enumeration, brute force, password-reset logic flaws, cookie tampering), file inclusion (path traversal, LFI, RFI), and API pentesting (BOLA, mass assignment, excessive data exposure) |
| **TryHackMe** | Jr. Penetration Tester → Vulnerability Knowledge | Researching vulnerabilities in public databases (CVE, NVD, Exploit-DB), scanning at scale, and manually validating whether a CVE actually applies to a target |
| **TryHackMe** | Jr. Penetration Tester → Password Attacks | Phishing, online login brute-forcing (Hydra), building targeted wordlists from OSINT, and offline password/hash cracking — chained in a final challenge |
| **TryHackMe** | Jr. Penetration Tester → Metasploit and Exploitation | The Metasploit Framework end to end — navigating `msfconsole`, scanning + exploiting live targets, Meterpreter post-exploitation, and generating custom payloads with `msfvenom` — plus the manual craft behind it (shells, listeners, payload delivery) |
| **TryHackMe** | Jr. Penetration Tester → Specialized Domains | Branching beyond core pentesting into specialized fields — mobile application security (static analysis, MobSF, OWASP Mobile Top 10), cloud security, and LLM pentesting |
| **TryHackMe** | Jr. Penetration Tester → Python Scripting Basics | Learning Python from zero through to real pentest tooling — core concepts, building scripts, and six security scripts (recon, ARP discovery, port scanning, hash cracking, SSH brute force) |
| **TryHackMe** | Jr. Penetration Tester → Privilege Escalation | Turning a foothold into full control on Linux and Windows — config-review baselines (CIS/STIG), enumeration, and two live jump challenges |
| **TryHackMe** | Web Application Pentesting → Authentication | A path focused specifically on web app pentesting, started after finishing Jr. Penetration Tester. Authentication module: enumerating auth mechanisms and brute force, OAuth 2.0 flow + attacks (redirect hijacking, CSRF, implicit-grant token theft), JWT security, and MFA/2FA implementation flaws (OTP leakage, logic bypass, Evilginx) |
| **TryHackMe** | Web Application Pentesting → Injection Attacks | Advanced SQL Injection (second-order, filter evasion, OOB exfiltration, header injection), NoSQL Injection (MongoDB), XXE Injection, Server-Side Template Injection, LDAP Injection, and ORM Injection — capped by the Injectics challenge |
| **TryHackMe** | Web Application Pentesting → Advanced Server-Side Attacks | Insecure deserialisation (cookie tampering, PHP object injection, gadget chains); SSRF (local-server file disclosure, internal-server pivoting, blind OOB/time-based, SSRF→DoS); advanced file inclusion/path traversal (PHP wrappers, base-directory breakouts, three LFI→RCE escalation paths); race conditions (TOCTOU, Burp Repeater parallel request groups); and prototype pollution (property-by-path XSS, merge/clone property injection, DoS) — capped by the Include challenge |
| **TryHackMe** | Web Application Pentesting → Advanced Client-Side Attacks | Browser-side attacks: XSS (reflected, stored, DOM-based and their root causes), CSRF (how it works, exploitation, defences), DOM-based vectors, and the Same-Origin Policy (SOP) / Cross-Origin Resource Sharing (CORS) — capped by the Whats Your Name? challenge |
| **TryHackMe** | Web Application Pentesting → HTTP Request Smuggling | Front-end/back-end parsing disagreements over request boundaries (CL.TE/TE.CL, Transfer-Encoding obfuscation), extended to HTTP/2, WebSocket upgrades, and browser-level desync — capped by the El Bandito challenge |
| **TryHackMe** | Vulnerabilities | Standalone, vulnerability-focused labs outside a single path — SQLi (login bypass, UNION, blind, second-order, plus automating exploitation with sqlmap), XSS (reflected, stored, DOM, blind + filter bypass), SSRF (vectors, defence bypasses, cloud metadata, plus a second advanced SSRF lab on local/internal-server pivoting and SSRF→DoS), IDOR (broken access control / BOLA) |
| **PortSwigger** | Web Security Academy → SQL Injection | In progress |

</details>

> More platforms, paths, and rooms/labs are added as I progress.

---

## ✍️ How a note is written

Every room file follows the same visual format so notes stay easy to scan:

| Element | Purpose |
| --- | --- |
| 🧭 **Table of contents** | Jump straight to the section needed |
| 📊 **Tables** | Fingerprinting signals, comparisons, confidence levels |
| 💻 **Code blocks** | Exact, runnable commands / payloads |
| 🟦 **Info / 🟧 Warning callouts** | Gotchas, caveats, version-specific behavior |
| 💬 **"In plain English" recap** | Jargon-free summary + terms explained, for quick review without an SE background |

> [!TIP]
> Weaponization-heavy rooms are written as **mechanism + root cause + fix** with placeholders (e.g. `ATTACKER_IP`) instead of turnkey, copy-paste attack payloads. These notes are for learning *why* an attack works and how to defend against it — not a drop-in toolkit.

---

## 📈 Progress

<p align="center">
  <img alt="Done" src="https://img.shields.io/badge/completed-50%2B%20rooms-22c55e?style=flat-square">
  <img alt="In progress" src="https://img.shields.io/badge/in%20progress-1-f59e0b?style=flat-square">
  <img alt="Queued" src="https://img.shields.io/badge/queued-28-64748b?style=flat-square">
</p>

<details open>
<summary><strong>🔴 TryHackMe · Jr. Penetration Tester</strong> — core offensive-security path <em>(completed)</em></summary>

<br>

**Web Application Security Fundamentals**

| Room | Focus | Status |
| --- | --- | --- |
| Modern Web Stacks | Fingerprint a stack from HTTP signals, exploit one CVE each (MERN, Next.js, Django, LAMP) + Nikto | ✅ Done |
| Web Server Attacks - I | Attack the server software itself — Apache, Nginx, Python HTTP server, Node/Express | ✅ Done |
| Web Server Attacks - II | The IIS / Windows Server attack chain: fingerprinting → tilde enumeration → WebDAV shell upload → ASPX shells → misconfigurations | ✅ Done |

**Web Application Vulnerabilities II**

| Room | Focus | Status |
| --- | --- | --- |
| Session Management | Session lifecycle (creation → tracking → expiry → termination), IAAA, cookies vs tokens, fixation/authorisation-bypass/expiry/logout flaws, and mapping a live app's lifecycle in DevTools | ✅ Done |
| Broken Authentication | Username enumeration + brute force with ffuf, a password-reset parameter-pollution logic flaw, and plain/hashed/base64 cookie manipulation — plus mitigations | ✅ Done |
| File Inclusion | Path traversal (reading files outside the web root), LFI → RCE, filter bypasses (null byte, `....//`, forced-prefix), RFI via `allow_url_fopen`, and testing every input channel (URL/cookie/POST) | ✅ Done |
| Command Injection | Injecting OS commands via shell operators (`;` `&&` `\|`), verbose vs blind detection (time delay / write-to-file), Linux & Windows payloads, and remediation (server-side allowlisting, least privilege, why filters get bypassed) | ✅ Done |
| API Pentesting | RESTful fundamentals (methods, 401 vs 403, JWTs), then the OWASP API Top 10 headliners — BOLA/IDOR, broken authentication, excessive data exposure, mass assignment, rate limiting — and chaining them from customer to admin | ✅ Done |
| Support (challenge) | — | ⬜ Not started |

**Vulnerability Knowledge**

| Room | Focus | Status |
| --- | --- | --- |
| Understanding Vulnerability Databases | What vuln databases are and why they exist; the CVE/CVSS/CPE/CWE/CNA building blocks; severity vs risk; and the division of labour between the CVE List (MITRE), NVD (NIST), and Exploit-DB | ✅ Done |
| Vulnerability Scanning Tools | Nmap (host discovery → `-sV` → `-A` → NSE scripts like `ftp-anon`), Nikto web-server checks (version leak, missing headers, `phpinfo`, directory indexing, RFI lead), OpenVAS/Greenbone targets → tasks → reports, and scanning best practices | ✅ Done |
| Basic Vulnerability Identification Techniques | — | ⬜ Not started |
| NoScope: Finding RCE (CVE-2026-35482) | Alf.io JavaScript sandbox escape: the injected `returnClass` object exposes `Class.forName()`, so a class name passed as a string bypasses the source-text blocklist, reaching Java reflection → `Runtime.exec()` → RCE; plus NoScope/continuous-pentesting and blocklist-vs-allowlist mitigations | ✅ Done |
| n8n: CVE-2025-68613 | Node.js expression-injection RCE: n8n `{{ }}` expressions run as unsandboxed JavaScript, so `this.process.mainModule.require('child_process')` reaches command execution with the app's privileges; plus root cause (missing context isolation), detection (proxy body logging + Sigma on `/rest/workflows`), and mitigations | ✅ Done |
| AD: BadSuccessor | Active Directory privesc abusing delegated Managed Service Accounts (dMSA): OU write access lets you create a dMSA and fake a completed migration (`msDS-ManagedAccountPrecededByLink` / `msDS-DelegatedMSAState`) linking it to Administrator, then Kerberos ticket abuse (SharpSuccessor → Rubeus `tgtdeleg`/`asktgs /dmsa` → pass-the-ticket, or bloodyAD/Impacket on Linux) inherits domain admin; plus least-privilege + attribute-lockdown mitigations | ✅ Done |
| Fragnesia (CVE-2026-46300) | — | ⬜ Not started |
| Nginx Rift (CVE-2026-42945) | — | ⬜ Not started |

**Password Attacks**

| Room | Focus | Status |
| --- | --- | --- |
| Phishing Basics | — | ⬜ Not started |
| Hydra | — | ⬜ Not started |
| Introduction to Wordlists | Building *targeted* wordlists from OSINT (CeWL site-scrape, `strings` on PDFs, `awk` name→username formats, `crunch` pattern passwords), cleaning them (dedupe/lowercase/filter), then using them with `ffuf` for directory discovery and Hydra to brute-force a login form | ✅ Done |
| Password Cracking | — | ⬜ Not started |
| Checkmate (challenge) | — | ⬜ Not started |

**Metasploit and Exploitation**

| Room | Focus | Status |
| --- | --- | --- |
| Metasploit: The Basics | What the framework is (Pro vs Framework, the three pillars), the vulnerability → exploit → payload chain, the seven module categories, staged vs single payloads, and navigating `msfconsole` — `search`/filters, exploit rankings, and `info` | 🚧 In progress |
| Metasploit: Scanning and Exploitation | The scan → store → identify → exploit cycle: `portscan`/`db_nmap` + service scanners, the Metasploit database (workspaces, `hosts`/`services`/`creds`, `-R` auto-fill), version-string → CVE vuln scanning, then two contrasting exploits (EternalBlue → SYSTEM Meterpreter; vsftpd 2.3.4 backdoor → root command shell) proving the workflow is OS-agnostic | ✅ Done |
| Metasploit: Post-Exploitation | — | ⬜ Not started |
| Metasploit: Payload Generation | Generating standalone payloads with `msfvenom` — the flag reference, staged vs stageless, executable vs transform output formats, a per-scenario recipe table, why encoding is bad-character removal (not AV evasion), template injection (`-x`/`-k`) and its detection trade-offs, and catching shells with `exploit/multi/handler` (the payload/LHOST/LPORT "must match exactly" rule) | ✅ Done |
| Exploitation and Weaponisation | The consultant's exploitation → weaponisation workflow: analysing a finding (affected functionality, confirm exploitability with auxiliary scanners / module `check`, reproduce twice, decide whether to exploit), controlled *minimum-proof* exploitation (`getuid`→SYSTEM, `SYSTEM_USER`=`sa`), context-aware weaponisation (trust boundaries, access ≠ business impact, abusing legitimate high-value actions), chaining flaws into a realistic attack path (FTP→SSH lateral movement; stored XSS→admin session), and interpreting tool output honestly | ✅ Done |
| Shells & Listeners Fundamentals | Manual shell craft: reverse vs bind shells (who initiates + which firewall each beats), the netcat→rlwrap→socat→msfvenom/`multi-handler` tool ladder, netcat listener flags + `-e` caveat, socat address specs (Linux/Windows, `EXEC:"bash -li"`), shell stabilisation (Python `pty.spawn` + `stty raw -echo` + terminal sizing, `rlwrap`, full socat TTY bundle `pty,stderr,sigint,setsid,sane`), and TLS-encrypted shells (self-signed cert + socat `OPENSSL` to blend into HTTPS) | ✅ Done |
| Shell Payload Generation & Delivery | Generating and *delivering* payloads: manual reverse/bind shells (netcat FIFO, Python `dup2`, bash `/dev/tcp`, PowerShell) and the payload-selection checklist, `msfvenom` (syntax, staged vs stageless naming, Meterpreter, output formats, weak-evasion encoding), the `multi/handler` listener (the `PAYLOAD`/`LHOST`/`LPORT`-must-match rule, jobs vs sessions), and webshells (the read-param → OS → output pattern, upgrading a webshell into a full reverse shell, detection + cleanup hygiene) | ✅ Done |

**Specialized Domains** *(branching beyond the core path)*

| Room | Focus | Status |
| --- | --- | --- |
| Mobile Application Security | How mobile apps are packaged (manifest, sandbox, components), the four-phase methodology, static analysis (reading the manifest, hunting hardcoded secrets, MobSF), dynamic analysis (traffic interception, SSL pinning, Frida/Objection, insecure logging), the common vulnerability categories mapped to the OWASP Mobile Top 10, and a MobSF static-analysis CTF (Leaky Package — APK + IPA) | ✅ Done |
| Cloud Security Fundamentals | — | ⬜ Not started |
| LLM Pentesting | — | ⬜ Not started |

**Python Scripting Basics** *(learning Python from zero, ending in real pentest scripts)*

| Room | Focus | Status |
| --- | --- | --- |
| Python: Simple Demo | A guided first look at what a basic Python program looks like | ✅ Done |
| Python: Core Concepts | Type conversion and f-strings, string methods (indexing, slicing, character checks), lists and dictionaries, the arithmetic and membership operators (`**`, `//`, `%`, `in`), and `for`/`while` loops with `range()`, `break` and `continue` | ✅ Done |
| Python: Building Scripts | Functions (parameters, return values, defaults, scope), `try`/`except` error handling, file I/O with `with` (`r`/`w`/`a` modes, wordlists), libraries and `pip`, capped by a Password Strength Checker that combines everything from both rooms | ✅ Done |
| Python: Pentesting Scripts | Six security tools — web recon (subdomain/directory enumeration), ARP network discovery (Scapy), TCP port scanning (sockets), automated downloads (streaming), hash cracking (hashlib dictionary attack), and SSH brute-forcing (Paramiko) — plus a menu-driven mini-toolkit tying three of them together | ✅ Done |

**Privilege Escalation** *(turning a foothold into full control, Linux + Windows)*

| Room | Focus | Status |
| --- | --- | --- |
| Host-Server Configuration Reviews | Vulnerability-based vs configuration-based privesc; CIS Benchmarks (L1/L2) and DISA STIGs (CAT I–III) as the "secure" baseline; compliance tooling (Nessus, Lynis, OpenSCAP, CIS-CAT) vs offensive tooling (LinPEAS/WinPEAS/PowerUp); the six misconfiguration categories (users/groups, file permissions, services, scheduled tasks, credential storage, network config); and a situational-awareness → category enumeration → prioritise/chain methodology | ✅ Done |
| Linux Privilege Escalation: Enumeration | — | ⬜ Not started |
| Linux Privilege Escalation: Basics | — | ⬜ Not started |
| Linux Privilege Escalation: Automation | — | ⬜ Not started |
| Windows Privilege Escalation | — | ⬜ Not started |
| Jump (challenge) | — | ⬜ Not started |
| Windows Jump (challenge) | — | ⬜ Not started |

</details>

<details open>
<summary><strong>🟣 TryHackMe · Web Application Pentesting</strong> — the current focus</summary>

<br>

**Authentication** *(Session Management skipped — covered under Web App Vulnerabilities II)*

| Room | Focus | Status |
| --- | --- | --- |
| Enumeration & Brute Force | Enumerating authentication mechanisms (valid usernames, password policies, verbose errors, predictable reset tokens, HTTP Basic Auth) and using it to drive a targeted brute-force attack | ✅ Done |
| Session Management | — | ⬜ Skipped |
| JWT Security | JWT structure/signing algorithms, sensitive data leaking into the payload, signature-validation failures (no check, `alg:none` downgrade, crackable HS256 secrets, RS256→HS256 algorithm confusion), missing `exp`, and audience-claim cross-service relay attacks | ✅ Done |
| OAuth Vulnerabilities | OAuth 2.0 roles/grant types, the Authorization Code flow end to end, identifying OAuth in the wild, and four attack classes — `redirect_uri` token hijacking, CSRF via missing `state`, implicit-grant token theft via XSS, and insufficient token expiry — plus the OAuth 2.1 hardening changes | ✅ Done |
| Multi-Factor Authentication | Factor categories (know/have/are/somewhere/something-you-do) and 2FA mechanisms (TOTP, push, SMS, hardware tokens), then four real-world implementation flaws — OTP leaking in an XHR response, a session flag set before the OTP step, automated brute force surviving an auto-logout, and Evilginx-style real-time phishing proxies stealing the post-MFA session | ✅ Done |
| Hammer (challenge) | — | ⬜ Not started |

**Injection Attacks** *(the SQLi room is dual-filed under Vulnerabilities → SQLi)*

| Room | Focus | Status |
| --- | --- | --- |
| Advanced SQL Injection | Second-order (stored) SQLi, filter evasion (character encoding, no-quote/no-space bypasses), out-of-band exfiltration (SMB/HTTP/DNS via `INTO OUTFILE`/`xp_cmdshell`/`UTL_HTTP`), HTTP header injection, stored-procedure/XML/JSON injection, automation tools, and best practices | ✅ Done |
| NoSQL Injection | MongoDB operator injection (`$ne`/`$nin` login bypass + account enumeration, `$regex` character-by-character password extraction) and syntax injection (`$where` JS query breakout) | ✅ Done |
| XXE Injection | XML/DTD/entity fundamentals, in-band file disclosure, out-of-band exfiltration via hosted DTD + `php://filter` base64 chain, XXE+SSRF internal port scanning via Burp Intruder, mitigation per language | ✅ Done |
| Server-Side Template Injection | Template-engine fundamentals, engine fingerprinting, Smarty/Pug/Jinja2 exploitation (`system()`, `child_process.spawnSync`, Python `__subclasses__` sandbox escape), SSTImap automation, sandboxing + mitigation per engine | ✅ Done |
| LDAP Injection | LDAP tree/filter fundamentals, authentication bypass (wildcard + tautology `*)(\|(&`), blind boolean extraction, Python automation script | ✅ Done |
| ORM Injection | ORM fundamentals + framework comparison, SQLi-vs-ORM-injection distinction, identifying vulnerable methods per framework, Laravel `whereRaw()` weak implementation, Laravel query-builder `sort`-parameter JSON-extract bypass CVE, per-framework secure patterns | ✅ Done |
| Injectics (challenge) | — | ⬜ Not started |

**Advanced Server-Side Attacks**

| Room | Focus | Status |
| --- | --- | --- |
| Insecure Deserialisation | Serialisation/deserialisation fundamentals (PHP, Python Pickle), cookie-tampering property flips, object injection via PHP magic methods, gadget-chain automation (PHPGGC, ysoserial) | ✅ Done |
| SSRF | Basic SSRF against the local server (unvalidated file-read parameter leaking `config.php`) and against an internal server (reaching a private-IP admin panel), blind SSRF (out-of-band callback, time-based inference), and SSRF-driven denial of service — also filed under Vulnerabilities → SSRF | ✅ Done |
| File Inclusion, Path Traversal | PHP wrappers (`php://filter` transforms, `data://` inline payloads), base-directory-breakout bypasses, encoding/obfuscation (double encoding, `....//`), and three LFI→RCE escalation paths — session-file poisoning, log poisoning, and the base64-wrapper one-shot RCE | ✅ Done |
| Race Conditions | TOCTOU (Time-of-Check to Time-of-Use) vulnerabilities, hidden intermediate states in business logic, and exploiting a mobile-credit-transfer demo with Burp Suite Repeater's parallel request-group sending (HTTP/2 single-packet vs HTTP/1 last-byte synchronisation) | ✅ Done |
| Prototype Pollution | JS prototype/inheritance fundamentals, exploitation via property-definition-by-path (Lodash `_.set` → stored XSS), object-merge/clone functions (`__proto__` → silent property injection across every instance), prototype-pollution-driven DoS (overriding `Object.prototype.toString`), detection challenges + tooling | ✅ Done |
| Include (challenge) | — | ⬜ Not started |

**Advanced Client-Side Attacks**

| Room | Focus | Status |
| --- | --- | --- |
| XSS | Reflected, stored and DOM-based XSS with vulnerable-vs-fixed code in PHP/Node.js/Flask/ASP.NET, CVE labs (copyparty CVE-2023-38501 reflected, Hospital Management System CVE-2021-38757 stored), context breakouts and filter evasion | ✅ Done |
| CSRF | CSRF mechanism + impact, 3 types (traditional/XHR/Flash), 4 chained weaknesses (no token, predictable double-submit token + subdomain cookie injection, SameSite=Lax over GET, the Lax 2-minute POST grace period), XHR/CORS/Referer vectors, defences | ✅ Done |
| DOM-Based Attacks | The DOM, SPA frontend-framework pitfalls, source/sink model, DOM-based open redirect, DOM-based XSS vs conventional XSS, the delivery problem, 2010 Twitter incident — capped by a non-guided Vue-app challenge (not solved/written up) | ✅ Done |
| CORS & SOP | SOP (origin matching), CORS headers/ACAO, simple vs preflight requests, 3 misconfigurations (arbitrary origin reflection, bad regex, trusted null origin chained with stored XSS), defences | ✅ Done |
| Whats Your Name? (challenge) | Take control of a web app using client-side exploitation skills | ⬜ Not started |

**HTTP Request Smuggling** *(newest module)*

| Room | Focus | Status |
| --- | --- | --- |
| HTTP Request Smuggling | — | ⬜ Not started |
| HTTP/2 Request Smuggling | — | ⬜ Not started |
| Request Smuggling: WebSockets | — | ⬜ Not started |
| HTTP Browser Desync | — | ⬜ Not started |
| El Bandito (challenge) | — | ⬜ Not started |

</details>

<details>
<summary><strong>🟠 TryHackMe · Vulnerabilities</strong> — standalone labs outside any single path</summary>

<br>

| Room | Focus | Status |
| --- | --- | --- |
| SQL Injection Lab | Hands-on SQLi against a Flask/SQLite app: login bypass, `UNION` dumps, boolean-based blind, and two second-order injections + sqlmap tamper scripts | ✅ Done |
| sqlmap | Automating SQL injection with sqlmap — basic/enumeration/OS-access flag reference, and full GET- and POST-based target walkthroughs (saving a request, session caching, enumerating DBs/tables/columns, dumping data) | ✅ Done (Tasks 1-2; challenge ⬜) |
| Advanced SQL Injection | Second-order (stored) SQLi, filter evasion, out-of-band exfiltration, HTTP header injection, and automation — also filed under Web Application Pentesting → Injection Attacks | ✅ Done |
| Cross-Site Scripting (XSS) | Reflected, stored, DOM-based and blind XSS; escaping different injection contexts, filter bypasses, cookie exfiltration, and mitigations | ✅ Done |
| Server-Side Request Forgery (SSRF) | The four input vectors, identifying/confirming blind SSRF, deny-list/allow-list/open-redirect bypasses, cloud metadata theft, and the Acme avatar traversal lab | ✅ Done |
| Advanced SSRF | Basic SSRF against the local server and against an internal server (HRMS app), blind SSRF (OOB + time-based), and SSRF-driven denial of service — also filed under Web Application Pentesting → Advanced Server-Side Attacks | ✅ Done |
| Insecure Direct Object Reference (IDOR) | Broken access control / BOLA; plaintext, encoded, hashed and unpredictable object references, the two-account technique, where vectors hide, and the Acme customer-API lab | ✅ Done |

</details>

<details>
<summary><strong>🟧 PortSwigger · Web Security Academy</strong></summary>

<br>

| Path | Focus | Status |
| --- | --- | --- |
| SQL Injection | Full SQLi learning path — fundamentals, detection, UNION attacks, examining the database, every blind technique (conditional responses/errors, time delays, OAST), different contexts, second-order, prevention + cheat sheet | ✅ Done |

</details>

---

<p align="center"><sub>Personal study repo — not affiliated with TryHackMe, PortSwigger, or the vendors/frameworks mentioned.<br>All techniques documented for authorized, educational, and defensive use only.</sub></p>
