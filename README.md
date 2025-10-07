# Proxmox Deployment Playbook aka The Beast Project
> Scope: Single-node Proxmox host with GPU passthrough, TrueNAS SCALE VM for ZFS bulk storage, Docker/Portainer-based app stacks (Plex/Jellyfin, Arr family, Homarr, Nginx), Ollama + Open WebUI VM with GPU for LLMs, GitLab CE, AMP, Steam cache, Twingate for zero-trust remote access, and a disposable cyber-security analysis VM.
> Target audience: experienced homelab/sysadmin performing direct implementation.

---

## 1.0 Overview & Conventions

1.1 Purpose, conventions and assumptions

* This playbook is a step-by-step technical reference. Commands assume Ubuntu-like environments for VMs and Proxmox VE on the host.
* Management network uses `10.0.10.0/24` (VLAN 10), internal services `10.0.20.0/24` (VLAN 20), DMZ/reverse-proxy `10.0.30.0/24` (VLAN 30). Adjust as needed.
* Hostname prefixes: `pve-` for Proxmox host, `vm-` for VMs.
* All code blocks are ready-to-run but always verify device IDs (`/dev/disk/by-id/...`, `lspci` output) before applying to your system.

1.2 Document structure

* Modules 1–13 correspond to previously agreed outline. Each module contains overview, prerequisites, step-by-step, integration notes, and maintenance items.

---

## 2.0 Module 1 — Hardware Foundation (Recap & Finalized Specs)

2.1 Selected hardware (recommended final build)

* CPU: **AMD Threadripper Pro 5975WX** (32c/64t).
* Motherboard: **WRX80** class (ASUS Pro WS WRX80E-SAGE SE or Supermicro M12SWA-TF).
* RAM: **128 GB ECC DDR4** (8×16 GB).
* GPU (transcode): **Intel Arc A380** (or integrated Intel GPU alternative if available).
* GPU (AI/Compute): **NVIDIA RTX 4070 Ti Super** (or 4080 depending on budget).
* NVMe (Proxmox OS / VM pool): **2 TB Samsung 990 PRO** (or equivalent, mirrored if possible).
* SSDs (apps): 2×2 TB SATA SSD (RAID1) for OS/containers if desired.
* HDDs (bulk storage): 4×10 TB NAS drives (RAIDZ2 in TrueNAS).
* PSU: 1000W+, Platinum; Case & cooling per earlier module.
* UPS: APC Smart-UPS 1500VA.

2.2 Key hardware configuration items (short)

* Enable IOMMU/SVM in BIOS, disable CSM, enable SR-IOV (if present), enable ECC, enable UEFI (OVMF) for VMs.

---

## 3.0 Module 2 — Proxmox Host Setup (Install, ZFS base, GPU prep)

3.1 Install Proxmox VE on NVMe

1. Flash Proxmox ISO to USB and install selecting NVMe for root (`rpool`).
2. Recommended during install: create mirrored rpool with two NVMe drives if available. If single NVMe, use it and plan regular backups.

3.2 Post-install initial commands

```bash
# Update host
apt update && apt full-upgrade -y
# Install zfs utils if not present
apt install -y zfsutils-linux
# Enable SSH, configure firewall if desired
systemctl enable ssh
```

3.3 Create ZFS dataset for VM disks (on rpool)

```bash
# Example: create zpool dataset for docker images and VM disks
zfs create -o compression=zstd rpool/vm-storage
# Add it to Proxmox via GUI: Datacenter -> Storage -> Add -> ZFS
```

3.4 Kernel & GRUB IOMMU settings (AMD host)
Edit `/etc/default/grub`:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt rd.driver.pre=vfio-pci"
```

Then:

```bash
update-grub
```

3.5 Enable VFIO modules / persist
Create `/etc/modules-load.d/vfio.conf`:

```
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```

Update initramfs:

```bash
update-initramfs -u
```

3.6 PCI device identification (example)

```bash
lspci -nn | grep -E "VGA|Audio|3D"
```

Take note of PCI IDs (format `0000:0a:00.0` and vendor:device `8086:56a5` etc).

3.7 Bind devices to vfio-pci (example)
Create `/etc/modprobe.d/vfio-pci.conf`:

```
options vfio-pci ids=8086:56a5,10de:2782,10de:22bc disable_vga=1
```

Then:

```bash
update-initramfs -u && reboot
```

3.8 Verify post-reboot

```bash
lsmod | grep vfio
lspci -k | grep -EA3 'VGA|Audio|3D'
dmesg | grep -i iommu
```

3.9 Network bridges + VLAN interfaces (example `/etc/network/interfaces`)

```bash
auto lo
iface lo inet loopback

