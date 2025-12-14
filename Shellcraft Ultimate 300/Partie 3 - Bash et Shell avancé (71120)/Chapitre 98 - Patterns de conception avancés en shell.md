# Chapitre 98 - Patterns de conception avancés en shell

> "Les patterns ne sont pas des recettes : ce sont des cristallisations d'expérience, des formes émergentes de la sagesse collective des programmeurs qui ont affronté les mêmes défis avant nous." - Christopher Alexander (adapté au shell)

## Introduction : L'architecture invisible du code

Imaginez-vous architecte d'un gratte-ciel : chaque pattern de conception est comme un élément structural invisible qui donne à l'ensemble sa solidité et son élégance. Les patterns de conception avancés en shell ne sont pas de simples recettes - ce sont des architectures éprouvées qui transforment le chaos du code complexe en harmonies maintenables.

Dans ce chapitre, nous explorerons les patterns architecturaux qui permettent d'écrire du code shell non seulement fonctionnel, mais profondément élégant et évolutif. Ces patterns transcendent les langages pour révéler les principes universels du logiciel bien conçu.

## Section 1 : Patterns structuraux fondamentaux

### 1.1 Pattern Builder : Construction d'objets complexes

Le Builder Pattern permet de construire des objets complexes étape par étape, isolant la logique de construction de la représentation finale :

