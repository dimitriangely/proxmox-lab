# TPs Proxmox VE — M2 Infrastructure Cloud et Devops (RNCP39582)
**Dimitri ANGELY** · M2 Infrastructure Cloud · Juin 2026

---

## Cadre de réalisation

### Environnement technique

Ces travaux pratiques ont été réalisés dans un environnement de **virtualisation imbriquée** (nested virtualization) : Proxmox VE tourne à l'intérieur d'une VM VMware Workstation, elle-même hébergée sur un poste Windows.

| Composant | Détail |
|-----------|--------|
| Hôte physique | Windows 11 — Intel i7-13650HX (14 cores / 20 threads) — 64 Go RAM |
| Hyperviseur hôte | VMware Workstation |
| Hyperviseur cible | Proxmox VE 9.2 (Debian Trixie) |
| Ressources Proxmox | 16 vCPUs · 32 Go RAM · 200 Go disque (sda) |
| Réseau | VMware NAT — 192.168.131.0/24 — passerelle 192.168.131.2 |
| Accès | SSH depuis Windows Terminal + interface web Proxmox (port 8006) |

### Méthode de travail appliquée

Chaque action a été précédée d'une phase de compréhension : **pourquoi** on fait cette action, **quelles précautions** s'imposent, **quels risques** sont associés. Après chaque étape, une vérification immédiate confirme le résultat attendu. Les erreurs rencontrées ont été diagnostiquées par lecture des messages d'erreur et identification de la cause racine avant correction.

Cette approche — comprendre avant d'exécuter, tester après chaque étape, diagnostiquer méthodiquement — est appliquée de manière constante du TP1 au TP4.

### Contraintes spécifiques

- **Virtualisation imbriquée** : certaines fonctionnalités (KVM hardware acceleration) nécessitent une configuration VMware spécifique (VBS/HVCI désactivé, exposition des instructions de virtualisation). Les performances I/O sont limitées par la couche d'abstraction supplémentaire.
- **Noeud unique** : un seul serveur Proxmox physique disponible. Les architectures multi-noeuds (cluster réel, HA automatique) sont décrites en termes de ce qu'elles nécessiteraient en production.
- **Stockage contraint** : un seul disque physique de 200 Go entièrement utilisé par Proxmox. Les pools ZFS ont été créés sur des fichiers loopback (technique standard en environnement lab).

---

## TP1 — Installation et premiers pas avec Proxmox VE

### Objectif

Prendre en main Proxmox VE, comprendre son architecture de stockage et de réseau, créer un template Debian réutilisable, déployer des VMs clonées et valider le cycle snapshot/rollback.

### Réalisations

