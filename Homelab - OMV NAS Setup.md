---
title: OMV NAS Setup
folder: Homelab
tags:
  - homelab
  - proxmox
  - nas
  - omv
  - tailscale
  - backup
date: 2026-04-21
status: completed
---

# OMV NAS Setup

## Overview

Setting up OpenMediaVault (OMV) as a NAS on Proxmox using 2 external USB drives.
Accessible remotely via Tailscale with automated rsync backups from the ThinkPad.

### Goals
- Attach external drives to Proxmox server as a NAS
- Access files from anywhere via Tailscale
- Schedule automated backups from ThinkPad

---

## Outline

- [x] VM created and configured
- [x] OMV installed
- [x] Tailscale installed on OMV
- [x] Drives mounted and shared
- [x] ThinkPad connected via SMB
- [x] Backup script created
- [X] Backup script fully cleaned up and verified
- [X] systemd timer configured
### Stack
| Component | Detail |
|---|---|
| Hypervisor | Proxmox on Dell Inspiron 7560 |
| NAS OS | OpenMediaVault (OMV) |
| File Sharing | Samba (SMB/CIFS) |
| Remote Access | Tailscale |
| Backups | rsync + systemd timer |

---

## Infrastructure Reference

| Item | Value |
|---|---|
| VM Name | `omv-nas` |
| VM ID | 300 |
| OMV Local IP | `10.0.0.24` |
| OMV Tailscale IP | `<omv-nas-tailscale-ip>` |
| Drive 1 | `/dev/sda` - 298 GB (ST320LT012) → `thinkpad-backups` |
| Drive 2 | `/dev/sdb` - 931 GB (ST1000LM035) → `archive` |
| NAS User | `elikyals` |
| ThinkPad Mount 1 | `/mnt/nas/thinkpad-backups` |
| ThinkPad Mount 2 | `/mnt/nas/archive` |

---

## Phase 1 - USB Passthrough

### 1.1 Identify USB Drives

In Proxmox host shell, run:

```bash
lsusb
```

Note the Vendor/Device IDs of your external drives. Example output:
```
Bus 002 Device 010: ID 152d:0578 JMicron JMS578 SATA 6Gb/s
Bus 002 Device 012: ID 152d:0562 JMicron JMS567 SATA 6Gb/s bridge
```

Also confirm stable device IDs:
```bash
ls -l /dev/disk/by-id/ | grep usb
```

### 1.2 Create the VM

In Proxmox web UI → Create VM with these settings:

**General**

| Setting       | Value   |
| ------------- | ------- |
| Name          | omv-nas |
| Start at boot | ✅       |


**OS**

| Setting       | Value                  |
| ------------- | ---------------------- |
| ISO           | OMV ISO (upload first) |
| Guest OS Type | Linux                  |
| Version       | 6.x - 2.6 Kernel       |


**System**

| Setting         | Value              |
| --------------- | ------------------ |
| Machine         | q35                |
| BIOS            | SeaBIOS            |
| SCSI Controller | VirtIO SCSI single |
| Qemu Agent      | ✅                  |


**Disks**

| Setting    | Value        |
| ---------- | ------------ |
| Bus/Device | VirtIO Block |
| Size       | 20GB         |
| Cache      | Write back   |
| Discard    | ✅            |


**CPU**

| Setting | Value |
| ------- | ----- |
| Cores   | 2     |
| Type    | host  |


**Memory**

| Setting | Value   |
| ------- | ------- |
| Memory  | 2048 MB |

**Network**

| Setting  | Value  |
| -------- | ------ |
| Bridge   | vmbr0  |
| Model    | VirtIO |
| Firewall | ❌     |

### 1.3 Add USB Passthrough

With the VM shut down, go to VM → Hardware → Add → USB Device:
- Select **Use USB Vendor/Device ID**
- Add each drive by its Vendor/Device ID
- Repeat for both drives

---

## Phase 2 - Install OMV

Boot the VM and open Console. Follow the installer:

