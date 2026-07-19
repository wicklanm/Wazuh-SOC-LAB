# Wazuh-SOC-LAB
I built an enterprise SOC lab demonstrating attack, detection, investigation, and incident response, portfolio-ready.



# Enterprise SOC Detection Lab — Detailed Build Guide

**Goal:** Build an enterprise SOC lab demonstrating attack, detection, investigation, and incident response, portfolio-ready.

**Before you start — host requirements:**
- Host machine: 32GB RAM minimum (16GB is workable but tight — you'll be running 4 VMs), 200GB+ free disk (SSD strongly preferred), CPU with virtualization extensions (VT-x/AMD-V) enabled in BIOS.
- Software: VirtualBox 7.x + VirtualBox Extension Pack (needed for USB/RDP features), 7-Zip for extracting ISOs if needed.
- Download all ISOs *before* starting Phase 1 — this saves you hours of waiting mid-build.

| VM | Purpose | RAM | vCPU | Disk |
|---|---|---|---|---|
| DC01 | Windows Server 2025 (AD DC) | 4GB | 2 | 60GB |
| WIN11 | Windows 11 Enterprise workstation | 4GB | 2 | 60GB |
| WAZUH | Ubuntu Server 24.04 (SIEM) | 8GB (4GB min, 8GB recommended for indexer) | 4 | 50GB |
| KALI | Kali Linux (attacker) | 4GB | 2 | 40GB |

---

## Phase 1 – Build the Enterprise Network

1. **Install VirtualBox + Extension Pack** on your host. Reboot after install (driver install requires it).
2. **Download ISOs first:**
   - Windows Server 2025 Evaluation (180-day) — Microsoft Evaluation Center.
   - Windows 11 Enterprise Evaluation (90-day) — Microsoft Evaluation Center (requires a free work/school-type registration).
   - Ubuntu Server 24.04 LTS — ubuntu.com/download/server.
   - Kali Linux — use the **pre-built VirtualBox OVA** from kali.org (faster than installing from ISO).
3. **Create the internal network:** You don't pre-create "SOC-LAB" anywhere globally — it's created implicitly the first time you assign a VM's network adapter to it. In VirtualBox: VM → Settings → Network → Adapter 1 → Attached to: **Internal Network** → Name: `SOC-LAB` (type it exactly the same on every VM — it's case-sensitive and must match character-for-character).
   <img width="1213" height="811" alt="Screenshot 2026-07-19 134718" src="https://github.com/user-attachments/assets/a98346a3-2a38-4a72-9863-5b8bf6cd4e1b" />

4. **Important gotcha:** Internal Network has *no DHCP and no internet access by default*. You have two options:
   - **Option A (recommended for this lab):** Keep everything fully internal/static IP, and add a **second NAT adapter** (Adapter 2) on each VM for internet access (Windows Updates, apt/pip installs, tool downloads). Traffic on Adapter 1 (internal) stays isolated; Adapter 2 (NAT) gets you online. This is the standard "airgapped-but-can-still-patch" pattern.
   - **Option B:** Use a VirtualBox "NAT Network" instead of "Internal Network," which supports inter-VM communication *and* internet access on one adapter, at the cost of being slightly less isolated. Either works — Option A is more realistic for a SOC lab.
