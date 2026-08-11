# Phase 9 – Threat Hunting

## Overview

Threat hunting is the process of proactively searching through logs and telemetry to find evidence of the attacks you ran in Phase 8 — before a rule fires, using raw data. Every search below corresponds directly to a Phase 8 scenario. The goal is to find the raw signal manually first, then document the exact query/field values you used — you'll reuse these almost verbatim when writing detection rules in Phase 10.

---

## Before You Start

**1. Open the Wazuh Dashboard** from your host machine browser:
```
https://<your-wazuh-host-only-IP>
```

**2. Navigate to the right view:**
Left sidebar → **Threat Hunting** (some versions call this **Security Events**)

**3. Set your time range** to cover when you ran Phase 8 — click the clock/calendar icon top-right → set to **Last 24 hours** or a custom range that covers your attack window. This is the most common reason hunts come back empty — time range is too narrow.

**4. Confirm agents are still active** before hunting:
Left sidebar → **Agents** → confirm DC01 and WIN11 both show **Active**. If either shows Disconnected, go fix that first or your hunts will return partial/empty results.

---

## How to Search in Wazuh

The search bar at the top of Threat Hunting accepts KQL (Kibana Query Language). Basic syntax:
```
field.name: value                    # exact match
field.name: *partial*                # wildcard match
field.name: value1 AND field.name2: value2   # combine conditions
field.name: value1 OR field.name: value2     # either condition
```

After running a search, click any event row to expand it and see all raw fields — this is how you discover the exact field names to use in your queries and in Phase 10's detection rules.

---

## Hunt 1 — Nmap Scan (Network Reconnaissance)

**What to look for:** a burst of network connection events from a single source IP hitting many ports in a short window.

**Search:**
```
agent.name: WIN11 AND data.win.system.eventID: 3
```
Sysmon Event ID 3 = network connection. Filter results to the timeframe of your nmap scan and look for a high volume of entries all sharing the same `data.win.eventdata.sourceIp` (192.168.100.30 — your Kali box).

<img width="1256" height="720" alt="Screenshot 2026-08-10 192057" src="https://github.com/user-attachments/assets/2f22824d-fc37-4269-a4c7-cd59c6832c47" />

**Refine to confirm Kali as source:**
```
data.win.eventdata.sourceIp: 192.168.100.30 AND data.win.system.eventID: 3
```

**What a positive hunt looks like:**
- Dozens or hundreds of Event ID 3 entries in a short window
- All from `sourceIp: 192.168.100.30`
- Hitting many different `destinationPort` values on WIN11

**Fields to document for Phase 10:**
- `data.win.system.eventID`
- `data.win.eventdata.sourceIp`
- `data.win.eventdata.destinationPort`

**MITRE:** T1046 — Network Service Discovery

---

## Hunt 2 — Password Spray (Failed Logins)

**What to look for:** multiple failed login attempts spread across different usernames in a short time window — the breadth-across-accounts pattern is what distinguishes a spray from a single-account brute force.

**Search:**
```
data.win.system.eventID: 4625
```
Windows Security Event ID 4625 = failed logon.

**Refine to show only the spray window:**
```
data.win.system.eventID: 4625 AND data.win.eventdata.ipAddress: 192.168.100.30
```

**Look at the results and confirm:**
- Multiple different `data.win.eventdata.targetUserName` values (John.Smith, Jane.Doe, Helpdesk, etc.)
- All from the same `data.win.eventdata.ipAddress` (192.168.100.30)
- All within a short time window
- `data.win.eventdata.logonType` = `3` (network logon)

**Also search for account lockouts if your domain policy triggered them:**
```
data.win.system.eventID: 4740
```
Event ID 4740 = account locked out. If this fires, it means the spray triggered the lockout threshold — a strong secondary detection signal.

**Fields to document for Phase 10:**
- `data.win.system.eventID` (4625)
- `data.win.eventdata.targetUserName`
- `data.win.eventdata.ipAddress`
- `data.win.eventdata.logonType`

**MITRE:** T1110.003 — Password Spraying

---

## Hunt 3 — Successful RDP Login (Initial Access)

**What to look for:** a successful logon from Kali's IP, Logon Type 10 (RemoteInteractive = RDP).

**Search:**
```
data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 10
```
Windows Security Event ID 4624 = successful logon. Logon Type 10 = RemoteInteractive (RDP specifically).

**Refine to confirm source:**
```
data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 10 AND data.win.eventdata.ipAddress: 192.168.100.30
```

