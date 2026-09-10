# Virtualisation

## Proxmox VE

L'hôte `pve` exécute Proxmox VE 9.2.11 sur Debian 13, avec le noyau `7.0.14-14-pve` lors de l'audit. Les workloads de production sont des conteneurs LXC non privilégiés.

## Inventaire LXC

| ID | Nom | IP | CPU | RAM | État | Rôle |
|---:|:---|:---|---:|---:|:---:|:---|
| 102 | homebridge | `.15`, `.102` | 4 | 5 GiB | actif | Homebridge, Beszel agent |
| 104 | qbittorrent | `.10` | 4 | 2 GiB | actif | qBittorrent sous Gluetun |
| 110 | ios-node | `.154` | 1 | 512 MiB | actif | AltServer, Anisette, pymobiledevice3 |
| 112 | docker | `.20` | 5 | 4 GiB | actif | Veille Sociale, SearXNG, Umami, Portainer, FlareSolverr |
| 114 | vaultwarden | `.14` | 4 | 6 GiB | actif | Vaultwarden, Beszel agent |
| 118 | nginxproxymanager | `.18` | 2 | 1 GiB | actif | reverse proxy NPM/OpenResty |
| 119 | adguard | `.12` | 1 | 1 GiB | actif | AdGuard Home |
| 120 | gitea | `.16` | 2 | 2 GiB | actif | Gitea, MariaDB, stockage NFS |
| 121 | portfolio | `.21` | 1 | 512 MiB | actif | portfolio Docker |
| 122 | azerothdb | `.184` | 4 | 4 GiB | actif | AzerothDB, MariaDB, Nginx |
| 125 | codeman | `.125` | 2 | 2 GiB | actif | Codeman Web |
| 126 | ntfy | `.126` | 1 | 512 MiB | actif | notifications ntfy |
| 128 | beszel | `.128` | 2 | 1 GiB | actif | hub Beszel |
| 129 | edge-tunnel | `.129` | 1 | 512 MiB | actif | cloudflared |
| 130 | media-hub | `.13` | 4 | 4 GiB | actif | Sonarr, Radarr, Bazarr, Prowlarr, Seerr, Shelfmark |
| 140 | sparky | `.140` | 1 | 1 GiB | arrêté | environnement Sparky |
| 202 | jarvis | `.17` | 4 | 8 GiB | arrêté | ancien Jarvis/Hermes |
| 203 | jarvis2 | `.22` | 4 | 8 GiB | actif | Hermes Gateway |
| 210 | llm-gateway | `.210` | 2 | 1 GiB | actif | passerelle LLM Docker |
| 211 | nex-llm | `.211` | 8 | 16 GiB | actif | llama.cpp, 2× Quadro P5000 |

Les IP abrégées appartiennent toutes à `192.168.1.0/24`. Total : **20 LXC, 18 actifs et 2 arrêtés**.

## GPU

Le LXC 211 reçoit les deux GPU par bind mounts de devices tout en restant non privilégié. `nvidia-smi` fonctionne dans le conteneur et `llama-server.service` est actif. Il s'agit d'un partage de périphériques LXC, pas d'un PCI passthrough vers une VM.

## Machines virtuelles

Une VM `9000`, `debian12-cloudinit`, est arrêtée. Sa petite configuration et son nom indiquent un template cloud-init plutôt qu'un workload de production.
