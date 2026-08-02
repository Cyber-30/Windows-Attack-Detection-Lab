# Screenshot Evidence Walkthrough

This document walks through each screenshot in this folder in the order the lab was performed, explaining what it shows and why it matters.

---

## Setup Verification

### 01 — Sysmon Install Confirmation
`sysmon64.exe -accepteula -i sysmonconfig-export.xml` run in an elevated PowerShell session on the Windows 11 VM. Output confirms Sysmon v15.21 loaded the configuration file (schema version 4.50, Sysmon schema 4.91), validated it successfully, and started both the `SysmonDrv` driver and the `Sysmon64` service. This confirms logging was active before any attack was launched.

### 02 — Sysmon Baseline Verification
Event Viewer's Sysmon Operational log showing normal background activity (Event ID 1 – Process Creation, Event ID 22 – DNS query, Event ID 8 – CreateRemoteThread, Event ID 5 – Process termination) prior to any attack traffic. This baseline confirms Sysmon was actively logging system behavior and gave a "before" reference point to distinguish attack-generated events from routine noise.

---

## Attack 1: Reconnaissance — Port Scan

### 03 — Nmap Port Scan Output
`nmap -sT -p 1-1000 192.168.56.101` run from Fedora. Result: three open ports — `135/tcp` (msrpc), `139/tcp` (netbios-ssn), `445/tcp` (microsoft-ds) — with 997 ports reported closed (connection refused). Confirms the target was reachable on the isolated network and identifies the exact ports probed.

### 04 — Sysmon Event ID 3 Filtered List
Event Viewer filtered to Sysmon Event ID 3 (Network Connection). Three matching events appear, timestamped within one second of each other (10:35:36–10:35:37 PM), all tagged `Network connection detected (rule: NetworkConnect)`. This tight time clustering across multiple ports is the core evidence of scanning behavior.

### 05 — Event ID 3 General Tab
Detail view of one Event ID 3 entry. Key fields: `SourceIp: 192.168.56.1` (Fedora), `DestinationIp: 192.168.56.101` (Windows target), `DestinationPort: 445`, `Protocol: tcp`, `Initiated: false` (confirming this was an inbound connection to the target, not outbound from it).

### 06 — Event ID 3 Details (Friendly View)
Same event shown in Event Viewer's structured Friendly View, listing all fields (`ProcessGuid`, `SourcePort: 57756`, `DestinationHostname: DESKTOP-DIQN3N8`, `DestinationPortName: microsoft-ds`) in a cleanly parsed format — useful for confirming exact field names when writing detection logic later.

### 07 — Event ID 3 Details (XML View)
The same event in raw XML form, showing the full underlying event schema (`Provider Name: Microsoft-Windows-Sysmon`, `EventID: 3`, `TimeCreated SystemTime: 2026-08-02T05:35:37`). Included to demonstrate the ability to read raw log data directly, not just the GUI-rendered summary — a skill relevant to SIEM/log pipeline work.

---

## Attack 2: Credential Access — SMB Brute-Force

### 08 — SMB Protocol Check
`nmap -p 445 --script smb-protocols 192.168.56.101` confirming the target supports SMBv1 (`NT LM 0.12`) alongside SMB 2.0.2 through 3.1.1. SMB1 was deliberately, temporarily enabled on the target for this exercise since hydra's SMB module does not support SMB2/3 — this check confirmed the protocol downgrade was in effect before launching the attack.

### 09 — Hydra Brute-Force Run
`hydra -l labuser -P password.txt smb://192.168.56.101` executed from Fedora. Hydra ran 2,491 login attempts (`l:1/p:2491`) and completed in roughly one second. Note: this run reported `0 valid password found` because the wordlist used did not contain the account's actual password — the goal of this run was to generate failed-logon telemetry, not to demonstrate credential discovery.

### 10 — Event ID 4625 XML Detail
Raw XML detail of a single Event ID 4625 (Failed Logon) entry from the Windows Security log. Key fields: `TargetUserName: labuser`, `LogonType: 3` (network logon), `AuthenticationPackageName: NTLM`, `WorkstationName: \\192.168.56.1`, `IpAddress: 192.168.56.1`. This confirms the failed attempt was correctly attributed to the Fedora attacker machine.

### 11 — Event ID 4740 Lockout (General Tab)
Event ID 4740 confirming `labuser` was automatically locked out after exceeding Windows' default failed-logon threshold. `Caller Computer Name: \\192.168.56.1` ties the lockout directly back to the source of the brute-force attempts.

### 12 — Security Log Filtered List
Event Viewer filtered to Event IDs `4625, 2491, 4740` in the Security log. Shows a dense cluster of `Audit Failure` (4625) entries all within the same second (11:10:54 PM), followed by a single `Audit Success` / `4740` entry — visually demonstrating the failed-logon-cluster-then-lockout pattern described in the incident write-up.
