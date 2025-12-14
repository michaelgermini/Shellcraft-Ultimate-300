# Chapitre 42 - Transfert de fichiers (scp, rsync)

## Table des matières
- [Introduction](#introduction)
- [SCP : transfert sécurisé simple](#scp--transfert-sécurisé-simple)
- [SFTP : protocole de transfert de fichiers](#sftp--protocole-de-transfert-de-fichiers)
- [Rsync : synchronisation avancée](#rsync--synchronisation-avancée)
- [Options avancées de rsync](#options-avancées-de-rsync)
- [Optimisation des transferts](#optimisation-des-transferts)
- [Transferts multi-hôtes](#transferts-multi-hôtes)
- [Sauvegardes avec rsync](#sauvegardes-avec-rsync)
- [Automation et scripting](#automation-et-scripting)
- [Sécurité et bonnes pratiques](#sécurité-et-bonnes-pratiques)
- [Alternatives modernes](#alternatives-modernes)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Le transfert de fichiers constitue une opération quotidienne dans l'administration système. Que ce soit pour déployer du code, sauvegarder des données, ou synchroniser des configurations, maîtriser les outils de transfert permet d'effectuer ces opérations de manière fiable, sécurisée, et efficace.

Imaginez le transfert de fichiers comme un service de messagerie express : SCP offre la livraison standard sécurisée, tandis que rsync agit comme un système de mise à jour intelligent qui n'envoie que les modifications, économisant temps et bande passante. Dans un environnement Linux moderne, ces outils sont essentiels pour maintenir la cohérence des données entre systèmes et assurer des sauvegardes efficaces.

## SCP : transfert sécurisé simple

### Utilisation de base

**Syntaxe SCP** :
```bash
scp [OPTIONS] SOURCE DESTINATION
```

**Exemples de base** :
```bash
#!/bin/bash
# Utilisation de base de SCP

# Copier un fichier vers le serveur distant
scp file.txt user@host:/path/to/destination/

# Copier depuis le serveur distant vers local
scp user@host:/path/to/file.txt /local/destination/

# Copier un répertoire récursivement
scp -r directory/ user@host:/path/to/destination/

# Copier plusieurs fichiers
scp file1.txt file2.txt file3.txt user@host:/destination/

# Avec port personnalisé
scp -P 2222 file.txt user@host:/destination/

# Afficher la progression
scp -v file.txt user@host:/destination/

# Mode silencieux
scp -q file.txt user@host:/destination/
```

### Options SCP

**Options principales** :
```bash
#!/bin/bash
# Options SCP

# -r : Récursif (répertoires)
scp -r /local/dir/ user@host:/remote/dir/

# -p : Préserver les permissions et timestamps
scp -p file.txt user@host:/destination/

# -v : Verbose (détails)
scp -v file.txt user@host:/destination/

# -q : Quiet (silencieux)
scp -q file.txt user@host:/destination/

# -C : Compression
scp -C large_file.tar.gz user@host:/destination/

# -i : Clé privée spécifique
scp -i ~/.ssh/id_rsa file.txt user@host:/destination/

# -l : Limiter la bande passante (KB/s)
scp -l 1000 file.txt user@host:/destination/

# -o : Options SSH
scp -o StrictHostKeyChecking=no file.txt user@host:/destination/
scp -o UserKnownHostsFile=/dev/null file.txt user@host:/destination/
```

**Exemples pratiques** :
```bash
#!/bin/bash
# Exemples pratiques SCP

# Copier un fichier avec préservation des attributs
scp -p backup.tar.gz alice@server.example.com:/home/alice/backups/

# Copier un répertoire avec compression
scp -rC /var/log/ user@host:/backup/logs/

# Copier depuis distant avec port non standard
scp -P 2222 user@host:/var/log/app.log ./logs/

# Copier avec limitation de bande passante
scp -l 500 large_file.iso user@host:/iso/

# Copier avec clé spécifique
scp -i ~/.ssh/deploy_key app.tar.gz deploy@server:/apps/
```

## SFTP : protocole de transfert de fichiers

### Mode interactif

**Utilisation interactive** :
```bash
#!/bin/bash
# Utilisation interactive SFTP

# Connexion SFTP
sftp user@host

# Commandes SFTP dans la session :
# ls                    # Lister fichiers distants
# lls                   # Lister fichiers locaux
# cd /remote/path       # Changer répertoire distant
# lcd /local/path       # Changer répertoire local
# pwd                   # Répertoire distant actuel
# lpwd                  # Répertoire local actuel
# put file.txt          # Envoyer fichier
# get file.txt          # Télécharger fichier
# put -r directory/     # Envoyer répertoire
# get -r directory/     # Télécharger répertoire
# mkdir newdir          # Créer répertoire distant
# rmdir olddir          # Supprimer répertoire distant
# rm file.txt           # Supprimer fichier distant
# rename old new        # Renommer fichier
# chmod 755 file.txt    # Changer permissions
# help                  # Aide
# quit                  # Quitter
```

### Mode non-interactif

**Scripts SFTP** :
```bash
#!/bin/bash
# SFTP non-interactif

# Télécharger un fichier
sftp user@host:/path/to/file.txt /local/destination/

# Envoyer un fichier avec script
sftp user@host << EOF
put /local/file.txt /remote/destination/
quit
EOF

# Script SFTP complet
sftp_batch_upload() {
    local host="$1"
    local user="$2"
    local local_dir="$3"
    local remote_dir="$4"
    
    sftp "$user@$host" << EOF
cd $remote_dir
put -r $local_dir/*
quit
EOF
}

# Téléchargement avec script
sftp_batch_download() {
    local host="$1"
    local user="$2"
    local remote_dir="$3"
    local local_dir="$4"
    
    sftp "$user@$host" << EOF
lcd $local_dir
cd $remote_dir
get -r *
quit
EOF
}
```

## Rsync : synchronisation avancée

### Utilisation de base

**Syntaxe rsync** :
```bash
rsync [OPTIONS] SOURCE DESTINATION
```

**Exemples de base** :
```bash
#!/bin/bash
# Utilisation de base rsync

# Synchronisation locale simple
rsync -av source/ destination/

# Synchronisation via SSH
rsync -avz -e ssh /local/dir/ user@host:/remote/dir/

# Synchronisation avec progression
rsync -avh --progress source/ destination/

# Mode dry-run (simulation)
rsync -av --dry-run source/ destination/

# Synchronisation avec suppression des fichiers obsolètes
rsync -av --delete source/ destination/
```

### Options essentielles

**Options principales** :
```bash
#!/bin/bash
# Options rsync essentielles

# -a : Archive (équivalent à -rlptgoD)
# -r : Récursif
# -l : Liens symboliques
# -p : Préserver permissions
# -t : Préserver timestamps
# -g : Préserver groupe
# -o : Préserver propriétaire
# -D : Préserver périphériques spéciaux

rsync -a source/ destination/

# -v : Verbose
rsync -av source/ destination/

# -z : Compression
rsync -avz source/ destination/

# -h : Format humain
rsync -avh source/ destination/

# --progress : Progression détaillée
rsync -avh --progress source/ destination/

# --delete : Supprimer fichiers obsolètes
rsync -av --delete source/ destination/

# --exclude : Exclure des patterns
rsync -av --exclude "*.tmp" source/ destination/

# --include : Inclure des patterns
rsync -av --include "*.txt" --exclude "*" source/ destination/
```

## Options avancées de rsync

### Filtres et exclusions

**Filtres avancés** :
```bash
#!/bin/bash
# Filtres rsync avancés

# Exclure plusieurs patterns
rsync -av --exclude "*.tmp" --exclude "*.log" --exclude ".git" source/ dest/

# Exclure depuis un fichier
rsync -av --exclude-from=exclude.txt source/ dest/

# Fichier exclude.txt :
# *.tmp
# *.log
# .git/
# node_modules/

# Inclure seulement certains fichiers
rsync -av --include "*.txt" --include "*.md" --exclude "*" source/ dest/

# Filtres complexes
rsync -av --include "*/" --include "*.txt" --exclude "*" source/ dest/

# Exclure par taille
rsync -av --max-size=100M source/ dest/
rsync -av --min-size=1M source/ dest/
```

### Synchronisation incrémentielle

**Sauvegardes incrémentielles** :
```bash
#!/bin/bash
# Sauvegardes incrémentielles avec rsync

# Utiliser hard links pour économiser l'espace
rsync -a --link-dest=/backup/previous /source /backup/current

# Script de sauvegarde incrémentielle
incremental_backup() {
    local source="$1"
    local backup_base="$2"
    local current_backup="$backup_base/$(date +%Y%m%d_%H%M%S)"
    local previous_backup=$(ls -td "$backup_base"/*/ 2>/dev/null | head -1)
    
    if [ -n "$previous_backup" ]; then
        rsync -a --link-dest="$previous_backup" "$source/" "$current_backup/"
    else
        rsync -a "$source/" "$current_backup/"
    fi
    
    echo "Sauvegarde créée : $current_backup"
}

# Utilisation
# incremental_backup /home/user /backup/user
```

### Synchronisation bidirectionnelle

**Synchronisation bidirectionnelle** :
```bash
#!/bin/bash
# Synchronisation bidirectionnelle (approximation)

# rsync est unidirectionnel, mais on peut créer un script
bidirectional_sync() {
    local dir1="$1"
    local dir2="$2"
    
    # Synchroniser dir1 -> dir2
    rsync -av --delete "$dir1/" "$dir2/"
    
    # Synchroniser dir2 -> dir1
    rsync -av --delete "$dir2/" "$dir1/"
}

# Note : Pour une vraie synchronisation bidirectionnelle,
# utiliser unison ou rclone
```

## Optimisation des transferts

### Compression et bande passante

**Optimisation** :
```bash
#!/bin/bash
# Optimisation des transferts

# Compression (utile pour fichiers texte)
rsync -avz source/ destination/

# Limiter la bande passante (KB/s)
rsync -av --bwlimit=1000 source/ destination/

# Transfert en parallèle
# Utiliser plusieurs connexions rsync simultanées
parallel_rsync() {
    local source="$1"
    local dest="$2"
    local num_jobs="${3:-4}"
    
    # Diviser en sous-répertoires et transférer en parallèle
    find "$source" -mindepth 1 -maxdepth 1 -type d | \
    xargs -P "$num_jobs" -I {} rsync -av {} "$dest/"
}

# Transfert avec checksums (plus lent mais plus sûr)
rsync -av --checksum source/ destination/

# Transfert sans vérification de taille/temps (plus rapide)
rsync -av --size-only source/ destination/
```

### Transferts partiels

**Reprendre un transfert interrompu** :
```bash
#!/bin/bash
# Reprendre un transfert

# rsync reprend automatiquement les transferts interrompus
# Les fichiers partiellement transférés sont détectés

# Forcer la reprise
rsync -av --partial source/ destination/

# Garder les fichiers partiels
rsync -av --partial --progress source/ destination/

# Supprimer les fichiers partiels après transfert complet
rsync -av --partial --partial-dir=.rsync-partial source/ destination/
```

## Transferts multi-hôtes

### Synchronisation vers plusieurs serveurs

**Transfert vers plusieurs destinations** :
```bash
#!/bin/bash
# Transfert vers plusieurs serveurs

sync_to_multiple_hosts() {
    local source="$1"
    shift
    local hosts=("$@")
    
    for host in "${hosts[@]}"; do
        echo "Synchronisation vers $host..."
        rsync -avz -e ssh "$source/" "$host:/backup/" &
    done
    
    # Attendre la fin de tous les transferts
    wait
    echo "Tous les transferts terminés"
}

# Utilisation
# sync_to_multiple_hosts /data server1:/backup server2:/backup server3:/backup
```

### Synchronisation depuis plusieurs sources

**Agrégation de plusieurs sources** :
```bash
#!/bin/bash
# Synchronisation depuis plusieurs sources

sync_from_multiple_sources() {
    local dest="$1"
    shift
    local sources=("$@")
    
    for source in "${sources[@]}"; do
        echo "Synchronisation depuis $source..."
        rsync -avz -e ssh "$source/" "$dest/$(basename $source)/"
    done
}
```

## Sauvegardes avec rsync

### Scripts de sauvegarde

**Sauvegarde complète** :
```bash
#!/bin/bash
# Script de sauvegarde avec rsync

backup_with_rsync() {
    local source="$1"
    local dest="$2"
    local exclude_file="${3:-}"
    
    local rsync_opts="-avh --progress --delete"
    
    if [ -n "$exclude_file" ] && [ -f "$exclude_file" ]; then
        rsync_opts="$rsync_opts --exclude-from=$exclude_file"
    fi
    
    rsync $rsync_opts "$source/" "$dest/"
    
    echo "Sauvegarde terminée : $source -> $dest"
}

# Utilisation
# backup_with_rsync /home/user /backup/user /path/to/exclude.txt
```

**Sauvegarde avec rotation** :
```bash
#!/bin/bash
# Sauvegarde avec rotation

backup_with_rotation() {
    local source="$1"
    local backup_base="$2"
    local keep_backups="${3:-7}"  # Garder 7 sauvegardes
    
    local current_backup="$backup_base/backup_$(date +%Y%m%d_%H%M%S)"
    local previous_backup=$(ls -td "$backup_base"/backup_* 2>/dev/null | head -1)
    
    # Créer la sauvegarde
    if [ -n "$previous_backup" ]; then
        rsync -av --link-dest="$previous_backup" "$source/" "$current_backup/"
    else
        rsync -av "$source/" "$current_backup/"
    fi
    
    # Supprimer les anciennes sauvegardes
    ls -td "$backup_base"/backup_* 2>/dev/null | tail -n +$((keep_backups + 1)) | xargs rm -rf
    
    echo "Sauvegarde créée : $current_backup"
}
```

## Automation et scripting

### Scripts d'automatisation

**Synchronisation automatique** :
```bash
#!/bin/bash
# Synchronisation automatique

set -euo pipefail

auto_sync() {
    local source="$1"
    local dest="$2"
    local interval="${3:-3600}"  # 1 heure par défaut
    
    while true; do
        echo "[$(date)] Synchronisation..."
        rsync -av --delete "$source/" "$dest/"
        echo "[$(date)] Synchronisation terminée"
        sleep "$interval"
    done
}

# Utilisation
# auto_sync /local/data user@remote:/backup/data 3600
```

**Synchronisation avec monitoring** :
```bash
#!/bin/bash
# Synchronisation avec monitoring

sync_with_monitoring() {
    local source="$1"
    local dest="$2"
    local log_file="${3:-/var/log/rsync.log}"
    
    {
        echo "=== Synchronisation $(date) ==="
        rsync -avh --progress --delete "$source/" "$dest/" 2>&1
        echo "=== Terminé $(date) ==="
        echo ""
    } >> "$log_file"
    
    # Envoyer un email en cas d'erreur
    if [ $? -ne 0 ]; then
        echo "Erreur lors de la synchronisation" | mail -s "Erreur rsync" admin@example.com
    fi
}
```

## Sécurité et bonnes pratiques

### Bonnes pratiques

**Recommandations** :
```bash
#!/bin/bash
# Bonnes pratiques rsync/scp

# 1. Toujours utiliser SSH pour les transferts distants
rsync -avz -e ssh source/ user@host:/dest/

# 2. Utiliser des clés SSH au lieu de mots de passe
scp -i ~/.ssh/id_rsa file.txt user@host:/dest/

# 3. Vérifier les checksums pour les fichiers critiques
rsync -av --checksum source/ dest/

# 4. Utiliser --dry-run avant les opérations destructives
rsync -av --delete --dry-run source/ dest/

# 5. Logger les opérations importantes
rsync -av source/ dest/ | tee sync.log

# 6. Limiter la bande passante pour ne pas saturer le réseau
rsync -av --bwlimit=1000 source/ dest/

# 7. Exclure les fichiers temporaires et caches
rsync -av --exclude "*.tmp" --exclude ".cache" source/ dest/
```

### Sécurité SSH

**Configuration SSH pour transferts** :
```bash
#!/bin/bash
# Configuration SSH sécurisée

# Utiliser des options SSH sécurisées
rsync -avz -e "ssh -o StrictHostKeyChecking=yes -o UserKnownHostsFile=~/.ssh/known_hosts" \
    source/ user@host:/dest/

# Désactiver la compression si déjà compressé
rsync -av -e "ssh -o Compression=no" source/ user@host:/dest/

# Utiliser un port non standard
rsync -avz -e "ssh -p 2222" source/ user@host:/dest/
```

## Alternatives modernes

### rclone

**Introduction à rclone** :
```bash
#!/bin/bash
# rclone : rsync pour cloud storage

# Installation
# sudo apt install rclone  # Debian/Ubuntu
# brew install rclone       # macOS

# Configuration
rclone config

# Synchronisation
rclone sync /local/dir remote:backup/dir

# Copie
rclone copy /local/file remote:backup/

# Liste
rclone ls remote:backup/
```

### borgbackup

**BorgBackup** :
```bash
#!/bin/bash
# BorgBackup : sauvegarde dédupliquée

# Créer un repository
borg init /backup/repo

# Créer une sauvegarde
borg create /backup/repo::backup-$(date +%Y%m%d) /source

# Lister les sauvegardes
borg list /backup/repo

# Restaurer
borg extract /backup/repo::backup-20240115
```

## Scripts d'automatisation

### Gestionnaire de transfert

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de transfert de fichiers

set -euo pipefail

transfer_manager() {
    local action="$1"
    shift
    
    case "$action" in
        scp)
            scp_transfer "$@"
            ;;
        rsync)
            rsync_transfer "$@"
            ;;
        sftp)
            sftp_transfer "$@"
            ;;
        backup)
            create_backup "$@"
            ;;
        sync)
            sync_directories "$@"
            ;;
        *)
            echo "Usage: transfer_manager {scp|rsync|sftp|backup|sync}"
            exit 1
            ;;
    esac
}

scp_transfer() {
    local source="$1"
    local dest="$2"
    local options="${3:--p}"
    
    scp $options "$source" "$dest"
    echo "Transfert SCP terminé : $source -> $dest"
}

rsync_transfer() {
    local source="$1"
    local dest="$2"
    local options="${3:--avz}"
    
    rsync $options "$source" "$dest"
    echo "Transfert rsync terminé : $source -> $dest"
}

sftp_transfer() {
    local host="$1"
    local user="$2"
    local local_file="$3"
    local remote_file="$4"
    
    sftp "$user@$host" << EOF
put $local_file $remote_file
quit
EOF
    
    echo "Transfert SFTP terminé"
}

create_backup() {
    local source="$1"
    local dest="$2"
    local backup_name="backup_$(date +%Y%m%d_%H%M%S)"
    
    rsync -av --link-dest="$dest/latest" "$source/" "$dest/$backup_name/"
    rm -f "$dest/latest"
    ln -s "$backup_name" "$dest/latest"
    
    echo "Sauvegarde créée : $dest/$backup_name"
}

sync_directories() {
    local dir1="$1"
    local dir2="$2"
    
    # Synchroniser dir1 -> dir2
    rsync -av --delete "$dir1/" "$dir2/"
    
    # Synchroniser dir2 -> dir1
    rsync -av --delete "$dir2/" "$dir1/"
    
    echo "Synchronisation terminée"
}

# Utilisation
# transfer_manager scp file.txt user@host:/dest/
# transfer_manager rsync /source/ user@host:/dest/
# transfer_manager backup /data /backup
```

## Conclusion

Le transfert de fichiers avec SCP et rsync est essentiel pour l'administration système moderne. SCP offre une solution simple et sécurisée pour les transferts ponctuels, tandis que rsync permet des synchronisations efficaces et des sauvegardes incrémentielles sophistiquées.

Un système bien administré utilise ces outils de manière appropriée : SCP pour les transferts simples, rsync pour les synchronisations régulières et les sauvegardes, et des scripts automatisés pour maintenir la cohérence des données. La compréhension approfondie de ces outils permet de créer des solutions de transfert robustes, efficaces et sécurisées.

Dans le chapitre suivant, nous explorerons les tunnels SSH, découvrant comment créer des connexions sécurisées pour le port forwarding, les proxies SOCKS, et les VPN basés sur SSH.
