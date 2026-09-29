# Mobile Application Security

Mobile apps hold some of the most sensitive data we own — banking, messages, identity — which makes them high-value red-team targets, yet mobile testing is often treated as a "later" skill. This room is the foundation: **how mobile apps are built**, the **four-phase methodology** testers follow, **static** and **dynamic** analysis, the **common vulnerability categories** (mapped to the OWASP Mobile Top 10), and a **MobSF static-analysis CTF** to tie it together.

> This is the **TryHackMe "Mobile Application Security" room** (Jr. Penetration Tester → Specialized Domains, room 1). Whole room (Tasks 1–7). These are **study notes for authorised testing** — the room is conceptual + static analysis (no exploit weaponisation). The Task 7 challenge secrets are **planted CTF answers in a fictional "Helix Solutions" training package** (not real credentials), recorded here as room answers.
>
> 🔗 First room of the **Specialized Domains** module. Mobile apps talk to the same backends as web apps, so this overlaps with [API Pentesting](../Web-Application-Vulnerabilities-II/API-Pentesting.md) (the backend the app calls) and [Session Management](../Web-Application-Vulnerabilities-II/Session-Management.md) (tokens/auth reused on device).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Glossary](#glossary)
- [How mobile applications work (Task 2)](#how-mobile-applications-work-task-2)
- [The mobile pentesting methodology (Task 3)](#the-mobile-pentesting-methodology-task-3)
- [Static analysis (Task 4)](#static-analysis-task-4)
- [Dynamic analysis (Task 5)](#dynamic-analysis-task-5)
- [Common mobile vulnerabilities (Task 6)](#common-mobile-vulnerabilities-task-6)
- [Practical challenge: Leaky Package (Task 7)](#practical-challenge-leaky-package-task-7)
- [Key takeaways](#key-takeaways)

---

## In plain English

A mobile app isn't one file — it's a **zip-like package** containing the compiled code, a **manifest** (a declaration to the OS: what permissions it wants, which of its parts other apps can touch), and resources (images, strings, small databases). Because you can get that package and **unzip it**, a huge amount of testing happens *before you ever run the app*.

Testing splits into two halves:

- **Static analysis** — read the app without running it. Unpack it, read the manifest for over-broad permissions and exposed components, and grep the decompiled code for **hardcoded secrets** (API keys, DB passwords). This is where the quickest, most embarrassing findings come from. A tool called **MobSF** automates most of it.
- **Dynamic analysis** — run the app and watch it behave. Intercept its network traffic (is it using HTTPS?), read what it writes to logs, and use runtime tools (**Frida**/**Objection**) to hook into functions, bypass **SSL pinning**, or defeat a biometric check.

Every finding gets mapped to the **OWASP Mobile Top 10** (M1–M10) so developers know how serious it is. The two platforms mirror each other: Android's `AndroidManifest.xml` ≈ iOS's `Info.plist`; Android `.apk` ≈ iOS `.ipa`.

## Glossary

| Term | Plain meaning |
| --- | --- |
| **Package** | The single distributable app file — Android `.apk`, iOS `.ipa`. A zip of code + config + resources. |
| **Manifest** | The app's declaration to the OS: name, version, permissions, component config. Android: `AndroidManifest.xml`; iOS: `Info.plist`. |
| **Sandbox** | OS isolation — each app runs in its own container and can't read another app's files by default. |
| **Component** | A distinct part of an app (UI screen, background service, event receiver, data provider). |
| **Exported component** | A component reachable by *other apps* on the device. If it wasn't meant to be, it's an attack surface. |
| **Decompiler** | Tool that turns the compiled binary back into readable-ish code (`jadx`, `apktool`). |
| **MobSF** | Mobile Security Framework — open-source automated **static** (and dynamic) analysis tool; works on APK *and* IPA. |
| **SSL pinning** | App trusts only a specific certificate → defeats proxy interception. A control, not a vuln; testers bypass it. |
| **Frida** | Runtime instrumentation framework — hook/modify functions in a *running* app. |
| **Objection** | Runtime toolkit built on Frida — disables SSL pinning, inspects data, no source needed. |
| **OWASP Mobile Top 10** | Industry checklist of the 10 most critical mobile risks (M1–M10). |
| **ATS** | iOS App Transport Security — forces HTTPS. Disabled by `NSAllowsArbitraryLoads = true`. |

---

## How mobile applications work (Task 2)

Know what you're pulling apart *before* you start.

- **The package** — build output is one package file bundling the app binary, a config/manifest file, and supporting resources (images, fonts, local DBs, stored strings). Get the package and you can dissect it without running a line of code.
- **The manifest / config file** — the app's declaration to the OS: name, version, **requested permissions**, and how internal **components** are configured (internal vs accessible to other apps). *A goldmine* — misconfigs here (over-exposed components, over-broad permissions) are common real-engagement findings.
- **The sandbox model** — each app runs isolated in its own container; by default it can't read another app's files or touch its processes. It shapes the attack surface: breaking out, or an app weakening *its own* isolation, is a notable finding.
- **Application components** — apps are modular. Each component can be **internal** (in-app only) or **exported** (other apps can call it). An exported component that wasn't meant to be exposed = attack surface (revisited in Task 6).
- **Why structure matters** — knowing where things live makes static analysis systematic: *config/permissions → manifest*, *logic → compiled code*, *keys/hardcoded values → resource files*.

---

## The mobile pentesting methodology (Task 3)

Repeatable process > running tools and hoping. Four phases, findings categorised by the **OWASP Mobile Top 10**.

| Phase | What happens |
| --- | --- |
| **1. Reconnaissance** | Gather context before testing: what the app does, who uses it, which backends it talks to, where to get the package. More context → more focused testing. |
| **2. Static analysis** | Examine *without running*: unpack, read config, inspect decompiled code for visible issues. Often the quickest, most impactful findings. |
| **3. Dynamic analysis** | *Run* it and observe: intercept traffic, watch runtime data handling, probe components. Surfaces bugs only visible when operating. |
| **4. Reporting** | Document findings, explain risk, give clear reproduction + actionable fixes. A finding nobody can reproduce is useless. |

**OWASP Mobile Top 10** — the closest thing to a universal mobile checklist. Current categories include improper credential use, inadequate supply-chain security, insecure authentication, insufficient input/output validation, insecure communication, and insufficient binary protections. Don't memorise it — know it exists and *map findings to it* to communicate risk.

**Testing approaches** (how much you know going in):

| Approach | Access |
| --- | --- |
| **Black-box** | No prior knowledge — just the app, like an external attacker. |
| **Grey-box** | *Most common in engagements* — some info (test creds, partial source). |
| **White-box** | Full source, docs, architecture — most thorough, needs most time. |

**Scope & reporting** — agree up front which app versions, which backend environments, and which actions are permitted (testing out of scope, even accidentally, has real consequences). Findings prioritised **critical / high / medium / low**, each with description, reproduction steps, impact, and recommended fix.

---

## Static analysis (Task 4)

Examining the app **without running it** — almost always the first technical step, because it needs nothing but the package and often yields findings fast.

**1. Obtain & unpack.** Get the package (from the client, pulled off a device, or an app store), then unpack with a **decompiler** — output isn't identical to source but is close enough to read logic, follow data flow, and spot how sensitive data is handled.

**2. Read the manifest — Android (`AndroidManifest.xml`).** First place to look. Hunt for:

- **Over-requested permissions** — does it ask for access it doesn't need? Broad permissions widen the blast radius if compromised.
- **Exported components** — activities/services/providers marked accessible to other apps when they shouldn't be. An exported component handling sensitive data/privileged actions is a direct attack surface.
- **Insecure configuration flags** — insecure defaults that weaken posture (e.g. `debuggable`, `allowBackup`).

**3. Read the manifest — iOS (`Info.plist`).** Same purpose, same categories. Watch especially for **`NSAllowsArbitraryLoads = true`** inside the `NSAppTransportSecurity` dictionary — it disables **App Transport Security**, letting the app talk plain **HTTP**. Maps to **M5: Insecure Communication** + **M8: Security Misconfiguration**.

**4. Hunt for hardcoded secrets.** One of the most common findings. Developers leave API keys, passwords, encryption keys, internal URLs in code or resources — trivial to extract once you have the package. Look in: decompiled code, string resource files, bundled config files, local DB files in assets. On iOS, `.plist` files are a prime location (fully readable once the IPA is unpacked). A simple pattern search (key/token/password-looking strings) surfaces them fast.

**5. Automate with MobSF.** Open-source tool that statically analyses a package and reports permissions, hardcoded secrets, insecure config, and more — **cross-platform (APK + IPA)**. Use it for a fast posture overview, then spend manual effort on what automation misses (business-logic flaws, subtle data handling).

**Static findings → OWASP:**

- **M1: Improper Credential Usage** — hardcoded credentials/API keys/tokens in code or resources.
- **M8: Security Misconfiguration** — over-requested permissions, wrongly-exported components, insecure manifest flags.

---

## Dynamic analysis (Task 5)

*Conceptual — no lab needed.* Observing and interacting with the **running** app: what it sends over the network, how it stores data, what it logs, how it responds when you probe components. The difference from static: **behaviour, not code**. An app can look clean on paper but mishandle data at runtime (and vice versa) — you need both phases.

- **Traffic interception** — route the app's comms through a **proxy** (e.g. Burp) to see every request/response: what it sends, whether it's over HTTPS/TLS, how tokens are handled, whether it talks to unexpected third parties. Unencrypted traffic is an immediate finding → **M5: Insecure Communication**.
- **SSL pinning** — a *defence* where the app trusts only specific certificate(s), so a proxy's cert is rejected and the app refuses to talk. Not a vuln — a control — but testers must work around it to see traffic. **Objection** (built on Frida) disables pinning on a running app without source access, and can also interact with exported components and inspect runtime data.
- **Runtime instrumentation** — attach to a running app and observe/modify it from outside (like a debugger built for security). **Frida** is the standard: list classes/methods, hook functions to see data passing through, bypass auth checks, extract in-memory data. Finds bugs static analysis never would. Objection sits on top of Frida; when its built-ins don't cover a hook you need, write a custom Frida script.
- **Insecure logging** — apps log for debugging; production logging should be minimal and never sensitive. Devs sometimes forget to strip debug logs, leaking usernames/tokens/API responses/passwords to a log other apps can read. Reviewing log output while using the app is quick and rewarding → **M9: Insecure Data Storage**.

---

## Common mobile vulnerabilities (Task 6)

A reference to return to during the challenge and future engagements. Each grounded in the OWASP Mobile Top 10.

| Vulnerability | What it is | Concrete example | OWASP |
| --- | --- | --- | --- |
| **Insecure data storage** | Sensitive data stored where/how it's easy to read without authorisation | Plaintext tokens in config files; unencrypted local DB; private data in shared storage; caching API responses with sensitive contents | **M9** |
| **Improper platform usage** | Misusing (or ignoring) OS security features | Requesting camera+contacts+location+mic when only one is needed → attacker inherits all if compromised; passing sensitive data between components other apps can intercept | **M1 / M8** |
| **Insecure auth & session mgmt** | Mobile mirrors web auth flaws, with nuances | Weak non-expiring tokens; tokens stored insecurely; missing re-auth for sensitive actions; **biometric bypass** — if the app just asks the OS "was the biometric accepted?" and trusts the yes/no with no server-side check, it's bypassable at runtime with instrumentation | **M3** |
| **Exposed application components** | An exported component with no access control | A component that shows account details, callable by *any* app on the device → a malicious app retrieves the data with no credentials | **M8** |
| **Insufficient binary protections** | Missing controls that make reversing/tampering harder | No **obfuscation** (code readable after decompile); no **tamper detection**; no **root/jailbreak detection** (banking/health apps are expected to detect a rooted/jailbroken device and warn or refuse — its absence is a finding) | **M7** |

---

## Practical challenge: Leaky Package (Task 7)

**Scenario:** *Helix Solutions* rushed an internal employee-portal app to release; sensitive config may have been left inside the packages. Two packages built from the same codebase — an Android **APK** and an iOS **IPA** — analysed in **MobSF** via static analysis. (MobSF login: `mobsf` / `mobsf`.)

### Part 1 — Android APK

| Step | Finding | Detail | OWASP |
| --- | --- | --- | --- |
| Overview | Security Score **40/100**; `com.tryhackme.leakypackage`; **1/2 exported activities** | Low score signals multiple issues; one activity reachable by other apps | — |
| Permissions | 5 **dangerous** perms | `ACCESS_FINE_LOCATION`, `CAMERA`, `READ_CONTACTS`, `READ_EXTERNAL_STORAGE`, `RECORD_AUDIO` — no legit reason for an employee portal → over-requested | **M8** |
| Manifest | `android:debuggable="true"` (**HIGH**) | Lets an attacker with ADB/physical access attach a debugger, dump heap, intercept methods. Never in production | **M8** |
| Manifest | `AdminPanelActivity` **exported, unprotected** (WARNING) | `android:exported="true"` with no permission attr → any installed app can launch it via explicit intent, bypassing the app's own auth flow. Also `allowBackup="true"` | **M8** |
| Code | Hardcoded secrets in `HelixConfig.java` + `DatabaseHelper.java` | See answers below — extractable in under a minute with `jadx`/`apktool` | **M1** |

**Recorded challenge answers (planted CTF values):**

- **Exported activity:** `com.tryhackme.leakypackage.AdminPanelActivity`
- **Hardcoded API key** (`HelixConfig.java`): `AIzaSyHX3mR9vKcT8nP2wY5dL0qJ7eZbFgVuN4o`; internal API URL `http://internal.helixsolutions.local/api/v2`
- **Hardcoded DB credentials** (`DatabaseHelper.java`): host `db.internal.helixsolutions.local`, db `helix_employee_portal`, port `5432` (PostgreSQL), user `helix_admin`, **password `Helix@2024!DBroot`** → full JDBC connection string embedded. Impact: an attacker has everything needed to connect directly to the production database.

### Part 2 — iOS IPA

| Step | Finding | Detail | OWASP |
| --- | --- | --- | --- |
| Overview | Security Score **45/100**; `com.tryhackme.LeakyPackage` | Decompiled Assets: View Info.plist / Class Dump / Download IPA | — |
| Info.plist | **`NSAllowsArbitraryLoads = true`** (HIGH) | Inside `NSAppTransportSecurity` → disables ATS entirely → app can send data over unencrypted HTTP to any server. Confirmed under *Transport Security* | **M5 / M8** |
| Files | Sensitive value in a bundled `.plist` | `internal_config.plist` shipped inside the IPA — no production app should bundle an internal secret | **M9** |

**Recorded challenge answers (planted CTF values):**

- **ATS-disabling key:** `NSAllowsArbitraryLoads` (set to `true`)
- **Bundled internal secret** (`LeakyPackage.app/internal_config.plist`): key `internal_secret` = **`HelixiOS_Secret_9mK2P!wX`**

> 🟦 **Android ↔ iOS mirror:** same codebase, same *classes* of bug expressed differently — hardcoded creds (M1) in Java constants vs a bundled `.plist`; misconfig (M8) as `debuggable`/exported-activity vs ATS disabled. This is why the OWASP mapping matters more than platform specifics.

---

## Key takeaways

- A mobile app is an **unpackable bundle** — code + manifest + resources — so most findings come from **static analysis** *before* you run anything.
- The **manifest** (`AndroidManifest.xml` / `Info.plist`) is the highest-yield file: over-broad **permissions**, **exported components**, and insecure flags (`debuggable`, `allowBackup`, `NSAllowsArbitraryLoads`) all live here.
- **Hardcoded secrets** (API keys, DB passwords, internal URLs, bundled `.plist` tokens) are the classic, trivially-extracted finding → **M1 / M9**.
- **Dynamic analysis** catches what static can't: unencrypted traffic (M5), insecure logging (M9), biometric/auth bypass (M3) — using a proxy, **Frida**, and **Objection** (which also beats SSL pinning).
- Always **map findings to the OWASP Mobile Top 10** — it's how you turn "this looks bad" into a risk a developer can act on.
- **MobSF** is the fast first pass (APK + IPA); manual review then finds the business-logic and subtle data-handling flaws automation misses.

---

*Specialized Domains room 1. Next in module: Cloud Security Fundamentals · LLM Pentesting. Related: [API Pentesting](../Web-Application-Vulnerabilities-II/API-Pentesting.md) · [Session Management](../Web-Application-Vulnerabilities-II/Session-Management.md).*
