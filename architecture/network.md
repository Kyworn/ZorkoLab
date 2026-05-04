# Architecture Réseau

Topologie réseau du homelab — flux de trafic externes (Cloudflare Zero Trust) et internes (NPM + AdGuard DNS).

## Diagramme des Flux Réseau

```mermaid
graph TD
    subgraph INTERNET["🌐 Internet"]
        U_EXT[Utilisateur Externe]
    end

    subgraph LAN["🏠 LAN 192.168.1.0/24"]
        U_INT[Utilisateur Interne]
    end

    subgraph CF["☁️ Cloudflare"]
        CF_ACCESS[Access / WAF]
        CF_TUNNEL[Tunnel zserv]
    end

    subgraph FBX["📡 Freebox Delta"]
        ROUTER[Routeur — 192.168.1.1]
        AG_VM[🛡️ AdGuard Home VM<br/>192.168.1.189:80<br/>DNS + DNSSEC]
    end

    subgraph PVE["⚙️ Proxmox — 192.168.1.61"]
        CT110[Cloudflared — CT110]
        CT118[Nginx Proxy Manager<br/>CT118 — 10.10.10.18]

        subgraph SERVICES["Services LXC"]
            CT130[Media Hub — 10.10.20.30<br/>Radarr · Sonarr · Prowlarr]
            CT104[qBittorrent — 10.10.20.10]
            CT114[Vaultwarden — 10.10.20.14]
            CT120[Gitea — 10.10.20.20]
            CT115[Grafana — 10.10.10.10]
            CT_AUTRES[... autres services]
        end
    end

    %% Flux externe via Cloudflare
    U_EXT -->|HTTPS| CF_ACCESS
    CF_ACCESS -->|Tunnel chiffré| CF_TUNNEL
    CF_TUNNEL -->|Connexion sortante| CT110
    CT110 --> CT118

    %% Flux interne via NPM
    U_INT -->|"*.zorko.xyz"| AG_VM
    AG_VM -->|"wildcard → 10.10.10.18"| CT118

    %% Routage NPM
    CT118 --> CT130 & CT104 & CT114 & CT120 & CT115 & CT_AUTRES

    %% DNS Freebox → AdGuard
    ROUTER -.DNS primaire.-> AG_VM
```

## Description des Flux

### Flux Externe (Cloudflare Zero Trust)

1. L'utilisateur accède à `vault.zorko.xyz` ou `plex.zorko.xyz` depuis Internet
2. **Cloudflare Access** vérifie l'authentification (email OTP, session 24h)
3. Le **Tunnel zserv** achemine la requête de façon chiffrée vers **Cloudflared** (CT 110) — aucun port ouvert sur la Freebox
4. **Cloudflared** transmet vers **NPM** qui route vers le service cible

### Flux Interne (LAN → AdGuard → NPM)

1. L'utilisateur sur le LAN accède à `radarr.zorko.xyz`
2. **AdGuard Home** (DNS Freebox) résout `*.zorko.xyz → 10.10.10.18` (wildcard)
3. **NPM** reçoit la requête, termine le SSL, route vers le service interne
4. Accès direct au service sans passer par Cloudflare

### VPN (accès administration)

- **WireGuard** intégré Freebox — accès complet au réseau 192.168.1.0/24
- Utilisé pour administration Proxmox (port 8006), TrueNAS, SSH

## Tableau de Routage NPM (17 hôtes)

| Domaine | Backend | Accessible depuis |
|---------|---------|-------------------|
| ad.zorko.xyz | 192.168.1.189:80 | LAN only |
| cockpit.zorko.xyz | 10.10.10.1:9090 | LAN only |
| git.zorko.xyz | 10.10.20.20:3000 | LAN + Cloudflare |
| grafana.zorko.xyz | 10.10.10.10:3000 | LAN only |
| home.zorko.xyz | 10.10.20.102:8581 | LAN only |
| kuma.zorko.xyz | 192.168.1.42:3001 | LAN only |
| nas.zorko.xyz | 192.168.1.109:80 | LAN only |
| npm.zorko.xyz | 10.10.10.18:81 | LAN only |
| petio.zorko.xyz | 10.10.20.30:5055 | LAN + Cloudflare |
| plex.zorko.xyz | 192.168.1.108:32400 | LAN + Cloudflare |
| port.zorko.xyz | 10.10.30.12:9443 | LAN only |
| prow.zorko.xyz | 10.10.20.30:9696 | LAN only |
| pve.zorko.xyz | 10.10.10.1:8006 | LAN only |
| qbit.zorko.xyz | 10.10.20.10:8090 | LAN only |
| radarr.zorko.xyz | 10.10.20.30:7878 | LAN only |
| sonarr.zorko.xyz | 10.10.20.30:8989 | LAN only |
| vault.zorko.xyz | 10.10.20.14:8000 | LAN + Cloudflare |

---

**Dernière mise à jour** : 2026-04-14
**Données récupérées via** : NPM API + AdGuard API + Freebox API
