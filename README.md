# Turning a Malware-Infected H618 Android TV Box Into a Secure Armbian + Pi-hole Server

# Overview
I ordered a cheap Android TV box 'T95Max' and found it came preloaded with malware.
Instead of throwing it away:

- Wiped Android completely  
- Installed **Armbian**  
- Turned the device into a **network-wide ad blocker (Pi-hole)**  
- Added hardened security  
- Configured DHCP  
- Added backups  
- Installed **Unbound** for full recursive DNS  
- Solved boot, USB, and network issues  

---

# Hardware & Software Requirements

### Hardware
- Allwinner **H618** Android TV box  
- MicroSD (16–32GB recommended, quality matters)  
- HDMI cable & monitor
- Ethernet cable
- Router with access to DNS settings  

### Software
- Balena Etcher / Rufus  
- Armbian image for H618  
- SSH client

---

# Identify Your Android TV Box (H618)

Android boxes rarely match their advertised specs.  
Signs you have an **H618**:

- CPU reported as **Allwinner H618** (check in AIDA64 / CPU-Z)
- Board name containing:  
  `H618`, `T95-H618`, `TX6S-H618`, `Q96-H618`, etc.

> ⚠️ **Choosing the correct DTB (Device Tree Blob)** is critical for boot success.

---

# Clean the Device (Malware Removal Strategy)

Android malware survives factory resets, so you must:

1. **Boot Armbian from SD**  
2. Never boot Android again  
3. Eventually remove or overwrite internal eMMC/NAND  

*Do NOT connect the tv box to your home network until you verified Android OS removed from eMMC!

This guide uses SD boot only (safe, no need to wipe internal NAND).

---

# Download the Correct Armbian Image

### Recommended image source  

Download:

- `The image I used:` https://github.com/NickAlilovic/build/releases/download/20241125/Armbian-20241125-unofficial_24.11.0-trunk_Transpeed-8k618-t_bookworm_edge_6.10.10_xfce_desktop.tar.gz
- You can download it from my current repo releases
- `Extract it twice. (File type:Disk Image File / extension:.img)`
---

# Flash Armbian to SD Card

Use **Rufus** or **Balena Etcher**  
```
Select image → Select SD → Flash
```

If SD refuses to boot, use the troubleshooting section below.

---

# Fix SD/USB Boot Failures

Typical H618 boot problems:

### ❌ *Stuck on Boot Logo*
- Wrong DTB  
- Damaged SD  
- Armbian image not for H618  

### ❌ *Boots only from USB, not SD*
- SD reader on board is unstable  
- Try a different SD brand (SanDisk recommended)  
- Boot via USB → Works → You can stay like that  

### ❌ *Kernel panic*
Solution:  
Try DTBs one by one:

```
/boot/dtb/allwinner/sun50i-h618*
```

Edit `armbianEnv.txt`:

```
fdtfile=sun50i-h618-t95.dtb
```

---

# First Boot & Serial/HDMI Notes

Boot with keyboard/HDMI or SSH.

Default login:

```
root / 1234
```

You will be forced to:

- Change password  
- Create a new user  
- Update packages  

---

# Secure the OS (Armbian Hardening)

Run:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install fail2ban ufw -y
sudo ufw allow 22/tcp
sudo ufw allow 53/tcp
sudo ufw allow 53/udp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Disable root SSH:

```
sudo nano /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
```

Restart SSH:

```
sudo systemctl restart ssh
```

---

# Install Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

Set interface to **eth0**, DNS: `1.1.1.1` or later Unbound.

Access admin:

```
http://pi.hole/admin
```

---

# Make Pi‑hole Survive Reboots

Enable service:

```bash
sudo systemctl enable pihole-FTL
sudo systemctl enable lighttpd
```

---

# Install Pi-hole DHCP Correctly

Before enabling DHCP:

1. **Disable your router's DHCP first**  
2. Set Pi‑hole static IP (open default gateway IP to access router panel)
3. Enable Pi-hole DHCP range  

---

# Install Unbound (Full Local DNS Resolver)

Unbound = privacy + no upstream DNS.

Install:

```bash
sudo apt install unbound -y
```

Config:

```
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```

Paste standard config (official Pi-hole docs).

Restart:

```
sudo systemctl restart unbound
```

Set Pi-hole → Settings → DNS:

```
127.0.0.1#5335
```

---

# Automated Pi-hole Backups + Rotation

Create script:

```
/home/USER/pihole-backup.sh
```

Content:

```bash
#!/bin/bash
BACKUP_DIR="/home/USER/backups"
MAX_BACKUPS=7

mkdir -p "$BACKUP_DIR"

FILE="$BACKUP_DIR/pihole-$(date +%Y%m%d-%H%M).tar.gz"
pihole -a backup > "$FILE"

# Rotate
COUNT=$(ls $BACKUP_DIR | wc -l)
if [ "$COUNT" -gt "$MAX_BACKUPS" ]; then
  ls -t $BACKUP_DIR | tail -n +$((MAX_BACKUPS+1)) | xargs rm --
fi
```

Make executable:

```
chmod +x ~/pihole-backup.sh
```

Add cron:

```
crontab -e
0 3 * * * /home/USER/pihole-backup.sh
```

---
