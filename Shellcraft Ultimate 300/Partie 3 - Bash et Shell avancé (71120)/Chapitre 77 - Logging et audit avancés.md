# Chapitre 77 - Logging et audit avancés

## Table des matières
- [Introduction](#introduction)
- [Architecture de logging moderne](#architecture-de-logging-moderne)
- [Frameworks de logging pour Bash](#frameworks-de-logging-pour-bash)
- [Rotation et archivage des logs](#rotation-et-archivage-des-logs)
- [Audit et traçabilité](#audit-et-traçabilité)
- [Monitoring des logs en temps réel](#monitoring-des-logs-en-temps-réel)
- [Sécurité et conformité](#sécurité-et-conformité)
- [Conclusion](#conclusion)

## Introduction

Le logging et l'audit avancés constituent les yeux et les oreilles d'un système automatisé. Au-delà de simples messages de débogage, un système de logging sophistiqué permet de tracer l'exécution des scripts, d'auditer les actions sensibles, et de maintenir une traçabilité complète pour le débogage et la conformité.

Imaginez le logging comme un journal de bord détaillé d'un navire : il enregistre chaque décision, chaque action, chaque anomalie, permettant aux administrateurs de retracer les événements passés et d'anticiper les problèmes futurs.

## Architecture de logging moderne

### Niveaux de logging structurés

**Hiérarchie des niveaux de log** :
```bash
# Définition des niveaux (inspiré de syslog)
declare -r -i LOG_EMERGENCY=0   # Système inutilisable
declare -r -i LOG_ALERT=1       # Action immédiate requise
declare -r -i LOG_CRITICAL=2    # Conditions critiques
declare -r -i LOG_ERROR=3       # Erreurs
declare -r -i LOG_WARNING=4     # Avertissements
declare -r -i LOG_NOTICE=5      # Événements normaux mais importants
declare -r -i LOG_INFO=6        # Informations générales
declare -r -i LOG_DEBUG=7       # Informations de débogage

# Variable globale pour le niveau actuel
LOG_LEVEL=${LOG_LEVEL:-$LOG_INFO}
```

**Fonction de logging structuré** :
```bash
# Fonction de logging avec niveaux
log() {
    local level="$1"
    local message="$2"
    local facility="${3:-user}"  # user, mail, daemon, etc.
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local hostname=$(hostname)
    local pid=$$
    local script_name="${0##*/}"

    # Conversion du niveau en numérique
    local -i level_num
    case "$level" in
        EMERGENCY|emerg) level_num=$LOG_EMERGENCY ;;
        ALERT|alert) level_num=$LOG_ALERT ;;
        CRITICAL|crit) level_num=$LOG_CRITICAL ;;
        ERROR|err) level_num=$LOG_ERROR ;;
        WARNING|warn) level_num=$LOG_WARNING ;;
        NOTICE|notice) level_num=$LOG_NOTICE ;;
        INFO|info) level_num=$LOG_INFO ;;
        DEBUG|debug) level_num=$LOG_DEBUG ;;
        *) level_num=$LOG_INFO ;;
    esac

    # Filtrage selon le niveau configuré
    if (( level_num > LOG_LEVEL )); then
        return 0
    fi

    # Format syslog standard
    local log_entry="$timestamp $hostname $script_name[$pid]: [$level] $message"

    # Écriture selon la destination
    case "$LOG_DESTINATION" in
        file)
            echo "$log_entry" >> "$LOG_FILE"
            ;;
        syslog)
            logger -p "${facility}.${level,,}" -t "$script_name" "$message"
            ;;
        journald)
            echo "MESSAGE=$message" | systemd-cat -t "$script_name" -p "$level_num"
            ;;
        both)
            echo "$log_entry" >> "$LOG_FILE"
            logger -p "${facility}.${level,,}" -t "$script_name" "$message"
            ;;
        *)
            echo "$log_entry" >&2
            ;;
    esac
}

# Utilisation
log "INFO" "Script démarré"
log "WARNING" "Configuration manquante, utilisation des valeurs par défaut"
log "ERROR" "Échec de connexion à la base de données"
```

### Configuration centralisée du logging

**Système de configuration flexible** :
```bash
# Configuration par défaut
declare -A LOG_CONFIG=(
    [level]="INFO"
    [destination]="both"  # file, syslog, journald, both, stderr
    [file]="/var/log/${0##*/}.log"
    [facility]="user"
    [max_size]="10485760"  # 10MB
    [max_files]="5"
    [format]="text"  # text, json, xml
    [compression]="gzip"
)

# Fonction d'initialisation du logging
init_logging() {
    # Lecture de la configuration depuis les variables d'environnement
    LOG_LEVEL="${LOG_LEVEL:-${LOG_CONFIG[level]}}"
    LOG_DESTINATION="${LOG_DESTINATION:-${LOG_CONFIG[destination]}}"
    LOG_FILE="${LOG_FILE:-${LOG_CONFIG[file]}}"
    LOG_FACILITY="${LOG_FACILITY:-${LOG_CONFIG[facility]}}"
    LOG_MAX_SIZE="${LOG_MAX_SIZE:-${LOG_CONFIG[max_size]}}"
    LOG_MAX_FILES="${LOG_MAX_FILES:-${LOG_CONFIG[max_files]}}"
    LOG_FORMAT="${LOG_FORMAT:-${LOG_CONFIG[format]}}"

    # Conversion du niveau en numérique
    case "$LOG_LEVEL" in
        EMERGENCY) LOG_LEVEL_NUM=$LOG_EMERGENCY ;;
        ALERT) LOG_LEVEL_NUM=$LOG_ALERT ;;
        CRITICAL) LOG_LEVEL_NUM=$LOG_CRITICAL ;;
        ERROR) LOG_LEVEL_NUM=$LOG_ERROR ;;
        WARNING) LOG_LEVEL_NUM=$LOG_WARNING ;;
        NOTICE) LOG_LEVEL_NUM=$LOG_NOTICE ;;
        INFO) LOG_LEVEL_NUM=$LOG_INFO ;;
        DEBUG) LOG_LEVEL_NUM=$LOG_DEBUG ;;
        *) LOG_LEVEL_NUM=$LOG_INFO ;;
    esac

    # Création du répertoire de logs si nécessaire
    local log_dir=$(dirname "$LOG_FILE")
    if [[ ! -d "$log_dir" ]]; then
        mkdir -p "$log_dir" 2>/dev/null || {
            echo "Erreur: Impossible de créer le répertoire de logs: $log_dir" >&2
            return 1
        }
    fi

    # Test d'écriture
    if ! echo "=== Initialisation du logging: $(date) ===" >> "$LOG_FILE" 2>/dev/null; then
        echo "Erreur: Impossible d'écrire dans le fichier de log: $LOG_FILE" >&2
        return 1
    fi

    log "INFO" "Système de logging initialisé - Niveau: $LOG_LEVEL, Destination: $LOG_DESTINATION"
    return 0
}

# Initialisation automatique
init_logging || {
    echo "Échec de l'initialisation du logging" >&2
    exit 1
}
```

## Frameworks de logging pour Bash

### Framework de logging orienté objet

**Classe Logger en Bash** :
```bash
# Framework de logging inspiré des langages OO
Logger() {
    # Méthode statique pour créer une instance
    local self="logger_instance_$$_$RANDOM"

    # Attributs d'instance
    declare -g "${self}_level"="${LOG_LEVEL:-INFO}"
    declare -g "${self}_destination"="${LOG_DESTINATION:-stderr}"
    declare -g "${self}_file"="${LOG_FILE:-/tmp/app.log}"
    declare -g "${self}_name"="${1:-AppLogger}"

    # Méthodes
    eval "
    ${self}_log() {
        local level=\"\$1\"
        local message=\"\$2\"
        Logger__log_message \"$self\" \"\$level\" \"\$message\"
    }

    ${self}_info() {
        ${self}_log 'INFO' \"\$1\"
    }

    ${self}_error() {
        ${self}_log 'ERROR' \"\$1\"
    }

    ${self}_debug() {
        ${self}_log 'DEBUG' \"\$1\"
    }

    ${self}_set_level() {
        declare -g \"${self}_level\"=\"\$1\"
    }

    ${self}_destroy() {
        unset \"${self}_level\"
        unset \"${self}_destination\"
        unset \"${self}_file\"
        unset \"${self}_name\"
        unset \"${self}_log\"
        unset \"${self}_info\"
        unset \"${self}_error\"
        unset \"${self}_debug\"
        unset \"${self}_set_level\"
        unset \"${self}_destroy\"
    }
    "

    # Retourner le nom de l'instance
    echo "$self"
}

# Méthode privée pour le logging
Logger__log_message() {
    local instance="$1"
    local level="$2"
    local message="$3"

    # Récupération des attributs
    local instance_level=$(eval "echo \${${instance}_level}")
    local instance_destination=$(eval "echo \${${instance}_destination}")
    local instance_file=$(eval "echo \${${instance}_file}")
    local instance_name=$(eval "echo \${${instance}_name}")

    # Vérification du niveau
    local -i level_num message_num
    Logger__get_level_num "$level"
    level_num=$?
    Logger__get_level_num "$instance_level"
    message_num=$?

    if (( level_num > message_num )); then
        return 0
    fi

    # Format du message
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local formatted_msg="[$timestamp] [$instance_name] [$level] $message"

    # Écriture selon la destination
    case "$instance_destination" in
        file)
            echo "$formatted_msg" >> "$instance_file"
            ;;
        stderr)
            echo "$formatted_msg" >&2
            ;;
        both)
            echo "$formatted_msg" >> "$instance_file"
            echo "$formatted_msg" >&2
            ;;
    esac
}

# Fonction utilitaire pour la conversion des niveaux
Logger__get_level_num() {
    case "${1^^}" in
        EMERGENCY|EMERG) return $LOG_EMERGENCY ;;
        ALERT) return $LOG_ALERT ;;
        CRITICAL|CRIT) return $LOG_CRITICAL ;;
        ERROR|ERR) return $LOG_ERROR ;;
        WARNING|WARN) return $LOG_WARNING ;;
        NOTICE) return $LOG_NOTICE ;;
        INFO) return $LOG_INFO ;;
        DEBUG) return $LOG_DEBUG ;;
        *) return $LOG_INFO ;;
    esac
}

# Utilisation du framework
main_logger=$(Logger "MainApp")

$main_logger info "Application démarrée"
$main_logger debug "Configuration chargée"
$main_logger error "Erreur de connexion"

# Changement de niveau
$main_logger set_level "DEBUG"
$main_logger debug "Mode debug activé"

# Nettoyage
$main_logger destroy
```

### Logging asynchrone avec buffer

**Système de logging avec buffer pour les performances** :
```bash
# Buffer circulaire pour le logging asynchrone
declare -a LOG_BUFFER=()
declare -i LOG_BUFFER_SIZE=100
declare -i LOG_BUFFER_INDEX=0
declare LOG_BUFFER_LOCK="/tmp/log_buffer_$$.lock"

# Fonction de logging avec buffer
log_buffered() {
    local level="$1"
    local message="$2"

    # Vérification du niveau
    if ! log_should_write "$level"; then
        return 0
    fi

    # Format du message
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local formatted_msg="[$timestamp] [$level] $message"

    # Acquisition du verrou (simple)
    while [[ -f "$LOG_BUFFER_LOCK" ]]; do
        sleep 0.01
    done
    touch "$LOG_BUFFER_LOCK"

    # Ajout au buffer
    LOG_BUFFER[$LOG_BUFFER_INDEX]="$formatted_msg"
    LOG_BUFFER_INDEX=$(( (LOG_BUFFER_INDEX + 1) % LOG_BUFFER_SIZE ))

    # Libération du verrou
    rm -f "$LOG_BUFFER_LOCK"

    # Flush automatique si buffer plein
    if (( LOG_BUFFER_INDEX == 0 )); then
        log_flush
    fi
}

# Fonction de flush du buffer
log_flush() {
    # Acquisition du verrou
    while [[ -f "$LOG_BUFFER_LOCK" ]]; do
        sleep 0.01
    done
    touch "$LOG_BUFFER_LOCK"

    # Écriture de tous les messages
    local i
    for ((i=0; i<LOG_BUFFER_SIZE; i++)); do
        if [[ -n "${LOG_BUFFER[$i]}" ]]; then
            log_write_message "${LOG_BUFFER[$i]}"
            LOG_BUFFER[$i]=""  # Nettoyage
        fi
    done

    # Libération du verrou
    rm -f "$LOG_BUFFER_LOCK"
}

# Écriture réelle du message
log_write_message() {
    local message="$1"

    case "$LOG_DESTINATION" in
        file)
            echo "$message" >> "$LOG_FILE"
            ;;
        syslog)
            logger -t "${0##*/}" "$message"
            ;;
        both)
            echo "$message" >> "$LOG_FILE"
            logger -t "${0##*/}" "$message"
            ;;
        *)
            echo "$message" >&2
            ;;
    esac
}

# Fonction utilitaire
log_should_write() {
    local level="$1"
    local -i level_num message_num

    case "$level" in
        EMERGENCY) level_num=$LOG_EMERGENCY ;;
        ALERT) level_num=$LOG_ALERT ;;
        CRITICAL) level_num=$LOG_CRITICAL ;;
        ERROR) level_num=$LOG_ERROR ;;
        WARNING) level_num=$LOG_WARNING ;;
        NOTICE) level_num=$LOG_NOTICE ;;
        INFO) level_num=$LOG_INFO ;;
        DEBUG) level_num=$LOG_DEBUG ;;
        *) level_num=$LOG_INFO ;;
    esac

    case "$LOG_LEVEL" in
        EMERGENCY) message_num=$LOG_EMERGENCY ;;
        ALERT) message_num=$LOG_ALERT ;;
        CRITICAL) message_num=$LOG_CRITICAL ;;
        ERROR) message_num=$LOG_ERROR ;;
        WARNING) message_num=$LOG_WARNING ;;
        NOTICE) message_num=$LOG_NOTICE ;;
        INFO) message_num=$LOG_INFO ;;
        DEBUG) message_num=$LOG_DEBUG ;;
        *) message_num=$LOG_INFO ;;
    esac

    (( level_num <= message_num ))
}

# Flush automatique à la fin du script
trap 'log_flush' EXIT

# Utilisation
for i in {1..50}; do
    log_buffered "INFO" "Message $i"
    sleep 0.01
done

echo "Buffer flush final..."
```

## Rotation et archivage des logs

### Système de rotation automatique

**Rotation basée sur la taille** :
```bash
# Fonction de rotation des logs
rotate_logs() {
    local log_file="$1"
    local max_size="${2:-10485760}"  # 10MB par défaut
    local max_files="${3:-5}"

    # Vérification de l'existence du fichier
    if [[ ! -f "$log_file" ]]; then
        return 0
    fi

    # Vérification de la taille
    local current_size=$(stat -f%z "$log_file" 2>/dev/null || stat -c%s "$log_file" 2>/dev/null)
    if (( current_size < max_size )); then
        return 0
    fi

    log "INFO" "Rotation du fichier de log: $log_file (taille: $current_size octets)"

    # Rotation des anciens fichiers
    local i
    for ((i=max_files; i>=1; i--)); do
        local old_file="${log_file}.${i}"
        local new_file="${log_file}.$((i+1))"

        if [[ -f "$old_file" ]]; then
            mv "$old_file" "$new_file" 2>/dev/null || true
        fi
    done

    # Compression de l'ancien fichier principal
    if [[ -f "$log_file" ]]; then
        local compressed_file="${log_file}.1"
        case "$LOG_COMPRESSION" in
            gzip)
                gzip -c "$log_file" > "${compressed_file}.gz" && mv "${compressed_file}.gz" "$compressed_file"
                ;;
            bzip2)
                bzip2 -c "$log_file" > "${compressed_file}.bz2" && mv "${compressed_file}.bz2" "$compressed_file"
                ;;
            xz)
                xz -c "$log_file" > "${compressed_file}.xz" && mv "${compressed_file}.xz" "$compressed_file"
                ;;
            *)
                cp "$log_file" "$compressed_file"
                ;;
        esac

        # Troncation du fichier original
        : > "$log_file"

        log "INFO" "Rotation terminée: ancien log compressé dans ${compressed_file}"
    fi
}

# Rotation basée sur la date
rotate_logs_date() {
    local log_file="$1"
    local date_format="${2:-%Y%m%d}"

    local current_date=$(date "+$date_format")
    local archive_name="${log_file}.${current_date}"

    # Vérifier si la rotation est déjà faite aujourd'hui
    if [[ -f "$archive_name" ]] || [[ -f "${archive_name}.gz" ]]; then
        return 0
    fi

    # Créer l'archive
    if [[ -f "$log_file" ]] && [[ -s "$log_file" ]]; then
        cp "$log_file" "$archive_name"
        : > "$log_file"  # Troncation

        # Compression optionnelle
        if [[ "$LOG_COMPRESSION" != "none" ]]; then
            case "$LOG_COMPRESSION" in
                gzip) gzip "$archive_name" ;;
                bzip2) bzip2 "$archive_name" ;;
                xz) xz "$archive_name" ;;
            esac
        fi

        log "INFO" "Rotation datée effectuée: $archive_name"
    fi
}

# Nettoyage des anciens logs
cleanup_old_logs() {
    local log_file="$1"
    local max_age="${2:-30}"  # 30 jours par défaut

    log "INFO" "Nettoyage des anciens fichiers de log (âge max: ${max_age} jours)"

    # Suppression des fichiers compressés trop anciens
    find "$(dirname "$log_file")" \
         -name "$(basename "$log_file").[0-9]*" \
         -type f \
         -mtime "+$max_age" \
         -exec rm -f {} \; \
         -print | while read -r file; do
             log "INFO" "Supprimé: $file"
         done
}

# Intégration dans le système de logging
log_with_rotation() {
    local level="$1"
    local message="$2"

    # Rotation automatique si nécessaire
    rotate_logs "$LOG_FILE"
    rotate_logs_date "$LOG_FILE"
    cleanup_old_logs "$LOG_FILE"

    # Logging normal
    log "$level" "$message"
}
```

### Archivage intelligent

**Système d'archivage avec métadonnées** :
```bash
# Archivage avec métadonnées
archive_logs() {
    local source_dir="$1"
    local archive_dir="$2"
    local retention_days="${3:-90}"

    local archive_name="logs_archive_$(date +%Y%m%d_%H%M%S)"
    local archive_path="$archive_dir/$archive_name"

    log "INFO" "Création de l'archive: $archive_name"

    # Création de l'archive
    mkdir -p "$archive_path"

    # Copie des fichiers de log
    find "$source_dir" -name "*.log*" -type f -mtime "-$retention_days" \
         -exec cp {} "$archive_path/" \;

    # Création du fichier de métadonnées
    cat > "$archive_path/metadata.txt" << EOF
Archive créée le: $(date)
Répertoire source: $source_dir
Période de rétention: $retention_days jours
Nombre de fichiers: $(find "$archive_path" -type f -name "*.log*" | wc -l)
Taille totale: $(du -sh "$archive_path" | cut -f1)

Contenu:
$(find "$archive_path" -type f -name "*.log*" -exec basename {} \; | sort)
EOF

    # Compression de l'archive
    cd "$archive_dir"
    tar -czf "${archive_name}.tar.gz" "$archive_name"
    rm -rf "$archive_name"

    log "INFO" "Archive créée: ${archive_path}.tar.gz"

    # Vérification de l'intégrité
    if tar -tzf "${archive_name}.tar.gz" >/dev/null 2>&1; then
        log "INFO" "Intégrité de l'archive vérifiée"
    else
        log "ERROR" "Erreur d'intégrité de l'archive"
        return 1
    fi

    return 0
}

# Fonction de restauration
restore_logs() {
    local archive_file="$1"
    local restore_dir="$2"

    log "INFO" "Restauration de l'archive: $archive_file"

    if [[ ! -f "$archive_file" ]]; then
        log "ERROR" "Archive introuvable: $archive_file"
        return 1
    fi

    mkdir -p "$restore_dir"

    if tar -xzf "$archive_file" -C "$restore_dir"; then
        log "INFO" "Archive restaurée dans: $restore_dir"
        return 0
    else
        log "ERROR" "Échec de la restauration"
        return 1
    fi
}
```

## Audit et traçabilité

### Audit des actions sensibles

**Système d'audit automatique** :
```bash
# Journal d'audit pour les actions sensibles
declare -a AUDIT_LOG=()
declare AUDIT_FILE="${AUDIT_FILE:-/var/log/script_audit.log}"

# Fonction d'audit
audit() {
    local action="$1"
    local target="$2"
    local user="${3:-$USER}"
    local result="${4:-success}"
    local details="${5:-}"

    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local hostname=$(hostname)
    local pid=$$
    local script="${0##*/}"

    local audit_entry="$timestamp|$hostname|$script|$pid|$user|$action|$target|$result|$details"

    # Stockage en mémoire
    AUDIT_LOG+=("$audit_entry")

    # Écriture immédiate dans le fichier d'audit
    echo "$audit_entry" >> "$AUDIT_FILE"

    # Logging normal si activé
    log "AUDIT" "Action: $action, Target: $target, Result: $result"
}

# Fonctions d'audit spécialisées
audit_file_access() {
    local file="$1"
    local operation="$2"  # read, write, execute, delete

    if [[ -f "$file" ]]; then
        audit "file_$operation" "$file" "$USER" "success" "$(stat -c%s "$file" 2>/dev/null || echo 'unknown size')"
    else
        audit "file_$operation" "$file" "$USER" "not_found"
    fi
}

audit_command_execution() {
    local command="$1"
    local result="$2"

    audit "command_execution" "$command" "$USER" "$result"
}

# Wrapper pour les commandes sensibles
execute_audited() {
    local command="$1"
    shift

    audit_command_execution "$command" "started"

    if eval "$command" "$@"; then
        audit_command_execution "$command" "success"
        return 0
    else
        local exit_code=$?
        audit_command_execution "$command" "failed:$exit_code"
        return $exit_code
    fi
}

# Surveillance des accès fichiers
 monitored_cp() {
    local source="$1"
    local destination="$2"

    audit_file_access "$source" "read"
    audit_file_access "$destination" "write"

    if cp "$source" "$destination"; then
        audit "file_copy" "$source -> $destination" "$USER" "success"
        return 0
    else
        audit "file_copy" "$source -> $destination" "$USER" "failed"
        return 1
    fi
}

# Rapport d'audit
generate_audit_report() {
    local output_file="${1:-audit_report_$(date +%Y%m%d).txt}"

    {
        echo "=== RAPPORT D'AUDIT ==="
        echo "Généré le: $(date)"
        echo "Script: ${0##*/}"
        echo "Utilisateur: $USER"
        echo "Nombre d'actions auditées: ${#AUDIT_LOG[@]}"
        echo

        echo "=== RÉSUMÉ PAR ACTION ==="
        printf '%-20s %s\n' "Action" "Nombre"
        printf '%-20s %s\n' "------" "------"

        # Analyse des actions
        local -A action_counts=()
        local entry
        for entry in "${AUDIT_LOG[@]}"; do
            local action=$(echo "$entry" | cut -d'|' -f6)
            ((action_counts[$action]++))
        done

        for action in "${!action_counts[@]}"; do
            printf '%-20s %d\n' "$action" "${action_counts[$action]}"
        done

        echo
        echo "=== DÉTAIL DES ACTIONS ==="
        printf '%-19s %-15s %-10s %-8s %-10s %s\n' \
               "Timestamp" "Script" "User" "Action" "Result" "Target/Details"
        echo "--------------------------------------------------------------------------------"

        for entry in "${AUDIT_LOG[@]}"; do
            IFS='|' read -r timestamp hostname script pid user action target result details <<< "$entry"
            printf '%-19s %-15s %-10s %-8s %-10s %s\n' \
                   "$timestamp" "$script" "$user" "$action" "$result" "$target $details"
        done

    } > "$output_file"

    log "INFO" "Rapport d'audit généré: $output_file"
}
```

### Traçabilité des modifications

**Système de versioning des configurations** :
```bash
# Traçabilité des modifications de fichiers
trace_file_changes() {
    local file="$1"
    local change_type="$2"  # create, modify, delete
    local old_hash="${3:-}"
    local new_hash="${4:-}"

    if [[ -f "$file" ]]; then
        new_hash="${new_hash:-$(sha256sum "$file" 2>/dev/null | cut -d' ' -f1)}"
    fi

    audit "file_change" "$file" "$USER" "$change_type" "old:$old_hash,new:$new_hash"

    # Archiver l'ancienne version si elle existe
    if [[ "$change_type" == "modify" ]] && [[ -n "$old_hash" ]]; then
        local backup_dir="${TRACE_BACKUP_DIR:-/var/backups/traces}"
        mkdir -p "$backup_dir"

        local backup_file="$backup_dir/$(basename "$file").$old_hash"
        cp "$file" "$backup_file" 2>/dev/null || true

        log "INFO" "Ancienne version sauvegardée: $backup_file"
    fi
}

# Surveillance des modifications
monitor_file_changes() {
    local file="$1"
    local interval="${2:-5}"  # secondes

    if [[ ! -f "$file" ]]; then
        log "ERROR" "Fichier à surveiller introuvable: $file"
        return 1
    fi

    local last_hash=$(sha256sum "$file" 2>/dev/null | cut -d' ' -f1)
    local last_mtime=$(stat -c%Y "$file" 2>/dev/null || stat -f%m "$file" 2>/dev/null)

    log "INFO" "Début de surveillance du fichier: $file"

    while true; do
        sleep "$interval"

        if [[ ! -f "$file" ]]; then
            trace_file_changes "$file" "delete" "$last_hash"
            log "WARNING" "Fichier supprimé: $file"
            return 0
        fi

        local current_mtime=$(stat -c%Y "$file" 2>/dev/null || stat -f%m "$file" 2>/dev/null)

        if (( current_mtime != last_mtime )); then
            local current_hash=$(sha256sum "$file" 2>/dev/null | cut -d' ' -f1)

            if [[ "$current_hash" != "$last_hash" ]]; then
                trace_file_changes "$file" "modify" "$last_hash" "$current_hash"
                log "INFO" "Fichier modifié: $file"
                last_hash="$current_hash"
            fi

            last_mtime="$current_mtime"
        fi
    done
}
```

## Monitoring des logs en temps réel

### Surveillance avec tail et grep

**Monitoring en temps réel** :
```bash
# Surveillance des logs avec alertes
monitor_logs_realtime() {
    local log_file="$1"
    local patterns="$2"  # motifs à surveiller (séparés par |)
    local alert_command="${3:-}"

    if [[ ! -f "$log_file" ]]; then
        log "ERROR" "Fichier de log introuvable: $log_file"
        return 1
    fi

    log "INFO" "Début de surveillance du fichier: $log_file"

    # Utilisation de tail -f avec grep
    tail -f "$log_file" | grep --line-buffered -E "$patterns" | while read -r line; do
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

        # Logging de l'alerte
        log "ALERT" "Motif détecté dans $log_file: $line"

        # Exécution de la commande d'alerte si fournie
        if [[ -n "$alert_command" ]]; then
            if ! eval "$alert_command" "$line"; then
                log "ERROR" "Échec de la commande d'alerte: $alert_command"
            fi
        fi

        # Gestion des alertes multiples (éviter le spam)
        sleep 1
    done
}

# Fonction d'alerte par email
alert_by_email() {
    local message="$1"
    local recipient="${LOG_ALERT_EMAIL:-admin@example.com}"

    echo "Alerte de log: $message" | mail -s "Alerte système" "$recipient" 2>/dev/null || \
    log "WARNING" "Impossible d'envoyer l'email d'alerte"
}

# Surveillance avec alertes par email
monitor_critical_logs() {
    local log_patterns="ERROR|FATAL|CRITICAL|PANIC"

    # Surveillance des logs système
    monitor_logs_realtime "/var/log/syslog" "$log_patterns" "alert_by_email" &

    # Surveillance des logs d'application
    monitor_logs_realtime "$LOG_FILE" "$log_patterns" "alert_by_email" &

    # Surveillance des logs d'accès (tentatives de connexion échouées)
    monitor_logs_realtime "/var/log/auth.log" "Failed password|Invalid user" "alert_by_email" &

    log "INFO" "Surveillance des logs critique démarrée"
}
```

### Analyse statistique des logs

**Statistiques et métriques** :
```bash
# Analyse statistique des logs
analyze_log_stats() {
    local log_file="$1"
    local output_file="${2:-log_stats_$(date +%Y%m%d).txt}"

    log "INFO" "Analyse statistique des logs: $log_file"

    {
        echo "=== ANALYSE STATISTIQUE DES LOGS ==="
        echo "Fichier analysé: $log_file"
        echo "Date d'analyse: $(date)"
        echo "Taille du fichier: $(du -h "$log_file" | cut -f1)"
        echo "Nombre de lignes: $(wc -l < "$log_file")"
        echo

        echo "=== RÉPARTITION PAR NIVEAU ==="
        grep -E '\[(ERROR|WARN|INFO|DEBUG)\]' "$log_file" | \
        sed -E 's/.*\[([^]]+)\].*/\1/' | \
        sort | uniq -c | sort -nr | \
        while read count level; do
            printf "%-8s %6d\n" "$level:" "$count"
        done
        echo

        echo "=== ERREURS LES PLUS FRÉQUENTES ==="
        grep -i error "$log_file" | \
        sed -E 's/.*ERROR(.*)/\1/' | \
        sed 's/^[[:space:]]*//' | \
        cut -d':' -f1 | \
        sort | uniq -c | sort -nr | head -10 | \
        while read count error; do
            printf "%6d %s\n" "$count" "$error"
        done
        echo

        echo "=== ACTIVITÉ PAR HEURE ==="
        awk '{
            # Extraction de l'heure depuis le timestamp
            split($1, time_parts, ":")
            hour = time_parts[1]
            if (hour ~ /^[0-9]+$/) {
                hours[hour]++
            }
        } END {
            for (h = 0; h <= 23; h++) {
                printf "%02d:00 %6d\n", h, hours[sprintf("%02d", h)]
            }
        }' "$log_file" | sort
        echo

        echo "=== TOP 10 DES MESSAGES ==="
        grep -v '^=' "$log_file" | \
        sed -E 's/^[0-9-]+ [0-9:]+ [^ ]+ [^[]+\[[^]]+\]: //' | \
        sort | uniq -c | sort -nr | head -10 | \
        while read count message; do
            printf "%6d %s\n" "$count" "$message"
        done

    } > "$output_file"

    log "INFO" "Analyse terminée: $output_file"
}

# Génération de rapports périodiques
generate_periodic_reports() {
    local interval="${1:-3600}"  # 1 heure par défaut

    while true; do
        analyze_log_stats "$LOG_FILE"
        monitor_critical_logs  # Redémarrage de la surveillance si nécessaire

        log "INFO" "Prochaine analyse dans $interval secondes"
        sleep "$interval"
    done
}
```

## Sécurité et conformité

### Chiffrement des logs sensibles

**Logs chiffrés** :
```bash
# Fonction de logging chiffré
log_encrypted() {
    local level="$1"
    local message="$2"
    local key_file="${3:-$HOME/.log_key}"

    # Génération de la clé si elle n'existe pas
    if [[ ! -f "$key_file" ]]; then
        openssl rand -base64 32 > "$key_file"
        chmod 600 "$key_file"
        log "INFO" "Nouvelle clé de chiffrement générée: $key_file"
    fi

    # Format du message avec timestamp
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local formatted_msg="[$timestamp] [$level] $message"

    # Chiffrement du message
    echo "$formatted_msg" | openssl enc -aes-256-cbc -salt -pass file:"$key_file" >> "${LOG_FILE}.enc"

    # Logging en clair pour le débogage (optionnel)
    if [[ "$LOG_DEBUG" == "true" ]]; then
        log "$level" "[ENCRYPTED] $message"
    fi
}

# Fonction de déchiffrement pour consultation
decrypt_logs() {
    local encrypted_file="$1"
    local key_file="${2:-$HOME/.log_key}"
    local output_file="${3:-/dev/stdout}"

    if [[ ! -f "$encrypted_file" ]]; then
        log "ERROR" "Fichier chiffré introuvable: $encrypted_file"
        return 1
    fi

    openssl enc -d -aes-256-cbc -pass file:"$key_file" -in "$encrypted_file" -out "$output_file" 2>/dev/null

    if [[ $? -eq 0 ]]; then
        log "INFO" "Logs déchiffrés: $encrypted_file"
    else
        log "ERROR" "Échec du déchiffrement"
        return 1
    fi
}

# Consultation sécurisée des logs
view_encrypted_logs() {
    local temp_file=$(mktemp)

    if decrypt_logs "${LOG_FILE}.enc" "$HOME/.log_key" "$temp_file"; then
        # Affichage avec less pour la navigation
        less "$temp_file"
    fi

    # Nettoyage automatique
    rm -f "$temp_file"
}
```

### Conformité et archivage légal

**Archivage conforme aux normes** :
```bash
# Archivage légal avec empreinte et signature
archive_compliant() {
    local source_dir="$1"
    local archive_name="audit_archive_$(date +%Y%m%d_%H%M%S)"
    local archive_dir="${2:-/var/audit/archives}"
    local retention_years="${3:-7}"

    mkdir -p "$archive_dir"

    log "INFO" "Création d'archive conforme: $archive_name"

    # Calcul des empreintes de tous les fichiers
    local manifest_file="$archive_dir/${archive_name}_manifest.txt"
    {
        echo "=== MANIFESTE D'ARCHIVE ==="
        echo "Créé le: $(date)"
        echo "Utilisateur: $USER"
        echo "Système: $(hostname)"
        echo "Période de rétention: $retention_years ans"
        echo
        echo "=== EMPREINTES DES FICHIERS ==="

        find "$source_dir" -type f -name "*.log*" -exec sha256sum {} \; | \
        while read hash file; do
            echo "$hash  ${file#$source_dir/}"
        done
    } > "$manifest_file"

    # Création de l'archive
    tar -czf "$archive_dir/${archive_name}.tar.gz" \
        -C "$source_dir" \
        $(find "$source_dir" -name "*.log*" -printf "%P\n") \
        "$manifest_file"

    # Signature de l'archive (si gpg disponible)
    if command -v gpg >/dev/null 2>&1; then
        gpg --detach-sign "$archive_dir/${archive_name}.tar.gz"
        log "INFO" "Archive signée: ${archive_name}.tar.gz.sig"
    fi

    # Métadonnées de conformité
    cat > "$archive_dir/${archive_name}_metadata.json" << EOF
{
    "archive_name": "$archive_name",
    "creation_date": "$(date -Iseconds)",
    "created_by": "$USER",
    "system": "$(hostname)",
    "source_directory": "$source_dir",
    "retention_years": $retention_years,
    "file_count": $(find "$source_dir" -name "*.log*" | wc -l),
    "total_size": $(du -sb "$source_dir" | cut -f1),
    "compliance": {
        "gdpr_compliant": true,
        "audit_trail": true,
        "tamper_proof": true
    }
}
EOF

    log "INFO" "Archive conforme créée: $archive_name"
}

# Nettoyage des archives expirées
cleanup_expired_archives() {
    local archive_dir="$1"
    local current_year=$(date +%Y)

    log "INFO" "Nettoyage des archives expirées dans: $archive_dir"

    # Recherche des métadonnées d'archive
    find "$archive_dir" -name "*_metadata.json" -type f | while read -r metadata_file; do
        local retention_years=$(jq -r '.retention_years' "$metadata_file" 2>/dev/null || echo "7")
        local creation_year=$(jq -r '.creation_date' "$metadata_file" 2>/dev/null | cut -d'-' -f1 || date +%Y)

        local age_years=$(( current_year - creation_year ))

        if (( age_years >= retention_years )); then
            local archive_name=$(jq -r '.archive_name' "$metadata_file" 2>/dev/null)

            log "INFO" "Suppression d'archive expirée: $archive_name (âge: ${age_years} ans)"

            # Suppression de tous les fichiers liés
            rm -f "$archive_dir/${archive_name}.tar.gz" \
                  "$archive_dir/${archive_name}.tar.gz.sig" \
                  "$metadata_file"
        fi
    done
}
```

## Conclusion

Le logging et l'audit avancés transforment les scripts Bash en systèmes fiables et traçables. En implémentant des frameworks de logging structurés, des systèmes de rotation automatique, et des mécanismes d'audit complets, vous créez des applications qui non seulement fonctionnent correctement, mais qui maintiennent également une traçabilité complète de leurs actions.

Comme un journal de bord détaillé permet aux navigateurs de retracer leur voyage et d'apprendre de leurs erreurs, un système de logging sophistiqué permet aux administrateurs système de comprendre le comportement de leurs scripts, de diagnostiquer les problèmes, et d'assurer la conformité aux exigences légales et réglementaires.

Dans le chapitre suivant, nous explorerons la gestion des erreurs dans les scripts Bash, découvrant comment anticiper les problèmes, gérer les situations exceptionnelles, et créer des scripts robustes capables de fonctionner de manière fiable dans des environnements hostiles.
