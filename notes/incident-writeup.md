# 📝 Incident Write-Up: Windows Attack Detection Lab

| **Analyst**     | Cyber-30                                                   |
| --------------- | ---------------------------------------------------------- |
| **Environment** | Isolated Home Lab (Fedora Attacker + Windows 10 VM Target) |
| **Date**        | 02 Aug 2026                                                |

---

# 🌐 Environment Summary

The lab consists of two machines connected through a fully isolated VirtualBox Host-Only network.

| System       | Details                                                                                                                                    |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Attacker** | Fedora Linux (`192.168.56.1`) running **nmap** and **hydra** natively *(no dedicated Kali VM used to reduce RAM overhead on an 8 GB host)* |
| **Target**   | Windows 10 VM (`192.168.56.101`) running **Sysmon** with a modified **SwiftOnSecurity** configuration                                      |

No component of this lab has access to the real LAN or the internet. All traffic analyzed below originates and terminates entirely within the isolated **192.168.56.0/24** subnet.

> **Configuration Note**
>
> The default SwiftOnSecurity Sysmon configuration excludes network connection logging for common system processes (`System`, `svchost.exe`) to reduce noise in production environments.
>
> Since the target ports used in this lab (**135**, **139**, **445**) are handled by these exact processes, an explicit **include** rule was added and scoped only to the lab subnet. This allowed the required traffic to be captured while preserving the production-oriented filtering behavior everywhere else.

---

# 🔍 Attack 1 — Reconnaissance (Port Scan)

## Action Taken

A TCP connect scan was executed from Fedora against the first 1000 TCP ports on the Windows target.

```bash
nmap -sT -p 1-1000 192.168.56.101
```

---

## Result

Three open ports were identified.

| Port    | Service      |
| ------- | ------------ |
| **135** | msrpc        |
| **139** | netbios-ssn  |
| **445** | microsoft-ds |

---

## Detection

**Log Source**

* Sysmon Operational Log
* **Event ID 3 — Network Connection**

After filtering Event Viewer to **Event ID 3**, three connection events appeared with the following shared characteristics:

| Field                 | Value                                                                      |
| --------------------- | -------------------------------------------------------------------------- |
| **Source IP**         | `192.168.56.1` (Fedora)                                                    |
| **Destination IP**    | `192.168.56.101` (Windows Target)                                          |
| **Destination Ports** | 135, 139, 445                                                              |
| **UTC Time**          | All three events clustered within a **[FILL IN — e.g. "2-second"]** window |

---

## Analyst Notes

A single source IP connecting to multiple distinct ports within a short time window is a textbook reconnaissance / port-scanning signature.

No individual connection is inherently malicious—SMB and RPC traffic occur legitimately during normal Windows operation—but the **behavioral pattern** (rapid, sequential, multi-port probing from one host) distinguishes scanning activity from routine network traffic.

This represents the type of low-severity but high-value signal an L1 SOC analyst would document and monitor for follow-on activity rather than escalate independently.

### MITRE ATT&CK

**T1046 – Network Service Discovery**

---

# 🔐 Attack 2 — Credential Access (SMB Brute Force)

## Action Taken

A dictionary-based brute-force attack was executed against a local Windows account (`labuser`) over SMB.

```bash
hydra -l labuser -P passwords.txt smb://192.168.56.101
```

### Technical Note

Hydra's SMB module only supports the legacy **SMB1** protocol.

Since Windows 10 defaults to SMB2/3, SMB1 was temporarily enabled on the target VM:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName SMB1Protocol
```

This was performed solely to allow the attack to complete within the isolated lab.

> **⚠️ Lab-Only Configuration**
>
> SMB1 is deprecated and insecure (associated with EternalBlue/WannaCry) and was enabled only because the VM is completely isolated from production networks and the internet.
>
> This configuration should **never** be used on a production or internet-facing system.

---

## Result

The attack completed after processing the supplied wordlist.

Partway through execution, the Windows account was **automatically locked out** by the default local account lockout policy after reaching the configured threshold of failed authentication attempts.

As a result, the remaining password attempts failed immediately against a locked account rather than performing normal credential validation.

---

## Detection

**Log Source**

Windows Security Log

### Event ID 4625 — Failed Logon

A cluster of failed authentication events appeared, each containing:

* **Source Network Address:** `192.168.56.1`
* **Account Name:** `labuser`

### Event ID 4740 — Account Lockout

A single account lockout event followed the Event ID 4625 cluster, confirming that the account reached the configured failure threshold.

---

## Analyst Notes

This is a canonical brute-force detection pattern:

* Multiple failed logons
* Same source IP
* Same target account
* Automatic account lockout

Production SIEM platforms such as **Splunk**, **Microsoft Sentinel**, and **Wazuh** commonly implement detection rules around this exact **4625 → 4740** sequence.

Although account lockout is primarily a Windows security control, it also serves as a valuable secondary detection indicator by confirming both:

* an attack occurred, and
* the defensive control functioned as expected.

### MITRE ATT&CK

**T1110 – Brute Force**

---

# 📌 Overall Conclusion

Both exercises demonstrate the same core analyst skill: recognizing attacker behavior from raw log patterns rather than relying on a single alert.

The port scan illustrates how individually legitimate network connections become suspicious when viewed collectively as a behavioral pattern.

The brute-force exercise demonstrates how a standard Windows security mechanism (account lockout) can itself become a reliable detection signal.

Together, these attacks—**Reconnaissance** followed by **Credential Access**—mirror the early stages of a real intrusion attempt (**MITRE ATT&CK: Discovery → Credential Access**).

The workflow used throughout this investigation—

> isolate event type → filter by Event ID → correlate source IP and timestamps → map observed behavior to MITRE ATT&CK

—is representative of the triage methodology commonly performed by L1 SOC analysts.

---

# 📎 Evidence

Supporting screenshots are available in the **`/screenshots`** directory.

| Screenshot                     | Description                                   |
| ------------------------------ | --------------------------------------------- |
| `01-nmap-scan-output.png`      | Scan results from Fedora                      |
| `02-eventid3-filtered-log.png` | Filtered Sysmon Event ID 3 view               |
| `03-eventid3-detail-view.png`  | Expanded Event ID 3 showing correlated fields |
| `04-hydra-bruteforce-run.png`  | Hydra brute-force execution                   |
| `05-eventid4625-cluster.png`   | Failed logon cluster                          |
| `06-eventid4740-lockout.png`   | Account lockout event                         |
