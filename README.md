# 🏠 Homelab Infrastructure - Zorko

> Infrastructure auto-hébergée complète avec virtualisation, stockage ZFS, et accès sécurisé via Cloudflare Zero Trust

![Proxmox](https://img.shields.io/badge/Proxmox-VE_9.1.9-E57000?logo=proxmox&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-Trixie_13-A81D33?logo=debian&logoColor=white)
![TrueNAS](https://img.shields.io/badge/TrueNAS-Scale_26.0.0--BETA.1-0095D5?logo=truenas&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-Zero_Trust-F38020?logo=cloudflare&logoColor=white)
![ZFS](https://img.shields.io/badge/ZFS-RAID1-00979D?logo=openzfs&logoColor=white)
![LXC](https://img.shields.io/badge/LXC-12_Conteneurs-success)
![Storage](https://img.shields.io/badge/Storage-70.9%25_Used-orange)

---

## 📊 Vue d'Ensemble de l'Infrastructure

### Statistiques Actuelles

| Métrique | Valeur | Détails |
|----------|--------|---------|
| **Conteneurs LXC** | 12 actifs + 1 stoppé | Production 24/7 |
| **Machines Virtuelles** | 1 stoppée | debian12-cloudinit (template) |
| **Capacité Stockage** | 932 GB | 661 GB utilisés (70.9%) — 271 GB libres |
| **RAM Proxmox** | 32 GB DDR4 | ~6.4 GB utilisés (19.5%) |
| **CPU Hyperviseur** | AMD Ryzen 5 5600X | 6 cores / 12 threads — Corsair 680X |
| **Uptime Proxmox** | 9+ jours | PVE 9.1.7 / kernel 7.0.0-3-pve |
| **Freebox Delta** | FW 4.9.18.1 | Uptime: 10 jours — 10 Gbps ↓ / 900 Mbps ↑ |

### Répartition du Stockage TrueNAS (Tank — 661 GB / 932 GB)

```mermaid
pie title Utilisation Stockage Tank (70.9%)
    "Backups Proxmox" : 302
    "Films" : 183
    "Share" : 36
    "Séries TV" : 82
    "Downloads" : 49
    "Projets Git" : 2
    "Espace Libre" : 271
```

---

## 🏗️ Architecture Globale

### Flux de Trafic Internet → Services

```mermaid
graph LR
    A[🌐 Internet] -->|HTTPS| B[☁️ Cloudflare Edge]
    B -->|WAF + DDoS Protection| C{🔐 Access Control}
    C -->|✅ Authentifié| D[🔒 Tunnel zserv]
    C -->|❌ Bloqué| E[⛔ Access Denied]
    D -->|Connexions HA| F[📡 Cloudflared (PVE Host)]
    F -->|Reverse Proxy| G[🔀 Nginx Proxy Manager LXC 118]
    G -->|Route vers| H[🎯 Services LXC]

    style B fill:#f38020,stroke:#333,stroke-width:2px,color:#fff
    style C fill:#f9a825,stroke:#333,stroke-width:2px
    style D fill:#4caf50,stroke:#333,stroke-width:2px,color:#fff
    style E fill:#f44336,stroke:#333,stroke-width:2px,color:#fff
    style F fill:#2196f3,stroke:#333,stroke-width:2px,color:#fff
    style G fill:#9c27b0,stroke:#333,stroke-width:2px,color:#fff
    style H fill:#00bcd4,stroke:#333,stroke-width:2px,color:#fff
```

### Architecture Infrastructure Complète

```mermaid
graph TB
    subgraph CLOUD["☁️ Cloudflare Zero Trust"]
        DNS[🌍 DNS zorko.xyz]
        TUNNEL[🔒 Tunnel zserv]
        WAF[🛡️ WAF + DDoS]
        ACCESS[🔐 Access Apps]
    end

    subgraph EDGE["🏠 Réseau Domestique - FTTH 10G"]
        ROUTER[📡 Freebox Delta fbxgw7r<br/>10 Gbit/s ↓ / 900 Mbit/s ↑<br/>FW 4.9.18.1]
        ADGUARD[🛡️ AdGuard Home VM<br/>192.168.1.189<br/>DNS filtrant + DNSSEC]
    end

    subgraph COMPUTE["💻 Proxmox VE 9.1.9 — Debian Trixie"]
        PVE[⚙️ AMD Ryzen 5 5600X<br/>6C/12T — 32 GB RAM — Corsair 680X<br/>2× Quadro P5000 — 192.168.1.61]

        subgraph LXC_INFRA["Infrastructure (3 CT)"]
            CT_NPM[🔀 Nginx Proxy Manager — 118<br/>10.10.10.18]
            CT_DOCKER[🐋 Docker Host — 112<br/>10.10.30.12]
            CF[📡 Cloudflared — Host]
        end

        subgraph LXC_MEDIA["Média (3 CT)"]
            CT_QBIT[⬇️ qBittorrent — 104<br/>10.10.20.10<br/>VPN Kill Switch]
            CT_MEDIAHUB[🎬 Media Hub — 130<br/>10.10.20.30<br/>Radarr + Sonarr + Prowlarr]
            CT_AGENTDVR[📹 AgentDVR — 123<br/>10.10.20.23]
        end

        subgraph LXC_DEV["Dev & Sécurité (4 CT)"]
            CT_GITEA[🗂️ Gitea — 120<br/>10.10.20.20]
            CT_VAULT[🔒 Vaultwarden — 114<br/>10.10.20.14]
            CT_PORTFOLIO[🌐 Portfolio — 121<br/>10.10.30.21]
            end

        subgraph LXC_MON["Monitoring & Home (3 CT)"]
            CT_GRAF[📈 Grafana — 115<br/>10.10.10.10]
            CT_HB[🏡 Homebridge — 102<br/>10.10.20.102]
            CT_COCKPIT[🖥️ Cockpit — accès via pve.lan:9090]
        end

        subgraph LXC_AI["IA / GPU (1 CT)"]
            CT_INFER[🤖 Inference — 201<br/>10.10.40.10<br/>Ollama — 2× P5000]
        end
    end

    subgraph STORAGE["💿 TrueNAS Scale — Stockage ZFS"]
        NAS[🗄️ Pool Tank RAID 1<br/>932 GB Total — 661 GB Utilisés<br/>192.168.1.109]

        subgraph DATASETS["📁 Datasets"]
            DS_BACKUP[💾 Backups: 302 GB]
            DS_FILM[🎬 Films: 183 GB]
            DS_SERIES[📺 Séries: 82 GB]
            DS_SHARE[📂 Share: 36 GB]
            DS_DL[⬇️ Downloads: 49 GB]
            DS_GIT[🗂️ Git: 2 GB]
        end
    end

    DNS --> TUNNEL
    WAF --> TUNNEL
    ACCESS --> TUNNEL
    TUNNEL ==>|Chiffré| ROUTER
    ROUTER ==>|1 Gbit/s| PVE
    ROUTER --> ADGUARD

    PVE --> LXC_INFRA
    PVE --> LXC_MEDIA
    PVE --> LXC_DEV
    PVE --> LXC_MON
    PVE --> LXC_AI

    NAS -.NFS.-> CT_MEDIAHUB
    NAS -.NFS.-> CT_QBIT
    NAS -.NFS.-> PVE
    NAS --> DATASETS

    CT_GRAF -.Métriques.-> PVE

    style CLOUD fill:#f38020,stroke:#333,stroke-width:3px,color:#fff
    style EDGE fill:#4caf50,stroke:#333,stroke-width:3px,color:#fff
    style COMPUTE fill:#2196f3,stroke:#333,stroke-width:3px,color:#fff
    style STORAGE fill:#9c27b0,stroke:#333,stroke-width:3px,color:#fff
```

---

## 📁 Services Exposés (Nginx Proxy Manager — 17 hôtes)

| Status | Domaine | Backend | SSL |
|--------|---------|---------|-----|
| 🟢 | ad.zorko.xyz | 192.168.1.189:80 | ✅ |
| 🟢 | cockpit.zorko.xyz | 10.10.10.1:9090 | ✅ |
| 🟢 | git.zorko.xyz | 10.10.20.20:3000 | ✅ |
| 🟢 | grafana.zorko.xyz | 10.10.10.10:3000 | ✅ |
| 🟢 | home.zorko.xyz | 10.10.20.102:8581 | ✅ |
| 🟢 | kuma.zorko.xyz | 192.168.1.42:3001 | ✅ |
| 🟢 | nas.zorko.xyz | 192.168.1.109:80 | ✅ |
| 🟢 | npm.zorko.xyz | 10.10.10.18:81 | ✅ |
| 🟢 | petio.zorko.xyz | 10.10.20.30:5055 | ✅ |
| 🟢 | plex.zorko.xyz | 192.168.1.108:32400 | ✅ |
| 🟢 | port.zorko.xyz | 10.10.30.12:9443 | ✅ |
| 🟢 | prow.zorko.xyz | 10.10.20.30:9696 | ✅ |
| 🟢 | pve.zorko.xyz | 10.10.10.1:8006 | ✅ |
| 🟢 | qbit.zorko.xyz | 10.10.20.10:8090 | ✅ |
| 🟢 | radarr.zorko.xyz | 10.10.20.30:7878 | ✅ |
| 🟢 | sonarr.zorko.xyz | 10.10.20.30:8989 | ✅ |
| 🟢 | vault.zorko.xyz | 10.10.20.14:8000 | ✅ |

> + 14 autres hôtes de dev/tests (CityClaw, Skynet, etc.)
> DNS wildcard `*.zorko.xyz → 10.10.10.18` géré par AdGuard Home — NPM centralise tous les reverse proxy internes.

---

## 🎯 Points Forts Techniques

### Sécurité
- ✅ **Zero-Trust Access** via Cloudflare Tunnel (aucun port ouvert sur Internet)
- ✅ **AdGuard Home** (VM Freebox) — DNS filtrant avec DNSSEC, 6 listes actives
- ✅ **Vaultwarden** (LXC 114) — admin token argon2id, inscriptions désactivées, backups sqlite3 automatiques
- ✅ **qBittorrent VPN Kill Switch** — iptables OUTPUT DROP + bind tun0 (Windscribe)
- ✅ **Firewall Proxmox** — règles Datacenter + Node, isolation VLAN stricte
- ✅ **VLANs Dédiés** — Management (10), Apps (20), Dev (30), IA (40)
- ✅ **CrowdSec** — Bouncer nftables (blocage communautaire massif)
- ✅ **Containers Unprivileged** — permissions cron durcies et prévention IP spoofing
- ✅ **WAF Cloudflare** avec protection DDoS intégrée
- ✅ **Reverse Proxy SSL** centralisé (Nginx Proxy Manager)

### Virtualisation & Infrastructure
- ✅ **12 conteneurs LXC** en production 24/7
- ✅ **LXC 201 Inference** — Ollama avec 2× NVIDIA Quadro P5000 (GPU passthrough)
- ✅ **Proxmox VE 9.1.9** sur Debian Trixie (kernel 7.0.0-3-pve)
- ✅ **Monitoring** : Grafana + Uptime Kuma + Cockpit

### Stockage & Données
- ✅ **ZFS RAID 1** (miroir) — 932 GB total, 70.9% utilisé
- ✅ **NFS** pour stockage Proxmox (backups, médias, downloads)
- ✅ **Snapshots ZFS automatiques** — quotidiens, rétention 14 jours
- ✅ **Backups Proxmox vzdump** — hebdomadaires (dim. 01:00), 10 CTs, rétention 2

### Réseau
- ✅ **Freebox Delta** — FTTH 10 Gbps ↓ / 900 Mbps ↑ (FW 4.9.18.1)
- ✅ **DNS AdGuard** avec DNSSEC et 1 077k règles de blocage
- ✅ **Cloudflare DNS** zone zorko.xyz (Free plan)

---

## 📂 Documentation Détaillée

- **[Architecture Réseau](./architecture/network.md)**
- **[Stockage](./architecture/storage.md)**
- **[Virtualisation](./architecture/virtualization.md)**
- **[Compute](./hardware/compute.md)**
- **[Hardware Storage](./hardware/storage.md)**
- **[Hardware Réseau](./hardware/network.md)**
- **[Sécurité](./security/access_control.md)**
- **[Cloudflare Zero Trust](./security/cloudflare_zero_trust.md)**
- **[Backups](./automation/backups.md)**

---

## 🛠️ Technologies Utilisées

**Virtualisation & Conteneurs**
- Proxmox VE 9.1.9 (LXC + QEMU/KVM) sur Debian Trixie
- Docker dans LXC dédié (112)
- Ollama (LXC 201 avec GPU passthrough 2× P5000)

**Stockage**
- TrueNAS Scale — ZFS RAID 1, compression LZ4

**Réseau & Sécurité**
- Cloudflare Zero Trust (Tunnel + DNS + Access)
- AdGuard Home (VM Freebox) — DNS filtrant + DNSSEC
- Nginx Proxy Manager — reverse proxy SSL
- WireGuard VPN (intégré Freebox)
- iptables kill switch VPN (qBittorrent)

**Services Applicatifs**
- Stack Média : Radarr, Sonarr, Prowlarr, qBittorrent, Plex
- Gitea (Git auto-hébergé)
- Vaultwarden (gestionnaire de mots de passe)
- Homebridge (HomeKit)
- Grafana + Uptime Kuma (monitoring)
- AgentDVR (surveillance vidéo)

---

**Dernière mise à jour** : 2026-04-14
**Données récupérées en direct** : APIs Proxmox, TrueNAS, NPM, Cloudflare, Freebox Delta, AdGuard
