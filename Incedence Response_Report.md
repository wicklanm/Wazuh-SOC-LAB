# SOC Lab — Incident Response Documentation

**Environment:** SOCLAB.LOCAL  
**Lab Network:** 192.168.100.0/24  
**Analyst:** Michael Wickland
**Platform:** Wazuh 4.x / Windows Server 2025 / Windows 11 Enterprise

---

## Asset Inventory

| Hostname | IP Address | Role |
| :---- | :---- | :---- |
| DC01 | 192.168.100.10 | Domain Controller / DNS |
| WIN11 | 192.168.100.20 | Domain-joined Workstation |
| KALI | 192.168.100.30 | Attacker Machine (Kali Linux) |
| WAZUH | 192.168.100.40 | SIEM (Wazuh Manager \+ Dashboard) |

---

## Master MITRE ATT\&CK Coverage Map

| Report | Technique ID | Technique Name | Tactic | Detected |
| :---- | :---- | :---- | :---- | :---- |
| IR-001 | T1046 | Network Service Discovery | Discovery | ⚠️ Gap |
| IR-002 | T1110.003 | Password Spraying | Credential Access | ✓ |
| IR-002 | T1078 | Valid Accounts | Initial Access | ✓ |
| IR-003 | T1021.001 | Remote Desktop Protocol | Lateral Movement | ✓ |
| IR-004 | T1059.001 | PowerShell | Execution | ✓ |
| IR-004 | T1027 | Obfuscated Files or Information | Defense Evasion | ✓ |
| IR-005 | T1053.005 | Scheduled Task | Persistence | ✓ |
| IR-005 | T1547.001 | Registry Run Keys | Persistence | ⚠️ Gap |
| IR-006 | T1003.001 | LSASS Memory | Credential Access | ⚠️ Gap |
| IR-006 | T1218.011 | Signed Binary Proxy Execution: Rundll32 | Defense Evasion | ⚠️ Gap |
| IR-007 | T1087.002 | Domain Account Discovery | Discovery | ✓ |
| IR-007 | T1018 | Remote System Discovery | Discovery | ✓ |
| IR-008 | T1021.006 | Windows Remote Management | Lateral Movement | ✓ |
| IR-008 | T1021.002 | SMB/Windows Admin Shares | Lateral Movement | ✓ |

---

---

# IR-001 — Network Reconnaissance (Nmap Scan)

**Report ID:** IR-001  
**Date:** Aug 12, 2026  
**Severity:** Low  
**Status:** Detection Gap — Documented

---

## 1\. Executive Summary

An attacker on the internal network performed an automated port scan against domain-joined systems to identify open services, operating system versions, and network topology. No systems were compromised during this phase, but the scan provided the attacker with a detailed blueprint of the environment used to plan all subsequent attack activity. This event was not captured by Wazuh due to Windows Firewall connection auditing not being enabled prior to the scan — a detection gap identified and documented below.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 12, 2026 (estimated) | Nmap \-sV \-O scan launched from KALI (192.168.100.30) against 192.168.100.0/24 |
| Aug 12, 2026 (estimated) | Targeted \-sC \-sV scan run against DC01 and WIN11 |
| Aug 12, 2026 (estimated) | Attacker maps open ports: 445, 3389, 53, 88, 389 on DC01 |
| Post-scan | No Wazuh alert generated — audit policy gap identified |

> **Note:** Exact timestamps unavailable. Windows Firewall connection auditing (Event ID 5156/5157) was not enabled on WIN11 or DC01 prior to the scan, resulting in no log data being generated or forwarded to Wazuh.

---

## 3\. Attack Description

The attacker ran an nmap service version and OS detection scan (`-sV -O`) against the full `192.168.100.0/24` subnet from the Kali Linux attacker machine (192.168.100.30), followed by a targeted script scan (`-sC -sV`) against DC01 and WIN11. The scan returned detailed service banners and OS fingerprinting data revealing the Active Directory environment structure, open RDP and SMB ports, and Kerberos/LDAP services — providing the attacker with the full attack surface map used in all subsequent phases.

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic |
| :---- | :---- | :---- |
| T1046 | Network Service Discovery | Discovery |

---

## 5\. Indicators of Compromise

| Type | Value |
| :---- | :---- |
| Source IP | 192.168.100.30 (Kali attacker box) |
| Target subnet | 192.168.100.0/24 |
| Target hosts | 192.168.100.10 (DC01), 192.168.100.20 (WIN11) |
| Tool | nmap |
| Expected Event IDs | Windows Firewall 5156/5157 (not generated — audit not enabled) |

---

## 6\. Detection

- **Wazuh Rule Written:** 100001 — SOC-LAB: Possible network scan detected from Kali attacker box  
- **Detection Status:** ⚠️ Rule written and loaded but did not fire — no events generated at source  
- **Root Cause:** Windows Filtering Platform connection auditing was not enabled on WIN11 or DC01 prior to scan execution. Event IDs 5156 and 5157 are only generated when this audit subcategory is active.  
- **Remediation for future detection:** Run `auditpol /set /subcategory:"Filtering Platform Connection" /success:enable /failure:enable` on all endpoints before attack simulation

