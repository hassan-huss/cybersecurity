# Nmap Advanced Port Scans

This room covered several advanced Nmap scan techniques that go beyond basic port discovery.

## Advanced Scan Types

| Scan Type | Example Command | Purpose |
| --- | --- | --- |
| TCP Null Scan | `sudo nmap -sN MACHINE_IP` | Sends a packet with no TCP flags set. |
| TCP FIN Scan | `sudo nmap -sF MACHINE_IP` | Sends a TCP packet with only the FIN flag set. |
| TCP Xmas Scan | `sudo nmap -sX MACHINE_IP` | Sends a packet with the FIN, PSH, and URG flags set. |
| TCP Maimon Scan | `sudo nmap -sM MACHINE_IP` | Uses a special flag combination to infer port state. |
| TCP ACK Scan | `sudo nmap -sA MACHINE_IP` | Helps map firewall rules rather than open services. |
| TCP Window Scan | `sudo nmap -sW MACHINE_IP` | Uses TCP window size responses to infer port status. |
| Custom TCP Scan | `sudo nmap --scanflags URGACKPSHRSTSYNFIN MACHINE_IP` | Lets you define a custom set of TCP flags. |
| Spoofed Source IP | `sudo nmap -S SPOOFED_IP MACHINE_IP` | Sends traffic with a forged source IP address. |
| Spoofed MAC Address | `--spoof-mac SPOOFED_MAC` | Changes the source MAC address used by the scan. |
| Decoy Scan | `nmap -D DECOY_IP,ME MACHINE_IP` | Mixes your scan traffic with decoy sources. |
| Idle (Zombie) Scan | `sudo nmap -sI ZOMBIE_IP MACHINE_IP` | Uses an idle host to infer port states without direct interaction. |

## Packet Fragmentation Options

| Option | Purpose |
| --- | --- |
| `-f` | Fragment IP data into 8-byte chunks. |
| `-ff` | Fragment IP data into 16-byte chunks. |

## Additional Scan Options

| Option | Purpose |
| --- | --- |
| `--source-port PORT_NUM` | Specify the source port number to use. |
| `--data-length NUM` | Append random data to reach the desired packet length. |
| `--reason` | Explain how Nmap reached its conclusion about a port state. |
| `-v` | Enable verbose output. |
| `-vv` | Enable more detailed verbose output. |
| `-d` | Enable debugging output. |
| `-dd` | Provide even more detail for debugging. |

## How These Scans Work

These scan types rely on setting TCP flags in unusual ways to elicit a response from ports. Null, FIN, and Xmas scans are commonly used to provoke a response from closed ports, while Maimon, ACK, and Window scans can provide information about both open and closed ports.

## Example Commands

```bash
sudo nmap -sN MACHINE_IP
sudo nmap -sF MACHINE_IP
sudo nmap -sX MACHINE_IP
sudo nmap -sA MACHINE_IP
sudo nmap -sW MACHINE_IP
sudo nmap -sI ZOMBIE_IP MACHINE_IP
nmap -D DECOY_IP,ME MACHINE_IP
```

## Quick Tips

- Use `-sN`, `-sF`, and `-sX` for stealth-oriented scans that rely on unusual TCP flag combinations.
- Use `-sA` and `-sW` to gather firewall or filtering behavior.
- Use `-sI` carefully, since it depends on a legitimate idle host.
- Use `-D` when you want to obscure the origin of the scan traffic.
- Use `-v`, `-vv`, and `-d` when troubleshooting or validating scan behavior.

## Notes

- Some advanced scans may be less reliable depending on target OS and firewall behavior.
- Stealth and spoofing techniques can be noisy and may trigger defensive systems.
- Always use these techniques only against systems you own or are authorized to test.