```bash
#!/bin/bash

# Pattern Builder : Construction d'objets complexes
echo "=== Pattern Builder ==="

# Builder Pattern Implementation
Builder() {
    local self="$1"
    
    declare -A $self._build_parts
    declare -A $self._build_config
    
    # Méthode de construction pour les scripts
    $self.build_script() {
        local script_type="$1"
        local output_file="$2"
        shift 2
        
        echo "Construction du script: $script_type -> $output_file"
        
        # Initialisation du builder
        $self._reset
        
        # Application des parties de construction
        case "$script_type" in
            monitoring)
                $self._build_monitoring_script "$@"
                ;;
            backup)
                $self._build_backup_script "$@"
                ;;
            deployment)
                $self._build_deployment_script "$@"
                ;;
            service)
                $self._build_service_script "$@"
                ;;
            *)
                echo "Type de script non supporté: $script_type" >&2
                return 1
                ;;
        esac
        
        # Génération finale
        $self._generate_script "$output_file"
    }
    
    # Reset du builder
    $self._reset() {
        $self._build_parts=()
        $self._build_config=()
    }
    
    # Ajout d'une partie au script
    $self._add_part() {
        local part_name="$1"
        local part_content="$2"
        
        $self._build_parts["$part_name"]="${part_content}"
    }
    
    # Configuration du builder
    $self._set_config() {
        local key="$1"
        local value="$2"
        
        $self._build_config["$key"]="$value"
    }
    
    # Construction d'un script de monitoring
    $self._build_monitoring_script() {
        local target="$1"
        local metrics="${2:-cpu,memory,disk}"
        local interval="${3:-30}"
        
        $self._set_config "script_type" "monitoring"
        $self._set_config "target" "$target"
        $self._set_config "metrics" "$metrics"
        $self._set_config "interval" "$interval"
        
        # Partie header
        $self._add_part "header" "#!/bin/bash
# Script de monitoring généré automatiquement
# Cible: $target
# Métriques: $metrics
# Intervalle: ${interval}s
# Généré le: $(date)
"

        # Partie configuration
        $self._add_part "config" "
# Configuration
MONITOR_TARGET=\"$target\"
MONITOR_INTERVAL=\"$interval\"
MONITOR_METRICS=\"$metrics\"
DATA_FILE=\"/tmp/monitoring_\${MONITOR_TARGET}_\$(date +%Y%m%d).log\"
ALERT_THRESHOLD=\"80\"
"

        # Partie fonctions de collecte
        $self._add_part "functions" "
# Fonctions de collecte de métriques
collect_cpu() {
    top -bn1 | grep \"Cpu(s)\" | sed \"s/.*, *\\([0-9.]*\\)%* id.*/\\1/\" | awk '{print 100 - \$1}'
}

collect_memory() {
    free | grep Mem | awk '{printf \"%.1f\", \$3/\$2 * 100.0}'
}

collect_disk() {
    df / | tail -1 | awk '{print \$5}' | sed 's/%//'
}

collect_network() {
    local interface=\$(ip route | grep default | awk '{print \$5}' | head -1)
    if [[ -n \"\$interface\" ]]; then
        cat /proc/net/dev | grep \"\$interface\" | awk '{print \$2 \" \" \$10}'
    else
        echo \"0 0\"
    fi
}

# Fonction principale de collecte
collect_metrics() {
    local timestamp=\$(date +%s)
    local data_line=\"\$timestamp\"
    
    IFS=',' read -ra METRIC_ARRAY <<< \"\$MONITOR_METRICS\"
    for metric in \"\${METRIC_ARRAY[@]}\"; do
        case \"\$metric\" in
            cpu)
                local value=\$(collect_cpu)
                data_line=\"\$data_line,\${value:-0}\"
                ;;
            memory)
                local value=\$(collect_memory)
                data_line=\"\$data_line,\${value:-0}\"
                ;;
            disk)
                local value=\$(collect_disk)
                data_line=\"\$data_line,\${value:-0}\"
                ;;
            network)
                local values=\$(collect_network)
                data_line=\"\$data_line,\${values:-0 0}\"
                ;;
        esac
    done
    
    echo \"\$data_line\" >> \"\$DATA_FILE\"
    echo \"Métriques collectées: \$timestamp\"
}

# Fonction d'alerte
check_alerts() {
    # Logique d'alerte simplifiée
    local latest_data=\$(tail -1 \"\$DATA_FILE\" 2>/dev/null)
    if [[ -n \"\$latest_data\" ]]; then
        local cpu_value=\$(echo \"\$latest_data\" | cut -d',' -f2)
        if (( \$(echo \"\$cpu_value > \$ALERT_THRESHOLD\" | bc -l 2>/dev/null || echo 0) )); then
            echo \"ALERTE: CPU > \$ALERT_THRESHOLD% (valeur: \$cpu_value%)\"
        fi
    fi
}
"

        # Partie main
        $self._add_part "main" "
# Point d'entrée principal
main() {
    echo \"Démarrage du monitoring pour \$MONITOR_TARGET\"
    echo \"Intervalle: \$MONITOR_INTERVAL secondes\"
    echo \"Métriques surveillées: \$MONITOR_METRICS\"
    echo \"Données: \$DATA_FILE\"
    echo
    
    # Création du fichier de données
    echo \"timestamp,\$MONITOR_METRICS\" > \"\$DATA_FILE\"
    
    # Boucle de monitoring
    while true; do
        collect_metrics
        check_alerts
        sleep \"\$MONITOR_INTERVAL\"
    done
}

# Gestion des signaux
trap 'echo \"Arrêt du monitoring...\"; exit 0' INT TERM

# Exécution
if [[ \"\${BASH_SOURCE[0]}\" == \"\$0\" ]]; then
    main \"\$@\"
fi
"
    }
    
    # Construction d'un script de sauvegarde
    $self._build_backup_script() {
        local source_dir="$1"
        local dest_dir="$2"
        local compression="${3:-gzip}"
        local retention="${4:-7}"
        
        $self._set_config "script_type" "backup"
        $self._set_config "source" "$source_dir"
        $self._set_config "destination" "$dest_dir"
        $self._set_config "compression" "$compression"
        $self._set_config "retention" "$retention"
        
        $self._add_part "header" "#!/bin/bash
# Script de sauvegarde généré automatiquement
# Source: $source_dir
# Destination: $dest_dir
# Compression: $compression
# Rétention: ${retention} jours
# Généré le: $(date)
"

        $self._add_part "config" "
# Configuration
SOURCE_DIR=\"$source_dir\"
DEST_DIR=\"$dest_dir\"
COMPRESSION=\"$compression\"
RETENTION_DAYS=\"$retention\"
BACKUP_PREFIX=\"backup_\$(date +%Y%m%d_%H%M%S)\"
LOG_FILE=\"/var/log/backup.log\"
"

        $self._add_part "functions" "
# Fonctions de sauvegarde
create_backup() {
    local backup_name=\"\$1\"
    local source_path=\"\$2\"
    local dest_path=\"\$3\"
    
    echo \"Création de la sauvegarde: \$backup_name\" | tee -a \"\$LOG_FILE\"
    
    case \"\$COMPRESSION\" in
        gzip)
            if tar -czf \"\$dest_path\" -C \"\$source_path\" . 2>/dev/null; then
                echo \"Sauvegarde gzip créée: \$dest_path\" | tee -a \"\$LOG_FILE\"
                return 0
            fi
            ;;
        bzip2)
            if tar -cjf \"\$dest_path\" -C \"\$source_path\" . 2>/dev/null; then
                echo \"Sauvegarde bzip2 créée: \$dest_path\" | tee -a \"\$LOG_FILE\"
                return 0
            fi
            ;;
        xz)
            if tar -cJf \"\$dest_path\" -C \"\$source_path\" . 2>/dev/null; then
                echo \"Sauvegarde xz créée: \$dest_path\" | tee -a \"\$LOG_FILE\"
                return 0
            fi
            ;;
        none)
            if cp -r \"\$source_path\"/* \"\$dest_path\" 2>/dev/null; then
                echo \"Sauvegarde copie créée: \$dest_path\" | tee -a \"\$LOG_FILE\"
                return 0
            fi
            ;;
    esac
    
    echo \"ÉCHEC de création de la sauvegarde\" | tee -a \"\$LOG_FILE\"
    return 1
}

cleanup_old_backups() {
    local backup_dir=\"\$1\"
    
    echo \"Nettoyage des anciennes sauvegardes...\" | tee -a \"\$LOG_FILE\"
    
    local old_backups=\$(find \"\$backup_dir\" -name \"backup_*.tar.*\" -mtime +\$RETENTION_DAYS 2>/dev/null)
    
    if [[ -n \"\$old_backups\" ]]; then
        echo \"\$old_backups\" | while read -r old_backup; do
            echo \"Suppression: \$old_backup\" | tee -a \"\$LOG_FILE\"
            rm -f \"\$old_backup\"
        done
    else
        echo \"Aucune ancienne sauvegarde à supprimer\" | tee -a \"\$LOG_FILE\"
    fi
}

verify_backup() {
    local backup_file=\"\$1\"
    
    echo \"Vérification de la sauvegarde: \$backup_file\" | tee -a \"\$LOG_FILE\"
    
    if [[ -f \"\$backup_file\" ]]; then
        local file_size=\$(stat -f%z \"\$backup_file\" 2>/dev/null || stat -c%s \"\$backup_file\" 2>/dev/null || echo \"0\")
        echo \"Taille: \$file_size octets\" | tee -a \"\$LOG_FILE\"
        
        # Test d'intégrité basique
        if [[ \"\$COMPRESSION\" != \"none\" ]]; then
            if tar -tf \"\$backup_file\" >/dev/null 2>&1; then
                echo \"Archive intacte\" | tee -a \"\$LOG_FILE\"
                return 0
            else
                echo \"ERREUR: Archive corrompue\" | tee -a \"\$LOG_FILE\"
                return 1
            fi
        else
            echo \"Vérification limitée pour sauvegarde non compressée\" | tee -a \"\$LOG_FILE\"
            return 0
        fi
    else
        echo \"ERREUR: Fichier de sauvegarde introuvable\" | tee -a \"\$LOG_FILE\"
        return 1
    fi
}
"

        $self._add_part "main" "
# Point d'entrée principal
main() {
    local timestamp=\$(date +%Y%m%d_%H%M%S)
    local backup_name=\"backup_\${timestamp}\"
    
    case \"\$COMPRESSION\" in
        none)
            local backup_file=\"\$DEST_DIR/\${backup_name}\"
            ;;
        *)
            local backup_file=\"\$DEST_DIR/\${backup_name}.tar.\${COMPRESSION}\"
            ;;
    esac
    
    echo \"=== SAUVEGARDE AUTOMATISÉE ===\" | tee -a \"\$LOG_FILE\"
    echo \"Source: \$SOURCE_DIR\" | tee -a \"\$LOG_FILE\"
    echo \"Destination: \$backup_file\" | tee -a \"\$LOG_FILE\"
    echo \"Début: \$(date)\" | tee -a \"\$LOG_FILE\"
    echo | tee -a \"\$LOG_FILE\"
    
    # Création du répertoire de destination
    mkdir -p \"\$DEST_DIR\"
    
    # Exécution de la sauvegarde
    if create_backup \"\$backup_name\" \"\$SOURCE_DIR\" \"\$backup_file\"; then
        if verify_backup \"\$backup_file\"; then
            echo \"✓ Sauvegarde réussie\" | tee -a \"\$LOG_FILE\"
            
            # Nettoyage
            cleanup_old_backups \"\$DEST_DIR\"
            
            echo \"Sauvegarde terminée avec succès\" | tee -a \"\$LOG_FILE\"
            return 0
        fi
    fi
    
    echo \"❌ Échec de la sauvegarde\" | tee -a \"\$LOG_FILE\"
    return 1
}

# Gestion des signaux
trap 'echo \"Interruption détectée, annulation de la sauvegarde\" | tee -a \"\$LOG_FILE\"; exit 1' INT TERM

# Exécution
if [[ \"\${BASH_SOURCE[0]}\" == \"\$0\" ]]; then
    main \"\$@\"
fi
"
    }
    
    # Génération finale du script
    $self._generate_script() {
        local output_file="$1"
        
        {
            echo "${$self._build_parts[header]}"
            echo "${$self._build_parts[config]}"
            echo "${$self._build_parts[functions]}"
            echo "${$self._build_parts[main]}"
        } > "$output_file"
        
        chmod +x "$output_file"
        
        local script_type="${$self._build_config[script_type]}"
        echo "✓ Script $script_type généré: $output_file"
    }
}

# Démonstration du Pattern Builder
echo "--- Pattern Builder ---"

Builder "script_builder"

echo
echo "--- Construction d'un script de monitoring ---"
script_builder.build_script "monitoring" "/tmp/monitoring_script.sh" "localhost" "cpu,memory,disk" "60"

echo
echo "Script de monitoring généré:"
head -15 /tmp/monitoring_script.sh

echo
echo "--- Construction d'un script de sauvegarde ---"
script_builder.build_script "backup" "/tmp/backup_script.sh" "/home/user/documents" "/tmp/backups" "gzip" "30"

echo
echo "Script de sauvegarde généré:"
head -15 /tmp/backup_script.sh

echo
echo "--- Test du script de monitoring ---"
timeout 3 bash /tmp/monitoring_script.sh &
sleep 2
kill %1 2>/dev/null || true

# Nettoyage
rm -f /tmp/monitoring_script.sh /tmp/backup_script.sh /tmp/monitoring_localhost_*.log
```

