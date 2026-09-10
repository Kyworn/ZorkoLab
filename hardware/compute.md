# Nœud de calcul

## Matériel

| Élément | Configuration vérifiée |
|:---|:---|
| CPU | AMD Ryzen 5 5600X, 6 cœurs / 12 threads |
| RAM | 32 GiB DDR4 |
| GPU | 2× NVIDIA Quadro P5000, 16 GiB chacune |
| Stockage local | NVMe avec LVM-thin pour les disques LXC/VM |
| Réseau | `eno1` vers `vmbr0`, adresse `192.168.1.61/24` |
| Châssis | Corsair 680X, d'après l'inventaire physique existant |

## Logiciel

| Élément | Version au 10 septembre 2026 |
|:---|:---|
| Proxmox VE Manager | 9.2.11 |
| Méta-paquet Proxmox | 9.2.0 |
| Base | Debian 13 Trixie |
| Noyau actif | `7.0.14-14-pve` |
| Pilote NVIDIA | 580.126.18 |

Le noyau `7.0.14-15-pve` était installé mais pas encore actif au moment de l'audit.

## Capacité et état

- racine Proxmox : 194 GiB, 25 GiB utilisés ;
- pool `nvme-biwin-storage` : 270,8 GiB, 70,9 % utilisés après décommissionnement des LXC 140 et 202 ;
- NVMe : SMART réussi, 49 °C, 4 % d'usure, aucune erreur média ;
- mémoire observée : environ 16 GiB utilisés et 14 GiB disponibles ;
- uptime observé : environ deux semaines.

Les deux P5000 sont affectées au LXC 211 `nex-llm`. Elles sont visibles depuis le conteneur et alimentent le service llama.cpp.