| Step          | Value                                          |
| ------------- | ---------------------------------------------- |
| Language      | English                                        |
| Location      | Your country                                   |
| Locale        | `en_US.UTF-8`                                  |
| Keyboard      | American English                               |
| Hostname      | `omv-nas`                                      |
| Domain        | *(leave blank)*                                |
| Root password | *(set a strong password)*                      |
| Install disk  | `vda` - Virtual disk (NOT the external drives) |

After installation, remove the ISO before rebooting:
- VM → Hardware → CD/DVD Drive → Edit → Do not use any media

### Install QEMU Guest Agent

After first boot, log in as `root` and run:

```bash
apt install qemu-guest-agent -y
systemctl enable --now qemu-guest-agent
reboot
```

> **Note:** You may see a warning that qemu-guest-agent has no Install section in its unit file. This is normal on Debian - it starts automatically via udev.

Confirm it's working: Proxmox web UI → `omv-nas` → Summary → IP address should now show.

---

## Phase 3 - Tailscale on OMV

Since Tailscale is already running on the Proxmox host, install it inside the OMV VM too so it gets its own Tailscale IP:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```

Authenticate via the URL it provides. OMV will now appear in your Tailscale dashboard with its own IP.

Access the OMV web UI from anywhere at:
```
http://<omv-nas-tailscale-ip>
```

---

## Phase 4 - Configure OMV

### 4.1 First Login

- URL: `http://<omv-nas-tailscale-ip>`
- Username: `admin`
- Default password: `openmediavault`
- **Change the password immediately** after first login

### 4.2 Mount the Drives

Storage → File Systems → Mount (plug icon):
- Select each external drive from the dropdown
- Leave Usage Warning Threshold at 85%
- Mount both drives
- Click **Apply Changes**

### 4.3 Create Shared Folders

Storage → Shared Folders → +:

| Setting | Drive 1 | Drive 2 |
|---|---|---|
| Name | `thinkpad-backups` | `archive` |
| File system | 298 GB drive | 931 GB drive |
| Relative path | `/` | `/` |
| Permissions | Admin: RW, Users: RW, Others: RO | Admin: RW, Users: RW, Others: RO |

> **Important:** Set relative path to `/` (root of drive) not the default subfolder name, otherwise existing files on the drive won't be visible.

Click **Apply Changes**.

### 4.4 Enable SMB

Services → SMB/CIFS → Settings → Enable → Save → Apply Changes

Services → SMB/CIFS → Shares → + for each share:

| Setting | Value |
|---|---|
| Enabled | ✅ |
| Shared folder | select the folder |
| Public | No |

Apply Changes.

### 4.5 Create a User

Users → Users → +:

| Setting  | Value               |
| -------- | ------------------- |
| Name     | `elikyals`          |
| Password | *(strong password)* |
| Groups   | `users`             |

Apply Changes.

### 4.6 Dashboard

Enable these widgets: System Information, CPU, Memory, File Systems, Network Interfaces, Disk Temperatures.

---

## Phase 5 - Mount on ThinkPad

### 5.1 Install cifs-utils

```bash
sudo dnf install cifs-utils -y
```

### 5.2 Create Mount Points

```bash
sudo mkdir -p /mnt/nas/archive
sudo mkdir -p /mnt/nas/thinkpad-backups
```

### 5.3 Create Credentials File

```bash
nano ~/.smbcredentials
```

```
username=elikyals
password=<your-omv-password>
```

```bash
chmod 600 ~/.smbcredentials
```

### 5.4 Test Manual Mount

```bash
sudo mount -t cifs //<omv-nas-tailscale-ip>/archive /mnt/nas/archive \
  -o credentials=/home/elikyals/.smbcredentials,uid=1000,gid=1000,iocharset=utf8
```

### 5.5 Add to fstab (Permanent)

```bash
sudo nano /etc/fstab
```

Add these two lines:

