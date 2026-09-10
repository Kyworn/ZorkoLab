# Sauvegardes et contrôles automatiques

## Responsabilités

La chaîne cible a été volontairement réduite à deux responsabilités :

- Proxmox crée les archives complètes des LXC sélectionnés avec VZDump ;
- TrueNAS stocke ces archives et protège séparément certains datasets avec ZFS.

Hermes, Beszel et les autres applications ne doivent pas être des orchestrateurs de sauvegarde du lab. L'audit approfondi a toutefois retrouvé une tâche Hermes historique encore active, décrite ci-dessous.

## Jobs VZDump

| Politique | Calendrier | Rétention | LXC |
|:---|:---|:---|:---|
| `backup-daily` | tous les jours à 04:00 | 3 versions | 102, 112, 114, 118, 119, 120, 122, 125, 126, 129 |
| `backup-weekly` | dimanche à 05:00 | 2 versions | 130, 203, 210 |
| `backup-none` | aucun job | aucune | 104, 110, 121, 128, 211 |

Les deux jobs utilisent le mode snapshot, la compression Zstandard et le stockage NFS `Backup` sur TrueNAS.

Les premiers passages manuels des deux groupes se sont terminés avec succès le 10 septembre 2026. La notification finale via `pve-ntfy`, initialement refusée avec un code 401, a été réparée puis validée par un test et par le job hebdomadaire.

### Automatisation historique à retirer

La tâche Hermes `pve-snapshot-daily` reste active chaque jour à 05:00 dans le LXC 203. Elle ouvre des sessions SSH root vers Proxmox et tente de conserver deux snapshots LVM-thin locaux pour les LXC compatibles. Elle a contribué au remplissage du pool thin et fait double emploi avec VZDump.

Le cron équivalent présent directement sur l'hôte est déjà désactivé. La tâche Hermes doit encore être désactivée avant sa prochaine exécution afin que Proxmox redevienne l'unique orchestrateur des sauvegardes.

### Choix assumés

- qBittorrent, ios-node, Portfolio et Beszel sont reconstruisibles ;
- `nex-llm` contient principalement un build llama.cpp et 22 GiB de modèles récupérables ;
- media-hub est hebdomadaire car ses configurations sont utiles mais volumineuses ;
- Jarvis2 est hebdomadaire en attendant une éventuelle sauvegarde ciblée de ses données Hermes ;
- les montages NFS de Gitea et media-hub sont exclus des archives VZDump.

## Backups internes aux applications

Deux applications créent aussi des copies cohérentes de leurs données dans leur propre rootfs :

- Vaultwarden copie sa base SQLite chaque jour à 03:00 dans `/opt/vaultwarden/backups` ;
- Veille Sociale exporte sa base et son état chaque jour à 06:30 dans `/opt/veille-backups`.

Ces copies sont ensuite incluses dans le VZDump du LXC. Elles facilitent une restauration applicative, mais ne constituent pas à elles seules une seconde sauvegarde indépendante.

## Configuration Proxmox

`/root/backup-pve-config.sh` s'exécute chaque jour à 02:00 et écrit les archives dans `Backup/configs`, avec 30 versions conservées.

## Snapshots TrueNAS

| Dataset | Fréquence | Rétention | État au 10 septembre |
|:---|:---|:---|:---|
| `Tank/server/backup` | quotidien à 12:00 | 6 jours | désactivé, dernier run 8 jours auparavant |
| `Tank/server/project/gitea` | quotidien à 12:00 | 6 jours | actif |
| `Tank/share` | quotidien à 12:00 | 6 jours | actif |

Aucune tâche de réplication, rsync ou cloud sync n'était visible dans TrueNAS. Le miroir ZFS et les snapshots locaux ne constituent donc pas une sauvegarde hors site.

## Audit quotidien

`/opt/health-check.sh` s'exécute chaque jour à 06:00 et contrôle notamment :

- le pool thin, les systèmes de fichiers et les montages NFS ;
- le certificat wildcard et le dry-run Certbot du dimanche ;
- les snapshots, mises à jour de sécurité, données SMART et unités systemd ;
- les LXC arrêtés ;
- les backups `backup-daily` avec un seuil de 48 heures ;
- les backups `backup-weekly` avec un seuil de 8 jours.

Les LXC `backup-none` sont volontairement ignorés par ce contrôle. Les alertes passent par ntfy et un heartbeat récapitulatif est envoyé le lundi.

Un nettoyage quotidien est lancé à 03:00. Point restant à revoir : il recherche des backups `*.gz`, alors que VZDump produit des archives `*.tar.zst`.
