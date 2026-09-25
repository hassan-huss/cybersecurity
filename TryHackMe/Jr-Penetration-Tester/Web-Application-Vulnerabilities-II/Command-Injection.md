# Command Injection

Smuggling extra **operating-system commands** into a web app that builds a shell command out of your input. The app meant to run one command with your value inside it; you use a shell operator (`;`, `&`, `&&`, `|`) to bolt on a second command of your own. The injected command runs with the **web server's privileges** — read files, exfiltrate data, or land a reverse shell.

> This is the **TryHackMe "Command Injection" room** (Jr. Penetration Tester → Web Application Vulnerabilities II). It covers what command injection is, how it arises in PHP/Python code, detecting blind vs verbose, exploitation payloads on Linux and Windows, a live practical, and remediation.
>
> 🔗 Same module: [Session Management](./Session-Management.md), [Broken Authentication](./Broken-Authentication.md), [File Inclusion](./File-Inclusion.md). Related standalone labs: [SQLi](../../Vulnerabilities/SQLi/SQL-Injection-Lab.md), [SSRF](../../Vulnerabilities/SSRF/SSRF.md). Command injection, SQLi and file inclusion are all **injection** bugs — untrusted input crossing into a trusted interpreter (a shell, a database, a filesystem path).

---

**Table of contents**

