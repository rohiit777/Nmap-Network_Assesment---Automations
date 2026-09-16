# Network Reconnaissance & Vulnerability Assessment Lab

A hands-on network assessment and host enumeration project conducted from an isolated Kali Linux auditing node targeting a Windows 10 workstation. 

This project demonstrates practical cybersecurity workflows: discovering active hosts, discovering open ports via stealth scans, detecting operating systems and running service versions, auditing SMB configurations using Nmap Scripting Engine (NSE), and generating automated executive HTML reports.

---

## Lab Architecture & Topology

```text
                        +-----------------------------------------------------------+
                        |               Isolated Virtual Network Subnet             |
                        |                      192.168.137.0/24                     |
                        +-----------------------------------------------------------+
                                                      |
                                    +-----------------+-----------------+
                                    |                                   |
                                    v                                   v
                        +-----------------------+           +-----------------------+
                        |      Kali Linux       |           |      Windows 10       |
                        |    (Auditing Node)    |           |    (Target System)    |
                        |   192.168.137.135     |           |   192.168.137.134     |
                        +-----------------------+           +-----------------------+
                        ---
        +------------------------------------------------------------------------------+
        |Node          | OS                |  IP Address        |     Purpose          |
        |Auditing      | SystemKali Linux  | 192.168.137.135    |   Scanner & Auditor  |
        |Target Machine|  Windows 10 Pro   |  192.168.137.134   |   Target |Workstation|
        +------------------------------------------------------------------------------+

---

```
# Step 1: Host Discovery (Is the target alive?)

## Purpose

To quickly identify whether the target machine is powered on and active on the local subnet without scanning individual ports.

```text

# Command

Bash
sudo nmap -sn 192.168.137.134

```
## Explanation
* -sn: Disables port scanning.

* Because both systems reside on the same local subnet, Nmap automatically uses Layer 2 ARP requests (-PR), which reliably detects the target and its MAC address even if Windows Defender Firewall blocks ICMP echo (ping) packets.

---

<img width="555" height="154" alt="Screenshot 2026-09-16 190000" src="https://github.com/user-attachments/assets/dda48b7d-9edf-4c5b-b360-4e49ee29f0ac" />

---

# Step 2: Port Scanning (Which doors are open?)

## Purpose

To discover active network ports and accessible services across the target endpoint.

```
Bash
sudo nmap -sS -p 1-1000 192.168.137.134

---
```
## Explanation
* -sS: Performs a TCP SYN (Stealth / Half-Open) Scan.

* Nmap transmits a TCP packet with the SYN flag set:

* If the target replies with SYN-ACK, the port is open.

* Kali immediately responds with a RST (Reset) packet instead of sending an ACK.

* Because the three-way handshake is never completed, application-layer sessions are not established, keeping the footprint minimal.

  <img width="592" height="264" alt="Screenshot 2026-09-16 191228" src="https://github.com/user-attachments/assets/a5f59b8a-cd5e-401c-bd24-9d861bfee338" />

---

# Step 3: Service & OS Detection (What is running?)

## Purpose

To identify the exact service banners, software names, and operating system release running on the host.

```
Bash
sudo nmap -sV -O -p 135,139,445 192.168.137.134

---
```
## Explanation

* -sV: Probes open ports to determine service and protocol version numbers.

* -O: Fingerprints the operating system by analyzing TCP sequence predictability, TTL values (Windows initial TTL = 128), and TCP options.

* Findings:

* 135/tcp: Microsoft Windows RPC (Remote Procedure Call)

* 139/tcp: Microsoft Windows netbios-ssn (NetBIOS Session Service)

* 445/tcp: microsoft-ds (Server Message Block - SMB)

* OS Match: Identified as Microsoft Windows 10 (Build 1709 - 22H2).

<img width="843" height="345" alt="Screenshot 2026-09-16 191744" src="https://github.com/user-attachments/assets/d2bb4375-123a-4200-814b-373762a335db" />

---

# Step 4: NSE Script Scanning (Audit SMB Configuration)

## Purpose

To assess security settings on the Server Message Block (SMB) service without performing disruptive or brute-force attacks.

```

Bash
sudo nmap -p 445 --script smb2-security-mode 192.168.137.134

---
```
## Explanation

* --script smb2-security-mode: Executes an Nmap Scripting Engine (NSE) script against port 445.

* Checks whether SMB Message Signing is required or merely enabled / supported.

* Security Significance: If SMB signing is not strictly enforced (required: false), the endpoint can be vulnerable to NTLM Relay and Man-in-the-Middle (MitM) attacks on a local network.

<img width="563" height="288" alt="Screenshot 2026-09-16 192430" src="https://github.com/user-attachments/assets/ae411aca-912d-4c06-8f49-d49ed16260d6" />

---

# Step 5: Automated HTML Report Generation

## Purpose

To export technical findings into an executive-ready, styled HTML dashboard rather than raw terminal text.

Command

