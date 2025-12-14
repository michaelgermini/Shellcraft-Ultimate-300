# Chapitre 20 - Projet pratique : votre premier script

## Table des matières
- [Introduction](#introduction)
- [Objectif du projet](#objectif-du-projet)
- [Analyse des besoins](#analyse-des-besoins)
- [Conception du script](#conception-du-script)
- [Implémentation étape par étape](#implémentation-étape-par-étape)
- [Améliorations et bonnes pratiques](#améliorations-et-bonnes-pratiques)
- [Tests et validation](#tests-et-validation)
- [Documentation et maintenance](#documentation-et-maintenance)
- [Extensions possibles](#extensions-possibles)
- [Conclusion](#conclusion)

## Introduction

Après avoir exploré les concepts fondamentaux du terminal et du shell, il est temps de mettre en pratique toutes ces connaissances dans un projet concret. Ce chapitre vous guide à travers la création complète d'un script fonctionnel, depuis la conception initiale jusqu'à la version finale robuste et documentée.

Ce projet pratique synthétise tous les concepts appris : variables, redirections, pipelines, gestion d'erreurs, et automatisation. En suivant ce guide, vous créerez non seulement un script utile, mais vous développerez également les compétences essentielles pour créer vos propres outils d'automatisation.

## Objectif du projet

### Script de sauvegarde automatique

**Fonctionnalité principale** : Créer un script qui sauvegarde automatiquement un répertoire spécifié avec horodatage, compression, et rotation des anciennes sauvegardes.

**Caractéristiques** :
- Sauvegarde d'un répertoire source vers un répertoire de destination
- Nom de fichier avec horodatage
- Compression automatique (tar.gz)
- Rotation des anciennes sauvegardes (garder seulement les N dernières)
- Logging des opérations
- Gestion d'erreurs robuste
- Configuration via variables ou fichier de config

### Compétences développées

**Concepts appliqués** :
- Variables et environnement
- Redirections et pipelines
- Gestion d'erreurs
- Fonctions et modularité
- Arguments de ligne de commande
- Tests conditionnels
- Boucles et itérations
- Logging et reporting

## Analyse des besoins

### Fonctionnalités requises

**Fonctionnalités essentielles** :
1. Accepter un répertoire source en argument
2. Créer une archive compressée avec horodatage
3. Sauvegarder dans un répertoire de destination
4. Logger les opérations
5. Gérer les erreurs gracieusement

**Fonctionnalités avancées** :
1. Rotation automatique des sauvegardes
2. Configuration via fichier
3. Options de ligne de commande
4. Vérification de l'espace disque
5. Notifications (optionnel)

### Structure du script

**Organisation proposée** :
```bash
#!/bin/bash
# 1. En-tête et configuration
# 2. Définition des fonctions
# 3. Parsing des arguments
# 4. Validation et vérifications
# 5. Exécution principale
# 6. Nettoyage et reporting
```

## Conception du script

### Architecture modulaire

**Fonctions principales** :
- `usage()` : Afficher l'aide
- `log_message()` : Logger les messages
- `check_dependencies()` : Vérifier les dépendances
- `check_space()` : Vérifier l'espace disque
- `create_backup()` : Créer la sauvegarde
- `rotate_backups()` : Rotation des anciennes sauvegardes
- `cleanup()` : Nettoyage en cas d'erreur
- `main()` : Fonction principale

### Flux d'exécution

**Étapes principales** :
1. Initialisation (configuration, logging)
2. Validation (arguments, dépendances, espace)
3. Création de la sauvegarde
4. Rotation des anciennes sauvegardes
5. Rapport final

## Implémentation étape par étape

### Version 1 : Script de base

**Première version fonctionnelle** :
```bash
#!/bin/bash
# backup.sh - Script de sauvegarde simple
# Version 1.0

# Configuration
SOURCE_DIR="$1"
BACKUP_DIR="${2:-$HOME/backups}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}.tar.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

# Vérifications de base
if [ -z "$SOURCE_DIR" ]; then
    echo "Usage: $0 <source_directory> [backup_directory]"
    exit 1
fi

if [ ! -d "$SOURCE_DIR" ]; then
    echo "Erreur: Le répertoire source '$SOURCE_DIR' n'existe pas"
    exit 1
fi

# Créer le répertoire de sauvegarde si nécessaire
mkdir -p "$BACKUP_DIR"

# Créer la sauvegarde
echo "Création de la sauvegarde..."
tar -czf "$BACKUP_PATH" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if [ $? -eq 0 ]; then
    echo "Sauvegarde créée: $BACKUP_PATH"
    ls -lh "$BACKUP_PATH"
else
    echo "Erreur lors de la création de la sauvegarde"
    exit 1
fi
```

**Test de la version 1** :
```bash
# Rendre exécutable
chmod +x backup.sh

# Test
./backup.sh /home/user/documents
./backup.sh /home/user/documents /mnt/backup
```

### Version 2 : Ajout du logging

**Amélioration avec logging** :
```bash
#!/bin/bash
# backup.sh - Script de sauvegarde avec logging
# Version 2.0

set -euo pipefail

# Configuration
SOURCE_DIR="${1:-}"
BACKUP_DIR="${2:-$HOME/backups}"
LOG_FILE="${BACKUP_DIR}/backup.log"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}.tar.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

# Fonction de logging
log_message() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" | tee -a "$LOG_FILE"
}

# Vérifications
if [ -z "$SOURCE_DIR" ]; then
    echo "Usage: $0 <source_directory> [backup_directory]"
    exit 1
fi

if [ ! -d "$SOURCE_DIR" ]; then
    log_message "ERROR" "Le répertoire source '$SOURCE_DIR' n'existe pas"
    exit 1
fi

# Créer le répertoire de sauvegarde si nécessaire
mkdir -p "$BACKUP_DIR"

# Créer la sauvegarde
log_message "INFO" "Début de la sauvegarde de '$SOURCE_DIR'"
log_message "INFO" "Destination: $BACKUP_PATH"

tar -czf "$BACKUP_PATH" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_PATH" | cut -f1)
    log_message "SUCCESS" "Sauvegarde créée: $BACKUP_PATH (Taille: $SIZE)"
    exit 0
else
    log_message "ERROR" "Échec de la création de la sauvegarde"
    exit 1
fi
```

### Version 3 : Gestion d'erreurs robuste

**Version avec gestion d'erreurs complète** :
```bash
#!/bin/bash
# backup.sh - Script de sauvegarde robuste
# Version 3.0

set -euo pipefail

# Configuration
SOURCE_DIR="${1:-}"
BACKUP_DIR="${2:-$HOME/backups}"
LOG_FILE="${BACKUP_DIR}/backup.log"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}.tar.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

# Variables de statut
BACKUP_SUCCESS=false
CLEANUP_NEEDED=false

# Fonction de logging
log_message() {
    local level="$1"
    shift
    local message="$*"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" | tee -a "$LOG_FILE"
}

# Fonction de nettoyage
cleanup() {
    if [ "$CLEANUP_NEEDED" = true ] && [ -f "$BACKUP_PATH" ]; then
        log_message "INFO" "Nettoyage du fichier de sauvegarde incomplet"
        rm -f "$BACKUP_PATH"
    fi
}

# Trap pour le nettoyage
trap cleanup EXIT ERR

# Fonction d'aide
usage() {
    cat << EOF
Usage: $0 <source_directory> [backup_directory] [options]

Options:
    -h, --help          Afficher cette aide
    -k, --keep N        Garder seulement les N dernières sauvegardes
    -v, --verbose       Mode verbeux
    -q, --quiet         Mode silencieux (seulement les erreurs)

Exemples:
    $0 /home/user/documents
    $0 /home/user/documents /mnt/backup
    $0 /home/user/documents /mnt/backup --keep 5
EOF
}

# Vérification des dépendances
check_dependencies() {
    local missing=()
    
    for cmd in tar gzip; do
        if ! command -v "$cmd" >/dev/null 2>&1; then
            missing+=("$cmd")
        fi
    done
    
    if [ ${#missing[@]} -gt 0 ]; then
        log_message "ERROR" "Dépendances manquantes: ${missing[*]}"
        exit 1
    fi
}

# Vérification de l'espace disque
check_space() {
    local source_size
    local available_space
    
    source_size=$(du -s "$SOURCE_DIR" 2>/dev/null | cut -f1)
    available_space=$(df "$BACKUP_DIR" | tail -1 | awk '{print $4}')
    
    # Convertir en KB (approximatif)
    if [ "$source_size" -gt "$available_space" ]; then
        log_message "ERROR" "Espace disque insuffisant"
        log_message "ERROR" "Nécessaire: $(du -h "$SOURCE_DIR" | cut -f1), Disponible: $(df -h "$BACKUP_DIR" | tail -1 | awk '{print $4}')"
        exit 1
    fi
}

# Vérifications initiales
if [ -z "$SOURCE_DIR" ] || [ "$SOURCE_DIR" = "-h" ] || [ "$SOURCE_DIR" = "--help" ]; then
    usage
    exit 0
fi

if [ ! -d "$SOURCE_DIR" ]; then
    log_message "ERROR" "Le répertoire source '$SOURCE_DIR' n'existe pas"
    exit 1
fi

# Créer le répertoire de sauvegarde si nécessaire
mkdir -p "$BACKUP_DIR"

# Vérifications
check_dependencies
check_space

# Créer la sauvegarde
log_message "INFO" "Début de la sauvegarde de '$SOURCE_DIR'"
log_message "INFO" "Destination: $BACKUP_PATH"

CLEANUP_NEEDED=true
tar -czf "$BACKUP_PATH" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"
CLEANUP_NEEDED=false

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_PATH" | cut -f1)
    log_message "SUCCESS" "Sauvegarde créée: $BACKUP_PATH (Taille: $SIZE)"
    BACKUP_SUCCESS=true
else
    log_message "ERROR" "Échec de la création de la sauvegarde"
    exit 1
fi
```

### Version 4 : Rotation des sauvegardes

**Ajout de la rotation automatique** :
```bash
#!/bin/bash
# backup.sh - Script de sauvegarde avec rotation
# Version 4.0

set -euo pipefail

# Configuration par défaut
SOURCE_DIR="${1:-}"
BACKUP_DIR="${2:-$HOME/backups}"
KEEP_BACKUPS="${KEEP_BACKUPS:-5}"
LOG_FILE="${BACKUP_DIR}/backup.log"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}.tar.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

# Variables de statut
BACKUP_SUCCESS=false
CLEANUP_NEEDED=false

# Fonction de logging
log_message() {
    local level="$1"
    shift
    local message="$*"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message" | tee -a "$LOG_FILE"
}

# Fonction de nettoyage
cleanup() {
    if [ "$CLEANUP_NEEDED" = true ] && [ -f "$BACKUP_PATH" ]; then
        log_message "INFO" "Nettoyage du fichier de sauvegarde incomplet"
        rm -f "$BACKUP_PATH"
    fi
}

trap cleanup EXIT ERR

# Fonction d'aide
usage() {
    cat << EOF
Usage: $0 <source_directory> [backup_directory] [options]

Options:
    -h, --help          Afficher cette aide
    -k, --keep N        Garder seulement les N dernières sauvegardes (défaut: 5)
    -v, --verbose       Mode verbeux
    -q, --quiet         Mode silencieux

Exemples:
    $0 /home/user/documents
    $0 /home/user/documents /mnt/backup --keep 10
EOF
}

# Rotation des sauvegardes
rotate_backups() {
    local keep_count="$1"
    local backup_pattern="${BACKUP_DIR}/backup_*.tar.gz"
    
    # Compter les sauvegardes existantes
    local backup_count
    backup_count=$(ls -1 $backup_pattern 2>/dev/null | wc -l)
    
    if [ "$backup_count" -le "$keep_count" ]; then
        log_message "INFO" "Nombre de sauvegardes ($backup_count) <= limite ($keep_count), pas de rotation nécessaire"
        return 0
    fi
    
    # Trier par date (plus ancien en premier) et supprimer les excédentaires
    local to_remove
    to_remove=$((backup_count - keep_count))
    
    log_message "INFO" "Rotation: suppression de $to_remove ancienne(s) sauvegarde(s)"
    
    ls -1t $backup_pattern 2>/dev/null | tail -n "$to_remove" | while read -r old_backup; do
        log_message "INFO" "Suppression de l'ancienne sauvegarde: $(basename "$old_backup")"
        rm -f "$old_backup"
    done
}

# Vérification des dépendances
check_dependencies() {
    local missing=()
    
    for cmd in tar gzip; do
        if ! command -v "$cmd" >/dev/null 2>&1; then
            missing+=("$cmd")
        fi
    done
    
    if [ ${#missing[@]} -gt 0 ]; then
        log_message "ERROR" "Dépendances manquantes: ${missing[*]}"
        exit 1
    fi
}

# Vérification de l'espace disque
check_space() {
    local source_size
    local available_space
    
    source_size=$(du -s "$SOURCE_DIR" 2>/dev/null | cut -f1)
    available_space=$(df "$BACKUP_DIR" | tail -1 | awk '{print $4}')
    
    if [ "$source_size" -gt "$available_space" ]; then
        log_message "ERROR" "Espace disque insuffisant"
        exit 1
    fi
}

# Parsing des arguments
parse_arguments() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help)
                usage
                exit 0
                ;;
            -k|--keep)
                KEEP_BACKUPS="$2"
                shift 2
                ;;
            -v|--verbose)
                set -x
                shift
                ;;
            -q|--quiet)
                exec 1>/dev/null
                shift
                ;;
            *)
                if [ -z "$SOURCE_DIR" ]; then
                    SOURCE_DIR="$1"
                elif [ "$BACKUP_DIR" = "$HOME/backups" ]; then
                    BACKUP_DIR="$1"
                fi
                shift
                ;;
        esac
    done
}

# Vérifications initiales
parse_arguments "$@"

if [ -z "$SOURCE_DIR" ]; then
    usage
    exit 1
fi

if [ ! -d "$SOURCE_DIR" ]; then
    log_message "ERROR" "Le répertoire source '$SOURCE_DIR' n'existe pas"
    exit 1
fi

# Créer le répertoire de sauvegarde si nécessaire
mkdir -p "$BACKUP_DIR"

# Vérifications
check_dependencies
check_space

# Créer la sauvegarde
log_message "INFO" "Début de la sauvegarde de '$SOURCE_DIR'"
log_message "INFO" "Destination: $BACKUP_PATH"

CLEANUP_NEEDED=true
tar -czf "$BACKUP_PATH" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")"
CLEANUP_NEEDED=false

if [ $? -eq 0 ]; then
    SIZE=$(du -h "$BACKUP_PATH" | cut -f1)
    log_message "SUCCESS" "Sauvegarde créée: $BACKUP_PATH (Taille: $SIZE)"
    BACKUP_SUCCESS=true
    
    # Rotation des sauvegardes
    rotate_backups "$KEEP_BACKUPS"
else
    log_message "ERROR" "Échec de la création de la sauvegarde"
    exit 1
fi
```

## Améliorations et bonnes pratiques

### Version finale : Script complet et robuste

**Script final avec toutes les améliorations** :
```bash
#!/bin/bash
# backup.sh - Script de sauvegarde automatique robuste
# Version 5.0
# Auteur: Votre Nom
# Date: $(date +%Y-%m-%d)

set -euo pipefail

# ============================================================================
# CONFIGURATION
# ============================================================================

SCRIPT_NAME=$(basename "$0")
SCRIPT_VERSION="5.0"
SOURCE_DIR="${1:-}"
BACKUP_DIR="${2:-$HOME/backups}"
KEEP_BACKUPS="${KEEP_BACKUPS:-5}"
LOG_FILE="${BACKUP_DIR}/backup.log"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_NAME="backup_${TIMESTAMP}.tar.gz"
BACKUP_PATH="${BACKUP_DIR}/${BACKUP_NAME}"

# Options
VERBOSE=false
QUIET=false

# ============================================================================
# FONCTIONS
# ============================================================================

# Fonction de logging
log_message() {
    local level="$1"
    shift
    local message="$*"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    if [ "$QUIET" = false ] || [ "$level" = "ERROR" ]; then
        echo "[$timestamp] [$level] $message" | tee -a "$LOG_FILE"
    else
        echo "[$timestamp] [$level] $message" >> "$LOG_FILE"
    fi
}

# Fonction de nettoyage
cleanup() {
    if [ "${CLEANUP_NEEDED:-false}" = true ] && [ -f "$BACKUP_PATH" ]; then
        log_message "INFO" "Nettoyage du fichier de sauvegarde incomplet"
        rm -f "$BACKUP_PATH"
    fi
}

# Fonction d'aide
usage() {
    cat << EOF
$SCRIPT_NAME v$SCRIPT_VERSION - Script de sauvegarde automatique

Usage: $SCRIPT_NAME <source_directory> [backup_directory] [options]

Arguments:
    source_directory      Répertoire à sauvegarder (requis)
    backup_directory       Répertoire de destination (défaut: ~/backups)

Options:
    -h, --help            Afficher cette aide
    -k, --keep N          Garder seulement les N dernières sauvegardes (défaut: 5)
    -v, --verbose         Mode verbeux (afficher les commandes exécutées)
    -q, --quiet           Mode silencieux (seulement les erreurs dans la sortie)
    --version              Afficher la version

Exemples:
    $SCRIPT_NAME /home/user/documents
    $SCRIPT_NAME /home/user/documents /mnt/backup
    $SCRIPT_NAME /home/user/documents /mnt/backup --keep 10
    $SCRIPT_NAME /home/user/documents --verbose

Variables d'environnement:
    KEEP_BACKUPS          Nombre de sauvegardes à conserver (défaut: 5)

EOF
}

# Vérification des dépendances
check_dependencies() {
    local missing=()
    
    for cmd in tar gzip; do
        if ! command -v "$cmd" >/dev/null 2>&1; then
            missing+=("$cmd")
        fi
    done
    
    if [ ${#missing[@]} -gt 0 ]; then
        log_message "ERROR" "Dépendances manquantes: ${missing[*]}"
        log_message "ERROR" "Installation: sudo apt install tar gzip"
        return 1
    fi
    
    return 0
}

# Vérification de l'espace disque
check_space() {
    local source_size
    local available_space
    
    if ! source_size=$(du -s "$SOURCE_DIR" 2>/dev/null | cut -f1); then
        log_message "ERROR" "Impossible de déterminer la taille du répertoire source"
        return 1
    fi
    
    if ! available_space=$(df "$BACKUP_DIR" 2>/dev/null | tail -1 | awk '{print $4}'); then
        log_message "ERROR" "Impossible de déterminer l'espace disponible"
        return 1
    fi
    
    # Marge de sécurité (10%)
    local required_space=$((source_size + (source_size / 10)))
    
    if [ "$required_space" -gt "$available_space" ]; then
        local source_human=$(du -h "$SOURCE_DIR" | cut -f1)
        local available_human=$(df -h "$BACKUP_DIR" | tail -1 | awk '{print $4}')
        log_message "ERROR" "Espace disque insuffisant"
        log_message "ERROR" "Nécessaire: ~$source_human, Disponible: $available_human"
        return 1
    fi
    
    return 0
}

# Rotation des sauvegardes
rotate_backups() {
    local keep_count="$1"
    local backup_pattern="${BACKUP_DIR}/backup_*.tar.gz"
    
    # Vérifier s'il y a des sauvegardes
    if ! ls $backup_pattern 1>/dev/null 2>&1; then
        log_message "INFO" "Aucune sauvegarde existante à faire tourner"
        return 0
    fi
    
    # Compter les sauvegardes existantes
    local backup_count
    backup_count=$(ls -1 $backup_pattern 2>/dev/null | wc -l)
    
    if [ "$backup_count" -le "$keep_count" ]; then
        log_message "INFO" "Nombre de sauvegardes ($backup_count) <= limite ($keep_count), pas de rotation nécessaire"
        return 0
    fi
    
    # Trier par date (plus ancien en premier) et supprimer les excédentaires
    local to_remove
    to_remove=$((backup_count - keep_count))
    
    log_message "INFO" "Rotation: suppression de $to_remove ancienne(s) sauvegarde(s)"
    
    ls -1t $backup_pattern 2>/dev/null | tail -n "$to_remove" | while read -r old_backup; do
        local old_size=$(du -h "$old_backup" | cut -f1)
        log_message "INFO" "Suppression: $(basename "$old_backup") (Taille: $old_size)"
        rm -f "$old_backup"
    done
    
    log_message "SUCCESS" "Rotation terminée: $keep_count sauvegarde(s) conservée(s)"
}

# Création de la sauvegarde
create_backup() {
    local source="$1"
    local dest="$2"
    
    log_message "INFO" "Début de la sauvegarde de '$source'"
    log_message "INFO" "Destination: $dest"
    
    # Afficher la taille source
    local source_size=$(du -sh "$source" | cut -f1)
    log_message "INFO" "Taille du répertoire source: $source_size"
    
    # Créer la sauvegarde
    CLEANUP_NEEDED=true
    if tar -czf "$dest" -C "$(dirname "$source")" "$(basename "$source")" 2>&1 | tee -a "$LOG_FILE"; then
        CLEANUP_NEEDED=false
        
        local backup_size=$(du -h "$dest" | cut -f1)
        log_message "SUCCESS" "Sauvegarde créée avec succès"
        log_message "INFO" "Fichier: $dest"
        log_message "INFO" "Taille: $backup_size"
        
        return 0
    else
        log_message "ERROR" "Échec de la création de la sauvegarde"
        return 1
    fi
}

# Parsing des arguments
parse_arguments() {
    while [[ $# -gt 0 ]]; do
        case $1 in
            -h|--help)
                usage
                exit 0
                ;;
            --version)
                echo "$SCRIPT_NAME version $SCRIPT_VERSION"
                exit 0
                ;;
            -k|--keep)
                if ! [[ "$2" =~ ^[0-9]+$ ]] || [ "$2" -le 0 ]; then
                    log_message "ERROR" "Le nombre de sauvegardes à conserver doit être un entier positif"
                    exit 1
                fi
                KEEP_BACKUPS="$2"
                shift 2
                ;;
            -v|--verbose)
                VERBOSE=true
                set -x
                shift
                ;;
            -q|--quiet)
                QUIET=true
                shift
                ;;
            -*)
                log_message "ERROR" "Option inconnue: $1"
                usage
                exit 1
                ;;
            *)
                if [ -z "$SOURCE_DIR" ]; then
                    SOURCE_DIR="$1"
                elif [ "$BACKUP_DIR" = "$HOME/backups" ]; then
                    BACKUP_DIR="$1"
                else
                    log_message "ERROR" "Trop d'arguments: $1"
                    usage
                    exit 1
                fi
                shift
                ;;
        esac
    done
}

# ============================================================================
# FONCTION PRINCIPALE
# ============================================================================

main() {
    # Parsing des arguments
    parse_arguments "$@"
    
    # Vérifications initiales
    if [ -z "$SOURCE_DIR" ]; then
        log_message "ERROR" "Répertoire source non spécifié"
        usage
        exit 1
    fi
    
    if [ ! -d "$SOURCE_DIR" ]; then
        log_message "ERROR" "Le répertoire source '$SOURCE_DIR' n'existe pas"
        exit 1
    fi
    
    if [ ! -r "$SOURCE_DIR" ]; then
        log_message "ERROR" "Pas de permission de lecture sur '$SOURCE_DIR'"
        exit 1
    fi
    
    # Créer le répertoire de sauvegarde si nécessaire
    if ! mkdir -p "$BACKUP_DIR" 2>/dev/null; then
        log_message "ERROR" "Impossible de créer le répertoire de sauvegarde '$BACKUP_DIR'"
        exit 1
    fi
    
    if [ ! -w "$BACKUP_DIR" ]; then
        log_message "ERROR" "Pas de permission d'écriture sur '$BACKUP_DIR'"
        exit 1
    fi
    
    # Configuration du trap
    trap cleanup EXIT ERR
    
    # Vérifications
    if ! check_dependencies; then
        exit 1
    fi
    
    if ! check_space; then
        exit 1
    fi
    
    # Créer la sauvegarde
    if ! create_backup "$SOURCE_DIR" "$BACKUP_PATH"; then
        exit 1
    fi
    
    # Rotation des sauvegardes
    rotate_backups "$KEEP_BACKUPS"
    
    # Rapport final
    log_message "SUCCESS" "Sauvegarde terminée avec succès"
    
    exit 0
}

# Exécution
main "$@"
```

## Tests et validation

### Tests de base

**Tests fonctionnels** :
```bash
#!/bin/bash
# test_backup.sh - Tests pour backup.sh

TEST_DIR="/tmp/test_backup"
SOURCE_DIR="$TEST_DIR/source"
BACKUP_DIR="$TEST_DIR/backups"

# Setup
mkdir -p "$SOURCE_DIR"
echo "test file" > "$SOURCE_DIR/test.txt"

# Test 1: Sauvegarde de base
echo "Test 1: Sauvegarde de base"
./backup.sh "$SOURCE_DIR" "$BACKUP_DIR"
if [ $? -eq 0 ] && [ -f "$BACKUP_DIR/backup_"*.tar.gz ]; then
    echo "✓ Test 1 réussi"
else
    echo "✗ Test 1 échoué"
fi

# Test 2: Rotation
echo "Test 2: Rotation des sauvegardes"
for i in {1..7}; do
    ./backup.sh "$SOURCE_DIR" "$BACKUP_DIR" --keep 5
done
backup_count=$(ls -1 "$BACKUP_DIR"/backup_*.tar.gz 2>/dev/null | wc -l)
if [ "$backup_count" -eq 5 ]; then
    echo "✓ Test 2 réussi"
else
    echo "✗ Test 2 échoué (attendu: 5, obtenu: $backup_count)"
fi

# Test 3: Gestion d'erreur
echo "Test 3: Gestion d'erreur"
./backup.sh "/nonexistent/dir" "$BACKUP_DIR" 2>/dev/null
if [ $? -ne 0 ]; then
    echo "✓ Test 3 réussi"
else
    echo "✗ Test 3 échoué"
fi

# Nettoyage
rm -rf "$TEST_DIR"
```

### Tests avancés

**Tests de performance** :
```bash
# Test avec un grand répertoire
mkdir -p /tmp/large_test
# Créer des fichiers de test
for i in {1..1000}; do
    dd if=/dev/urandom of="/tmp/large_test/file_$i" bs=1M count=1 2>/dev/null
done

time ./backup.sh /tmp/large_test /tmp/backup_test
```

## Documentation et maintenance

### Documentation du script

**En-tête de documentation** :
```bash
#!/bin/bash
# ============================================================================
# backup.sh - Script de sauvegarde automatique robuste
# ============================================================================
#
# DESCRIPTION:
#   Ce script crée une sauvegarde compressée d'un répertoire source avec
#   horodatage, rotation automatique des anciennes sauvegardes, et logging
#   complet des opérations.
#
# AUTEUR:
#   Votre Nom <votre.email@example.com>
#
# VERSION:
#   5.0
#
# DATE:
#   2024-01-15
#
# LICENCE:
#   MIT License - Utilisation libre pour usage personnel et commercial
#
# DEPENDANCES:
#   - tar
#   - gzip
#   - bash 4.0+
#
# EXEMPLES:
#   # Sauvegarde simple
#   ./backup.sh /home/user/documents
#
#   # Sauvegarde avec destination personnalisée
#   ./backup.sh /home/user/documents /mnt/backup
#
#   # Sauvegarde avec rotation (garder 10 sauvegardes)
#   ./backup.sh /home/user/documents /mnt/backup --keep 10
#
# VARIABLES D'ENVIRONNEMENT:
#   KEEP_BACKUPS    Nombre de sauvegardes à conserver (défaut: 5)
#
# CODES DE RETOUR:
#   0   Succès
#   1   Erreur (voir les logs pour plus de détails)
#
# FICHIERS:
#   $BACKUP_DIR/backup.log    Fichier de log des opérations
#   $BACKUP_DIR/backup_*.tar.gz   Archives de sauvegarde
#
# ============================================================================
```

### Maintenance

**Changelog** :
```markdown
# Changelog

## [5.0] - 2024-01-15
### Added
- Rotation automatique des sauvegardes
- Vérification de l'espace disque
- Gestion d'erreurs robuste avec cleanup
- Options de ligne de commande complètes
- Mode verbose et quiet
- Documentation complète

### Changed
- Amélioration du logging
- Meilleure gestion des erreurs

### Fixed
- Correction de bugs de rotation
- Amélioration de la portabilité

## [4.0] - 2024-01-10
### Added
- Rotation des sauvegardes

## [3.0] - 2024-01-05
### Added
- Gestion d'erreurs robuste

## [2.0] - 2024-01-01
### Added
- Système de logging

## [1.0] - 2023-12-25
### Added
- Version initiale
```

## Extensions possibles

### Améliorations futures

**Fonctionnalités avancées** :
1. **Sauvegarde incrémentielle** : Ne sauvegarder que les fichiers modifiés
2. **Chiffrement** : Chiffrer les sauvegardes avec GPG
3. **Sauvegarde distante** : Envoyer vers un serveur distant (rsync, SCP)
4. **Notifications** : Envoyer des emails ou notifications système
5. **Sauvegarde de bases de données** : Intégration avec MySQL, PostgreSQL
6. **Planification** : Intégration avec cron ou systemd
7. **Interface web** : Dashboard pour visualiser les sauvegardes
8. **Restauration** : Script de restauration automatique

**Exemple d'extension : Sauvegarde distante** :
```bash
# Ajouter à la fonction create_backup
upload_to_remote() {
    local backup_file="$1"
    local remote_host="${REMOTE_HOST:-backup.example.com}"
    local remote_user="${REMOTE_USER:-backup}"
    local remote_path="${REMOTE_PATH:-/backups}"
    
    log_message "INFO" "Upload vers $remote_host:$remote_path"
    
    if scp "$backup_file" "$remote_user@$remote_host:$remote_path/"; then
        log_message "SUCCESS" "Upload réussi"
        return 0
    else
        log_message "ERROR" "Échec de l'upload"
        return 1
    fi
}
```

## Conclusion

Ce projet pratique vous a guidé à travers la création complète d'un script de sauvegarde fonctionnel, depuis une version simple jusqu'à une solution robuste et professionnelle. En suivant cette progression, vous avez appliqué tous les concepts fondamentaux appris dans les chapitres précédents.

Les compétences développées ici - conception modulaire, gestion d'erreurs, logging, tests, et documentation - sont applicables à tous vos futurs projets de scripting. Ce script peut servir de base pour vos propres besoins d'automatisation, et les extensions suggérées offrent des pistes d'amélioration continue.

La maîtrise du scripting vient de la pratique régulière. Continuez à créer des scripts pour automatiser vos tâches quotidiennes, et vous développerez rapidement une expertise solide en automatisation système.

Félicitations ! Vous avez complété la Partie 1 de Shellcraft Ultimate 300. Vous êtes maintenant prêt à explorer les parties avancées : Linux approfondi, Bash avancé, PowerShell, APIs, DevOps, et Intelligence Artificielle.

