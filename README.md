# Network Traffic Analysis using Wireshark

## Overview
In this project I built a small lab, ran a SYN port scan against my own target machine, and analyzed the traffic in Wireshark. I then sent controlled SYN traffic with hping3 to compare how a port scan looks against repeated connection attempts to one service.

**Tools:** Wireshark, Nmap, hping3, Kali Linux, Ubuntu (VirtualBox)

## Lab Setup
| Machine | Role | IP Address |
|---|---|---|
| Kali Linux | Attacker / capture machine | 192.168.56.102 |
| Ubuntu | Target | 192.168.56.101 |

Both VMs run on a VirtualBox host-only network, so all traffic stays inside my own lab.

## 01. Lab Connectivity Test
I pinged the target from Kali to confirm the machines could communicate. All 4 packets were received with 0% packet loss.

<img src="screenshots/01_ping_test.png" width="700" alt="Ping test">

## 02. Nmap SYN Scan
I ran `sudo nmap -sS 192.168.56.101` from Kali. The scan finished in 1.57 seconds and found 6 open TCP ports: 21 (FTP), 22 (SSH), 80 (HTTP), 139 (NetBIOS), 445 (SMB) and 8080 (HTTP-proxy). The other 994 ports were closed.

<img src="screenshots/02_nmap_scan.png" width="700" alt="Nmap SYN scan">

## 03. SYN Packet Capture
I captured on eth1 and used this display filter to show only the first packet of each connection attempt:

`tcp.flags.syn == 1 && tcp.flags.ack == 0`

All the SYN packets came from one source IP and one source port (48017) and went to many different destination ports in random order. This is the pattern of a port scan (MITRE ATT&CK T1046, Network Service Discovery).

<img src="screenshots/03_syn_filter.png" width="700" alt="SYN filter in Wireshark">

## 04. TCP Conversations
The Conversations window shows 1,000 separate TCP conversations, each with a single 58-byte packet. Each one is just a SYN, with no completed connection. I had "Limit to display filter" turned on, so only the SYN packets are counted. That is why packets from the target (B to A) show as 0.

<img src="screenshots/04_conversations.png" width="700" alt="TCP conversations">

## 05. Network Endpoints
The Endpoints window shows that 192.168.56.102 sent 1,000 packets (58 kB) and received none, while 192.168.56.101 received 1,000 packets. This identifies the scanner as the top talker.

<img src="screenshots/05_endpoints.png" width="700" alt="Network endpoints">

## 06. Controlled hping3 SYN Traffic
For comparison I sent controlled SYN traffic to port 80 using hping3:

`<paste the exact hping3 command you used>`

I filtered with `tcp.port == 80 && tcp.flags.syn == 1`. Here the source port increases each time (2981, 2982, 2983...) and the destination is always port 80, with about one packet every 0.2 seconds. The target replied with SYN-ACK each time because port 80 is open. This was low-rate test traffic, not a denial-of-service attack.

<img src="screenshots/06_hping3.png" width="700" alt="hping3 SYN traffic">

## Scan vs. hping3 Comparison
| | Nmap SYN scan | hping3 test |
|---|---|---|
| Source port | Same for all packets (48017) | Changes every packet |
| Destination ports | Many, in random order | Always port 80 |
| Meaning | Reconnaissance (finding open ports) | Repeated attempts to one service |

## Conclusion
A port scan shows one source port going to many destination ports. The hping3 traffic showed many source ports going to one destination port. Looking at the port pattern is how I can tell reconnaissance from repeated connection attempts to one service.

If this were a real alert, my next steps would be to check firewall and server logs for the source IP, report it for blocking if it is not an authorized scanner, and look for follow-up activity on the open ports (SSH, FTP, web and SMB).