---
```
Bash
sudo nmap -sV -p 135,139,445 -oX scan.xml 192.168.137.134 && xsltproc scan.xml -o report.html

---
```
## Explanation

* -oX scan.xml: Exports full scan data into structured XML format.

* xsltproc scan.xml -o report.html: Transforms the XML dataset into an interactive HTML report using standard XSL stylesheets.

* The resulting report.html can be opened directly in a browser (firefox report.html) for presentation and documentation.

<img width="801" height="272" alt="Screenshot 2026-09-16 193943" src="https://github.com/user-attachments/assets/305b4e7f-cc6c-4cac-91dc-56c693beab91" />

<img width="1883" height="913" alt="Screenshot 2026-09-16 194953" src="https://github.com/user-attachments/assets/d4ad508e-dfb6-4719-b3cc-f0ecb4afc631" />

---

# Security Recommendations

1> Enforce SMB Message Signing:

* Configure via Group Policy: Computer Configuration -> Windows Settings -> Security Settings -> Local Policies -> Security Options -> Microsoft network server: Digitally sign communications (always) -> Enabled.

2> Restrict Unnecessary Inbound Ports:

* Filter ports 135, 139, and 445 at the host firewall level so they are only accessible from authorized management workstations or administrative subnets.

3> Disable Legacy Protocols:

* Ensure NetBIOS over TCP/IP and SMBv1 are disabled across all network adapters.

---

# Step 6

## Universal Automation Tool: `autonmap.sh`

To scale the assessment across varied targets and environments without modifying source code, an automated wrapper (`autonmap.sh`) was engineered with configurable CLI flags and reporting pipelines.

### Features
- **Dynamic Target Support:** Accepts single hosts, domain names, CIDR ranges, and port specifications via standard CLI arguments.
- **Configurable Profiles:**
  - `quick`: Rapid triage scanning top 100 ports.
  - `standard`: Top 1000 ports with service banners and OS fingerprinting.
  - `full`: Complete 65,535 TCP port audit.
  - `vuln`: Automated security assessment using curated safe and vulnerability-checking NSE categories.
- **Reporting Pipeline:** Generates `.nmap`, `.xml`, and `.gnmap` files, automatically compiling structured XML into styled HTML reports via `xsltproc`.

<img width="474" height="72" alt="Screenshot 2026-09-16 213917" src="https://github.com/user-attachments/assets/2772a470-c45e-44fd-b60e-8fbe8c14320d" />
<img width="291" height="70" alt="Screenshot 2026-09-16 213927" src="https://github.com/user-attachments/assets/7db015e1-7315-4425-8502-ec5203b143e6" />

