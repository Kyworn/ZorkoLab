# Architecture Réseau

L'architecture réseau est conçue pour garantir des performances optimales (10G) tout en isolant rigoureusement les environnements via des VLANs.

## Topologie L2 / L3

L'hyperviseur Proxmox gère le routage entre les différents sous-réseaux virtuels.

```mermaid
graph TD
    classDef hardware fill:#34495e,stroke:#fff,stroke-width:2px,color:#fff
    classDef vlan fill:#2980b9,stroke:#fff,stroke-width:2px,color:#fff

    WAN((Internet FTTH 10G)) --> BOX[Freebox Delta 192.168.1.1]:::hardware
    BOX --> SW[Switch Gigabit]:::hardware
    
    SW --> PVE[Proxmox Host 192.168.1.61]:::hardware
    SW --> NAS[TrueNAS 192.168.1.109]:::hardware
    
    subgraph Proxmox Bridges
        VMBR0[vmbr0: LAN 192.168.1.x]:::vlan
        VMBR10[vmbr10: Management 10.10.10.x]:::vlan
        VMBR20[vmbr20: Apps 10.10.20.x]:::vlan
        VMBR30[vmbr30: Dev 10.10.30.x]:::vlan
        VMBR40[vmbr40: IA 10.10.40.x]:::vlan
    end

    PVE --> VMBR0
    VMBR0 --> VMBR10
    VMBR0 --> VMBR20
    VMBR0 --> VMBR30
    VMBR0 --> VMBR40
    
    %% NAT / Routing
    VMBR10 -.->|NAT Masquerade| VMBR0
    VMBR20 -.->|NAT Masquerade| VMBR0
```

## Plan d'Adressage (IPAM)

| Sous-réseau | Rôle | Passerelle (Gateway) | Politique Pare-Feu |
|:---|:---|:---|:---|
| **`192.168.1.0/24`** | Réseau Physique (LAN) | `192.168.1.1` (Freebox) | Fait confiance au réseau local |
| **`10.10.10.0/24`** | VLAN 10 : Management | `10.10.10.1` (Proxmox) | Accès aux autres VLANs autorisé |
| **`10.10.20.0/24`** | VLAN 20 : Applications | `10.10.20.1` (Proxmox) | Isolé (Sortie Internet via NAT) |
| **`10.10.30.0/24`** | VLAN 30 : Développement | `10.10.30.1` (Proxmox) | Isolé |
| **`10.10.40.0/24`** | VLAN 40 : IA | `10.10.40.1` (Proxmox) | Isolé |

## Résolution DNS (AdGuard Home)

Toutes les requêtes DNS du réseau local pointent vers une VM AdGuard Home (`192.168.1.189`).
- **Filtrage :** 6 listes actives bloquant traqueurs et publicités (1.07M de règles).
- **DNS Local (Rewrites) :** Le domaine `*.zorko.xyz` est redirigé vers l'IP de Nginx Proxy Manager (`10.10.10.18`), qui se charge du reverse proxying vers le bon VLAN.
