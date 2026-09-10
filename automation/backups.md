# Sauvegardes et contrôles automatiques

## VZDump Proxmox

| Paramètre | Configuration actuelle |
|:---|:---|
| Fréquence | tous les jours à 04:00 |
| Destination | stockage NFS `Backup` sur TrueNAS |
| Mode | snapshot |
| Compression | Zstandard |
| Rétention | dernier backup uniquement |
| Workloads inclus | 102, 104, 110, 112, 114, 118, 119, 120, 121, 122, 125, 126, 128, 129, 130 |

Les LXC 203, 210 et 211 ne sont pas inclus. Les anciens LXC 140 et 202 ont été retirés du job lors de leur décommissionnement. Les jobs des 9 et 10 septembre 2026 se sont terminés avec des erreurs. Le log du 10 septembre relie l'échec à l'impossibilité de créer de nouveaux snapshots LVM-thin lorsque le seuil d'espace libre est atteint.

## Snapshots TrueNAS

| Dataset | Fréquence | Rétention | État au 10 septembre |
|:---|:---|:---|:---|
| `Tank/server/backup` | quotidien à 12:00 | 6 jours | désactivé, dernier run 8 jours auparavant |
| `Tank/server/project/gitea` | quotidien à 12:00 | 6 jours | actif |
| `Tank/share` | quotidien à 12:00 | 6 jours | actif |

Aucune tâche de réplication, rsync ou cloud sync n'était visible dans l'interface Data Protection. Le miroir ZFS et les snapshots locaux ne constituent donc pas une sauvegarde hors site.

## Audit quotidien

`/opt/health-check.sh` s'exécute chaque jour à 06:00 et contrôle notamment :

- l'espace du pool thin, des systèmes de fichiers et des montages NFS ;
- l'expiration du certificat wildcard et un dry-run Certbot le dimanche ;
- le nombre de snapshots ;
- les mises à jour de sécurité ;
- SMART, les unités systemd en échec et les LXC arrêtés ;
- la fraîcheur des sauvegardes.

Les alertes passent par ntfy. Un heartbeat récapitulatif est envoyé le lundi. Un nettoyage quotidien est lancé à 03:00 pour les journaux, les fichiers temporaires et certains objets Docker.

Point technique à revoir : le nettoyage recherche des backups `*.gz`, alors que la configuration actuelle produit des archives `*.tar.zst`.
