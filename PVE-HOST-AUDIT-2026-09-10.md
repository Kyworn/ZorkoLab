# Audit du nœud Proxmox, 10 septembre 2026

## Verdict

Le nœud est sain côté matériel et les services Proxmox essentiels fonctionnent. Les automatisations concurrentes, comptes orphelins, services fantômes et scripts sans appel trouvés pendant l'audit ont été retirés. Le principal chantier restant est le durcissement des règles réseau, à traiter séparément pour ne pas couper l'administration.

Le risque immédiat n'est pas une panne matérielle. La charge d'autostart est maintenant compatible avec les 32 GiB installés, mais l'ordre de démarrage reste à formaliser avant de considérer un redémarrage complet comme totalement prévisible.

## État vérifié

| Élément | État au 10 septembre 2026 |
|:---|:---|
| Plateforme | Proxmox VE 9.2.11, nœud autonome sans Corosync ni HA |
| Processeur | Ryzen 5 5600X, 6 cœurs et 12 threads |
| Mémoire | 32 GiB physiques, 23 GiB configurés sur les LXC en démarrage automatique après redimensionnement |
| Stockage local | NVMe BIWIN 512 Go, SMART OK, 4 % d'usure |
| LVM-thin | 270,8 GiB, 49,12 % de données et 29,46 % de métadonnées |
| Températures | CPU 48,5 °C, NVMe 50 °C, GPU 35 et 36 °C |
| Invités | 18 LXC actifs et un template VM arrêté |
| Réseau | bridge unique `vmbr0`, LAN `192.168.1.0/24` |
| Stockages distants | six exports NFS TrueNAS actifs |
| Services PVE | cluster filesystem, API, proxy, scheduler et firewall actifs |
| Unités en échec | aucune après nettoyage |
| Redémarrage | requis pour charger le noyau `7.0.14-15-pve` |
| Certificat PVE | valide jusqu'au 8 août 2027 pour `pve.zserv.local` et `192.168.1.61` |
| Abonnement | aucun abonnement Proxmox configuré |

La pression mémoire était nulle pendant le contrôle et aucun OOM n'a été observé sur le boot courant. Les 5,2 GiB de swap utilisés sont principalement compatibles avec des pages froides conservées après des périodes plus chargées.

## Priorité 1 : reprendre le contrôle des automatisations

### Snapshots Hermes encore actifs

Le cron local historique `/usr/local/bin/pve-daily-snapshot.sh` est bien commenté dans `/etc/crontab`. Il n'est plus la source des snapshots.

La tâche Hermes `pve-snapshot-daily`, hébergée dans le LXC 203, était activée à `05:00` chaque jour. Elle se connectait au nœud en SSH comme `root` et tentait de créer deux snapshots locaux pour tous les LXC compatibles. C'est elle qui a rempli le pool thin avant l'audit.

Action appliquée : la tâche Hermes est en pause. Les deux jobs VZDump validés sont maintenant l'unique mécanisme automatique de sauvegarde des LXC. Les deux snapshots manuels pré-upgrade de 102 et 114 restent indépendants et peuvent être conservés jusqu'à validation de leurs mises à niveau.

### Démarrage automatique et mémoire

Les LXC configurés avec `onboot=1` totalisaient 42 GiB de limites mémoire pour 32 GiB physiques. Après comparaison avec les pics enregistrés sur un mois, neuf limites ont été réduites avec une marge importante. L'autostart totalise maintenant 23 GiB et tous les LXC ainsi que leurs services principaux ont été validés actifs.

| LXC | Limite avant | Limite actuelle | Pic observé sur 30 jours |
|---:|---:|---:|---:|
| 102 Homebridge | 5 GiB | 1,5 GiB | 0,63 GiB |
| 104 qBittorrent | 2 GiB | 1 GiB | 0,23 GiB |
| 112 Docker | 4 GiB | 3 GiB | 1,54 GiB |
| 114 Vaultwarden | 6 GiB | 1 GiB | 0,14 GiB |
| 120 Gitea | 2 GiB | 1 GiB | 0,25 GiB |
| 122 AzerothDB | 4 GiB | 3 GiB | 1,65 GiB |
| 125 Codeman | 2 GiB | 1,5 GiB | 0,87 GiB |
| 130 Media Hub | 4 GiB | 2 GiB | 1,08 GiB |
| 203 Jarvis2 | 8 GiB | 4 GiB | 2,39 GiB |

Les LXC 210 et 211 n'ont pas de démarrage automatique. Il faut conserver ce choix et ajouter des ordres et délais de démarrage aux services essentiels avant de réévaluer les limites mémoire.

