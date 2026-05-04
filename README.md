# 🏠 ZorkoLab : Infrastructure & Homelab

> Architecture d'hébergement privé haute performance, orientée sécurité (Zero-Trust), virtualisation et intelligence artificielle.

![Proxmox](https://img.shields.io/badge/Proxmox_VE-9.1.9-E57000?style=for-the-badge&logo=proxmox&logoColor=white)
![Debian](https://img.shields.io/badge/Debian_13_Trixie-A81D33?style=for-the-badge&logo=debian&logoColor=white)
![TrueNAS](https://img.shields.io/badge/TrueNAS_Scale-26.0_BETA-0095D5?style=for-the-badge&logo=truenas&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Zero_Trust-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)
![ZFS](https://img.shields.io/badge/ZFS_RAID_1-00979D?style=for-the-badge)

---

## 🏗️ Architecture du Système

L'infrastructure est bâtie sur un modèle hyper-convergé séparant logiquement le stockage (TrueNAS) du calcul (Proxmox), tout en assurant une isolation stricte des services via des VLANs.

```mermaid
graph TD
    %% Styles
    classDef cloud fill:#F38020,stroke:#fff,stroke-width:2px,color:#fff
    classDef pve fill:#E57000,stroke:#fff,stroke-width:2px,color:#fff
    classDef truenas fill:#0095D5,stroke:#fff,stroke-width:2px,color:#fff
    classDef lxc fill:#4CAF50,stroke:#fff,stroke-width:2px,color:#fff
    classDef vlan fill:#2c3e50,stroke:#bdc3c7,stroke-width:2px,color:#fff,stroke-dasharray: 5 5

    subgraph Internet ["🌐 Internet Edge"]
        CF["☁️ Cloudflare Zero Trust<br/>(WAF, DDoS Protection, Access)"]:::cloud
    end

    subgraph Network ["🏠 Réseau Domestique (Freebox Delta 10G)"]
        ADG["🛡️ AdGuard Home<br/>(DNS Filtrant + DNSSEC)"]:::lxc
    end

    subgraph Proxmox ["⚙️ Proxmox VE (Compute Node) — 192.168.1.61"]
        CF_TUN["🔒 Cloudflared Tunnel<br/>(Host Daemon)"]:::pve

        subgraph VLAN10 ["🟦 VLAN 10 : Management (10.10.10.x)"]
            NPM["🔀 Nginx Proxy Manager (118)"]:::lxc
            GRAF["📈 Grafana (115)"]:::lxc
            JARVIS["🤖 Jarvis/Hermes (202)"]:::lxc
        end

        subgraph VLAN20 ["🟩 VLAN 20 : Applications (10.10.20.x)"]
            VAULT["🔒 Vaultwarden (114)"]:::lxc
            GIT["🗂️ Gitea (120)"]:::lxc
            MEDIA["🎬 Media Hub (130)"]:::lxc
            QBIT["⬇️ qBittorrent (104)"]:::lxc
            HB["🏡 Homebridge (102)"]:::lxc
        end

        subgraph VLAN30 ["🟨 VLAN 30 : Développement (10.10.30.x)"]
            DOCKER["🐋 Docker Host (112)"]:::lxc
            PORTFOLIO["🌐 Portfolio (121)"]:::lxc
        end

        subgraph VLAN40 ["🟪 VLAN 40 : Intelligence Artificielle (10.10.40.x)"]
            INFER["🧠 Inference llama.cpp (201)<br/>(Passthrough 2x P5000)"]:::lxc
        end
    end

    subgraph Storage ["💿 TrueNAS Scale (Storage Node) — 192.168.1.109"]
        ZFS["🗄️ ZFS Pool: Tank (RAID 1)<br/>932 GB"]:::truenas
    end

    %% Routing
    CF -->|Trafic Web Sécurisé| CF_TUN
    CF_TUN -->|Reverse Proxy HTTP/S| NPM
    
    NPM --> VLAN10
    NPM --> VLAN20
    NPM --> VLAN30
    NPM --> VLAN40
    
    VLAN20 -.->|NFS Mounts| ZFS
    VLAN10 -.->|Backups| ZFS
```

---

## 📊 État de l'Infrastructure

| Catégorie | Détail | Statut / Métrique |
|:---|:---|:---|
| **Réseau** | Freebox Delta (FTTH) | **10 Gbps ↓** / **900 Mbps ↑** |
| **Sécurité** | CrowdSec, Proxmox Firewall, AdGuard | **Actifs & Durcis** |
| **Compute** | AMD Ryzen 5 5600X (6C/12T) / 32 GB RAM | ~20% d'utilisation RAM |
| **GPU** | 2× NVIDIA Quadro P5000 (16 GB) | Allouées à l'IA (LXC 201) |
| **Containers** | 12 LXC "Unprivileged" | Production 24/7 |
| **Stockage** | ZFS Miroir (TrueNAS) | **661 GB** / 932 GB (70.9% utilisés) |

---

## 🛡️ Sécurité & Isolation

Le système a été conçu avec une approche **Zero-Trust** et "Défense en Profondeur" :

1. **Aucun port entrant ouvert** sur le routeur. Tout le trafic externe transite via un **Tunnel Cloudflare**.
2. **Isolation L2/L3** : Les conteneurs sont répartis dans des **VLANs stricts** gérés par le pare-feu natif de Proxmox.
3. **Hardening Système** :
   - Tous les conteneurs sont en mode **Unprivileged**.
   - Permissions durcies sur les crons et fichiers sensibles (`/etc/shadow`).
   - Protection contre l'IP Spoofing (`rp_filter` strict).
4. **Active Defense** :
   - **CrowdSec** bloque en temps réel les IPs malveillantes via des listes communautaires et `nftables`.
   - **Fail2Ban** protège les accès SSH locaux.

---

## 🌐 Services Accessibles (NPM & Cloudflare)

Plus de 31 services sont routés via Nginx Proxy Manager. Voici les principaux :

| Service | Endpoint Interne | Domaine | Accès |
|:---|:---|:---|:---|
| **Proxmox VE** | `10.10.10.1:8006` | `pve.zorko.xyz` | 🔒 LAN Only |
| **TrueNAS** | `192.168.1.109:80` | `nas.zorko.xyz` | 🔒 LAN Only |
| **Vaultwarden**| `10.10.20.14:8000` | `vault.zorko.xyz` | 🌍 Public (WAF Auth) |
| **Gitea** | `10.10.20.20:3000` | `git.zorko.xyz` | 🌍 Public (WAF Auth) |
| **Grafana** | `10.10.10.10:3000` | `grafana.zorko.xyz` | 🔒 LAN Only |
| **Media Hub** | `10.10.20.30:xxxx` | `radarr.*`, `sonarr.*` | 🔒 LAN Only |
| **qBittorrent**| `10.10.20.10:8090` | `qbit.zorko.xyz` | 🔒 LAN Only |

> *La résolution DNS interne (`*.zorko.xyz` -> `10.10.10.18`) est assurée par AdGuard Home.*

---

## 📚 Documentation Détaillée

Explorez les spécifications techniques de chaque brique de l'infrastructure :

### 🏛️ Architecture logicielle
- [Virtualisation (Proxmox & LXC)](./architecture/virtualization.md)
- [Stockage & ZFS](./architecture/storage.md)
- [Réseau & VLANs](./architecture/network.md)

### 💻 Matériel
- [Serveurs de Calcul (Compute)](./hardware/compute.md)
- [Nœud de Stockage (NAS)](./hardware/storage.md)
- [Équipements Réseau](./hardware/network.md)

### 🔐 Sécurité & Automatisation
- [Règles Pare-feu & Contrôle d'Accès](./security/access_control.md)
- [Cloudflare Zero Trust](./security/cloudflare_zero_trust.md)
- [Stratégie de Sauvegarde](./automation/backups.md)
