# Nmap Basic Port Scanning

This section covers the most common Nmap scan types used to discover open TCP and UDP ports on a target host.

## Common Scan Types

| Scan Type | Example Command | Purpose |
| --- | --- | --- |
| TCP Connect Scan | `nmap -sT MACHINE_IP` | Performs a full TCP connection attempt to each target port. |
| TCP SYN Scan | `sudo nmap -sS MACHINE_IP` | Sends SYN packets without completing the full TCP handshake. Commonly used for stealthier scans. |
| UDP Scan | `sudo nmap -sU MACHINE_IP` | Checks for open UDP services, which are often missed by TCP-only scans. |

These scan types give you a solid starting point for identifying services that are listening on a host.

## Useful Port Selection Options

| Option | Purpose |
| --- | --- |
| `-p-` | Scan all ports. |
| `-p1-1023` | Scan only ports 1 through 1023. |
| `-F` | Scan the 100 most common ports. |
| `-r` | Scan ports in consecutive order instead of random order. |
| `-T<0-5>` | Set timing; `-T0` is the slowest and `-T5` is the fastest. |
| `--max-rate 50` | Limit the scan to a maximum of 50 packets per second. |
| `--min-rate 15` | Keep the scan rate at or above 15 packets per second. |
| `--min-parallelism 100` | Ensure at least 100 probes are sent in parallel. |

## Example Commands

```bash
nmap -sT MACHINE_IP
sudo nmap -sS MACHINE_IP
sudo nmap -sU MACHINE_IP
nmap -p1-1023 MACHINE_IP
nmap -F MACHINE_IP
nmap -T4 -p- MACHINE_IP
```

## Quick Tips

- Use `-sT` for a straightforward TCP scan.
- Use `-sS` when you want a faster, stealthier SYN scan.
- Use `-sU` to check for UDP services.
- Use `-F` for a quick scan of common ports.
- Use `-p-` when you want to scan every port on the host.

## Notes

- `-sS` typically requires root or administrator privileges.
- UDP scans can be slower and may require more time to complete.
- Always use Nmap responsibly and only against systems you own or are authorized to test.