auto enp6s0
iface enp6s0 inet manual

auto vmbr0
iface vmbr0 inet static
    address 10.0.10.10/24
    gateway 10.0.10.1
    bridge_ports enp6s0
    bridge_stp off
    bridge_fd 0

auto vmbr0.20
iface vmbr0.20 inet manual
    vlan-raw-device vmbr0

auto vmbr0.30
iface vmbr0.30 inet manual
    vlan-raw-device vmbr0
```

3.10 Host tuning tips (quick)

* Set VM CPU type to `host`, disable ballooning in VM options, enable I/O threads for heavy disk VMs, use `virtio-scsi-pci` for disk controllers.

---

## 4.0 Module 3 — Network Architecture (VLANs, Twingate, Reverse Proxy)

4.1 VLAN layout (canonical)

* VLAN 10 — Management: `10.0.10.0/24` (Proxmox, Twingate connector)
* VLAN 20 — Internal services: `10.0.20.0/24` (TrueNAS, media, GitLab, Ollama, containers)
* VLAN 30 — DMZ/Reverse-proxy: `10.0.30.0/24` (Nginx, public endpoints)

4.2 Switch/Router configuration guidelines

* Configure Proxmox NIC as trunk (tag VLANs 10,20,30).
* Configure access ports for devices: management workstation to VLAN10, NAS to VLAN20 (if physical), DMZ web host to VLAN30.
* Implement ACLs on router/L3 switch per Module 3.6 firewall rules.

4.3 Twingate integration (zero-trust access)

* Deploy Twingate Connector VM (Ubuntu) on VLAN10. Use Twingate admin to download connector configuration. Example Docker deploy (on the connector VM):

```yaml
version: "3.8"
services:
  twingate:
    image: twingate/connector:latest
    restart: unless-stopped
    volumes:
      - ./cfg:/app/cfg
    environment:
      - TWINGATE_TOKEN=<your-token>
