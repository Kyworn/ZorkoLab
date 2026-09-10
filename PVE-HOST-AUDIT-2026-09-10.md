# Audit du nœud Proxmox, 10 septembre 2026

## Verdict

Le nœud est sain côté matériel et les services Proxmox essentiels fonctionnent. Il n'est toutefois pas encore simple à exploiter sereinement : plusieurs anciennes automatisations se chevauchent, des accès techniques ne sont plus maîtrisés et la configuration autorise davantage que nécessaire.

Le risque immédiat n'est pas une panne matérielle. Il vient surtout d'une action automatique oubliée ou d'un redémarrage qui lance trop de conteneurs à la fois.

## État vérifié

| Élément | État au 10 septembre 2026 |
|:---|:---|
| Plateforme | Proxmox VE 9.2.11, nœud autonome sans Corosync ni HA |
| Processeur | Ryzen 5 5600X, 6 cœurs et 12 threads |
| Mémoire | 32 GiB physiques, 42 GiB configurés sur les LXC en démarrage automatique |
| Stockage local | NVMe BIWIN 512 Go, SMART OK, 4 % d'usure |
| LVM-thin | 270,8 GiB, 49,12 % de données et 29,46 % de métadonnées |
| Températures | CPU 48,5 °C, NVMe 50 °C, GPU 35 et 36 °C |
| Invités | 18 LXC actifs et un template VM arrêté |
| Réseau | bridge unique `vmbr0`, LAN `192.168.1.0/24` |
| Stockages distants | six exports NFS TrueNAS actifs |
| Services PVE | cluster filesystem, API, proxy, scheduler et firewall actifs |
| Unités en échec | une, `nvidia-persistenced.service` |
| Redémarrage | requis pour charger le noyau `7.0.14-15-pve` |
| Certificat PVE | valide jusqu'au 8 août 2027 pour `pve.zserv.local` et `192.168.1.61` |
| Abonnement | aucun abonnement Proxmox configuré |

La pression mémoire était nulle pendant le contrôle et aucun OOM n'a été observé sur le boot courant. Les 5,2 GiB de swap utilisés sont principalement compatibles avec des pages froides conservées après des périodes plus chargées. Le surengagement reste néanmoins trop élevé pour considérer un redémarrage complet comme prévisible.

## Priorité 1 : reprendre le contrôle des automatisations

### Snapshots Hermes encore actifs

Le cron local historique `/usr/local/bin/pve-daily-snapshot.sh` est bien commenté dans `/etc/crontab`. Il n'est plus la source des snapshots.

La tâche Hermes `pve-snapshot-daily`, hébergée dans le LXC 203, reste activée à `05:00` chaque jour. Elle se connecte au nœud en SSH comme `root` et tente de créer deux snapshots locaux pour tous les LXC compatibles. C'est elle qui a rempli le pool thin avant l'audit et qui recréera des snapshots dès sa prochaine exécution.

Action recommandée : désactiver cette tâche Hermes. Les deux jobs VZDump validés deviennent l'unique mécanisme de sauvegarde des LXC. Les deux snapshots manuels pré-upgrade de 102 et 114 restent indépendants et peuvent être conservés jusqu'à validation de leurs mises à niveau.

### Démarrage automatique et mémoire

Les LXC configurés avec `onboot=1` totalisent 42 GiB de limites mémoire pour 32 GiB physiques. Le total de tous les LXC atteint 59 GiB. Ce n'est pas une consommation permanente, mais un redémarrage du nœud peut lancer trop de services simultanément et provoquer du swap ou des OOM.

Les LXC 210 et 211 n'ont pas de démarrage automatique. Il faut conserver ce choix et ajouter des ordres et délais de démarrage aux services essentiels avant de réévaluer les limites mémoire.

Ordre conseillé : réseau et DNS, reverse proxy et tunnel, données et applications, média, puis IA.

## Priorité 2 : réduire les accès inutiles

### SSH

L'authentification par mot de passe est désactivée et `root` n'accepte que les clés, ce qui est correct. En revanche :

