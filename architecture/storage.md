# Architecture du stockage

## Vue logique

TrueNAS expose le pool ZFS `Tank` à Proxmox et aux postes du LAN. Le pool est un miroir de deux disques de 931,51 GiB, soit 920 GiB utilisables dans l'interface TrueNAS.

| Mesure au 10 septembre 2026 | Valeur |
|:---|:---|
| État du pool | ONLINE, aucune erreur |
| Organisation | 1× MIRROR, 2 disques |
| Capacité utilisable | 920 GiB |
| Espace utilisé | 321,1 GiB, 34,9 % |
| Espace disponible | 598,9 GiB |
| Dernier scrub affiché | 23 août 2026, 0 erreur |
| Planification du scrub | dimanche à 13:00 |

## Datasets

| Dataset | Utilisation observée | Usage principal |
|:---|---:|:---|
| `Tank/server/backup` | 40,93 GiB | sauvegardes VZDump |
| `Tank/server/download` | 11,03 GiB | téléchargements |
| `Tank/server/Film` | 121,86 GiB | films |
| `Tank/server/Livres` | 936 KiB | livres |
| `Tank/server/project` | 2,23 GiB | projets, dont Gitea |
| `Tank/server/Series` | 88,79 GiB | séries |
| `Tank/share` | 53,96 GiB | fichiers partagés |

Les chiffres de datasets et ceux du tableau de bord peuvent différer légèrement en raison de la comptabilisation ZFS, des snapshots et des unités affichées.

## Partages

- SMB : 6 partages actifs, `Film`, `Series`, `backups`, `download`, `project` et `share`.
- NFS : 8 exports actifs, `Film`, `Livres`, `Series`, `backup`, `download`, `project`, `project/gitea` et `share`.
- iSCSI : service arrêté, avec un target `pc` configuré.

Les exports NFS sont limités au réseau `192.168.1.0/24`. Proxmox monte notamment `Backup`, `Film`, `Series`, `Download`, `Livres` et `gitea`.

La stratégie de snapshots et de sauvegardes est détaillée dans [automation/backups.md](../automation/backups.md).