### 1.2 Pattern Factory : Création d'objets spécialisés

Le Factory Pattern encapsule la logique de création d'objets, permettant de créer des instances spécialisées selon les besoins :

```bash
#!/bin/bash

# Pattern Factory : Création d'objets spécialisés
echo "=== Pattern Factory ==="

# Factory Pattern Implementation
Factory() {
    local self="$1"
    
    declare -A $self._registered_types
    declare -A $self._creation_strategies
    
    # Enregistrement d'un type créable
    $self.register_type() {
        local type_name="$1"
        local creation_function="$2"
        local validation_function="${3:-}"
        
        $self._registered_types["$type_name"]="$creation_function"
        
        if [[ -n "$validation_function" ]]; then
            $self._registered_types["${type_name}_validator"]="$validation_function"
        fi
        
        echo "✓ Type enregistré: $type_name"
    }
    
    # Création d'une instance
    $self.create_instance() {
        local type_name="$1"
        shift
        
        local creation_func="${$self._registered_types[$type_name]}"
        
        if [[ -z "$creation_func" ]]; then
            echo "❌ Type non enregistré: $type_name" >&2
            return 1
        fi
        
        echo "Création d'instance: $type_name"
        
        # Validation des paramètres si un validateur existe
        local validator="${$self._registered_types[${type_name}_validator]}"
        if [[ -n "$validator" ]]; then
            if ! $validator "$@"; then
                echo "❌ Paramètres invalides pour $type_name" >&2
                return 1
            fi
        fi
        
        # Exécution de la fonction de création
        $creation_func "$@"
    }
    
    # Factory Method spécialisé pour les connecteurs de base de données
    $self.create_database_connector() {
        local db_type="$1"
        shift
        
        case "$db_type" in
            mysql|mariadb)
                $self.create_instance "mysql_connector" "$@"
                ;;
            postgresql)
                $self.create_instance "postgres_connector" "$@"
                ;;
            sqlite)
                $self.create_instance "sqlite_connector" "$@"
                ;;
            mongodb)
                $self.create_instance "mongodb_connector" "$@"
                ;;
            *)
                echo "❌ Type de base de données non supporté: $db_type" >&2
                return 1
                ;;
        esac
    }
    
    # Factory pour les gestionnaires de cache
    $self.create_cache_manager() {
        local cache_type="$1"
        shift
        
        case "$cache_type" in
            redis)
                $self.create_instance "redis_cache" "$@"
                ;;
            memcached)
                $self.create_instance "memcached_cache" "$@"
                ;;
            filesystem)
                $self.create_instance "filesystem_cache" "$@"
                ;;
            memory)
                $self.create_instance "memory_cache" "$@"
                ;;
            *)
                echo "❌ Type de cache non supporté: $cache_type" >&2
                return 1
                ;;
        esac
    }
    
    # Factory pour les loggers
    $self.create_logger() {
        local log_type="$1"
        shift
        
        case "$log_type" in
            file)
                $self.create_instance "file_logger" "$@"
                ;;
            syslog)
                $self.create_instance "syslog_logger" "$@"
                ;;
            journald)
                $self.create_instance "journald_logger" "$@"
                ;;
            elasticsearch)
                $self.create_instance "elasticsearch_logger" "$@"
                ;;
            *)
                echo "❌ Type de logger non supporté: $log_type" >&2
                return 1
                ;;
        esac
    }
}

# Fonctions de création pour les connecteurs de base de données
create_mysql_connector() {
    local host="$1"
    local port="${2:-3306}"
    local database="$3"
    local username="$4"
    local password="$5"
    
    cat << EOF
# Connecteur MySQL généré
MYSQL_HOST="$host"
MYSQL_PORT="$port"
MYSQL_DATABASE="$database"
MYSQL_USERNAME="$username"
MYSQL_PASSWORD="$password"

mysql_connect() {
    mysql -h"\$MYSQL_HOST" -P"\$MYSQL_PORT" -u"\$MYSQL_USERNAME" -p"\$MYSQL_PASSWORD" "\$MYSQL_DATABASE"
}

mysql_query() {
    local query="\$1"
    mysql -h"\$MYSQL_HOST" -P"\$MYSQL_PORT" -u"\$MYSQL_USERNAME" -p"\$MYSQL_PASSWORD" -e"\$query" "\$MYSQL_DATABASE"
}

mysql_backup() {
    local output_file="\${1:-backup_\$(date +%Y%m%d_%H%M%S).sql}"
    mysqldump -h"\$MYSQL_HOST" -P"\$MYSQL_PORT" -u"\$MYSQL_USERNAME" -p"\$MYSQL_PASSWORD" "\$MYSQL_DATABASE" > "\$output_file"
    echo "Sauvegarde MySQL créée: \$output_file"
}
EOF
}

validate_mysql_params() {
    local host="$1"
    local port="$2"
    local database="$3"
    local username="$4"
    local password="$5"
    
    [[ -n "$host" && -n "$database" && -n "$username" && -n "$password" ]] || return 1
    
    # Validation du port
    [[ "$port" =~ ^[0-9]+$ && "$port" -ge 1 && "$port" -le 65535 ]] || return 1
    
    return 0
}

create_postgres_connector() {
    local host="$1"
    local port="${2:-5432}"
    local database="$3"
    local username="$4"
    local password="$5"
    
    cat << EOF
# Connecteur PostgreSQL généré
PGHOST="$host"
PGPORT="$port"
PGDATABASE="$database"
PGUSER="$username"
PGPASSWORD="$password"

postgres_connect() {
    psql -h"\$PGHOST" -p"\$PGPORT" -U"\$PGUSER" -d"\$PGDATABASE"
}

postgres_query() {
    local query="\$1"
    psql -h"\$PGHOST" -p"\$PGPORT" -U"\$PGUSER" -d"\$PGDATABASE" -c"\$query"
}

postgres_backup() {
    local output_file="\${1:-backup_\$(date +%Y%m%d_%H%M%S).sql}"
    pg_dump -h"\$PGHOST" -p"\$PGPORT" -U"\$PGUSER" -d"\$PGDATABASE" > "\$output_file"
    echo "Sauvegarde PostgreSQL créée: \$output_file"
}
EOF
}

create_sqlite_connector() {
    local db_file="$1"
    
    cat << EOF
# Connecteur SQLite généré
SQLITE_DB_FILE="$db_file"

sqlite_connect() {
    sqlite3 "\$SQLITE_DB_FILE"
}

sqlite_query() {
    local query="\$1"
    sqlite3 "\$SQLITE_DB_FILE" "\$query"
}

sqlite_backup() {
    local output_file="\${1:-\${SQLITE_DB_FILE}.backup}"
    sqlite3 "\$SQLITE_DB_FILE" ".backup \$output_file"
    echo "Sauvegarde SQLite créée: \$output_file"
}
EOF
}

# Fonctions de création pour les gestionnaires de cache
create_redis_cache() {
    local host="${1:-localhost}"
    local port="${2:-6379}"
    local password="$3"
    
    cat << EOF
# Cache Redis généré
REDIS_HOST="$host"
REDIS_PORT="$port"
REDIS_PASSWORD="$password"

redis_set() {
    local key="\$1"
    local value="\$2"
    local ttl="\${3:-3600}"
    
    if [[ -n "\$REDIS_PASSWORD" ]]; then
        redis-cli -h "\$REDIS_HOST" -p "\$REDIS_PORT" -a "\$REDIS_PASSWORD" SET "\$key" "\$value" EX "\$ttl"
    else
        redis-cli -h "\$REDIS_HOST" -p "\$REDIS_PORT" SET "\$key" "\$value" EX "\$ttl"
    fi
}

redis_get() {
    local key="\$1"
    
    if [[ -n "\$REDIS_PASSWORD" ]]; then
        redis-cli -h "\$REDIS_HOST" -p "\$REDIS_PORT" -a "\$REDIS_PASSWORD" GET "\$key"
    else
        redis-cli -h "\$REDIS_HOST" -p "\$REDIS_PORT" GET "\$key"
    fi
}

redis_delete() {
    local key="\$1"
    
    if [[ -n "\$REDIS_PASSWORD" ]]; then
        redis-cli -h "\$REDIS_HOST" -p "\$REDIS_PORT" -a "\$REDIS_PASSWORD" DEL "\$key"
    else
        redis-cli -h "\$REDIS_HOST" -p "\$REDIS_PORT" DEL "\$key"
    fi
}
EOF
}

create_filesystem_cache() {
    local cache_dir="${1:-/tmp/cache}"
    
    cat << EOF
# Cache système de fichiers généré
CACHE_DIR="$cache_dir"

fs_cache_set() {
    local key="\$1"
    local value="\$2"
    local ttl="\${3:-3600}"
    
    mkdir -p "\$CACHE_DIR"
    local cache_file="\$CACHE_DIR/\$(echo "\$key" | md5sum | cut -d' ' -f1)"
    
    # Stockage avec timestamp d'expiration
    local expiry=\$(( \$(date +%s) + ttl ))
    echo "\$expiry" > "\$cache_file.expiry"
    echo "\$value" > "\$cache_file"
}

fs_cache_get() {
    local key="\$1"
    
    local cache_file="\$CACHE_DIR/\$(echo "\$key" | md5sum | cut -d' ' -f1)"
    
    if [[ -f "\$cache_file" && -f "\$cache_file.expiry" ]]; then
        local expiry=\$(cat "\$cache_file.expiry")
        local now=\$(date +%s)
        
        if (( now < expiry )); then
            cat "\$cache_file"
            return 0
        else
            # Cache expiré, nettoyage
            rm -f "\$cache_file" "\$cache_file.expiry"
        fi
    fi
    
    return 1
}

fs_cache_delete() {
    local key="\$1"
    
    local cache_file="\$CACHE_DIR/\$(echo "\$key" | md5sum | cut -d' ' -f1)"
    rm -f "\$cache_file" "\$cache_file.expiry"
}
EOF
}

# Fonctions de création pour les loggers
create_file_logger() {
    local log_file="$1"
    local max_size="${2:-10485760}"  # 10MB par défaut
    local max_files="${3:-5}"
    
    cat << EOF
# Logger fichier généré
LOG_FILE="$log_file"
LOG_MAX_SIZE="$max_size"
LOG_MAX_FILES="$max_files"

file_log() {
    local level="\$1"
    local message="\$2"
    local timestamp=\$(date '+%Y-%m-%d %H:%M:%S')
    
    # Rotation si nécessaire
    if [[ -f "\$LOG_FILE" ]]; then
        local size=\$(stat -f%z "\$LOG_FILE" 2>/dev/null || stat -c%s "\$LOG_FILE" 2>/dev/null || echo "0")
        if (( size > LOG_MAX_SIZE )); then
            rotate_log_file
        fi
    fi
    
    echo "[\$timestamp] [\$level] \$message" >> "\$LOG_FILE"
}

rotate_log_file() {
    # Rotation des anciens fichiers
    for ((i=LOG_MAX_FILES; i>=1; i--)); do
        if [[ -f "\${LOG_FILE}.\$i" ]]; then
            if (( i == LOG_MAX_FILES )); then
                rm -f "\${LOG_FILE}.\$i"
            else
                mv "\${LOG_FILE}.\$i" "\${LOG_FILE}.\$((i+1))"
            fi
        fi
    done
    
    # Renommage du fichier actuel
    if [[ -f "\$LOG_FILE" ]]; then
        mv "\$LOG_FILE" "\${LOG_FILE}.1"
    fi
}
EOF
}

create_syslog_logger() {
    local facility="${1:-user}"
    local priority="${2:-info}"
    
    cat << EOF
# Logger syslog généré
SYSLOG_FACILITY="$facility"
SYSLOG_PRIORITY="$priority"

syslog_log() {
    local level="\$1"
    local message="\$2"
    
    # Mapping des niveaux aux priorités syslog
    case "\$level" in
        DEBUG) prio="debug" ;;
        INFO) prio="info" ;;
        WARN|WARNING) prio="warning" ;;
        ERROR) prio="err" ;;
        CRITICAL|FATAL) prio="crit" ;;
        *) prio="info" ;;
    esac
    
    logger -p "\${SYSLOG_FACILITY}.\$prio" "\$message"
}
EOF
}

# Démonstration du Pattern Factory
echo "--- Pattern Factory ---"

Factory "object_factory"

# Enregistrement des types
echo "Enregistrement des types de connecteurs de base de données..."
object_factory.register_type "mysql_connector" "create_mysql_connector" "validate_mysql_params"
object_factory.register_type "postgres_connector" "create_postgres_connector"
object_factory.register_type "sqlite_connector" "create_sqlite_connector"

echo
echo "Enregistrement des types de cache..."
object_factory.register_type "redis_cache" "create_redis_cache"
object_factory.register_type "filesystem_cache" "create_filesystem_cache"

echo
echo "Enregistrement des types de loggers..."
object_factory.register_type "file_logger" "create_file_logger"
object_factory.register_type "syslog_logger" "create_syslog_logger"

echo
echo "--- Création d'un connecteur MySQL ---"
object_factory.create_database_connector "mysql" "localhost" "3306" "mydb" "user" "password" > /tmp/mysql_connector.sh

echo "Connecteur MySQL créé:"
head -10 /tmp/mysql_connector.sh

echo
echo "--- Création d'un cache Redis ---"
object_factory.create_cache_manager "redis" "localhost" "6379" "" > /tmp/redis_cache.sh

echo "Cache Redis créé:"
head -10 /tmp/redis_cache.sh

echo
echo "--- Création d'un logger fichier ---"
object_factory.create_logger "file" "/var/log/myapp.log" "10485760" "10" > /tmp/file_logger.sh

echo "Logger fichier créé:"
head -10 /tmp/file_logger.sh

echo
echo "--- Création d'un logger syslog ---"
object_factory.create_logger "syslog" "local0" "info" > /tmp/syslog_logger.sh

echo "Logger syslog créé:"
head -10 /tmp/syslog_logger.sh

# Nettoyage
rm -f /tmp/mysql_connector.sh /tmp/redis_cache.sh /tmp/file_logger.sh /tmp/syslog_logger.sh
```