Ordre conseillé : réseau et DNS, reverse proxy et tunnel, données et applications, média, puis IA.

## Priorité 2 : réduire les accès inutiles

### SSH

L'authentification par mot de passe est désactivée et `root` n'accepte que les clés, ce qui est correct. En revanche :

- `authorized_keys` contenait douze copies identiques de l'ancienne clé `jarvis@host202` ;
- la clé `jarvis@openclaw` permet à Hermes de se connecter directement en root ;
- X11 forwarding et TCP forwarding restent autorisés globalement ;
- plusieurs vieilles clés de machines sont encore présentes sans inventaire d'usage.

Action appliquée : les douze clés `jarvis@host202`, le doublon Hermes et les anciennes clés sans usage constaté ont été retirés. Trois clés restent autorisées : l'accès courant `zorko-proxmox`, la clé de secours `zmac` et `jarvis@openclaw` pour les contrôles Hermes. Les accès courant et Hermes ont été retestés après le nettoyage. Il faudra remplacer à terme la clé root Hermes par une méthode limitée.

### Comptes API

Trois identités techniques subsistaient :

- `Prometheus@pve` possède `PVEAdmin` sur tout le cluster, ce qui est beaucoup trop large ;
- `exporter@pve` possède `PVEAuditor`, mais son ancien service n'existe plus ;
- le token `root@pam!OC_Jarvis` possède `PVEAuditor`, sans expiration.

Aucune utilisation de ces identités n'a été trouvée dans les logs disponibles. Un ancien fichier `/opt/pve-exporter.cfg`, lisible par tous les utilisateurs locaux, contenait encore un mot de passe en clair. Le service associé était absent.

Action appliquée : les deux utilisateurs, le token et leurs ACL ont été révoqués. Les fichiers `pve-exporter` orphelins ont été supprimés. Le secret exposé n'est plus accepté par Proxmox.

### Comptes Linux et ancien dépôt interne

Les comptes locaux `zorko`, `crowdsec` et `git` n'avaient aucun rôle de connexion encore nécessaire. Ils ont été supprimés avec leurs petits répertoires personnels. Seul `root` conserve un shell interactif sur l'hôte.

L'ancien compte `git` desservait le dépôt bare `/srv/git/infra-bus.git`, sans activité SSH récente et sans commit depuis le 9 août. Le dépôt a été exporté dans `Backup/configs/infra-bus-final-2026-09-10.bundle`, validé avec `git bundle verify`, puis supprimé du nœud.

L'ancien service Prometheus local, node_exporter, leurs binaires et leur configuration ont été retirés. Leur petite configuration a été archivée dans `Backup/configs/legacy-host-cleanup-2026-09-10.tar.zst`. Beszel reste l'unique agent de métriques de l'hôte. CrowdSec et son bouncer restent actifs malgré le retrait du compte local inutilisé.

### Pare-feu

Le pare-feu Proxmox est actif, mais les règles du nœud autorisent SSH, HTTP et HTTPS depuis n'importe quelle source. Elles rendent sans effet pratique les restrictions LAN similaires placées au niveau datacenter. Deux règles pour l'ancien réseau `10.10.0.0/16` sont également obsolètes.

Le port 9100 n'écoute plus. Le port 111 de `rpcbind` est limité au LAN. `rpc.statd` utilise toutefois aussi des ports dynamiques écoutant sur toutes les interfaces. Tous les LXC ne portent pas le marqueur `firewall=1`, et aucune segmentation VLAN n'isole les workloads.

Les deux moteurs pare-feu Proxmox étaient démarrés en parallèle. La configuration ne sélectionnait pas le backend nftables expérimental et seules les chaînes iptables `PVEFW-*` étaient effectives. `proxmox-firewall.service` a donc été désactivé ; `pve-firewall.service` reste actif et ses chaînes ont été vérifiées après l'opération.

Seuls les LXC 112, 125, 126, 128 et 129 ont à la fois un fichier de règles et le firewall activé sur leur interface. Le LXC 210 possède un fichier de règles, mais pas le marqueur réseau nécessaire. Il ne faut donc pas présenter le firewall Proxmox comme une isolation généralisée entre les conteneurs.

Action recommandée : limiter l'administration du nœud au LAN et à Tailscale, retirer les règles VLAN historiques et fixer ou filtrer les ports NFS auxiliaires. Les règles des LXC doivent être traitées dans un chantier séparé pour éviter de couper des services publics.

## Priorité 3 : nettoyer l'hôte sans casser Proxmox

### Éléments retirés après validation

