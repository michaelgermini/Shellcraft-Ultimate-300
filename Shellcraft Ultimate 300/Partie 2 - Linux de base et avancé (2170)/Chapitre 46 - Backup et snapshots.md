# Chapitre 46 - Backup et snapshots

## Table des matières
- [Introduction](#introduction)
- [Stratégies de sauvegarde](#stratégies-de-sauvegarde)
- [Règle 3-2-1](#règle-3-2-1)
- [Types de sauvegardes](#types-de-sauvegardes)
- [Outils de backup Linux](#outils-de-backup-linux)
- [Sauvegardes avec tar](#sauvegardes-avec-tar)
- [Sauvegardes avec rsync](#sauvegardes-avec-rsync)
- [Snapshots système](#snapshots-système)
- [Snapshots LVM](#snapshots-lvm)
- [Snapshots btrfs](#snapshots-btrfs)
- [Backup automatique](#backup-automatique)
- [Sauvegardes incrémentielles](#sauvegardes-incrémentielles)
- [Restauration et récupération](#restauration-et-récupération)
- [Stockage et archivage](#stockage-et-archivage)
- [Vérification d'intégrité](#vérification-dintégrité)
- [Planification et conformité](#planification-et-conformité)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Les stratégies de sauvegarde constituent l'assurance-vie de tout système informatique. Au-delà de la simple copie de fichiers, il s'agit de concevoir des architectures de protection des données qui permettent de récupérer rapidement et complètement en cas de sinistre.

Imaginez les sauvegardes comme un système d'assurance sophistiqué : elles couvrent différents types de risques (perte de données, corruption, attaques), offrent différents niveaux de couverture (complet, incrémental, différentiel), et permettent de restaurer votre système dans l'état souhaité. Dans un environnement Linux moderne, les sauvegardes s'étendent des simples copies de fichiers aux snapshots système instantanés, en passant par les sauvegardes incrémentielles intelligentes.

## Stratégies de sauvegarde

### Principes fondamentaux

**Objectifs de la sauvegarde** :
- Protection contre la perte de données
- Récupération après sinistre
- Conformité réglementaire
- Historique des versions
- Récupération granulaire

**Critères de qualité** :
- RTO (Recovery Time Objective) : Temps de récupération
- RPO (Recovery Point Objective) : Point de récupération acceptable
- Fiabilité : Taux de succès des restaurations
- Sécurité : Chiffrement et protection des sauvegardes

## Règle 3-2-1

### Principe 3-2-1

**Règle 3-2-1** :
```bash
#!/bin/bash
# Règle 3-2-1 de sauvegarde

# 3 copies des données
# - Original
# - Sauvegarde locale
# - Sauvegarde distante

# 2 types de stockage différents
# - Disque dur local
# - Stockage cloud / NAS

# 1 copie hors site
# - Sauvegarde géographiquement distante
```

**Implémentation** :
```bash
#!/bin/bash
# Implémentation de la règle 3-2-1

implement_321_backup() {
    local source="$1"
    local local_backup="/backup/local"
    local remote_backup="user@remote-server:/backup"
    
    # Copie 1 : Sauvegarde locale
    rsync -av "$source/" "$local_backup/"
    
    # Copie 2 : Sauvegarde distante
    rsync -av "$source/" "$remote_backup/"
    
    # Copie 3 : Cloud (exemple avec rclone)
    rclone sync "$source/" remote:backup/
    
    echo "Règle 3-2-1 appliquée"
}
```

## Types de sauvegardes

### Sauvegarde complète

**Sauvegarde complète** :
```bash
#!/bin/bash
# Sauvegarde complète

full_backup() {
    local source="$1"
    local dest="$2"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    tar -czf "${dest}/full_backup_${timestamp}.tar.gz" -C "$(dirname "$source")" "$(basename "$source")"
    
    echo "Sauvegarde complète créée : full_backup_${timestamp}.tar.gz"
}

# Avantages : Restauration simple, complète
# Inconvénients : Temps long, espace important
```

### Sauvegarde incrémentielle

**Sauvegarde incrémentielle** :
```bash
#!/bin/bash
# Sauvegarde incrémentielle

incremental_backup() {
    local source="$1"
    local dest="$2"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local last_backup=$(ls -t "${dest}"/backup_* 2>/dev/null | head -1)
    
    if [ -n "$last_backup" ]; then
        # Sauvegarde incrémentielle (seulement les changements)
        tar -czf "${dest}/incr_backup_${timestamp}.tar.gz" \
            --newer-mtime="$(stat -c %Y "$last_backup")" \
            -C "$(dirname "$source")" "$(basename "$source")"
    else
        # Première sauvegarde = complète
        full_backup "$source" "$dest"
    fi
}

# Avantages : Rapide, peu d'espace
# Inconvénients : Restauration nécessite toutes les sauvegardes
```

### Sauvegarde différentielle

**Sauvegarde différentielle** :
```bash
#!/bin/bash
# Sauvegarde différentielle

differential_backup() {
    local source="$1"
    local dest="$2"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local last_full=$(ls -t "${dest}"/full_backup_* 2>/dev/null | head -1)
    
    if [ -n "$last_full" ]; then
        # Sauvegarde différentielle (depuis la dernière complète)
        tar -czf "${dest}/diff_backup_${timestamp}.tar.gz" \
            --newer-mtime="$(stat -c %Y "$last_full")" \
            -C "$(dirname "$source")" "$(basename "$source")"
    else
        full_backup "$source" "$dest"
    fi
}

# Avantages : Restauration plus simple que incrémentielle
# Inconvénients : Plus d'espace que incrémentielle
```

## Outils de backup Linux

### Outils disponibles

**Outils principaux** :
```bash
#!/bin/bash
# Outils de backup Linux

# tar : Archivage et compression
# rsync : Synchronisation efficace
# duplicity : Sauvegardes chiffrées incrémentielles
# borgbackup : Déduplication et compression
# rsnapshot : Snapshots avec rsync
# timeshift : Snapshots système (GUI)
# btrfs/ZFS : Snapshots natifs du système de fichiers
```

## Sauvegardes avec tar

### Utilisation de base

**Sauvegardes tar** :
```bash
#!/bin/bash
# Sauvegardes avec tar

# Sauvegarde simple
tar -czf backup.tar.gz /home/user

# Sauvegarde avec exclusion
tar -czf backup.tar.gz --exclude="*.tmp" --exclude=".cache" /home/user

# Sauvegarde avec liste d'exclusions
tar -czf backup.tar.gz -X exclude.txt /home/user

# Sauvegarde avec vérification
tar -czf backup.tar.gz /home/user && tar -tzf backup.tar.gz > /dev/null

# Sauvegarde avec progression
tar -czf backup.tar.gz --checkpoint=.1000 --checkpoint-action=echo="%u fichiers archivés" /home/user
```

### Scripts tar avancés

**Scripts de sauvegarde tar** :
```bash
#!/bin/bash
# Scripts de sauvegarde tar

backup_with_tar() {
    local source="$1"
    local dest="$2"
    local name="${3:-backup}"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    local backup_file="${dest}/${name}_${timestamp}.tar.gz"
    
    # Créer la sauvegarde
    tar -czf "$backup_file" \
        --exclude="*.tmp" \
        --exclude="*.log" \
        --exclude=".cache" \
        -C "$(dirname "$source")" "$(basename "$source")"
    
    # Vérifier l'intégrité
    if tar -tzf "$backup_file" > /dev/null 2>&1; then
        echo "✓ Sauvegarde réussie : $backup_file"
        
        # Créer le checksum
        sha256sum "$backup_file" > "${backup_file}.sha256"
        
        return 0
    else
        echo "✗ Échec de la sauvegarde"
        rm -f "$backup_file"
        return 1
    fi
}
```

## Sauvegardes avec rsync

### Sauvegardes incrémentielles avec hard links

**Sauvegardes rsync avec hard links** :
```bash
#!/bin/bash
# Sauvegardes rsync avec hard links

rsync_incremental_backup() {
    local source="$1"
    local dest_base="$2"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local current_backup="${dest_base}/backup_${timestamp}"
    local previous_backup=$(ls -td "${dest_base}"/backup_* 2>/dev/null | head -1)
    
    mkdir -p "$current_backup"
    
    if [ -n "$previous_backup" ]; then
        # Utiliser hard links pour économiser l'espace
        rsync -av --link-dest="$previous_backup" "$source/" "$current_backup/"
    else
        # Première sauvegarde
        rsync -av "$source/" "$current_backup/"
    fi
    
    echo "Sauvegarde créée : $current_backup"
}

# Avantage : Chaque sauvegarde semble complète mais utilise peu d'espace
```

### Rotation des sauvegardes

**Rotation automatique** :
```bash
#!/bin/bash
# Rotation des sauvegardes

rotate_backups() {
    local backup_dir="$1"
    local keep_daily="${2:-7}"
    local keep_weekly="${3:-4}"
    local keep_monthly="${4:-12}"
    
    # Supprimer les sauvegardes quotidiennes anciennes
    find "$backup_dir" -name "backup_*" -type d -mtime +$keep_daily ! -name "backup_*-*-01" ! -name "backup_*-*-0[1-7]" -exec rm -rf {} +
    
    # Garder les sauvegardes hebdomadaires (premier jour de la semaine)
    # Garder les sauvegardes mensuelles (premier jour du mois)
    
    echo "Rotation des sauvegardes terminée"
}
```

## Snapshots système

### Concept de snapshot

**Qu'est-ce qu'un snapshot ?** :
- Image instantanée d'un système de fichiers à un moment donné
- Création très rapide (quelques secondes)
- Utilise peu d'espace (copie à l'écriture)
- Permet la restauration rapide

**Types de snapshots** :
- Snapshots LVM : Au niveau du volume logique
- Snapshots btrfs : Natifs du système de fichiers
- Snapshots ZFS : Système de fichiers avancé
- Snapshots rsnapshot : Basés sur rsync

## Snapshots LVM

### Création de snapshots LVM

**Snapshots LVM** :
```bash
#!/bin/bash
# Snapshots LVM

# Créer un snapshot
create_lvm_snapshot() {
    local lv_name="$1"
    local snapshot_name="${lv_name}_snap"
    local snapshot_size="${2:-5G}"
    
    sudo lvcreate -L "$snapshot_size" -s -n "$snapshot_name" "/dev/vg/$lv_name"
    
    echo "Snapshot créé : $snapshot_name"
}

# Monter un snapshot
mount_lvm_snapshot() {
    local snapshot_name="$1"
    local mount_point="$2"
    
    sudo mkdir -p "$mount_point"
    sudo mount "/dev/vg/$snapshot_name" "$mount_point"
}

# Supprimer un snapshot
remove_lvm_snapshot() {
    local snapshot_name="$1"
    
    sudo umount "/dev/vg/$snapshot_name" 2>/dev/null || true
    sudo lvremove -f "/dev/vg/$snapshot_name"
    
    echo "Snapshot supprimé"
}
```

**Script de snapshot automatique** :
```bash
#!/bin/bash
# Script de snapshot automatique LVM

auto_lvm_snapshot() {
    local lv_name="$1"
    local snapshot_size="${2:-10%}"
    local keep_snapshots="${3:-5}"
    
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local snapshot_name="${lv_name}_snap_${timestamp}"
    
    # Créer le snapshot
    sudo lvcreate -l "$snapshot_size" -s -n "$snapshot_name" "/dev/vg/$lv_name"
    
    # Supprimer les anciens snapshots
    local old_snapshots=$(sudo lvs --noheadings -o name | grep "^${lv_name}_snap_" | sort | head -n -$keep_snapshots)
    
    for snap in $old_snapshots; do
        sudo lvremove -f "/dev/vg/$snap"
    done
    
    echo "Snapshot créé : $snapshot_name"
}
```

## Snapshots btrfs

### Utilisation des snapshots btrfs

**Snapshots btrfs** :
```bash
#!/bin/bash
# Snapshots btrfs

# Créer un snapshot
create_btrfs_snapshot() {
    local subvolume="$1"
    local snapshot_path="$2"
    
    sudo btrfs subvolume snapshot "$subvolume" "$snapshot_path"
    
    echo "Snapshot btrfs créé : $snapshot_path"
}

# Lister les snapshots
list_btrfs_snapshots() {
    local subvolume="$1"
    
    sudo btrfs subvolume list -s "$subvolume"
}

# Supprimer un snapshot
remove_btrfs_snapshot() {
    local snapshot_path="$1"
    
    sudo btrfs subvolume delete "$snapshot_path"
    
    echo "Snapshot supprimé"
}

# Restaurer depuis un snapshot
restore_from_btrfs_snapshot() {
    local snapshot_path="$1"
    local target="$2"
    
    # Supprimer le subvolume actuel
    sudo btrfs subvolume delete "$target"
    
    # Créer un snapshot du snapshot (restauration)
    sudo btrfs subvolume snapshot "$snapshot_path" "$target"
    
    echo "Restauration depuis snapshot terminée"
}
```

**Snapshots automatiques btrfs** :
```bash
#!/bin/bash
# Snapshots automatiques btrfs

auto_btrfs_snapshots() {
    local subvolume="$1"
    local snapshot_dir="${2:-$subvolume/.snapshots}"
    local keep_snapshots="${3:-10}"
    
    mkdir -p "$snapshot_dir"
    
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local snapshot_path="${snapshot_dir}/snapshot_${timestamp}"
    
    # Créer le snapshot
    sudo btrfs subvolume snapshot "$subvolume" "$snapshot_path"
    
    # Supprimer les anciens snapshots
    local old_snapshots=$(ls -1t "$snapshot_dir"/snapshot_* 2>/dev/null | tail -n +$((keep_snapshots + 1)))
    
    for snap in $old_snapshots; do
        sudo btrfs subvolume delete "$snap"
    done
    
    echo "Snapshot créé : $snapshot_path"
}
```

## Backup automatique

### Scripts de backup automatique

**Script de backup complet** :
```bash
#!/bin/bash
# Script de backup automatique complet

set -euo pipefail

BACKUP_ROOT="/backups"
LOG_FILE="/var/log/backup.log"
SOURCES=(
    "/home"
    "/etc"
    "/var/www"
)

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

backup_source() {
    local source="$1"
    local dest="${BACKUP_ROOT}/$(basename "$source")"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    mkdir -p "$dest"
    
    log "Sauvegarde de $source"
    
    # Utiliser rsync avec hard links
    local previous=$(ls -td "$dest"/backup_* 2>/dev/null | head -1)
    
    if [ -n "$previous" ]; then
        rsync -av --link-dest="$previous" "$source/" "${dest}/backup_${timestamp}/"
    else
        rsync -av "$source/" "${dest}/backup_${timestamp}/"
    fi
    
    log "Sauvegarde terminée : backup_${timestamp}"
}

main() {
    log "=== Début de la sauvegarde ==="
    
    for source in "${SOURCES[@]}"; do
        if [ -d "$source" ]; then
            backup_source "$source"
        else
            log "ERREUR: $source n'existe pas"
        fi
    done
    
    log "=== Sauvegarde terminée ==="
}

main "$@"
```

### Backup avec cron

**Configuration cron** :
```bash
#!/bin/bash
# Configuration cron pour backups

# Éditer crontab
crontab -e

# Sauvegarde quotidienne à 2h du matin
0 2 * * * /usr/local/bin/backup.sh

# Sauvegarde hebdomadaire le dimanche à 1h
0 1 * * 0 /usr/local/bin/weekly_backup.sh

# Sauvegarde mensuelle le premier du mois à minuit
0 0 1 * * /usr/local/bin/monthly_backup.sh
```

## Sauvegardes incrémentielles

### Stratégie incrémentielle complète

**Système de sauvegarde incrémentielle** :
```bash
#!/bin/bash
# Système de sauvegarde incrémentielle

incremental_backup_system() {
    local source="$1"
    local dest_base="$2"
    local backup_type="${3:-daily}"  # daily, weekly, monthly
    
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_dir="${dest_base}/${backup_type}_${timestamp}"
    local last_backup=$(ls -td "${dest_base}"/${backup_type}_* 2>/dev/null | head -1)
    
    mkdir -p "$backup_dir"
    
    if [ -n "$last_backup" ]; then
        # Sauvegarde incrémentielle
        rsync -av --link-dest="$last_backup" "$source/" "$backup_dir/"
    else
        # Première sauvegarde = complète
        rsync -av "$source/" "$backup_dir/"
    fi
    
    # Rotation
    case "$backup_type" in
        daily)
            # Garder 7 jours
            find "$dest_base" -name "daily_*" -type d -mtime +7 -exec rm -rf {} +
            ;;
        weekly)
            # Garder 4 semaines
            find "$dest_base" -name "weekly_*" -type d -mtime +28 -exec rm -rf {} +
            ;;
        monthly)
            # Garder 12 mois
            find "$dest_base" -name "monthly_*" -type d -mtime +365 -exec rm -rf {} +
            ;;
    esac
    
    echo "Sauvegarde $backup_type créée : $backup_dir"
}
```

## Restauration et récupération

### Restauration depuis tar

**Restauration tar** :
```bash
#!/bin/bash
# Restauration depuis tar

restore_from_tar() {
    local backup_file="$1"
    local dest="$2"
    
    # Vérifier le checksum
    if [ -f "${backup_file}.sha256" ]; then
        sha256sum -c "${backup_file}.sha256" || {
            echo "ERREUR: Checksum invalide"
            return 1
        }
    fi
    
    # Extraire
    mkdir -p "$dest"
    tar -xzf "$backup_file" -C "$dest"
    
    echo "Restauration terminée dans $dest"
}
```

### Restauration depuis rsync

**Restauration rsync** :
```bash
#!/bin/bash
# Restauration depuis rsync

restore_from_rsync() {
    local backup_dir="$1"
    local dest="$2"
    
    rsync -av "${backup_dir}/" "$dest/"
    
    echo "Restauration terminée"
}
```

### Restauration granulaire

**Restauration de fichiers spécifiques** :
```bash
#!/bin/bash
# Restauration granulaire

restore_specific_files() {
    local backup_file="$1"
    local files_to_restore=("$2")
    local dest="$3"
    
    # Extraire seulement les fichiers spécifiés
    for file in "${files_to_restore[@]}"; do
        tar -xzf "$backup_file" -C "$dest" "$file"
    done
    
    echo "Fichiers restaurés"
}
```

## Stockage et archivage

### Stockage local

**Organisation du stockage** :
```bash
#!/bin/bash
# Organisation du stockage

organize_backup_storage() {
    local backup_root="/backups"
    
    # Structure
    mkdir -p "$backup_root"/{daily,weekly,monthly,archive}
    
    # Quotas par type
    # daily : 7 jours
    # weekly : 4 semaines
    # monthly : 12 mois
    # archive : Long terme
}
```

### Stockage distant

**Sauvegarde distante** :
```bash
#!/bin/bash
# Sauvegarde distante

remote_backup() {
    local source="$1"
    local remote_host="$2"
    local remote_path="$3"
    
    # Via rsync/SSH
    rsync -avz -e ssh "$source/" "${remote_host}:${remote_path}/"
    
    # Via rclone (cloud)
    # rclone sync "$source/" remote:backup/
    
    echo "Sauvegarde distante terminée"
}
```

## Vérification d'intégrité

### Checksums et vérification

**Vérification d'intégrité** :
```bash
#!/bin/bash
# Vérification d'intégrité

verify_backup() {
    local backup_file="$1"
    
    # Vérifier le checksum
    if [ -f "${backup_file}.sha256" ]; then
        if sha256sum -c "${backup_file}.sha256"; then
            echo "✓ Checksum valide"
        else
            echo "✗ Checksum invalide"
            return 1
        fi
    fi
    
    # Vérifier l'archive tar
    if [[ "$backup_file" == *.tar.gz ]]; then
        if tar -tzf "$backup_file" > /dev/null 2>&1; then
            echo "✓ Archive valide"
        else
            echo "✗ Archive corrompue"
            return 1
        fi
    fi
    
    return 0
}

# Vérification automatique après backup
backup_with_verification() {
    local source="$1"
    local dest="$2"
    
    backup_with_tar "$source" "$dest"
    local backup_file=$(ls -t "$dest"/*.tar.gz | head -1)
    
    if verify_backup "$backup_file"; then
        echo "Sauvegarde vérifiée avec succès"
    else
        echo "ERREUR: Sauvegarde invalide"
        return 1
    fi
}
```

## Planification et conformité

### Plan de sauvegarde

**Plan de sauvegarde** :
```bash
#!/bin/bash
# Plan de sauvegarde

backup_schedule() {
    # Quotidien : Sauvegardes incrémentielles
    # Hebdomadaire : Sauvegarde complète
    # Mensuel : Archive complète
    # Annuel : Archive longue durée
    
    local day_of_week=$(date +%u)
    local day_of_month=$(date +%d)
    
    if [ "$day_of_month" = "01" ]; then
        # Sauvegarde mensuelle
        monthly_backup
    elif [ "$day_of_week" = "7" ]; then
        # Sauvegarde hebdomadaire (dimanche)
        weekly_backup
    else
        # Sauvegarde quotidienne
        daily_backup
    fi
}
```

## Scripts d'automatisation

### Gestionnaire de backup complet

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de backup complet

set -euo pipefail

backup_manager() {
    local action="$1"
    shift
    
    case "$action" in
        create)
            create_backup "$@"
            ;;
        restore)
            restore_backup "$@"
            ;;
        list)
            list_backups "$@"
            ;;
        verify)
            verify_backup "$@"
            ;;
        rotate)
            rotate_backups "$@"
            ;;
        *)
            echo "Usage: backup_manager {create|restore|list|verify|rotate}"
            exit 1
            ;;
    esac
}

create_backup() {
    local source="$1"
    local dest="${2:-/backups}"
    local type="${3:-daily}"
    
    incremental_backup_system "$source" "$dest" "$type"
}

restore_backup() {
    local backup_path="$1"
    local dest="$2"
    
    if [ -d "$backup_path" ]; then
        restore_from_rsync "$backup_path" "$dest"
    elif [[ "$backup_path" == *.tar.gz ]]; then
        restore_from_tar "$backup_path" "$dest"
    else
        echo "Type de backup non reconnu"
        exit 1
    fi
}

list_backups() {
    local backup_dir="${1:-/backups}"
    
    echo "=== Sauvegardes disponibles ==="
    find "$backup_dir" -type d -name "*backup_*" -o -name "*.tar.gz" | sort
}

rotate_backups() {
    local backup_dir="${1:-/backups}"
    rotate_backups "$backup_dir" 7 4 12
}

# Utilisation
# backup_manager create /home /backups daily
# backup_manager restore /backups/home/backup_20240115 /restore
# backup_manager list /backups
# backup_manager verify /backups/backup.tar.gz
```

## Conclusion

Les stratégies de sauvegarde et de snapshots sont essentielles pour protéger les données et assurer la continuité des opérations. En combinant différents types de sauvegardes (complètes, incrémentielles), des snapshots système rapides, et une vérification d'intégrité régulière, vous pouvez créer un système de protection des données robuste et fiable.

Un système bien sauvegardé utilise plusieurs couches : snapshots pour la récupération rapide, sauvegardes incrémentielles pour l'efficacité, et sauvegardes distantes pour la protection contre les sinistres. La compréhension approfondie de ces mécanismes permet de créer des stratégies de sauvegarde qui répondent aux besoins spécifiques de chaque environnement.

Dans le chapitre suivant, nous explorerons les scripts d'automatisation multi-serveur, découvrant comment orchestrer des opérations complexes sur plusieurs systèmes simultanément.
