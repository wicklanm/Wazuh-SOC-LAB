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

### Attack Chain Overview

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

If you have not done so, you may want to install an RDP session. Evil-WINRM did not work for me as I was unable to download some pre-requisites. Run this to install XfreeRDP:

- sudo apt install -y freerdp2-x11

Once you know (or set) a valid credential pair, connect via xfreerdp to confirm access (below is a sample username and passord from the WIN11 machine):
```bash
xfreerdp /v:192.168.100.20 -u John.Smith -p 'ChangeMe123!'
```
You are now remoted in.

<img width="1284" height="706" alt="Screenshot 2026-08-09 124029" src="https://github.com/user-attachments/assets/0a6df8d4-4dd5-466a-a0c5-99fa48b2700c" />

**Verify in Wazuh:** Windows Security Event ID 4624 (successful logon) — note the Logon Type (3 = network, typical for WinRM) as a field you'll reference in Phase 9/11.

---

### Scenario 4 — Encoded PowerShell Execution

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

<img width="391" height="713" alt="Screenshot 2026-08-09 130038" src="https://github.com/user-attachments/assets/89f8dfe7-ba92-4929-96bd-cf370cdc4a7f" />

**Verify in Wazuh:** Sysmon Event ID 1 (process creation) — filter for `CommandLine` containing `-enc` or `-EncodedCommand`. The `-WindowStyle Hidden` variant is particularly worth noting as a common real-world evasion flag.

**MITRE:** T1059.001 — PowerShell

---

### Scenario 5 — Scheduled Task Persistence

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

### Scenario 6 — Credential Dumping (No Mimikatz / Defender-Safe)

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

### Scenario 7 — Domain Enumeration (Pre-Lateral Movement Recon)

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

<img width="761" height="587" alt="Screenshot 2026-08-09 131647" src="https://github.com/user-attachments/assets/1beaa75d-c97e-4227-8ff1-6f489c23c179" />
<img width="730" height="219" alt="Screenshot 2026-08-09 131732" src="https://github.com/user-attachments/assets/711c8bc4-1b62-4b5d-b1d4-dfdcf9be8d8b" />
<img width="736" height="501" alt="Screenshot 2026-08-09 132400" src="https://github.com/user-attachments/assets/6778316e-add9-4980-baae-6c053c09ab97" />
<img width="595" height="557" alt="Screenshot 2026-08-09 132905" src="https://github.com/user-attachments/assets/d7b4a60e-bc62-4876-9126-d79d59786fc9" />

**Verify in Wazuh:** Sysmon Event ID 1 for `net.exe` calls, Windows Security Event ID 4661 (object handle requested) for directory service queries.

**MITRE:** T1087.002 (Domain Account Discovery), T1018 (Remote System Discovery)

---

### Scenario 8 — Lateral Movement (WIN11 → DC01)

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

<img width="761" height="587" alt="Screenshot 2026-08-09 131647" src="https://github.com/user-attachments/assets/1e0da96a-13b9-4199-a3ed-771e05db8127" />


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

### Scenario 9 — Atomic Red Team Simulations

*Each test simulates a specific MITRE technique in a controlled, documented way — they're not actual malware, they're proof-of-concept executions that mimic what malware would do*

Run from PowerShell (Admin) on WIN11 inside the RDP session:

```powershell
Import-Module "C:\AtomicRedTeam\invoke-atomicredteam\Invoke-AtomicRedTeam.psd1" -Force
```

<img width="1285" height="693" alt="Screenshot 2026-08-09 132914" src="https://github.com/user-attachments/assets/a051f5f3-900e-4341-b5a0-5b20e27c0fb0" />

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

<img width="1289" height="719" alt="Screenshot 2026-08-09 133153" src="https://github.com/user-attachments/assets/d4df5e50-538d-46c2-9226-c394caba808b" />

**Check prerequisites before running** (some tests need specific tools/conditions):
```powershell
Invoke-AtomicTest T1003.001 -CheckPrereqs
```

