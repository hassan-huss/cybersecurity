# Nmap Live Host Discovery

**Live host discovery** is the first step in network reconnaissance: confirm which systems are online before doing deeper port scans or exploitation work.

---

**Table of contents**

- [Discovery methods](#discovery-methods)
- [Useful Nmap flags](#useful-nmap-flags)
- [Key takeaway](#key-takeaway)

---

<div style="background:#eef8ff;border-left:4px solid #2b8cf0;padding:12px;border-radius:6px;margin:8px 0">
<strong>Info:</strong> Any response from a host indicates that it is online. Host discovery is used to map the live population before deeper enumeration.
</div>

## Discovery methods

Below is a quick summary of the Nmap options covered for host discovery.

| Scan type | Example command |
| --- | --- |
| ARP Scan | `sudo nmap -PR -sn 10.200.6.0/24` |
| ICMP Echo Scan | `sudo nmap -PE -sn 10.200.6.0/24` |
| ICMP Timestamp Scan | `sudo nmap -PP -sn 10.200.6.0/24` |
| ICMP Address Mask Scan | `sudo nmap -PM -sn 10.200.6.0/24` |
| TCP SYN Ping Scan | `sudo nmap -PS22,80,443 -sn 10.200.6.0/30` |
| TCP ACK Ping Scan | `sudo nmap -PA22,80,443 -sn 10.200.6.0/30` |
| UDP Ping Scan | `sudo nmap -PU53,161,162 -sn 10.200.6.0/30` |

---

## Useful Nmap flags

| Option | Purpose |
| --- | --- |
| `-n` | No DNS lookup |
| `-R` | Reverse-DNS lookup for all hosts |
| `-sn` | Host discovery only (no port scan) |

<div style="background:#fff7ed;border-left:4px solid #f59e0b;padding:12px;border-radius:6px;margin:8px 0">
<strong>Warning:</strong> If you are only interested in host discovery, add `-sn`. If you omit it, Nmap will default to scanning live hosts for open ports.
</div>

---

## Key takeaway

Effective live host discovery with Nmap lays the foundation for any successful penetration test. Knowing which systems are online ensures accurate scoping and efficient follow-up scans.

<div style="background:#ecfdf5;border-left:4px solid #10b981;padding:12px;border-radius:6px;margin:8px 0">
<strong>Tip:</strong> Treat host discovery as a scope filter: discover active hosts first, then choose the right follow-up scan for each target.
</div>

---

_Last updated: 2026-09-09 — organized into a cleaner study layout with styled callouts._