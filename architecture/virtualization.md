# Virtualisation

## Proxmox VE

L'hôte `pve` exécute Proxmox VE 9.2.11 sur Debian 13, avec le noyau `7.0.14-14-pve` lors de l'audit. Les workloads de production sont des conteneurs LXC non privilégiés.

## Convention d'organisation

Les IDs restent stables et ne codent aucune fonction. Renuméroter un LXC n'apporterait aucun bénéfice opérationnel et pourrait casser des références externes. L'organisation visible dans Proxmox repose sur un seul tag fonctionnel par conteneur :

| Tag | LXC | Rôle |
|:---|:---|:---|
| `core` | 102, 114, 118, 119, 120, 126, 129 | services structurants du lab |
| `apps` | 110, 112, 121, 122, 125, 128 | applications et outils |
| `media` | 104, 130 | téléchargement et gestion média |
| `ai` | 203, 210, 211 | assistants, gateway et inférence |

La politique de sauvegarde utilise un second tag indépendant : `backup-daily`, `backup-weekly` ou `backup-none`. Le rôle fonctionnel ne sert donc jamais à déduire implicitement la protection du LXC.

Chaque LXC possède également une note opérationnelle courte dans Proxmox. Le format reste identique partout : rôle, emplacement des données, protection actuelle, dépendances et éventuel point d'attention. Les anciennes descriptions publicitaires générées par les scripts d'installation ont été retirées.

## Inventaire LXC

| ID | Nom | IP | CPU | RAM | État | Rôle |
|---:|:---|:---|---:|---:|:---:|:---|
| 102 | homebridge | `.15`, `.102` | 4 | 5 GiB | actif | Homebridge, Beszel agent |
| 104 | qbittorrent | `.10` | 4 | 2 GiB | actif | qBittorrent sous Gluetun |
| 110 | ios-node | `.154` | 1 | 512 MiB | actif | AltServer, Anisette, pymobiledevice3 |
| 112 | docker | `.20` | 5 | 4 GiB | actif | Veille Sociale, SearXNG, Umami, Portainer, FlareSolverr |
| 114 | vaultwarden | `.14` | 4 | 6 GiB | actif | Vaultwarden |
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
| 203 | jarvis2 | `.22` | 4 | 8 GiB | actif | Hermes Gateway |
| 210 | llm-gateway | `.210` | 2 | 1 GiB | actif | passerelle LLM Docker |
| 211 | nex-llm | `.211` | 8 | 16 GiB | actif | llama.cpp, 2× Quadro P5000 |

Les IP abrégées appartiennent toutes à `192.168.1.0/24`. Total : **18 LXC actifs**.

## GPU

Le LXC 211 reçoit les deux GPU par bind mounts de devices tout en restant non privilégié. `nvidia-smi` fonctionne dans le conteneur et `llama-server.service` est actif. Il s'agit d'un partage de périphériques LXC, pas d'un PCI passthrough vers une VM.

## Machines virtuelles

Une VM `9000`, `debian12-cloudinit`, est arrêtée. Sa petite configuration et son nom indiquent un template cloud-init plutôt qu'un workload de production.