**Expand an event and note:**
- `data.win.eventdata.targetUserName` — who logged in
- `data.win.eventdata.ipAddress` — 192.168.100.30 (Kali)
- `data.win.eventdata.logonType` — 10 (RDP)
- Timestamp — correlate this with your Phase 8 timeline

**Fields to document for Phase 10:**
- `data.win.system.eventID` (4624)
- `data.win.eventdata.logonType` (10)
- `data.win.eventdata.ipAddress`

**MITRE:** T1021.001 — Remote Desktop Protocol

---

## Hunt 4 — Encoded PowerShell

**What to look for:** PowerShell processes launched with `-enc` or `-EncodedCommand` in the command line — a classic obfuscation technique.

**Search:**
```
data.win.system.eventID: 1 AND data.win.eventdata.commandLine: *-enc*
```
Sysmon Event ID 1 = process creation. The `*-enc*` wildcard catches both `-enc` and `-EncodedCommand`.

**Also search for the hidden window flag used in Scenario 4:**
```
data.win.system.eventID: 1 AND data.win.eventdata.commandLine: *-WindowStyle Hidden*
```

**Expand an event and note:**
- `data.win.eventdata.image` — should show `powershell.exe`
- `data.win.eventdata.commandLine` — the full command including the base64 blob
- `data.win.eventdata.parentImage` — what process launched PowerShell (useful for parent/child chain analysis)
- `data.win.eventdata.user` — which account ran it

**Fields to document for Phase 10:**
- `data.win.system.eventID` (1)
- `data.win.eventdata.image`
- `data.win.eventdata.commandLine` (contains `-enc`)

**MITRE:** T1059.001 — PowerShell

---

## Hunt 5 — Scheduled Task Persistence

**What to look for:** schtasks.exe execution creating a new task, and the corresponding Windows Security audit event.

**Search for the process creation:**
```
data.win.system.eventID: 1 AND data.win.eventdata.image: *schtasks*
```

**Search for the audit event:**
```
data.win.system.eventID: 4698
```
Windows Security Event ID 4698 = a scheduled task was created.

**Expand the 4698 event and look for:**
- `data.win.eventdata.taskName` — WindowsUpdateHelper (or whatever you named it)
- `data.win.eventdata.taskContent` — the full XML definition of the task
- `data.win.eventdata.subjectUserName` — which account created it

**Also search for the registry run key write:**
```
data.win.system.eventID: 13 AND data.win.eventdata.targetObject: *CurrentVersion\\Run*
```
Sysmon Event ID 13 = RegistryEvent (value set).

**Expand and note:**
- `data.win.eventdata.targetObject` — the full registry path
- `data.win.eventdata.details` — the value written (calc.exe in your case)
- `data.win.eventdata.image` — what process wrote the key (powershell.exe)

**Fields to document for Phase 10:**
- `data.win.system.eventID` (1, 4698, 13)
- `data.win.eventdata.image` (schtasks.exe)
- `data.win.eventdata.targetObject` (Run key path)

**MITRE:** T1053.005 (Scheduled Task), T1547.001 (Registry Run Key)

---

## Hunt 6 — LSASS Access / Credential Dumping

**What to look for:** a process opening a handle to lsass.exe — the definitive signal for credential dumping attempts regardless of what tool was used.

**Search:**
```
data.win.system.eventID: 10 AND data.win.eventdata.targetImage: *lsass*
```
Sysmon Event ID 10 = ProcessAccess (a process opened a handle to another process).

**This is one of the highest-value detections in the entire lab.** Expand the event and note:
- `data.win.eventdata.sourceImage` — what process accessed LSASS (rundll32.exe for the comsvcs method, or taskmgr.exe for the Task Manager method)
- `data.win.eventdata.targetImage` — lsass.exe
- `data.win.eventdata.grantedAccess` — the access rights requested (0x1FFFFF or 0x1010 are common dumping signatures)
- `data.win.eventdata.sourceProcessId` — the PID of the accessing process

**Also search for the dump file creation:**
```
data.win.system.eventID: 11 AND data.win.eventdata.targetFilename: *lsass*
```
Sysmon Event ID 11 = FileCreate. This catches the `.dmp` file being written to disk.

**Fields to document for Phase 10:**
- `data.win.system.eventID` (10)
- `data.win.eventdata.targetImage` (lsass.exe)
- `data.win.eventdata.sourceImage`
- `data.win.eventdata.grantedAccess`

**MITRE:** T1003.001 — LSASS Memory