## Section 2 : Patterns comportementaux avancés

### 2.1 Pattern Strategy : Algorithmes interchangeables

Le Strategy Pattern permet de sélectionner dynamiquement des algorithmes ou stratégies d'exécution :

```bash
#!/bin/bash

# Pattern Strategy : Algorithmes interchangeables
echo "=== Pattern Strategy ==="

# Strategy Pattern Implementation
StrategyContext() {
    local self="$1"
    
    declare -A $self._strategies
    declare -A $self._current_strategy
    
    # Enregistrement d'une stratégie
    $self.register_strategy() {
        local strategy_name="$1"
        local strategy_function="$2"
        local description="$3"
        
        $self._strategies["${strategy_name}_function"]="$strategy_function"
        $self._strategies["${strategy_name}_description"]="$description"
        
        echo "✓ Stratégie enregistrée: $strategy_name"
    }
    
    # Sélection d'une stratégie
    $self.set_strategy() {
        local strategy_name="$1"
        
        if [[ -z "${$self._strategies[${strategy_name}_function]}" ]]; then
            echo "❌ Stratégie inconnue: $strategy_name" >&2
            return 1
        fi
        
        $self._current_strategy["name"]="$strategy_name"
        $self._current_strategy["function"]="${$self._strategies[${strategy_name}_function]}"
        
        echo "✓ Stratégie sélectionnée: $strategy_name"
    }
    
    # Exécution avec la stratégie courante
    $self.execute_strategy() {
        local current_func="${$self._current_strategy[function]}"
        
        if [[ -z "$current_func" ]]; then
            echo "❌ Aucune stratégie sélectionnée" >&2
            return 1
        fi
        
        echo "Exécution avec stratégie: ${$self._current_strategy[name]}"
        $current_func "$@"
    }
    
    # Exécution avec sélection automatique de stratégie
    $self.execute_with_auto_selection() {
        shift
        local -a args=("$@")
        
        # Analyse des arguments pour sélection automatique
        local strategy_name=""
        
        # Sélection basée sur la taille des données
        if [[ ${#args[@]} -gt 10 ]]; then
            strategy_name="parallel_processing"
        elif [[ -n "${args[0]}" && "${args[0]}" =~ ^[0-9]+$ && "${args[0]}" -gt 1000 ]]; then
            strategy_name="optimized_sorting"
        elif [[ -n "${args[0]}" && -f "${args[0]}" ]]; then
            local file_size
            file_size="$(stat -f%z "${args[0]}" 2>/dev/null || stat -c%s "${args[0]}" 2>/dev/null || echo "0")"
            if (( file_size > 1048576 )); then  # > 1MB
                strategy_name="memory_efficient"
            else
                strategy_name="standard_processing"
            fi
        else
            strategy_name="standard_processing"
        fi
        
        $self.set_strategy "$strategy_name"
        $self.execute_strategy "${args[@]}"
    }
    
    # Liste des stratégies disponibles
    $self.list_strategies() {
        echo "Stratégies disponibles:"
        
        for strategy_key in "${!$self._strategies[@]}"; do
            if [[ "$strategy_key" =~ _description$ ]]; then
                local strategy_name="${strategy_key%_description}"
                local description="${$self._strategies[$strategy_key]}"
                
                echo "  $strategy_name: $description"
            fi
        done
    }
}

# Stratégies de traitement de données
data_processing_strategy_standard() {
    echo "=== STRATÉGIE STANDARD ==="
    
    local -a data=("$@")
    
    echo "Traitement séquentiel des ${#data[@]} éléments..."
    
    for item in "${data[@]}"; do
        # Traitement simulé
        echo "  Traitement: $item -> ${item}_processed"
        sleep 0.1
    done
    
    echo "✓ Traitement standard terminé"
}

data_processing_strategy_parallel() {
    echo "=== STRATÉGIE PARALLÈLE ==="
    
    local -a data=("$@")
    
    echo "Traitement parallèle des ${#data[@]} éléments..."
    
    # Traitement parallèle simulé
    local pids=()
    
    for item in "${data[@]}"; do
        (
            echo "  Traitement parallèle: $item -> ${item}_parallel_processed"
            sleep 0.2
        ) &
        pids+=($!)
    done
    
    # Attente de tous les processus
    for pid in "${pids[@]}"; do
        wait "$pid"
    done
    
    echo "✓ Traitement parallèle terminé"
}

data_processing_strategy_memory_efficient() {
    echo "=== STRATÉGIE MÉMOIRE EFFICACE ==="
    
    local file_path="$1"
    
    echo "Traitement en flux du fichier: $file_path"
    
    if [[ ! -f "$file_path" ]]; then
        echo "❌ Fichier introuvable: $file_path"
        return 1
    fi
    
    local line_count=0
    while IFS= read -r line; do
        # Traitement ligne par ligne pour économiser la mémoire
        echo "  Ligne $((++line_count)): ${line:0:50}... -> processed"
    done < "$file_path"
    
    echo "✓ Traitement en flux terminé ($line_count lignes)"
}

data_processing_strategy_optimized_sorting() {
    echo "=== STRATÉGIE TRI OPTIMISÉ ==="
    
    local array_size="$1"
    
    echo "Tri optimisé pour tableau de taille $array_size"
    
    if (( array_size > 1000 )); then
        echo "Utilisation de l'algorithme de tri externe..."
        # Simulation d'un tri externe
        echo "  Phase 1: Division en chunks"
        echo "  Phase 2: Tri des chunks"
        echo "  Phase 3: Fusion des chunks triés"
    else
        echo "Utilisation du tri rapide standard..."
        # Simulation d'un tri rapide
        echo "  Partitionnement récursif"
        echo "  Tri des partitions"
    fi
    
    echo "✓ Tri optimisé terminé"
}

# Stratégies de sauvegarde
backup_strategy_full() {
    echo "=== STRATÉGIE SAUVEGARDE COMPLÈTE ==="
    
    local source="$1"
    local destination="$2"
    
    echo "Sauvegarde complète: $source -> $destination"
    
    # Simulation de sauvegarde complète
    mkdir -p "$destination"
    echo "Copie complète de tous les fichiers..."
    echo "✓ Sauvegarde complète terminée"
}

backup_strategy_incremental() {
    echo "=== STRATÉGIE SAUVEGARDE INCRÉMENTALE ==="
    
    local source="$1"
    local destination="$2"
    
    echo "Sauvegarde incrémentale: $source -> $destination"
    
    # Simulation de sauvegarde incrémentale
    echo "Analyse des fichiers modifiés depuis la dernière sauvegarde..."
    echo "Sauvegarde des fichiers modifiés uniquement..."
    echo "✓ Sauvegarde incrémentale terminée"
}

backup_strategy_differential() {
    echo "=== STRATÉGIE SAUVEGARDE DIFFÉRENTIELLE ==="
    
    local source="$1"
    local destination="$2"
    
    echo "Sauvegarde différentielle: $source -> $destination"
    
    # Simulation de sauvegarde différentielle
    echo "Comparaison avec la dernière sauvegarde complète..."
    echo "Sauvegarde des fichiers modifiés..."
    echo "✓ Sauvegarde différentielle terminée"
}

# Stratégies de déploiement
deployment_strategy_immediate() {
    echo "=== STRATÉGIE DÉPLOIEMENT IMMÉDIAT ==="
    
    local app_name="$1"
    local version="$2"
    
    echo "Déploiement immédiat de $app_name v$version"
    
    # Simulation de déploiement immédiat
    echo "Arrêt de l'ancienne version..."
    echo "Déploiement de la nouvelle version..."
    echo "Démarrage de la nouvelle version..."
    echo "✓ Déploiement immédiat terminé"
}

deployment_strategy_blue_green() {
    echo "=== STRATÉGIE DÉPLOIEMENT BLUE/GREEN ==="
    
    local app_name="$1"
    local version="$2"
    
    echo "Déploiement blue/green de $app_name v$version"
    
    # Simulation de déploiement blue/green
    echo "Déploiement vers l'environnement green..."
    echo "Tests de santé de l'environnement green..."
    echo "Basculement du trafic vers green..."
    echo "✓ Déploiement blue/green terminé"
}

deployment_strategy_canary() {
    echo "=== STRATÉGIE DÉPLOIEMENT CANARY ==="
    
    local app_name="$1"
    local version="$2"
    local percentage="${3:-10}"
    
    echo "Déploiement canary de $app_name v$version ($percentage%)"
    
    # Simulation de déploiement canary
    echo "Déploiement initial à $percentage% du trafic..."
    echo "Surveillance des métriques..."
    echo "Augmentation progressive du trafic..."
    echo "✓ Déploiement canary terminé"
}

# Démonstration du Pattern Strategy
echo "--- Pattern Strategy ---"

StrategyContext "strategy_context"

# Enregistrement des stratégies de traitement de données
strategy_context.register_strategy "standard_processing" "data_processing_strategy_standard" "Traitement séquentiel standard"
strategy_context.register_strategy "parallel_processing" "data_processing_strategy_parallel" "Traitement parallèle pour gros volumes"
strategy_context.register_strategy "memory_efficient" "data_processing_strategy_memory_efficient" "Traitement en flux pour économiser la mémoire"
strategy_context.register_strategy "optimized_sorting" "data_processing_strategy_optimized_sorting" "Tri optimisé selon la taille des données"

# Enregistrement des stratégies de sauvegarde
strategy_context.register_strategy "full_backup" "backup_strategy_full" "Sauvegarde complète de tous les fichiers"
strategy_context.register_strategy "incremental_backup" "backup_strategy_incremental" "Sauvegarde des fichiers modifiés uniquement"
strategy_context.register_strategy "differential_backup" "backup_strategy_differential" "Sauvegarde différentielle depuis la dernière complète"

# Enregistrement des stratégies de déploiement
strategy_context.register_strategy "immediate_deployment" "deployment_strategy_immediate" "Déploiement immédiat avec interruption"
strategy_context.register_strategy "blue_green_deployment" "deployment_strategy_blue_green" "Déploiement sans interruption"
strategy_context.register_strategy "canary_deployment" "deployment_strategy_canary" "Déploiement progressif avec tests"

echo
echo "--- Liste des stratégies ---"
strategy_context.list_strategies

echo
echo "--- Exécution avec sélection manuelle ---"

echo "1. Traitement de données standard:"
strategy_context.set_strategy "standard_processing"
strategy_context.execute_strategy "data1" "data2" "data3"

echo
echo "2. Sauvegarde complète:"
strategy_context.set_strategy "full_backup"
strategy_context.execute_strategy "/home/user" "/backup/full"

echo
echo "3. Déploiement blue/green:"
strategy_context.set_strategy "blue_green_deployment"
strategy_context.execute_strategy "myapp" "2.1.0"

echo
echo "--- Exécution avec sélection automatique ---"

echo "4. Traitement avec sélection automatique (petite liste):"
strategy_context.execute_with_auto_selection "data1" "data2"

echo
echo "5. Traitement avec sélection automatique (grande liste):"
# Création d'une grande liste pour déclencher la stratégie parallèle
large_data=()
for i in {1..15}; do
    large_data+=("data$i")
done
strategy_context.execute_with_auto_selection "${large_data[@]}"

echo
echo "6. Traitement de fichier volumineux:"
echo "test content line 1" > /tmp/large_test.txt
for i in {2..100}; do
    echo "test content line $i with more data to make it larger" >> /tmp/large_test.txt
done
strategy_context.execute_with_auto_selection "/tmp/large_test.txt"

# Nettoyage
rm -f /tmp/large_test.txt
```