**Clean up after each test:**
```powershell
Invoke-AtomicTest T1003.001 -Cleanup
Invoke-AtomicTest T1053.005 -Cleanup
```

<img width="881" height="679" alt="Screenshot 2026-08-09 134247" src="https://github.com/user-attachments/assets/07a95435-ce05-4611-810d-f28d20fff077" />

**Verify in Wazuh:** same Event IDs as the corresponding manual scenarios above — running both confirms your detections aren't tied to one specific tool's behavior.

<img width="1911" height="828" alt="Screenshot 2026-08-09 133830" src="https://github.com/user-attachments/assets/18eebfdf-a1ff-478a-9db9-28c446610b60" />

<img width="523" height="694" alt="Screenshot 2026-08-09 134101" src="https://github.com/user-attachments/assets/ddca66d6-3d65-48e9-a2a7-b65ee0c8b0cd" />

---

### Full MITRE ATT&CK Reference

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

### Common Failure Points

| Symptom | Likely Cause / Fix |
|---|---|
| `Enter-PSSession` fails with access denied | WinRM not enabled on DC01 — run `Enable-PSRemoting -Force` on DC01 |
| comsvcs.dll dump creates empty file | Not running PowerShell as Administrator — reopen elevated |
| Sysmon Event ID 10 not appearing in Wazuh for LSASS dump | Confirm the Sysmon `<localfile>` block is still in ossec.conf on WIN11 and the WazuhSvc was restarted after adding it |
| Atomic Red Team import fails | Path may differ — run `dir C:\AtomicRedTeam` to confirm install location |
| `net use` to DC01 fails | Confirm DC01 has File and Printer Sharing enabled and the Windows Firewall rule for it is active |

---

## Phase 9 – Threat Hunting

### Overview

Threat hunting is the process of proactively searching through logs and telemetry to find evidence of the attacks you ran in Phase 8 — before a rule fires, using raw data. Every search below corresponds directly to a Phase 8 scenario. The goal is to find the raw signal manually first, then document the exact query/field values you used — you'll reuse these almost verbatim when writing detection rules in Phase 10.

---

### Before You Start

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

### How to Search in Wazuh

The search bar at the top of Threat Hunting accepts KQL (Kibana Query Language). Basic syntax:
```
field.name: value                    # exact match
field.name: *partial*                # wildcard match
field.name: value1 AND field.name2: value2   # combine conditions
field.name: value1 OR field.name: value2     # either condition
```

After running a search, click any event row to expand it and see all raw fields — this is how you discover the exact field names to use in your queries and in Phase 10's detection rules.

---

### Hunt 1 — Nmap Scan (Network Reconnaissance)

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
- In my case specifically, it will be connections via RDP
- Hitting many different `destinationPort` values on WIN11

**Fields to document for Phase 10:**
- `data.win.system.eventID`
- `data.win.eventdata.sourceIp`
- `data.win.eventdata.destinationPort`

**MITRE:** T1046 — Network Service Discovery

---

### Hunt 2 — Password Spray (Failed Logins)

**What to look for:** multiple failed login attempts spread across different usernames in a short time window — the breadth-across-accounts pattern is what distinguishes a spray from a single-account brute force.

**Search:**
```
data.win.system.eventID: 4625
```
Windows Security Event ID 4625 = failed logon.

<img width="1256" height="640" alt="hunt_2_password_spray" src="https://github.com/user-attachments/assets/44b875b0-37f6-4305-9237-6f95619e5cbb" />

<img width="632" height="678" alt="Details_hunt_2_password_spray" src="https://github.com/user-attachments/assets/e5ec989f-66e2-4a74-b513-67379226aa9f" />

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

### Hunt 3 — Successful RDP Login (Initial Access)

**What to look for:** a successful logon from Kali's IP, Logon Type 10 (RemoteInteractive = RDP).