---

## Hunt 7 — Domain Enumeration

**What to look for:** net.exe commands querying domain users, groups, and computers in a short window.

**Search:**
```
data.win.system.eventID: 1 AND data.win.eventdata.image: *net.exe*
```

**Refine to catch domain-specific queries:**
```
data.win.system.eventID: 1 AND data.win.eventdata.commandLine: *net user /domain*
```

**Also search for the directory service access events:**
```
data.win.system.eventID: 4661
```
Windows Security Event ID 4661 = a handle to an object was requested (fires when AD objects are queried).

**Look at the burst pattern** — multiple net.exe executions in quick succession is the hunting signal, not any single event in isolation.

**Fields to document for Phase 10:**
- `data.win.system.eventID` (1)
- `data.win.eventdata.image` (net.exe)
- `data.win.eventdata.commandLine`

**MITRE:** T1087.002 — Domain Account Discovery

---

## Hunt 8 — Lateral Movement (WIN11 → DC01)

**What to look for:** a successful network logon on DC01 originating from WIN11's IP (192.168.100.20), using explicit credentials.

**Search on DC01's agent specifically:**
```
agent.name: DC01 AND data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 3
```
Logon Type 3 = network logon (covers PSRemoting, net use, WMI-based movement).

**Also search for explicit credential use:**
```
agent.name: DC01 AND data.win.system.eventID: 4648
```
Windows Security Event ID 4648 = logon using explicit credentials — fires when a process uses credentials other than the currently logged-on user's.

**For PSRemoting specifically, also search:**
```
agent.name: DC01 AND data.win.system.eventID: 1 AND data.win.eventdata.parentImage: *wsmprovhost*
```
`wsmprovhost.exe` is the WinRM host process — any child processes it spawns on DC01 are commands run via PowerShell Remoting from WIN11.

**Expand events and note:**
- `data.win.eventdata.ipAddress` — should show 192.168.100.20 (WIN11)
- `data.win.eventdata.targetUserName` — ITAdmin
- `data.win.eventdata.logonType` — 3

**Fields to document for Phase 10:**
- `data.win.system.eventID` (4624, 4648)
- `data.win.eventdata.logonType` (3)
- `data.win.eventdata.ipAddress`
- `agent.name` (DC01)

**MITRE:** T1021.006 (PowerShell Remoting), T1021.002 (SMB Shares)

---

## Hunt Summary Table

Use this to track what you found — fill in the "Found?" column as you complete each hunt. This becomes the basis of your Phase 10 detection rules.

| Hunt | Event IDs | Key Field | MITRE | Found? |
|---|---|---|---|---|
| Nmap scan | Sysmon 3 | sourceIp: 192.168.100.30 | T1046 | |
| Password spray | Security 4625 | Multiple targetUserName, same IP | T1110.003 | |
| RDP login | Security 4624 | logonType: 10 | T1021.001 | |
| Encoded PowerShell | Sysmon 1 | commandLine: *-enc* | T1059.001 | |
| Scheduled task | Sysmon 1, Security 4698 | image: *schtasks*, taskName | T1053.005 | |
| Registry run key | Sysmon 13 | targetObject: *CurrentVersion\Run* | T1547.001 | |
| LSASS access | Sysmon 10 | targetImage: *lsass* | T1003.001 | |
| Domain enumeration | Sysmon 1 | image: *net.exe*, commandLine: */domain* | T1087.002 | |
| Lateral movement | Security 4624, 4648 | logonType: 3, agent: DC01 | T1021.006 | |

---

## If a Hunt Returns No Results

Work through this checklist before assuming the attack didn't generate logs:

1. **Check the time range** — widen it to Last 7 days as a test
2. **Confirm the agent is Active** — Agents → check WIN11 and DC01 status
3. **Confirm Sysmon is still running** on the target:
```powershell
Get-Service Sysmon64
```
4. **Confirm the Sysmon localfile block is still in ossec.conf:**
```powershell
Select-String -Path "C:\Program Files (x86)\ossec-agent\ossec.conf" -Pattern "Sysmon"
```
5. **Trigger a fresh test event** and watch for it in real time — go to Discover in Wazuh, search `agent.name: WIN11`, set time to Last 15 minutes, then open Notepad on WIN11 and refresh — a Sysmon Event ID 1 for `notepad.exe` should appear within 60 seconds. If it doesn't, the pipeline is broken somewhere between Sysmon → agent → manager.
6. **Check the agent log on WIN11 for errors:**
```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```