- `authorized_keys` contient douze copies identiques de l'ancienne clé `jarvis@host202` ;
- la clé `jarvis@openclaw` permet à Hermes de se connecter directement en root ;
- X11 forwarding et TCP forwarding restent autorisés globalement ;
- plusieurs vieilles clés de machines sont encore présentes sans inventaire d'usage.

Action recommandée : retirer les doublons et les clés décommissionnées, puis remplacer l'accès root d'Hermes par une méthode limitée si une automatisation Proxmox doit être conservée.

### Comptes API

Trois identités techniques subsistent :

- `Prometheus@pve` possède `PVEAdmin` sur tout le cluster, ce qui est beaucoup trop large ;
- `exporter@pve` possède `PVEAuditor`, mais son ancien service n'existe plus ;
- le token `root@pam!OC_Jarvis` possède `PVEAuditor`, sans expiration.

Aucune utilisation de ces identités n'a été trouvée dans les logs disponibles. Un ancien fichier `/opt/pve-exporter.cfg`, lisible par tous les utilisateurs locaux, contient encore un mot de passe en clair. Le service associé est absent et `node_exporter` assure aujourd'hui les métriques sur le port 9100.

Action recommandée : révoquer les identités confirmées inutiles, faire tourner le secret exposé, puis supprimer les fichiers `pve-exporter` orphelins. Ne pas se contenter d'effacer le fichier contenant le secret.

### Pare-feu

Le pare-feu Proxmox est actif, mais les règles du nœud autorisent SSH, HTTP et HTTPS depuis n'importe quelle source. Elles rendent sans effet pratique les restrictions LAN similaires placées au niveau datacenter. Deux règles pour l'ancien réseau `10.10.0.0/16` sont également obsolètes.

Le port 9100 de `node_exporter` et le port 111 de `rpcbind` sont limités au LAN. `rpc.statd` utilise toutefois aussi des ports dynamiques écoutant sur toutes les interfaces. Tous les LXC ne portent pas le marqueur `firewall=1`, et aucune segmentation VLAN n'isole les workloads.

Seuls les LXC 112, 125, 126, 128 et 129 ont à la fois un fichier de règles et le firewall activé sur leur interface. Le LXC 210 possède un fichier de règles, mais pas le marqueur réseau nécessaire. Il ne faut donc pas présenter le firewall Proxmox comme une isolation généralisée entre les conteneurs.

Action recommandée : limiter l'administration du nœud au LAN et à Tailscale, retirer les règles VLAN historiques et fixer ou filtrer les ports NFS auxiliaires. Les règles des LXC doivent être traitées dans un chantier séparé pour éviter de couper des services publics.

## Priorité 3 : nettoyer l'hôte sans casser Proxmox

### Candidats sûrs après validation

- Cockpit est installé mais son socket est désactivé et aucun port 9090 n'écoute.
- Ollama est lancé au boot sur `127.0.0.1:11434`, sans modèle installé ni requête trouvée. L'inférence utile tourne dans le LXC 211.
- dnsmasq écoute sur toutes les interfaces au port 53 sans configuration spécifique ni activité journalisée. Le DNS du LAN est assuré par AdGuard et le résolveur de l'hôte par Tailscale.
- deux paquets `.deb` de Proxmox Backup Server, environ 65 Mo au total, restent dans `/root` alors que PBS Server n'est pas installé.
- le dépôt `pbs-no-subscription` est configuré sans serveur PBS local.
- plusieurs scripts historiques sans appel actif restent sous `/opt`, `/root` et `/usr/local/bin`.

Ces éléments peuvent être désactivés puis observés avant suppression. dnsmasq doit être testé depuis le LAN avant retrait, même s'il semble redondant.

Le nettoyage quotidien limite le journal systemd à trois jours. Cette rétention est trop courte pour analyser un incident découvert après un week-end ou comparer deux cycles de sauvegarde hebdomadaire. Une durée de 14 à 30 jours, ou une limite par taille, serait plus exploitable pour un coût disque raisonnable.

### Ne pas supprimer Ceph à l'aveugle

