# 🏠 Homelab — Proxmox ▶ TrueNAS SCALE ▶ Plex & *arr Stack

> A completely self-hosted media server running on local hardware, documented step-by-step so **you** can fork, replicate, and hack on it.

---


<!-- 2 ▸ Glance / Start-page dashboard -->
![Dashboard screenshot](cybersecurity%20125642.png)


https://github.com/bunnyhp/HOmeLab/blob/main/cybersecurity%20125605.png?raw=true 
## 📜 Table of Contents

- [Architecture](#architecture)
- [Hardware](#hardware)
- [Host & VM Setup](#host--vm-setup)
- [Storage Layout](#storage-layout)
- [Applications](#applications)
- [Networking & Remote Access](#networking--remote-access)
- [Back-ups & Maintenance](#back-ups--maintenance)
- [Roadmap](#roadmap)
- [License](#license)

---

## Architecture

```text
┌────────────────────────────┐
│  Proxmox VE (Host)         │
│  • Intel i5-10400F         │
│  • 32 GB DDR4 ECC          │
│  • 1 × 1 TB NVMe (ZFS boot)│
└───────┬────────────────────┘
        │ virt-IO (10 Gb)
┌───────▼────────────────────┐
│  VM — TrueNAS SCALE (24 GB)│
│  • tank ⇢ 4 × 4 TB HDD RAID-Z1 │
│  • apps ⇢ 1 × 1 TB SSD striped │
│  • Kubernetes + Docker      │
│                              │
│  ┌────────────────────────┐ │
│  │  Plex Media Server     │ │
│  │  qBittorrent-nox       │ │
│  │  Radarr / Sonarr       │ │
│  │  Lidarr / Bazarr       │ │
│  │  Jellyseerr            │ │
│  └────────────────────────┘ │
└─────────────────────────────┘
```

*Everything* lives inside the TrueNAS SCALE VM. Apps are deployed via the **official Apps catalogue** (Kubernetes), but you can also reproduce them with Docker Compose.

---
<!-- 1 ▸ Architecture diagram -->
![Architecture overview](cybersecurity%20125605.png)
## Hardware

| Part | Model | Purpose |
|------|-------|---------|
| **Server** | Dell OptiPlex 7080 SFF | Cheap, quiet, plenty of PCIe lanes |
| **CPU** | Intel Core i5-10400F | 6C/12T; quick Plex transcodes |
| **RAM** | 32 GB DDR4 ECC | ZFS ARC + containers |
| **Boot** | 1 TB PCIe NVMe (WD Black SN770) | Proxmox install & VMs |
| **Pool** | 4 × 4 TB Seagate IronWolf (RAID-Z1) | `tank` dataset (media) |
| **App SSD** | 1 TB Crucial MX500 SSD | `apps` dataset (containers) |
| **Network** | Intel X550-T2 (10 GbE) | Fast LAN transfers |

*Swap in whatever hardware you have — just make sure **TrueNAS sees direct-attached disks** (pass-through or VirtIO SCSI).*  

---

## Host & VM Setup

### 1 · Provision Proxmox

```bash
# ISO installer → ‘ZFS (RAID-1) on NVMe’ for boot mirror if you have 2 drives
pveam update
```

Snapshot-friendly, hands-off backups (see [Back-ups](#back-ups--maintenance)).

### 2 · Create TrueNAS SCALE VM

| Setting  | Value |
|----------|-------|
| **Cores** | 6 (host-passthrough) |
| **Memory** | 24 GB static |
| **Disks** | Pass-through each HDD/SSD with `qm set <id> -scsi<number> /dev/disk/by-id/…` |
| **NIC** | VirtIO paravirtualised |

Install TrueNAS SCALE **24.04**. Enable **Applications** (Kubernetes) under *System Settings → Advanced*.

---

## Storage Layout

```text
tank/
├── media/
│   ├── movies/
│   ├── tv/
│   ├── music/
│   └── photos/
└── downloads/
    ├── incomplete/
    └── complete/
```
<!-- 3 ▸ Plex library view -->
![Plex library](cybersecurity%20125743.png)

1. `media/*` — readonly for Plex  
2. `downloads/incomplete` — qBittorrent temp files  
3. `downloads/complete` — finished torrents/usenet; watched by the *arr stack

> **Permissions**: create an `apps` group, add each Kubernetes workload’s UID/GID, then `chown -R :apps tank/*` and `chmod 775` so containers can write.

---

## Applications

| App | Purpose | URL | Deployment | Notes |
|-----|---------|-----|------------|-------|
| **Plex** | Streams local media to any device | `http://truenas:32400` | TrueNAS Chart → `plex-official` | Grab claim token from <https://plex.tv/claim> |
| **qBittorrent** | Torrent client with Web UI | `http://truenas:8080` | Chart `qbittorrent-nox` | Set `DOWNLOAD_DIR=/downloads/incomplete` |
| **Radarr** | Movie automation | `http://truenas:7878` | Chart `radarr` | Connect to qBittorrent + Plex |
| **Sonarr** | TV automation | `http://truenas:8989` | Chart `sonarr` | Same indexers as Radarr |
| **Lidarr** | Music automation | `http://truenas:8686` | Chart `lidarr` | Warm-up MusicBrainz cache (first run is slow) |
| **Bazarr** | Subtitle management | `http://truenas:6767` | Chart `bazarr` | Link Radarr/Sonarr paths |
| **Jellyseerr** | Media request portal | `http://truenas:5055` | Chart `jellyseerr` | Auth via Plex OAuth |

**Container images** come from LinuxServer.io where possible; tags pinned to known-good versions (see `/charts/values`).

### One-shot install script (optional)

```bash
apps=( plex radarr sonarr lidarr bazarr qbittorrent jellyseerr )
for chart in "${apps[@]}"; do
  truenas scale chart ingress install "$chart" --values "charts/$chart.yaml"
  sleep 10
done
```

---

## Networking & Remote Access

* **Local DNS** — `media.local` via Pi-hole or Unbound  
* **Reverse Proxy** — Nginx Proxy Manager in an LXC (optional). SSL certs via Let’s Encrypt Wildcard.  
* **Secure tunnel** — Tailscale for zero-trust access when away from home.

| Port | Service | Exposed? |
|------|---------|----------|
| 32400 | Plex | ✅ (WAN) |
| 7878/8989/8686/6767 | *arr apps | ❌ LAN only |
| 8080 | qBittorrent | ❌ LAN only |
| 5055 | Jellyseerr | ✅ via subdomain `requests.media.example.com` |

<!-- 5 ▸ TrueNAS SCALE Apps screen -->
![TrueNAS Apps](cybersecurity%20125840.png)

<!-- 6 ▸ VM layout in Proxmox -->
![Proxmox VM layout](cybersecurity%20125908.png)

<!-- 7 ▸ ZFS pool / datasets -->
![ZFS datasets](cybersecurity%20125934.png)
---

## Back-ups & Maintenance

1. **ZFS Snapshots** — hourly (24h), daily (7d), weekly (4w), monthly (6m).  
2. **App Configs** — Restic push to Backblaze B2 nightly.  
3. **Media** — not backed up (ripped discs can be re-encoded) but protected by RAID-Z1.  
4. **Updates** — `sudo kubectl-apps upgrade --all` weekly via cron.
   
<!-- 4 ▸ *arr stack status widgets -->
![*arr stack](cybersecurity%20125809.png)
---

## Roadmap

- [ ] Add **Immich** photo management
- [ ] Migrate RSS indexers to **Prowlarr**
- [ ] Deploy **Glance** dashboard as startpage

PRs welcome — open an issue or fork away!

---

## License

[MIT](LICENSE)
