# ZorkoLab

> Homelab self-hosted autour de Proxmox, TrueNAS, Cloudflare et de l'inférence LLM locale.

![Proxmox VE](https://img.shields.io/badge/Proxmox_VE-9.2.11-E57000?style=flat-square&logo=proxmox&logoColor=white)
![TrueNAS](https://img.shields.io/badge/TrueNAS-26.0.0--BETA.3-0095D5?style=flat-square&logo=truenas&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-12%20%7C%2013-A81D33?style=flat-square&logo=debian&logoColor=white)
![ZFS](https://img.shields.io/badge/ZFS-mirror-2F5F8F?style=flat-square)
![Audit](https://img.shields.io/badge/audit-2026--09--10-2ea44f?style=flat-square)

Ce dépôt décrit l'infrastructure telle qu'elle fonctionne réellement. Les chiffres ci-dessous viennent d'un audit direct de Proxmox, des conteneurs, de Nginx Proxy Manager et de TrueNAS réalisé le **10 septembre 2026**.

## Vue d'ensemble

```mermaid
flowchart LR
    Internet((Internet)) --> CF[Cloudflare Tunnel]
    CF --> EDGE[CT 129<br/>cloudflared]
    EDGE --> NPM[CT 118<br/>Nginx Proxy Manager]
    NPM --> APPS[Services LXC et Docker]

    LAN[LAN 192.168.1.0/24] --> PVE[Proxmox VE<br/>Ryzen 5 5600X]
    LAN --> NAS[TrueNAS<br/>Intel N100]
    PVE --> CT[18 LXC<br/>tous actifs]
    PVE --> GPU[CT 211<br/>2× Quadro P5000]
    NAS --> ZFS[Tank<br/>miroir 2× 1 To]
    ZFS -. NFS .-> PVE
```

| Brique | État audité |
|:---|:---|
| Compute | Ryzen 5 5600X, 32 GiB de RAM, Proxmox VE 9.2.11 |
| Accélération | 2× Quadro P5000 16 GiB affectées au LXC 211 |
| Virtualisation | 18 LXC actifs, plus un template VM arrêté |
| Exposition web | 19 hôtes proxy actifs sur 24 configurés dans NPM |
| Stockage | miroir ZFS de 920 GiB utilisables, 34,9 % occupés, aucune erreur |
| Réseau | LAN unique `192.168.1.0/24`, passerelle `192.168.1.254` |
| Défense hôte | pare-feu Proxmox, CrowdSec et Fail2Ban actifs |
| Supervision | audit quotidien à 06:00 et heartbeat hebdomadaire via ntfy |

## Services

| Domaine | Services principaux |
|:---|:---|
| Edge et réseau | cloudflared, Nginx Proxy Manager, AdGuard Home |
| Développement | Gitea, Codeman, Portfolio, Docker/Portainer |
| Média | qBittorrent + Gluetun, Sonarr, Radarr, Bazarr, Prowlarr, Seerr, Shelfmark |
| Maison | Homebridge |
| IA | llama.cpp sur 2× P5000, LLM Gateway, Jarvis 2 / Hermes |
| Sécurité et suivi | Vaultwarden, Beszel, CrowdSec, Fail2Ban, ntfy |
| Données | AzerothDB, Umami, SearXNG, Veille Sociale |

L'inventaire détaillé, avec IDs, ressources et états, se trouve dans [architecture/virtualization.md](./architecture/virtualization.md).

## État opérationnel au 10 septembre 2026

Le calcul, le réseau, le tunnel Cloudflare et le pool ZFS sont opérationnels. L'audit a toutefois relevé des points à traiter :

- le pool LVM-thin Proxmox est occupé à 49,1 % après le retrait des LXC décommissionnés, la suppression de snapshots obsolètes et un TRIM complet ;
- la nouvelle politique VZDump a été testée avec succès sur les groupes quotidien et hebdomadaire ;
- le LXC 211 ne reçoit volontairement aucun VZDump, car ses modèles et son build sont reconstruisibles ;
- la tâche de snapshots TrueNAS du dataset `backup` est désactivée ;
- 14 mises à jour de sécurité Debian sont en attente sur l'hôte ;
- `nvidia-persistenced.service` est en échec, même si les deux GPU et llama.cpp fonctionnent ;
- une tâche Hermes crée encore des snapshots locaux quotidiens et doit être désactivée pour laisser VZDump seul responsable des sauvegardes ;
- les LXC en démarrage automatique totalisent 42 GiB de limites mémoire sur 32 GiB physiques ;
- d'anciennes règles pare-feu visant `10.10.0.0/16` subsistent et les règles d'administration du nœud sont trop larges.

Le premier passage de simplification a supprimé les LXC décommissionnés 140 et 202, retiré 14 snapshots automatiques obsolètes, exécuté un TRIM complet, organisé les LXC avec des tags et désactivé l'agent Beszel présent à côté de Vaultwarden. Les dernières archives des deux anciens LXC restent temporairement sur TrueNAS.

Le détail global, les preuves vérifiées et les limites de l'inspection sont consignés dans [AUDIT-2026-09-10.md](./AUDIT-2026-09-10.md). Le contrôle approfondi de l'hôte se trouve dans [PVE-HOST-AUDIT-2026-09-10.md](./PVE-HOST-AUDIT-2026-09-10.md).

## Documentation

| Sujet | Document |
|:---|:---|
| Audit approfondi du nœud Proxmox | [PVE-HOST-AUDIT-2026-09-10.md](./PVE-HOST-AUDIT-2026-09-10.md) |
| Topologie réseau | [architecture/network.md](./architecture/network.md) |
| Conteneurs et virtualisation | [architecture/virtualization.md](./architecture/virtualization.md) |
| Stockage et datasets | [architecture/storage.md](./architecture/storage.md) |
| Nœud de calcul | [hardware/compute.md](./hardware/compute.md) |
| Nœud TrueNAS | [hardware/storage.md](./hardware/storage.md) |
| Matériel réseau | [hardware/network.md](./hardware/network.md) |
| Contrôle d'accès | [security/access_control.md](./security/access_control.md) |
| Cloudflare et reverse proxy | [security/cloudflare_zero_trust.md](./security/cloudflare_zero_trust.md) |
| Sauvegardes et contrôles | [automation/backups.md](./automation/backups.md) |

## Portée

Ce dépôt est une documentation d'exploitation, pas une promesse de haute disponibilité. Le miroir ZFS protège contre la panne d'un disque, mais ne remplace pas une sauvegarde hors site. Les états et taux d'occupation sont des instantanés datés et évolueront avec le lab.