```
//<omv-nas-tailscale-ip>/archive  /mnt/nas/archive  cifs  credentials=/home/elikyals/.smbcredentials,uid=1000,gid=1000,iocharset=utf8,_netdev  0  0
//<omv-nas-tailscale-ip>/thinkpad-backups  /mnt/nas/thinkpad-backups  cifs  credentials=/home/elikyals/.smbcredentials,uid=1000,gid=1000,iocharset=utf8,_netdev  0  0
```

> **Note:** `_netdev` tells Fedora to wait for network before mounting. Essential for Tailscale-based mounts.

```bash
sudo mount -a
systemctl daemon-reload
```

Verify:
```bash
df -h | grep nas
```

---

## Phase 6 - Automated Backups

### 6.1 Setup Log Directory

```bash
sudo mkdir -p /var/log/nas-backup
sudo chown elikyals:elikyals /var/log/nas-backup
```

### 6.2 Configure Logrotate

```bash
sudo nano /etc/logrotate.d/nas-backup
```

```
/var/log/nas-backup/nas-backup.log {
    rotate 10
    compress
    missingok
    notifempty
    dateext
    dateformat -%Y-%m-%d
}
```

Keeps last 10 backup logs, compressed, named by date.

### 6.3 Create Backup Script

```bash
mkdir -p ~/scripts
nano ~/scripts/nas-backup.sh
```

```bash
#!/bin/bash

DESTINATION="/mnt/nas/thinkpad-backups"
SOURCE="/home/elikyals"
LOG="/var/log/nas-backup/nas-backup.log"

rsync -avz --progress --copy-links \
  \
  `# --- Lenovo files ---` \
  --exclude='Generic_Safety_and_Compliance_Notices.pdf' \
  --exclude='Lenovo_Limited_Warranty.pdf' \
  --exclude='Software_Product_License_Agreement.pdf' \
  --exclude='UserGuide_ThinkPad_T16_Gen_4.html' \
  \
  `# --- Reinstallable / expendable top-level ---` \
  --exclude='.ansible' \
  --exclude='.virtualenvs' \
  --exclude='.pki' \
  --exclude='.npm' \
  --exclude='.dotnet' \
  --exclude='.th-client' \
  --exclude='.redhat' \
  --exclude='.console-ninja' \
  --exclude='.joinnow' \
  --exclude='.copilot' \
  --exclude='Downloads' \
  --exclude='*.rpm' \
  --exclude='*.iso' \
  --exclude='*.vmdk' \
  --exclude='*.qcow2' \
  --exclude='scripts' \
  --exclude='logs' \
  --exclude='.mozilla' \
  --exclude='.var/' \
  --exclude='.gradle' \
  --exclude='.java' \
  --exclude='.m2' \
  --exclude='.cargo' \
  --exclude='.rustup' \
  --exclude='.nvm' \
  --exclude='.pyenv' \
  --exclude='go/' \
  --exclude='.local/share/Steam' \
  --exclude='.config/Brave-Browser' \
  --exclude='.config/Code' \
  --exclude='.config/Code - OSS' \
  --exclude='.local/share/Code' \
  --exclude='.vscode/' \
  \
  `# --- .config: exclude junk, keep KDE/app configs ---` \
  --exclude='.config/evolution' \
  --exclude='.config/gnome-session' \
  --exclude='.config/gnome-initial-setup-done' \
  --exclude='.config/goa-1.0' \
  --exclude='.config/gtk-3.0' \
  --exclude='.config/gtk-4.0' \
  --exclude='.config/ibus' \
  --exclude='.config/imsettings' \
  --exclude='.config/mozilla' \
  --exclude='.config/pulse' \
  --exclude='.config/session' \
  --exclude='.config/vlc' \
  --exclude='.config/abrt' \
  --exclude='.config/dconf' \
  --exclude='.config/libaccounts-glib' \
  --exclude='.config/org.gnome.Ptyxis' \
  --exclude='.config/nautilus' \
  --exclude='.config/freerdp' \
  --exclude='.config/.gsd-keyboard.settings-ported' \
  \
  `# --- .local/share: exclude large/expendable data ---` \
  --exclude='.local/share/containers' \
  --exclude='.local/share/flatpak' \
  --exclude='.local/share/gnome-shell' \
  --exclude='.local/share/gnome-settings-daemon' \
  --exclude='.local/share/gvfs-metadata' \
  --exclude='.local/share/evolution' \
  --exclude='.local/share/folks' \
  --exclude='.local/share/kactivitymanagerd' \
  --exclude='.local/share/klipper' \
  --exclude='.local/share/sounds' \
  --exclude='.local/share/vlc' \
  --exclude='.local/share/GitKrakenCLI' \
  --exclude='.local/share/gstreamer-1.0' \
  --exclude='.local/share/ibus-typing-booster' \
  --exclude='.local/share/nautilus' \
  --exclude='.local/share/org.gnome.Ptyxis' \
  --exclude='.local/share/recently-used.xbel' \
  --exclude='.local/share/recently-used.xbel.bak' \
  --exclude='.local/share/recently-used.xbel.tbcache' \
  --exclude='.local/share/remoteview' \
  --exclude='.local/share/baloo' \
  --exclude='.local/share/kwrite' \
  --exclude='.local/share/recently-used.xbel*' \
  \
  `# --- .local/state: all expendable runtime state ---` \
  --exclude='.local/state' \
  \
  `# --- Cache and trash ---` \
  --exclude='.cache' \
  --exclude='.local/share/Trash' \
  \
  `# --- Git objects (repo history, already on GitHub) ---` \
  --exclude='.git' \
  --exclude='node_modules/' \
  --exclude='.venv/' \
  --exclude='venv/' \
  --exclude='__pycache__/' \
  --exclude='*.pyc' \
  \
  "$SOURCE" "$DESTINATION" >> "$LOG" 2>&1

