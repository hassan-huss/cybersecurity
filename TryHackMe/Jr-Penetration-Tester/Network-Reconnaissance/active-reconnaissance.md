**Active reconnaissance** — interacting directly with target systems to discover information (use only on authorized targets).

---

**Table of contents**

- [Browser extensions](#browser-extensions)
- [Traceroute](#traceroute)
- [Quick tips](#quick-tips)

---

## Browser extensions

Browser extensions can turn a regular browser into a powerful reconnaissance platform. Below are useful, actively maintained extensions and why they matter.

- **FoxyProxy** — switch between proxies (Burp Suite, ZAP, SOCKS5). Useful when intercepting or routing traffic through different tools during an engagement.
- **User-Agent Switcher and Manager** — modify the User-Agent string to emulate different browsers, OSes, or devices. Helpful for discovering mobile-specific endpoints or version-dependent behavior. Note: some WAFs and CDNs detect rapid or suspicious User-Agent changes.
- **Wappalyzer** — passive technology fingerprinting (CMS, web server, JS frameworks, CDNs, analytics). Runs while you browse and gives quick insight into stack components.

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> Only use browser extensions against authorized targets. Extensions can leak credentials or telemetry to third-party services; treat them like any other external integration.
</div>

---

## Traceroute

### What it is
Traceroute maps the path between your system and a target by identifying each network hop (router) and, ideally, the round-trip time to that hop.

### How it works
Traceroute exploits the IP TTL (Time To Live) field: send packets with TTL=1, then TTL=2, TTL=3, etc. Each router that decrements a packet's TTL to zero is expected to respond with an ICMP "Time Exceeded" message, revealing that hop's IP.

### Limitations
- Some routers drop expired packets silently (shown as `*`), so you may know a hop exists without seeing its identity.
- Firewalls and filtering devices may block or rate-limit ICMP responses.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> ICMP and traceroute responses are frequently filtered or rate-limited by network operators. Do not assume silence means a clean path—corroborate with other tools.
</div>

### Why it matters
Knowing where latency or filtering occurs in the path helps with topology mapping, identifying choke points, and deciding follow-up techniques.

### Examples
Windows:
```powershell
tracert example.com
```

Linux / macOS:
```bash
traceroute example.com
```

---

<div style="background:#ecfdf5;border-left:4px solid #10b981;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong>
<ul>
	<li><strong>Record context:</strong> save timestamps, target hostnames, and the tool/flags used.</li>
	<li><strong>Use proxies carefully:</strong> when chaining tools, ensure you don't leak local credentials.</li>
	<li><strong>Respect rate limits:</strong> aggressive probing can trigger alarms.</li>
</ul>
</div>

---
## Telnet & banner grabbing

Telnet's original purpose was remote command-line access (developed 1969) and it has been superseded by SSH because Telnet transmits credentials in cleartext.

In reconnaissance, Telnet is useful for banner grabbing: opening a raw TCP connection to a service to read its initial response (the "banner"), which often discloses software name and version (for example: `Server: nginx/1.6.2`). Use that information to search CVE databases or exploit repositories.

Examples (basic):
```bash
# Connect to port 80 and view the HTTP response header
telnet example.com 80
# then type: GET / HTTP/1.1\nHost: example.com\n\n
# Use openssl for TLS services
openssl s_client -connect example.com:443 -servername example.com

# Quick HTTP header fetch with curl
curl -I https://example.com
```

Limits and alternatives:
- Some protocols require protocol-specific input (HTTP needs a GET; SMTP/FTP may advertise banners automatically).
- Telnet cannot inspect encrypted channels — use `openssl s_client` or `curl --head` for TLS/HTTPS.
- Banner information can be forged or hidden; corroborate with other sources.

---

_Last updated: 2026-09-08 — reorganized Telnet section and added examples._
Listening with Netcat
Netcat can also act as a server, listening on a specified port. This is useful for testing connectivity, transferring simple data, or setting up basic communication channels during an engagement.

On the server system, run nc -vnlp 1234 to start listening on port 1234. On the client system, run nc 10.128.190.112 1234 to connect. Once the connection is established, any text typed on one side is transmitted to the other. As you may recall from the Linux Fundamentals module, the exact order of the flags does not matter as long as the port number is preceded directly by -p.

Option	Meaning
-l	Listen mode
-p	Specify the port number
-n	Numeric only; no DNS resolution of hostnames
-v	Verbose output, useful for debugging
-vv	Very verbose output
-k	Keep listening after the client disconnects
The -p flag must appear directly before the port number. The -n flag avoids DNS lookups and associated warnings. Port numbers below 1024 require root privileges to listen on. For IPv6 listening, add the -6 flag with nc -6 -lp 1234. If you need encryption for sensitive data transfer, use ncat --ssl or pair nc with a tool like stunnel.