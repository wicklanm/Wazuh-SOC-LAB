# Enterprise SOC Detection Lab

**A full-cycle enterprise SOC lab demonstrating attack simulation, threat hunting, detection engineering, and incident response across an Active Directory environment — built on VirtualBox with Wazuh SIEM.**

---

## Lab Overview

This lab simulates a small enterprise network containing a Windows Server 2025 Domain Controller, a Windows 11 Enterprise workstation, a Kali Linux attacker machine, and a Wazuh SIEM stack — all isolated on a private internal network. The lab was built from scratch, including Active Directory configuration, domain user provisioning, Sysmon deployment, Wazuh agent enrollment, and custom detection rule authoring.

The full kill chain was executed from the attacker machine through to domain controller compromise, with every technique mapped to MITRE ATT&CK, detected in Wazuh, and documented in formal incident response reports.

---

## Lab Architecture

```
┌─────────────────────────────────────────────────┐
│              SOC-LAB (VirtualBox Internal Net)  │
│                  192.168.100.0/24               │
│                                                 │
│  ┌──────────────┐        ┌──────────────────┐   │
│  │     DC01     │        │      WIN11       │   │
│  │ 192.168.100.10│◄──────►│  192.168.100.20  │   │
│  │ Windows      │        │  Windows 11      │   │
│  │ Server 2025  │        │  Enterprise      │   │
│  │ AD DS / DNS  │        │  Wazuh Agent     │   │
│  │ Wazuh Agent  │        │  Sysmon          │   │
│  │ Sysmon       │        └──────────────────┘   │
│  └──────────────┘                 │             │
│          │                        │             │
│          ▼                        ▼             │
│  ┌──────────────┐        ┌──────────────────┐   │
│  │    WAZUH     │        │      KALI        │   │
│  │ 192.168.100.40│       │  192.168.100.30  │   │
│  │ Ubuntu 24.04 │        │  Kali Linux      │   │
│  │ Wazuh Manager│        │  Attacker Box    │   │
│  │ Wazuh Indexer│        │  nmap / nxc /    │   │
│  │ Wazuh        │        │  Impacket / ART  │   │
│  │ Dashboard    │        └──────────────────┘   │
│  └──────────────┘                               │
└─────────────────────────────────────────────────┘
```

**Domain:** SOCLAB.LOCAL  
**Domain Controller:** DC01 (192.168.100.10)  
**Workstation:** WIN11 (192.168.100.20)  
**Attacker:** KALI (192.168.100.30)  
**SIEM:** WAZUH (192.168.100.40)  

---

## Lab Goal

To build a fully functional enterprise SOC lab that demonstrates the complete security operations workflow:

1. **Build** — Deploy and configure an Active Directory environment with realistic user accounts, domain-joined workstations, Sysmon endpoint telemetry, and Wazuh SIEM
2. **Attack** — Execute a realistic attack kill chain from initial reconnaissance through to domain controller compromise using industry-standard attacker tooling
3. **Detect** — Hunt for attack artifacts in raw Wazuh telemetry and author custom detection rules mapped to MITRE ATT&CK techniques
4. **Respond** — Produce industry-standard incident response reports for each attack scenario including timeline, IOCs, containment steps, and hardening recommendations

---

## Phases Completed

| Phase | Description | Status |
|---|---|---|
| 1 | Enterprise network design and VM provisioning | ✅ |
| 2 | Windows Server 2025 — Active Directory, DNS, domain users | ✅ |
| 3 | Windows 11 Enterprise — domain join, workstation setup | ✅ |
| 4 | Wazuh SIEM — manager, indexer, dashboard install | ✅ |
| 5 | Sysmon — SwiftOnSecurity config on DC01 and WIN11 | ✅ |
| 6 | Wazuh agents — enrollment and Sysmon log forwarding | ✅ |
| 7 | Kali Linux — attacker tooling setup | ✅ |
| 8 | Attack scenarios — full kill chain execution | ✅ |
| 9 | Threat hunting — manual detection in Wazuh | ✅ |
| 10 | Detection engineering — custom Wazuh XML rules | ✅ |
| 11 | Incident response — formal IR reports for all scenarios | ✅ |
| 12 | Portfolio — GitHub documentation | ✅ |

