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

Les règles datacenter autorisent l'administration SSH et HTTPS depuis le LAN. Des règles placées au niveau du nœud autorisent toutefois SSH, HTTP et HTTPS depuis toute source et élargissent donc la portée effective. Les ports 9100 et 111 font l'objet de règles LAN suivies d'un rejet, mais `rpc.statd` écoute également sur des ports dynamiques.

L'accès SSH root par mot de passe est désactivé. L'inventaire des clés a cependant retrouvé douze copies d'une même ancienne clé `jarvis@host202` et un accès root utilisé par l'automatisation Hermes. Trois identités API techniques subsistent, dont une identité Prometheus dotée du rôle global `PVEAdmin`. Un ancien fichier d'exporter contient aussi un secret en clair et doit être traité comme compromis avant suppression.

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
- limiter SSH et l'interface Proxmox au LAN et à Tailscale ;
- retirer les clés SSH décommissionnées et révoquer les comptes API orphelins ;
- appliquer les mises à jour de sécurité en attente après validation ;
- revoir les règles HTTP/HTTPS ouvertes au niveau hôte ;
- documenter séparément les politiques Cloudflare Access si elles doivent faire partie du modèle de confiance.