5. **Create all 4 VMs now** (Settings only — don't install OS yet): assign RAM/CPU/disk per the table above, attach the ISO to each VM's virtual optical drive, set networking as above. Enable "Nested VT-x/AMD-V" only if your host supports it and you plan to run anything virtualized inside a VM (not required here).
6. **IP plan** (static, assign during OS install/config, not now):

| Host | IP | Role |
|---|---|---|
| DC01 | 192.168.100.10 | Domain Controller / DNS |
| WIN11 | 192.168.100.20 | Domain-joined workstation |
| KALI | 192.168.100.30 | Attacker box |
| WAZUH | 192.168.100.40 | SIEM |

Subnet mask `255.255.255.0`, no gateway needed on the internal adapter (gateway only matters on the NAT adapter, which VirtualBox handles automatically).

---

## Phase 2 – Install Windows Server 2025 (DC01)

1. Boot the VM off the ISO. Choose **Windows Server 2025 Standard (Desktop Experience)** — not Server Core; you want the GUI for this lab.
2. Custom install (not upgrade), delete/create partition on the virtual disk, let it install and reboot (it will reboot once automatically — this is normal, don't touch anything).
3. Set the local Administrator password when prompted.
4. **Rename the computer to DC01** first, *before* promoting to a DC (renaming after is more disruptive): Server Manager → Local Server → Computer name → Change → reboot.
5. **Set the static IP on the internal adapter:** Network & Internet settings (or `ncpa.cpl`) → the internal-network NIC → IPv4 properties:
   - IP: 192.168.100.10 / 255.255.255.0
   - Default gateway: leave blank (or point to nothing) on this adapter
   - Preferred DNS: **127.0.0.1** (it will be its own DNS server once AD DS is installed)
   - Leave the NAT adapter on DHCP for internet access.
6. **Install AD DS role:** Server Manager → Add Roles and Features → Server Roles → check "Active Directory Domain Services" → accept the feature prompts → Install.
7. **Promote to Domain Controller:** After install finishes, click the flag/notification in Server Manager → "Promote this server to a domain controller" → **Add a new forest** → Root domain name: `SOCLAB.LOCAL` → set DSRM password (write it down somewhere safe — you'll rarely need it, but if AD breaks, you need it) → accept defaults for NetBIOS name (SOCLAB) → Install. The server will reboot automatically.
8. **Verify:** After reboot, log in as `SOCLAB\Administrator`. Open **Active Directory Users and Computers** (dsa.msc) — you should see the SOCLAB.LOCAL domain tree. Open a command prompt and run `nslookup soclab.local` — it should resolve to itself.
9. **Create OUs first, then users** (keeps things organized for GPOs later): In ADUC, right-click the domain → New → Organizational Unit. Suggested OUs: `Employees`, `IT`, `ServiceAccounts`.
10. **Create user accounts** (ADUC → right-click OU → New → User, or faster via PowerShell):
```powershell
Import-Module ActiveDirectory
New-ADUser -Name "John Smith" -SamAccountName "John.Smith" -UserPrincipalName "John.Smith@soclab.local" -Path "OU=Employees,DC=soclab,DC=local" -AccountPassword (ConvertTo-SecureString "ChangeMe123!" -AsPlainText -Force) -Enabled $true -ChangePasswordAtLogon $true
```
Repeat for Jane.Doe, Helpdesk, ITAdmin, Accounting (adjust OU as appropriate — e.g., ITAdmin into the IT OU). Consider adding ITAdmin to **Domain Admins** for later lateral-movement scenarios, and leave the others as standard users — that privilege gap is what makes credential-theft scenarios meaningful later.
11. **Common failure point:** If DNS doesn't resolve or AD DS promotion fails, it's almost always because the static IP wasn't set *before* promotion, or DNS was pointed somewhere other than 127.0.0.1. Fix the IP/DNS and retry.

---

## Phase 3 – Windows 11 Workstation (WIN11)

1. Install Windows 11 Enterprise Evaluation normally. At the "Who's going to use this device" screen, choose **Domain join instead** if offered, or just create a local account and join the domain after install (simpler — do this).
2. Set static IP: 192.168.100.20 / 255.255.255.0, **DNS = 192.168.100.10** (this is critical — the workstation must use DC01 as DNS or domain join will fail with "cannot locate domain controller").
3. Join the domain: Settings → Accounts → Access work or school → or classic route: This PC → Properties → Advanced system settings → Computer Name → Change → Domain: `SOCLAB.LOCAL` → enter Administrator credentials when prompted → reboot.
4. **Verify domain join:** After reboot, the login screen should show "Other user" / `SOCLAB\username`. Log in as `John.Smith`. On DC01, in ADUC, confirm the WIN11 computer object appears under Computers.
5. Install Chrome, Firefox, 7-Zip (download via the NAT adapter's internet access). Office is optional and only needed if you want document-based attack scenarios (e.g., macro payloads) later — skip it if you're keeping scope tight.
6. **Common failure point:** "Domain not found" during join almost always = DNS misconfigured on WIN11, or the internal-network adapter name/label doesn't actually match `SOC-LAB` on both VMs. Double check both.

---

## Phase 4 – Wazuh SIEM (WAZUH)

1. Install Ubuntu Server 24.04 on the WAZUH VM. During install, enable OpenSSH server (makes remote administration far easier than working in the VirtualBox console). Set a static IP via netplan or the installer's network step: 192.168.100.40.
2. Update the system first: `sudo apt update && sudo apt upgrade -y`.
3. Resource note: the Wazuh indexer (OpenSearch-based) is memory-hungry. 4GB will technically boot it but 8GB avoids random OOM kills during heavy log ingestion — bump the VM RAM now if you allocated only 4GB.
4. Use the **Wazuh quickstart all-in-one installer** (installs manager + indexer + dashboard in one go) rather than doing each component manually your first time — much lower chance of a broken cross-component config:
```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```
(Check the current version number on the official Wazuh docs before running — the exact script name/version changes over releases.)
5. The installer prints the **admin password for the dashboard at the end of the run** — copy it immediately, it's not shown again (you can regenerate it later with the wazuh-passwords-tool if you lose it).
6. Verify the dashboard: from your host browser, go to `https://192.168.100.40` (you'll need host-to-VM connectivity — if you only have Internal Network + NAT, temporarily add a Host-Only adapter to WAZUH so your host browser can reach it, or use VirtualBox port forwarding on the NAT adapter). Accept the self-signed cert warning. Log in as `admin` with the password from step 5.
7. **Common failure point:** If the dashboard won't load, check `sudo systemctl status wazuh-dashboard wazuh-manager wazuh-indexer` — indexer failing to start is usually a memory or `vm.max_map_count` issue (the installer should set this, but verify with `sysctl vm.max_map_count`, should be ≥262144).

---

## Phase 5 – Sysmon (on DC01 and WIN11)

1. Download Sysmon from Microsoft Sysinternals (via NAT adapter internet access).
2. Download the SwiftOnSecurity `sysmonconfig-export.xml` from its GitHub repo — this is a widely-used, well-tuned baseline config that filters noise while keeping security-relevant events.
3. Install from an elevated PowerShell/cmd prompt:
```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```
4. **Verify:** Open Event Viewer → Applications and Services Logs → Microsoft → Windows → **Sysmon** → Operational. You should see Event ID 1 (process creation) entries populating as you use the machine. If the log is empty, check `Get-Service sysmon64` — it should show Running.
5. Repeat identically on WIN11.

---

## Phase 6 – Wazuh Agents (on DC01 and WIN11)

1. On the Wazuh dashboard: Agents → Add agent → select Windows → it generates a install command with the manager IP pre-filled (192.168.100.40) and an agent name.
2. Run the generated PowerShell command on DC01 and WIN11 (downloads and installs the agent MSI, registers with the manager automatically).
3. Start the agent service: `NET START WazuhSvc` (the installer usually does this, but confirm with `Get-Service WazuhSvc`).
4. **Critical step people miss:** by default the Wazuh agent does *not* automatically forward the Sysmon operational log — you must add it explicitly to the agent's `ossec.conf` (Program Files\ossec-agent\ossec.conf) inside a `<localfile>` block:
```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```
Restart the WazuhSvc service after editing.
5. **Verify end-to-end:** In the Wazuh dashboard, go to Agents — DC01 and WIN11 should show status "Active." Then go to the Discover/Events view and filter for `rule.groups: sysmon` or search for `location: Microsoft-Windows-Sysmon/Operational`. Trigger a test event (e.g., run `notepad.exe`) and confirm it appears in the dashboard within ~30-60 seconds.
6. **Common failure point:** Agent shows "Never connected" or "Disconnected" — usually a firewall issue. Ensure Windows Firewall on DC01/WIN11 allows outbound to WAZUH on TCP/UDP 1514 and 1515, and that the WAZUH Ubuntu firewall (ufw, if enabled) allows those ports inbound.

---

## Phase 7 – Kali Setup

1. Import the pre-built Kali OVA into VirtualBox (File → Import Appliance) rather than installing from scratch — saves significant time and comes with most tools preinstalled.
2. Set static IP 192.168.100.30 on the internal adapter (via `/etc/netplan/*.yaml` or NetworkManager, depending on your Kali image).
3. Update first: `sudo apt update && sudo apt full-upgrade -y`.
4. Most of your tool list ships by default on Kali (nmap, hydra, Responder) — verify with `which nmap hydra responder`. For the rest:
```bash
sudo apt install -y crackmapexec bloodhound
pipx install impacket
sudo apt install -y evil-winrm   # or: gem install evil-winrm
```
5. **SharpHound**: download the collector matching your BloodHound version (mismatched versions are a very common source of "BloodHound won't ingest this data" errors) from the BloodHound GitHub releases page — run it *on WIN11 or DC01* (it's a Windows collector), not on Kali, then pull the resulting zip back to Kali for analysis.
6. **Atomic Red Team**: this runs on the *target* Windows systems, not Kali. Install via PowerShell on WIN11/DC01:
```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
Install-AtomicRedTeam -getAtomics
```
7. Start Neo4j for BloodHound before first use: `sudo neo4j start`, then launch `bloodhound` — set a Neo4j password on first login.
8. **Common failure point:** Kali's internal-network interface sometimes comes up as a different interface name than expected (`eth0` vs `eth1`) — run `ip a` to confirm before configuring static IP.

---

## Phase 8 – Attack Scenarios

Run each scenario individually, confirm the Sysmon event fires *and* lands in Wazuh, before moving to the next — this makes Phase 9/10 far easier since you already know which raw events to hunt for.

1. **Nmap scans** (from Kali against the 192.168.100.0/24 range): start with `nmap -sV -O 192.168.100.0/24` for discovery, then targeted `-sC -sV` scans against DC01/WIN11 for service enumeration.
2. **Password spraying with Hydra**: target WinRM or SMB on a workstation using a small list of the accounts you created and a handful of common passwords. Keep attempts low/slow initially — this is also a good opportunity to observe account lockout policy behavior (or lack thereof, if you haven't configured one yet — consider setting a lockout policy in Default Domain Policy first if you want that signal).
3. **Successful login**: use the correct credentials once spraying "succeeds," via evil-winrm from Kali: `evil-winrm -i 192.168.100.20 -u John.Smith -p 'ChangeMe123!'`.
4. **Encoded PowerShell**: from the evil-winrm session (or RDP), run a base64 `-EncodedCommand` payload — something benign like a `whoami` or `Get-Process` wrapped in `-enc` — this specifically tests Sysmon Event ID 1 / command-line logging visibility.
5. **Scheduled task persistence**: create a scheduled task pointing at a benign script/binary using `schtasks /create` — this maps to Sysmon Event ID 1 (process create for schtasks.exe) and Windows Security Event ID 4698.
6. **Credential dumping (Mimikatz)**: run from an elevated session on WIN11 or DC01 (note: modern Defender will likely flag/quarantine Mimikatz — you may need to temporarily disable real-time protection or add an exclusion in your isolated lab, since the point here is to observe *Sysmon's* detection of LSASS access, e.g. Event ID 10, not to evade AV). `sekurlsa::logonpasswords` is the classic module.
7. **Lateral movement**: from Kali, use captured/known credentials via `evil-winrm` or Impacket's `psexec.py` to move from one host to another (e.g., WIN11 → DC01, or vice versa, depending on privilege).
8. **Atomic Red Team simulations**: run a handful of specific technique tests rather than the whole library, so each maps cleanly to a MITRE ID for Phase 11. Example: `Invoke-AtomicTest T1003.001` (LSASS credential dumping) or `Invoke-AtomicTest T1053.005` (scheduled task).
9. After each scenario, check the Wazuh dashboard's Discover view immediately — if nothing shows up, that's your cue to fix log forwarding/rules before stacking more attacks on top of a blind spot.

---

## Phase 9 – Threat Hunting

For each attack in Phase 8, go into Wazuh and manually search/build a query for the corresponding raw signal *before* writing a formal rule — this is what makes Phase 10 meaningful rather than copy-pasted rules.

- Encoded PowerShell → search Sysmon EID 1 where `CommandLine` contains `-enc` or `-EncodedCommand`.
- LSASS access → Sysmon EID 10 where `TargetImage` contains `lsass.exe`.
- Scheduled tasks → Sysmon EID 1 for `schtasks.exe`/`taskeng.exe`, plus Windows Security EID 4698.
- Registry changes → Sysmon EID 13 (RegistryEvent), especially Run keys under `HKLM\...\CurrentVersion\Run`.
- Failed logins → Windows Security EID 4625; look for bursts against multiple accounts (spray signature) vs. repeated attempts on one account (brute force signature).
- Suspicious processes → Sysmon EID 1 with unusual parent/child relationships (e.g., `winword.exe` spawning `powershell.exe`), or processes launched from unusual paths (`\Users\...\AppData\Local\Temp\`).

Document, for each, the exact Wazuh query/filter you used — you'll reuse this almost verbatim as the "detection logic" section in Phase 10 and Phase 11.

---

## Phase 10 – Detection Engineering

1. Write one Sigma rule per attack scenario, based directly on the field/values you identified while hunting in Phase 9 — don't guess at field names, pull them from the actual Wazuh event JSON (expand an event in Discover to see raw fields).
2. Sigma rules target the log source's field taxonomy (Sysmon field names, in this case) — validate structure with the `sigma-cli`/`pySigma` tool (`pip install sigma-cli`) using `sigma check` before converting.
3. Convert to Wazuh-native rules: Wazuh doesn't consume Sigma directly in older versions — you'll either hand-translate the Sigma logic into a custom Wazuh XML rule (`/var/ossec/etc/rules/local_rules.xml`) or use a Sigma-to-Wazuh converter, depending on your Wazuh version's Sigma support. Check current Wazuh docs for the supported method at your installed version.
4. After adding each custom rule, restart the manager (`sudo systemctl restart wazuh-manager`), re-run the corresponding attack from Phase 8, and confirm the new rule fires (check Discover for your custom rule ID/description) — this closes the loop from "I can find this manually" to "the SIEM tells me automatically."

---

## Phase 11 – Incident Response Documentation

For **each** attack scenario, produce a short report containing:
- Executive summary (2-3 sentences, non-technical)
- Timeline (timestamped, pulled straight from Wazuh event timestamps)
- MITRE ATT&CK technique ID and tactic (e.g., T1003.001 — Credential Access)
- IOCs (process names, command lines, file hashes if applicable, source/destination IPs)
- Detection (which Wazuh/Sigma rule fired, or would need to)
- Containment/eradication steps you'd take in a real environment
- Recovery steps
- Lessons learned / recommended hardening

Keep a consistent template across all scenarios — this consistency is what reads as "professional" to anyone reviewing your portfolio.

---

## Phase 12 – GitHub Portfolio

1. Structure the repo clearly, e.g.:
```
/architecture/       - network diagram, AD structure diagram
/screenshots/         - Wazuh dashboard views, ADUC, attack execution proof
/detections/          - Sigma rules + Wazuh XML rules
/incident-reports/    - one markdown file per Phase 8 scenario, using the Phase 11 template
/mitre-mapping/        - a single table mapping every technique tested to detection status
README.md
```
2. **Redact before publishing:** strip real usernames/passwords used in testing (even lab ones — habit-forming for real engagements), internal IPs are fine to leave since it's a private range, but scrub any host-specific identifiers you wouldn't want public if you ever reuse this repo as a live reference.
3. A single MITRE ATT&CK Navigator-style heatmap image showing covered techniques is a strong portfolio centerpiece — MITRE provides a free web-based Navigator tool for generating this from a technique list.
