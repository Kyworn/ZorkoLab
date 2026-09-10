# Nœud de stockage

## TrueNAS

| Élément | Configuration vérifiée |
|:---|:---|
| CPU | Intel N100 |
| RAM | 15,4 GiB visibles |
| Système | TrueNAS Community Edition 26.0.0-BETA.3 |
| Réseau | `192.168.1.109/24`, lien 1 Gbit/s |
| Châssis | mini PC AOOSTAR, d'après l'inventaire physique existant |
| Disque de boot | NVMe 447,13 GiB |

## Pool `Tank`

| Élément | Valeur au 10 septembre 2026 |
|:---|:---|
| VDEV | 1× miroir, 2 disques |
| Disques | 2× 931,51 GiB Western Digital |
| Capacité utilisable | 920 GiB |
| Utilisation | 321,1 GiB, soit 34,9 % |
| Disponible | 598,9 GiB |
| Santé | ONLINE, 0 erreur |
| Températures | 38 à 49 °C, moyenne affichée 44,7 °C |

Le dernier scrub visible s'est terminé le 23 août 2026 sans erreur. Un scrub est planifié le dimanche à 13:00.

## Services de fichiers

- SMB actif avec 6 partages ;
- NFS actif avec 8 exports limités au LAN ;
- iSCSI arrêté, un target `pc` reste configuré ;
- NVMe-oF et WebShare arrêtés.

Une notification TrueNAS indique que de nouveaux feature flags ZFS sont disponibles. Aucune mise à niveau du pool n'a été lancée pendant l'audit, car cette opération est à sens unique et doit être décidée séparément.