---

## Attack Kill Chain

The following attack scenarios were executed from the Kali Linux attacker box against the SOCLAB.LOCAL environment:

| # | Scenario | Tool | MITRE Technique | Detected |
|---|---|---|---|---|
| 1 | Network reconnaissance | nmap | T1046 | ⚠️ Gap documented |
| 2 | Password spraying | NetExec (nxc) | T1110.003 | ✅ |
| 3 | Unauthorized RDP access | xfreerdp | T1021.001 | ✅ |
| 4 | Encoded PowerShell execution | PowerShell -enc | T1059.001, T1027 | ✅ |
| 5 | Scheduled task persistence | schtasks.exe | T1053.005 | ✅ |
| 5b | Registry Run key persistence | Set-ItemProperty | T1547.001 | ⚠️ Gap documented |
| 6 | LSASS credential dumping | comsvcs.dll / Mimikatz | T1003.001 | ⚠️ Gap documented |
| 7 | Domain enumeration | net.exe / PowerShell | T1087.002, T1018 | ✅ |
| 8 | Lateral movement to DC01 | Enter-PSSession | T1021.006 | ✅ |

---

## Custom Detection Rules

Nine custom Wazuh detection rules were authored and validated against live attack telemetry:

| Rule ID | Description | Severity | MITRE |
|---|---|---|---|
| 100001 | Network scan from attacker IP | 7 | T1046 |
| 100002 | Password spray — same IP, multiple accounts | 10 | T1110.003 |
| 100003 | Successful RDP login from attacker IP | 10 | T1021.001 |
| 100004 | Encoded PowerShell (-enc / -EncodedCommand) | 12 | T1059.001 |
| 100005 | Scheduled task created | 10 | T1053.005 |
| 100006 | Registry Run key modified | 10 | T1547.001 |
| 100007 | LSASS process accessed | 15 | T1003.001 |
| 100008 | Domain enumeration via net.exe | 7 | T1087.002 |
| 100009 | Network logon to DC01 from WIN11 | 12 | T1021.002 |

Rules are available in [`/detections/local_rules.xml`](./detections/local_rules.xml).

---

## Tools Used

**Infrastructure:**
- Oracle VirtualBox 7.x
- Windows Server 2025 Evaluation
- Windows 11 Enterprise Evaluation
- Ubuntu Server 24.04 LTS
- Kali Linux

**Detection & Monitoring:**
- Wazuh 4.x (Manager, Indexer, Dashboard)
- Microsoft Sysmon with SwiftOnSecurity configuration
- Windows Security Event logging (Audit Policy)

**Attacker Tooling:**
- nmap — network reconnaissance
- NetExec (nxc) — SMB authentication and password spraying
- xfreerdp — Remote Desktop Protocol client
- Impacket suite (psexec.py, secretsdump.py, wmiexec.py)
- Atomic Red Team — MITRE ATT&CK technique simulation
- PowerShell (native) — encoded command execution, scheduled tasks, domain enumeration

**Documentation & Frameworks:**
- MITRE ATT&CK Framework
- Sigma rule standard (detection rule authoring reference)
- KQL (Kibana Query Language) for Wazuh threat hunting

---

## Skills Demonstrated

**Security Operations:**
- SIEM deployment, configuration, and agent management
- Log source identification and custom log forwarding configuration
- Threat hunting using KQL across Windows Security and Sysmon event logs
- Alert triage and investigation methodology
- Detection gap identification and root cause analysis

**Detection Engineering:**
- Custom Wazuh XML rule authoring (100001–100009)
- PCRE2 regex matching for command-line detection
- Frequency-based correlation rules (password spray detection)
- Parent/child process relationship detection (wsmprovhost lateral movement)
- MITRE ATT&CK technique mapping to detection logic

**Incident Response:**
- Industry-standard IR report authoring (executive summary through lessons learned)
- Timeline construction from SIEM event timestamps
- IOC identification and documentation
- Containment, eradication, and recovery planning
- Detection gap documentation with remediation paths