### 2.2 Pattern Observer : Notification d'événements

Le Observer Pattern permet à des objets de s'abonner à des événements et d'être notifiés automatiquement :

```bash
#!/bin/bash

# Pattern Observer : Notification d'événements
echo "=== Pattern Observer ==="

# Observer Pattern Implementation
Observable() {
    local self="$1"
    
    declare -A $self._observers
    declare -A $self._events
    
    # Ajout d'un observateur
    $self.add_observer() {
        local event_type="$1"
        local observer_id="$2"
        local callback_function="$3"
        
        local observer_key="${event_type}_${observer_id}"
        $self._observers["$observer_key"]="$callback_function"
        
        echo "✓ Observateur ajouté: $observer_id pour $event_type"
    }
    
    # Suppression d'un observateur
    $self.remove_observer() {
        local event_type="$1"
        local observer_id="$2"
        
        local observer_key="${event_type}_${observer_id}"
        unset $self._observers["$observer_key"]
        
        echo "✓ Observateur supprimé: $observer_id pour $event_type"
    }
    
    # Notification des observateurs
    $self.notify_observers() {
        local event_type="$1"
        shift
        local -a event_data=("$@")
        
        echo "📢 Notification événement: $event_type"
        
        local observer_count=0
        
        for observer_key in "${!$self._observers[@]}"; do
            if [[ "$observer_key" =~ ^${event_type}_ ]]; then
                local callback="${$self._observers[$observer_key]}"
                local observer_id="${observer_key#${event_type}_}"
                
                echo "  Notification de $observer_id"
                
                # Exécution du callback en arrière-plan
                (
                    eval "$callback" "${event_data[@]}"
                ) &
                
                ((observer_count++))
            fi
        done
        
        if (( observer_count == 0 )); then
            echo "  Aucun observateur pour cet événement"
        else
            echo "  $observer_count observateur(s) notifié(s)"
        fi
        
        # Historique des événements
        local event_record="$(date +%s):$event_type:${event_data[*]}"
        $self._events["$(date +%s_%N)"]="$event_record"
    }
    
    # Émission d'un événement personnalisé
    $self.emit_event() {
        local event_type="$1"
        shift
        
        $self.notify_observers "$event_type" "$@"
    }
    
    # Liste des observateurs
    $self.list_observers() {
        local event_filter="${1:-}"
        
        echo "Observateurs enregistrés:"
        
        for observer_key in "${!$self._observers[@]}"; do
            if [[ -z "$event_filter" || "$observer_key" =~ ^${event_filter}_ ]]; then
                local event_type="${observer_key%%_*}"
                local observer_id="${observer_key#*_}"
                local callback="${$self._observers[$observer_key]}"
                
                echo "  $event_type -> $observer_id (${callback:0:30}...)"
            fi
        done
    }
    
    # Statistiques des événements
    $self.get_event_stats() {
        echo "Statistiques des événements:"
        
        local total_events="${#$self._events[@]}"
        local -A event_types
        
        for event_key in "${!$self._events[@]}"; do
            local event_record="${$self._events[$event_key]}"
            local event_type="$(echo "$event_record" | cut -d: -f2)"
            ((event_types["$event_type"]++))
        done
        
        echo "  Total événements: $total_events"
        
        for event_type in "${!event_types[@]}"; do
            echo "  $event_type: ${event_types[$event_type]}"
        done
    }
}

# Exemple d'observables spécialisés
SystemMonitor() {
    local self="$1"
    
    # Héritage de l'observable de base
    Observable "$self"
    
    # Méthodes spécialisées pour la surveillance système
    $self.monitor_cpu() {
        local threshold="${1:-80}"
        
        while true; do
            local cpu_usage
            cpu_usage="$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')"
            
            if (( $(echo "$cpu_usage > $threshold" | bc -l) )); then
                $self.emit_event "cpu_high" "$cpu_usage" "$threshold"
            fi
            
            sleep 5
        done
    }
    
    $self.monitor_memory() {
        local threshold="${1:-85}"
        
        while true; do
            local mem_usage
            mem_usage="$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100.0}')"
            
            if (( $(echo "$mem_usage > $threshold" | bc -l) )); then
                $self.emit_event "memory_high" "$mem_usage" "$threshold"
            fi
            
            sleep 10
        done
    }
    
    $self.monitor_disk() {
        local threshold="${1:-90}"
        local mount_point="${2:-/}"
        
        while true; do
            local disk_usage
            disk_usage="$(df "$mount_point" | tail -1 | awk '{print $5}' | sed 's/%//')"
            
            if (( disk_usage > threshold )); then
                $self.emit_event "disk_high" "$disk_usage" "$threshold" "$mount_point"
            fi
            
            sleep 60
        done
    }
}

FileWatcher() {
    local self="$1"
    
    Observable "$self"
    
    declare -A $self._watched_files
    
    $self.watch_file() {
        local file_path="$1"
        
        if [[ ! -f "$file_path" ]]; then
            echo "❌ Fichier introuvable: $file_path"
            return 1
        fi
        
        local initial_mtime
        initial_mtime="$(stat -c %Y "$file_path" 2>/dev/null || stat -f %m "$file_path")"
        $self._watched_files["$file_path"]="$initial_mtime"
        
        echo "✓ Surveillance démarrée: $file_path"
        
        while true; do
            if [[ ! -f "$file_path" ]]; then
                $self.emit_event "file_deleted" "$file_path"
                unset $self._watched_files["$file_path"]
                break
            fi
            
            local current_mtime
            current_mtime="$(stat -c %Y "$file_path" 2>/dev/null || stat -f %m "$file_path")"
            local last_mtime="${$self._watched_files[$file_path]}"
            
            if [[ "$current_mtime" != "$last_mtime" ]]; then
                local file_size
                file_size="$(stat -c %s "$file_path" 2>/dev/null || stat -f %z "$file_path")"
                
                $self.emit_event "file_modified" "$file_path" "$file_size" "$current_mtime"
                $self._watched_files["$file_path"]="$current_mtime"
            fi
            
            sleep 2
        done
    }
    
    $self.watch_directory() {
        local dir_path="$1"
        
        if [[ ! -d "$dir_path" ]]; then
            echo "❌ Répertoire introuvable: $dir_path"
            return 1
        fi
        
        echo "✓ Surveillance démarrée: $dir_path"
        
        # Obtention de l'état initial
        local initial_state
        initial_state="$(find "$dir_path" -type f -exec stat -c '%Y %n' {} \; 2>/dev/null | sort)"
        
        while true; do
            local current_state
            current_state="$(find "$dir_path" -type f -exec stat -c '%Y %n' {} \; 2>/dev/null | sort)"
            
            if [[ "$current_state" != "$initial_state" ]]; then
                $self.emit_event "directory_changed" "$dir_path"
                initial_state="$current_state"
            fi
            
            sleep 5
        done
    }
}

# Fonctions callback pour les observateurs
cpu_alert_handler() {
    local cpu_usage="$1"
    local threshold="$2"
    
    echo "🚨 ALERTE CPU: $cpu_usage% (seuil: $threshold%)"
    echo "Recommandations: Vérifier les processus consommateurs, envisager redémarrage"
}

memory_alert_handler() {
    local mem_usage="$1"
    local threshold="$2"
    
    echo "🚨 ALERTE MÉMOIRE: $mem_usage% (seuil: $threshold%)"
    echo "Recommandations: Libérer la mémoire, vérifier les fuites"
}

disk_alert_handler() {
    local disk_usage="$1"
    local threshold="$2"
    local mount_point="$3"
    
    echo "🚨 ALERTE DISQUE: $disk_usage% utilisé sur $mount_point (seuil: $threshold%)"
    echo "Recommandations: Nettoyer les fichiers temporaires, ajouter de l'espace"
}

file_modified_handler() {
    local file_path="$1"
    local file_size="$2"
    local mtime="$3"
    
    echo "📝 FICHIER MODIFIÉ: $file_path"
    echo "  Taille: $file_size octets"
    echo "  Modifié: $(date -d "@$mtime" '+%Y-%m-%d %H:%M:%S')"
}

log_error_handler() {
    local log_line="$1"
    
    if echo "$log_line" | grep -qi "error\|exception\|fail"; then
        echo "🚨 ERREUR DÉTECTÉE dans les logs:"
        echo "  $log_line"
    fi
}

# Démonstration du Pattern Observer
echo "--- Pattern Observer ---"

# Création des observables
SystemMonitor "system_monitor"
FileWatcher "file_watcher"

# Enregistrement des observateurs pour le monitoring système
system_monitor.add_observer "cpu_high" "cpu_alert" "cpu_alert_handler"
system_monitor.add_observer "memory_high" "memory_alert" "memory_alert_handler"
system_monitor.add_observer "disk_high" "disk_alert" "disk_alert_handler"

# Enregistrement des observateurs pour la surveillance de fichiers
file_watcher.add_observer "file_modified" "file_logger" "file_modified_handler"

echo
echo "--- Liste des observateurs ---"
system_monitor.list_observers
file_watcher.list_observers

echo
echo "--- Test des événements (simulation) ---"

echo "1. Simulation d'événement CPU élevé:"
system_monitor.emit_event "cpu_high" "85.5" "80"

echo
echo "2. Simulation d'événement mémoire élevé:"
system_monitor.emit_event "memory_high" "90.2" "85"

echo
echo "3. Simulation d'événement disque plein:"
system_monitor.emit_event "disk_high" "95" "90" "/"

echo
echo "4. Test de surveillance de fichier:"
echo "Création d'un fichier de test..."
echo "Contenu initial" > /tmp/test_file.txt

# Lancement de la surveillance en arrière-plan
file_watcher.watch_file "/tmp/test_file.txt" &
watcher_pid=$!

sleep 1

echo "Modification du fichier..."
echo "Contenu modifié à $(date)" >> /tmp/test_file.txt

sleep 3

# Arrêt de la surveillance
kill $watcher_pid 2>/dev/null || true

echo
echo "--- Statistiques des événements ---"
system_monitor.get_event_stats
file_watcher.get_event_stats

# Nettoyage
rm -f /tmp/test_file.txt
```

