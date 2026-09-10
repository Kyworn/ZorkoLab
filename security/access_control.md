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

L'accès SSH root par mot de passe est désactivé. Trois clés seulement restent autorisées : l'accès courant `zorko-proxmox`, une clé de secours `zmac` et la clé `jarvis@openclaw` nécessaire aux contrôles Hermes. Les anciennes clés, dont douze copies de `jarvis@host202`, ont été retirées.

Seul `root` conserve un shell Linux sur l'hôte. Les comptes locaux inutilisés `zorko`, `crowdsec` et `git` ont été supprimés. Les deux anciens utilisateurs Proxmox techniques, le token `OC_Jarvis`, leurs ACL et les fichiers de l'ancien exporter contenant un secret en clair ont également été retirés. L'ancien dépôt `infra-bus` a été exporté et vérifié sur TrueNAS avant sa suppression du nœud.

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
- remplacer à terme la clé root Hermes par un accès limité aux contrôles nécessaires ;
- appliquer les mises à jour de sécurité en attente après validation ;
- revoir les règles HTTP/HTTPS ouvertes au niveau hôte ;
- documenter séparément les politiques Cloudflare Access si elles doivent faire partie du modèle de confiance.
