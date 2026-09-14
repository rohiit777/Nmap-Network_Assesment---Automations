# Enterprise Network Reconnaissance, Fingerprinting & Automation Suite

An end-to-end network assessment, protocol verification, and automated vulnerability scanning project targeting a Windows 10 enterprise workstation from an isolated Kali Linux auditing node. 

This repository documents Layer 2/3/4 reconnaissance, packet-level TCP handshake analysis, Windows SMB/RPC enumeration via the Nmap Scripting Engine (NSE), firewall evasion testing, and an automated multi-format reporting pipeline via Bash and XSLT.

---

## Architecture & Lab Topology


```text
+-----------------------------------------------------------+
|               Isolated Virtual Network Subnet             |
|                       192.168.137.0/24                     |
+-----------------------------------------------------------+
                              |
            +-----------------+-----------------+
            |                                   |
            v                                   v
+-----------------------+           +-----------------------+
|      Kali Linux       |           |      Windows 10       |
|    (Auditing Node)    |           |    (Target System)    |
|   IP: 192.168.137.10  |           |   IP: 192.168.137.134|
+-----------------------+           +-----------------------+
```


| Node | Operating System | IP Address | Primary Role |
| :--- | :--- | :--- | :--- |
| Auditing Node | Kali Linux (Rolling) | `192.168.137.10` | Port Scanning, Packet Capture, NSE Auditing |
| Target System | Windows 10 Pro | `192.168.137.134` | Endpoint Target (SMB, MSRPC, NetBIOS) |
| Hypervisor Network | Host-Only / Internal | `192.168.137.0/24` | Layer 2 Broadcast Domain Isolation |

---

## Phase 1: Host Discovery & Layer 2/3 Validation

### Methodology
Standard ICMP Echo probes (`-PE`) are frequently dropped by software firewalls like Windows Defender. Because both virtual machines share a Layer 2 broadcast domain, Layer 2 ARP requests (`-PR`) provide 100% reliable discovery regardless of endpoint firewall rules.

### Execution
```bash
# Perform Layer 2 ARP discovery
sudo nmap -sn -PR 192.168.137.134

---
```
## Protocol Mechanics
ARP Request: Kali broadcasts Who has 192.168.137.101? Tell 192.168.137.134.

Target Response: The Windows 10 network interface card driver responds directly at the data link layer (192.168.137.134 is at 08:00:27:xx:xx:xx), confirming the host is online without passing traffic through the Windows TCP/IP firewall filters.

<img width="679" height="229" alt="Screenshot 2026-09-14 212044" src="https://github.com/user-attachments/assets/19ac67e1-0bff-4395-8764-766af6f64da0" />


---

# Evidence
## Phase 2: Full-Port Surface Scanning & Handshake Analysis
Methodology
Scan all 65,535 TCP ports using half-open SYN scanning (-sS) to map the entire attack surface. Simultaneously, capture network traffic via tcpdump to verify connection termination mechanics.

### Terminal 1: Run full TCP range sweep with aggressive timing
sudo nmap -sS -p- -T4 -v 192.168.137.134

<img width="795" height="709" alt="Screenshot 2026-09-14 211313" src="https://github.com/user-attachments/assets/75c94d64-279a-4fa2-a7a0-54eca24a2421" />


### Terminal 2: Sniff SYN-ACK-RST exchange on port 445
sudo tcpdump -i eth0 host 192.168.137.134 and port 445 -nn -vv







