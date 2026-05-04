# Matériel : Réseau & Connectivité

## Internet Edge (Routeur)

La passerelle Internet principale est gérée par une **Freebox Delta v7**.

| Caractéristique | Spécification |
|:---|:---|
| **Connexion Physique** | Fibre Optique FTTH 10 Gbps (Port SFP) |
| **Débit Mesuré (Crête)** | 10 Gbps ↓ / 900 Mbps ↑ |
| **Firmware** | 4.9.18.1 |
| **Passerelle LAN** | `192.168.1.1` |
| **Fonctionnalités Actives** | DHCP, WireGuard VPN (Serveur distant) |

## Adressage & DNS (AdGuard Home)

Toute la résolution DNS de la maison est confiée à une Machine Virtuelle (VM) hébergée directement sur le processeur de la Freebox Delta.

| Paramètre | Configuration |
|:---|:---|
| **IP DNS** | `192.168.1.189` |
| **Règles de Filtrage** | 6 Listes / +1 077 000 domaines bloqués |
| **Protocoles** | DNSSEC Actif, DoH (DNS over HTTPS) Upstream |
| **Rewrites (Locaux)** | Résolution de `*.zorko.xyz` vers Nginx Proxy Manager (`10.10.10.18`) |

## Équipements de Couche 2 (L2)

*   **Switch Principal :** HPE Gigabit Switch (IP Admin: `192.168.1.77`). Relie la Freebox au Proxmox et au TrueNAS.
*   **Infrastructure Sans-Fil (Wi-Fi 7) :**
    *   3× Points d'Accès **TP-Link Deco BE25** (Mesh).
    *   Le Wi-Fi natif de la Freebox est **désactivé** pour éviter les interférences et bénéficier de l'itinérance (roaming) parfaite des Deco en bande 6GHz.