---
```
#!/usr/bin/env bash
# ==============================================================================
# Script Name: autonmap.sh
# Description: Universal Automated Network Reconnaissance & Assessment Engine
# ==============================================================================

set -e

# Require root/sudo for raw socket operations (-sS, -O, etc.)
if [ "$EUID" -ne 0 ]; then
  echo "[-] Error: Please execute this tool with root privileges (sudo)."
  exit 1
fi

# Help / Usage Menu
display_help() {
  echo "================================================================="
  echo "         AutoNmap: Universal Reconnaissance Engine               "
  echo "================================================================="
  echo "Usage: sudo ./autonmap.sh -t <TARGET> [OPTIONS]"
  echo ""
  echo "Required Argument:"
  echo "  -t <TARGET>      Target IP, Hostname, or Subnet (e.g., 192.168.1.1, scanme.nmap.org, 10.0.0.0/24)"
  echo ""
  echo "Scan Modes (Choose one, default is standard):"
  echo "  -m quick         Host discovery and top 100 ports (-T4 -F)"
  echo "  -m standard      Top 1000 ports + Service versions + OS detection"
  echo "  -m full          All 65535 ports + Service versions"
  echo "  -m vuln          Standard ports + Safe NSE scripts & Vulnerability checks (--script=vuln,safe)"
  echo ""
  echo "Additional Flags:"
  echo "  -p <PORTS>       Specify custom ports (e.g., -p 80,443,445 or -p 1-10000)"
  echo "  -Pn              Treat all hosts as online (skip initial discovery ping)"
  echo "  -h               Show this help menu"
  echo "================================================================="
  exit 0
}

# Default settings
MODE="standard"
CUSTOM_PORTS=""
TREAT_ONLINE=false
TARGET=""

# Parse command-line arguments
while [[ $# -gt 0 ]]; do
  case "$1" in
    -t)
      TARGET="$2"
      shift 2
      ;;
    -m)
      MODE="$2"
      shift 2
      ;;
    -p)
      CUSTOM_PORTS="$2"
      shift 2
      ;;
    -Pn)
      TREAT_ONLINE=true
      shift
      ;;
    -h|--help)
      display_help
      ;;
    *)
      echo "[-] Unknown option: $1"
      display_help
      ;;
  esac
done

# Validate target input
if [ -z "$TARGET" ]; then
  echo "[-] Error: Target is required."
  display_help
fi

# Setup output structure
REPORT_DIR="reports"
mkdir -p "$REPORT_DIR"

# Clean target string for safe file naming (e.g., 192.168.1.0/24 -> 192.168.1.0_24)
SAFE_TARGET_NAME=$(echo "$TARGET" | sed 's/[^a-zA-Z0-9.-]/_/g')
TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BASENAME="${REPORT_DIR}/${SAFE_TARGET_NAME}_${MODE}_${TIMESTAMP}"

# Assemble Nmap base flags
SCAN_FLAGS=("-sS")

if [ "$TREAT_ONLINE" = true ]; then
  SCAN_FLAGS+=("-Pn")
fi

# Build arguments based on chosen profile
case "$MODE" in
  quick)
    SCAN_FLAGS+=("-T4" "-F")
    ;;
  standard)
    SCAN_FLAGS+=("-T4" "-sV" "-O" "--osscan-guess")
    ;;
  full)
    SCAN_FLAGS+=("-T4" "-p-" "-sV")
    ;;
  vuln)
    SCAN_FLAGS+=("-T4" "-sV" "--script=safe,vuln")
    ;;
  *)
    echo "[-] Invalid mode: $MODE. Fallback to standard."
    SCAN_FLAGS+=("-T4" "-sV" "-O" "--osscan-guess")
    ;;
esac

# Append custom ports if provided
if [ -n "$CUSTOM_PORTS" ]; then
  SCAN_FLAGS+=("-p" "$CUSTOM_PORTS")
fi

# Add multi-format output flags (XML, standard text, and grepable)
SCAN_FLAGS+=("-oA" "$BASENAME")

echo "================================================================="
echo "[*] Target Identified: $TARGET"
echo "[*] Execution Profile: $MODE"
echo "[*] Command Executing: nmap ${SCAN_FLAGS[*]} $TARGET"
echo "================================================================="

# Execute Scan
nmap "${SCAN_FLAGS[@]}" "$TARGET"

echo "-----------------------------------------------------------------"
echo "[+] Scan completed successfully."

# Automated HTML Report Compilation via xsltproc
if command -v xsltproc >/dev/null 2>&1; then
  if [ -f "${BASENAME}.xml" ]; then
    echo "[+] Generating styled HTML dashboard: ${BASENAME}.html"
    xsltproc "${BASENAME}.xml" -o "${BASENAME}.html"
  fi
else
  echo "[!] Notice: 'xsltproc' not found. Run 'sudo apt install xsltproc' to enable HTML generation."
fi

echo "[+] Scan outputs stored at: ${BASENAME}.*"
echo "================================================================="

---

```
## RESULTS OF SCRIPT SCANS

## View Help / Options:
```
Bash
sudo ./autonmap.sh -h
---
```
<img width="817" height="359" alt="Screenshot 2026-09-16 214828" src="https://github.com/user-attachments/assets/6bb66b97-30f1-4450-89d4-0927a5cd927b" />

---

## Standard Scan (Windows Lab or authorized server):
```
Bash
sudo ./autonmap.sh -t 192.168.137.134
---
```
<img width="1020" height="512" alt="Screenshot 2026-09-16 214847" src="https://github.com/user-attachments/assets/db518f9d-01bb-4c6e-ba83-7b7f7d9be46e" />

---


### Vulnerability & Configuration Audit (Uses NSE safe & vuln scripts):
```
Bash
sudo ./autonmap.sh -t 192.168.137.134 -m vuln

```
<img width="1002" height="693" alt="Screenshot 2026-09-16 214908" src="https://github.com/user-attachments/assets/be88ba2a-eeab-47d0-ae73-5a2977b425ad" />
<img width="945" height="696" alt="Screenshot 2026-09-16 214937" src="https://github.com/user-attachments/assets/2f7a20fa-0f19-46b0-8413-69cec65c77a8" />
<img width="910" height="685" alt="Screenshot 2026-09-16 214955" src="https://github.com/user-attachments/assets/f4a3702d-3139-4f4f-afc4-1ca462adebe2" />
<img width="1048" height="684" alt="Screenshot 2026-09-16 215020" src="https://github.com/user-attachments/assets/c8d4d167-dd73-4632-946a-37636aab1e8b" />

---
## Target Specific Ports (e.g., checking only SMB/RDP):
```
Bash
sudo ./autonmap.sh -t 192.168.137.134 -p 135,139,445,3389 -m standard

---
```
<img width="1055" height="558" alt="Screenshot 2026-09-16 215052" src="https://github.com/user-attachments/assets/f88ff59d-6f79-4e22-a245-8418a681b792" />

---

## Scan an Entire Subnet:
```
Bash
sudo ./autonmap.sh -t 192.168.137.0/24 -m quick

---
```
<img width="915" height="672" alt="Screenshot 2026-09-16 215112" src="https://github.com/user-attachments/assets/af8ae885-f254-416b-b527-ce6f4eb7e899" />











