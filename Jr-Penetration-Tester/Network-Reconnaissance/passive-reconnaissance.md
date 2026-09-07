Key tools and techniques:

WHOIS: Domain registration details including registrar, dates, and name servers. Most personal details are now redacted for privacy.
DNS lookups: A/AAAA (IP addresses), MX (mail servers), TXT (SPF/DMARC/verification), and other record types, queried via public resolvers like 1.1.1.1.
Subdomain enumeration: DNSDumpster for DNS aggregation and graphing, and crt.sh for Certificate Transparency log searches, which is the most effective passive method for discovering subdomains via public SSL/TLS certificates.
Exposed services: Shodan.io for device banners, ports, and hosting information.
Purpose	Command-line Example
Lookup WHOIS record	whois tryhackme.com
Lookup DNS A records (legacy)	nslookup -type=A tryhackme.com
Lookup DNS MX records at specific server (legacy)	nslookup -type=MX tryhackme.com 1.1.1.1
Lookup DNS TXT records (legacy)	nslookup -type=TXT tryhackme.com
Lookup DNS A records (recommended)	dig tryhackme.com A
Lookup DNS MX records at specific server (recommended)	dig @1.1.1.1 tryhackme.com MX
Lookup DNS TXT records (recommended)	dig tryhackme.com TXT
Passive subdomain discovery (browser-based)	Visit https://crt.sh and search %.tryhackme.com