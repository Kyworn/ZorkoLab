# Stratégie de Sauvegardes (Backups)

La protection des données est structurée en plusieurs niveaux de rétention et de redondance.

## 1. Snapshots ZFS (Chaud - Instantané)

Le nœud TrueNAS exécute des snapshots ZFS réguliers sur le pool `Tank`.
- **Fréquence :** Quotidienne.
- **Rétention :** 14 jours.
- **Avantage :** Protection instantanée contre les suppressions accidentelles et les ransomwares. Coût de stockage presque nul pour les données statiques.

## 2. Sauvegardes Proxmox (Froid - VZDump)

Proxmox sauvegarde l'état complet des conteneurs LXC et des VMs.
- **Destination :** Partage NFS `/mnt/pve/Backup` (fourni par TrueNAS).
- **Fréquence :** Hebdomadaire (Dimanche à 01:00 AM).
- **Rétention :** 2 dernières sauvegardes conservées.
- **Format :** LZO (Compressé).

## 3. Sécurisation du Montage

Afin de prévenir tout risque (par exemple un malware dans un conteneur qui chiffrerait les backups), le dossier `/mnt/pve/Backup` sur le TrueNAS a fait l'objet d'un durcissement :
- **Permissions :** `700` (Propriétaire `root` uniquement).
- **MapRoot NFS :** Verrouillé.

*Note de migration à venir : Le passage vers **Proxmox Backup Server (PBS)** est planifié pour remplacer VZDump et bénéficier de la déduplication au bloc.*
