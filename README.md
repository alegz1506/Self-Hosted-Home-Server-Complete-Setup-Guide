# 🏠 Self-Hosted Home Server Guide
### Debian + Docker | Jellyfin · Immich · Nextcloud · NPM · Tailscale
 
This is a personal guide documenting how I actually set up my home server from scratch — the process, the config files, and every real error I ran into along the way. It's not a theoretical walkthrough. Everything written here came from hands-on experience.
 
---
 
## 📋 Table of Contents
 
1. [Hardware](#-hardware)
2. [Installing Debian](#-installing-debian)
3. [Getting Online & SSH Access](#-getting-online--ssh-access)
4. [Permanent HDD Mounting](#-permanent-hdd-mounting)
5. [Firewall (UFW)](#-firewall-ufw)
6. [Docker Setup & Folder Structure](#-docker-setup--folder-structure)
7. [Networking — DDNS & Reverse Proxy](#-networking--ddns--reverse-proxy)
8. [App Stack](#-app-stack)
   - [Jellyfin](#jellyfin)
   - [Immich](#immich)
   - [Nextcloud](#nextcloud)
   - [Gluetun](#gluetun)
   - [Homepage](#homepage)
   - [Anubis](#anubis)
   - [Portainer](#portainer)
   - [Uptime Kuma](#uptime-kuma)
   - [Watchtower](#watchtower)
   - [Autoheal](#autoheal)
9. [Secure Remote Access — Tailscale](#-secure-remote-access--tailscale)
10. [Security Hardening](#-security-hardening)
11. [Troubleshooting](#-troubleshooting)
12. [Quick Reference](#-quick-reference)
 
---
 
## 🖥️ Hardware
 
My server is built from repurposed desktop parts, which works perfectly well. The things that genuinely matter:
 
- **CPU** — A 4–8 core desktop chip is plenty. I'm running an Intel i5.
- **RAM** — 8 GB is the bare minimum. If you're running Immich with machine learning enabled, plan for 16 GB+. The ML container alone will comfortably eat 4–6 GB, and that's before Postgres, Redis, and everything else.
- **GPU** — A dedicated GPU matters specifically for hardware transcoding in Jellyfin. Without one, your CPU will max out the moment someone tries to stream a 4K file remotely. I use an AMD Radeon RX 470, which works well with VAAPI.
- **Boot drive** — An SSD (250 GB+) for the OS, Docker configs, and app databases.
- **Mass storage** — A spinning HDD for media. Make sure it's **CMR** (Conventional Magnetic Recording), not SMR. SMR drives have terrible sustained write performance and will cause headaches under load. Always verify via the manufacturer's spec sheet.
- **Network** — Wired Ethernet only. No exceptions for a server.
 
---
 
## 🐧 Installing Debian
 
A minimal headless Debian install — no desktop environment.
 
1. Download the Debian netinstall ISO from [debian.org](https://www.debian.org/distrib/) and flash it to a USB.
2. During installation, **uncheck** `Debian desktop environment` and **check** `SSH server` and `standard system utilities`.
3. Create a non-root user during setup. You'll use this for everything.
 
After first boot, `sudo` isn't installed by default on Debian minimal. Fix that immediately:
 
```bash
su -
apt update && apt install -y sudo
usermod -aG sudo youruser
exit
```
 
Log out and back in for the group change to apply.
 
---
 
## 🌐 Getting Online & SSH Access
 
### The HDMI trap
 
The first thing I tried was plugging my laptop into the server via HDMI to use it as a display. This doesn't work. The HDMI port on a laptop is **output-only** — it sends video, it can't receive it. Plugging two outputs together does nothing. The correct way to control a headless server is SSH.
 
### Finding the server's IP
 
Since SSH was enabled during installation, the service is already running. You just need the IP. The easiest method is logging into your router's admin panel (usually `192.168.1.1` or `192.168.0.1`) and looking for "Connected Devices" or "DHCP Clients". Look for a device named `debian`.
 
Or scan from another machine on the same network:
 
```bash
sudo pacman -S nmap    # Arch
sudo apt install nmap  # Debian/Ubuntu
 
nmap -sn 192.168.1.0/24
```
 
Once you have it:
 
```bash
ssh youruser@192.168.1.x
```
 
### Static IP
 
For a server, the IP needs to stay constant. The cleanest approach is a **DHCP reservation on your router** — find your server's MAC address with `ip link show`, then bind it to a fixed IP in the router's admin panel.
 
Alternatively, set it directly in Debian:
 
```bash
sudo nano /etc/network/interfaces
```
 
Replace the DHCP entry for your interface (e.g. `enp3s0`) with:
 
```
auto enp3s0
iface enp3s0 inet static
    address 192.168.1.x
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```
 
```bash
sudo systemctl restart networking
```
 
### Essential packages
 
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano htop ncdu tmux \
  ca-certificates gnupg lsb-release ufw
```
 
---
 
## 💾 Permanent HDD Mounting
 
Debian doesn't auto-mount HDDs on reboot. This must be hardcoded in `/etc/fstab` — if you skip it, the drive simply isn't there after a restart.
 
### Find your drive
 
```bash
sudo fdisk -l
# Look for the large drive, e.g. /dev/sdb
```
 
### Format it (first time only)
 
> ⚠️ **Skip this if the drive already has data you want to keep.**
 
```bash
sudo mkfs.ext4 /dev/sdb1
```
 
### Get the UUID
 
Always use the UUID, not the device path. Device names like `/dev/sdb1` can change between reboots; UUIDs never do.
 
```bash
sudo blkid /dev/sdb1
# UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="ext4"
```
 
### Mount point and fstab entry
 
```bash
sudo mkdir -p /mnt/media
sudo nano /etc/fstab
```
 
Add at the very bottom:
 
```
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /mnt/media  ext4  defaults  0  2
```
 
Test without rebooting:
 
```bash
sudo mount -a
df -h | grep media
```
 
Fix ownership:
 
```bash
sudo chown -R youruser:youruser /mnt/media
```
 
---
 
## 🔥 Firewall (UFW)
 
Two things will silently break things if you don't handle them:
 
1. Docker uses internal bridge networks (`172.16.0.0/12`) for container-to-container traffic. Block that range and your containers can't talk to each other.
2. Tailscale uses its own interface (`tailscale0`) which UFW doesn't know about by default.
 
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
 
sudo ufw allow 80/tcp       # HTTP (for NPM)
sudo ufw allow 443/tcp      # HTTPS (for NPM)
sudo ufw allow ssh          # Port 22
 
# Your local home network
sudo ufw allow from 192.168.1.0/24
 
# Docker's internal bridge — required for container routing
sudo ufw allow from 172.16.0.0/12
 
# Tailscale interface
sudo ufw allow in on tailscale0
 
sudo ufw enable
sudo ufw status
```
 
> Port 81 (NPM admin) is deliberately not opened here. Access it only through Tailscale.
 
---
 
## 🐳 Docker Setup & Folder Structure
 
### Install Docker
 
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker youruser
newgrp docker
```
 
Verify:
 
```bash
docker run hello-world
docker compose version
```
 
If `docker compose` is missing:
 
```bash
sudo apt install docker-compose-plugin
```
 
### The folder structure
 
The single most important architectural decision. **All Docker configs stay on the SSD. All media lives on the HDD.** The reason is hardlinks — when files are on the same drive, you can hardlink them instead of copying, which saves enormous disk space. If they're on different drives, hardlinks are impossible.
 
```
/mnt/media/                ← HDD
└── data/
    ├── media/
    │   ├── movies/
    │   ├── tv/
    │   └── music/
    └── photos/            ← Immich library
 
~/docker/                  ← SSD (small config files)
├── jellyfin/
├── immich/
├── nextcloud/
├── npm/
└── ...
```
 
Create it:
 
```bash
sudo mkdir -p /mnt/media/data/media/{movies,tv,music}
sudo mkdir -p /mnt/media/data/photos
mkdir -p ~/docker/{jellyfin,immich,nextcloud,npm,duckdns,gluetun,homepage,anubis,portainer,watchtower,autoheal,uptime-kuma}
sudo chown -R youruser:youruser /mnt/media
```
 
### Find your PUID and PGID
 
```bash
id youruser
# uid=1000(youruser) gid=1000(youruser)
```
 
---
 
## 🌍 Networking — DDNS & Reverse Proxy
 
Your home IP changes periodically. DuckDNS keeps a domain name pointing at your current IP. NPM routes incoming traffic to the right container and handles SSL.
 
**Router setup first:** Forward ports **80** and **443** to your server's static IP. Never forward port 81.
 
### DuckDNS
 
Sign up at [duckdns.org](https://www.duckdns.org), create a subdomain, copy your token.
 
**`~/docker/duckdns/docker-compose.yml`:**
```yaml
services:
  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Your/Timezone
      - SUBDOMAINS=yoursubdomain
      - TOKEN=your-duckdns-token
      - LOG_FILE=true
    volumes:
      - ./config:/config
    restart: unless-stopped
```
 
```bash
cd ~/docker/duckdns && docker compose up -d
docker compose logs -f
# Should say "Your IP was set"
```
 
### Nginx Proxy Manager (NPM)
 
**`~/docker/npm/docker-compose.yml`:**
```yaml
services:
  app:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      - TZ=Your/Timezone
```
 
```bash
cd ~/docker/npm && docker compose up -d
```
 
Admin UI: `http://192.168.1.x:81`
Default login: `admin@example.com` / `changeme` — change both on first login.
 
**Creating a proxy host with SSL:**
1. Hosts → Proxy Hosts → Add Proxy Host
2. Domain: `jellyfin.yoursubdomain.duckdns.org`
3. Scheme: `http`, Forward Host: your server IP, Port: `8096`
4. SSL tab → Request a new SSL Certificate → Force SSL → Save
 
**Restricting management apps to local network only:**
 
You probably don't need management apps (like Portainer) accessible from the internet. In NPM, go to the **Access Lists** tab, create a list called "Local Only", add `allow 192.168.1.0/24` and `deny all`. Then edit any proxy host you want to lock down and set its Access List to "Local Only". Someone trying to reach that URL from outside your network will get a `403 Forbidden`.
 
---
 
## 🚀 App Stack
 
### Jellyfin
 
Your personal media server.
 
**`~/docker/jellyfin/docker-compose.yml`:**
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Your/Timezone
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /mnt/media/data/media:/data/media:ro
    devices:
      - /dev/dri:/dev/dri       # GPU passthrough for hardware transcoding
    group_add:
      - "render"                # Required for AMD/Intel GPU access
```
 
The `group_add: render` entry is important. Without it, the container can see `/dev/dri` but can't actually use the GPU. If `render` doesn't work, find the numeric GID and use that instead:
 
```bash
stat -c '%g' /dev/dri/renderD128
# Usually 104 or 108
```
 
**Enabling hardware transcoding:**
Dashboard → Playback → Transcoding → Hardware Acceleration: **VAAPI** → Hardware encoding device: `/dev/dri/renderD128` → Save.
 
To verify it's working: play something, go to Dashboard → Active Streams. You should see `(hw)` next to the codec name. If you see `(sw)`, the GPU isn't being used.
 
---
 
### Immich
 
Self-hosted Google Photos with AI facial recognition and mobile backup.
 
> ⚠️ Immich updates constantly and has frequent breaking changes. Always fetch the official compose file — don't copy it from somewhere else.
 
```bash
cd ~/docker/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```
 
Edit `.env`:
```env
UPLOAD_LOCATION=/mnt/media/data/photos
DB_PASSWORD=use_a_strong_password_here
TZ=Your/Timezone
```
 
```bash
docker compose up -d
```
 
This starts five containers: the main server, microservices, machine learning, Redis, and Postgres. The ML container handles facial recognition and CLIP search — expect it to use several GB of RAM and take 30–60 minutes to index a large library the first time.
 
**Importing from Google Takeout:**
After a large import (e.g., a Google Takeout dump), you'll often see broken thumbnails with "Error loading image". This is normal — the thumbnail generation worker got overwhelmed by the queue. Fix it by going to Administration (gear icon) → Jobs, and manually triggering the **Generate Thumbnails** and **Machine Learning** jobs. Let them run to completion.
 
**Updating Immich** (always read the release notes first):
```bash
docker compose pull && docker compose up -d
```
 
---
 
### Nextcloud
 
Self-hosted file sync — a personal Dropbox.
 
> ⚠️ Nextcloud runs as its internal `www-data` user (UID 33), not your system user. You must give that user ownership of the data folder or Nextcloud will refuse to start.
 
```bash
sudo mkdir -p /mnt/media/data/nextcloud
sudo chown -R 33:33 /mnt/media/data/nextcloud
```
 
**`~/docker/nextcloud/docker-compose.yml`:**
```yaml
services:
  nextcloud-db:
    image: mariadb:10.6
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      - ./db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=strong_root_password
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=strong_db_password
 
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - '9001:80'
    volumes:
      - ./config:/var/www/html
      - /mnt/media/data/nextcloud:/var/www/html/data
    environment:
      - MYSQL_HOST=nextcloud-db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=strong_db_password
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.yoursubdomain.duckdns.org
      - TRUSTED_PROXIES=192.168.1.x
      - OVERWRITEPROTOCOL=https
      - TZ=Your/Timezone
    depends_on:
      - nextcloud-db
```
 
The `TRUSTED_PROXIES` and `OVERWRITEPROTOCOL` lines are important — without them, Nextcloud doesn't know it's sitting behind a reverse proxy and will generate broken HTTP links.
 
---
 
### Gluetun
 
A VPN container. You can route other containers through it by setting `network_mode: service:gluetun` in their compose service instead of defining their own network.
 
**`~/docker/gluetun/docker-compose.yml`:**
```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=mullvad      # Change to your provider
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=your_key
      - WIREGUARD_ADDRESSES=10.x.x.x/32
      - TZ=Your/Timezone
```
 
---
 
### Homepage
 
A dashboard with live widgets for all your services. Replaced Heimdall after outgrowing it.
 
**`~/docker/homepage/docker-compose.yml`:**
```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - '3000:3000'
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /mnt/media:/mnt/media:ro        # So Homepage can show disk usage
    environment:
      - TZ=Your/Timezone
      - PUID=1000
      - PGID=1000
```
 
**`~/docker/homepage/config/services.yaml`:**
```yaml
- Media & Photos:
    - Jellyfin:
        icon: jellyfin.png
        href: https://jellyfin.yoursubdomain.duckdns.org   # Use your DuckDNS link here
        description: Personal Media Server
        widget:
          type: jellyfin
          url: http://192.168.1.x:8096   # Use local IP for the API widget
          key: your_jellyfin_api_key
 
    - Immich:
        icon: immich.png
        href: https://immich.yoursubdomain.duckdns.org
        description: Photo Backup
        widget:
          type: immich
          url: http://192.168.1.x:2283
          key: your_immich_api_key
 
- Personal Cloud:
    - Nextcloud:
        icon: nextcloud.png
        href: https://cloud.yoursubdomain.duckdns.org
        description: Private File Storage
 
- Server Management:
    - Portainer:
        icon: portainer.png
        href: http://192.168.1.x:9000    # Keep local-IP only, not exposed publicly
        description: Docker Management
```
 
> **href vs widget URL:** Use your DuckDNS HTTPS links for `href` (what you click when you're away from home). Use the local IP for widget `url` (what Homepage uses to pull live stats in the background — no point routing through the internet for something on the same machine).
 
**`~/docker/homepage/config/widgets.yaml`:**
```yaml
- resources:
    cpu: true
    cputemp: true
    memory: true
    disk:
      - /
      - /mnt/media     # Shows both OS drive and media HDD
 
- datetime:
    text_size: xl
    format:
      dateStyle: short
      timeStyle: short
```
 
**`~/docker/homepage/config/docker.yaml`:**
```yaml
my-docker:
  socket: /var/run/docker.sock
```
 
Get API keys: Jellyfin → Dashboard → API Keys. Immich → User Settings → API Keys.
 
---
 
### Anubis
 
A bot protection container. Acts as a proof-of-work challenge layer that sits in front of Homepage and blocks automated scrapers and bots.
 
**`~/docker/anubis/docker-compose.yml`:**
```yaml
services:
  anubis-homepage:
    image: ghcr.io/techarohq/anubis:latest
    container_name: anubis-homepage
    restart: unless-stopped
    ports:
      - '3001:3001'
    environment:
      - TARGET=http://192.168.1.x:3000   # Points to Homepage
      - TZ=Your/Timezone
```
 
In NPM, point your Homepage proxy host to port `3001` (Anubis) instead of `3000` (Homepage directly). Legitimate browsers pass the challenge instantly; bots get blocked.
 
---
 
### Portainer
 
A web UI for managing Docker containers, viewing logs, editing compose files, and inspecting volumes/networks without using the terminal.
 
**`~/docker/portainer/docker-compose.yml`:**
```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - '9000:9000'
      - '9443:9443'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
    environment:
      - TZ=Your/Timezone
```
 
> ⚠️ Never expose port 9000 publicly via NPM. Portainer gives full Docker control. Access it only through Tailscale or your local IP.
 
---
 
### Uptime Kuma
 
Monitors whether your services are actually reachable and alerts you if something goes down. Replaced a brief Netdata experiment (Netdata's virtual memory graphs caused confusion and it was removed).
 
**`~/docker/uptime-kuma/docker-compose.yml`:**
```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - '3002:3001'
    volumes:
      - ./data:/app/data
    environment:
      - TZ=Your/Timezone
```
 
After setup, add monitors for each of your services. For Immich, the correct ping endpoint is:
 
```
http://192.168.1.x:2283/api/server/ping
```
 
Note: the old path was `/api/server-info/ping` — Immich removed that endpoint in a major update. If your monitor shows a `404`, this is why. Update it to `/api/server/ping`.
 
---
 
### Watchtower
 
Checks for updated Docker images nightly and automatically pulls and restarts containers.
 
**`~/docker/watchtower/docker-compose.yml`:**
```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Your/Timezone
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_SCHEDULE=0 0 3 * * *    # 3:00 AM daily
      - WATCHTOWER_INCLUDE_RESTARTING=true
```
 
To exclude a container from auto-updates:
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```
 
**Important:** Never put Autoheal on database containers (Postgres, MariaDB, Redis). If Autoheal restarts a database mid-write, you risk corrupting your data. Let databases take as long as they need.
 
---
 
### Autoheal
 
Watchtower creates a race condition: when it updates multiple containers at once, apps that depend on a database or network can boot before their dependency is ready and freeze. Autoheal monitors containers with healthchecks and restarts any that go unhealthy.
 
**`~/docker/autoheal/docker-compose.yml`:**
```yaml
services:
  autoheal:
    image: willfarrell/autoheal:latest
    container_name: autoheal
    restart: unless-stopped
    environment:
      - AUTOHEAL_CONTAINER_LABEL=autoheal   # Only watches containers you tag
      - AUTOHEAL_INTERVAL=5
      - TZ=Your/Timezone
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
 
To enable monitoring on a specific container, add a `healthcheck` and a label to its compose service:
 
```yaml
labels:
  - "autoheal=true"
healthcheck:
  test: curl -f http://localhost:PORT/ping || exit 1
  interval: 60s
  timeout: 10s
  retries: 3
```
 
Replace `PORT` with that app's internal port. For Immich: `2283`, for NPM admin: `81`, for Jellyfin: `8096/Health`.
 
**Apps that need Autoheal:** NPM, Nextcloud, Immich, Gluetun
(anything that depends on a database or VPN tunnel).
 
**Apps that should NOT have Autoheal:** Databases (Postgres, MariaDB, Redis), DuckDNS, Watchtower itself. Never watch the watchdog or the databases.
 
---
 
## 🔐 Secure Remote Access — Tailscale
 
Tailscale creates a WireGuard-based encrypted mesh network between your devices. The purpose here is a secure management backdoor — SSH and Portainer access from anywhere — without exposing those ports to the open internet.
 
**What went wrong the first time:** A standard Tailscale install overwrites `/etc/resolv.conf` via a feature called MagicDNS. This silently breaks NPM and DuckDNS. The DuckDNS container stops updating, Nginx can't resolve domains, and everything goes dark. This happened, Tailscale was uninstalled, and then re-installed with the correct flags.
 
### Install and start (muzzled)
 
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --accept-dns=false --accept-routes=false
```
 
`--accept-dns=false` is the critical flag. It tells Tailscale not to touch DNS. Without it, MagicDNS overwrites your resolver and breaks NPM.
 
### Disable MagicDNS in the web dashboard
 
Log into [login.tailscale.com](https://login.tailscale.com) → DNS → disable **MagicDNS**. This is separate from the command-line flag and also needs to be done.
 
### Enable on boot
 
```bash
sudo systemctl enable tailscaled
```
 
If Tailscale ever causes DNS issues again, the universal fix is:
 
```bash
sudo rm -f /etc/resolv.conf
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
 
### Remote access
 
Once your other devices are connected to the same Tailscale network:
 
```bash
ssh youruser@100.x.x.x          # SSH from anywhere
http://100.x.x.x:9000           # Portainer
http://100.x.x.x:81             # NPM admin
http://100.x.x.x:8096           # Jellyfin (if DuckDNS is unreachable)
```
 
**Why TCP forwarding over SSH is no longer needed:** Before Tailscale, accessing private dashboards remotely required SSH port forwarding. With Tailscale handling routing at the network level, you don't need that at all — which is also why `AllowTcpForwarding no` in SSH is safe.
 
---
 
## 🔒 Security Hardening
 
### SSH key authentication
 
Do this before tightening anything else. Disable password auth before keys are working and you'll lock yourself out.
 
```bash
# On your LOCAL machine, not the server:
ssh-keygen -t ed25519
ssh-copy-id youruser@192.168.1.x
 
# Test key login works before proceeding
ssh youruser@192.168.1.x
```
 
### SSH hardening
 
Add this block to the **bottom** of `/etc/ssh/sshd_config`. Settings at the bottom override conflicting defaults higher in the file.
 
```bash
sudo nano /etc/ssh/sshd_config
```
 
```
PermitRootLogin no
PasswordAuthentication no
 
# Security hardening (Lynis recommendations)
AllowTcpForwarding no
AllowAgentForwarding no
X11Forwarding no
MaxAuthTries 3
MaxSessions 2
ClientAliveCountMax 2
TCPKeepAlive no
LogLevel VERBOSE
```
 
These are all safe with a Tailscale setup. `AllowTcpForwarding no` is fine because you're using Tailscale for remote access rather than SSH tunneling. `X11Forwarding no` is irrelevant on a headless server.
 
```bash
sudo systemctl restart sshd
```
 
### Security audit (Lynis)
 
Running Lynis is worth doing once just to confirm nothing is obviously wrong:
 
```bash
sudo apt install lynis clamav rkhunter -y
sudo lynis audit system
```
 
A score around **66** on a home server with 15 Docker containers is genuinely good. Most of Lynis's suggestions are enterprise-level overkill (separate `/var` partitions, disabling USB, process auditing that would fill your SSD). The ones worth acting on are the SSH tweaks above.
 
Once the audit is done, remove the scanners — they're not needed running permanently:
 
```bash
sudo apt remove --purge rkhunter lynis clamav clamav-freshclam -y
sudo apt autoremove -y
```
 
### Unattended security upgrades
 
Makes Debian automatically apply critical security patches nightly. Only security fixes, not feature updates, so it won't break Docker apps.
 
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
# A prompt will appear — select Yes
```
 
### Quality-of-life packages
 
```bash
sudo apt install needrestart apt-listbugs -y
```
 
- **needrestart** — after `apt upgrade`, tells you if a reboot is needed.
- **apt-listbugs** — checks Debian's bug database before installing an update. Silent by default; only speaks up when something is actually wrong.
 
---
 
## 🚨 Troubleshooting
 
### Storage fills up — hardlinks and duplicates
 
If disk usage grows way faster than expected, files are being physically copied instead of hardlinked. This happens when your media and Docker data are on different drives — which is exactly why the unified folder structure matters.
 
Run `jdupes` inside `tmux` so it survives if your SSH session drops:
 
```bash
sudo apt install jdupes -y
 
tmux new -s dedup
 
# Scan first (no changes yet)
sudo jdupes -r -S /mnt/media/data/
 
# Convert duplicates to hardlinks once you're happy with the output
sudo jdupes -r -L /mnt/media/data/
```
 
> ⚠️ Don't close your SSH window while this runs. The process is tied to your session — closing it kills jdupes mid-scan. If you need to disconnect, use `tmux` or prefix the command with `nohup` and `&`.
 
---
 
### `docker system df` shows misleading numbers
 
`docker system df` regularly shows enormous "reclaimable" space that doesn't exist — it double-counts shared image layers. Don't rely on it. Use `ncdu` instead:
 
```bash
sudo apt install ncdu -y
sudo ncdu /
```
 
Common actual culprits:
- Immich ML model cache inside its volume
- Jellyfin transcoding leftovers in `~/docker/jellyfin/cache/` — `.ts` files from interrupted streams, safe to delete when Jellyfin is stopped
- System logs:
  ```bash
  sudo journalctl --disk-usage
  sudo journalctl --vacuum-size=200M
  ```
 
---
 
### Container stuck on "Starting" in Portainer
 
**Symptom:** App works fine in the browser but Portainer shows "Starting" forever.
 
**Cause:** The `healthcheck` is pinging an endpoint that no longer exists. Immich is the main example — it moved its server port from `3001` to `2283` in a major update, and also changed the API path from `/api/server-info/ping` to `/api/server/ping`.
 
Diagnose:
```bash
docker inspect --format "{{json .State.Health.Log}}" container_name | python3 -m json.tool
```
 
Fix: update the healthcheck URL in `docker-compose.yml`, then:
```bash
docker compose down && docker compose up -d
```
 
---
 
### Watchtower crashes after `docker system prune`
 
**Symptom:** After `docker system prune -a`, a container won't start:
```
failed to set up container networking: network not found
```
 
**Cause:** `docker system prune -a` is more aggressive than it looks — it deletes all unused Docker networks, which can include the one a container was attached to.
 
**Fix:** Recreate the compose stack (this also recreates the missing network):
```bash
cd ~/docker/affected_container
docker compose down && docker compose up -d
```
 
---
 
### DuckDNS stops working after installing/reinstalling Tailscale
 
**Cause:** Tailscale's MagicDNS feature overwrites `/etc/resolv.conf`, which kills DNS resolution on the server. NPM and DuckDNS stop working.
 
**Fix:**
```bash
sudo rm -f /etc/resolv.conf
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
 
Then always start Tailscale with `--accept-dns=false` to prevent it happening again.
 
---
 
### NPM SSL certificate won't renew
 
Let's Encrypt certificates expire after 90 days. NPM handles renewal automatically, but needs port 80 reachable from the internet. If renewal fails:
 
1. Verify your domain resolves to your current public IP — use [dnschecker.org](https://dnschecker.org) from outside your home network
2. Confirm port 80 is still forwarded on your router
3. Check NPM logs: `docker logs npm 2>&1 | grep -i "renew\|error\|cert"`
4. Force renewal: NPM Admin → SSL Certificates → edit the cert → Save
 
---
 
## 📦 Quick Reference
 
| Service | Local URL | Port | Expose to Internet? |
|---------|-----------|------|---------------------|
| NPM Admin | `http://server:81` | 81 | ❌ Tailscale only |
| Jellyfin | `http://server:8096` | 8096 | ✅ via NPM |
| Immich | `http://server:2283` | 2283 | ✅ via NPM |
| Nextcloud | `http://server:9001` | 9001 | ✅ via NPM |
| Homepage | `http://server:3000` | 3000 | ✅ via Anubis → NPM |
| Anubis | `http://server:3001` | 3001 | ✅ via NPM (fronts Homepage) |
| Portainer | `http://server:9000` | 9000 | ❌ Tailscale only |
| Uptime Kuma | `http://server:3002` | 3002 | ❌ Tailscale only |
| Gluetun | Internal only | — | ❌ |
| DuckDNS | Internal only | — | ❌ |
 
---
 
## 🔄 Useful Commands
 
```bash
# See all running containers
docker ps
 
# Follow logs live
docker logs -f container_name
 
# Restart a container
docker restart container_name
 
# Update and redeploy a stack
cd ~/docker/app_name && docker compose pull && docker compose up -d
 
# Remove a container and its folder permanently
cd ~/docker/app_name
docker compose down
cd ~
sudo rm -rf ~/docker/app_name
docker image prune -a    # Clean up leftover images
 
# Remove orphaned containers after editing a compose file
docker compose up -d --remove-orphans
 
# Real disk usage
sudo ncdu /mnt/media
 
# Clean up dangling images and stopped containers (safe)
docker system prune
 
# Check HDD health
sudo apt install smartmontools -y
sudo smartctl -a /dev/sdb
 
# System resource usage
htop
```
 
---
 
## 📝 A Note on This Guide
 
Everything documented here — the configs, the specific steps, and especially the troubleshooting entries — comes from actually building and running this setup. The errors aren't things I researched; they're things I ran into personally and had to debug. The Tailscale DNS issue, the Immich port change breaking healthchecks, the jdupes scan sitting at 99% while wondering if it was safe to close the terminal — all of it happened. If something reads oddly specific, that's why.
 
---
 

*Built with ❤️ on Debian + Docker. Contributions welcome.*
# 🏠 Self-Hosted Home Server Guide
### Debian + Docker | Jellyfin · Immich · Nextcloud · NPM · Tailscale
 
This is a personal guide documenting how I actually set up my home server from scratch — the process, the config files, and every real error I ran into along the way. It's not a theoretical walkthrough. Everything written here came from hands-on experience.
 
---
 
## 📋 Table of Contents
 
1. [Hardware](#-hardware)
2. [Installing Debian](#-installing-debian)
3. [Getting Online & SSH Access](#-getting-online--ssh-access)
4. [Permanent HDD Mounting](#-permanent-hdd-mounting)
5. [Firewall (UFW)](#-firewall-ufw)
6. [Docker Setup & Folder Structure](#-docker-setup--folder-structure)
7. [Networking — DDNS & Reverse Proxy](#-networking--ddns--reverse-proxy)
8. [App Stack](#-app-stack)
   - [Jellyfin](#jellyfin)
   - [Immich](#immich)
   - [Nextcloud](#nextcloud)
   - [Gluetun](#gluetun)
   - [Homepage](#homepage)
   - [Anubis](#anubis)
   - [Portainer](#portainer)
   - [Uptime Kuma](#uptime-kuma)
   - [Watchtower](#watchtower)
   - [Autoheal](#autoheal)
9. [Secure Remote Access — Tailscale](#-secure-remote-access--tailscale)
10. [Security Hardening](#-security-hardening)
11. [Troubleshooting](#-troubleshooting)
12. [Quick Reference](#-quick-reference)
 
---
 
## 🖥️ Hardware
 
My server is built from repurposed desktop parts, which works perfectly well. The things that genuinely matter:
 
- **CPU** — A 4–8 core desktop chip is plenty. I'm running an Intel i5.
- **RAM** — 8 GB is the bare minimum. If you're running Immich with machine learning enabled, plan for 16 GB+. The ML container alone will comfortably eat 4–6 GB, and that's before Postgres, Redis, and everything else.
- **GPU** — A dedicated GPU matters specifically for hardware transcoding in Jellyfin. Without one, your CPU will max out the moment someone tries to stream a 4K file remotely. I use an AMD Radeon RX 470, which works well with VAAPI.
- **Boot drive** — An SSD (250 GB+) for the OS, Docker configs, and app databases.
- **Mass storage** — A spinning HDD for media. Make sure it's **CMR** (Conventional Magnetic Recording), not SMR. SMR drives have terrible sustained write performance and will cause headaches under load. Always verify via the manufacturer's spec sheet.
- **Network** — Wired Ethernet only. No exceptions for a server.
 
---
 
## 🐧 Installing Debian
 
A minimal headless Debian install — no desktop environment.
 
1. Download the Debian netinstall ISO from [debian.org](https://www.debian.org/distrib/) and flash it to a USB.
2. During installation, **uncheck** `Debian desktop environment` and **check** `SSH server` and `standard system utilities`.
3. Create a non-root user during setup. You'll use this for everything.
 
After first boot, `sudo` isn't installed by default on Debian minimal. Fix that immediately:
 
```bash
su -
apt update && apt install -y sudo
usermod -aG sudo youruser
exit
```
 
Log out and back in for the group change to apply.
 
---
 
## 🌐 Getting Online & SSH Access
 
### The HDMI trap
 
The first thing I tried was plugging my laptop into the server via HDMI to use it as a display. This doesn't work. The HDMI port on a laptop is **output-only** — it sends video, it can't receive it. Plugging two outputs together does nothing. The correct way to control a headless server is SSH.
 
### Finding the server's IP
 
Since SSH was enabled during installation, the service is already running. You just need the IP. The easiest method is logging into your router's admin panel (usually `192.168.1.1` or `192.168.0.1`) and looking for "Connected Devices" or "DHCP Clients". Look for a device named `debian`.
 
Or scan from another machine on the same network:
 
```bash
sudo pacman -S nmap    # Arch
sudo apt install nmap  # Debian/Ubuntu
 
nmap -sn 192.168.1.0/24
```
 
Once you have it:
 
```bash
ssh youruser@192.168.1.x
```
 
### Static IP
 
For a server, the IP needs to stay constant. The cleanest approach is a **DHCP reservation on your router** — find your server's MAC address with `ip link show`, then bind it to a fixed IP in the router's admin panel.
 
Alternatively, set it directly in Debian:
 
```bash
sudo nano /etc/network/interfaces
```
 
Replace the DHCP entry for your interface (e.g. `enp3s0`) with:
 
```
auto enp3s0
iface enp3s0 inet static
    address 192.168.1.x
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```
 
```bash
sudo systemctl restart networking
```
 
### Essential packages
 
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano htop ncdu tmux \
  ca-certificates gnupg lsb-release ufw
```
 
---
 
## 💾 Permanent HDD Mounting
 
Debian doesn't auto-mount HDDs on reboot. This must be hardcoded in `/etc/fstab` — if you skip it, the drive simply isn't there after a restart.
 
### Find your drive
 
```bash
sudo fdisk -l
# Look for the large drive, e.g. /dev/sdb
```
 
### Format it (first time only)
 
> ⚠️ **Skip this if the drive already has data you want to keep.**
 
```bash
sudo mkfs.ext4 /dev/sdb1
```
 
### Get the UUID
 
Always use the UUID, not the device path. Device names like `/dev/sdb1` can change between reboots; UUIDs never do.
 
```bash
sudo blkid /dev/sdb1
# UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="ext4"
```
 
### Mount point and fstab entry
 
```bash
sudo mkdir -p /mnt/media
sudo nano /etc/fstab
```
 
Add at the very bottom:
 
```
UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx  /mnt/media  ext4  defaults  0  2
```
 
Test without rebooting:
 
```bash
sudo mount -a
df -h | grep media
```
 
Fix ownership:
 
```bash
sudo chown -R youruser:youruser /mnt/media
```
 
---
 
## 🔥 Firewall (UFW)
 
Two things will silently break things if you don't handle them:
 
1. Docker uses internal bridge networks (`172.16.0.0/12`) for container-to-container traffic. Block that range and your containers can't talk to each other.
2. Tailscale uses its own interface (`tailscale0`) which UFW doesn't know about by default.
 
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
 
sudo ufw allow 80/tcp       # HTTP (for NPM)
sudo ufw allow 443/tcp      # HTTPS (for NPM)
sudo ufw allow ssh          # Port 22
 
# Your local home network
sudo ufw allow from 192.168.1.0/24
 
# Docker's internal bridge — required for container routing
sudo ufw allow from 172.16.0.0/12
 
# Tailscale interface
sudo ufw allow in on tailscale0
 
sudo ufw enable
sudo ufw status
```
 
> Port 81 (NPM admin) is deliberately not opened here. Access it only through Tailscale.
 
---
 
## 🐳 Docker Setup & Folder Structure
 
### Install Docker
 
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker youruser
newgrp docker
```
 
Verify:
 
```bash
docker run hello-world
docker compose version
```
 
If `docker compose` is missing:
 
```bash
sudo apt install docker-compose-plugin
```
 
### The folder structure
 
The single most important architectural decision. **All Docker configs stay on the SSD. All media lives on the HDD.** The reason is hardlinks — when files are on the same drive, you can hardlink them instead of copying, which saves enormous disk space. If they're on different drives, hardlinks are impossible.
 
```
/mnt/media/                ← HDD
└── data/
    ├── media/
    │   ├── movies/
    │   ├── tv/
    │   └── music/
    └── photos/            ← Immich library
 
~/docker/                  ← SSD (small config files)
├── jellyfin/
├── immich/
├── nextcloud/
├── npm/
└── ...
```
 
Create it:
 
```bash
sudo mkdir -p /mnt/media/data/media/{movies,tv,music}
sudo mkdir -p /mnt/media/data/photos
mkdir -p ~/docker/{jellyfin,immich,nextcloud,npm,duckdns,gluetun,homepage,anubis,portainer,watchtower,autoheal,uptime-kuma}
sudo chown -R youruser:youruser /mnt/media
```
 
### Find your PUID and PGID
 
```bash
id youruser
# uid=1000(youruser) gid=1000(youruser)
```
 
---
 
## 🌍 Networking — DDNS & Reverse Proxy
 
Your home IP changes periodically. DuckDNS keeps a domain name pointing at your current IP. NPM routes incoming traffic to the right container and handles SSL.
 
**Router setup first:** Forward ports **80** and **443** to your server's static IP. Never forward port 81.
 
### DuckDNS
 
Sign up at [duckdns.org](https://www.duckdns.org), create a subdomain, copy your token.
 
**`~/docker/duckdns/docker-compose.yml`:**
```yaml
services:
  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Your/Timezone
      - SUBDOMAINS=yoursubdomain
      - TOKEN=your-duckdns-token
      - LOG_FILE=true
    volumes:
      - ./config:/config
    restart: unless-stopped
```
 
```bash
cd ~/docker/duckdns && docker compose up -d
docker compose logs -f
# Should say "Your IP was set"
```
 
### Nginx Proxy Manager (NPM)
 
**`~/docker/npm/docker-compose.yml`:**
```yaml
services:
  app:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    environment:
      - TZ=Your/Timezone
```
 
```bash
cd ~/docker/npm && docker compose up -d
```
 
Admin UI: `http://192.168.1.x:81`
Default login: `admin@example.com` / `changeme` — change both on first login.
 
**Creating a proxy host with SSL:**
1. Hosts → Proxy Hosts → Add Proxy Host
2. Domain: `jellyfin.yoursubdomain.duckdns.org`
3. Scheme: `http`, Forward Host: your server IP, Port: `8096`
4. SSL tab → Request a new SSL Certificate → Force SSL → Save
 
**Restricting management apps to local network only:**
 
You probably don't need management apps (like Portainer) accessible from the internet. In NPM, go to the **Access Lists** tab, create a list called "Local Only", add `allow 192.168.1.0/24` and `deny all`. Then edit any proxy host you want to lock down and set its Access List to "Local Only". Someone trying to reach that URL from outside your network will get a `403 Forbidden`.
 
---
 
## 🚀 App Stack
 
### Jellyfin
 
Your personal media server.
 
**`~/docker/jellyfin/docker-compose.yml`:**
```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Your/Timezone
    volumes:
      - ./config:/config
      - ./cache:/cache
      - /mnt/media/data/media:/data/media:ro
    devices:
      - /dev/dri:/dev/dri       # GPU passthrough for hardware transcoding
    group_add:
      - "render"                # Required for AMD/Intel GPU access
```
 
The `group_add: render` entry is important. Without it, the container can see `/dev/dri` but can't actually use the GPU. If `render` doesn't work, find the numeric GID and use that instead:
 
```bash
stat -c '%g' /dev/dri/renderD128
# Usually 104 or 108
```
 
**Enabling hardware transcoding:**
Dashboard → Playback → Transcoding → Hardware Acceleration: **VAAPI** → Hardware encoding device: `/dev/dri/renderD128` → Save.
 
To verify it's working: play something, go to Dashboard → Active Streams. You should see `(hw)` next to the codec name. If you see `(sw)`, the GPU isn't being used.
 
---
 
### Immich
 
Self-hosted Google Photos with AI facial recognition and mobile backup.
 
> ⚠️ Immich updates constantly and has frequent breaking changes. Always fetch the official compose file — don't copy it from somewhere else.
 
```bash
cd ~/docker/immich
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```
 
Edit `.env`:
```env
UPLOAD_LOCATION=/mnt/media/data/photos
DB_PASSWORD=use_a_strong_password_here
TZ=Your/Timezone
```
 
```bash
docker compose up -d
```
 
This starts five containers: the main server, microservices, machine learning, Redis, and Postgres. The ML container handles facial recognition and CLIP search — expect it to use several GB of RAM and take 30–60 minutes to index a large library the first time.
 
**Importing from Google Takeout:**
After a large import (e.g., a Google Takeout dump), you'll often see broken thumbnails with "Error loading image". This is normal — the thumbnail generation worker got overwhelmed by the queue. Fix it by going to Administration (gear icon) → Jobs, and manually triggering the **Generate Thumbnails** and **Machine Learning** jobs. Let them run to completion.
 
**Updating Immich** (always read the release notes first):
```bash
docker compose pull && docker compose up -d
```
 
---
 
### Nextcloud
 
Self-hosted file sync — a personal Dropbox.
 
> ⚠️ Nextcloud runs as its internal `www-data` user (UID 33), not your system user. You must give that user ownership of the data folder or Nextcloud will refuse to start.
 
```bash
sudo mkdir -p /mnt/media/data/nextcloud
sudo chown -R 33:33 /mnt/media/data/nextcloud
```
 
**`~/docker/nextcloud/docker-compose.yml`:**
```yaml
services:
  nextcloud-db:
    image: mariadb:10.6
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    volumes:
      - ./db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=strong_root_password
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=strong_db_password
 
  nextcloud:
    image: nextcloud:latest
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - '9001:80'
    volumes:
      - ./config:/var/www/html
      - /mnt/media/data/nextcloud:/var/www/html/data
    environment:
      - MYSQL_HOST=nextcloud-db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=strong_db_password
      - NEXTCLOUD_TRUSTED_DOMAINS=cloud.yoursubdomain.duckdns.org
      - TRUSTED_PROXIES=192.168.1.x
      - OVERWRITEPROTOCOL=https
      - TZ=Your/Timezone
    depends_on:
      - nextcloud-db
```
 
The `TRUSTED_PROXIES` and `OVERWRITEPROTOCOL` lines are important — without them, Nextcloud doesn't know it's sitting behind a reverse proxy and will generate broken HTTP links.
 
---
 
### Gluetun
 
A VPN container. You can route other containers through it by setting `network_mode: service:gluetun` in their compose service instead of defining their own network.
 
**`~/docker/gluetun/docker-compose.yml`:**
```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=mullvad      # Change to your provider
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=your_key
      - WIREGUARD_ADDRESSES=10.x.x.x/32
      - TZ=Your/Timezone
```
 
---
 
### Homepage
 
A dashboard with live widgets for all your services. Replaced Heimdall after outgrowing it.
 
**`~/docker/homepage/docker-compose.yml`:**
```yaml
services:
  homepage:
    image: ghcr.io/gethomepage/homepage:latest
    container_name: homepage
    restart: unless-stopped
    ports:
      - '3000:3000'
    volumes:
      - ./config:/app/config
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /mnt/media:/mnt/media:ro        # So Homepage can show disk usage
    environment:
      - TZ=Your/Timezone
      - PUID=1000
      - PGID=1000
```
 
**`~/docker/homepage/config/services.yaml`:**
```yaml
- Media & Photos:
    - Jellyfin:
        icon: jellyfin.png
        href: https://jellyfin.yoursubdomain.duckdns.org   # Use your DuckDNS link here
        description: Personal Media Server
        widget:
          type: jellyfin
          url: http://192.168.1.x:8096   # Use local IP for the API widget
          key: your_jellyfin_api_key
 
    - Immich:
        icon: immich.png
        href: https://immich.yoursubdomain.duckdns.org
        description: Photo Backup
        widget:
          type: immich
          url: http://192.168.1.x:2283
          key: your_immich_api_key
 
- Personal Cloud:
    - Nextcloud:
        icon: nextcloud.png
        href: https://cloud.yoursubdomain.duckdns.org
        description: Private File Storage
 
- Server Management:
    - Portainer:
        icon: portainer.png
        href: http://192.168.1.x:9000    # Keep local-IP only, not exposed publicly
        description: Docker Management
```
 
> **href vs widget URL:** Use your DuckDNS HTTPS links for `href` (what you click when you're away from home). Use the local IP for widget `url` (what Homepage uses to pull live stats in the background — no point routing through the internet for something on the same machine).
 
**`~/docker/homepage/config/widgets.yaml`:**
```yaml
- resources:
    cpu: true
    cputemp: true
    memory: true
    disk:
      - /
      - /mnt/media     # Shows both OS drive and media HDD
 
- datetime:
    text_size: xl
    format:
      dateStyle: short
      timeStyle: short
```
 
**`~/docker/homepage/config/docker.yaml`:**
```yaml
my-docker:
  socket: /var/run/docker.sock
```
 
Get API keys: Jellyfin → Dashboard → API Keys. Immich → User Settings → API Keys.
 
---
 
### Anubis
 
A bot protection container. Acts as a proof-of-work challenge layer that sits in front of Homepage and blocks automated scrapers and bots.
 
**`~/docker/anubis/docker-compose.yml`:**
```yaml
services:
  anubis-homepage:
    image: ghcr.io/techarohq/anubis:latest
    container_name: anubis-homepage
    restart: unless-stopped
    ports:
      - '3001:3001'
    environment:
      - TARGET=http://192.168.1.x:3000   # Points to Homepage
      - TZ=Your/Timezone
```
 
In NPM, point your Homepage proxy host to port `3001` (Anubis) instead of `3000` (Homepage directly). Legitimate browsers pass the challenge instantly; bots get blocked.
 
---
 
### Portainer
 
A web UI for managing Docker containers, viewing logs, editing compose files, and inspecting volumes/networks without using the terminal.
 
**`~/docker/portainer/docker-compose.yml`:**
```yaml
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - '9000:9000'
      - '9443:9443'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/data
    environment:
      - TZ=Your/Timezone
```
 
> ⚠️ Never expose port 9000 publicly via NPM. Portainer gives full Docker control. Access it only through Tailscale or your local IP.
 
---
 
### Uptime Kuma
 
Monitors whether your services are actually reachable and alerts you if something goes down. Replaced a brief Netdata experiment (Netdata's virtual memory graphs caused confusion and it was removed).
 
**`~/docker/uptime-kuma/docker-compose.yml`:**
```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - '3002:3001'
    volumes:
      - ./data:/app/data
    environment:
      - TZ=Your/Timezone
```
 
After setup, add monitors for each of your services. For Immich, the correct ping endpoint is:
 
```
http://192.168.1.x:2283/api/server/ping
```
 
Note: the old path was `/api/server-info/ping` — Immich removed that endpoint in a major update. If your monitor shows a `404`, this is why. Update it to `/api/server/ping`.
 
---
 
### Watchtower
 
Checks for updated Docker images nightly and automatically pulls and restarts containers.
 
**`~/docker/watchtower/docker-compose.yml`:**
```yaml
services:
  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=Your/Timezone
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_SCHEDULE=0 0 3 * * *    # 3:00 AM daily
      - WATCHTOWER_INCLUDE_RESTARTING=true
```
 
To exclude a container from auto-updates:
```yaml
labels:
  - "com.centurylinklabs.watchtower.enable=false"
```
 
**Important:** Never put Autoheal on database containers (Postgres, MariaDB, Redis). If Autoheal restarts a database mid-write, you risk corrupting your data. Let databases take as long as they need.
 
---
 
### Autoheal
 
Watchtower creates a race condition: when it updates multiple containers at once, apps that depend on a database or network can boot before their dependency is ready and freeze. Autoheal monitors containers with healthchecks and restarts any that go unhealthy.
 
**`~/docker/autoheal/docker-compose.yml`:**
```yaml
services:
  autoheal:
    image: willfarrell/autoheal:latest
    container_name: autoheal
    restart: unless-stopped
    environment:
      - AUTOHEAL_CONTAINER_LABEL=autoheal   # Only watches containers you tag
      - AUTOHEAL_INTERVAL=5
      - TZ=Your/Timezone
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```
 
To enable monitoring on a specific container, add a `healthcheck` and a label to its compose service:
 
```yaml
labels:
  - "autoheal=true"
healthcheck:
  test: curl -f http://localhost:PORT/ping || exit 1
  interval: 60s
  timeout: 10s
  retries: 3
```
 
Replace `PORT` with that app's internal port. For Immich: `2283`, for NPM admin: `81`, for Jellyfin: `8096/Health`.
 
**Apps that need Autoheal:** NPM, Nextcloud, Immich, Gluetun
(anything that depends on a database or VPN tunnel).
 
**Apps that should NOT have Autoheal:** Databases (Postgres, MariaDB, Redis), DuckDNS, Watchtower itself. Never watch the watchdog or the databases.
 
---
 
## 🔐 Secure Remote Access — Tailscale
 
Tailscale creates a WireGuard-based encrypted mesh network between your devices. The purpose here is a secure management backdoor — SSH and Portainer access from anywhere — without exposing those ports to the open internet.
 
**What went wrong the first time:** A standard Tailscale install overwrites `/etc/resolv.conf` via a feature called MagicDNS. This silently breaks NPM and DuckDNS. The DuckDNS container stops updating, Nginx can't resolve domains, and everything goes dark. This happened, Tailscale was uninstalled, and then re-installed with the correct flags.
 
### Install and start (muzzled)
 
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --accept-dns=false --accept-routes=false
```
 
`--accept-dns=false` is the critical flag. It tells Tailscale not to touch DNS. Without it, MagicDNS overwrites your resolver and breaks NPM.
 
### Disable MagicDNS in the web dashboard
 
Log into [login.tailscale.com](https://login.tailscale.com) → DNS → disable **MagicDNS**. This is separate from the command-line flag and also needs to be done.
 
### Enable on boot
 
```bash
sudo systemctl enable tailscaled
```
 
If Tailscale ever causes DNS issues again, the universal fix is:
 
```bash
sudo rm -f /etc/resolv.conf
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
 
### Remote access
 
Once your other devices are connected to the same Tailscale network:
 
```bash
ssh youruser@100.x.x.x          # SSH from anywhere
http://100.x.x.x:9000           # Portainer
http://100.x.x.x:81             # NPM admin
http://100.x.x.x:8096           # Jellyfin (if DuckDNS is unreachable)
```
 
**Why TCP forwarding over SSH is no longer needed:** Before Tailscale, accessing private dashboards remotely required SSH port forwarding. With Tailscale handling routing at the network level, you don't need that at all — which is also why `AllowTcpForwarding no` in SSH is safe.
 
---
 
## 🔒 Security Hardening
 
### SSH key authentication
 
Do this before tightening anything else. Disable password auth before keys are working and you'll lock yourself out.
 
```bash
# On your LOCAL machine, not the server:
ssh-keygen -t ed25519
ssh-copy-id youruser@192.168.1.x
 
# Test key login works before proceeding
ssh youruser@192.168.1.x
```
 
### SSH hardening
 
Add this block to the **bottom** of `/etc/ssh/sshd_config`. Settings at the bottom override conflicting defaults higher in the file.
 
```bash
sudo nano /etc/ssh/sshd_config
```
 
```
PermitRootLogin no
PasswordAuthentication no
 
# Security hardening (Lynis recommendations)
AllowTcpForwarding no
AllowAgentForwarding no
X11Forwarding no
MaxAuthTries 3
MaxSessions 2
ClientAliveCountMax 2
TCPKeepAlive no
LogLevel VERBOSE
```
 
These are all safe with a Tailscale setup. `AllowTcpForwarding no` is fine because you're using Tailscale for remote access rather than SSH tunneling. `X11Forwarding no` is irrelevant on a headless server.
 
```bash
sudo systemctl restart sshd
```
 
### Security audit (Lynis)
 
Running Lynis is worth doing once just to confirm nothing is obviously wrong:
 
```bash
sudo apt install lynis clamav rkhunter -y
sudo lynis audit system
```
 
A score around **66** on a home server with 15 Docker containers is genuinely good. Most of Lynis's suggestions are enterprise-level overkill (separate `/var` partitions, disabling USB, process auditing that would fill your SSD). The ones worth acting on are the SSH tweaks above.
 
Once the audit is done, remove the scanners — they're not needed running permanently:
 
```bash
sudo apt remove --purge rkhunter lynis clamav clamav-freshclam -y
sudo apt autoremove -y
```
 
### Unattended security upgrades
 
Makes Debian automatically apply critical security patches nightly. Only security fixes, not feature updates, so it won't break Docker apps.
 
```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
# A prompt will appear — select Yes
```
 
### Quality-of-life packages
 
```bash
sudo apt install needrestart apt-listbugs -y
```
 
- **needrestart** — after `apt upgrade`, tells you if a reboot is needed.
- **apt-listbugs** — checks Debian's bug database before installing an update. Silent by default; only speaks up when something is actually wrong.
 
---
 
## 🚨 Troubleshooting
 
### Storage fills up — hardlinks and duplicates
 
If disk usage grows way faster than expected, files are being physically copied instead of hardlinked. This happens when your media and Docker data are on different drives — which is exactly why the unified folder structure matters.
 
Run `jdupes` inside `tmux` so it survives if your SSH session drops:
 
```bash
sudo apt install jdupes -y
 
tmux new -s dedup
 
# Scan first (no changes yet)
sudo jdupes -r -S /mnt/media/data/
 
# Convert duplicates to hardlinks once you're happy with the output
sudo jdupes -r -L /mnt/media/data/
```
 
> ⚠️ Don't close your SSH window while this runs. The process is tied to your session — closing it kills jdupes mid-scan. If you need to disconnect, use `tmux` or prefix the command with `nohup` and `&`.
 
---
 
### `docker system df` shows misleading numbers
 
`docker system df` regularly shows enormous "reclaimable" space that doesn't exist — it double-counts shared image layers. Don't rely on it. Use `ncdu` instead:
 
```bash
sudo apt install ncdu -y
sudo ncdu /
```
 
Common actual culprits:
- Immich ML model cache inside its volume
- Jellyfin transcoding leftovers in `~/docker/jellyfin/cache/` — `.ts` files from interrupted streams, safe to delete when Jellyfin is stopped
- System logs:
  ```bash
  sudo journalctl --disk-usage
  sudo journalctl --vacuum-size=200M
  ```
 
---
 
### Container stuck on "Starting" in Portainer
 
**Symptom:** App works fine in the browser but Portainer shows "Starting" forever.
 
**Cause:** The `healthcheck` is pinging an endpoint that no longer exists. Immich is the main example — it moved its server port from `3001` to `2283` in a major update, and also changed the API path from `/api/server-info/ping` to `/api/server/ping`.
 
Diagnose:
```bash
docker inspect --format "{{json .State.Health.Log}}" container_name | python3 -m json.tool
```
 
Fix: update the healthcheck URL in `docker-compose.yml`, then:
```bash
docker compose down && docker compose up -d
```
 
---
 
### Watchtower crashes after `docker system prune`
 
**Symptom:** After `docker system prune -a`, a container won't start:
```
failed to set up container networking: network not found
```
 
**Cause:** `docker system prune -a` is more aggressive than it looks — it deletes all unused Docker networks, which can include the one a container was attached to.
 
**Fix:** Recreate the compose stack (this also recreates the missing network):
```bash
cd ~/docker/affected_container
docker compose down && docker compose up -d
```
 
---
 
### DuckDNS stops working after installing/reinstalling Tailscale
 
**Cause:** Tailscale's MagicDNS feature overwrites `/etc/resolv.conf`, which kills DNS resolution on the server. NPM and DuckDNS stop working.
 
**Fix:**
```bash
sudo rm -f /etc/resolv.conf
echo -e "nameserver 1.1.1.1\nnameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
 
Then always start Tailscale with `--accept-dns=false` to prevent it happening again.
 
---
 
### NPM SSL certificate won't renew
 
Let's Encrypt certificates expire after 90 days. NPM handles renewal automatically, but needs port 80 reachable from the internet. If renewal fails:
 
1. Verify your domain resolves to your current public IP — use [dnschecker.org](https://dnschecker.org) from outside your home network
2. Confirm port 80 is still forwarded on your router
3. Check NPM logs: `docker logs npm 2>&1 | grep -i "renew\|error\|cert"`
4. Force renewal: NPM Admin → SSL Certificates → edit the cert → Save
 
---
 
## 📦 Quick Reference
 
| Service | Local URL | Port | Expose to Internet? |
|---------|-----------|------|---------------------|
| NPM Admin | `http://server:81` | 81 | ❌ Tailscale only |
| Jellyfin | `http://server:8096` | 8096 | ✅ via NPM |
| Immich | `http://server:2283` | 2283 | ✅ via NPM |
| Nextcloud | `http://server:9001` | 9001 | ✅ via NPM |
| Homepage | `http://server:3000` | 3000 | ✅ via Anubis → NPM |
| Anubis | `http://server:3001` | 3001 | ✅ via NPM (fronts Homepage) |
| Portainer | `http://server:9000` | 9000 | ❌ Tailscale only |
| Uptime Kuma | `http://server:3002` | 3002 | ❌ Tailscale only |
| Gluetun | Internal only | — | ❌ |
| DuckDNS | Internal only | — | ❌ |
 
---
 
## 🔄 Useful Commands
 
```bash
# See all running containers
docker ps
 
# Follow logs live
docker logs -f container_name
 
# Restart a container
docker restart container_name
 
# Update and redeploy a stack
cd ~/docker/app_name && docker compose pull && docker compose up -d
 
# Remove a container and its folder permanently
cd ~/docker/app_name
docker compose down
cd ~
sudo rm -rf ~/docker/app_name
docker image prune -a    # Clean up leftover images
 
# Remove orphaned containers after editing a compose file
docker compose up -d --remove-orphans
 
# Real disk usage
sudo ncdu /mnt/media
 
# Clean up dangling images and stopped containers (safe)
docker system prune
 
# Check HDD health
sudo apt install smartmontools -y
sudo smartctl -a /dev/sdb
 
# System resource usage
htop
```
 
---
 
## 📝 A Note on This Guide
 
Everything documented here — the configs, the specific steps, and especially the troubleshooting entries — comes from actually building and running this setup. The errors aren't things I researched; they're things I ran into personally and had to debug. The Tailscale DNS issue, the Immich port change breaking healthchecks, the jdupes scan sitting at 99% while wondering if it was safe to close the terminal — all of it happened. If something reads oddly specific, that's why.
 
---
 

*Built with ❤️ on Debian + Docker. Contributions welcome.*