```

* Create access rules in Twingate: Admins (full access), Users (limited resources).

4.4 Reverse proxy design (Nginx Proxy Manager recommended)

* Host reverse proxy on VLAN30, public IP forwarded to it. Use Nginx Proxy Manager (NPM) Docker container for managing many subdomains and Let's Encrypt.

4.5 DNS suggestions

* Public DNS: Register domain and create subdomains (`plex.example.com`, `git.example.com`, etc.).
* Internal DNS: Deploy AdGuard Home or Pi-hole on VLAN20 for internal hostname resolution and local overrides.

4.6 Firewall rules (L3/router) — examples

* Allow: VLAN10 → VLAN20 (management)
* Allow: VLAN30 → VLAN20 ports 80/443 only (reverse proxy)
* Deny: VLAN20 → VLAN10 (internal shouldn't manage Proxmox)
* Deny: Internet → VLAN20 (no direct access; use Twingate or reverse proxy)

---

## 5.0 Module 4 — TrueNAS VM (ZFS Bulk Storage)

5.1 Purpose & design decisions

* TrueNAS SCALE VM runs real ZFS on passed-through HDDs for best performance and data integrity. Use RAIDZ2 for 4+ disks.

5.2 Create TrueNAS VM — configuration checklist

* BIOS: OVMF (UEFI), Machine: `q35`, CPU: host type, Sockets/Cores: e.g., 8 cores, Memory: 32 GB recommended for ARC, System disk: 64 GB NVMe virtual drive (do not put data disks here), Network: `vmbr0.20`.

5.3 Passthrough HDDs & NVMe partitions (example)

1. Identify disks:

```bash
ls -l /dev/disk/by-id/
```

2. Edit VM config `/etc/pve/qemu-server/100.conf` add:

```
scsi1: /dev/disk/by-id/ata-ST10000VN0004-2GS11L_ZHZ1A123
scsi2: /dev/disk/by-id/ata-ST10000VN0004-2GS11L_ZHZ1A124
scsi3: /dev/disk/by-id/ata-ST10000VN0004-2GS11L_ZHZ1A125
scsi4: /dev/disk/by-id/ata-ST10000VN0004-2GS11L_ZHZ1A126
# NVMe SLOG/L2ARC (optional)
scsi5: /dev/nvme0n1p4
scsi6: /dev/nvme0n1p5
```

3. Boot TrueNAS VM and perform pool creation via TrueNAS UI.

5.4 ZFS pool creation (TrueNAS UI steps)

* Go to Storage → Pools → Create Pool → select disks → RAIDZ2 → add SLOG (ZIL) and L2ARC if present → enable compression `zstd` → create datasets: `media`, `apps`, `backups`, `gitlab`, `ai`.

5.5 NFS / SMB / iSCSI shares

* NFS for Linux VMs: create NFS share for `media` and `apps`. Limit to `10.0.20.0/24`.
* iSCSI for block-level volumes (optional for GitLab). TrueNAS SCALE supports dynamic LUN mapping.

5.6 Snapshots & replication

* Configure Periodic Snapshot Tasks (e.g., hourly for config datasets, daily for media).
* Configure Replication Tasks to remote host (Proxmox Backup Server or remote TrueNAS) with encryption and schedule.

5.7 Performance & safety notes

* SLOG device should be low-latency, high-endurance NVMe (Intel Optane recommended).
* L2ARC only if sufficient RAM is not available — L2ARC benefits are workload-specific.
* Use dataset quotas to avoid full pool issues.

5.8 Mounting NFS on VMs (example)

```bash
# create mount point
sudo mkdir -p /mnt/media
# fstab entry
10.0.20.11:/mnt/bulk-storage/media  /mnt/media  nfs  defaults,_netdev  0 0
sudo mount -a
```

---

## 6.0 Module 5 — Media Stack (Plex / Jellyfin + Docker + Portainer + Intel iGPU)

6.1 VM specs & roles

* VM name: `vm-media` (10.0.20.12)
* 6 vCores, 8–16 GB RAM, 64 GB SSD, Bridge: `vmbr0.20`, Intel GPU passthrough.

6.2 Create VM and passthrough Intel GPU

* Use `hostpci0: 00:02.0,pcie=1` in VM config (PCI ID from `lspci`). For Intel GPUs ensure `i915` is not bound on host (use vfio binding if necessary per Module 2).

6.3 Install Docker & Portainer (within VM)

```bash
# docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# docker-compose plugin or docker compose v2
apt install -y docker-compose-plugin
# portainer
docker volume create portainer_data
docker run -d -p 9000:9000 -p 9443:9443 \
  --name portainer --restart=unless-stopped \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data portainer/portainer-ce:latest
```

6.4 Mount TrueNAS NFS datasets
Add `/etc/fstab` entries and mount:

```
10.0.20.11:/mnt/bulk-storage/media  /mnt/media  nfs  defaults,_netdev  0 0
10.0.20.11:/mnt/bulk-storage/apps   /mnt/config nfs  defaults,_netdev  0 0
```

6.5 Docker Compose for Plex & Jellyfin (example)
File: `/opt/media/docker-compose.yml`

```yaml
version: "3.8"
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    network_mode: host
    environment:
      - PUID=1000
      - PGID=1000
      - VERSION=docker
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /mnt/config/plex:/config
      - /mnt/media:/media
    restart: unless-stopped

  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    network_mode: host
    devices:
      - /dev/dri:/dev/dri
    volumes:
      - /mnt/config/jellyfin:/config
      - /mnt/media:/media
    restart: unless-stopped
