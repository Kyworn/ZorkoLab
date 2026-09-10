# Contrôle d'accès

## Défenses actives sur Proxmox

| Composant | État vérifié |
|:---|:---|
| Pare-feu Proxmox, niveau cluster | activé |
| Pare-feu Proxmox, niveau nœud | activé |
| `proxmox-firewall` | actif |
| `pve-firewall` | actif |
| CrowdSec | actif, version 1.7.8 |
| Fail2Ban | actif, jail `sshd` |

Les règles observées autorisent l'administration SSH et HTTPS depuis le LAN. Les ports de supervision et RPC font l'objet de règles LAN suivies d'un rejet. Des règles HTTP/HTTPS plus larges existent également et doivent être évaluées avec la politique réelle du routeur et du tunnel.

## Isolation des workloads

Les 18 conteneurs LXC sont non privilégiés. L'accès aux GPU du LXC 211 passe par des devices explicitement montés. Certains conteneurs montent des exports NFS TrueNAS nécessaires à leur rôle. L'agent Beszel auparavant installé dans le LXC Vaultwarden a été désactivé afin de réduire le périmètre du gestionnaire de mots de passe.

Le réseau est actuellement plat. Il n'existe donc pas d'isolation VLAN entre les services : toute description de « défense en profondeur » doit tenir compte de cette limite.

## Protection applicative

- l'exposition web publique passe par Cloudflare Tunnel et Nginx Proxy Manager ;
- AdGuard Home filtre le DNS du LAN ;
- qBittorrent tourne derrière Gluetun ;
- Vaultwarden est isolé dans son propre LXC.

Les politiques Cloudflare Access, les redirections du routeur et les droits détaillés de chaque application n'ont pas été exportés pendant cet audit. Ils ne sont donc pas présentés ici comme vérifiés.

## Points de suivi

- retirer ou mettre à jour les règles historiques pour `10.10.0.0/16` ;
- appliquer les mises à jour de sécurité en attente après validation ;
- revoir les règles HTTP/HTTPS ouvertes au niveau hôte ;
- documenter séparément les politiques Cloudflare Access si elles doivent faire partie du modèle de confiance.
