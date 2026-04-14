# Matériel : Serveurs de Calcul

Ce document liste le matériel physique utilisé pour l'hébergement des machines virtuelles et des conteneurs.

## Serveur Proxmox VE

| Composant | Modèle / Spécification | Notes |
|-----------|------------------------|-------|
| **CPU** | AMD Ryzen 5 5600X | 6 cores / 12 threads — jusqu'à 4.6 GHz |
| **RAM** | 32 GB DDR4 | ~6.4 GB utilisés en prod |
| **Châssis** | Corsair 680X | Tour ATX |
| **GPU** | 2× NVIDIA Quadro P5000 | 16 GB VRAM chacune — passthrough vers LXC 201 |
| **OS** | Proxmox VE 9.1.7 | Debian Trixie (13), kernel 6.17.4-2-pve |
| **Stockage OS** | 512 GB NVMe (nvme-biwin) | LVM-thin, 71% utilisé |
| **IP** | 192.168.1.61:8006 | |
| **Uptime** | ~9 jours | Stable |

## Charges de Travail sur Proxmox

### Conteneurs LXC (12 actifs)

| CT | Nom | IP | Rôle |
|----|-----|----|------|
| 102 | homebridge | 192.168.1.13 | Domotique HomeKit |
| 104 | qbittorrent | 192.168.1.52 | Client torrent (VPN Windscribe, kill switch) |
| 112 | docker | 192.168.1.62 | Hôte Docker (Portainer) |
| 113 | passbolt | — | Gestionnaire de mots de passe équipe (stopped) |
| 114 | vaultwarden | 192.168.1.110 | Gestionnaire de mots de passe personnel |
| 115 | grafana | 192.168.1.194 | Monitoring / métriques |
| 118 | nginxproxymanager | 192.168.1.186 | Reverse proxy SSL |
| 120 | gitea | 192.168.1.93 | Git auto-hébergé |
| 121 | portfolio | 192.168.1.155 | Site portfolio |
| 123 | agentdvr | 192.168.1.39 | Surveillance vidéo |
| 130 | media-hub | 192.168.1.51 | Radarr + Sonarr + Prowlarr + Petio |
| 201 | inference | 192.168.1.11 | Ollama (2× P5000 GPU) |

### Machines Virtuelles

| VM | Nom | Statut | Usage |
|----|-----|--------|-------|
| 9000 | debian12-cloudinit | stopped | Template cloud-init |

### Stockage Proxmox

| Volume | Type | Taille | Utilisation |
|--------|------|--------|-------------|
| local | dir | 203 GB | 17.3% (OS + ISOs) |
| nvme-biwin-storage | LVM-thin | 284 GB | 71.1% (disques LXC/VM) |
| Backup (NFS TrueNAS) | nfs | 470 GB | 39.5% |
| Film (NFS TrueNAS) | nfs | 477 GB | 40.4% |
| Series (NFS TrueNAS) | nfs | 371 GB | 23.3% |
| Download (NFS TrueNAS) | nfs | 336 GB | 15.4% |
| gitea (NFS TrueNAS) | nfs | 286 GB | 0.6% |

---

**Dernière mise à jour** : 2026-04-14
**Données récupérées via** : API Proxmox + SSH
