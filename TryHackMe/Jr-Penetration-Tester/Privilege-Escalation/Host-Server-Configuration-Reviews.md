# Host-Server Configuration Reviews

Privilege escalation is moving from a low-privileged foothold to a higher-privileged account. This room is the **conceptual foundation** before any hands-on Linux/Windows privesc: the two flavours of escalation, the industry baselines ("secure" is a published, checkable standard — not a vibe), the tools that audit at scale, the six misconfiguration categories that recur on every host, and a structured methodology for approaching a new target.

> This is the **TryHackMe "Host-Server Configuration Reviews" room** (Jr. Penetration Tester → Privilege Escalation, room 1 of 7). Tasks 1–7 and 9 (there is no Task 8 in the room). No hands-on exploitation here — that's the rooms that follow.
>
> 🔗 First room of the **Privilege Escalation** module. Next: Linux Privilege Escalation: Enumeration.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Glossary](#glossary)
- [Two kinds of privilege escalation (Task 1)](#two-kinds-of-privilege-escalation-task-1)
- [What a configuration review is (Task 2)](#what-a-configuration-review-is-task-2)
- [Security baselines and frameworks (Task 3)](#security-baselines-and-frameworks-task-3)
- [Automated compliance tooling (Task 4)](#automated-compliance-tooling-task-4)
- [The six misconfiguration categories (Task 5)](#the-six-misconfiguration-categories-task-5)
- [Structured enumeration methodology (Task 6)](#structured-enumeration-methodology-task-6)
- [Reading a CIS Benchmark recommendation (Task 7)](#reading-a-cis-benchmark-recommendation-task-7)
- [Key takeaways](#key-takeaways)

---

## In plain English

Every secure system has a published checklist of "correct" settings — who can log in, what permissions files should have, which services should run as what user. Admins drift from that checklist over time through convenience or oversight. **A configuration review is just comparing the real system against that checklist and listing the gaps.**

The twist: that exact same checklist serves two completely different people reading it for opposite reasons.

- A **defender** uses it to find and **fix** gaps (harden the system).
- An **attacker/pentester** uses it to find the *same* gaps and **exploit** them (escalate privileges).

It's the same document, the same findings — only the next step differs. This room teaches you the checklist categories and the standards they come from, so the hands-on rooms that follow (Linux/Windows privesc) have something to aim at instead of randomly poking around a host.

## Glossary

| Term | Plain meaning |
| --- | --- |
| **Privilege escalation (privesc)** | Going from a lower-privileged account to a higher one (user → root/admin). |
| **Vulnerability-based escalation** | Exploiting a *software bug* (kernel CVE, buffer overflow). Patching removes it. |
| **Configuration-based escalation** | Exploiting *how the system was set up* (bad permissions, exposed creds). Patching doesn't fix it — only reconfiguring does. |
| **Security baseline** | A documented, specific standard for "correctly configured" (not a vague best-practice feeling). |
| **CIS Benchmark** | Center for Internet Security's consensus-built configuration standard, per OS/platform. |
| **DISA STIG** | US DoD's mandatory, more prescriptive hardening guide for government/military systems. |
| **CAT I / II / III** | STIG severity tiers: I = highest risk, III = lowest. |
| **Level 1 / Level 2 (CIS)** | L1 = broadly safe to apply everywhere; L2 = deeper hardening, may break functionality. |
| **SUID bit** | Linux permission bit: the program runs as its *owner* (often root), not as whoever launched it. |
| **ACL** | Access Control List — Windows' permission model for files/registry keys. |
| **Unquoted service path** | A Windows service path with spaces and no quotes — Windows tries each word-boundary as a possible executable, which can be hijacked. |
| **SCAP** | Security Content Automation Protocol — a standard format for machine-readable compliance checks (used by OpenSCAP). |
| **LinPEAS / WinPEAS / PowerUp** | Automated *offensive* enumeration scripts — same checks as compliance tools, but output aimed at exploitation. |

---

## Two kinds of privilege escalation (Task 1)

| | Vulnerability-based | Configuration-based |
| --- | --- | --- |
| Targets | Bugs in software (kernel CVEs, buffer overflows) | How the admin set the system up |
| Needs | An unpatched system | Works even on a **fully patched** system |
| Fixed by | Patching | Re-hardening the configuration |
| Real-world frequency | Less common in mature, well-patched shops | **Most common vector in practice** |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>💡 Why this matters</strong> — a fully patched box with zero known CVEs can still be trivially owned if its permissions, services or stored credentials are sloppy. Patch management alone doesn't make a host safe. This room is entirely about the configuration side.
</div>

---

## What a configuration review is (Task 2)

A **configuration review** = audit a host's settings/permissions/services/policies against an accepted baseline, and list the deviations.

| | Vulnerability scan (Nessus/Qualys) | Configuration review |
| --- | --- | --- |
| Asks | "What software flaws/missing patches exist?" | "How is this system set up?" |
| Can pass one but fail the other | A host can have **zero CVEs** and still fail hard on configuration (over-privileged services, broad permissions, exposed creds) | |

**Exploit dev vs. config review** — different mindsets: exploitation asks *"what's broken in this software?"*; configuration review asks *"what's set up incorrectly on this system?"*

**When it happens in a pentest:** typically **post-exploitation** — after you already have a shell and are hunting for a path to a higher-privileged account. It can also run standalone, as a compliance/hardening audit with no exploitation at all.

**Same process, opposite intent:**

| | Defensive audit | Offensive enumeration |
| --- | --- | --- |
| Step 1 | Identify deviations | Identify weaknesses |
| Step 2 | Assess risk | Prioritise |
| Step 3 | Remediate | Escalate privileges |

Learning what "secure" looks like automatically teaches you what "insecure" looks like — and where to find it.

**Applies to any audit-able system:** workstations, file/database servers, domain controllers, web servers, network appliances, cloud instances. The categories of misconfiguration (Task 5) are consistent across all of them, even though the exact commands differ by OS.

---

## Security baselines and frameworks (Task 3)

A **baseline** replaces "what feels secure" with specific, measurable, published recommendations.

### CIS Benchmarks

- Published by the **Center for Internet Security**, consensus-built across government/industry/academia.
- Each recommendation includes: description, **rationale** (why it matters), **audit** procedure (how to check), **remediation** (how to fix).
- Two profile levels:

| Level | Meaning |
| --- | --- |
| **Level 1** | Practical, broadly applicable, minimal functional impact |
| **Level 2** | Deeper defence-in-depth hardening, may restrict functionality — test before deploying |

> Example (Ubuntu, L1): `PermitRootLogin` in `sshd_config` must be `no`. **Rationale:** direct root SSH login gives an attacker who guesses/brute-forces the root password immediate privileged access with no individual-account accountability.

### DISA STIGs

- Published by the US **Defense Information Systems Agency** — more prescriptive than CIS, **mandatory** on US government/military networks.
- Severity tiers:

| Tier | Risk |
| --- | --- |
| **CAT I** | Highest — exploit could directly cost confidentiality/integrity/availability |
| **CAT II** | Medium |
| **CAT III** | Low |

### How these connect to compliance law/standards

Broader frameworks (**PCI-DSS**, **ISO 27001**, **NIST 800-53**) often point *at* CIS/STIG hardening as their actual implementation guidance — e.g. "comply with PCI-DSS" in practice often means "apply this CIS Benchmark."

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>⚠️ Why a pentester should care about the framework, not just the finding</strong> — a deviation from the baseline the client itself adopted is the <em>most defensible</em> finding in a report ("you said you follow CIS; here's where you don't"). The benchmark also hands you a ready-made, comprehensive checklist instead of ad-hoc poking.
</div>

---

## Automated compliance tooling (Task 4)

Manually checking hundreds of recommendations across thousands of hosts doesn't scale. These tools do it programmatically.

| Tool | Runs | Audience | Notes |
| --- | --- | --- | --- |
| **Nessus** | Remote scan | Commercial, enterprise | Known for vuln scanning, but also does compliance audits against CIS/STIG/custom policies. A shared Nessus compliance report in grey/white-box engagements = a pre-built misconfiguration list. |
| **Lynis** | Locally, on the host | Open-source, Linux/macOS/Unix | Produces a hardening-index score + findings/warnings/suggestions. Useful both for admins hardening a box *and* a pentester with shell access — but running extra tooling on a target risks detection and may breach RoE. |
| **OpenSCAP** | Local/remote, SCAP-based | Open-source, gov/enterprise | Implements **SCAP** — a standard format for security policy checks; evaluates CIS/STIG-formatted profiles; machine-readable output. |
| **CIS-CAT** | Local/remote | CIS's own tool | Purpose-built for CIS Benchmark compliance; free **Lite** + commercial **Pro** versions; output maps directly to benchmark recommendation numbers. |

**Offensive counterparts:** **LinPEAS**, **WinPEAS**, **PowerUp** run on a compromised host and check many of the *same* things (writable files, weak service perms, stored creds) — but their output is written for exploitation, not remediation.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:10px 14px;border-radius:4px;margin:12px 0;">
<strong>💡 Two sides of one coin</strong> — a CIS check for "SSH root login disabled" and a LinPEAS flag on the SSH config are checking the <em>same thing</em>. Audience and intended next action differ; the underlying weakness doesn't.
</div>

---

## The six misconfiguration categories (Task 5)

A mental framework that applies regardless of OS or host role.

### 1. Users and groups
Violations of least privilege: unnecessary admin-group membership, over-privileged service accounts, un-disabled default accounts, weak/absent password policy.
> If a compromised account is already in an admin group, no further exploitation is even needed.

### 2. File and directory permissions

| Linux | Windows |
| --- | --- |
| **SUID bit** set on a binary that doesn't need it — the program runs as its *owner* (often root) regardless of who launched it; if it allows arbitrary command/file access, it's a direct escalation path | **ACLs** granting excessive access |
| World-writable scripts/binaries that a privileged process executes | Writable directories in the system **PATH** |
| `/etc/shadow` or SSH private keys readable too broadly | Weak default perms on install directories; misconfigured registry key ACLs |

### 3. Service configurations

Secure baseline: minimum necessary privilege, restrictive access controls, full unambiguous executable paths.

- Services running as **root/LocalSystem** when they don't need to.
- Service config files/binaries **writable** by a non-privileged user.
- **Unquoted service paths with spaces (Windows)** — classic and well-known:

```
C:\Program Files\My App\service.exe
```
Unquoted, Windows tries each space-delimited token as a candidate path **in order**:
1. `C:\Program.exe`
2. `C:\Program Files\My.exe`
3. `C:\Program Files\My App\service.exe`

If a non-privileged user can write to any **intermediate** directory (e.g. `C:\`), they drop a malicious `Program.exe` there, and it runs with the **service account's** privileges.

### 4. Scheduled tasks and cron jobs

Linux → **cron** (crontab files); Windows → **Task Scheduler**. Risk: a privileged task references a script/binary a non-privileged user can modify.

**Wildcard injection** example: a root cron job runs `tar cf /backup/archive.tar *` inside a directory. If you can create files there, name them `--checkpoint=1` and `--checkpoint-action=exec=sh shell.sh` — `tar`'s wildcard expansion feeds these as **command-line flags**, and your script runs as root.

### 5. Credential storage

Not a permissions bug — an **operational hygiene** failure: secrets left somewhere other accounts can read.

| Linux | Windows |
| --- | --- |
| `~/.bash_history` | `cmdkey /list` (Credential Manager) |
| Environment variables | `runas /savecred` saved creds |
| Config files with DB strings/API keys | Cleartext passwords in the registry |
| Over-permissive SSH private keys | `Unattend.xml` / Sysprep files, PowerShell history, `web.config` |

> Often the **most direct** escalation path — no technical exploit needed, just finding what's already sitting there.

### 6. Network configuration

Lower priority for *local* privesc, included for completeness — more relevant to lateral movement/attack-surface.

- A service bound to `0.0.0.0` (all interfaces) instead of `127.0.0.1` (localhost-only) — e.g. MySQL reachable from the network instead of local-only, with weak/no auth → data access or even command execution via `INTO OUTFILE` (MySQL) / `COPY TO PROGRAM` (PostgreSQL).
- Unnecessary open ports, overly permissive firewall rules on management interfaces, cleartext remote-management protocols.

---

## Structured enumeration methodology (Task 6)

OS-agnostic — defines **what** to check, not the exact commands (those come in the Linux/Windows rooms).

### Phase 1 — Situational awareness
Before checking anything specific: who am I (user, groups, privileges)? What OS/version/architecture? What's the hostname and the host's **role** (workstation / web server / DB server / domain controller)? Is it **domain-joined**? → this context shapes which categories are worth chasing first.

### Phase 2 — Category-based enumeration
Walk every category from Task 5, in a **consistent** order so nothing gets skipped:

1. **Users/groups** — all accounts, group memberships, who's admin/root, default accounts still enabled, password policy.
2. **File/directory permissions** — SUID/SGID + world-writable (Linux); PATH/install-dir/registry ACLs (Windows).
3. **Services** — list all, note the run-as account, check writability of binary/config, unquoted paths + security descriptors on Windows.
4. **Scheduled tasks/cron** — enumerate all, flag privileged ones, check permissions on what they execute.
5. **Credential storage** — history files, env vars, configs, registry, credential stores, deployment files.
6. **Network config** — listening ports, bound interfaces, firewall rules, unnecessary exposure.

### Phase 3 — Prioritisation and exploitation
Not all findings are equal — prefer the most **direct, reliable** route to the **highest** privilege.

- Stored root/admin credentials > a writable file in an hourly cron job (both valid, first needs zero waiting).
- **Chained findings** matter: a root cron job (Task category 4) running a **world-writable** script (category 2) is nothing alone, but combined = modify the script → wait for cron → root. Recognising cross-category chains is a core privesc skill.

**Automated tools fit here as a supplement, not a replacement** — they map to this same methodology and flag severity, but understanding *why* a check exists lets you catch false positives and gaps the tool misses. If compliance output (Nessus/Lynis) is already available, use it to decide which category to dig into first.

---

## Reading a CIS Benchmark recommendation (Task 7)

Every recommendation has the same shape — learn to read it once, read every benchmark forever.

| Field | What it holds | `/etc/shadow` example |
| --- | --- | --- |
| **Title** | The specific setting being assessed | "Ensure permissions on `/etc/shadow` are configured" |
| **Profile applicability** | Level 1 or Level 2 | — |
| **Description** | Plain-language requirement | Owned by `root`, group `shadow`, perms so only root can read/write, group can read |
| **Rationale** | The risk being mitigated | `/etc/shadow` holds password hashes — readable by non-privileged users means offline hash cracking becomes possible |
| **Audit** | How to check | `stat /etc/shadow`, compare to expected owner/perms |
| **Remediation** | How to fix | `chown`/`chmod` to the correct values |

**Reading it offensively** — don't ask "is this compliant?"; ask **"if it's not compliant, what can I do with that?"** For `/etc/shadow`: if the audit fails and it's readable, copy it, extract hashes, crack offline with **John the Ripper** or **Hashcat** — a cracked root hash is direct escalation.

Not every failed check is directly exploitable — some are defence-in-depth only. The real skill: separating **directly actionable** findings, findings that only matter **chained** with another, and low-priority informational noise.

**Mapping to tool output:** a Nessus/compliance result like `7.1.5 - Ensure permissions on /etc/shadow are configured - FAILED` is the *exact same* audit procedure as the benchmark text; a LinPEAS flag on a world-readable shadow file is the offensive mirror of the same check. Knowing the benchmark behind the tool output lets you judge severity/exploitability with more confidence than trusting the tool's label alone.

---

## Key takeaways

- Privesc splits into **vulnerability-based** (needs unpatched software) and **configuration-based** (works even fully patched, and is the more common real-world vector).
- A **configuration review** checks *how a system is set up*, not *what bugs its software has* — it's the same process whether the goal is remediation (defender) or escalation (attacker).
- **CIS Benchmarks** (consensus, L1/L2 profiles) and **DISA STIGs** (mandatory in US gov, CAT I–III severity) are the published "secure" standards — deviations from a baseline the client already adopted are the most defensible pentest findings.
- **Nessus, Lynis, OpenSCAP, CIS-CAT** automate compliance checks at scale; **LinPEAS/WinPEAS/PowerUp** run the same categories of check but aimed at exploitation.
- Six recurring misconfiguration categories on every host: **users/groups, file permissions (SUID/ACLs), services (unquoted paths), scheduled tasks/cron (wildcard injection), credential storage, network config**.
- Enumeration methodology: **situational awareness → category-by-category checks → prioritise/chain findings → exploit**. The most valuable findings are often **chains** across categories (writable script + privileged cron job), not single isolated issues.
- Reading a benchmark recommendation offensively means translating "audit/rationale" into "what can I actually do with this if it fails" — e.g. a readable `/etc/shadow` → offline cracking with John/Hashcat.

---

*Privilege Escalation room 1 of 7. Next: Linux Privilege Escalation: Enumeration.*
