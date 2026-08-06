# Phase 7 – Kali Linux Setup (Attacker Box)

Installed via the official Kali ISO in VirtualBox, joined to the `SOC-LAB` internal network alongside DC01, WIN11, and WAZUH.

## Part A: Installing Kali from ISO

### 1. Confirm which ISO variant you downloaded

Kali offers a few variants — this determines what happens during install:

- **Installer** (full offline installer, ~4GB+, most tools pre-included) — recommended, and almost certainly what you have if you grabbed the default download from kali.org/get-kali
- **NetInst** (small ISO, downloads packages during install — requires internet mid-install)
- **Live** (boots into a live desktop first; installing from there is a different flow)

If unsure, check the file size — several GB indicates the Installer image.

### 2. Attach the ISO and boot the VM

Choose **Graphical install** at the boot menu (not "Live" or text-mode "Install").

### 3. Work through the installer screens

- **Language/Location/Keyboard:** defaults are fine
- **Hostname:** `kali`
- **Domain name:** leave blank
- **User account:** modern Kali installers create a non-root sudo user during setup — set a username/password you'll remember; you'll use this to log in and `sudo` for everything going forward
- **Partitioning:** Guided → use entire disk → select the virtual disk → "All files in one partition" → confirm and write changes
- **Software selection:** choose **Xfce** for the desktop environment, and make sure the **default tools collection** stays checked (not just a bare desktop) — this determines whether nmap/hydra/Responder etc. are present immediately after install
- **GRUB bootloader:** install to `/dev/sda`

Let it finish, remove the ISO from the virtual optical drive (VM Settings → Storage), and reboot.

### 4. Log in and confirm access

```bash
whoami
sudo -i
```

---

## Part B: Networking

Same pattern as the WAZUH VM — one internal adapter, one NAT adapter.

### 5. Set VM network adapters (VM powered off)

- Adapter 1 → Internal Network → `SOC-LAB`
- Adapter 2 → NAT

Boot Kali.

### 6. Identify interfaces and set a static IP

```bash
ip a
systemctl status NetworkManager
```

**If NetworkManager is active** (typical on a desktop-flavor install), use `nmtui`:

```bash
sudo nmtui
```

Edit the internal-network interface (likely `eth0`) → IPv4 CONFIGURATION → **Manual** → Address: `192.168.100.30/24` → leave Gateway/DNS blank on this interface → OK → Activate the connection.

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

### 7. Verify connectivity

```bash
ping -c 3 192.168.100.10   # DC01
ping -c 3 192.168.100.40   # WAZUH
ping -c 3 8.8.8.8          # internet
```

---

## Part C: Update and Install Tools

### 8. Update the system first

```bash
sudo apt update && sudo apt full-upgrade -y
```

### 9. Check what's already installed

```bash
which nmap hydra responder crackmapexec bloodhound 2>/dev/null
```

Anything that returns a path is already present — skip reinstalling it.

### 10. Use NetExec instead of (or alongside) CrackMapExec

CrackMapExec has been unmaintained since 2023. Kali still packages it for compatibility, but the actively maintained successor, **NetExec**, is the better choice for anything beyond following older tutorials verbatim:

```bash
sudo apt install -y netexec
nxc --version
```

`nxc` is the command-line entry point (`netexec` also works). CrackMapExec can still be installed alongside it if needed:

```bash
sudo apt install -y crackmapexec
```

### 11. Install remaining tools

```bash
sudo apt install -y responder bloodhound neo4j
```

### 12. Impacket (via pipx)

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

### 13. Evil-WinRM

```bash
sudo gem install evil-winrm
evil-winrm --version
```

If `gem` is not found:

```bash
sudo apt install -y ruby-full ruby-dev
```

### 14. BloodHound (requires Neo4j running first)

```bash
sudo neo4j start
```

Wait 15-20 seconds for it to fully start, then open `http://localhost:7474` in a browser inside Kali (e.g. `firefox-esr &`) — log in with the default `neo4j`/`neo4j` credentials and set a new password when prompted. Then launch:

```bash
bloodhound &
```

Log into BloodHound using the Neo4j credentials just set.

### 15. Note: SharpHound and Atomic Red Team are Windows-side

These run directly on DC01/WIN11, not on Kali — nothing to install here for these two; covered in Phase 8.

---

## Common Failure Points

| Symptom | Likely Cause / Fix |
|---|---|
| `apt install` fails with "unable to locate package" | Re-run `sudo apt update` — Kali is a rolling release and package indexes go stale quickly |
| `pip install impacket` errors with "externally managed environment" | Expected on modern Kali — use `pipx` instead of raw `pip` |
| Neo4j / BloodHound login loop | Password mismatch between Neo4j browser console and BloodHound login. Reset with `neo4j stop` then `sudo neo4j-admin dbms set-initial-password newpassword` |
| No internet during `apt update` | Confirm Adapter 2 is enabled and set to NAT |