- Cockpit et ses sept paquets ont été purgés.
- Ollama, son service, son utilisateur et son binaire ont été retirés. L'inférence utile dans le LXC 211 a été validée après l'opération.
- dnsmasq a été purgé. La résolution DNS de l'hôte fonctionne toujours et le port 53 n'écoute plus sur Proxmox.
- les deux paquets `.deb` de Proxmox Backup Server et le dépôt `pbs-no-subscription` ont été retirés.
- les anciens scripts locaux de snapshots, backup et exporter sans appel actif ont été supprimés.
- le service de failover `clawdbot` qui écrivait chaque minute pour l'ancien LXC 124 a été retiré avec ses scripts et 16 Mo de journal ;
- Wazuh, inactif mais encore installé, et le doublon Cloudflared de l'hôte ont été purgés ; Cloudflared reste actif dans le LXC 129 ;
- les helpers de stockage inutilisés, les unités NVIDIA auxiliaires incohérentes et le backend firewall alternatif ont été désactivés ;
- les outils de développement root, anciennes migrations, builds LLM et scripts sans appel ont été archivés si nécessaire puis retirés.

Les ports 53, 9090, 9100 et 11434 ne sont plus ouverts sur le nœud. Aucun `apt autoremove` n'a été lancé : sa proposition actuelle inclut des composants NVIDIA et pourrait casser l'accès GPU.

Le nettoyage quotidien limitait le journal systemd à trois jours. La rétention est maintenant de 14 jours, soit deux cycles de sauvegarde hebdomadaire.

### Ne pas supprimer Ceph à l'aveugle

Le nœud n'utilise aucun cluster ni stockage Ceph, mais les paquets Ceph sont liés aux bibliothèques de stockage Proxmox dans l'état actuel des dépendances. La simulation de purge retirerait notamment `proxmox-ve`, `pve-manager`, `pve-container` et `qemu-server`. Aucun nettoyage Ceph ne doit être lancé avec une commande de purge globale.

### Dépôts et mises à jour

Les occurrences `contrib` en double dans les sources Debian et le dépôt PBS inutile ont été retirés. `apt update` fonctionne après le nettoyage. Le dépôt CrowdSec vise encore Debian Bookworm alors que l'hôte est sous Trixie. Le dépôt NVIDIA CUDA propose une montée majeure vers la série 615.

Sept correctifs de sécurité sont en attente après actualisation des dépôts, mais une mise à niveau globale mélangerait ces correctifs avec Proxmox, Ceph, CrowdSec et NVIDIA. Il faut séparer la maintenance de sécurité du chantier GPU.

## GPU

Les deux Quadro P5000 fonctionnent réellement et sont visibles dans le LXC 211 avec le pilote 580.126.18. Les unités auxiliaires `nvidia-persistenced`, suspend, resume et hibernate provenaient de la pile 550 et ne sont pas nécessaires au passthrough actuel ; elles ont été désactivées. L'installateur 580.126.18 est conservé sur l'hôte pour la prochaine maintenance.

Une mise à niveau NVIDIA complète vers 615 ne doit pas être faite sans fenêtre de maintenance. Le chemin sûr consiste à choisir une seule source de paquets, aligner modules, bibliothèques et outils sur une même version, redémarrer, puis valider `nvidia-smi` sur l'hôte et dans le LXC 211.

## Sauvegardes et reprise

Les deux jobs VZDump simplifiés ont été testés avec succès. Le stockage `Backup` contient encore une archive pour chacun des anciens IDs 127, 140 et 202, soit environ 14,6 GiB. Elles sont conservées temporairement et peuvent être supprimées après décision explicite.

La configuration Proxmox est archivée chaque nuit à 02:00 sur TrueNAS avec 30 versions. Cette archive exclut volontairement les clés, certificats et le répertoire privé de `/etc/pve`. Elle permet de reconstruire la topologie et les invités, mais pas de restaurer à elle seule tous les secrets et accès du nœud.

Les six montages NFS sont actifs et aucune erreur NFS récente n'a été trouvée. Cinq utilisent NFS 4.2. L'export `gitea` utilise encore NFSv3, ce qui explique la dépendance à `rpcbind`, `rpc.statd` et à leurs ports auxiliaires. Migrer cet export vers NFSv4 permettrait de simplifier la surface réseau du nœud.

Le fichier `/etc/resolv.conf` est entièrement géré par Tailscale et utilise MagicDNS. C'est fonctionnel, mais la résolution DNS de l'hôte dépend donc directement de `tailscaled`. Cette dépendance doit être connue avant toute intervention sur Tailscale.

