# Pare-Feu et Contrôle d'Accès

Le réseau est sécurisé en interne par le **Pare-feu natif de Proxmox** (remplaçant UFW) et la protection dynamique **CrowdSec**.

## 1. Pare-feu Proxmox (PVE Firewall)

Géré au niveau du **Datacenter** et du **Nœud**, il bloque par défaut les connexions inter-VLANs.

### Règles d'Hôte (Host Firewall)
| Direction | Protocole | Port | Source | Action |
|:---|:---|:---|:---|:---|
| IN | TCP | `8006` (WebUI) | `192.168.1.0/24` (LAN) | **ACCEPT** |
| IN | TCP | `22` (SSH) | `192.168.1.0/24` (LAN) | **ACCEPT** |
| IN | TCP | `80`, `443` | *Any* | **ACCEPT** (Pour NPM) |
| IN | - | *All* | *Any* | **DROP** |

### Routage Inter-VLAN
Pour que le Reverse Proxy (NPM) puisse atteindre les services sans ouvrir le pare-feu global, le conteneur **Nginx Proxy Manager** dispose d'une interface réseau virtuelle (eth1, eth2...) dans **chaque VLAN**. Cela évite le routage inter-VLANs au niveau de l'hôte.

## 2. Défense Active : CrowdSec & Fail2Ban

### CrowdSec
- Bouncer configuré sur `nftables`.
- Liste blanche locale (`192.168.1.0/24`, `10.10.0.0/16`) pour éviter un auto-bannissement.
- **Statut actuel :** Plusieurs milliers d'adresses IP bloquées en temps réel grâce à la base de données collaborative.

### Fail2Ban
- Surveille activement le démon `sshd` de l'hôte.
- **Politique :** 3 échecs = Bannissement de 24 heures.

## 3. Sécurité du Conteneur qBittorrent (VPN Kill Switch)

Pour éviter toute fuite IP (DNS/Traffic leak), le conteneur 104 est verrouillé via `iptables` en interne :
- Politique `OUTPUT DROP` par défaut.
- Autorise uniquement le trafic vers l'interface `tun0` (Tunnel WireGuard/OpenVPN).
- Autorise les réponses au réseau local (LAN).
