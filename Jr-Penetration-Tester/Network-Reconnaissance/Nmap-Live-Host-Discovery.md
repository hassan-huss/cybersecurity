You have learned how ARP, ICMP, TCP, and UDP can detect live hosts by completing this room. Any response from a host indicates that it is online. Below is a quick summary of the Nmap command-line options we covered.

Scan Type	Example Command
ARP Scan	sudo nmap -PR -sn 10.200.6.0/24
ICMP Echo Scan	sudo nmap -PE -sn 10.200.6.0/24
ICMP Timestamp Scan	sudo nmap -PP -sn 10.200.6.0/24
ICMP Address Mask Scan	sudo nmap -PM -sn 10.200.6.0/24
TCP SYN Ping Scan	sudo nmap -PS22,80,443 -sn 10.200.6.0/30
TCP ACK Ping Scan	sudo nmap -PA22,80,443 -sn 10.200.6.0/30
UDP Ping Scan	sudo nmap -PU53,161,162 -sn 10.200.6.0/30
Remember to add -sn if you are only interested in host discovery without port-scanning. Omitting -sn will let Nmap default to scanning live hosts for ports.

Option	Purpose
-n	no DNS lookup
-R	reverse-DNS lookup for all hosts
-sn	host discovery only
Effective live host discovery with Nmap lays the foundation for any successful penetration test. Knowing which hosts are online ensures accurate scoping and efficient follow-up scans. Stay tuned for more hands-on recon and enumeration insights.