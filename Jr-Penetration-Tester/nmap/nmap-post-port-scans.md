# Nmap Post-Port Scans

In this room, we learned how to gather more detail about a target after identifying open ports. This includes detecting running services, determining service versions, identifying the host operating system, enabling traceroute, selecting useful Nmap scripts, and saving results in multiple output formats.

## Key Options

| Option | Meaning |
| --- | --- |
| `-sV` | Determine service and version information for open ports. |
| `-sV --version-light` | Try the most likely probes for version detection. |
| `-sV --version-all` | Try all available probes for deeper version detection. |
| `-O` | Detect the target operating system. |
| `--traceroute` | Run traceroute to the target. |
| `--script=SCRIPTS` | Run one or more Nmap scripts. |
| `-sC` or `--script=default` | Run the default scripts. |
| `-A` | Equivalent to `-sV -O -sC --traceroute`. |
| `-oN` | Save output in normal text format. |
| `-oG` | Save output in grepable format. |
| `-oX` | Save output in XML format. |
| `-oA` | Save output in normal, XML, and grepable formats at the same time. |

## Example Commands

```bash
nmap -sV MACHINE_IP
nmap -sV --version-light MACHINE_IP
nmap -sV --version-all MACHINE_IP
nmap -O MACHINE_IP
nmap --traceroute MACHINE_IP
nmap --script=http-enum MACHINE_IP
nmap -sC MACHINE_IP
nmap -A MACHINE_IP
nmap -oN results.txt MACHINE_IP
nmap -oG results.grep MACHINE_IP
nmap -oX results.xml MACHINE_IP
nmap -oA results MACHINE_IP
```

## Quick Tips

- Use `-sV` when you want to identify what service is running on an open port.
- Use `-O` when you need a rough guess of the target operating system.
- Use `-A` for a broad, high-information scan that combines version detection, OS detection, scripts, and traceroute.
- Use `-oA` if you want multiple output formats from a single scan.

## Notes

- Version detection can take longer than a simple port scan.
- Scripts can provide valuable information, but some may be noisy or slower than others.
- Saving scan results helps with documentation, comparison, and follow-up analysis.