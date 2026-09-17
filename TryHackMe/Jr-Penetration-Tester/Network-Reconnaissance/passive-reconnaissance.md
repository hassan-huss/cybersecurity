**Passive reconnaissance** — collecting information from public sources without interacting directly with the target systems.

---

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> Passive reconnaissance relies on public sources and should only be used against authorized targets. Data from public services may be incomplete or rate-limited.
</div>

## At-a-glance

- **WHOIS** — domain registration metadata (registrar, creation/expiry dates, name servers). Note: many personal details are now redacted for privacy.
- **DNS lookups** — A/AAAA, MX, TXT, NS records; use public resolvers (1.1.1.1, 8.8.8.8) to avoid local cache effects.
- **Subdomain enumeration** — Certificate Transparency logs (crt.sh), DNS aggregators (DNSDumpster), and passive sources for discovering subdomains without active probing.
- **Exposed services** — Shodan and similar search engines reveal services, banners, and host information from internet-wide scans.

---

## Tools & techniques

- **WHOIS**
	- Purpose: discover registrar, name servers, and registration dates.
	- Example:
```bash
whois example.com
```

- **DNS lookups**
	- Purpose: resolve A/AAAA (IP), MX (mail), TXT (SPF/DMARC), NS records.
	- Recommended: `dig` (more flexible and script-friendly).
	- Examples:
```bash
# Query A record
dig example.com A

# Query MX record using a specific resolver
dig @1.1.1.1 example.com MX

# Query TXT record
dig example.com TXT
```

- **Legacy (nslookup)** — still available on Windows; example:
```powershell
nslookup -type=A example.com
nslookup -type=MX example.com 1.1.1.1
nslookup -type=TXT example.com
```

- **Subdomain discovery**
	- Use Certificate Transparency search at https://crt.sh (search for `%.example.com`).
	- DNSDumpster and other aggregation sites can reveal historical DNS data and mappings.

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> Certificate logs and historical DNS may expose legacy or decommissioned subdomains; verify relevance before acting on them.
</div>

- **Shodan / Censys**
	- Purpose: find exposed devices, open ports, and service banners discovered by internet-wide scans.
	- Use web UI or API for targeted lookups.

---

## Quick checklist

<div style="background:#ecfdf5;border-left:4px solid #10b981;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong>
<ul>
	<li><strong>Record context:</strong> note timestamps, resolvers/services, and query parameters.</li>
	<li><strong>Correlate sources:</strong> combine certificates, DNS, and Shodan results before active probing.</li>
	<li><strong>Respect legal limits:</strong> only enumerate targets you are authorized to test and avoid data exfiltration.</li>
</ul>
</div>

---

_Last updated: 2026-09-08 — added styled admonitions for Info, Warning, and Tip._