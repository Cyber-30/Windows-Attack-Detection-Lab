# 🛡️ Windows Attack Detection Lab

> **Home SOC Lab — Attack Simulation & Log Analysis**

A self-contained, isolated lab environment built to generate real attack traffic and analyze it the way a SOC L1 analyst would—using genuine Windows Event Logs and Sysmon telemetry instead of simulated or pre-built datasets.

---

## 📖 Overview

This project consists of two machines connected through a fully isolated internal network:

| Machine | Role |
|---------|------|
| **Windows 11 VM** | Monitored endpoint running Sysmon for enhanced logging |
| **Fedora Linux Host** | Attacker machine running offensive security tools natively |

Attacks are launched from Fedora against the Windows VM, producing real Windows Event Logs that are manually correlated to identify:

- Source IP addresses
- Event timestamps
- Attack techniques
- Behavioral patterns

This mirrors the workflow a SOC analyst performs when investigating security alerts.

---

# 🏗️ Lab Architecture

```text
                 Windows Attack Detection Lab

        Fedora Linux (Attacker)
             192.168.56.1
                    │
                    │
──────────────────────────────────────────────────────────
      VirtualBox Host-Only Network
          192.168.56.0/24
──────────────────────────────────────────────────────────
                    │
                    │
       Windows 11 VM (Target)
          192.168.56.101
```

### Environment

| Component | Details |
|-----------|---------|
| Hypervisor | VirtualBox |
| Network | Host-Only (Isolated) |
| Windows Logging | Sysmon (SwiftOnSecurity Configuration) |
| Native Windows Logs | Security Event Log |
| Offensive Tools | `nmap`, `hydra` |

> **Note:** No traffic leaves the isolated network. The lab has no connectivity to the internet or the host LAN.

---

# 🛠️ Tools Used

| Tool | Purpose |
|------|---------|
| VirtualBox | Hypervisor with host-only networking |
| Sysmon (SwiftOnSecurity Config) | Enhanced Windows endpoint logging |
| Nmap | Network reconnaissance / port scanning |
| Hydra | SMB brute-force credential attack |
| Windows Event Viewer | Log analysis and event correlation |

---

# 🎯 Attack Simulations

## 1️⃣ Reconnaissance — Port Scan

### Command

```bash
nmap -sT -p 1-1000 192.168.56.101
```

### Detection

- **Source:** Sysmon
- **Event ID:** 3 (Network Connection)

### Findings

- Multiple outbound connections
- Destination ports:
  - 135
  - 139
  - 445
- Same source IP
- Occurring within a 1–3 second window

This represents the classic behavior of a TCP port scan.

### MITRE ATT&CK

**T1046 – Network Service Discovery**

---

## 2️⃣ Credential Access — SMB Brute Force

### Command

```bash
hydra -l labuser -P passwords.txt smb://192.168.56.101
```

### Detection

- **Windows Security Log**
  - Event ID **4625** – Failed Logon
  - Event ID **4740** – Account Lockout

### Findings

- Cluster of failed authentication attempts
- Same source IP
- Automatic account lockout after the Windows threshold was reached

This follows the standard brute-force detection pattern commonly used in SOC environments.

### MITRE ATT&CK

**T1110 – Brute Force**

---

# 📂 Repository Structure

```text
Windows-Attack-Detection-Lab/
│
├── README.md
│
├── notes/
│   └── incident-writeup.md
│
└── screenshots/
    ├── README.md
    ├── 01-sysmon-install-confirmation.png
    ├── 02-sysmon-baseline-verification.png
    ├── 03-nmap-portscan-output.png
    ├── 04-eventid3-filtered-list.png
    ├── 05-eventid3-general-tab.png
    ├── 06-eventid3-details-friendly-view.png
    ├── 07-eventid3-details-xml-view.png
    ├── 08-smb-protocol-check.png
    ├── 09-hydra-bruteforce-run.png
    ├── 10-eventid4625-xml-detail.png
    ├── 11-eventid4740-lockout-general.png
    ├── 12-security-log-filtered-list.png
    └── description.md
```
---

# ⚙️ Configuration & Wordlist Details

### Sysmon Configuration

Based on the [SwiftOnSecurity sysmon-config](https://github.com/SwiftOnSecurity/sysmon-config) baseline. The default config excludes network connection logging for common system processes (`System`, `svchost.exe`) to reduce noise in production environments. Since this lab's target ports (135, 139, 445) are handled by exactly those processes, a scoped `NetworkConnect` include rule was added for the lab subnet:

```xml
<RuleGroup name="" groupRelation="or">
  <NetworkConnect onmatch="include">
    <SourceIp condition="is">192.168.56.1</SourceIp>
    <DestinationIp condition="is">192.168.56.101</DestinationIp>
  </NetworkConnect>
</RuleGroup>
```

### Wordlist

A small custom dictionary was used for the brute-force test, containing common weak passwords (`password`, `123456`, `admin`, `letmein`) plus the actual test account's password, to generate both failed (Event ID 4625) and — when unlocked and re-run — successful (Event ID 4624) logon telemetry for comparison.

---

# 🧠 Key Skills Demonstrated

- Isolated VM networking (Host-Only configuration & subnet verification)
- Sysmon deployment and configuration tuning
- Editing include/exclude rules within Sysmon configuration
- Windows Security Event Log analysis
- Sysmon Event Log analysis
- Manual log correlation using:
  - Timestamp
  - Source IP
  - Event ID
  - Attack behavior
- Mapping activity to the MITRE ATT&CK Framework
- Writing analyst-style incident reports

---

# 📝 Notes

This lab intentionally uses a non-default setup:

- **Fedora Linux** is used as the attacker machine instead of Kali Linux to reduce RAM usage by requiring only one VM to run at a time.
- **SMB brute-force** is used instead of RDP because Windows 11 Home Edition does not support inbound Remote Desktop hosting.
- **SMB1 was temporarily enabled** on the Windows target to provide compatibility with Hydra's SMB module, which does not support SMB2/SMB3.

> **⚠️ Lab-Only Configuration**
>
> SMB1 is deprecated and insecure. It was enabled solely for testing within this isolated lab environment and should **never** be enabled on production or internet-connected systems.

---

## 📄 Documentation

A complete analyst-style investigation, including attack timelines, event correlation, and findings, is available in: [`notes/incident-writeup.md`](notes/incident-writeup.md).