Le nœud n'utilise aucun cluster ni stockage Ceph, mais les paquets Ceph sont liés aux bibliothèques de stockage Proxmox dans l'état actuel des dépendances. La simulation de purge retirerait notamment `proxmox-ve`, `pve-manager`, `pve-container` et `qemu-server`. Aucun nettoyage Ceph ne doit être lancé avec une commande de purge globale.

### Dépôts et mises à jour

Les sources Debian contiennent `contrib` en double. Le dépôt CrowdSec vise Debian Bookworm alors que l'hôte est sous Trixie. Le dépôt NVIDIA CUDA propose une montée majeure vers la série 615.

Quatorze correctifs de sécurité sont en attente, mais une mise à niveau globale mélangerait ces correctifs avec Proxmox, Ceph, Cockpit, CrowdSec et NVIDIA. Il faut séparer la maintenance de sécurité du chantier GPU.

## GPU

Les deux Quadro P5000 fonctionnent réellement et sont visibles dans le LXC 211. Le pilote chargé est en version 580.126.18, tandis que le binaire `nvidia-persistenced` installé est en 550.163.01. Ce décalage explique très probablement l'échec du service.

Une mise à niveau NVIDIA complète vers 615 ne doit pas être faite sans fenêtre de maintenance. Le chemin sûr consiste à choisir une seule source de paquets, aligner modules, bibliothèques et outils sur une même version, redémarrer, puis valider `nvidia-smi` sur l'hôte et dans le LXC 211.

## Sauvegardes et reprise

Les deux jobs VZDump simplifiés ont été testés avec succès. Le stockage `Backup` contient encore une archive pour chacun des anciens IDs 127, 140 et 202, soit environ 14,6 GiB. Elles sont conservées temporairement et peuvent être supprimées après décision explicite.

La configuration Proxmox est archivée chaque nuit à 02:00 sur TrueNAS avec 30 versions. Cette archive exclut volontairement les clés, certificats et le répertoire privé de `/etc/pve`. Elle permet de reconstruire la topologie et les invités, mais pas de restaurer à elle seule tous les secrets et accès du nœud.

Les six montages NFS sont actifs et aucune erreur NFS récente n'a été trouvée. Cinq utilisent NFS 4.2. L'export `gitea` utilise encore NFSv3, ce qui explique la dépendance à `rpcbind`, `rpc.statd` et à leurs ports auxiliaires. Migrer cet export vers NFSv4 permettrait de simplifier la surface réseau du nœud.

Le fichier `/etc/resolv.conf` est entièrement géré par Tailscale et utilise MagicDNS. C'est fonctionnel, mais la résolution DNS de l'hôte dépend donc directement de `tailscaled`. Cette dépendance doit être connue avant toute intervention sur Tailscale.

Le nettoyage quotidien à 03:00 purge Docker dans les LXC 112 et 130, le journal au-delà de trois jours et les fichiers temporaires. Sa recherche `*.gz` concerne surtout les archives de configuration ; la rétention VZDump est déjà gérée par les jobs Proxmox.

Aucun test réel de restauration n'a encore été effectué. C'est le dernier contrôle indispensable avant de considérer la chaîne comme validée.

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

1. Désactiver `pve-snapshot-daily` dans Hermes avant la prochaine exécution de 05:00.
2. Révoquer les accès techniques orphelins et nettoyer les clés SSH décommissionnées.
3. Corriger les règles firewall d'administration sans toucher encore aux règles applicatives des LXC.
4. Définir l'ordre de démarrage et réduire les limites mémoire manifestement surdimensionnées.
5. Désactiver puis retirer Cockpit, Ollama et dnsmasq après observation.
6. Nettoyer les dépôts et appliquer les correctifs de sécurité hors pile NVIDIA.
7. Planifier séparément l'alignement NVIDIA et le redémarrage sur le nouveau noyau.
8. Effectuer un test de restauration d'un petit LXC depuis TrueNAS.
9. Décider du sort des archives 127, 140 et 202.

## Limites

L'audit est une inspection en lecture seule. Il ne comprend ni test d'intrusion, ni restauration destructive, ni redémarrage du nœud. L'exposition réelle depuis Internet dépend également du routeur et de Cloudflare, qui ne sont pas couverts par cette note dédiée au nœud Proxmox.
