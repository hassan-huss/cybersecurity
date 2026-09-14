# Web Server Attacks - I

Fingerprinting and attacking web server software directly — Apache, Nginx, Python's built-in HTTP server, and Node/Express — using response headers, default error pages, and known misconfigurations per server.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Identifying web servers](#identifying-web-servers)
- [Python HTTP server exposure](#python-http-server-exposure)
- [Apache2](#apache2)
- [Node.js / Express](#nodejs--express)
- [Nginx](#nginx)

---

## In plain English

| Topic | Spot it by | Takeaway |
| --- | --- | --- |
| **Identifying web servers** | `Server` header, `X-Powered-By` header, default error pages | Every server **announces itself** somewhere — check the headers first; if those are hidden, the default error page usually gives it away instead. |
| **Python HTTP server** | `python3 -m http.server` running anywhere reachable | It has **one mode: serve everything** in the folder — no auth, no hidden files, no exceptions. Finding it exposed isn't hacking, it's just reading what it was already handing out. |
| **Apache2** | Version in `Server` header, `/server-status`, `.bak` files via Gobuster | Four checks catch most misconfigured Apache boxes: **read the version, browse anything that lists, visit `/server-status`, brute-force for files nothing links to.** |
| **Node.js / Express** | `X-Powered-By: Express`, JSON responses, debug endpoints | The app is **code, not files** — and developers ship it with the debugging left on, so it happily **tells you its own routes, its stack traces, and its secrets** if you just ask. |
| **Nginx** | `Server: nginx/x`, `autoindex` listings, `/nginx_status` | Same three checks as Apache, **different vocabulary** (`server_tokens`, `autoindex`, `stub_status`). It usually sits out front as a reverse proxy, so what it leaks hints at the internal setup behind it. |

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>How to use these notes:</strong> these are a lookup sheet, not something to memorise. Remember the one-line hook above; come back here for the exact commands when you need them.
</div>

---

## Identifying web servers

The server software in use shapes everything that follows — which misconfigurations are possible, which paths are worth checking, which tools work best. Identifying it isn't a formality; it's the first decision every later step depends on. Fortunately, most servers give themselves away through default configuration.

### The `Server` response header

The most direct signal. Request headers only with a `HEAD` request:

```bash
# -s  suppress the progress bar
# -I  send a HEAD request — headers only, no body
curl -sI http://10.129.173.9:80
```

```
HTTP/1.1 200 OK
Date: Wed, 08 Apr 2026 13:59:00 GMT
Server: Apache/2.4.58 (Ubuntu)
Last-Modified: Fri, 03 Apr 2026 18:12:44 GMT
ETag: "29af-64e9243796aa2"
Accept-Ranges: bytes
Content-Length: 10671
Vary: Accept-Encoding
Content-Type: text/html
```

Default `Server` header per server, in this lab:

| Port | Server software | Default `Server` header |
| --- | --- | --- |
| 80 | Apache2 | `Apache/2.4.x (Ubuntu)` |
| 8000 | Python HTTP server | `SimpleHTTP/0.6 Python/3.xx.x` |
| 3000 | Node.js Express | *(none — not set by default)* |
| 8080 | Nginx | `nginx/1.xx.x` |

Express is the outlier: neither Express nor the Node.js HTTP layer beneath it sets a `Server` header by default — a developer has to add one explicitly. On a real Express app, a **missing** `Server` header is itself a signal worth following up.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> a hardened deployment may show a bare <code>Apache</code> with no version, or suppress the header entirely. Defaults are verbose; production configs are often deliberately not. Don't assume every target will be this easy — fall back to the signals below when it isn't.
</div>

### The `X-Powered-By` header

Some frameworks add this to reveal the application layer running behind (or in place of) the server software:

```
X-Powered-By: Express
```

For Express specifically, this — not `Server` — is the primary fingerprint, since Express relies on it as its self-identifier instead of a `Server` banner. Check for it on any port where `Server` is missing or unhelpfully generic.

### Browser DevTools

Same information, no extra tooling: open the target in Firefox, right-click → **Inspect** (or `F12`) → **Network** tab, refresh to capture the request, select the main request, and read **Response Headers**.

### Default error pages

A `HEAD` request (`-I`) returns no body, so it can't show an error page. Switch to a plain `GET`:

```bash
# HEAD request: headers only, no body
curl -sI http://10.129.173.9:PORT/

# GET request: full response including body
curl -s http://10.129.173.9:PORT/nonexistent-page-xyz
```

Requesting a path that doesn't exist returns each server's **default 404 page**, and those pages look distinctly different:

| Server | 404 page style |
| --- | --- |
| Python HTTP server | Plain text response |
| Nginx | Version number in the HTML footer |
| Apache | Server name in the page body |

This is the fallback fingerprint for when the `Server` header has been suppressed — the error page format alone can still identify the software underneath.

---

## Python HTTP server exposure

Python ships a built-in HTTP server anyone can start with one command:

```bash
# Serves the current working directory over HTTP on port 8000
python3 -m http.server 8000
```

It's meant for quickly sharing files or testing a static site on a local network. The problem is it's **convenient enough to forget about** — left running on a public-facing box, an internal share, or a cloud instance where port 8000 got opened and never closed. There's no access control, no authentication, and no logging beyond whatever the OS captures.

### What it serves

Everything in the working directory — **including dotfiles** like `.env`. There's no `.htaccess` equivalent, no blocklist, no config file to restrict paths. Apache and Nginx disable directory listing by default and let admins restrict individual paths; Python's HTTP server has exactly one mode: **serve everything**.

### Directory listing

If the folder has no `index.html`, the server auto-generates an HTML page listing every file it can see:

```bash
curl -s http://10.129.173.9:8000/
```

Page title `Directory listing for /` confirms it — everything listed there is directly downloadable.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> if the root doesn't show a listing, an <code>index.html</code> is probably being served instead — try requesting a path directly, or browse into subdirectories.
</div>

### Accessing dotfiles

Linux hides dotfiles from normal directory browsing by convention only — Python's HTTP server doesn't respect that convention and serves them like any other file. `.env` is a prime target since it commonly holds real credentials:

```bash
curl -s http://10.129.173.9:8000/.env
```

```
SECRET_KEY=dev-secret-key-do-not-use
DATABASE_URL=postgresql://webapp:S3cur3DBPass!@localhost/production
DEBUG=True
```

### Downloading and inspecting archives

A `.zip` or `.tar.gz` sitting in the listing is worth pulling down — developers sometimes leave backup archives in the same folder they're serving, containing source code, database dumps, or config files:

```bash
curl -s http://10.129.173.9:8000/backup.zip -o backup.zip
unzip backup.zip -d backup-contents/
cat backup-contents/db_dump.sql
```

### Why this matters

There's no vulnerability to trigger here — the server is working exactly as designed. The finding *is* the misconfiguration: it's running where it shouldn't be, serving files that shouldn't be public. On a real engagement, documenting this means stating not just that the server exists, but exactly what it exposes and what an attacker could do with that.

---

## Apache2

The most widely deployed web server in the world — it shows up in nearly every engagement touching web infrastructure. Ubuntu's default Apache configuration leaves several things enabled that testers routinely find and report. Four checks cover most of it.

### 1. Version disclosure

```bash
curl -sI http://10.129.173.9:80 | grep -i server
# -> Server: Apache/2.4.58 (Ubuntu)
```

Ubuntu's Apache defaults to `ServerTokens OS`, which includes the OS label alongside the version — enough to check for known CVEs and understand the server's capabilities.

### 2. Directory listing

The `Options +Indexes` directive makes Apache show a file listing whenever a directory has no `index.html`. Sometimes intentional (an internal file share), sometimes an accident on a path with sensitive data.

```bash
curl -s http://10.129.173.9/files/
```

A page titled `Index of /files` lists filenames, sizes, and last-modified dates for everything in that directory.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong> when you find a directory listing, read <em>every</em> file in it. CSV exports, internal notes, and backup files often end up in directories meant only for casual internal use.
</div>

### 3. The `mod_status` page

Apache ships a built-in status page via `mod_status`. Correctly configured, it's reachable only from `localhost`; misconfigured with `Require all granted`, it's reachable from anywhere:

```bash
curl -s http://10.129.173.9:80/server-status
```

It shows active connections and the paths they're requesting, total requests served since startup, worker states (idle/writing/reading/closing), and the exact server version and start time — real-time information about other users' activity and internal paths, on top of confirming the version.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> <code>mod_status</code> ships <em>enabled</em> by default on Ubuntu with a <code>Require local</code> restriction in <code>conf-available/security.conf</code> — but a <code>Require all granted</code> directive placed anywhere in a virtual host config silently overrides that restriction. The module config itself never has to be touched. Always check <code>/server-status</code>, even on a server that looks like it's using safe defaults.
</div>

### 4. Finding unlinked files with Gobuster

Backup files, old config copies, and forgotten test files often sit in the document root with nothing linking to them. Gobuster finds them by brute-forcing paths from a wordlist:

```bash
# -u  target URL
# -w  wordlist path
# -x  also try these extensions on every word
gobuster dir -u http://TARGET_IP:80 \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
  -x bak,txt,html -t 20
```

```
/.htpasswd            (Status: 403) [Size: 275]
/.htaccess            (Status: 403) [Size: 275]
/backup.bak           (Status: 200) [Size: 178]
/files                (Status: 301) [Size: 308] [--> http://10.129.173.9/files/]
/server-status        (Status: 200) [Size: 17539]
```

A `.bak` file returning `200` is almost always worth pulling:

```bash
curl -s http://10.129.173.9:80/backup.bak
```

```
# Apache config backup - DO NOT COMMIT
ServerName company.internal
DocumentRoot /var/www/html
# DB credentials below
# user: dbadmin pass: Backup2024!
# Last updated: 2024-11-15
```

A `.htpasswd` file is worth just as much attention: Apache uses it to store usernames and **hashed** passwords for HTTP Basic Auth. Finding one exposed hands you a hash to crack offline, and confirms that some part of the site uses Basic Auth — telling you which paths are worth trying authenticated access on.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong> <code>common.txt</code> is large, and <code>-x</code> triples the requests (one per extension per word) — a full run can take several minutes against a remote target. Run without <code>-x</code> first to find live directories, then re-run with a targeted extension like <code>-x bak</code> once you know where to look.
</div>

### Putting it together

The pattern is consistent across Apache targets: **check the version header → browse anything that lists a directory → visit `/server-status` → Gobuster for unlinked files.** These four steps cover the majority of what a misconfigured Apache server exposes.

---

## Node.js / Express

Node apps behave differently from Apache and the Python server. They aren't serving static files from a document root — they run **application code** that decides what to return for every request. That flexibility is powerful, and it's exactly where mistakes creep in.

The recurring theme: **development-mode features left enabled in production.** Debug endpoints, verbose errors, and exposed environment variables all come from the same habit — code that worked in dev went live as-is. The result is an app that tells you how it's built and, sometimes, what credentials it uses.

### 1. Framework fingerprinting

```bash
curl -sI http://MACHINE_IP:3000
# -> X-Powered-By: Express
```

Express sets `X-Powered-By: Express` automatically unless the developer explicitly disables it. That confirms the framework and tells you what to look for next.

### 2. Reading the application version

Many Express apps return a JSON status response at the root:

```bash
curl -s http://MACHINE_IP:3000
# -> {"status":"ok","app":"company-portal","version":"1.2.0"}
```

Note any `version` field — matching it against known CVEs or changelogs can surface useful leads.

### 3. Triggering verbose errors

Express's built-in error handler hides stack traces when `NODE_ENV=production`. But developers often write **custom** error handlers that leak traces regardless of `NODE_ENV` — so a `production` setting only protects you if nothing overrides the default handler. Hit an endpoint that will fail:

```bash
curl -s http://MACHINE_IP:3000/api/users | python3 -m json.tool
```

```json
{
    "error": "connect ECONNREFUSED 127.0.0.1:5432",
    "stack": "Error: connect ECONNREFUSED 127.0.0.1:5432\n    at /opt/nodeapp/app.js:16:15\n    at Layer.handle ...",
    "query": "SELECT * FROM users"
}
```

A `500` from an API endpoint is worth chasing. The **stack trace** is the prize: it leaks internal file paths (`/opt/nodeapp/app.js`), module names, and sometimes the exact database query that failed — internals a bare status code would never reveal.

### 4. Enumerating routes via debug endpoints

One of the most useful things a misconfigured Express app can do is hand you a list of all its own routes. Developers build a debug endpoint for convenience during development, then forget to remove it:

```bash
curl -s http://MACHINE_IP:3000/api/routes
```

```json
[{"method":"GET","path":"/"},{"method":"GET","path":"/api/users"},
 {"method":"GET","path":"/api/routes"},{"method":"GET","path":"/api/debug/env"}]
```

This shows every path the app handles up front — no Gobuster guessing needed.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> a route-lister works by reading Express's internal <code>app._router.stack</code>. That's an internal property whose shape changes between Express versions — Express 5 reworked the router internals enough to break Express-4-style implementations. If a route endpoint returns an odd format or errors out, the target may be on a different Express version than this lab.
</div>

### 5. Exposed environment variables

Env vars in Node apps often hold DB credentials, API keys, and config flags. A debug endpoint that returns `process.env` is a major finding:

```bash
curl -s http://MACHINE_IP:3000/api/debug/env
```

```json
{"NODE_ENV":"development","DB_PASSWORD":"NodeDBPass2024!","PORT":"3000",
 "DB_HOST":"localhost:5432","APP_NAME":"company-portal"}
```

Document any `DB_PASSWORD`, `SECRET_KEY`, etc. The `NODE_ENV` value is telling in itself: `development` on a production box signals the app was deployed without hardening.

### 6. Static file serving

Express commonly uses `express.static()` to serve front-end assets. Client-side JavaScript often embeds API URLs, internal hostnames, or debug flags as constants. Once you know the routes, check what's served statically:

```bash
curl -s http://MACHINE_IP:3000/static/config.js
```

```javascript
// Client-side configuration
const API_BASE = 'http://internal-api.company.local:8080';
const DEBUG = true;
const VERSION = '1.2.0';
```

Config files served as static assets are easy to overlook because they're *technically* meant to be public (the browser needs them) — but "meant to be public" isn't the same as "contains only public information."

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> <code>express.static()</code> returns a silent <code>404</code> for dotfiles (files starting with <code>.</code>) by default — the <em>opposite</em> of Python's HTTP server, which serves them freely. A <code>404</code> on <code>.env</code> over an Express static route doesn't mean the file is absent; the middleware is blocking it. You'd need shell access or a different route to reach it.
</div>

### Putting it together

Each step narrows the scope for the next: **headers** confirm the framework → **errors** reveal the internals → **debug endpoints** enumerate the routes → **env vars** expose credentials → **static files** show what the developers assumed was safe to expose. The chain works because every finding points you at where to look next.

---

## Nginx

Nginx occupies a different space from Apache and Node. It's most often a **reverse proxy, load balancer, or high-performance static file server** — in production it usually sits *in front of* an application server and handles public-facing traffic. That front-line positioning is what makes its configuration matter: a misconfigured instance can expose internal structure, reveal operational data, or serve files it shouldn't.

The misconfiguration categories are the **same three as Apache** — version disclosure, directory listing, exposed status page — just with Nginx's own directive vocabulary.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> in this lab Nginx runs on port <strong>8080</strong> because Apache already holds port 80 on the same machine. In a real deployment Nginx would typically be on port 80 or 443.
</div>

### 1. Version disclosure

```bash
curl -sI http://10.130.184.217:8080 | grep -i server
# -> Server: nginx/1.24.0 (Ubuntu)
```

Nginx exposes its version in the `Server` header by default. Where Apache uses `ServerTokens`, Nginx uses **`server_tokens`** (default `on`). That one directive controls *both* the `Server` header and the version string in default error pages — set `server_tokens off` and it's suppressed in both places at once. So if the header is missing, a `404` reveals whether it's genuinely off or only half-configured:

```bash
curl -s http://10.130.184.217:8080/nonexistent-path
```

```html
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx/1.24.0 (Ubuntu)</center>
</body>
</html>
```

### 2. Directory listing with `autoindex`

Nginx does **not** list directories by default. A developer enables it by adding `autoindex on` to a location block:

```nginx
location /files/ {
    autoindex on;
    root /var/www/nginx/;
}
```

Legitimate for file-sharing setups — a misconfiguration only when it's on a path holding sensitive files, or left on in production without access controls.

```bash
curl -s http://10.130.184.217:8080/files/
```

```html
<h1>Index of /files/</h1><hr><pre><a href="../">../</a>
<a href="deploy-notes.txt">deploy-notes.txt</a>        03-Apr-2026 18:23    148
<a href="old-backup.tar.gz">old-backup.tar.gz</a>       03-Apr-2026 18:23    236
<a href="server-config.txt">server-config.txt</a>       03-Apr-2026 18:23    135
</pre><hr>
```

Nginx's autoindex format is a simple filename / date / size table. Read every file listed — these paths are often configured as shared storage and then filled with operational data.

### 3. The `nginx_status` endpoint

Nginx's **`stub_status`** module exposes real-time connection metrics at a configurable URL. Secure config restricts it to localhost; the misconfigured version allows any IP:

```nginx
location /nginx_status {
    stub_status;
    allow all;   # Should be: allow 127.0.0.1; deny all;
}
```

```bash
curl -s http://10.130.184.217:8080/nginx_status
```

```
Active connections: 1
server accepts handled requests
 15 15 15
Reading: 0 Writing: 1 Waiting: 0
```

The compact, unlabeled output reads as:

| Line | Meaning |
| --- | --- |
| `Active connections` | Connections open right now |
| `15 15 15` | Totals since startup: **accepted**, **handled**, **requests** (in that order) |
| `Reading / Writing / Waiting` | Current active connections broken down by state |

Not directly exploitable, but it leaks server load and usage patterns — and on a real engagement it's a finding because it confirms the internal monitoring setup and hints that other monitoring endpoints may be similarly exposed.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong> Nginx config lives in <code>/etc/nginx/</code> on Ubuntu. With shell access, reading <code>/etc/nginx/sites-available/</code> shows you exactly which directories are exposed and which modules are enabled.
</div>

### Putting it together

Parallel structure to the Apache investigation — **check the version header → browse any `autoindex` directory → hit the status endpoint.** The paths and directives differ (`server_tokens`, `autoindex`, `stub_status`), but the investigative approach is identical.