**Search:**
```
data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 10
```
Windows Security Event ID 4624 = successful logon. Logon Type 10 = RemoteInteractive (RDP specifically).

<img width="1255" height="452" alt="Hunt3_Successful_RDP_Login" src="https://github.com/user-attachments/assets/d47e359c-f266-4a72-98fd-75a7be5f47eb" />
<img width="1265" height="558" alt="Hunt3_Successful_RDP_Login_refined" src="https://github.com/user-attachments/assets/db992f4e-4ff5-4a61-95a1-3c576815c7e6" />

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

### Hunt 4 — Encoded PowerShell

**What to look for:** PowerShell processes launched with `-enc` or `-EncodedCommand` in the command line — a classic obfuscation technique.

**Search:**
```
data.win.system.eventID: 1 AND data.win.eventdata.commandLine: *-enc*
```
Sysmon Event ID 1 = process creation. The `*-enc*` wildcard catches both `-enc` and `-EncodedCommand`.

<img width="1266" height="580" alt="Hunt4_Encoded_Powershell_Obfuscation_technique" src="https://github.com/user-attachments/assets/91fbfd9a-92e1-4964-9279-9d2543deaa64" />

<img width="957" height="863" alt="Hunt4_Encoded_Powershell_Obfuscation_technique_Details" src="https://github.com/user-attachments/assets/766c99cb-1199-4c00-af4a-a0b5e34f055f" />

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

### Hunt 5 — Scheduled Task Persistence

**What to look for:** schtasks.exe execution creating a new task, and the corresponding Windows Security audit event.

**Search for the process creation:**
```
data.win.system.eventID: 1 AND data.win.eventdata.image: *schtasks*
```

<img width="1190" height="507" alt="Screenshot 2026-08-12 184717" src="https://github.com/user-attachments/assets/f48861e8-b734-4460-959c-0d1158f0d56c" />

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

### Hunt 6 — LSASS Access / Credential Dumping

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

### Hunt 7 — Domain Enumeration

**What to look for:** net.exe commands querying domain users, groups, and computers in a short window.

**Search:**
```
data.win.system.eventID: 1 AND data.win.eventdata.image: *net.exe*
```

<img width="1261" height="814" alt="Hunt7_Domain_Enumeration" src="https://github.com/user-attachments/assets/d05b6066-6d07-4d31-8b60-b172aeb1c1ed" />

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

### Hunt 8 — Lateral Movement (WIN11 → DC01)

**What to look for:** a successful network logon on DC01 originating from WIN11's IP (192.168.100.20), using explicit credentials.

**Search on DC01's agent specifically:**
```
agent.name: DC01 AND data.win.system.eventID: 4624 AND data.win.eventdata.logonType: 3
```
Logon Type 3 = network logon (covers PSRemoting, net use, WMI-based movement).

<img width="1223" height="812" alt="Hunt8_LateralMovement_searching_for logging in on DC01 completely" src="https://github.com/user-attachments/assets/3ad6b492-0cd8-42eb-a590-a8dbf4ae9dd2" />

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

### Hunt Summary Table

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

### If a Hunt Returns No Results

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

<img width="1411" height="654" alt="Screenshot 2026-08-10 201936" src="https://github.com/user-attachments/assets/c0cf467b-f3b2-4824-8361-9a79f7ec4196" />

6. **Check the agent log on WIN11 for errors:**
```powershell
Get-Content "C:\Program Files (x86)\ossec-agent\ossec.log" -Tail 50
```

<img width="1510" height="817" alt="Screenshot 2026-08-10 202207" src="https://github.com/user-attachments/assets/d39c3097-d561-46c2-90bb-01f1e2ced8cf" />
---

## Phase 10 – Detection Engineering

### Overview

Detection engineering is the process of turning the raw signals you found hunting in Phase 9 into automated rules that fire alerts whenever those patterns appear again. Every rule you write here maps directly to one Phase 8 attack scenario and one Phase 9 hunt query — nothing should be invented from scratch. The workflow for each rule is:

```
Phase 9 query (what you searched for manually)
    ↓
Identify exact field names and values from expanded Wazuh events
    ↓
Write a custom Wazuh XML rule in local_rules.xml
    ↓
Test with wazuh-logtest
    ↓
Re-run the Phase 8 attack
    ↓
Confirm the rule fires in the dashboard
```

---

### Critical Rule Before You Start

<cite index="2-1,4-1">All custom rules belong in `/var/ossec/etc/rules/local_rules.xml` on the WAZUH machine — never in `/var/ossec/ruleset/rules/`. That directory is owned by Wazuh and gets overwritten on every upgrade, wiping any rules you put there. Custom rules in `local_rules.xml` survive upgrades.</cite>

<cite index="4-1">Custom rule IDs must be 100000 or above</cite> — built-in Wazuh rules use lower numbers and you must never overlap with them.

<cite index="4-1">Rule levels run 0 to 16. Level 0 means store but do not alert. Higher levels indicate higher severity.</cite> Use these as a guide:

| Level | Use For |
|---|---|
| 3 | Informational / low noise |
| 7 | Suspicious activity worth reviewing |
| 10 | Likely malicious — should alert |
| 12 | High confidence attack |
| 15 | Critical — immediate response |

---

### Part A: Opening local_rules.xml

**SSH into WAZUH** from a terminal on your host machine (or use the VirtualBox console):
```bash
sudo nano /var/ossec/etc/rules/local_rules.xml
```

The file already exists and contains a default comment. All your rules go inside the outer `<group>` tags. The basic structure of the whole file should look like this:

```xml
<group name="local,soc-lab,">

  <!-- Your rules go here, one after another -->

</group>
```