echo "Backup completed: $(date)" >> "$LOG"
```

```bash
chmod +x ~/scripts/nas-backup.sh
```

Test manually:
```bash
bash ~/scripts/nas-backup.sh
tail -f /var/log/nas-backup/nas-backup.log
```

### 6.4 Automate with systemd Timer

Create the service:
```bash
sudo nano /etc/systemd/system/nas-backup.service
```

```ini
[Unit]
Description=Backup ThinkPad to NAS

[Service]
Type=oneshot
ExecStart=/bin/bash /home/elikyals/scripts/nas-backup.sh
```

Create the timer:
```bash
sudo nano /etc/systemd/system/nas-backup.timer
```

```ini
[Unit]
Description=Run NAS backup weekly

[Timer]
OnCalendar=Sat *-*-* 12:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Enable:
```bash
sudo systemctl enable --now nas-backup.timer
```

Verify timer is active:
```bash
systemctl status nas-backup.timer
```

---

## Key File Locations

| File | Path |
|---|---|
| Backup script | `~/scripts/nas-backup.sh` |
| Backup log | `/var/log/nas-backup/nas-backup.log` |
| Logrotate config | `/etc/logrotate.d/nas-backup` |
| SMB credentials | `~/.smbcredentials` |
| fstab | `/etc/fstab` |
| NAS mount points | `/mnt/nas/archive`, `/mnt/nas/thinkpad-backups` |
| systemd service | `/etc/systemd/system/nas-backup.service` |
| systemd timer | `/etc/systemd/system/nas-backup.timer` |

---

## Troubleshooting

**Shares not mounting after reboot**
```bash
systemctl daemon-reload
sudo mount -a
```

**SMB connection refused**
- Check OMV VM is running in Proxmox
- Check Tailscale is connected on both ThinkPad and OMV
- Verify SMB is enabled in OMV web UI

**Existing files not visible on NAS share**
- Check shared folder relative path in OMV is set to `/` not a subfolder

**Backup taking too long**
- Check for large excluded directories still present on NAS from earlier runs
- Run: `du -sh /mnt/nas/thinkpad-backups/elikyals/*/` to identify large folders
- Delete anything that should be excluded

**rsync symlink errors (Operation not supported 95)**
- Normal on NTFS destination - use `--copy-links` flag (already in script)