---

## 7\. Containment Steps

- Block 192.168.100.30 at host firewall on all internal machines  
- Isolate the source machine from the network segment  
- Preserve any available NetFlow or switch port data covering the scan window

---

## 8\. Eradication & Recovery

- Confirm no exploitation followed the reconnaissance phase  
- Enable firewall connection auditing on all hosts immediately  
- Review IDS/IPS logs for scan signatures if available at the network level

---

## 9\. Lessons Learned & Recommendations

- **Enable Windows Firewall auditing as a baseline on all endpoints** before any attack simulation — this is a prerequisite, not an optional step  
- Deploy a network-level IDS (e.g., Suricata integrated with Wazuh) to catch port scan patterns independent of host-based logging  
- Reduce DC01's exposed attack surface by disabling unused services and restricting which ports are accessible from workstation network segments  
- **Detection gap closed by:** enabling `auditpol /set /subcategory:"Filtering Platform Connection"` and re-running the nmap scenario in a future lab iteration

---

---

# IR-002 — Password Spraying

**Report ID:** IR-002  
**Date:** Aug 8, 2026
**Severity:** High  
**Status:** Resolved

---

## 1\. Executive Summary

An attacker conducted a low-and-slow password spraying attack against domain accounts over SMB, attempting credentials against multiple accounts from the Kali attacker machine. The attack successfully authenticated against the ITAdmin domain account, providing the attacker with privileged credentials used in all subsequent attack phases. Failed logon events were captured by Wazuh and confirmed the spray pattern.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 8, 2026 @ 12:33:51.086 | First failed logon attempt — ITAdmin from 192.168.100.30 (Wazuh Rule 60122\) |
| Aug 8, 2026 @ 12:33:57.880 | Second failed logon attempt — ITAdmin from 192.168.100.30 (Wazuh Rule 60122\) |
| Aug 8, 2026 (estimated) | Successful authentication discovered for John.Smith (J@cker123\#$) |

---

## 3\. Attack Description

The attacker used NetExec (nxc) from Kali (192.168.100.30) to attempt SMB authentication against WIN11 (192.168.100.20) using a list of known domain usernames and a small password wordlist. The spray was designed to stay below account lockout thresholds by attempting only a small number of passwords per account. Failed logon attempts were recorded against the ITAdmin account. The credential `John.Smith:J@cker123#$` was successfully identified and used to establish the initial RDP foothold in IR-003.

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic |
| :---- | :---- | :---- |
| T1110.003 | Password Spraying | Credential Access |
| T1078 | Valid Accounts | Initial Access |

---

## 5\. Indicators of Compromise

| Type | Value |
| :---- | :---- |
| Source IP | 192.168.100.30 (Kali) |
| Target IP | 192.168.100.20 (WIN11) |
| Accounts targeted | ITAdmin, John.Smith (and others in spray list) |
| Account compromised | John.Smith (credential: J@cker123\#$) |
| Protocol | SMB (TCP 445\) |
| Tool | NetExec (nxc) |
| Event IDs | Security 4625 (Logon Type 3\) |
| Wazuh Rule | 60122 — Logon Failure \- Unknown user or bad password |

---

## 6\. Detection

- **Wazuh Rule Fired:** 60122 — Logon Failure \- Unknown user or bad password (Level 5\)  
- **Custom Rule Written:** 100002 — Password spray pattern (frequency-based, same IP / different accounts)  
- **Event ID:** 4625 (Failed Logon), Logon Type 3 (Network)  
- **Key Detection Fields:**  
  - `data.win.eventdata.ipAddress` \= 192.168.100.30  
  - `data.win.eventdata.targetUserName` \= ITAdmin / John.Smith  
  - `data.win.eventdata.logonType` \= 3

---

## 7\. Containment Steps

- Immediately lock all accounts targeted in the spray  
- Block 192.168.100.30 at the host and network firewall level  
- Reset passwords for all accounts included in the spray list  
- Review all authentication events from 192.168.100.30 across the full environment

---

## 8\. Eradication & Recovery

- Force password reset for John.Smith and all targeted accounts  
- Re-enable accounts only after password reset and MFA enrollment  
- Audit all activity performed under John.Smith credentials following the compromise timestamp  
- Confirm no persistence mechanisms were established using spray-obtained credentials

---

## 9\. Lessons Learned & Recommendations

- Implement an account lockout policy (recommended: 5 attempts / 15-minute lockout) to limit spray effectiveness — no lockout policy was configured in this environment  
- Enable Multi-Factor Authentication for all domain accounts, particularly privileged accounts  
- Implement Microsoft Entra ID Password Protection to block common and weak passwords at the domain level  
- Alert on 3+ failed logons from the same source IP within 60 seconds targeting different accounts — this is the core spray signature  
- Restrict SMB access from non-management workstations to reduce the spray surface

---

---

# IR-003 — Unauthorized RDP Access (Initial Foothold)

**Report ID:** IR-003  
**Date:** Aug 12, 2026
**Severity:** High  
**Status:** Resolved

---

## 1\. Executive Summary

Using credentials obtained during the password spraying attack (IR-002), an attacker established an interactive Remote Desktop Protocol session on WIN11 from the Kali attacker machine. This provided full GUI-level access to the workstation and served as the primary foothold from which all subsequent attack activity — including encoded PowerShell execution, scheduled task persistence, domain enumeration, and lateral movement — was launched.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 12, 2026 @ 17:02:28.358 | Successful RDP logon — John.Smith from 192.168.100.30 to WIN11 (Wazuh Rule 92653\) |
| Aug 12, 2026 @ 17:07:46 | Encoded PowerShell execution begins (IR-004) |
| Aug 12, 2026 @ 18:01:46 | Lateral movement to DC01 initiated from RDP session (IR-008) |
| Aug 12, 2026 @ 18:37:09 | Domain enumeration commands run from RDP session (IR-007) |

---

## 3\. Attack Description

Using `xfreerdp` from the Kali machine, the attacker connected to WIN11 over TCP port 3389 using compromised credentials (John.Smith / J@cker123\#$) obtained during the password spray. The RDP session provided a full interactive Windows desktop on WIN11 from which PowerShell was launched as Administrator. All subsequent attack techniques in this kill chain were executed from within this RDP session.

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic |
| :---- | :---- | :---- |
| T1021.001 | Remote Desktop Protocol | Lateral Movement |
| T1078 | Valid Accounts | Persistence / Defense Evasion |

---

## 5\. Indicators of Compromise

| Type | Value |
| :---- | :---- |
| Source IP | 192.168.100.30 (Kali) |
| Target IP | 192.168.100.20 (WIN11) |
| Target Port | TCP 3389 (RDP) |
| Account used | SOCLAB\\John.Smith |
| Credential | J@cker123\#$ (obtained via IR-002) |
| Tool | xfreerdp (Kali Linux) |
| Event IDs | Security 4624 (Logon Type 10 — RemoteInteractive) |
| Wazuh Rule | 92653 — User SOCLAB\\John.Smith logged using Remote Desktop Connection |

---

## 6\. Detection

- **Wazuh Rule Fired:** 92653 — User SOCLAB\\John.Smith logged using RDP from IP 192.168.100.30 (Level 3\)  
- **Custom Rule Written:** 100003 — SOC-LAB: Successful RDP login from Kali attacker box (Level 10\)  
- **Event ID:** 4624, Logon Type 10 (RemoteInteractive)  
- **Key Detection Fields:**  
  - `data.win.eventdata.logonType` \= 10  
  - `data.win.eventdata.ipAddress` \= 192.168.100.30  
  - `data.win.eventdata.targetUserName` \= John.Smith

---

## 7\. Containment Steps

- Terminate the active RDP session immediately via Task Manager → Users → Disconnect  
- Disable John.Smith's account in Active Directory  
- Block TCP 3389 inbound from 192.168.100.30 at the host firewall  
- Review all Sysmon Event ID 1 entries on WIN11 from the session window to enumerate attacker actions

---

## 8\. Eradication & Recovery

- Reset John.Smith's password and require MFA enrollment before re-enabling  
- Audit all changes made during the RDP session using Sysmon logs  
- Remove any persistence mechanisms created during the session (see IR-005)  
- Confirm no additional accounts were created or modified during attacker access

---

## 9\. Lessons Learned & Recommendations

- Restrict RDP access to known management IP addresses only via Windows Firewall inbound rules — no workstation should accept RDP from any arbitrary source  
- Require MFA for all RDP connections using a solution such as Duo or Microsoft Authenticator  
- Implement a jump server / bastion host architecture — administrative RDP should funnel through a single hardened access point, not directly to workstations  
- Consider disabling RDP entirely on standard workstations and enabling it only on demand via a PAM solution  
- The built-in Wazuh rule 92653 caught this independently of our custom rule — confirming layered detection is working

---

---

# IR-004 — Encoded PowerShell Execution

**Report ID:** IR-004  
**Date:** Aug 12, 2026 
**Severity:** High  
**Status:** Resolved

---

## 1\. Executive Summary

From the established RDP session on WIN11, the attacker executed obfuscated PowerShell commands using base64 encoding to conceal the true intent of the commands from basic string-matching detection. The encoded command was identified as a domain user enumeration query (`net user /domain`), confirming the attacker was conducting Active Directory reconnaissance under the cover of obfuscation. Wazuh's custom detection rule fired successfully on this activity.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 12, 2026 @ 17:02:28 | RDP foothold established on WIN11 (IR-003) |
| Aug 12, 2026 @ 17:07:46.150 | Encoded PowerShell command executed on WIN11 (Wazuh Rule 100004 fired) |
| Aug 12, 2026 @ 17:07:46.150 | Sysmon Event ID 1 captures full command line including base64 payload |

---

## 3\. Attack Description

From an elevated PowerShell session on WIN11 via the RDP connection, the attacker generated a base64-encoded PowerShell command encoding the string `net user /domain` and executed it using the `-enc` flag to bypass simple keyword-based detection. The parent process was also PowerShell, indicating a PowerShell-within-PowerShell execution chain. The `-WindowStyle Hidden` variant was also used in additional executions to suppress visible PowerShell windows. The decoded payload confirmed the attacker was enumerating domain user accounts as part of pre-lateral movement reconnaissance.

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic |
| :---- | :---- | :---- |
| T1059.001 | PowerShell | Execution |
| T1027 | Obfuscated Files or Information | Defense Evasion |

---

## 5\. Indicators of Compromise

| Type | Value |
| :---- | :---- |
| Target host | WIN11 (192.168.100.20) |
| Process | powershell.exe |
| Command line | `powershell.exe -enc bgBlAHQAIAB1AHMAZQByACAALwBkAG8AbQBhAGkAbgA=` |
| Decoded payload | `net user /domain` |
| Parent image | `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` |
| Execution flags | `-enc`, `-EncodedCommand`, `-WindowStyle Hidden` |
| Event IDs | Sysmon 1 (Process Creation) |
| Wazuh Rule | 100004 — SOC-LAB: Encoded PowerShell command detected |

---

## 6\. Detection

- **Wazuh Rule Fired:** 100004 — SOC-LAB: Encoded PowerShell command detected — possible obfuscated execution (Level 12\)  
- **Event ID:** Sysmon 1 (Process Creation)  
- **Detection Method:** PCRE2 regex match on `commandLine` field for `-enc` / `-EncodedCommand`  
- **Key Detection Fields:**  
  - `data.win.eventdata.image` \= powershell.exe  
  - `data.win.eventdata.commandLine` contains `-enc`  
  - `data.win.eventdata.parentImage` \= powershell.exe

---

## 7\. Containment Steps

- Kill all attacker-spawned powershell.exe processes on WIN11  
- Terminate the RDP session the commands were executed from  
- Preserve Sysmon Event ID 1 logs from the full session window  
- Decode and review all captured `-enc` payloads to understand full scope of encoded commands executed

---

## 8\. Eradication & Recovery

- Audit PowerShell Script Block logs for decoded command content (if Script Block Logging was enabled)  
- Review all processes spawned by PowerShell during the attack window for secondary payloads  
- Check for files dropped or registry changes made by the encoded commands  
- Confirm no scheduled tasks or services were created via encoded commands

---

## 9\. Lessons Learned & Recommendations

- **Enable PowerShell Script Block Logging** via Group Policy immediately — this logs the decoded command content rather than just the encoded blob, dramatically improving visibility: `Computer Config → Admin Templates → Windows Components → PowerShell → Turn on Script Block Logging = Enabled`  
- Enable PowerShell Transcription logging as a secondary source for full session capture  
- Implement Constrained Language Mode for non-administrative users to prevent advanced PowerShell usage  
- The parent/child PowerShell-spawning-PowerShell pattern is a high-fidelity indicator — consider adding a specific rule for this execution chain

---

---

# IR-005 — Persistence via Scheduled Task & Registry Run Key

**Report ID:** IR-005  
**Date:** Aug 12, 2026
**Severity:** High  
**Status:** Partially Detected — Registry Persistence Gap Documented

---

## 1\. Executive Summary

The attacker established persistence on WIN11 using two techniques: a series of scheduled tasks created via both manual and Atomic Red Team simulation methods, and a registry Run key entry designed to execute at user logon. The scheduled task activity was successfully detected by both built-in Wazuh rules and custom rule 100005\. The registry Run key modification was not captured in Wazuh logs, representing a detection gap documented below for remediation.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 12, 2026 @ 18:06:57.625 | First scheduled task created via cmd.exe — SCHTASKS /Create (Rule 92032\) |
| Aug 12, 2026 @ 18:07:02.402 | schtasks.exe /Create /F /TN "ATOMIC..." (Atomic Red Team T1053.005) |
| Aug 12, 2026 @ 18:07:06.368 | schtasks /Create /TN "CompMgmtBy..." |
| Aug 12, 2026 @ 18:07:08.012 | schtasks /Create /TN "EventViewerB..." |
| Aug 12, 2026 @ 18:18:08.026 | Additional T1053.005 Atomic Red Team task creation |
| Aug 12, 2026 @ 18:20:09.012 | schtasks /create /tn "T1053\_005\_OnS..." |
| Aug 12, 2026 @ 18:22:13.683 | schtasks.exe /Create /F /TN "ATOMIC..." |
| Aug 12, 2026 @ 18:22:18.553 | schtasks /Run /TN "EventViewerBypa..." (Discovery activity — Rule 92034\) |
| Aug 12, 2026 (estimated) | Registry Run key written to HKCU...\\CurrentVersion\\Run — NOT captured |

---

## 3\. Attack Description

From the WIN11 RDP session, the attacker created multiple scheduled tasks using `schtasks.exe` via both cmd.exe and PowerShell, including manually-created tasks and automated simulations via Atomic Red Team's T1053.005 test suite. Tasks were created with names designed to blend with legitimate Windows task names (EventViewerBypass, CompMgmtBy) and configured to run at logon or on-demand. A registry Run key entry was also added under `HKCU\Software\Microsoft\Windows\CurrentVersion\Run` to provide a secondary persistence mechanism; however, this event was not captured by Sysmon or forwarded to Wazuh.

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic | Detected |
| :---- | :---- | :---- | :---- |
| T1053.005 | Scheduled Task/Job | Persistence | ✓ |
| T1547.001 | Registry Run Keys / Startup Folder | Persistence | ⚠️ Gap |

---

## 5\. Indicators of Compromise

| Type | Value |
| :---- | :---- |
| Target host | WIN11 (192.168.100.20) |
| Process | schtasks.exe, cmd.exe |
| Parent image | C:\\Windows\\System32\\cmd.exe |
| Task names | ATOMIC, T1053\_005\_OnS, CompMgmtBy, EventViewerBypass |
| Registry key (undetected) | HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Run |
| Registry value (undetected) | UpdateHelper \= C:\\Windows\\System32\\calc.exe |
| Event IDs | Sysmon 1, Security 4698, Sysmon 13 (not captured) |
| Wazuh Rules | 92032, 92034, 100005 |

---

## 6\. Detection

**Scheduled Tasks — Detected:**

- **Wazuh Rules Fired:** 92032 (Suspicious Windows cmd shell execution, Level 3), 92034 (Discovery activity spawned via cmd shell execution, Level 3\)  
- **Custom Rule Written:** 100005 — SOC-LAB: Scheduled task created (Level 10\)  
- **Event IDs:** Sysmon 1 (schtasks.exe process creation), Security 4698 (task created)

**Registry Run Key — NOT Detected:**

- **Custom Rule Written:** 100006 — SOC-LAB: Registry Run key modified (Level 10\)  
- **Detection Status:** ⚠️ Sysmon Event ID 13 was not generated or not forwarded for HKCU Run key modifications  
- **Root Cause Under Investigation:** SwiftOnSecurity Sysmon config may be filtering HKCU registry events; alternatively the specific subkey path may not be covered by the current Sysmon configuration

---

## 7\. Containment Steps

- Delete all attacker-created scheduled tasks: `schtasks /query /fo LIST` to enumerate, then `schtasks /delete /tn "[name]" /f` for each  
- Remove the registry Run key entry: `Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "UpdateHelper"`  
- Audit all scheduled tasks against a known-good baseline  
- Search for any additional Run/RunOnce keys modified during the attack window

---

## 8\. Eradication & Recovery

- Perform a full scheduled task audit on WIN11 and DC01  
- Review full Run/RunOnce/RunServices registry hives for unexpected entries  
- Consider reimaging WIN11 if full scope of persistence cannot be confirmed  
- Re-enable endpoint monitoring after confirming clean state

---

## 9\. Lessons Learned & Recommendations

- Restrict `schtasks.exe` execution to administrators via AppLocker or WDAC policy  
- Implement application allow-listing to prevent unauthorized executables running even if persistence is established  
- **Registry detection gap:** Review and extend the SwiftOnSecurity Sysmon configuration to ensure HKCU Run key writes generate Event ID 13 — test with a known write and confirm the event appears in Wazuh before considering it covered  
- Regularly audit scheduled tasks and startup registry keys against a documented baseline  
- Monitor for scheduled task names that mimic legitimate Windows task naming conventions — the `EventViewerBypass` and `CompMgmtBy` names are designed specifically to blend in

---

---

# IR-006 — Credential Dumping (LSASS Memory Access)

**Report ID:** IR-006  
**Date:** Aug 12, 2026
**Severity:** Critical  
**Status:** Attack Not Completed — Detection Gap Documented

---

## 1\. Executive Summary

A credential dumping attack targeting the LSASS process was planned as part of this lab exercise using the native Windows `comsvcs.dll` method via `rundll32.exe`. Due to access and tooling constraints encountered during Phase 8, this attack was not successfully executed in the lab environment. The detection rule (100007) was written and validated for syntax but could not be confirmed as firing against real attack telemetry. This report documents the technique, the intended detection mechanism, the gap, and the remediation path for a future lab iteration.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 12, 2026 | LSASS dump attempted but not completed due to tooling constraints |
| Aug 12, 2026 | Wazuh rule 100007 written and loaded successfully |
| Aug 12, 2026 | No Sysmon Event ID 10 (ProcessAccess targeting lsass.exe) generated |

---

## 3\. Attack Description (Intended)

The planned attack would have used `rundll32.exe` with the built-in Windows DLL `comsvcs.dll` to create a full memory dump of the LSASS process from an elevated session on WIN11. This technique avoids third-party tool downloads by leveraging a Microsoft-signed binary, making it resistant to basic AV signature detection. The resulting dump file would contain NTLM hashes and Kerberos tickets for all users with active sessions, enabling pass-the-hash lateral movement.

**Intended command:**

rundll32.exe C:\\Windows\\System32\\comsvcs.dll MiniDump \<lsass\_pid\> C:\\Windows\\Temp\\lsass.dmp full

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic |
| :---- | :---- | :---- |
| T1003.001 | OS Credential Dumping: LSASS Memory | Credential Access |
| T1218.011 | Signed Binary Proxy Execution: Rundll32 | Defense Evasion |

---

## 5\. Indicators of Compromise (Expected)

| Type | Value |
| :---- | :---- |
| Process | rundll32.exe |
| Command | rundll32 comsvcs.dll MiniDump \[PID\] lsass.dmp full |
| Target process | lsass.exe |
| Expected dump path | C:\\Windows\\Temp\\lsass.dmp |
| Expected Event IDs | Sysmon 10 (ProcessAccess), Sysmon 11 (FileCreate) |
| Expected Wazuh Rule | 100007 — SOC-LAB: LSASS process accessed (Level 15\) |

---

## 6\. Detection

- **Custom Rule Written:** 100007 — SOC-LAB: LSASS process accessed by non-system process (Level 15 Critical)  
- **Detection Status:** ⚠️ Rule loaded and syntactically valid — not validated against live telemetry  
- **Expected Event ID:** Sysmon 10 (ProcessAccess) where targetImage \= lsass.exe  
- **Expected secondary signal:** Sysmon 11 (FileCreate) for lsass.dmp creation

---

## 7\. Containment Steps (Applicable in Real Incident)

- Immediately isolate the affected host from the network  
- Delete the dump file: `del C:\Windows\Temp\lsass.dmp`  
- Assume ALL domain credentials cached on the affected host are compromised  
- Force immediate password reset for every account with a session on the host during the compromise window

---

## 8\. Eradication & Recovery

- Reset passwords for ALL affected domain accounts including service accounts  
- Perform double krbtgt password reset (with 30-minute gap) to invalidate Kerberos tickets domain-wide  
- Review all authentication events following the dump timestamp for pass-the-hash activity  
- Reimage the affected workstation

---

## 9\. Lessons Learned & Recommendations

- **Enable Windows Credential Guard** (requires Hyper-V / VBS) as the primary mitigation — prevents LSASS memory access by isolating credential material in a virtualization-based security enclave  
- **Enable LSA Protection (RunAsPPL):** Set `HKLM\SYSTEM\CurrentControlSet\Control\Lsa\RunAsPPL = 1` (requires Secure Boot) to make LSASS a Protected Process, blocking most user-mode dumping techniques  
- Restrict `rundll32.exe` execution via AppLocker where possible  
- Never log privileged accounts (domain admins) into standard workstations — use a tiered administration model  
- **Future lab action:** Re-attempt this scenario in a dedicated iteration with Defender exclusions pre-configured and an elevated local session, and validate that Sysmon Event ID 10 fires and rule 100007 generates a Wazuh alert

---

---

# IR-007 — Domain Enumeration / Active Directory Reconnaissance

**Report ID:** IR-007  
**Date:** Aug 12, 2026
**Severity:** Medium  
**Status:** Resolved

---

## 1\. Executive Summary

Following the establishment of an RDP foothold on WIN11, the attacker conducted systematic enumeration of the SOCLAB.LOCAL Active Directory domain using both native Windows tools and encoded PowerShell commands. The attacker specifically queried domain user accounts and targeted the Administrator account directly, gathering the information required to select and authenticate to the Domain Controller in the subsequent lateral movement phase.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 12, 2026 @ 17:07:46 | Encoded `net user /domain` command executed via PowerShell \-enc (IR-004) |
| Aug 12, 2026 @ 18:09:30.034 | Domain enumeration via net.exe — Rule 100008 fires (WIN11) |
| Aug 12, 2026 @ 18:09:30.066 | Second net.exe domain query — Rule 100008 fires (WIN11) |
| Aug 12, 2026 @ 18:09:31.102 | Additional domain enumeration — Rule 100008 fires (WIN11) |
| Aug 12, 2026 @ 18:09:34.416 | Additional domain enumeration — Rule 100008 fires (WIN11) |
| Aug 12, 2026 @ 18:37:04.021 | net.exe domain query — Rule 100008 fires (WIN11) |
| Aug 12, 2026 @ 18:37:04.034 | net.exe domain query — Rule 100008 fires (WIN11) |
| Aug 12, 2026 @ 18:37:09.434 | `net user /domain` via PowerShell parent — Rule 100008 fires (WIN11) |
| Aug 12, 2026 @ 18:37:36.131 | `net user administrator /domain` via cmd.exe — Rule 100008 fires (WIN11) |

---

## 3\. Attack Description

From the WIN11 RDP session, the attacker used `net.exe` commands via both PowerShell and cmd.exe to enumerate the SOCLAB.LOCAL domain. The attacker's first enumeration attempt used base64-encoded PowerShell (IR-004) to conceal the `net user /domain` command. Subsequent queries were run in plain text, including a targeted query for the Administrator account specifically (`net user administrator /domain`). Two separate enumeration waves occurred — one around 18:09 and a second around 18:37 — indicating deliberate, repeated reconnaissance. This data directly enabled the lateral movement to DC01 documented in IR-008.

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic |
| :---- | :---- | :---- |
| T1087.002 | Account Discovery: Domain Account | Discovery |
| T1018 | Remote System Discovery | Discovery |
| T1069.002 | Permission Groups Discovery: Domain Groups | Discovery |

---

## 5\. Indicators of Compromise

| Type | Value |
| :---- | :---- |
| Target host | WIN11 (192.168.100.20) |
| Process | net.exe (`C:\WINDOWS\System32\net.exe`) |
| Command lines | `net user /domain`, `net user administrator /domain` |
| Parent images | `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe`, `C:\Windows\System32\cmd.exe` |
| Event IDs | Sysmon 1 (Process Creation) |
| Wazuh Rule | 100008 — SOC-LAB: Domain enumeration via net.exe (Level 7\) |
| Alert count | 10 separate rule 100008 alerts on Aug 12 |

---

## 6\. Detection

- **Wazuh Rule Fired:** 100008 — SOC-LAB: Domain enumeration via net.exe — attacker may be conducting reconnaissance (Level 7\)  
- **Alert Count:** 10 alerts fired across two enumeration waves  
- **Event ID:** Sysmon 1 (Process Creation)  
- **Key Detection Fields:**  
  - `data.win.eventdata.image` \= net.exe  
  - `data.win.eventdata.commandLine` contains `/domain`  
  - `data.win.eventdata.parentImage` \= powershell.exe / cmd.exe

---

## 7\. Containment Steps

- Terminate the attacker's active session on WIN11  
- Review what information was successfully retrieved — specifically which accounts and their privilege levels  
- Assess whether Administrator account details were obtained and act accordingly  
- Block 192.168.100.30 at the firewall

---

## 8\. Eradication & Recovery

- No direct system changes to revert from enumeration activity alone  
- Focus eradication on the accounts and systems identified as targets  
- Rotate credentials for any accounts confirmed as enumerated, particularly Administrator and ITAdmin

---

## 9\. Lessons Learned & Recommendations

- Implement tiered administration — standard domain user accounts should not be able to query sensitive domain group membership or Administrator account details  
- Deploy **Microsoft Defender for Identity (MDI)** which detects LDAP-based enumeration techniques that bypass net.exe command logging entirely  
- Restrict `net.exe` execution for non-administrative users via AppLocker policy  
- The two enumeration waves (18:09 and 18:37) suggest the attacker ran enumeration both before and after attempting lateral movement — this temporal pattern is worth building a correlation rule around

---

---

# IR-008 — Lateral Movement to Domain Controller (DC01)

**Report ID:** IR-008  
**Date:** Aug 12, 2026
**Severity:** Critical  
**Status:** Resolved

---

## 1\. Executive Summary

Using domain administrator credentials, the attacker successfully moved laterally from the compromised workstation (WIN11) to the Domain Controller (DC01) via PowerShell Remoting. Access to a Domain Controller represents full domain compromise — an attacker with this level of access has complete control over all systems, users, and policies in SOCLAB.LOCAL. The lateral movement was detected via a Sysmon process creation event on DC01 showing `wsmprovhost.exe` (the WinRM host process) as a parent — direct evidence of remotely-executed commands on the Domain Controller. The WinRM traffic from WIN11 to DC01 was also independently detected by Wazuh's built-in rule 92110\.

---

## 2\. Timeline

| Time | Event |
| :---- | :---- |
| Aug 12, 2026 @ 18:04:43.902 | WinRM traffic detected from 192.168.100.20 to 192.168.100.10 — Rule 92110 fires (WIN11) |
| Aug 12, 2026 @ 18:04:43.918 | Second WinRM connection event — Rule 92110 fires (WIN11) |
| Aug 12, 2026 @ 18:04:43.934 | Third WinRM connection event — Rule 92110 fires (WIN11) |
| Aug 12, 2026 @ 18:01:46.611 | net.exe execution on DC01 via wsmprovhost.exe parent — Rule 100008 fires (DC01) |

---

## 3\. Attack Description

From the WIN11 RDP session, the attacker used PowerShell's `Enter-PSSession` cmdlet with ITAdmin credentials to establish an interactive PowerShell Remoting session directly on DC01. The connection used WinRM over TCP port 5985\. Once connected, the attacker ran domain enumeration commands including `net user /domain` directly on the Domain Controller — confirmed by `wsmprovhost.exe` appearing as the parent process in Sysmon Event ID 1 on DC01. Built-in Wazuh rules detected the WinRM traffic from WIN11's perspective, while the custom rule 100008 caught the resulting command execution on DC01 itself — demonstrating detection at both the network and endpoint layer.

---

## 4\. MITRE ATT\&CK Mapping

| Technique ID | Name | Tactic |
| :---- | :---- | :---- |
| T1021.006 | Remote Services: Windows Remote Management | Lateral Movement |
| T1021.002 | Remote Services: SMB/Windows Admin Shares | Lateral Movement |
| T1078.002 | Valid Accounts: Domain Accounts | Privilege Escalation |

---

## 5\. Indicators of Compromise

| Type | Value |
| :---- | :---- |
| Source host | WIN11 (192.168.100.20) |
| Target host | DC01 (192.168.100.10) |
| Account used | SOCLAB\\ITAdmin |
| Protocol | WinRM (TCP 5985\) |
| Parent process on DC01 | `C:\Windows\System32\wsmprovhost.exe` |
| Child process on DC01 | net.exe (`net user /domain`) |
| Event IDs | Sysmon 1 (process creation, DC01), Security 4624 Logon Type 3, Sysmon 3 (network) |
| Wazuh Rules | 100008 (DC01 — custom), 92110 (WIN11 — built-in WinRM detection) |

---

## 6\. Detection

**Primary Detection — Command execution on DC01:**

- **Wazuh Rule Fired:** 100008 — SOC-LAB: Domain enumeration via net.exe (Level 7\) on agent DC01  
- **Event ID:** Sysmon 1 (Process Creation)  
- **Key indicator:** `data.win.eventdata.parentImage` \= `wsmprovhost.exe` on DC01  
- **Significance:** wsmprovhost.exe as parent confirms the command was executed remotely, not locally

**Secondary Detection — WinRM network traffic:**

- **Wazuh Rule Fired:** 92110 — Detected WinRM activity from 192.168.100.20 to 192.168.100.10 (Level 4\) on agent WIN11  
- **Fired 3 times** between 18:04:43.902 and 18:04:43.934

**Custom Rule Written:** 100009 — SOC-LAB: Network logon to DC01 from WIN11 (Level 12\) — written for 4624 Logon Type 3 detection; logon event not captured in this iteration but rule is in place for future detection.

---

## 7\. Containment Steps

- Immediately isolate DC01 from the network if active attacker session is confirmed  
- Terminate all active WinRM and RDP sessions on DC01  
- Reset ITAdmin password immediately — this is the account used for lateral movement  
- Reset the krbtgt account password **twice** with a 30-minute gap between resets — this invalidates all existing Kerberos tickets domain-wide  
- Isolate WIN11 as the source of the lateral movement  
- Revoke all active sessions for any accounts that authenticated to DC01 during the compromise window

---

## 8\. Eradication & Recovery

- Audit all Active Directory changes made during the attacker's DC01 session window — specifically check for new accounts, group membership changes, and GPO modifications  
- Review all domain admin logon events for the 72 hours prior to detection for signs of earlier undetected access  
- Rotate all privileged account credentials across the domain  
- Perform a full Active Directory audit against a known-good state  
- Consider a full rebuild of DC01 if the full scope of access cannot be confirmed with certainty

---

## 9\. Lessons Learned & Recommendations

- **Implement tiered administration immediately** — domain admin accounts (ITAdmin) should never be used for interactive sessions on standard workstations. A compromised workstation should never hold credentials capable of reaching a Domain Controller  
- **Segment the network** — workstations (192.168.100.20) should not be able to initiate WinRM or SMB connections directly to Domain Controllers (192.168.100.10). A firewall rule blocking this single traffic path would have stopped this lateral movement entirely  
- **Deploy a Privileged Access Workstation (PAW)** — all domain admin activity should originate from a dedicated, hardened workstation with no internet access and no email client  
- **The wsmprovhost.exe parent process signature is a high-fidelity lateral movement indicator** — consider building a dedicated Wazuh rule specifically for any process with wsmprovhost.exe as a parent on DC01, at a higher severity level (12+)  
- The layered detection in this scenario (custom rule 100008 on DC01 \+ built-in rule 92110 on WIN11) demonstrates the value of monitoring at multiple points in the kill chain — neither detection alone tells the full story, but together they confirm both the network connection and the resulting execution

---

## Document Control

| Version | Date | Change |
| :---- | :---- | :---- |
| 1.0 | Aug 12, 2026 | Initial documentation — all Phase 8 scenarios |

**Detection Summary:**

| Report | Detected | Rule Type |
| :---- | :---- | :---- |
| IR-001 Nmap Scan | ⚠️ Gap — audit policy not pre-enabled | Custom 100001 (unvalidated) |
| IR-002 Password Spray | ✓ | Built-in 60122 \+ Custom 100002 |
| IR-003 RDP Login | ✓ | Built-in 92653 \+ Custom 100003 |
| IR-004 Encoded PowerShell | ✓ | Custom 100004 |
| IR-005 Scheduled Task | ✓ | Built-in 92032/92034 \+ Custom 100005 |
| IR-005b Registry Run Key | ⚠️ Gap — Sysmon config gap | Custom 100006 (unvalidated) |
| IR-006 LSASS Dump | ⚠️ Gap — attack not completed | Custom 100007 (unvalidated) |
| IR-007 Domain Enumeration | ✓ | Custom 100008 |
| IR-008 Lateral Movement | ✓ | Built-in 92110 \+ Custom 100008 on DC01 |