```

Start:

```bash
cd /opt/media
docker compose up -d
```

6.6 Validate transcoding (commands)

```bash
# inside VM
ls /dev/dri
# check intel_gpu_top (install intel-gpu-tools)
sudo apt install -y intel-gpu-tools
intel_gpu_top
```

* Plex: Settings → Transcoder → Enable hardware acceleration.
* Jellyfin: Admin → Playback → Hardware acceleration → Intel QSV.

6.7 Reverse-proxy setup

* Configure Nginx Proxy Manager to proxy `plex.example.com` → `10.0.20.12:32400` or use direct Plex remote access if desired.

6.8 Backups and snapshots

* Rely on TrueNAS dataset snapshots for `/mnt/config/*`.
* Create zfs snapshot tasks in TrueNAS for `apps` dataset hourly; replicate `/backups` dataset offsite.

---

## 7.0 Module 6 — AMP (CubeCoders), Game Servers & Steam Cache

7.1 Purpose: AMP provides a management layer for game servers (Minecraft, Rust, etc.) with easy deployment. Use separate VM or docker host based on preference.

7.2 VM suggestion: `vm-amp` (10.0.20.16)

* 4–8 vCores, 8–16 GB RAM, SSD 100–200 GB (games need fast I/O), connect to `vmbr0.20`.

7.3 AMP deployment options

* **Option A — Docker** (recommended with Portainer)
  Use CubeCoders’ AMP docker setup; follow CubeCoders docs for licensing and images.

7.4 Steam Cache (LAN caching) — optional

* Purpose: locally cache Steam downloads to reduce WAN bandwidth and speed re-downloads for clients on LAN. Use `steamcache/steamcache` images or `swag/steamcache` stacks.
* Basic Docker Compose (example):

```yaml
version: '3'
services:
  steamcache:
    image: steamcache/redis:latest
    restart: unless-stopped
    ports:
      - 80:80
      - 443:443
      - 27036:27036/udp
    volumes:
      - /mnt/config/steamcache/cache:/data/cache
```

* Configure client DNS to point Steam CDN domains to the cache server (via internal DNS conditional forwarder).

7.5 Integration with reverse proxy & Twingate

* Expose AMP web UI only via Twingate or restrict via Nginx Proxy Manager access lists.

---

## 8.0 Module 7 — Arr Family (Radarr/Sonarr/Lidarr/Readarr/Prowlarr) + Homarr + Nginx

8.1 VM: `vm-arr` (10.0.20.13)

* 4 vCores, 4–8 GB RAM, Disk: 64–128 GB, Bridge: `vmbr0.20`.

8.2 Docker Compose for Arr family + Homarr (example `/opt/arr/docker-compose.yml`)

```yaml
version: "3.8"
services:
  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    environment:
      - PUID=1000
      - PGID=1000
    volumes:
      - /mnt/config/sonarr:/config
      - /mnt/media:/media
    ports:
      - "8989:8989"
    restart: unless-stopped

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    volumes:
      - /mnt/config/radarr:/config
      - /mnt/media:/media
    ports:
      - "7878:7878"
    restart: unless-stopped

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    volumes:
      - /mnt/config/lidarr:/config
      - /mnt/media:/media
    ports:
      - "8686:8686"
    restart: unless-stopped

  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    volumes:
      - /mnt/config/prowlarr:/config
    ports:
      - "9696:9696"
    restart: unless-stopped

  homarr:
    image: ajayyy/homarr:latest
    container_name: homarr
    ports:
      - "7575:7575"
    volumes:
      - /mnt/config/homarr:/app/data
    restart: unless-stopped
```

Start with `docker compose up -d`.

8.3 Configure download client (Deluge/qBittorrent) and media organization

* Install download client in same Docker host or separate VM; point Arr apps to download client credentials.
* Ensure download client moves completed downloads to `/mnt/media` (TrueNAS share) using proper UID/GID mapping.

8.4 Nginx reverse-proxy rules (via NPM)

* Proxy `radarr.example.com` → `10.0.20.13:7878`
* Use Access Lists if exposing to WAN; prefer Twingate for admin access.

8.5 Notifications & integration

* Prowlarr as indexer manager; link to Sonarr/Radarr/Lidarr.
* Configure Connectors (e.g., Discord, email) as needed.

---

## 9.0 Module 8 — Ollama + Open WebUI VM (GPU Passthrough) + SpiderFoot Integration

9.1 Objective

* Run local LLM inference with **Ollama** (or equivalent local LLM runtime) on a VM with NVIDIA GPU passthrough. Provide an Open WebUI (e.g., text-generation-webui) for UI. SpiderFoot will call Ollama via API for enrichment.

9.2 VM specs: `vm-ollama` (10.0.20.15)

* 8–16 vCores, 32–64 GB RAM (depends on models), GPU: RTX 4070 Ti Super passed through, Disk: 200–1000 GB NVMe for models.

9.3 NVIDIA passthrough (host) — quick steps recap

* Ensure `vfio-pci` bound to GPU on host or use `nvidia` driver on host only if you plan to share GPU with host (not recommended). For clean passthrough bind to vfio.
* Example VM config addition (PCI device):

```
hostpci0: 41:00.0,pcie=1
hostpci1: 41:00.1,pcie=1
```

(IDs from `lspci`.)

9.4 Install Ubuntu, NVIDIA drivers, CUDA in VM

```bash
# add repo
sudo apt update && sudo apt upgrade -y
# install build tools and dkms
sudo apt install -y build-essential dkms
# add NVIDIA repo and install driver (example)
sudo apt-get install -y nvidia-driver-535
# reboot
```

Verify `nvidia-smi` works.

9.5 Install Ollama (example pattern — follow Ollama docs for exact packages)

```bash
# Example, placeholder - use official Ollama install method
curl -s https://ollama.ai/install.sh | bash
# Then configure model downloads as required
ollama pull <model-name>
```

9.6 Open WebUI deployment (text-generation-webui or Auto-GPT-like UI)

* Option A: Use `text-generation-webui` connecting to local Ollama endpoint if supported.
* Docker approach example (pseudo):

```yaml
services:
  webui:
    image: textgeneration/webui:latest
    environment:
      - OLLAMA_URL=http://127.0.0.1:11434
    ports:
      - "8080:8080"
    restart: unless-stopped
```

9.7 SpiderFoot integration (SpiderFoot ↔ Ollama API)

* SpiderFoot can be extended to call an LLM for enrichment via API. Use SpiderFoot’s `sfp_ossint` or custom module to send content to Ollama API and ingest response. Example pseudo-process:

  1. SpiderFoot extracts entity (domain/email/file hash).
  2. SpiderFoot calls local Ollama REST endpoint with prompt: "Summarize threat indicators for X" and receives structured JSON.
  3. SpiderFoot saves enrichment to event store.

9.8 API example (calling Ollama from Python)

```python
import requests
url = "http://127.0.0.1:11434/api/generate"
payload = {"model":"<model>","prompt":"Summarize...","max_tokens":500}
r = requests.post(url, json=payload)
print(r.json())
```

9.9 Security & resource notes

* Limit external exposure of Ollama and WebUI; put behind reverse proxy with auth or only accessible via Twingate.
* Monitor GPU memory and model sizes; use quantized models for memory efficiency.

---

## 10.0 Module 9 — Developer Services: GitLab CE

10.1 VM: `vm-gitlab` (10.0.20.14)

* 8 vCores, 16–32 GB RAM, Disk: 200–500 GB (depending on repo sizes), Network: `vmbr0.20`.

10.2 Storage considerations

* Use iSCSI LUN from TrueNAS for GitLab’s primary storage if you need block-level performance OR mount NFS dataset for `/var/opt/gitlab` and `/var/log/gitlab` (NFS acceptable but slower for heavy CI).

10.3 GitLab CE Omnibus install (inside VM)

```bash
# Prepare
sudo apt update && sudo apt upgrade -y
sudo apt install -y ca-certificates curl openssh-server postfix
# Add GitLab repo and install
curl https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
sudo EXTERNAL_URL="https://git.example.com" apt-get install gitlab-ce
# Reconfigure
sudo gitlab-ctl reconfigure
```

10.4 HTTPS (via reverse proxy)

* Prefer to put GitLab behind Nginx Proxy Manager or route HTTPS through it. For Omnibus, if using proxy, disable internal LetsEncrypt and set `external_url` accordingly.

10.5 Backups for GitLab

* Use Omnibus `gitlab-backup` to produce tarballs to `/mnt/backups` (TrueNAS dataset) and replicate offsite daily.

```bash
# create backup
sudo gitlab-rake gitlab:backup:create
# automate via cron at 02:30 daily
```

* Also backup `/etc/gitlab` and SSL keys.

10.6 CI runner considerations

* Use separate runner host(s) (VM or containers) attached to VLAN20. Configure autoscaling runners or docker-runner for builds.

---

## 11.0 Module 10 — Apple Home Hub (Homebridge / Home Assistant)

11.1 Options

* For **HomeKit hub** functionality you require either Apple devices (Apple TV, HomePod, iPad) or Home Assistant with HomeKit integration acting as a bridge. If you need local HomeKit hub capabilities for automations, physical Apple hardware is recommended; Home Assistant can integrate many devices.

11.2 Minimal VM: `vm-home` (10.0.20.17)

* 2 vCores, 4 GB RAM, Disk 32 GB, Bridge: `vmbr0.20`.

11.3 Home Assistant (recommended)

* Install Home Assistant OS in a VM or run `homeassistant/home-assistant` container.
* If using Home Assistant OS, create a VM with OVMF and assign USB passthrough if you need Zigbee/Z-Wave dongles.

11.4 Homebridge (if you want HomeKit exposure)

* Homebridge docker container in the same VM or separate:

```yaml
services:
  homebridge:
    image: oznu/homebridge:latest
    restart: unless-stopped
    network_mode: host
    volumes:
      - /mnt/config/homebridge:/homebridge
    environment:
      - TZ=Europe/Prague
```

* Ensure mDNS/Bonjour works across VLANs if HomeKit devices are on different VLAN — typically requires multicast routing or putting Apple devices on VLAN20.

11.5 Networking tip

* For mDNS: either put Homebridge/Home Assistant and Apple devices on the same VLAN, or use mDNS reflector/Avahi reflector across VLANs. Keep security considerations in mind.

---

## 12.0 Module 11 — Steam Cache (Optional LAN Download Cache)

12.1 Purpose & quick install

* Set up `steamcache/steamcache` with `steamcache/generic` and `steamcache/redis` to cache Steam content.

12.2 Docker Compose example (basic)

```yaml
version: '3.8'
services:
  redis:
    image: steamcache/redis:latest
    volumes:
      - /mnt/config/redis:/data
    restart: unless-stopped

  steamcache:
    image: steamcache/steamcache:latest
    environment:
      - CACHE_MAX_SIZE=500G
    ports:
      - "80:80"
      - "443:443"
      - "27036:27036/udp"
    volumes:
      - /mnt/config/steamcache/cache:/data/cache
    restart: unless-stopped
```

12.3 Client configuration

* Configure local DNS (AdGuard) to redirect Steam CDN hostnames to the cache host internal IP to force caching. Detailed host list is available in steamcache docs.

---

## 13.0 Module 12 — Cyber Security Sandbox (Disposable Malware VM)

13.1 Purpose

* Provide a disposable VM to open suspect email attachments, analyze binaries, or run malware reconnaissance with network isolation and temporary logging.

13.2 VM design: `vm-malware` (ephemeral)

* Template-based deployment from saved snapshot. Use QEMU template with no persistent network (or attach to an isolated VLAN: `vmbr0.99`), enable `NoVNC` for console access.

13.3 Recommended setup & workflow

1. Build a **base template VM** (Ubuntu or Windows) with necessary analysis tools (Wireshark, Sysinternals, IDA/free tools, Python, VirusTotal uploader). Update and snapshot this template.
2. For each analysis, clone the template, attach to **isolated network** with a dedicated NAT/proxy that logs traffic, or no network at all. For networked analysis, route through an analysis gateway (a VM that logs and allows replay).
3. After analysis, **destroy** the VM and its disks. Optionally keep an exported QEMU disk for forensics if needed.

13.4 Example ephemeral automation script (Proxmox API + `qm clone` sample)

```bash
# clone template 900 to new ephemeral vm 950
qm clone 900 950 --name vm-malware-950 --full
qm set 950 --net0 virtio,bridge=vmbr0.99
qm start 950
# After done
qm stop 950
qm destroy 950 --purge
```

13.5 Safety measures

* Only log in to the exam VM via ephemeral credentials.
* Use full-disk encryption for VMs storing sensitive evidence.
* Reinstall template periodically to avoid persistent compromise.

---

## 14.0 Module 13 — Backups, Snapshots & Monitoring

14.1 Proxmox backups (vzdump)

* Use Proxmox Backup Server (PBS) or `vzdump` to backup VMs. Example cron job:

```bash
# /etc/cron.d/pve-backup
0 3 * * * root vzdump --compress zstd --mode snapshot --storage NFS-Backup --all 1
```

14.2 Proxmox Backup Server (recommended)

* Deploy PBS VM on VLAN10 or offsite host, store backup to TrueNAS `backups` dataset via NFS, or replicate PBS to remote PBS instance.

14.3 TrueNAS replication (ZFS)

* Configure replication tasks in TrueNAS UI to remote TrueNAS or S3-compatible object storage. Keep encryption on replication.

14.4 Offsite strategy

* Option A: ZFS replication to remote TrueNAS over SSH (encrypted).
* Option B: Cloud Sync to Backblaze B2 / S3 for critical config/backups (`backups` dataset only). Use `rclone` for mounting if needed.

14.5 Monitoring stack (Grafana / Prometheus / Node Exporter / Netdata)

* Deploy Prometheus and Grafana in Docker host or a dedicated VM. Use `node_exporter` on Proxmox host, `telegraf` on VMs, and `prometheus` to scrape metrics. Example quick-setup:

```yaml
# minimal prometheus docker-compose snippet
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
```

* Add dashboards for Proxmox, TrueNAS, GPU metrics (nvidia-smi exporter), and ZFS ARC.

14.6 Alerts & logging

* Configure alerting (Prometheus Alertmanager) for: disk failure, ZFS pool degradation, high GPU memory usage, failed backups, replication errors.
* Centralize logs using filebeat -> ELK or Loki/Grafana.

---

## 15.0 Security, Access & Hardening

15.1 Access controls

* Use Twingate as primary remote admin access.
* Reverse proxy for external services with mandatory HTTPS and 2FA where possible.
* Limit admin UI ports to VLAN10 and Twingate only.

15.2 SSH & Proxmox security

* Disable password auth; use key-based auth.
* Use Fail2ban on VMs and possibly Proxmox host.
* Keep `pve-manager` updates applied promptly; schedule maintenance windows.

15.3 Container & Docker security recommendations

* Use non-root users inside containers (PUID/PGID mapping).
* Use read-only mounts where possible.
* Limit privileges (avoid `--privileged`) and use capabilities sparingly.

15.4 Secrets management

* Store secrets in GitLab CI variables, Vault, or Docker secrets. Avoid plaintext files with credentials.

15.5 Network security appliances

* Consider running Suricata or Zeek on a mirrored port of your main switch to monitor for suspicious lateral traffic.

---

## 16.0 Automation & CI/CD (GitLab Integration)

16.1 GitLab CI runners

* Install a GitLab Runner VM or container, register runners with tags (docker, shell). Configure runners to mount workspace via NFS or use ephemeral docker-in-docker builds.

16.2 IaC for the environment (recommended)

* Store playbooks and config in GitLab repos: Ansible for VM config, Terraform for cloud resources and DNS, shell scripts for Proxmox API automation. Example Ansible roles: `proxmox`, `truenas`, `docker-host`, `ollama`.

16.3 Example GitLab CI stage snippet (deploy stack)

```yaml
stages:
  - deploy

deploy_arr:
  stage: deploy
  image: docker:stable
  services:
    - docker:dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD registry.example.com
    - docker compose -f docker-compose.arr.yml pull
    - docker compose -f docker-compose.arr.yml up -d
  only:
    - main
```

---

## 17.0 Operational Playbook & Day-to-Day Tasks

17.1 Daily checks

* Proxmox: `pveproxy`, VM statuses, backup success.
* TrueNAS: ZFS health `zpool status` and `scrub` schedule status.
* GPU: `nvidia-smi` usage on Ollama VM.
* Prometheus/Grafana: alert check.

17.2 Weekly

* Update containers (Portainer) and schedule Proxmox/VM OS updates.
* Verify replication and offsite backups.
* Rotate access keys and review Twingate logs.

17.3 Monthly

* Full system test: restore one VM from backup to verify backups.
* Replace failing drives proactively based on SMART.

17.4 Incident response (brief)

* If malware discovered, immediately isolate network, revoke Twingate access for compromised user account, spin up new analysis VM from template, capture images of affected host, escalate as needed.

---

## 18.0 Example Useful Scripts & Cron Jobs

18.1 Snapshot & replicate Proxmox config (cron)
`/usr/local/bin/pve-conf-backup.sh`

```bash
#!/bin/bash
DEST="/mnt/backups/pve-config-$(date +%F).tar.gz"
/bin/tar -czf $DEST /etc/pve /var/lib/pve-cluster
# optionally scp to offsite
```

Cron:

```
0 2 * * * root /usr/local/bin/pve-conf-backup.sh
```

18.2 ZFS scrub cron (TrueNAS runs it via UI, but for local ZFS on host)

```
0 4 1 * * root zpool scrub rpool
```

18.3 Automated ephemeral malware VM creation script (example referenced above in Module 12).

---

## 19.0 Integration Matrix (Quick Reference)

| Service        | VM/Container       | Storage                | Network VLAN      | External Access                           |
| -------------- | ------------------ | ---------------------- | ----------------- | ----------------------------------------- |
| Proxmox host   | Host               | rpool NVMe             | VLAN10            | Twingate                                  |
| TrueNAS SCALE  | VM                 | HDD passthrough RAIDZ2 | VLAN20            | Twingate                                  |
| Plex/Jellyfin  | Docker in vm-media | TrueNAS NFS            | VLAN20            | Nginx (VLAN30) / Twingate                 |
| Arr family     | Docker in vm-arr   | TrueNAS NFS            | VLAN20            | Nginx/Twingate                            |
| Nginx Proxy    | Docker in vm-proxy | SSD                    | VLAN30            | Public IP (Let's Encrypt)                 |
| Ollama + WebUI | VM with GPU        | NVMe model store       | VLAN20            | Twingate / Nginx (auth)                   |
| GitLab CE      | VM                 | iSCSI or NFS           | VLAN20            | Nginx + 2FA                               |
| AMP            | VM or container    | SSD                    | VLAN20            | Twingate /                                |
| Nginx          |                    |                        |                   |                                           |
| Steam Cache    | Docker             | SSD / cache            | VLAN20            | LAN only                                  |
| Malware VM     | Template ephemeral | local                  | VLAN99 (isolated) | No external (unless via controlled proxy) |

---

## 20.0 Troubleshooting Common Problems

20.1 GPU passthrough issues

* Symptom: VM boots but GPU not available.

  * Check `dmesg` for vfio errors.
  * Ensure host drivers are not binding GPU (blacklist host drivers) and `vfio-pci` has the correct IDs.
  * Verify IOMMU groups and PCI isolation (`find /sys/kernel/iommu_groups/ -maxdepth 2 -type l`).

20.2 ZFS pool degraded

* Steps: Identify failing disk with `zpool status`, replace with `zpool replace pool bad-disk new-disk`, monitor resilver.

20.3 NFS permission issues

* Ensure correct export options on TrueNAS and correct UID/GID mapping. Map container PUID/PGID to match TrueNAS owner.

20.4 Ollama model memory OOM

* Use smaller models or quantized models; restrict batch sizes; monitor GPU memory.

---

## 21.0 Appendices

### 21.1 Useful commands (Proxmox)

* Start/stop VM: `qm start 100` / `qm stop 100`
* List VM config: `cat /etc/pve/qemu-server/100.conf`
* Backup: `vzdump 100 --storage NFS-Backup --mode snapshot --compress zstd`

### 21.2 Useful commands (TrueNAS CLI)

* `zpool status`
* `zfs list -t snapshot`
* Use TrueNAS UI for most operations; prefer its replication scheduler.

### 21.3 Links (documentation to keep) — keep for reference

* Proxmox docs: [https://pve.proxmox.com/wiki/](https://pve.proxmox.com/wiki/)
* TrueNAS SCALE: [https://www.truenas.com/docs/scale/](https://www.truenas.com/docs/scale/)
* Docker: [https://docs.docker.com/](https://docs.docker.com/)
* GitLab CE: [https://about.gitlab.com/install/](https://about.gitlab.com/install/)
* Ollama: [https://ollama.ai](https://ollama.ai) (follow official install docs for exact commands)

---

## 22.0 Final Notes & Implementation Order (Suggested Execution Plan)

22.1 Implementation phased plan (recommended)

1. Build host hardware, configure BIOS (IOMMU/SVM/ECC).
2. Install Proxmox on NVMe and set up networking (vmbr + VLANs).
3. Prepare host VFIO & passthrough test (identify devices).
4. Deploy TrueNAS VM and create ZFS pool; expose NFS to hosts.
5. Create template VMs (Ubuntu minimal) with cloud-init and Portainer installed.
6. Deploy media VM (Plex/Jellyfin) and validate Intel GPU transcoding.
7. Deploy Arr stack and integrate with Plex/TrueNAS.
8. Deploy Nginx proxy on VLAN30 and configure domain + LetsEncrypt.
9. Deploy Ollama VM with NVIDIA passthrough and WebUI; test models.
10. Deploy GitLab CE, configure backups and runner.
11. Deploy AMP, Steam cache as needed.
12. Deploy monitoring + backups + alerting.
13. Harden and monitor, create operational runbooks.

22.2 Testing & validation

* Validate backups by restoring one VM monthly.
* Test network isolation and Twingate access before exposing services.
