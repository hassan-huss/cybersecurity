# Web Server Attacks - I

Fingerprinting and attacking web server software directly — Apache, Nginx, Python's built-in HTTP server, and Node/Express — using response headers, default error pages, and known misconfigurations per server.

---

**Table of contents**

- [In plain English](#in-plain-english)
- [Identifying web servers](#identifying-web-servers)

---

## In plain English

| Topic | Spot it by | Takeaway |
| --- | --- | --- |
| **Identifying web servers** | `Server` header, `X-Powered-By` header, default error pages | Every server **announces itself** somewhere — check the headers first; if those are hidden, the default error page usually gives it away instead. |

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