Le nettoyage quotidien à 03:00 purge Docker dans les LXC 112 et 130, le journal au-delà de 14 jours et les fichiers temporaires. Sa recherche `*.gz` concerne surtout les archives de configuration ; la rétention VZDump est déjà gérée par les jobs Proxmox.

## Scripts conservés sur l'hôte

| Chemin | Déclenchement | Rôle |
|:---|:---|:---|
| `/root/backup-pve-config.sh` | cron, tous les jours à 02:00 | archive la configuration PVE sur TrueNAS, 30 versions |
| `/opt/daily-cleanup.sh` | cron, tous les jours à 03:00 | rétention des journaux, prune Docker ciblé et nettoyage temporaire |
| `/opt/health-check.sh` | cron, tous les jours à 06:00 | contrôles stockage, SMART, certificats, LXC et backups, alertes ntfy |
| `/usr/local/bin/lxc-run` | manuel | exécute proprement une commande dans un LXC |
| `/usr/local/bin/lxc-docker` | manuel | helper Docker dans un LXC |
| `/usr/local/bin/lxc-qbit` | manuel | helper ciblé sur qBittorrent dans le LXC 104 |
| `/root/admin/ios-node/` | manuel | scripts de reconstruction et de diagnostic du LXC 110 |

Le binaire `/usr/local/bin/beszel-agent` appartient au service de supervision, ce n'est pas un script d'administration. Aucun autre script personnalisé exécutable n'est présent dans `/opt`, `/usr/local/bin`, `/usr/local/sbin` ou `/root/admin`.

Les éléments uniques retirés sont récupérables dans `Backup/configs/pve-legacy-cleanup-2026-09-10.tar.zst` et `Backup/configs/pve-root-tools-cleanup-2026-09-10.tar.zst`. Les builds reproductibles et caches n'ont pas été archivés.

Aucun test réel de restauration n'a encore été effectué. C'est le dernier contrôle indispensable avant de considérer la chaîne comme validée.

## Contrôles secondaires dans les LXC

La validation après redimensionnement a confirmé les 18 LXC actifs et leurs services principaux. Elle a aussi retrouvé des échecs systemd plus anciens, sans rapport avec la réduction de mémoire :

- les LXC 102 et 112 ne peuvent pas monter `sys-kernel-config`, comportement courant avec leur confinement LXC actuel ;
- les LXC 125, 126 et 129 échouent à créer le namespace de montage demandé par `logrotate`, `man-db`, `systemd-logind` et `systemd-networkd` ;
- les applications Codeman, ntfy et cloudflared restent actives, mais l'échec de `logrotate` mérite une correction séparée pour éviter une croissance silencieuse des journaux.

Ces unités n'ont pas été modifiées pendant le nettoyage du nœud.

## Points satisfaisants

- le NVMe est sain, peu usé et le TRIM hebdomadaire est déjà actif ;
- le pool thin est revenu de 82,55 % à 49,12 % ;
- les six montages NFS sont actifs et aucune erreur NFS récente n'a été relevée ;
- les températures et ventilateurs sont normaux ;
- chrony est synchronisé ;
- tous les LXC sont non privilégiés ;
- CrowdSec et Fail2Ban sont actifs ;
- aucun OOM ni défaut matériel n'a été trouvé sur le boot courant ;
- les services Proxmox essentiels sont actifs.

## Ordre de traitement conseillé

1. Terminé : désactiver `pve-snapshot-daily` dans Hermes.
2. Terminé : révoquer les accès techniques orphelins et nettoyer les clés SSH décommissionnées.
3. Différé : corriger les règles firewall d'administration après mise en place du second facteur.
4. Partiel : limites mémoire réduites ; ordre et délais de démarrage encore à définir.
5. Terminé : retirer Cockpit, Ollama, dnsmasq, Wazuh, les doublons de supervision et les vestiges PBS.
6. Partiel : sources Debian et PBS nettoyées ; dépôt CrowdSec et correctifs de sécurité encore à traiter hors pile NVIDIA.
7. Différé : planifier l'alignement NVIDIA et le redémarrage sur le nouveau noyau.
8. Accepté sans test : la chaîne VZDump reste validée par ses créations d'archives.
9. À décider : sort des archives 127, 140 et 202.

## Limites

L'audit initial était une inspection en lecture seule, suivie des actions explicitement consignées dans ce document. Il ne comprend ni test d'intrusion, ni restauration destructive, ni redémarrage du nœud. L'exposition réelle depuis Internet dépend également du routeur et de Cloudflare, qui ne sont pas couverts par cette note dédiée au nœud Proxmox.
