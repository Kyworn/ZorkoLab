# Architecture de Stockage

Le stockage est centralisé sur un nœud **TrueNAS Scale** distinct, garantissant la sécurité des données grâce au système de fichiers ZFS.

## ZFS Pool : Tank

*   **Topologie :** RAID 1 (Miroir) de 2 disques HDD WD Red de 1 TB.
*   **Capacité Utilisable :** 932 GB
*   **Performance :** Compression LZ4 activée par défaut.

## Datasets ZFS et Cas d'Usage

| Dataset | Taille | Description | Protocole de Partage | Sécurité |
|:---|:---|:---|:---|:---|
| `Tank/server/backup` | 302 GB | Sauvegardes automatisées de Proxmox (VZDump). | **NFS** | MapRoot = `root`, Lecture/Écriture |
| `Tank/server/Film` | 183 GB | Médiathèque Films. | **SMB / NFS** | Partage SMB protégé (pas d'invité) |
| `Tank/server/Series` | 82 GB | Médiathèque Séries. | **SMB / NFS** | Partage SMB protégé |
| `Tank/server/download`| 49 GB | Dossier tampon pour qBittorrent. | **NFS** | - |
| `Tank/share` | 36 GB | Fichiers personnels. | **SMB** | Accès authentifié strict |

## Communication Proxmox <-> TrueNAS

L'hôte Proxmox monte le partage `backup` via NFS pour y déposer quotidiennement les archives de sauvegarde des conteneurs LXC. 
Les conteneurs de média (`media-hub`, `qbittorrent`) montent directement les partages NFS depuis le TrueNAS en utilisant les interfaces réseau internes, garantissant un flux de données sans interférence avec l'hyperviseur.
