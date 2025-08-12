# Raspberry Pi 4 Homelab — 30‑Day Plan & Copy‑Paste Commands

##  Project Overview

<!-- Static status badges with icons -->
![Homelab Ready](https://img.shields.io/badge/Homelab-Ready-brightgreen?style=for-the-badge&logo=homeassistant&logoColor=white)
![Dockerized](https://img.shields.io/badge/Dockerized-Yes-blue?style=for-the-badge&logo=docker&logoColor=white)

##  Components

![Pi-hole](https://img.shields.io/badge/Pi--hole-Blocking_Bads-red?style=for-the-badge&logo=pihole&logoColor=white)
![Nextcloud](https://img.shields.io/badge/Nextcloud-Cloud_Ready-orange?style=for-the-badge&logo=nextcloud&logoColor=white)
![Uptime Kuma](https://img.shields.io/badge/Uptime_Kuma-Monitored-blue?style=for-the-badge&logo=uptimekuma&logoColor=white)

##  Monitoring Stack

![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-yellow?style=flat-square&logo=prometheus&logoColor=black)
![Grafana](https://img.shields.io/badge/Grafana-Dashboards-orange?style=flat-square&logo=grafana&logoColor=black)
![Loki](https://img.shields.io/badge/Loki-Logs-lightgrey?style=flat-square&logo=loki&logoColor=blue)

##  Access & Security

![Cloudflare Tunnel](https://img.shields.io/badge/Cloudflare-Tunnel-FF7300?style=plastic&logo=cloudflare&logoColor=white)
![SSH Keys](https://img.shields.io/badge/SSH-Keys_enabled-green?style=plastic&logo=ssh&logoColor=white)

##  Live Metrics (Dynamic)

![CPU Load](https://img.shields.io/badge/dynamic/json?url=https://example.com/api/metrics&query=$.cpu_load&label=CPU%20Load&color=informational)
![Docker Count](https://img.shields.io/badge/dynamic/json?url=https://example.com/api/metrics&query=$.docker_containers&label=Docker%20Containers&color=success)



**Hardware**: Raspberry Pi 4 (4–8 GB RAM recommended), microSD (32 GB+ or SSD), Ethernet.

**OS**: Raspberry Pi OS Lite (64‑bit) *or* Ubuntu Server 22.04 LTS (ARM64).

> Tip: This is written to be copy‑paste friendly. Replace placeholders in ALL‑CAPS.

---

## 0) Fresh Install + First Boot

1. Flash OS with Raspberry Pi Imager.
2. In Imager advanced settings enable: **SSH**, set **username/password**, set **Wi‑Fi/Ethernet**, and **hostname** (e.g., `pi-lab`).
3. Boot, then SSH in:

```bash
ssh USER@PI_IP
```

Update & basics:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl git ufw htop unzip jq ca-certificates gnupg
```

Set timezone & locale (optional):

```bash
sudo timedatectl set-timezone Europe/Berlin
sudo raspi-config nonint do_change_locale en_US.UTF-8
```

Static IP (NetworkManager default on Bookworm):

```bash
sudo nmtui  # Edit a connection → set IPv4 to Manual, set address/gateway/DNS → Activate
```

Harden SSH a bit:

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh
# on your client: ssh-keygen -t ed25519; ssh-copy-id USER@PI_IP
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

Firewall (optional):

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw enable
sudo ufw status
```

---

## 1) Days 1–3: Linux Fundamentals Refresher

Practice commands:

```bash
ls -la
pwd
mkdir -p ~/lab/{apps,stacks,scripts,backups}
cp /etc/hosts ~/lab/hosts.bak
sudo systemctl status ssh
ip -br a
curl -I http://example.com
```

Users, groups, permissions:

```bash
sudo adduser devops
sudo usermod -aG sudo devops
sudo chown -R $USER:$USER ~/lab
chmod -R 750 ~/lab
```

---

## 2) Days 4–7: Install Docker & Compose, Portainer, Basics

Install Docker Engine + Compose plugin:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
# Verify
docker version
docker compose version
```

Enable at boot:

```bash
sudo systemctl enable docker && sudo systemctl start docker
```

Create an apps folder:

```bash
mkdir -p ~/lab/stacks/portainer
```

**Portainer** (Docker UI):

```bash
docker volume create portainer_data
docker run -d \
  -p 9000:9000 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

Login at: `http://PI_IP:9000`.

**Quick test containers**:

```bash
docker run -d --name whoami -p 8000:80 traefik/whoami
curl http://PI_IP:8000
```

**Uptime Kuma**:

```bash
mkdir -p ~/lab/stacks/uptime-kuma
cat > ~/lab/stacks/uptime-kuma/docker-compose.yml <<'YAML'
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
YAML
cd ~/lab/stacks/uptime-kuma && docker compose up -d
```

Open: `http://PI_IP:3001`.

---

## 3) Days 8–12: Network Services — Pi‑hole, Reverse Proxy

### Pi-hole (network‑wide ad‑blocking)

```bash
mkdir -p ~/lab/stacks/pihole
cat > ~/lab/stacks/pihole/.env <<'ENV'
TZ=Europe/Berlin
WEBPASSWORD=CHANGE_ME
PIHOLE_DNS_1=1.1.1.1
PIHOLE_DNS_2=8.8.8.8
ENV

cat > ~/lab/stacks/pihole/docker-compose.yml <<'YAML'
services:
  pihole:
    image: pihole/pihole:latest
    container_name: pihole
    restart: unless-stopped
    hostname: pihole
    environment:
      TZ: ${TZ}
      WEBPASSWORD: ${WEBPASSWORD}
      PIHOLE_DNS_: ${PIHOLE_DNS_1};${PIHOLE_DNS_2}
    volumes:
      - ./etc-pihole:/etc/pihole
      - ./etc-dnsmasq.d:/etc/dnsmasq.d
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    cap_add:
      - NET_ADMIN
YAML
cd ~/lab/stacks/pihole && docker compose up -d
```

Web UI: `http://PI_IP:8080`.

### Reverse Proxy (Nginx Proxy Manager)

```bash
mkdir -p ~/lab/stacks/npm
cat > ~/lab/stacks/npm/docker-compose.yml <<'YAML'
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: unless-stopped
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
YAML
cd ~/lab/stacks/npm && docker compose up -d
```

Dashboard: `http://PI_IP:81` (default login `admin@example.com` / `changeme`).

---

## 4) Days 13–17: Remote Access with Cloudflare Tunnel (no port‑forwarding)

Install `cloudflared`:

```bash
mkdir -p ~/bin && cd ~/bin
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb -o cloudflared.deb
sudo apt install -y ./cloudflared.deb
cloudflared --version
```

Authenticate and create a tunnel (opens browser once):

```bash
cloudflared tunnel login
cloudflared tunnel create pi-tunnel
```

Create config:

```bash
mkdir -p ~/.cloudflared
cat > ~/.cloudflared/config.yml <<'YAML'
ingress:
  - hostname: kuma.YOURDOMAIN.com
    service: http://localhost:3001
  - hostname: portainer.YOURDOMAIN.com
    service: http://localhost:9000
  - service: http_status:404

warp-routing:
  enabled: false

tunnel: pi-tunnel
credentials-file: /home/$USER/.cloudflared/$(ls ~/.cloudflared | grep json)
YAML
```

Run as a service:

```bash
sudo cloudflared --config ~/.cloudflared/config.yml service install
sudo systemctl enable cloudflared && sudo systemctl start cloudflared
sudo systemctl status cloudflared
```

Add DNS CNAMEs in Cloudflare to the tunnel for `kuma` and `portainer`.

---

## 5) Days 18–22: Automation — Cron, Bash, Ansible

### Simple cron backup (compose folders → \~/backups)

```bash
cat > ~/scripts/backup_stacks.sh <<'SH'
#!/usr/bin/env bash
set -euo pipefail
TARGET=~/backups/$(date +%F)
mkdir -p "$TARGET"
rsync -a ~/lab/stacks/ "$TARGET" --exclude="*/node_modules" --exclude="*/tmp"
find ~/backups -type d -mtime +14 -exec rm -rf {} + || true
SH
chmod +x ~/scripts/backup_stacks.sh
(crontab -l 2>/dev/null; echo "0 3 * * * /home/$USER/scripts/backup_stacks.sh") | crontab -
```

### Install Ansible and manage localhost

```bash
sudo apt install -y ansible
mkdir -p ~/lab/ansible
cat > ~/lab/ansible/inventory.ini <<'INI'
[pi]
localhost ansible_connection=local
INI

cat > ~/lab/ansible/setup.yml <<'YAML'
---
- hosts: pi
  become: true
  tasks:
    - name: Ensure common packages
      apt:
        name: ["htop","git","curl"]
        state: present
        update_cache: yes
    - name: Ensure docker service is running
      service:
        name: docker
        state: started
        enabled: true
YAML
ansible-playbook -i ~/lab/ansible/inventory.ini ~/lab/ansible/setup.yml
```

---

## 6) Days 23–27: Monitoring & Logs — Prometheus, cAdvisor, Node Exporter, Grafana, Loki

```bash
mkdir -p ~/lab/stacks/monitoring
cat > ~/lab/stacks/monitoring/docker-compose.yml <<'YAML'
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./data-prom:/prometheus
  nodeexporter:
    image: prom/node-exporter:latest
    container_name: nodeexporter
    restart: unless-stopped
    pid: host
    network_mode: host
    command: ["--path.rootfs=/host"]
    volumes:
      - /:/host:ro,rslave
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    ports: ["8081:8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    ports: ["3000:3000"]
    volumes:
      - ./grafana:/var/lib/grafana
  loki:
    image: grafana/loki:2.9.8
    container_name: loki
    restart: unless-stopped
    ports: ["3100:3100"]
    command: ["-config.file=/etc/loki/config/loki-config.yml"]
    volumes:
      - ./loki-config:/etc/loki/config
      - ./loki-data:/loki
  promtail:
    image: grafana/promtail:2.9.8
    container_name: promtail
    restart: unless-stopped
    command: ["-config.file=/etc/promtail/config.yml"]
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yml:/etc/promtail/config.yml:ro
      - /etc/hostname:/etc/host_hostname:ro
YAML

cat > ~/lab/stacks/monitoring/prometheus.yml <<'YAML'
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs: [{ targets: ['prometheus:9090'] }]
  - job_name: 'nodeexporter'
    static_configs: [{ targets: ['localhost:9100'] }]
  - job_name: 'cadvisor'
    static_configs: [{ targets: ['cadvisor:8080'] }]
YAML

mkdir -p ~/lab/stacks/monitoring/loki-config
cat > ~/lab/stacks/monitoring/loki-config/loki-config.yml <<'YAML'
server:
  http_listen_port: 3100
schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
storage_config:
  boltdb_shipper:
    active_index_directory: /loki/index
    cache_location: /loki/boltdb-cache
  filesystem:
    directory: /loki/chunks
limits_config:
  ingestion_rate_mb: 4
  ingestion_burst_size_mb: 6
YAML

cat > ~/lab/stacks/monitoring/promtail-config.yml <<'YAML'
server:
  http_listen_port: 9080
  grpc_listen_port: 0
clients:
  - url: http://loki:3100/loki/api/v1/push
positions:
  filename: /tmp/positions.yaml
scrape_configs:
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: varlogs
          host: ${HOSTNAME}
          __path__: /var/log/*log
  - job_name: docker
    static_configs:
      - targets: [localhost]
        labels:
          job: docker
          host: ${HOSTNAME}
          __path__: /var/lib/docker/containers/*/*-json.log
YAML

cd ~/lab/stacks/monitoring && docker compose up -d
```

Grafana at `http://PI_IP:3000` (default admin/admin). Add Prometheus as a data source (`http://prometheus:9090`) and Loki (`http://loki:3100`). Import dashboards for Node Exporter and cAdvisor (IDs 1860, 14282).

---

## 7) Days 28–30: Small Project — Nextcloud + Jellyfin + Tunnel

### Nextcloud + MariaDB

```bash
mkdir -p ~/lab/stacks/nextcloud
cat > ~/lab/stacks/nextcloud/.env <<'ENV'
MYSQL_ROOT_PASSWORD=CHANGE_ME
MYSQL_PASSWORD=CHANGE_ME
MYSQL_DATABASE=nextcloud
MYSQL_USER=nextcloud
TZ=Europe/Berlin
ENV

cat > ~/lab/stacks/nextcloud/docker-compose.yml <<'YAML'
services:
  db:
    image: mariadb:11
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
    volumes:
      - ./db:/var/lib/mysql
  app:
    image: nextcloud:28-apache
    restart: unless-stopped
    ports: ["8082:80"]
    environment:
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_HOST=db
      - TZ=${TZ}
    volumes:
      - ./nextcloud:/var/www/html
    depends_on: [db]
YAML
cd ~/lab/stacks/nextcloud && docker compose up -d
```

Open: `http://PI_IP:8082`.

### Jellyfin (media server)

```bash
mkdir -p ~/lab/stacks/jellyfin/{config,cache,media}
cat > ~/lab/stacks/jellyfin/docker-compose.yml <<'YAML'
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Berlin
    volumes:
      - ./config:/config
      - ./cache:/cache
      - ./media:/data/media
YAML
cd ~/lab/stacks/jellyfin && docker compose up -d
```

Web UI (host network): `http://PI_IP:8096`.

### Publish via Cloudflare Tunnel

Append to `~/.cloudflared/config.yml`:

```yaml
  - hostname: nextcloud.YOURDOMAIN.com
    service: http://localhost:8082
  - hostname: jellyfin.YOURDOMAIN.com
    service: http://localhost:8096
```

Then:

```bash
sudo systemctl restart cloudflared && systemctl status cloudflared
```

Create the CNAMEs in Cloudflare to the tunnel.

---

## 8) Quality of Life & Maintenance

Update all:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo rpi-eeprom-update -a || true
```

Update containers:

```bash
docker compose -f ~/lab/stacks/uptime-kuma/docker-compose.yml pull && docker compose -f ~/lab/stacks/uptime-kuma/docker-compose.yml up -d
# repeat for other stacks or:
docker ps --format '{{.Names}}' | xargs -I {} docker pull $(docker inspect --format='{{.Config.Image}}' {})
```

Clean up:

```bash
docker system prune -f
```

Backup Portainer & stacks (in addition to cron):

```bash
tar czf ~/backups/portainer-$(date +%F).tgz /var/lib/docker/volumes/portainer_data
```

---

## 9) Optional Extras to Explore

* **WireGuard VPN** (`pivpn`) to access your LAN remotely
* **Home Assistant** stack
* **Gitea** (git server) + **Drone** CI
* **AdGuard Home** instead of Pi‑hole
* **k3s** to learn Kubernetes (NUC as future controller, Pi as worker)

---

## 10) Troubleshooting Cheats

Ports in use:

```bash
sudo lsof -i -P -n | grep LISTEN
```

Container logs:

```bash
docker logs -f CONTAINER_NAME
```

Repair permissions (common issue for vols):

```bash
sudo chown -R $USER:$USER ~/lab
```

Low disk space:

```bash
df -h
sudo du -sh /var/lib/docker/* | sort -h | tail -n 20
```

---

**You’re set.** Work through the sections in order. If you hit a snag, tell me which step/command and the exact error, and I’ll fix it with you fast.
