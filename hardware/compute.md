# Matériel : Serveurs de Calcul

Ce document inventorie le matériel physique de l'hyperviseur Proxmox.

## Nœud Proxmox VE (Compute)

| Spécification | Détail |
|:---|:---|
| **CPU** | AMD Ryzen 5 5600X (6 cœurs / 12 threads, jusqu'à 4.6 GHz) |
| **RAM** | 32 GB DDR4 |
| **Châssis** | Corsair 680X (Tour ATX) |
| **Accélération Matérielle** | 2× NVIDIA Quadro P5000 (16 GB VRAM chacune) — PCI Passthrough |
| **Stockage OS** | 512 GB NVMe SSD (LVM-thin) |
| **Système d'Exploitation** | Proxmox VE 9.1.9 (Debian Trixie 13) |
| **Noyau (Kernel)** | 7.0.0-3-pve |
| **Réseau** | IP: `192.168.1.61` (LAN) |

## Allocation des Ressources (LXC)

*Tous les conteneurs sont configurés en mode "Unprivileged" (sauf LXC 201).*

| ID | Application | IP | VLAN | Ressources Spécifiques |
|:---|:---|:---|:---|:---|
| **102** | Homebridge | `10.10.20.102` | Apps | - |
| **104** | qBittorrent | `10.10.20.10` | Apps | Kill Switch iptables |
| **112** | Docker Host | `10.10.30.12` | Dev | Nested Virtualization |
| **114** | Vaultwarden | `10.10.20.14` | Apps | - |
| **115** | Grafana | `10.10.10.10` | Mgmt | - |
| **118** | Nginx Proxy Manager | `10.10.10.18` | Mgmt | Interfaces eth1, eth2, eth3 |
| **120** | Gitea | `10.10.20.20` | Apps | - |
| **121** | Portfolio | `10.10.30.21` | Dev | - |
| **123** | AgentDVR | `10.10.20.23` | Apps | - |
| **130** | Media-Hub | `10.10.20.30` | Apps | - |
| **201** | Inference (llama.cpp) | `10.10.40.10` | IA | **2x GPU Quadro P5000** |
| **202** | Jarvis (Hermes) | `10.10.10.20` | Mgmt | - |

## Points de Montage Proxmox (LVM & NFS)

| Volume | Type | Utilisation |
|:---|:---|:---|
| `local` | Répertoire | Stockage ISOs et Templates |
| `nvme-biwin-storage` | LVM-thin | Disques racines des LXC / VM |
| `Backup` | NFS | Monté depuis TrueNAS (VZDump) |
| `Film`, `Series`, `Download` | NFS | Montés depuis TrueNAS (Médias) |