## Conclusion : L'architecture comme langage universel

Les patterns de conception avancés en shell transcendent les simples recettes de programmation pour devenir un langage architectural universel. Ils permettent d'exprimer des concepts complexes - construction d'objets, stratégies interchangeables, notification d'événements - dans un paradigme qui respecte les contraintes et exploite les forces du shell.

**Points clés à retenir :**

1. **Builder Pattern** : Construction étape par étape d'objets complexes avec séparation des responsabilités
2. **Factory Pattern** : Création d'objets spécialisés avec encapsulation de la logique d'instanciation
3. **Strategy Pattern** : Sélection dynamique d'algorithmes avec sélection automatique basée sur le contexte
4. **Observer Pattern** : Notification d'événements avec découplage entre producteurs et consommateurs

Dans le prochain chapitre, nous explorerons les patterns architecturaux pour systèmes complexes, où ces patterns se combinent pour former des architectures complètes capables de gérer les défis les plus sophistiqués de l'automatisation moderne.

---

**Exercice pratique :** Créez un système complet utilisant tous les patterns présentés :
- Builder pour générer des pipelines CI/CD complexes
- Factory pour créer différents types de connecteurs de services
- Strategy pour implémenter différentes politiques de déploiement
- Observer pour surveiller et réagir aux événements système

**Réflexion :** Comment ces patterns peuvent-ils être combinés pour créer des frameworks d'automatisation auto-adaptatifs capables d'évoluer en fonction de leur environnement et de leurs besoins ?

