# Chapitre 34 - Gestion des disques et partitions

## Table des matières
- [Introduction](#introduction)
- [Architecture des disques](#architecture-des-disques)
- [Découverte et identification des disques](#découverte-et-identification-des-disques)
- [Partitionnement avancé](#partitionnement-avancé)
- [Systèmes de fichiers](#systèmes-de-fichiers)
- [Montage et démontage](#montage-et-démontage)
- [Gestion de l'espace disque](#gestion-de-lespace-disque)
- [RAID et redondance](#raid-et-redondance)
- [LVM : Logical Volume Management](#lvm--logical-volume-management)
- [Monitoring et maintenance](#monitoring-et-maintenance)
- [Récupération et dépannage](#récupération-et-dépannage)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La gestion des disques et partitions représente l'une des compétences les plus critiques de l'administration système. Au-delà du simple stockage de données, il s'agit de concevoir des architectures de stockage robustes, performantes, et évolutives.

Imaginez les disques comme les fondations d'une maison : une bonne architecture permet d'ajouter des étages, de redistribuer l'espace, et de résister aux tempêtes sans compromettre la stabilité globale. Dans un système Linux moderne, la gestion du stockage va bien au-delà du partitionnement de base : elle inclut LVM pour la flexibilité, RAID pour la redondance, et des systèmes de fichiers avancés pour les performances et la fiabilité.

## Architecture des disques

### Types de disques

**Disques physiques** :
```bash
#!/bin/bash
# Types de disques

# Disques SATA/SAS
# /dev/sda, /dev/sdb, etc.

# Disques NVMe
# /dev/nvme0n1, /dev/nvme0n2, etc.

# Disques virtuels
# /dev/vda, /dev/vdb (dans les VMs)

# Partitions
# /dev/sda1, /dev/sda2, etc.
# /dev/nvme0n1p1, /dev/nvme0n1p2, etc.

# Vérifier le type de disque
lsblk -d -o NAME,TYPE,SIZE,MODEL
```

**Table de partitionnement** :
```bash
#!/bin/bash
# Types de tables de partitionnement

# MBR (Master Boot Record) - Legacy
# - Limite : 2TB par disque
# - Maximum 4 partitions primaires
# - Partitions étendues pour plus de partitions

# GPT (GUID Partition Table) - Moderne
# - Support disques > 2TB
# - Jusqu'à 128 partitions
# - Meilleure redondance
# - Support UEFI

# Vérifier le type de table
sudo fdisk -l /dev/sda | grep "Disklabel type"
sudo parted /dev/sda print | grep "Partition Table"
```

## Découverte et identification des disques

### lsblk : liste des blocs

**Utilisation de base** :
```bash
#!/bin/bash
# Utilisation de lsblk

# Vue d'ensemble
lsblk

# Vue détaillée avec système de fichiers
lsblk -f

# Vue avec UUID et mountpoints
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,UUID

# Vue en arbre
lsblk -t

# Informations sur un disque spécifique
lsblk /dev/sda

# Exclure les disques de boucle
lsblk -e 7
```

**Scripts utilitaires avec lsblk** :
```bash
#!/bin/bash
# Scripts avec lsblk

# Lister tous les disques (sans partitions)
list_disks() {
    lsblk -d -n -o NAME,SIZE,TYPE | grep disk
}

# Trouver les disques non partitionnés
find_unpartitioned_disks() {
    lsblk -d -n -o NAME,TYPE | grep disk | while read -r name type; do
        if ! lsblk "/dev/$name" | grep -q part; then
            echo "/dev/$name"
        fi
    done
}

# Espace total disponible
total_disk_space() {
    lsblk -b -d -n -o SIZE | awk '{sum+=$1} END {printf "%.2f TB\n", sum/1024/1024/1024/1024}'
}
```

### fdisk et outils de partitionnement

**fdisk : partitionnement classique** :
```bash
#!/bin/bash
# Utilisation de fdisk

# Lister les partitions
sudo fdisk -l

# Lister un disque spécifique
sudo fdisk -l /dev/sda

# Mode interactif
# sudo fdisk /dev/sda
# Commandes dans fdisk :
# n : Nouvelle partition
# d : Supprimer partition
# p : Afficher partitions
# t : Changer type
# w : Écrire et quitter
# q : Quitter sans sauvegarder

# Créer une partition non-interactif
echo -e "n\np\n1\n\n+10G\nw" | sudo fdisk /dev/sda
```

**parted : partitionnement moderne** :
```bash
#!/bin/bash
# Utilisation de parted

# Afficher les informations
sudo parted /dev/sda print

# Créer une table GPT
sudo parted /dev/sda mklabel gpt

# Créer une partition
sudo parted /dev/sda mkpart primary ext4 0% 50%

# Créer une partition avec taille spécifique
sudo parted /dev/sda mkpart primary ext4 50% 100%

# Supprimer une partition
sudo parted /dev/sda rm 1

# Redimensionner une partition (nécessite que le FS soit démonté)
sudo parted /dev/sda resizepart 1 20GB

# Script non-interactif
sudo parted /dev/sda << EOF
mklabel gpt
mkpart primary ext4 0% 50%
mkpart primary ext4 50% 100%
print
quit
EOF
```

**gdisk : pour GPT** :
```bash
#!/bin/bash
# Utilisation de gdisk

# Mode interactif
# sudo gdisk /dev/sda
# Commandes similaires à fdisk mais pour GPT

# Créer une partition GPT non-interactif
echo -e "n\n\n\n+10G\n8300\nw\ny" | sudo gdisk /dev/sda
```

## Partitionnement avancé

### Stratégies de partitionnement

**Partitionnement système** :
```bash
#!/bin/bash
# Stratégie de partitionnement système

# Partitionnement recommandé pour serveur :
# /boot : 512MB - 1GB (EFI) ou 256MB (BIOS)
# / : 20-50GB (système)
# /home : Reste de l'espace ou disque séparé
# /var : 10-20GB (logs, cache)
# /tmp : 5-10GB (fichiers temporaires)
# swap : 2x RAM (ou selon besoins)

create_system_partitions() {
    local disk="$1"
    local boot_size="${2:-512M}"
    local root_size="${3:-50G}"
    local swap_size="${4:-8G}"
    
    sudo parted "$disk" << EOF
mklabel gpt
mkpart ESP fat32 1MiB ${boot_size}
set 1 boot on
mkpart primary ext4 ${boot_size} ${root_size}
mkpart primary linux-swap ${root_size} $(echo "$root_size" | sed 's/G/ + '$swap_size'/')
mkpart primary ext4 $(echo "$root_size" | sed 's/G/ + '$swap_size'/') 100%
print
quit
EOF
    
    echo "Partitions créées"
}
```

**Partitionnement pour données** :
```bash
#!/bin/bash
# Partitionnement pour données

# Stratégie pour serveur de données :
# Partition séparée pour /data
# Plusieurs partitions pour organisation
# LVM pour flexibilité

create_data_partitions() {
    local disk="$1"
    
    sudo parted "$disk" << EOF
mklabel gpt
mkpart primary ext4 0% 30%
mkpart primary ext4 30% 60%
mkpart primary ext4 60% 100%
print
quit
EOF
}
```

### Scripts de partitionnement automatisés

**Partitionnement intelligent** :
```bash
#!/bin/bash
# Partitionnement automatisé intelligent

auto_partition_disk() {
    local disk="$1"
    local disk_size=$(sudo blockdev --getsize64 "$disk")
    local disk_size_gb=$((disk_size / 1024 / 1024 / 1024))
    
    echo "Taille du disque: ${disk_size_gb}GB"
    
    # Calculer les tailles selon la taille du disque
    if [ $disk_size_gb -lt 100 ]; then
        # Petit disque : partition simple
        echo "Petit disque détecté, partitionnement simple"
        sudo parted "$disk" mklabel gpt
        sudo parted "$disk" mkpart primary ext4 0% 100%
    elif [ $disk_size_gb -lt 500 ]; then
        # Disque moyen : partitionnement standard
        echo "Disque moyen détecté, partitionnement standard"
        sudo parted "$disk" mklabel gpt
        sudo parted "$disk" mkpart ESP fat32 1MiB 512MiB
        sudo parted "$disk" set 1 boot on
        sudo parted "$disk" mkpart primary ext4 512MiB 50GiB
        sudo parted "$disk" mkpart primary ext4 50GiB 100%
    else
        # Grand disque : utiliser LVM
        echo "Grand disque détecté, utilisation de LVM recommandée"
        sudo parted "$disk" mklabel gpt
        sudo parted "$disk" mkpart primary ext4 0% 100%
        # Créer un volume physique LVM ensuite
    fi
}
```

## Systèmes de fichiers

### Création de systèmes de fichiers

**mkfs : création de systèmes de fichiers** :
```bash
#!/bin/bash
# Création de systèmes de fichiers

# ext4 (recommandé pour la plupart des cas)
sudo mkfs.ext4 /dev/sda1
sudo mkfs.ext4 -L "DATA" /dev/sda1  # Avec label

# xfs (pour gros fichiers)
sudo mkfs.xfs /dev/sda1
sudo mkfs.xfs -L "DATA" /dev/sda1

# btrfs (avec fonctionnalités avancées)
sudo mkfs.btrfs /dev/sda1
sudo mkfs.btrfs -L "DATA" /dev/sda1

# Options avancées
sudo mkfs.ext4 -b 4096 -i 2048 -m 2 /dev/sda1
# -b : Taille de bloc
# -i : Ratio inodes
# -m : Pourcentage réservé pour root

# Vérifier avant formatage
sudo blkid /dev/sda1
```

**Options de formatage** :
```bash
#!/bin/bash
# Options de formatage avancées

# ext4 avec options
sudo mkfs.ext4 \
    -L "MY_DATA" \
    -b 4096 \
    -i 2048 \
    -m 1 \
    -O ^has_journal \
    /dev/sda1

# xfs avec options
sudo mkfs.xfs \
    -L "MY_DATA" \
    -d agcount=4 \
    -l size=128m \
    /dev/sda1

# btrfs avec options
sudo mkfs.btrfs \
    -L "MY_DATA" \
    -d single \
    -m single \
    /dev/sda1
```

### Vérification et réparation

**fsck : vérification et réparation** :
```bash
#!/bin/bash
# Vérification et réparation

# Vérifier un système de fichiers (sans réparation)
sudo fsck -n /dev/sda1

# Vérifier et réparer automatiquement
sudo fsck -a /dev/sda1

# Vérifier avec réparation interactive
sudo fsck -r /dev/sda1

# Vérifier avec affichage détaillé
sudo fsck -v /dev/sda1

# Forcer la vérification même si propre
sudo fsck -f /dev/sda1

# Vérifier tous les systèmes de fichiers
sudo fsck -A -y

# Vérifier au démarrage
# Ajouter 'fsck.mode=force' aux paramètres du kernel
```

**Outils spécifiques** :
```bash
#!/bin/bash
# Outils de vérification spécifiques

# ext4
sudo e2fsck -f /dev/sda1
sudo e2fsck -c /dev/sda1  # Vérifier les blocs défectueux

# xfs
sudo xfs_repair /dev/sda1
sudo xfs_check /dev/sda1  # Vérification seule

# btrfs
sudo btrfs check /dev/sda1
sudo btrfs scrub start /dev/sda1  # Nettoyage en ligne
```

## Montage et démontage

### Montage de base

**mount : montage manuel** :
```bash
#!/bin/bash
# Montage de systèmes de fichiers

# Montage simple
sudo mount /dev/sda1 /mnt

# Montage avec type spécifique
sudo mount -t ext4 /dev/sda1 /mnt

# Montage avec options
sudo mount -o rw,noatime /dev/sda1 /mnt

# Montage en lecture seule
sudo mount -o ro /dev/sda1 /mnt

# Montage avec UUID
sudo mount UUID="1234-5678" /mnt

# Montage avec label
sudo mount LABEL="MY_DATA" /mnt

# Montage depuis /etc/fstab
sudo mount /mnt  # Si entrée dans fstab
```

**Options de montage avancées** :
```bash
#!/bin/bash
# Options de montage avancées

# Options courantes
sudo mount -o \
    rw,noatime,nodiratime,noexec,nosuid,nodev \
    /dev/sda1 /mnt

# Explications :
# rw : Lecture/écriture
# ro : Lecture seule
# noatime : Ne pas mettre à jour les timestamps d'accès
# nodiratime : Pas d'atime pour les répertoires
# noexec : Empêcher l'exécution de binaires
# nosuid : Ignorer les bits SUID/SGID
# nodev : Ignorer les périphériques spéciaux
# defaults : Options par défaut (rw,suid,dev,exec,auto,nouser,async)

# Montage avec bind
sudo mount --bind /source /destination

# Montage récursif (bind)
sudo mount --rbind /source /destination

# Montage avec remount
sudo mount -o remount,rw /mnt
```

### /etc/fstab : montage automatique

**Configuration fstab** :
```bash
#!/bin/bash
# Configuration /etc/fstab

# Format : device mountpoint fstype options dump pass
# Exemple :
# UUID=1234-5678 /mnt/data ext4 defaults 0 2

# Vérifier la syntaxe
sudo mount -a  # Monte tous les systèmes de fichiers de fstab

# Ajouter une entrée
add_fstab_entry() {
    local uuid="$1"
    local mountpoint="$2"
    local fstype="${3:-ext4}"
    local options="${4:-defaults}"
    
    echo "UUID=$uuid $mountpoint $fstype $options 0 2" | sudo tee -a /etc/fstab
    
    # Créer le point de montage
    sudo mkdir -p "$mountpoint"
    
    # Tester le montage
    sudo mount "$mountpoint"
}

# Lister les entrées fstab
list_fstab_entries() {
    grep -v "^#" /etc/fstab | grep -v "^$"
}
```

**Génération automatique de fstab** :
```bash
#!/bin/bash
# Génération automatique de fstab

generate_fstab_entry() {
    local device="$1"
    local mountpoint="$2"
    local uuid=$(sudo blkid -s UUID -o value "$device")
    local fstype=$(sudo blkid -s TYPE -o value "$device")
    
    echo "UUID=$uuid $mountpoint $fstype defaults 0 2"
}

# Générer fstab pour tous les systèmes de fichiers montés
generate_complete_fstab() {
    {
        echo "# /etc/fstab: static file system information"
        echo "#"
        echo "# <file system> <mount point>   <type>  <options>       <dump>  <pass>"
        echo ""
        
        mount | awk '$3 !~ /^(proc|sys|dev|run|tmpfs|devpts)/ {
            device = $1
            mountpoint = $3
            fstype = $5
            
            # Obtenir UUID
            uuid_cmd = "blkid -s UUID -o value " device
            uuid_cmd | getline uuid
            close(uuid_cmd)
            
            if (uuid != "") {
                print "UUID=" uuid " " mountpoint " " fstype " defaults 0 2"
            }
        }'
    } > /tmp/fstab_generated
    
    echo "Fichier fstab généré: /tmp/fstab_generated"
    echo "Vérifiez avant de remplacer /etc/fstab"
}
```

### umount : démontage

**Démontage** :
```bash
#!/bin/bash
# Démontage de systèmes de fichiers

# Démontage simple
sudo umount /mnt

# Démontage par device
sudo umount /dev/sda1

# Démontage forcé (si occupé)
sudo umount -f /mnt

# Démontage paresseux (lazy)
sudo umount -l /mnt  # Démontage après que les processus se terminent

# Démontage récursif (bind mounts)
sudo umount -R /mnt

# Trouver ce qui empêche le démontage
find_mount_blockers() {
    local mountpoint="$1"
    
    echo "Processus utilisant $mountpoint:"
    lsof "$mountpoint" 2>/dev/null
    
    echo ""
    echo "Systèmes de fichiers montés dans $mountpoint:"
    mount | grep "$mountpoint"
}

# Démontage sécurisé
safe_umount() {
    local mountpoint="$1"
    
    # Vérifier si monté
    if ! mountpoint -q "$mountpoint"; then
        echo "$mountpoint n'est pas monté"
        return 1
    fi
    
    # Essayer démontage normal
    if sudo umount "$mountpoint" 2>/dev/null; then
        echo "Démonté avec succès"
        return 0
    fi
    
    # Trouver les processus bloquants
    local pids=$(lsof +D "$mountpoint" 2>/dev/null | awk 'NR>1 {print $2}' | sort -u)
    
    if [ -n "$pids" ]; then
        echo "Processus bloquants: $pids"
        echo "Arrêt des processus..."
        echo "$pids" | xargs sudo kill
        
        sleep 2
        
        # Réessayer
        sudo umount "$mountpoint"
    else
        # Démontage forcé
        echo "Démontage forcé..."
        sudo umount -f "$mountpoint"
    fi
}
```

## Gestion de l'espace disque

### df : espace disponible

**Utilisation avancée de df** :
```bash
#!/bin/bash
# Utilisation avancée de df

# Vue d'ensemble humaine
df -h

# Vue avec types de systèmes de fichiers
df -Th

# Vue avec inodes
df -ih

# Vue avec tailles en blocs
df -B 1G  # En GB

# Vue pour un système de fichiers spécifique
df -h /home

# Exclure certains types
df -h -x tmpfs -x devtmpfs

# Sortie en format scriptable
df -h --output=source,fstype,size,used,avail,pcent,target
```

**Scripts avec df** :
```bash
#!/bin/bash
# Scripts utilitaires avec df

# Alertes d'espace disque
check_disk_space() {
    local threshold="${1:-80}"  # Pourcentage par défaut
    
    df -h | awk -v threshold="$threshold" '
    NR>1 {
        gsub(/%/, "", $5)
        if ($5 > threshold) {
            print "⚠️  " $6 " est à " $5 "% (" $4 " disponible)"
        }
    }'
}

# Trouver les systèmes de fichiers presque pleins
find_full_filesystems() {
    df -h | awk '
    NR>1 {
        gsub(/%/, "", $5)
        if ($5 > 90) {
            print $6 " : " $5 "% utilisé (" $4 " disponible)"
        }
    }'
}

# Statistiques d'espace
disk_statistics() {
    echo "=== Statistiques d'espace disque ==="
    df -h | awk '
    NR>1 {
        total+=$2
        used+=$3
        avail+=$4
    }
    END {
        printf "Total: %.2f GB\n", total
        printf "Utilisé: %.2f GB (%.1f%%)\n", used, (used/total)*100
        printf "Disponible: %.2f GB (%.1f%%)\n", avail, (avail/total)*100
    }'
}
```

### du : utilisation de l'espace

**Utilisation avancée de du** :
```bash
#!/bin/bash
# Utilisation avancée de du

# Taille d'un répertoire
du -sh /home/user

# Taille avec profondeur limitée
du -h --max-depth=2 /home

# Trier par taille
du -h /home | sort -hr | head -10

# Exclure certains types
du -h --exclude="*.iso" --exclude="*.vmdk" /home

# Taille de tous les répertoires
du -h --max-depth=1 / | sort -hr

# Format personnalisé
du -h --time --time-style=iso /home

# Taille apparente (sans liens)
du -sh --apparent-size /home
```

**Scripts avec du** :
```bash
#!/bin/bash
# Scripts avec du

# Trouver les plus gros répertoires
find_largest_directories() {
    local path="${1:-.}"
    local count="${2:-10}"
    
    du -h "$path" 2>/dev/null | \
    sort -hr | \
    head -"$count"
}

# Trouver les fichiers les plus volumineux
find_largest_files() {
    local path="${1:-.}"
    local count="${2:-10}"
    
    find "$path" -type f -exec du -h {} + 2>/dev/null | \
    sort -hr | \
    head -"$count"
}

# Analyse d'espace par type de fichier
analyze_by_file_type() {
    local path="${1:-.}"
    
    find "$path" -type f | \
    sed 's/.*\.//' | \
    sort | uniq -c | sort -rn | \
    while read -r count ext; do
        if [ -n "$ext" ]; then
            size=$(find "$path" -name "*.$ext" -exec du -ch {} + 2>/dev/null | tail -1 | cut -f1)
            echo "$ext: $count fichiers, $size"
        fi
    done
}
```

## RAID et redondance

### mdadm : gestion RAID logiciel

**Création de RAID** :
```bash
#!/bin/bash
# Création de RAID avec mdadm

# RAID 1 (mirroring) - 2 disques minimum
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda /dev/sdb

# RAID 5 (parité distribuée) - 3 disques minimum
sudo mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sda /dev/sdb /dev/sdc

# RAID 6 (double parité) - 4 disques minimum
sudo mdadm --create /dev/md0 --level=6 --raid-devices=4 /dev/sda /dev/sdb /dev/sdc /dev/sdd

# RAID 10 (mirroring + striping) - 4 disques minimum
sudo mdadm --create /dev/md0 --level=10 --raid-devices=4 /dev/sda /dev/sdb /dev/sdc /dev/sdd

# Avec spare disk
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 --spare-devices=1 /dev/sda /dev/sdb /dev/sdc
```

**Gestion de RAID** :
```bash
#!/bin/bash
# Gestion de RAID

# Statut du RAID
sudo mdadm --detail /dev/md0
cat /proc/mdstat

# Arrêter un RAID
sudo mdadm --stop /dev/md0

# Assembler un RAID existant
sudo mdadm --assemble /dev/md0 /dev/sda /dev/sdb

# Ajouter un disque à un RAID
sudo mdadm --add /dev/md0 /dev/sde

# Retirer un disque
sudo mdadm --remove /dev/md0 /dev/sda

# Marquer un disque comme défaillant
sudo mdadm --fail /dev/md0 /dev/sda

# Sauvegarder la configuration
sudo mdadm --detail --scan > /etc/mdadm/mdadm.conf
```

## LVM : Logical Volume Management

### Concepts LVM

**Architecture LVM** :
```bash
#!/bin/bash
# Architecture LVM

# Niveaux LVM :
# Physical Volume (PV) : Disque ou partition physique
# Volume Group (VG) : Groupe de volumes physiques
# Logical Volume (LV) : Volume logique créé depuis un VG

# Créer un Physical Volume
sudo pvcreate /dev/sda1

# Créer un Volume Group
sudo vgcreate myvg /dev/sda1

# Créer un Logical Volume
sudo lvcreate -L 10G -n mylv myvg

# Formater le Logical Volume
sudo mkfs.ext4 /dev/myvg/mylv

# Monter
sudo mount /dev/myvg/mylv /mnt
```

### Gestion LVM avancée

**Opérations LVM** :
```bash
#!/bin/bash
# Opérations LVM avancées

# Étendre un Logical Volume
sudo lvextend -L +5G /dev/myvg/mylv
sudo resize2fs /dev/myvg/mylv  # Pour ext4

# Réduire un Logical Volume (dangereux)
sudo umount /mnt
sudo e2fsck -f /dev/myvg/mylv
sudo resize2fs /dev/myvg/mylv 5G
sudo lvreduce -L 5G /dev/myvg/mylv
sudo mount /dev/myvg/mylv /mnt

# Ajouter un Physical Volume à un VG
sudo pvcreate /dev/sdb1
sudo vgextend myvg /dev/sdb1

# Retirer un Physical Volume
sudo pvmove /dev/sda1
sudo vgreduce myvg /dev/sda1

# Snapshots LVM
sudo lvcreate -L 1G -s -n mysnapshot /dev/myvg/mylv
sudo mount /dev/myvg/mysnapshot /mnt/snapshot
```

## Monitoring et maintenance

### Surveillance des disques

**SMART : monitoring de santé** :
```bash
#!/bin/bash
# Monitoring SMART

# Installer smartmontools
# sudo apt install smartmontools

# Informations SMART
sudo smartctl -a /dev/sda

# Test SMART court
sudo smartctl -t short /dev/sda

# Test SMART long
sudo smartctl -t long /dev/sda

# Vérifier les erreurs
sudo smartctl -l error /dev/sda

# Surveillance continue
sudo smartctl -A /dev/sda | grep -E "(Reallocated|Pending|Uncorrectable)"
```

**Monitoring d'I/O** :
```bash
#!/bin/bash
# Monitoring d'I/O

# iostat (sysstat)
sudo apt install sysstat
iostat -x 1

# iotop
sudo apt install iotop
sudo iotop

# Script de monitoring
monitor_disk_io() {
    local device="${1:-sda}"
    
    while true; do
        clear
        echo "=== Monitoring I/O pour /dev/$device ==="
        iostat -x 1 2 | grep "$device"
        sleep 5
    done
}
```

## Récupération et dépannage

### Récupération de données

**Outils de récupération** :
```bash
#!/bin/bash
# Outils de récupération

# testdisk pour récupération de partitions
# sudo apt install testdisk
# sudo testdisk

# photorec pour récupération de fichiers
# sudo photorec

# extundelete pour ext3/ext4
# sudo apt install extundelete
# sudo extundelete /dev/sda1 --restore-all

# ddrescue pour copie de disque défaillant
# sudo apt install gddrescue
# sudo ddrescue /dev/sda /dev/sdb rescue.log
```

## Scripts d'automatisation

### Script complet de gestion

**Gestionnaire de disques** :
```bash
#!/bin/bash
# Gestionnaire complet de disques

set -euo pipefail

# Fonction principale
disk_manager() {
    local action="$1"
    shift
    
    case "$action" in
        list)
            list_all_disks
            ;;
        partition)
            partition_disk "$@"
            ;;
        format)
            format_partition "$@"
            ;;
        mount)
            mount_partition "$@"
            ;;
        umount)
            umount_partition "$@"
            ;;
        *)
            echo "Usage: disk_manager {list|partition|format|mount|umount}"
            exit 1
            ;;
    esac
}

list_all_disks() {
    echo "=== Disques disponibles ==="
    lsblk -d -o NAME,SIZE,TYPE,MODEL
    echo ""
    echo "=== Partitions ==="
    lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT
}

# Utilisation
# disk_manager list
# disk_manager partition /dev/sda gpt
# disk_manager format /dev/sda1 ext4 "MY_DATA"
```

## Conclusion

La gestion des disques et partitions est fondamentale pour l'administration système Linux. En maîtrisant les outils de partitionnement (fdisk, parted, gdisk), les systèmes de fichiers (ext4, xfs, btrfs), et les technologies avancées comme LVM et RAID, vous pouvez créer des architectures de stockage robustes, performantes et évolutives.

Un système de stockage bien conçu permet non seulement de stocker des données, mais aussi de les protéger contre les défaillances, de les optimiser pour les performances, et de les adapter aux besoins changeants sans interruption de service.

Dans le chapitre suivant, nous explorerons la gestion de la swap et de la mémoire, découvrant comment optimiser l'utilisation de la RAM et du stockage pour les performances système.