**Installation et configuration de Proxmox :**
- Installation de Proxmox VE 9.2 dans une VM VMware avec virtualisation imbriquée activée
- Configuration des dépôts APT sans abonnement (résolution d'un problème spécifique au format DEB822 de Debian Trixie — les fichiers `.sources` dans `/etc/apt/sources.list.d/` remplacent les `.list` traditionnels sur Proxmox 9.x)
- Vérification de l'interface réseau `vmbr0` (bridge sur `ens33`), de la passerelle et du DNS

**Création du template Debian 13 (VM 9000 — `tmpl-debian-13-base`) :**
- Installation d'une VM Debian 13.5.0 propre depuis l'ISO
- Installation et activation du QEMU Guest Agent (`qemu-guest-agent`) — permet à Proxmox de connaître l'IP de la VM sans scan réseau
- Configuration cloud-init pour provisionnement automatique (user, mot de passe, réseau DHCP)
- Nettoyage (machine-id, SSH host keys, cache apt) avant conversion en template
- Conversion en template Proxmox — la VM devient une base immuable depuis laquelle on clone

**Déploiement de 4 VMs clonées :**

| VM | ID | Type clone | Service | IP |
|----|-----|------------|---------|-----|
| web01 | 101 | Complet | Apache2 | 192.168.131.x |
| db01 | 102 | Lié | MariaDB | 192.168.131.x |
| admin01 | 103 | Lié | SSH / tests | 192.168.131.x |
| test-rollback01 | 104 | Lié | Apache2 | 192.168.131.x |

**Démonstration snapshot/rollback sur test-rollback01 :**
1. Installation d'Apache + page web de test
2. Création du snapshot `avant-panne`
3. Panne simulée : arrêt Apache + suppression de la page web
4. Constat de l'indisponibilité : `curl` → connexion refusée
5. Rollback vers le snapshot `avant-panne`
6. Vérification : service restauré, page web accessible

### Points techniques clés

- Différence entre **clone complet** (disque indépendant, plus lent à créer) et **clone lié** (partage le disque de base, économise l'espace — nécessite que le template reste intact)
- Le QEMU Guest Agent expose les IPs dans l'UI Proxmox et permet `qm guest exec` / `qm guest cmd` sans SSH
- Le snapshot Proxmox capture l'état disque et mémoire — le rollback est instantané contrairement à une restauration depuis backup

---

## TP2 — Architecture réseau segmentée avec VLANs

### Objectif

Construire une infrastructure réseau multi-zones avec isolation par VLANs, configurer un routeur/firewall Debian entre les zones, et valider le filtrage inter-VLAN.

### Architecture déployée

```
Internet (VMware NAT)
        |
      vmbr0 (192.168.131.0/24)
        |
      fw01 (VM 200) — routeur Debian avec 5 interfaces
     / | | | \
 VLAN10 VLAN20 VLAN30 VLAN40
  MGMT   SRV  USERS   DMZ
```

| VLAN | Zone | Réseau | Gateway sur fw01 | VM |
|------|------|--------|-------------------|----|
| 10 | MGMT | 192.168.10.0/24 | 192.168.10.254 | admin-mgmt01 (204) |
| 20 | SRV | 192.168.20.0/24 | 192.168.20.254 | db-srv01 (202) |
| 30 | USERS | 192.168.30.0/24 | 192.168.30.254 | client01 (203) |
| 40 | DMZ | 192.168.40.0/24 | 192.168.40.254 | web-dmz01 (201) |

### Réalisations

**Infrastructure réseau Proxmox :**
- Création du bridge `vmbr1` en mode VLAN-aware — permet d'attacher des VMs à des VLANs spécifiques via des tags
- Chaque VM dispose de deux interfaces : `eth0` sur `vmbr0` (accès SSH direct) et `ens19` sur `vmbr1` avec tag VLAN et IP statique dans sa zone

**Configuration de fw01 (VM 200 — routeur Debian) :**
- 5 interfaces réseau : `eth0` (DHCP sur vmbr0) + `ens19/20/21/22` (IP statiques par VLAN)
- Activation du routage IP : `net.ipv4.ip_forward = 1`
- NAT via nftables : masquerading sur `eth0` pour l'accès Internet
- Règles de filtrage inter-VLAN :
  - USERS (VLAN 30) → DMZ (VLAN 40) : **autorisé** (accès au web)
  - USERS (VLAN 30) → SRV (VLAN 20) : **bloqué** (isolation base de données)

**Validation des règles :**
- TTL = 63 sur les pings inter-VLAN (confirme que fw01 est bien le saut de routage, TTL décrémenté de 1)
- `client01` peut joindre `web-dmz01` (192.168.40.10) ✅
- `client01` ne peut pas joindre `db-srv01` (192.168.20.10) ✅

### Points techniques clés

- **VLAN-aware bridge** : Proxmox filtre les trames par tag VLAN — une VM sur VLAN 20 ne reçoit pas les trames VLAN 40, même si elles partagent le même bridge physique
- **TTL comme preuve de routage** : TTL = 64 = même sous-réseau, TTL = 63 = un saut de routage intermédiaire
- **nftables vs iptables** : Debian Trixie utilise nftables nativement. Les règles de filtrage sont déclarées dans des tables et chaînes explicites

---

## TP3 — Stockage Proxmox : iSCSI, NFS, LVM-thin, ZFS et résilience

### Objectif

Maîtriser les différentes technologies de stockage disponibles dans Proxmox, comprendre la distinction fondamentale entre stockage fichier et stockage bloc, et démontrer leur utilisation dans une infrastructure virtualisée avec sauvegarde et restauration.

### Architecture de stockage construite

| Stockage | Type | Usage | Technologie |
|----------|------|-------|-------------|
| local | Directory | ISO, backups, templates | /var/lib/vz |
| local-lvm | LVM-thin | Disques VM | pve/data (112.9 Go) |
| nfs-backup | NFS | Backups, ISO | VM 110 → /srv/nfs/proxmox |
| iscsi-store | iSCSI | LUN bloc brut | VM 110 → targetcli |
| zfs-tank | ZFS mirror | Disques VM, datasets | loop1 + loop2 (2×10 Go) |
| demo-simple | ZFS simple | Démonstration comparaison | loop0 (10 Go) |

### Réalisations

**Loopback ZFS (contrainte matérielle) :**
Aucun disque physique libre disponible. Création de 3 fichiers de 10 Go (`/var/lib/zfs-disks/disk0/1/2.img`) exposés via `losetup` comme périphériques blocs. Comportement identique à des disques physiques pour ZFS. Service systemd créé pour la persistance au reboot.

**VM srv-nfs-iscsi (VM 110 — 192.168.131.147) :**
Clone lié du template 9000. Héberge simultanément le serveur NFS (`nfs-kernel-server`) et la cible iSCSI (`targetcli-fb`). Correction SSH appliquée (`PermitRootLogin yes` + `PasswordAuthentication yes`) — le template Debian 13 désactive l'authentification par mot de passe pour root par défaut.

**NFS (Partie 2) :**
- Export `/srv/nfs/proxmox` accessible depuis `192.168.131.0/24`
- Stockage `nfs-backup` ajouté dans Proxmox avec `content backup,iso`
- Vérification : `showmount -e` + `pvesm status`

**iSCSI (Partie 3) :**
- LUN de 3 Go créé via `targetcli` sur la VM 110
- Cible : `iqn.2026-06.local.proxmox:storage` sur port 3260
- Connexion Proxmox initiateur → LUN visible en `raw/images` sans système de fichiers

**LVM-thin (Partie 4) :**
- Analyse du pool existant `pve/data` (112.9 Go, `Data%` = 23.26%)
- Démonstration de la différence taille provisionnée vs espace réellement consommé

**ZFS — Pool, Datasets, ZVOL (Parties 5-7) :**
- Pool `tank` (mirror loop1+loop2) et `demo-simple` (simple loop0) créés
- Datasets `tank/backups` et `tank/iso` montés automatiquement dans `/tank/`
- Compression `lz4` activée sur le pool → ratio `1.61x` mesuré
- Scrub lancé → `0 errors, 0B repaired`
- **Preuve visuelle dataset vs ZVOL** dans `zfs list` :
  - `tank/backups` → `MOUNTPOINT: /tank/backups` (système de fichiers)
  - `tank/vm-120-disk-0` → `MOUNTPOINT: -` (volume bloc)

**Sparse ZVOL (problème résolu) :**
ZFS refuse de créer un ZVOL de 20 Go sur un pool de 9.5 Go. Solution : `zfs create -V 20G -s` (flag `-s` = sparse, `refreservation=none`). Résultat : `volsize = 20G` annoncés, `referenced = 12K` réellement consommés — même principe que LVM-thin.

**VM zfs-test01 (VM 120 — 192.168.131.148) :**
Clone du template, disque déplacé de `local-lvm` vers `zfs-tank` via `qm move-disk` (démonstration Partie 12). Service nginx déployé, accessible depuis Proxmox.

**Backup et restauration (Parties 13-14) :**
- Backup de VM 120 : 8 Go de disque → 1.12 Go (`69%` de zéros détectés, compression zstd)
- Restauration VM 121 sur `local-lvm` avec `--unique 1`
- Service nginx fonctionnel sur la VM restaurée (192.168.131.149)

### Synthèse conceptuelle

> La distinction fondamentale est fichier vs bloc : NFS exporte un répertoire accessible en réseau comme un dossier monté, tandis qu'iSCSI expose un périphérique bloc (LUN/disque virtuel) que le client partitionne et formate selon ses besoins. ZFS unifie ces deux approches avec les datasets (stockage fichier, montés directement dans l'arborescence) et les ZVOLs (stockage bloc, visibles comme des périphériques de type disque avec un mountpoint à `-`). LVM-thin et ZFS sparse reposent sur la même logique de thin provisioning : présenter logiquement plus d'espace qu'on n'en consomme physiquement au départ, LVM-thin l'implémentant au niveau du Volume Group tandis que ZFS l'intègre nativement dans le pool. Enfin, ZFS se distingue par ses checksums de bout en bout associés à chaque bloc écrit ; le scrub les vérifie périodiquement et peut corriger automatiquement les corruptions silencieuses lorsqu'une redondance est disponible, ce que des systèmes de fichiers classiques comme ext4 ou XFS ne réalisent pas nativement.

---

## TP4 — Cluster Proxmox, haute disponibilité, quorum, migration et PRA

### Objectif

Construire un cluster Proxmox, comprendre Corosync et le quorum, mettre en place un stockage compatible avec la mobilité des VMs, déployer une VM critique, réaliser un PRA complet avec mesure du RTO.

### Contrainte principale

Un seul noeud Proxmox physique disponible. Le TP est réalisé en **mode PRA documenté** (Option E du sujet) : cluster mono-noeud démontrant Corosync et le quorum, PRA complet backup → panne contrôlée → restauration → RTO mesuré. Les conditions d'une HA complète (3 noeuds + stockage partagé + fencing) sont documentées et expliquées.

### Réalisations

**Cluster Proxmox (Parties 1-2) :**
- Création du cluster `cluster-m2-pve` (limite : 15 caractères max pour le nom Corosync)
- Corosync démarre automatiquement, génère la clé d'authentification dans `/etc/corosync/authkey`
- Transport `knet` (moderne, chiffré) activé par défaut sur Proxmox 9.x
- Vérification que les VMs du TP3 continuent de tourner après création du cluster

**Corosync et quorum (Partie 3) :**

```
pvecm status :
  Name:           cluster-m2-pve
  Transport:      knet
  Secure auth:    on
  Quorate:        Yes
  Expected votes: 1 / Total votes: 1 / Quorum: 1
  ring0_addr:     192.168.131.135:5405
```

Limite documentée : Corosync utilise l'interface management (`192.168.131.135`) faute d'interface dédiée. En production, un réseau cluster isolé (`10.10.10.x`) est nécessaire pour séparer le trafic Corosync du trafic management.

**Tableau quorum selon nombre de noeuds :**

| Noeuds | Quorum requis | Tolérance | Notre situation |
|--------|--------------|-----------|-----------------|
| 1 | 1/1 | Aucune | ← Ce TP |
| 2 | 2/2 | Aucune sans qdevice | — |
| 3 | 2/3 | 1 noeud | Cas idéal |

**Stockage partagé (Partie 4) :**
Réutilisation du stockage NFS `nfs-backup` du TP3. Dans un vrai cluster multi-noeuds, il suffirait d'exécuter `pvesm add nfs` sur chaque noeud pour que tous voient les mêmes fichiers. Limite identifiée : le serveur NFS tourne sur une VM hébergée par le même noeud Proxmox — en production, ce serait une appliance physique indépendante.

**VM app-prod01 (Parties 5-6) :**
- Clone complet de VM 120 (zfs-test01) → VM 130 (`app-prod01`) sur `local-lvm`
- IP : 192.168.131.150
- Service nginx : `curl http://192.168.131.150` → `app-prod01 operationnel`
- Warning LVM-thin noté : thin pool sur-provisionné à 116.54 Go annoncés pour 112 Go physiques

**PRA complet avec mesure RTO (Parties 7-9) :**

```
00:17:43 — Backup démarré
00:18:00 — Panne déclenchée (qm stop 130)
00:18:08 — Indisponibilité constatée (curl timeout)
00:18:08 — Reprise démarrée (qmrestore)
00:24:22 — Service restauré (curl http://192.168.131.151)

RTO mesuré : 6 minutes 22 secondes (conditions laboratoire)
RPO         : backup du 2026_06_10-00_17_44
```

- Backup : 8 Go → 1.13 Go (69% de zéros sparse + compression zstd)
- VM 131 (`app-prod01-restored`) restaurée sur `local-lvm` avec `--unique 1` (nouvelle MAC, nouvel UUID)
- Service nginx fonctionnel sur 192.168.131.151

**Analyse HA / PRA / Backup (Partie 10) :**

| Mécanisme | Automatique | RTO typique | Protège contre |
|-----------|-------------|-------------|----------------|
| HA Proxmox | Oui | 2-5 min | Panne noeud |
| PRA manuel | Non | 30 min – 2h | Panne grave |
| Backup | Planifié | Variable | Erreur humaine |

La HA ne protège pas contre : suppression de fichiers, corruption applicative, ransomware, erreur humaine. HA + Backup = couverture complémentaire nécessaire.

### RTO réaliste vs RTO de laboratoire

Le RTO de 6 minutes 22 secondes est un RTO de démonstration en conditions optimales (VM quasi vide, backup local, commandes préparées). En production avec des VMs de 50-100 Go, un stockage réseau distant et une procédure de décision humaine, un RTO réaliste pour ce type de PRA est de **30 minutes à 2 heures**. Pour descendre sous les 5 minutes automatiquement, il faut une HA complète : cluster 3 noeuds + stockage partagé temps réel (Ceph/iSCSI) + fencing IPMI.

---

## Vue d'ensemble de l'infrastructure finale

```
Noeud Proxmox (192.168.131.135)
├── Cluster : cluster-m2-pve (Corosync knet)
├── Stockages
│   ├── local (dir)       → /var/lib/vz — ISO, backups
│   ├── local-lvm (thin)  → pve/data 112.9G — disques VM
│   ├── nfs-backup (nfs)  → VM 110:/srv/nfs/proxmox — backups
│   ├── iscsi-store (iSCSI) → LUN 3G brut
│   ├── zfs-tank (mirror) → loop1+loop2 — VM + datasets
│   └── demo-simple (zfs) → loop0 — démonstration
├── VMs TP1/TP2 (stopped)
│   ├── 101 web01, 102 db01, 103 admin01, 104 test-rollback01
│   └── 200 fw01, 201 web-dmz01, 202 db-srv01, 203 client01, 204 admin-mgmt01
├── VMs TP3/TP4 (running)
│   ├── 110 srv-nfs-iscsi  → 192.168.131.147 (NFS + iSCSI)
│   ├── 120 zfs-test01     → 192.168.131.148 (nginx, disque ZFS)
│   ├── 121 zfs-test01     → 192.168.131.149 (restauration TP3)
│   ├── 130 app-prod01     → 192.168.131.150 (nginx, STOPPED — panne simulée)
│   └── 131 app-prod01-restored → 192.168.131.151 (nginx, restauration PRA)
└── Template
    └── 9000 tmpl-debian-13-base (Debian 13.5 + cloud-init + QEMU GA)
```

---

## Documents produits

| Fichier | Contenu |
|---------|---------|
| [TP1et2-Proxmox-DimitriANGELY-rendu.pdf](TP1_et_TP2/TP1et2-Proxmox-DimitriANGELY-rendu.pdf) | Documentation complète TP1 et 2 avec sorties terminales réelles |
| [TP3_Proxmox_Stockage_Dimitri_ANGELY-rendu.pdf](TP3/TP3_ProxmoxStockage_Dimitri_ANGELY-rendu.pdf) | Documentation complète TP3 avec sorties terminales réelles |
| [TP4_Proxmox_Cluster_HA_PRA_Dimitri_ANGELY-rendu.pdf](TP4/TP4_Proxmox_Cluster_HA_PRA_Dimitri_ANGELY-rendu.pdf) | Documentation complète TP4 avec timestamps RTO mesurés |
| [README.md](README.md) | Ce fichier |

---

*Dimitri ANGELY — M2 Infrastructure Cloud et Devops (RNCP39582) — Juin 2026*