Every rule block sits between those group tags. Save after adding each rule (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

---

### Part B: Rule XML Structure

Every rule follows this pattern:

```xml
<rule id="UNIQUE_ID" level="SEVERITY">
  <if_sid>PARENT_RULE_ID</if_sid>
  <field name="FIELD_NAME">VALUE_TO_MATCH</field>
  <description>Human readable alert text</description>
  <mitre>
    <id>TXXXX.XXX</id>
  </mitre>
</rule>
```

Key elements explained:

| Element | Purpose |
|---|---|
| `id` | Your unique rule ID — start at 100001, increment by 1 for each rule |
| `level` | Severity 0-16 |
| `if_sid` | Parent rule ID this fires on top of — Sysmon Event ID 1 is rule `61603`, Security events use `60103` (Windows base) |
| `field name` | The decoded field name from the Wazuh event JSON — copy these exactly from expanded events in the dashboard |
| `description` | The alert text analysts see — make it specific and actionable |
| `mitre` | MITRE ATT&CK technique ID for the alert |

**Finding the right `if_sid` value:** In Wazuh dashboard, expand any event you found during Phase 9 hunting → look for the `rule.id` field in the raw JSON. That number is what you put in `if_sid` — it tells Wazuh to only evaluate your rule when that parent rule already matched.

---

### Part C: Write the Rules

Add each rule below to `local_rules.xml` one at a time, testing each before adding the next.

#### Rule 1 — Nmap Scan Detection
```xml
<rule id="100001" level="7">
  <if_sid>60103</if_sid>
  <field name="win.system.eventID">^5157$</field>
  <field name="win.eventdata.sourceAddress">192.168.100.30</field>
  <description>Possible network scan detected from Kali attacker box (192.168.100.30)</description>
  <mitre>
    <id>T1046</id>
  </mitre>
</rule>
```

#### Rule 2 — Password Spray (Multiple Failed Logins)
```xml
<rule id="100002" level="10" frequency="5" timeframe="30">
  <if_matched_sid>60204</if_matched_sid>
  <same_field>win.eventdata.ipAddress</same_field>
  <different_field>win.eventdata.targetUserName</different_field>
  <description>Password spray detected — multiple failed logons from same IP against different accounts</description>
  <mitre>
    <id>T1110.003</id>
  </mitre>
</rule>
```
> Note: `frequency="5"` and `timeframe="30"` means 5 failures from the same IP hitting different usernames within 30 seconds triggers this rule. Adjust the numbers to match what you actually saw in Phase 9.

#### Rule 3 — Successful RDP Login from Attacker IP
```xml
<rule id="100003" level="10">
  <if_sid>60106</if_sid>
  <field name="win.system.eventID">^4624$</field>
  <field name="win.eventdata.logonType">^10$</field>
  <field name="win.eventdata.ipAddress">192.168.100.30</field>
  <description>Successful RDP login from Kali attacker box (192.168.100.30) — possible unauthorized access</description>
  <mitre>
    <id>T1021.001</id>
  </mitre>
</rule>
```

#### Rule 4 — Encoded PowerShell Execution
```xml
<rule id="100004" level="12">
  <if_sid>61603</if_sid>
  <field name="win.eventdata.commandLine" type="pcre2">(?i)-e(nc|ncodedcommand)\s</field>
  <description>Encoded PowerShell command detected — possible obfuscated execution</description>
  <mitre>
    <id>T1059.001</id>
  </mitre>
</rule>
```
> Note: `type="pcre2"` enables regex matching. This catches both `-enc` and `-EncodedCommand` in any case combination.

#### Rule 5a — Scheduled Task Creation
```xml
<rule id="100005" level="10">
  <if_sid>60103</if_sid>
  <field name="win.system.eventID">^4698$</field>
  <description>Scheduled task created — possible persistence mechanism</description>
  <mitre>
    <id>T1053.005</id>
  </mitre>
</rule>
```

#### Rule 5b — Registry Run Key Persistence
```xml
<rule id="100006" level="10">
  <if_sid>61603</if_sid>
  <field name="win.system.eventID">^13$</field>
  <field name="win.eventdata.targetObject" type="pcre2">CurrentVersion\\\\Run</field>
  <description>Registry Run key modified — possible persistence via startup entry</description>
  <mitre>
    <id>T1547.001</id>
  </mitre>
</rule>
```

#### Rule 6 — LSASS Memory Access (Credential Dumping)
```xml
<rule id="100007" level="15">
  <if_sid>61603</if_sid>
  <field name="win.system.eventID">^10$</field>
  <field name="win.eventdata.targetImage" type="pcre2">(?i)lsass\.exe</field>
  <description>LSASS process accessed by non-system process — likely credential dumping attempt</description>
  <mitre>
    <id>T1003.001</id>
  </mitre>
</rule>
```
> This is your highest-severity rule — level 15 is appropriate because legitimate processes rarely need to open a handle to LSASS with dump-level access rights.

#### Rule 7 — Domain Enumeration via net.exe
```xml
<rule id="100008" level="7">
  <if_sid>61603</if_sid>
  <field name="win.system.eventID">^1$</field>
  <field name="win.eventdata.image" type="pcre2">(?i)\\net\.exe$</field>
  <field name="win.eventdata.commandLine" type="pcre2">(?i)/domain</field>
  <description>Domain enumeration via net.exe — attacker may be conducting reconnaissance</description>
  <mitre>
    <id>T1087.002</id>
  </mitre>
</rule>
```

#### Rule 8 — Lateral Movement (Network Logon to DC01)
```xml
<rule id="100009" level="12">
  <if_sid>60106</if_sid>
  <field name="win.system.eventID">^4624$</field>
  <field name="win.eventdata.logonType">^3$</field>
  <field name="win.eventdata.ipAddress">192.168.100.20</field>
  <description>Network logon to DC01 from WIN11 — possible lateral movement</description>
  <mitre>
    <id>T1021.002</id>
  </mitre>
</rule>
```

---

### Part D: Test Each Rule Before Restarting

<cite index="3-1">Use the `wazuh-logtest` utility to validate rules without restarting the manager.</cite> On the WAZUH machine:

```bash
sudo /var/ossec/bin/wazuh-logtest
```

This opens an interactive prompt where you paste a raw log and it shows you which decoder and rule matched. This is the fastest way to catch syntax errors before committing.

**Also validate your XML syntax** before restarting — a malformed rule file will prevent the manager from loading any rules at all:
```bash
sudo /var/ossec/bin/wazuh-xml /var/ossec/etc/rules/local_rules.xml
```
If it returns nothing, the XML is clean. If it returns an error, fix the indicated line before restarting.

---

### Part E: Apply and Restart

After adding all rules and confirming clean XML:
```bash
sudo systemctl restart wazuh-manager
```

Verify it came back up cleanly:
```bash
sudo systemctl status wazuh-manager
```
Should show `active (running)`. If it shows `failed`, the rules file has a syntax error — check the manager log:
```bash
sudo tail -50 /var/ossec/logs/ossec.log
```
Look for `Error` lines pointing to your rule file and fix accordingly.

---

### Part F: Validate Each Rule Fires

For each rule, go back to the corresponding Phase 8 attack and re-run it, then check the dashboard:

1. In Wazuh dashboard → **Threat Hunting**
2. Filter by your custom rule IDs:
```
rule.id: 100001 OR rule.id: 100002 OR rule.id: 100003
```
Or filter by description text:
```
rule.description: *SOC-LAB* OR rule.description: *attacker*
```
3. Confirm the alert fired with the correct severity level and MITRE ID
4. Screenshot the alert for your Phase 12 GitHub portfolio

---

### Part G: View Rules via Wazuh Dashboard (Alternative to CLI)

You can also manage rules through the web UI without SSH:

1. Left sidebar → **Management** → **Rules**
2. Search for your rule IDs (100001-100009) to confirm they loaded

<img width="1225" height="665" alt="Screenshot 2026-08-12 190536" src="https://github.com/user-attachments/assets/f164b37f-d5e2-402b-8fef-20d49e315e6f" />

3. Click any rule to see its full definition

<img width="862" height="553" alt="Screenshot 2026-08-12 190811" src="https://github.com/user-attachments/assets/2f838ad2-a962-4cdd-9f83-cfcb1caff884" />

4. Use **Management** → **Log test** for the GUI version of `wazuh-logtest`

---

### Rule Summary Reference

| Rule ID | Attack Scenario | Level | MITRE |
|---|---|---|---|
| 100001 | Nmap scan | 7 | T1046 |
| 100002 | Password spray | 10 | T1110.003 |
| 100003 | RDP login from Kali | 10 | T1021.001 |
| 100004 | Encoded PowerShell | 12 | T1059.001 |
| 100005 | Scheduled task created | 10 | T1053.005 |
| 100006 | Registry run key write | 10 | T1547.001 |
| 100007 | LSASS access | 15 | T1003.001 |
| 100008 | Domain enumeration | 7 | T1087.002 |
| 100009 | Lateral movement to DC01 | 12 | T1021.002 |

---

### Common Failure Points

| Symptom | Likely Cause / Fix |
|---|---|
| Manager fails to start after adding rules | XML syntax error — run `wazuh-xml` check, look for unclosed tags or mismatched quotes |
| Rule never fires even when attack is re-run | Wrong `if_sid` parent — expand a real event in the dashboard and copy the actual `rule.id` value into your `if_sid` |
| Rule fires but wrong fields show in alert | Field name typo — field names are case-sensitive, copy them exactly from expanded event JSON |
| LSASS rule fires constantly on normal activity | Normal LSASS access by system processes — add a `<field name="win.eventdata.sourceImage">` filter to exclude known-good callers like `MsMpEng.exe` (Defender) |
| Password spray rule never fires | `frequency` threshold too high for your test — lower it to `3` and `timeframe` to `60` for lab testing |

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
