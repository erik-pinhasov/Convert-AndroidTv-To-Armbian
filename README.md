# 🛡️ Zero-Trust Home Infrastructure Stack (H618 ARM)

## Overview
This project documents the architectural conversion of a compromised, malware-infected Allwinner H618 Android TV Box into a secure, headless Armbian server. 

Rather than discarding the hardware, the internal storage was bypassed, and the device was re-provisioned as a containerized home infrastructure node. 

<img width="900" height="491" alt="1" src="https://github.com/user-attachments/assets/f8418af1-55e4-4b43-8f20-c76874b0ec5c" />

---

## 🏗️ Architecture & Stack
* **Base OS:** Armbian Noble (Ubuntu 24.04) headless server.
* **Network Security:** UFW, fail2ban, IPv6 Kernel Kill-Switch.
* **Zero-Trust VPN:** Tailscale with MagicDNS routing.
* **DNS/DHCP:** Pi-hole v6 (Ad-blocking) + Unbound (Recursive DNS).
* **Containerization:** Docker Engine with Compose.
* **Services:** Homepage (Dashboard), Filebrowser, UpSnap (WoL), Custom Web-Terminal (Root Shell via ttyd/nsenter).

---

## 💾 Hardware & Firmware
* **Hardware:** Allwinner H618 Android TV Box (e.g., T95, Vontar).
* **Storage:** 16GB-32GB High-Endurance MicroSD (SanDisk/Samsung).
* **Firmware Source:** [ophub/amlogic-s9xxx-armbian](https://github.com/ophub/amlogic-s9xxx-armbian/releases)
* **Target Image:** `Armbian_26.05.0_allwinner_vontar-h618_noble_6.18.29_server_2026.05.15.img.gz`
* **Download:** [Direct Link](https://github.com/ophub/amlogic-s9xxx-armbian/releases/download/Armbian_noble_arm64_server_2026.05/Armbian_26.05.0_allwinner_vontar-h618_noble_6.18.29_server_2026.05.15.img.gz)

### 🔒 Image Verification
This repository uses GitHub Actions to automatically run VirusTotal scans on the upstream image. Check the Releases page for public security audit links.

**Windows PowerShell Verification:**
```powershell
Get-FileHash .\Armbian_26.05.0_allwinner_vontar-h618_noble_6.18.29_server_2026.05.15.img.gz -Algorithm SHA256
```

---

## 🚀 Phase 1: OS Provisioning & Hardening

### Flashing & Booting
1. Flash the image using **Rufus** or **Balena Etcher**.
2. Boot from SD/USB. *Do not connect to the network until the Android OS is verified dead/bypassed.*
3. Default SSH Login: `root` / `1234`.

### Kernel-Level Network Hardening
To ensure absolute local DNS sinkholing and prevent IPv6 routing leaks, IPv6 is disabled at the kernel level.

```bash
echo "net.ipv6.conf.all.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
echo "net.ipv6.conf.lo.disable_ipv6 = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Security & Firewall (UFW)
Lock down inbound traffic and switch to SSH Key authentication.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install fail2ban ufw -y

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 53
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

---

## 🐳 Phase 2: Home Server Tools (Docker)
Services are isolated and managed via Docker Compose. The stack is optimized for low-RAM and low eMMC wear by utilizing temporary filesystems where necessary.

* **Filebrowser:** Shared storage folder on my home network to be accessed from any device.
* **Upsnap:** Turn ON/OFF devices configured and set on my home network.
* **Web-terminal:** Linux terminal to access the Armbian server from web.
* **Homepage:** Simple Web page to access all tools/apps running.

Create `docker-compose.yml`:

```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: always
    environment:
      - DOCKER_API_VERSION=1.40
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  web-terminal:
    image: alpine:latest
    container_name: web-terminal
    privileged: true
    pid: host
    ports:
      - "7681:7681"
    command: /bin/sh -c "apk add --no-cache ttyd util-linux && ttyd -W -c admin:ChangeThisPassword nsenter -t 1 -m -u -n -i bash"
    restart: always
    deploy:
      resources:
        limits:
          memory: 128M

  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: always
    ports:
      - "80:3000"
    volumes:
      - ./homepage/config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - HOMEPAGE_ALLOWED_HOSTS=*

  upsnap:
    image: seriousm4x/upsnap:latest
    container_name: upsnap
    network_mode: host
    restart: always
    volumes:
      - ./upsnap:/app/pb_data

  filebrowser:
    image: filebrowser/filebrowser:latest
    container_name: filebrowser
    restart: always
    user: "1000:1000"
    ports:
      - "8082:80"
    volumes:
      - ./shared:/srv
      - ./filebrowser:/database
    environment:
      - FB_DATABASE=/database/filebrowser.db
```
Deploy the stack: `docker compose up -d`

---

## 🕸️ Phase 3: Zero-Trust Remote Access (Tailscale)
No inbound ports are forwarded on the edge router. Access is handled via a WireGuard mesh network.
In simple words - Tailscale create secured VPN and used (among other things) to access my home network devices - from anywhere in the world.

```bash
curl -fsSL [https://tailscale.com/install.sh](https://tailscale.com/install.sh) | sh
sudo tailscale up --accept-dns=false
```

---

## 🕳️ Phase 4: Pi-hole v6 & DNS Configuration
Pi-hole is my network-wide ad blocker, running on my Armbian home server as a DNS sinkhole.
```bash
curl -sSL [https://install.pi-hole.net](https://install.pi-hole.net) | bash
```

### Strip IPv6 from Pi-hole v6
Because the OS kernel drops IPv6, Pi-hole must be configured to stop resolving it.

```bash
sudo pihole-FTL --config dns.reply.host.IPv6 false
sudo pihole-FTL --config dns.reply.blocking.IPv6 false
sudo pihole-FTL --config resolver.resolveIPv6 false
sudo systemctl restart pihole-FTL
```
Edit `/etc/pihole/pihole.toml` to remove any IPv6 upstreams from the `upstreams = []` array.

---

## 🖥️ Phase 5: Homepage Dashboard Configuration
Configure Homepage to display the deployed containers by editing the `services.yaml` file.

Open the configuration file:
```bash
nano ./homepage/config/services.yaml
```

Paste the following YAML (Replace `<SERVER_IP>` with your Armbian node's local IP address):

```yaml
- Home Page:
    - Pi-hole:
        icon: pihole.png
        href: http://<SERVER_IP>/admin
        description: Network DNS
    - Tailscale:
        icon: tailscale.png
        href: https://login.tailscale.com/admin/machines
    - Web-Terminal:
        icon: terminal.png
        href: http://<SERVER_IP>:7681
        description: Host Shell Access
        target: _self
    - UpSnap:
        icon: upsnap.png
        href: http://<SERVER_IP>:8090
        description: Wake-On-LAN Controller
    - Filebrowser:
        icon: filebrowser.png
        href: http://<SERVER_IP>:8082
        description: Shared Network Drive
```

---

## 🔄 Phase 6: Automated Maintenance
Automated backup script with a 7-day retention policy to prevent storage exhaustion.

Create `~/pihole-backup.sh`:

```bash
#!/bin/bash
BACKUP_DIR="$HOME/backups"
MAX_BACKUPS=7

mkdir -p "$BACKUP_DIR"

FILE="$BACKUP_DIR/pihole-$(date +%Y%m%d-%H%M).tar.gz"
pihole -a -t "$FILE"

COUNT=$(ls $BACKUP_DIR | wc -l)
if [ "$COUNT" -gt "$MAX_BACKUPS" ]; then
  ls -t $BACKUP_DIR | tail -n +$((MAX_BACKUPS+1)) | xargs rm --
fi
```
Make executable and add to cron:
```bash
chmod +x ~/pihole-backup.sh
crontab -e
# Add line: 0 3 * * * /home/username/pihole-backup.sh
```