- [In plain English](#in-plain-english)
- [What command injection is (and RCE)](#what-command-injection-is-and-rce)
- [How it arises in code](#how-it-arises-in-code)
- [Detecting it: verbose vs blind](#detecting-it-verbose-vs-blind)
- [Shell operators & payloads](#shell-operators--payloads)
- [Practical: reading the flag](#practical-reading-the-flag)
- [Remediation](#remediation)
- [Key takeaways](#key-takeaways)

---

## In plain English

Some web pages do their job by quietly running a real terminal command on the server — pinging a host, searching a file, converting an image. They stitch your input into that command as text. If nothing checks your input, you can end the intended command early and start your own, because the shell treats `;` (and friends) as "…and now run this next."

| Term | Plain meaning |
| --- | --- |
| **Command injection** | You make the server run *your* OS command by hiding it in an input |
| **Shell operator** | A character the shell reads as "chain another command" — `;` `&` `&&` `\|` |
| **Verbose** | The injected command's output shows up on the page (easy) |
| **Blind** | No output shown — you confirm it ran by side effects like a time delay |
| **RCE** | Remote Code Execution — the general goal; command injection is one route to it |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>Shell</strong> = the OS program that interprets command lines (bash on Linux, cmd/PowerShell on Windows). <strong>System call / dangerous function</strong> = code that hands a string to the shell (PHP <code>exec()</code>/<code>system()</code>/<code>shell_exec()</code>/<code>passthru()</code>, Python <code>subprocess</code>, Node <code>child_process.exec()</code>). <strong>Privileges</strong> = what the server's OS account is allowed to do; your command inherits them. <strong>Reverse shell</strong> = a shell that connects back to you, giving interactive control. <strong>Redirection (<code>&gt;</code>)</strong> = send a command's output into a file. <strong>Sanitisation / allowlist</strong> = reject anything that doesn't match an expected safe format. <strong>CWE-78</strong> = the formal ID for OS command injection.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not rote memorisation. The one idea to keep: <em>the dangerous function is never the bug — passing unvalidated input to it is</em>. Everything else is picking the right operator and confirming the command ran.
</div>

---

## What command injection is (and RCE)

A web app runs OS commands as part of normal functionality. When a developer passes user input into a system command **without checks**, an attacker can inject additional commands alongside the legitimate one.

- **Privileges:** the injected command runs as **the same OS user as the application**. If the server runs as `joe`, your command runs as `joe` with joe's permissions. If the app runs elevated, impact scales up.
- **Command injection vs RCE:** **RCE** = the broad outcome of running code on a remote system. **Command injection** = *one specific technique* to achieve RCE (others: insecure deserialization, memory corruption). All command injection is RCE; not all RCE is command injection.
- **OWASP / CWE:** OWASP Top 10:2025 **A05 Injection**; OS command injection specifically = **CWE-78** (Improper Neutralization of Special Elements used in an OS Command).

---

## How it arises in code

The languages give you functions that run shell commands — they're not dangerous alone; they become dangerous when **user input reaches them unvalidated**.

**PHP** — a song search that greps a file:

```php
<?php
$songs = "/var/www/html/songs";                          // 1 storage dir
if (isset($_GET["title"])) {
    $title = $_GET["title"];                             // 2 raw user input
    $command = "grep $title /var/www/html/songtitle.txt"; // 3 concatenated in, no checks
    $search = exec($command);                            // 4 executed by the shell
    ...
}
?>
```

Normal `?title=Yesterday` → `grep Yesterday /var/www/html/songtitle.txt` (harmless). But `?title=; cat /etc/passwd` builds:

```bash
grep ; cat /etc/passwd /var/www/html/songtitle.txt
```

The shell reads `;` as a separator: `grep` runs with no useful args and fails silently, then `cat /etc/passwd …` dumps the sensitive file. (Real code would query a DB, not grep — the point is the *pattern*: input → system call, nothing in between.)

**Python (Flask)** — an extreme "web terminal":

```python
import subprocess
from flask import Flask
app = Flask(__name__)

def execute_command(shell):
    return subprocess.Popen(shell, shell=True, stdout=subprocess.PIPE).stdout.read()

@app.route('/<shell>')
def command_server(shell):
    return execute_command(shell)     # whatever's in the URL path is run
```

Visiting `http://flaskapp.thm/whoami` runs `whoami` on the server. Same principle, any language: **user input → system call with no checks = command injection.**

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>The tell when reading code:</strong> a string built with user input via concatenation/interpolation, then handed to <code>exec/system/shell_exec/passthru</code> (PHP), <code>subprocess … shell=True</code> (Python), or <code>child_process.exec()</code> (Node). <code>shell=True</code> and string-built commands are the red flags.
</div>

---

## Detecting it: verbose vs blind

You confirm command injection by getting the app to run something *observable*. Two cases:

| | **Verbose** | **Blind** |
| --- | --- | --- |
| Output shown? | Yes — injected command's result appears in the response | No — page looks identical either way |
| Confirm with | `; whoami` → username appears | **side effects**: time delay, or write-to-file |
| Difficulty | Easy | Harder — needs experimentation |

**Verbose:** inject `; whoami` into (say) a ping utility; the username shows up below the normal output. `ping` and `whoami` are good first probes — their output is instantly recognisable.

**Blind — no output, so use side effects:**

- **Time delay** (most common): `; ping -c 10 127.0.0.1` should make the response take ~10s longer. If response time scales with the ping count, that's strong evidence it ran. `sleep 10` does the same when `ping` isn't available.
- **Write to a readable file:** `; whoami > /var/www/html/output.txt`, then browse to `http://target.thm/output.txt` to read the result (redirect output into the web root).
- **`curl` to send the payload directly** (URL-encoded — `%3B` is `;`, `%20` is space):

```bash
curl "http://vulnerable.app/process.php?search=The%20Beatles%3B%20whoami"
```

Blind testing is trial-and-error; Linux vs Windows syntax differs, so try several approaches.

---

## Shell operators & payloads

**Operators to chain a second command** (put the injected command after one of these): `;` (run next regardless), `&` (background/next), `&&` (run next only if the first succeeds), `|` (pipe first's output into next).

**Useful payloads once injection is confirmed:**

| Payload | Linux | Windows | Purpose |
| --- | --- | --- | --- |
| `whoami` | ✅ | ✅ | What user the app runs as |
| list dir | `ls` | `dir` | Find config/`.env` files with tokens, API keys, creds |
| time delay | `ping` / `sleep` | `ping` / `timeout` | Confirm **blind** injection (hang for a measurable period) |
| foothold | `nc` (netcat) | — | Spawn a **reverse shell** → interactive access, hunt privesc |

---

## Practical: reading the flag

Target: a web form that passes input into a system command; goal is `/home/tryhackme/flag.txt`. Methodical approach:

1. **Baseline** — submit normal input, see what the app does with it.
2. **Probe** — inject `; whoami` (verbose) or a time-delay payload (blind) to confirm injection and learn which it is.
3. **Chain the read** — append a command that prints the flag. Several operators achieve the same result — try more than one:

```bash
; cat /home/tryhackme/flag.txt          # semicolon: run regardless of grep's result
&& cat /home/tryhackme/flag.txt         # only if the first command succeeded
| cat /home/tryhackme/flag.txt          # pipe
```

If output isn't shown (blind), redirect it somewhere web-readable and browse to it:

```bash
; cat /home/tryhackme/flag.txt > /var/www/html/f.txt      # then open /f.txt
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>No flag value invented here</strong> — the flag is per-instance in your VM. The method above (baseline → confirm → chain <code>cat</code> on the flag path) retrieves it; the room deliberately has multiple working payloads, so if <code>;</code> is filtered, try <code>&&</code>, <code>|</code>, a newline, or backticks/<code>$( )</code> command substitution.
</div>

---

## Remediation

Same principles across languages (examples in PHP):

- **Avoid the dangerous functions** where possible — prefer a safe API/library that does the job without a shell (e.g. a native function instead of shelling out to a command).
- **Constrain input format.** Client-side `pattern` limits the field, but is only a *first* layer:

```html
<input type="text" name="ping" pattern="[0-9]+">    <!-- digits only -->
```
```php
echo passthru("/bin/ping -c 4 " . $_GET["ping"]);   <!-- still needs server-side checks -->
```

  Client-side validation is bypassable (curl/Burp send requests directly), so **server-side validation is essential**.
- **Server-side sanitisation / validation** — specify the exact allowed type/format and reject everything else; strip special chars like `>`, `&`, `/`. Example: validate a numeric param before use.

```php
if (!filter_input(INPUT_GET, "number", FILTER_VALIDATE_INT)) {
    // reject: not a valid number
}
```

- **Know that filters aren't bulletproof.** If an app strips certain characters, an attacker can encode the same string differently — e.g. hex-encoding `/etc/passwd`:

```php
$payload = "\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64";  // = /etc/passwd
```

  The filter sees hex and lets it through; the OS still resolves `/etc/passwd`.
- **Defence in depth** — no single layer catches everything. Combine **input validation + sanitisation + allowlisting + least privilege** (run the app as a low-privilege user so even a successful injection is contained).

---

## Key takeaways

- **Command injection = your OS command runs on the server** because user input is concatenated into a shell command with no validation. It runs with the **app's privileges**.
- **RCE is the outcome; command injection is one technique** for it (alongside deserialization, memory corruption).
- **Red flag in code:** user input → `exec/system/shell_exec/passthru` (PHP), `subprocess(..., shell=True)` (Python), `child_process.exec()` (Node) via string concatenation.
- **Operators to chain:** `;` `&` `&&` `|` (and newline / `` ` `` / `$( )` for substitution).
- **Verbose** = output on the page (`; whoami`). **Blind** = no output → confirm via **time delay** (`ping`/`sleep`, Windows `timeout`) or **redirect-to-file** then browse to it.
- **Payloads:** `whoami`, `ls`/`dir`, time-delay for blind, `nc` for a reverse shell foothold.
- **Fix:** avoid shell functions, validate/sanitise **server-side** (allowlist expected format), remember filters are bypassable (hex/encoding), and apply **defence in depth + least privilege**.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How this sits vs the others:</strong> it's the same shape as <a href="../../Vulnerabilities/SQLi/SQL-Injection-Lab.md">SQLi</a> and <a href="./File-Inclusion.md">File Inclusion</a> — untrusted input crossing into an interpreter — but the interpreter here is the <em>OS shell</em>, which is why the payoff (RCE) is usually the most direct. The defence rhymes too: never build the command/query/path from raw input; use a safe API and validate against an allowlist.
</div>
