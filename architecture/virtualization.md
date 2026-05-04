# Virtualisation & Compute

Le nœud Proxmox gère les conteneurs LXC et s'occupe de la distribution des ressources matérielles (CPU, RAM, GPU).

## Philosophie d'Hébergement

1. **Privilégier les LXC (LinuX Containers) :** 95% des services tournent dans des LXC pour maximiser les performances et minimiser l'empreinte mémoire.
2. **Sécurité "Unprivileged" :** Tous les conteneurs (à l'exception de l'inférence IA nécessitant le GPU) tournent avec un mappage d'UID/GID non privilégié.
3. **Infrastructure as Code (Scripts) :** Les conteneurs sont déployés via les scripts de la communauté (Proxmox Helper Scripts) pour garantir la reproductibilité.

## Répartition des Conteneurs (LXC)

| ID | Nom | VLAN | IP | Description |
|:---|:---|:---|:---|:---|
| **118** | `nginxproxymanager` | Management | `10.10.10.18` | Reverse Proxy. Interfaces sur tous les VLANs pour éviter le routage lourd. |
| **115** | `grafana` | Management | `10.10.10.10` | Dashboards et surveillance de l'hôte. |
| **202** | `jarvis` | Management | `10.10.10.20` | Assistant IA & Orchestrateur. |
| **104** | `qbittorrent` | Apps | `10.10.20.10` | Torrent avec VPN Windscribe. VPN Kill Switch forcé sur l'interface tun0. |
| **114** | `vaultwarden` | Apps | `10.10.20.14` | Mots de passe sécurisés (Backend Rust). |
| **120** | `gitea` | Apps | `10.10.20.20` | Forge Git légère. |
| **123** | `agentdvr` | Apps | `10.10.20.23` | NVR pour la surveillance caméra. |
| **130** | `media-hub` | Apps | `10.10.20.30` | La stack complète *Arr (Radarr, Sonarr, Prowlarr) et Petio. |
| **102** | `homebridge` | Apps | `10.10.20.102`| Pont domotique vers Apple HomeKit. |
| **112** | `docker` | Dev | `10.10.30.12` | Hôte Docker (Géré via Portainer). |
| **121** | `portfolio` | Dev | `10.10.30.21` | Environnement web de développement. |
| **201** | `inference` | IA | `10.10.40.10` | Modèles LLM via llama.cpp (Qwen 35B). **Passthrough de 2 GPU NVIDIA Quadro P5000**. |

## Hardware Passthrough (GPU)

Le conteneur `201` accède directement au matériel NVIDIA de l'hôte via des points de montage (`lxc.mount.entry`) vers `/dev/nvidia*`. Cela garantit des performances natives pour l'inférence des modèles de langage massifs.