**Active Directory & Windows:**
- Active Directory Domain Services deployment and configuration
- Organizational Unit and user account management via PowerShell
- Domain join, Group Policy awareness, and audit policy configuration
- Windows Firewall rule management
- WinRM and PowerShell Remoting configuration

**Networking & Virtualization:**
- VirtualBox multi-VM network design (Internal Network, NAT, Host-only)
- Static IP assignment via netplan (Ubuntu) and network adapter configuration
- Network segmentation concepts applied to lab isolation

**Attacker Techniques (Offensive Awareness):**
- Reconnaissance — port scanning and service enumeration
- Credential access — password spraying, LSASS memory access
- Execution — encoded PowerShell obfuscation
- Persistence — scheduled tasks and registry Run keys
- Discovery — Active Directory enumeration via native Windows tools
- Lateral movement — PowerShell Remoting and WinRM

---

## Key Findings & Detection Highlights

**Strongest Detections:**

- **Rule 100004 (Level 12)** — Encoded PowerShell detected via PCRE2 regex matching `-enc` / `-EncodedCommand` in Sysmon Event ID 1 command line. Successfully decoded payload confirmed as `net user /domain` — demonstrating that obfuscation did not prevent detection.

- **Rule 100008 on DC01 (Level 7)** — Domain enumeration commands detected on the Domain Controller with `wsmprovhost.exe` as the parent process — direct evidence of remote command execution via PowerShell Remoting. Corroborated by built-in Wazuh rule 92110 detecting WinRM traffic from WIN11 to DC01.

- **Built-in rule 92653** — Wazuh's native RDP logon detection independently caught the unauthorized RDP session from 192.168.100.30 without requiring a custom rule — confirming layered detection coverage.

**Detection Gaps Identified & Documented:**

- **Nmap scan (T1046):** Windows Firewall connection auditing not pre-enabled; Event ID 5157 not generated. Remediation: `auditpol /set /subcategory:"Filtering Platform Connection" /success:enable /failure:enable`

- **Registry Run key (T1547.001):** Sysmon Event ID 13 not captured for HKCU Run key writes. Likely filtered by SwiftOnSecurity config. Remediation: Review and extend Sysmon configuration to cover HKCU registry paths.

- **LSASS dumping (T1003.001):** Attack not completed due to tooling constraints in this lab iteration. Rule 100007 written and validated syntactically. Planned for re-execution in a follow-up iteration with pre-configured Defender exclusions.

---

## Incident Response Reports

Full incident response documentation for all eight attack scenarios is available in [`/incident-reports/`](./incident-reports/), including:

- Executive summary (non-technical)
- Timestamped timeline (pulled directly from Wazuh event data)
- MITRE ATT&CK technique mapping
- Indicators of Compromise (IOCs)
- Detection methodology (Wazuh rule IDs and event IDs)
- Containment and eradication steps
- Recovery recommendations
- Lessons learned and hardening recommendations

---

## Repository Structure

```
/
├── README.md                          ← This file
├── architecture/
│   └── network-diagram.png            ← Lab network topology
├── detections/
│   └── local_rules.xml                ← All custom Wazuh rules (100001-100009)
├── incident-reports/
│   ├── IR-001-nmap-scan.md
│   ├── IR-002-password-spray.md
│   ├── IR-003-rdp-access.md
│   ├── IR-004-encoded-powershell.md
│   ├── IR-005-persistence.md
│   ├── IR-006-lsass-dump.md
│   ├── IR-007-domain-enumeration.md
│   └── IR-008-lateral-movement.md
├── screenshots/
│   └── [Wazuh dashboard alert screenshots]
└── docs/
    ├── phase7-kali-setup.md
    ├── phase8-attack-scenarios-rdp.md
    ├── phase9-threat-hunting.md
    └── phase10-detection-engineering.md
```

---

## Disclaimer

All activity in this lab was performed in an isolated VirtualBox environment on privately-owned hardware. No production systems, real user accounts, or unauthorized networks were involved. All attack techniques were executed solely for educational purposes within a controlled lab environment.
