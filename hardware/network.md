# Matériel réseau

## Accès Internet et LAN

| Élément | Configuration |
|:---|:---|
| Routeur | Freebox Delta v7 |
| Accès | fibre FTTH, jusqu'à 10 Gbit/s descendant et 900 Mbit/s montant d'après l'inventaire existant |
| Passerelle actuelle | `192.168.1.254` |
| Plan d'adressage | `192.168.1.0/24` |

Le débit et le modèle du routeur n'ont pas été remesurés pendant l'audit logiciel du 10 septembre 2026.

## Commutation et Wi-Fi

- switch principal HPE Gigabit, adresse d'administration documentée `192.168.1.77` ;
- trois points d'accès TP-Link Deco BE25 en mesh ;
- Wi-Fi de la Freebox documenté comme désactivé.

Ces éléments physiques sont conservés depuis l'inventaire précédent. Leur firmware et leur configuration n'ont pas été interrogés pendant cet audit.

## Interfaces vérifiées

- Proxmox : lien principal `eno1` rattaché à `vmbr0` ;
- TrueNAS : `enp2s0` actif à 1 Gbit/s sur `192.168.1.109/24`, `enp3s0` inactif ;
- aucune interface VLAN de production observée sur Proxmox.

AdGuard Home tourne désormais dans le LXC 119 à l'adresse `192.168.1.12`. L'ancienne VM Freebox à `192.168.1.189` n'est plus la topologie documentée.
