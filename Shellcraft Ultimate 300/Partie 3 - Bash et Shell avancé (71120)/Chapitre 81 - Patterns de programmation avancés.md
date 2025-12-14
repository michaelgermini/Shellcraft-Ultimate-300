# Chapitre 81 - Patterns de programmation avancés

> "Un pattern n'est pas une invention à copier sans réfléchir, mais un modèle à adapter avec intelligence." - Christopher Alexander (adapté à la programmation)

## Introduction : De l'artisanat à l'architecture logicielle

Imaginez-vous architecte de cathédrales gothiques au Moyen Âge. Vous ne partez pas d'une page blanche à chaque projet : vous utilisez des patrons éprouvés - arcs-boutants pour supporter les murs, rosaces pour décorer les façades, voûtes pour couvrir les nefs. Les patterns de programmation sont ces patrons pour les développeurs : des solutions élégantes et réutilisables aux problèmes courants.

Dans ce chapitre, nous allons explorer comment appliquer les grands patterns de conception (Strategy, Factory, Observer, etc.) dans l'univers contraignant mais riche de Bash. Vous apprendrez à structurer vos scripts comme de véritables applications, avec séparation des responsabilités, extensibilité, et maintenabilité.

## Section 1 : Pattern Strategy (Stratégie)

### 1.1 Le concept Strategy

Le pattern Strategy permet de définir une famille d'algorithmes, de les encapsuler, et de les rendre interchangeables. Le client peut choisir l'algorithme approprié sans connaître ses détails internes.

**Analogies pédagogiques :**
- Comme choisir entre différents modes de transport : voiture, train, avion
- Comme sélectionner un algorithme de tri : quicksort, mergesort, bubblesort

### 1.2 Implémentation en Bash

```bash
#!/bin/bash

# Pattern Strategy : Algorithmes de compression interchangeables

# Interface commune pour tous les compresseurs
compress_file() {
    local strategy="$1"
    local input_file="$2"
    local output_file="$3"
    
    case "$strategy" in
        gzip)
            gzip_compress "$input_file" "$output_file"
            ;;
        bzip2)
            bzip2_compress "$input_file" "$output_file"
            ;;
        xz)
            xz_compress "$input_file" "$output_file"
            ;;
        *)
            echo "Stratégie de compression inconnue: $strategy" >&2
            return 1
            ;;
    esac
}

# Implémentation concrète : Gzip
gzip_compress() {
    local input="$1"
    local output="$2"
    
    if gzip -c "$input" > "$output.gz"; then
        echo "Compression gzip réussie: $output.gz"
        return 0
    else
        echo "Erreur lors de la compression gzip" >&2
        return 1
    fi
}

# Implémentation concrète : Bzip2
bzip2_compress() {
    local input="$1"
    local output="$2"
    
    if bzip2 -c "$input" > "$output.bz2"; then
        echo "Compression bzip2 réussie: $output.bz2"
        return 0
    else
        echo "Erreur lors de la compression bzip2" >&2
        return 1
    fi
}

# Implémentation concrète : XZ
xz_compress() {
    local input="$1"
    local output="$2"
    
    if xz -c "$input" > "$output.xz"; then
        echo "Compression xz réussie: $output.xz"
        return 0
    else
        echo "Erreur lors de la compression xz" >&2
        return 1
    fi
}

# Utilisation du pattern Strategy
main() {
    local strategy="${1:-gzip}"
    local input_file="${2:-/tmp/test_file.txt}"
    
    # Création d'un fichier de test
    echo "Ceci est un fichier de test pour la compression." > "$input_file"
    echo "Il contient plusieurs lignes de texte." >> "$input_file"
    echo "Nous allons le compresser avec différentes stratégies." >> "$input_file"
    
    # Application de la stratégie choisie
    if compress_file "$strategy" "$input_file" "${input_file}_compressed"; then
        echo "Compression terminée avec succès"
        
        # Comparaison des tailles
        original_size=$(stat -f%z "$input_file" 2>/dev/null || stat -c%s "$input_file")
        compressed_file="${input_file}_compressed.${strategy}"
        compressed_size=$(stat -f%z "$compressed_file" 2>/dev/null || stat -c%s "$compressed_file")
        
        echo "Taille originale: $original_size octets"
        echo "Taille compressée: $compressed_size octets"
        echo "Ratio: $(echo "scale=2; $compressed_size * 100 / $original_size" | bc)%"
    else
        echo "Erreur lors de la compression"
        exit 1
    fi
}

# Test des différentes stratégies
echo "=== Test Strategy Pattern ==="
main "gzip" "/tmp/strategy_test.txt"
echo "---"
main "bzip2" "/tmp/strategy_test.txt"
echo "---"
main "xz" "/tmp/strategy_test.txt"
```

### 1.3 Strategy avec configuration dynamique

Pour plus de flexibilité, chargez les stratégies dynamiquement :

```bash
#!/bin/bash

# Strategy Pattern avec chargement dynamique des stratégies

declare -A STRATEGIES

# Enregistrement des stratégies disponibles
register_strategy() {
    local name="$1"
    local function_name="$2"
    
    STRATEGIES["$name"]="$function_name"
    echo "Stratégie enregistrée: $name -> $function_name"
}

# Exécution d'une stratégie
execute_strategy() {
    local name="$1"
    shift
    
    local function_name="${STRATEGIES[$name]}"
    
    if [[ -z "$function_name" ]]; then
        echo "Stratégie inconnue: $name" >&2
        echo "Stratégies disponibles: ${!STRATEGIES[*]}" >&2
        return 1
    fi
    
    if declare -f "$function_name" >/dev/null; then
        "$function_name" "$@"
    else
        echo "Fonction de stratégie non définie: $function_name" >&2
        return 1
    fi
}

# Stratégies concrètes
strategy_encrypt_aes() {
    local input="$1"
    local output="$2"
    local key="$3"
    
    if command -v openssl >/dev/null 2>&1; then
        openssl enc -aes-256-cbc -salt -in "$input" -out "$output" -k "$key"
        echo "Chiffrement AES réussi: $output"
    else
        echo "OpenSSL non disponible pour le chiffrement AES" >&2
        return 1
    fi
}

strategy_encrypt_gpg() {
    local input="$1"
    local output="$2"
    local recipient="$3"
    
    if command -v gpg >/dev/null 2>&1; then
        gpg --encrypt --recipient "$recipient" --output "$output" "$input"
        echo "Chiffrement GPG réussi: $output"
    else
        echo "GPG non disponible pour le chiffrement" >&2
        return 1
    fi
}

# Enregistrement des stratégies
register_strategy "aes" "strategy_encrypt_aes"
register_strategy "gpg" "strategy_encrypt_gpg"

# Fonction principale utilisant le pattern
encrypt_file() {
    local method="$1"
    local input_file="$2"
    local output_file="$3"
    shift 3
    
    echo "Chiffrement avec méthode: $method"
    execute_strategy "$method" "$input_file" "$output_file" "$@"
}

# Démonstration
echo "=== Strategy Pattern Dynamique ==="

# Fichier de test
echo "Contenu sensible à chiffrer" > /tmp/secret.txt

# Test AES (nécessite openssl)
encrypt_file "aes" "/tmp/secret.txt" "/tmp/secret.aes" "ma_cle_secrete"

# Test GPG (nécessite gpg et configuration)
# encrypt_file "gpg" "/tmp/secret.txt" "/tmp/secret.gpg" "user@example.com"

# Nettoyage
rm -f /tmp/secret.txt /tmp/secret.aes /tmp/secret.gpg
```

## Section 2 : Pattern Factory (Fabrique)

### 2.1 Le concept Factory

Le pattern Factory définit une interface pour créer des objets, mais laisse aux sous-classes le soin de décider quelle classe instancier. Il délègue la création d'objets à des méthodes spécialisées.

**Analogies pédagogiques :**
- Comme une chaîne de montage automobile qui produit différents modèles selon la demande
- Comme un restaurant qui prépare différents plats selon la commande

### 2.2 Factory pour créer des objets complexes

```bash
#!/bin/bash

# Pattern Factory : Création d'objets complexes (configurations système)

# Classe abstraite : Configuration
Configuration() {
    local self="$1"
    
    # Interface commune
    $self.set_name() { echo "Configuration générique"; }
    $self.get_name() { echo "Configuration générique"; }
    $self.apply() { echo "Application de configuration générique"; }
}

# Classe concrète : Configuration Serveur Web
WebServerConfig() {
    local self="$1"
    
    # Héritage
    Configuration "$self"
    
    # Propriétés spécifiques
    $self.port=80
    $self.document_root="/var/www/html"
    $self.server_name="localhost"
    
    # Méthodes surchargées
    $self.set_name() { $self.server_name="$1"; }
    $self.get_name() { echo "$self.server_name"; }
    
    $self.apply() {
        echo "Configuration serveur web:"
        echo "  Port: $self.port"
        echo "  Document root: $self.document_root"
        echo "  Server name: $self.server_name"
        
        # Application réelle (simulation)
        echo "Création des répertoires..."
        echo "Configuration des permissions..."
        echo "Redémarrage du service..."
    }
}

# Classe concrète : Configuration Base de Données
DatabaseConfig() {
    local self="$1"
    
    Configuration "$self"
    
    # Propriétés spécifiques
    $self.db_name="myapp"
    $self.username="dbuser"
    $self.password=""
    $self.port=3306
    
    $self.get_name() { echo "$self.db_name"; }
    
    $self.apply() {
        echo "Configuration base de données:"
        echo "  Nom: $self.db_name"
        echo "  Utilisateur: $self.username"
        echo "  Port: $self.port"
        
        # Simulation de la configuration
        echo "Création de la base de données..."
        echo "Configuration des utilisateurs..."
        echo "Application des permissions..."
    }
}

# Factory pour créer les configurations
ConfigFactory() {
    create_config() {
        local type="$1"
        local name="$2"
        
        case "$type" in
            webserver)
                WebServerConfig "$name"
                ;;
            database)
                DatabaseConfig "$name"
                ;;
            *)
                echo "Type de configuration inconnu: $type" >&2
                return 1
                ;;
        esac
    }
}

# Utilisation du pattern Factory
echo "=== Pattern Factory ==="

# Création de la factory
ConfigFactory "factory"

# Création des configurations via la factory
factory.create_config "webserver" "mywebserver"
mywebserver.set_name "production.example.com"
mywebserver.port=443

factory.create_config "database" "mydatabase"
mydatabase.db_name="prod_db"
mydatabase.username="prod_user"

# Application des configurations
echo "--- Application Web Server ---"
mywebserver.apply

echo "--- Application Database ---"
mydatabase.apply
```

### 2.3 Factory avec paramètres et validation

Version plus robuste avec validation des paramètres :

```bash
#!/bin/bash

# Factory avancée avec validation et paramètres

# Factory de connexions réseau
NetworkConnectionFactory() {
    local self="$1"
    
    $self.create_connection() {
        local type="$1"
        local params="$2"  # JSON-like string ou tableau
        
        case "$type" in
            ssh)
                create_ssh_connection "$params"
                ;;
            ftp)
                create_ftp_connection "$params"
                ;;
            http)
                create_http_connection "$params"
                ;;
            *)
                echo "Type de connexion inconnu: $type" >&2
                return 1
                ;;
        esac
    }
}

# Créateurs concrets avec validation
create_ssh_connection() {
    local params="$1"
    
    # Parsing basique des paramètres (en réalité, utiliser jq ou similaire)
    local host=$(echo "$params" | grep -o 'host:[^,]*' | cut -d: -f2)
    local port=$(echo "$params" | grep -o 'port:[^,]*' | cut -d: -f2)
    local user=$(echo "$params" | grep -o 'user:[^,]*' | cut -d: -f2)
    
    # Validation
    if [[ -z "$host" ]]; then
        echo "Erreur: host requis pour connexion SSH" >&2
        return 1
    fi
    
    port="${port:-22}"
    user="${user:-$USER}"
    
    # Création de l'objet connexion
    cat << EOF
Connexion SSH créée:
  Host: $host
  Port: $port
  User: $user
  Commande: ssh -p $port $user@$host
EOF
}

create_ftp_connection() {
    local params="$1"
    
    local host=$(echo "$params" | grep -o 'host:[^,]*' | cut -d: -f2)
    local passive=$(echo "$params" | grep -o 'passive:[^,]*' | cut -d: -f2)
    
    if [[ -z "$host" ]]; then
        echo "Erreur: host requis pour connexion FTP" >&2
        return 1
    fi
    
    passive="${passive:-true}"
    
    cat << EOF
Connexion FTP créée:
  Host: $host
  Mode passif: $passive
  Commande: ftp ${passive:+ -p} $host
EOF
}

create_http_connection() {
    local params="$1"
    
    local url=$(echo "$params" | grep -o 'url:[^,]*' | cut -d: -f2)
    local timeout=$(echo "$params" | grep -o 'timeout:[^,]*' | cut -d: -f2)
    
    if [[ -z "$url" ]]; then
        echo "Erreur: URL requise pour connexion HTTP" >&2
        return 1
    fi
    
    timeout="${timeout:-30}"
    
    cat << EOF
Connexion HTTP créée:
  URL: $url
  Timeout: ${timeout}s
  Commande: curl --max-time $timeout "$url"
EOF
}

# Démonstration
echo "=== Factory avec validation ==="

NetworkConnectionFactory "net_factory"

# Créations réussies
echo "--- SSH Connection ---"
net_factory.create_connection "ssh" "host:example.com,port:2222,user:admin"

echo "--- FTP Connection ---"
net_factory.create_connection "ftp" "host:ftp.example.com,passive:true"

echo "--- HTTP Connection ---"
net_factory.create_connection "http" "url:https://api.example.com,timeout:60"

# Tentatives échouées
echo "--- Erreurs de validation ---"
net_factory.create_connection "ssh" "port:2222"  # host manquant
net_factory.create_connection "http" "timeout:30"  # url manquante
```

## Section 3 : Pattern Observer (Observateur)

### 3.1 Le concept Observer

Le pattern Observer définit une dépendance de type un-à-plusieurs entre objets, de sorte que lorsqu'un objet change d'état, tous ses dépendants en sont notifiés et mis à jour automatiquement.

**Analogies pédagogiques :**
- Comme un système de newsletter : les abonnés sont automatiquement informés des nouvelles publications
- Comme les listeners d'événements en programmation GUI

### 3.2 Système d'événements avec Observer

```bash
#!/bin/bash

# Pattern Observer : Système de monitoring et notifications

# Classe Subject (Observable)
Observable() {
    local self="$1"
    
    # Liste des observers
    declare -a $self._observers
    
    $self.add_observer() {
        local observer="$1"
        $self._observers+=("$observer")
    }
    
    $self.remove_observer() {
        local observer="$1"
        local -a new_observers
        
        for obs in "${$self._observers[@]}"; do
            if [[ "$obs" != "$observer" ]]; then
                new_observers+=("$obs")
            fi
        done
        
        $self._observers=("${new_observers[@]}")
    }
    
    $self.notify_observers() {
        local event="$1"
        shift
        
        for observer in "${$self._observers[@]}"; do
            if declare -f "$observer" >/dev/null; then
                "$observer" "$event" "$@"
            fi
        done
    }
}

# Subject concret : Moniteur de système
SystemMonitor() {
    local self="$1"
    
    Observable "$self"
    
    $self.check_cpu() {
        local cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
        
        if (( $(echo "$cpu_usage > 80" | bc -l) )); then
            $self.notify_observers "high_cpu" "$cpu_usage"
        fi
    }
    
    $self.check_memory() {
        local mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
        
        if (( mem_usage > 85 )); then
            $self.notify_observers "high_memory" "$mem_usage"
        fi
    }
    
    $self.check_disk() {
        local disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
        
        if (( disk_usage > 90 )); then
            $self.notify_observers "low_disk_space" "$disk_usage"
        fi
    }
    
    $self.monitor() {
        while true; do
            $self.check_cpu
            $self.check_memory
            $self.check_disk
            sleep 5
        done
    }
}

# Observers concrets
cpu_alert_observer() {
    local event="$1"
    local value="$2"
    
    if [[ "$event" == "high_cpu" ]]; then
        echo "[ALERTE CPU] Utilisation CPU élevée: ${value}%"
        echo "[ACTION] Envoi d'email à l'administrateur..."
        # Simulation d'envoi d'email
        echo "Email envoyé: 'CPU usage is ${value}%'"
    fi
}

memory_alert_observer() {
    local event="$1"
    local value="$2"
    
    if [[ "$event" == "high_memory" ]]; then
        echo "[ALERTE MÉMOIRE] Utilisation mémoire élevée: ${value}%"
        echo "[ACTION] Redémarrage automatique des services non critiques..."
        # Simulation de redémarrage de services
        echo "Services redémarrés pour libérer de la mémoire"
    fi
}

disk_alert_observer() {
    local event="$1"
    local value="$2"
    
    if [[ "$event" == "low_disk_space" ]]; then
        echo "[ALERTE DISQUE] Espace disque faible: ${value}% utilisé"
        echo "[ACTION] Nettoyage automatique des fichiers temporaires..."
        # Simulation de nettoyage
        echo "Fichiers temporaires nettoyés"
    fi
}

# Utilisation du pattern Observer
echo "=== Pattern Observer - Monitoring Système ==="

# Création du moniteur
SystemMonitor "monitor"

# Enregistrement des observers
monitor.add_observer "cpu_alert_observer"
monitor.add_observer "memory_alert_observer"
monitor.add_observer "disk_alert_observer"

echo "Moniteur démarré. Surveillance pendant 15 secondes..."
echo "Vous pouvez simuler des charges élevées dans un autre terminal:"
echo "  - CPU: while true; do true; done"
echo "  - Mémoire: stress --vm 1 --vm-bytes 1G"
echo "  - Disque: dd if=/dev/zero of=/tmp/fill_disk bs=1M count=1000"
echo

# Surveillance pendant 15 secondes (en vrai, tournerait indéfiniment)
timeout 15 bash -c 'monitor.monitor' 2>/dev/null || true

echo "Surveillance terminée."
```

### 3.3 Observer pour les logs et audit

Utilisation du pattern pour centraliser les logs :

```bash
#!/bin/bash

# Pattern Observer pour le système de logging

# Logger principal (Subject)
Logger() {
    local self="$1"
    
    Observable "$self"
    
    $self._level="INFO"
    
    $self.set_level() {
        $self._level="$1"
    }
    
    $self.log() {
        local level="$1"
        local message="$2"
        
        # Mapping des niveaux
        local level_num
        case "$level" in
            DEBUG) level_num=0 ;;
            INFO) level_num=1 ;;
            WARN) level_num=2 ;;
            ERROR) level_num=3 ;;
            FATAL) level_num=4 ;;
            *) level_num=1 ;;
        esac
        
        local current_level_num
        case "$self._level" in
            DEBUG) current_level_num=0 ;;
            INFO) current_level_num=1 ;;
            WARN) current_level_num=2 ;;
            ERROR) current_level_num=3 ;;
            FATAL) current_level_num=4 ;;
            *) current_level_num=1 ;;
        esac
        
        if (( level_num >= current_level_num )); then
            $self.notify_observers "log" "$level" "$message"
        fi
    }
    
    # Méthodes de convenience
    $self.debug() { $self.log "DEBUG" "$1"; }
    $self.info() { $self.log "INFO" "$1"; }
    $self.warn() { $self.log "WARN" "$1"; }
    $self.error() { $self.log "ERROR" "$1"; }
    $self.fatal() { $self.log "FATAL" "$1"; }
}

# Observer : Console Logger
ConsoleLogger() {
    local event="$1"
    local level="$2"
    local message="$3"
    
    if [[ "$event" == "log" ]]; then
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        local color=""
        local reset="\033[0m"
        
        case "$level" in
            DEBUG) color="\033[36m" ;;  # Cyan
            INFO) color="\033[32m" ;;   # Green
            WARN) color="\033[33m" ;;   # Yellow
            ERROR) color="\033[31m" ;;  # Red
            FATAL) color="\033[35m" ;;  # Magenta
        esac
        
        echo -e "${color}[$timestamp] [$level] $message${reset}"
    fi
}

# Observer : File Logger
FileLogger() {
    local log_file="${LOG_FILE:-/tmp/app.log}"
    
    local event="$1"
    local level="$2"
    local message="$3"
    
    if [[ "$event" == "log" ]]; then
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        echo "[$timestamp] [$level] $message" >> "$log_file"
    fi
}

# Observer : Email Notifier (pour erreurs)
EmailNotifier() {
    local event="$1"
    local level="$2"
    local message="$3"
    
    if [[ "$event" == "log" && ("$level" == "ERROR" || "$level" == "FATAL") ]]; then
        echo "[EMAIL] Envoi de notification: $level - $message"
        # Simulation d'envoi d'email
        echo "To: admin@example.com"
        echo "Subject: [$level] Erreur système"
        echo "Message: $message"
        echo "---"
    fi
}

# Utilisation
echo "=== Pattern Observer - Système de Logging ==="

# Configuration du logger
Logger "logger"
logger.set_level "DEBUG"

# Ajout des observers
ConsoleLogger "console_observer"
FileLogger "file_observer"
EmailNotifier "email_observer"

logger.add_observer "console_observer"
logger.add_observer "file_observer"
logger.add_observer "email_observer"

# Tests de logging
logger.debug "Démarrage de l'application"
logger.info "Connexion à la base de données réussie"
logger.warn "Utilisation mémoire élevée: 85%"
logger.error "Échec de connexion au service externe"
logger.fatal "Arrêt critique du système"

echo "Logs enregistrés dans: ${LOG_FILE:-/tmp/app.log}"
```

## Section 4 : Patterns composites et avancés

### 4.1 Pattern Command (Commande)

Encapsule une requête sous forme d'objet, permettant de paramétrer les clients avec différentes requêtes, de mettre en file d'attente les requêtes, et de supporter les opérations réversibles.

```bash
#!/bin/bash

# Pattern Command : Système de tâches différées

# Interface Command
Command() {
    local self="$1"
    
    $self.execute() { echo "Commande générique exécutée"; }
    $self.undo() { echo "Annulation non supportée"; }
}

# Commande concrète : Création de fichier
CreateFileCommand() {
    local self="$1"
    local filepath="$2"
    local content="$3"
    
    Command "$self"
    
    # Stockage pour undo
    $self._filepath="$filepath"
    $self._content="$content"
    $self._existed=false
    
    $self.execute() {
        if [[ -f "$self._filepath" ]]; then
            $self._existed=true
            $self._original_content=$(cat "$self._filepath")
        fi
        
        echo "$self._content" > "$self._filepath"
        echo "Fichier créé: $self._filepath"
    }
    
    $self.undo() {
        if [[ "$self._existed" == "true" ]]; then
            echo "$self._original_content" > "$self._filepath"
            echo "Fichier restauré: $self._filepath"
        else
            rm -f "$self._filepath"
            echo "Fichier supprimé: $self._filepath"
        fi
    }
}

# Commande concrète : Suppression de fichier
DeleteFileCommand() {
    local self="$1"
    local filepath="$2"
    
    Command "$self"
    
    $self._filepath="$filepath"
    $self._backup=""
    
    $self.execute() {
        if [[ -f "$self._filepath" ]]; then
            $self._backup=$(cat "$self._filepath")
            rm "$self._filepath"
            echo "Fichier supprimé: $self._filepath"
        else
            echo "Fichier inexistant: $self._filepath"
        fi
    }
    
    $self.undo() {
        if [[ -n "$self._backup" ]]; then
            echo "$self._backup" > "$self._filepath"
            echo "Fichier restauré: $self._filepath"
        fi
    }
}

# Invoker : Gestionnaire de commandes
CommandManager() {
    local self="$1"
    
    declare -a $self._commands
    declare -a $self._undone
    
    $self.execute() {
        local command="$1"
        
        if declare -f "$command.execute" >/dev/null; then
            $command.execute
            $self._commands+=("$command")
            # Vider la pile undo quand on exécute une nouvelle commande
            $self._undone=()
        else
            echo "Commande invalide: $command" >&2
        fi
    }
    
    $self.undo() {
        if (( ${#$self._commands[@]} > 0 )); then
            local last_command="${$self._commands[-1]}"
            unset $self._commands[-1]
            
            if declare -f "$last_command.undo" >/dev/null; then
                $last_command.undo
                $self._undone+=("$last_command")
            fi
        else
            echo "Aucune commande à annuler" >&2
        fi
    }
    
    $self.redo() {
        if (( ${#$self._undone[@]} > 0 )); then
            local command="${$self._undone[-1]}"
            unset $self._undone[-1]
            $self.execute "$command"
        else
            echo "Aucune commande à refaire" >&2
        fi
    }
}

# Démonstration
echo "=== Pattern Command ==="

CommandManager "cmd_mgr"

# Création de commandes
CreateFileCommand "create_cmd1" "/tmp/test1.txt" "Contenu du fichier 1"
CreateFileCommand "create_cmd2" "/tmp/test2.txt" "Contenu du fichier 2"
DeleteFileCommand "delete_cmd" "/tmp/test1.txt"

# Exécution
echo "--- Exécution des commandes ---"
cmd_mgr.execute "create_cmd1"
cmd_mgr.execute "create_cmd2"
ls -la /tmp/test*.txt

cmd_mgr.execute "delete_cmd"
ls -la /tmp/test*.txt

# Undo
echo "--- Annulation ---"
cmd_mgr.undo
ls -la /tmp/test*.txt

cmd_mgr.undo
ls -la /tmp/test*.txt

# Nettoyage
rm -f /tmp/test*.txt
```

### 4.2 Pattern Singleton (Singleton)

Assure qu'une classe n'a qu'une seule instance et fournit un point d'accès global à cette instance.

```bash
#!/bin/bash

# Pattern Singleton : Configuration globale unique

# Singleton : DatabaseConnection
DatabaseConnection() {
    # Vérifier si l'instance existe déjà
    if [[ -n "${DATABASE_INSTANCE:-}" ]]; then
        echo "Instance DatabaseConnection déjà créée" >&2
        return 1
    fi
    
    local self="$1"
    DATABASE_INSTANCE="$self"
    
    # Propriétés privées
    $self._host="localhost"
    $self._port=3306
    $self._database="myapp"
    $self._connected=false
    
    # Méthodes publiques
    $self.connect() {
        if [[ "$self._connected" == "true" ]]; then
            echo "Déjà connecté à la base de données"
            return 0
        fi
        
        echo "Connexion à $self._database sur $self._host:$self._port..."
        # Simulation de connexion
        sleep 0.5
        
        $self._connected=true
        echo "Connexion établie"
    }
    
    $self.disconnect() {
        if [[ "$self._connected" == "true" ]]; then
            echo "Déconnexion de la base de données..."
            $self._connected=false
            echo "Déconnexion terminée"
        else
            echo "Pas de connexion active"
        fi
    }
    
    $self.query() {
        local sql="$1"
        
        if [[ "$self._connected" != "true" ]]; then
            echo "Erreur: pas connecté à la base de données" >&2
            return 1
        fi
        
        echo "Exécution: $sql"
        # Simulation de requête
        case "$sql" in
            "SELECT * FROM users")
                echo "Résultat: 150 utilisateurs trouvés"
                ;;
            "SELECT COUNT(*) FROM orders")
                echo "Résultat: 1250 commandes"
                ;;
            *)
                echo "Résultat: Requête exécutée avec succès"
                ;;
        esac
    }
    
    $self.get_instance() {
        echo "$self"
    }
}

# Fonction pour obtenir l'instance singleton
get_database_connection() {
    if [[ -z "${DATABASE_INSTANCE:-}" ]]; then
        DatabaseConnection "db_conn"
    fi
    
    echo "$DATABASE_INSTANCE"
}

# Démonstration
echo "=== Pattern Singleton ==="

# Tentative de création multiple (seule la première réussit)
echo "--- Création instances ---"
db1=$(get_database_connection)
echo "Instance 1: $db1"

db2=$(get_database_connection)
echo "Instance 2: $db2"

# Vérification qu'il s'agit de la même instance
if [[ "$db1" == "$db2" ]]; then
    echo "✓ Singleton fonctionne: même instance"
else
    echo "✗ Singleton défaillant: instances différentes"
fi

# Utilisation de l'instance
echo "--- Utilisation ---"
$db1.connect
$db1.query "SELECT * FROM users"
$db1.query "SELECT COUNT(*) FROM orders"
$db1.disconnect

# Tentative d'accès via la deuxième référence
$db2.connect  # Devrait indiquer déjà connecté
```

### 4.3 Pattern Decorator (Décorateur)

Attache dynamiquement des responsabilités supplémentaires à un objet. Les décorateurs fournissent une alternative flexible à l'héritage pour étendre les fonctionnalités.

```bash
#!/bin/bash

# Pattern Decorator : Logging et monitoring des fonctions

# Composant de base : DataProcessor
DataProcessor() {
    local self="$1"
    
    $self.process() {
        local data="$1"
        echo "Traitement de base: $data"
        # Traitement simulé
        echo "${data}_processed"
    }
}

# Décorateur de base
DataProcessorDecorator() {
    local self="$1"
    local wrapped="$2"
    
    $self._wrapped="$wrapped"
    
    # Délégation par défaut
    $self.process() {
        $self._wrapped.process "$1"
    }
}

# Décorateur concret : Logging
LoggingDecorator() {
    local self="$1"
    local wrapped="$2"
    
    DataProcessorDecorator "$self" "$wrapped"
    
    $self.process() {
        local data="$1"
        local timestamp=$(date '+%H:%M:%S')
        
        echo "[$timestamp] [LOG] Début traitement: $data"
        local result
        result=$($self._wrapped.process "$data")
        echo "[$timestamp] [LOG] Fin traitement: $result"
        
        echo "$result"
    }
}

# Décorateur concret : Timing
TimingDecorator() {
    local self="$1"
    local wrapped="$2"
    
    DataProcessorDecorator "$self" "$wrapped"
    
    $self.process() {
        local data="$1"
        local start_time=$(date +%s.%N)
        
        local result
        result=$($self._wrapped.process "$data")
        
        local end_time=$(date +%s.%N)
        local duration=$(echo "$end_time - $start_time" | bc)
        
        echo "[TIMING] Traitement terminé en ${duration}s"
        echo "$result"
    }
}

# Décorateur concret : Validation
ValidationDecorator() {
    local self="$1"
    local wrapped="$2"
    
    DataProcessorDecorator "$self" "$wrapped"
    
    $self.process() {
        local data="$1"
        
        # Validation d'entrée
        if [[ -z "$data" ]]; then
            echo "[VALIDATION] Erreur: données vides" >&2
            return 1
        fi
        
        if [[ ${#data} -gt 100 ]]; then
            echo "[VALIDATION] Erreur: données trop longues (${#data} caractères)" >&2
            return 1
        fi
        
        echo "[VALIDATION] Données valides"
        local result
        result=$($self._wrapped.process "$data")
        
        # Validation de sortie
        if [[ "$result" == *"_processed" ]]; then
            echo "[VALIDATION] Sortie valide"
        else
            echo "[VALIDATION] Attention: format de sortie inattendu" >&2
        fi
        
        echo "$result"
    }
}

# Utilisation avec chaînage de décorateurs
echo "=== Pattern Decorator ==="

# Processeur de base
DataProcessor "base_processor"

# Application des décorateurs (chaînage)
ValidationDecorator "validated_processor" "base_processor"
TimingDecorator "timed_processor" "validated_processor"
LoggingDecorator "logged_processor" "timed_processor"

# Tests
echo "--- Test données valides ---"
logged_processor.process "test_data"

echo "--- Test données vides ---"
logged_processor.process ""

echo "--- Test données longues ---"
logged_processor.process "$(printf 'x%.0s' {1..150})"
```

## Section 5 : Patterns pour la gestion d'erreurs et la robustesse

### 5.1 Pattern Circuit Breaker (Disjoncteur)

Le Circuit Breaker empêche un système de faire des appels répétés vers un service défaillant, permettant au service de récupérer.

```bash
#!/bin/bash

# Pattern Circuit Breaker : Protection contre les services défaillants

CircuitBreaker() {
    local self="$1"
    local failure_threshold="${2:-3}"
    local recovery_timeout="${3:-60}"
    
    $self._state="CLOSED"  # CLOSED, OPEN, HALF_OPEN
    $self._failure_count=0
    $self._last_failure_time=0
    $self._failure_threshold="$failure_threshold"
    $self._recovery_timeout="$recovery_timeout"
    
    $self.call() {
        local service_call="$1"
        local current_time=$(date +%s)
        
        case "$self._state" in
            OPEN)
                # Vérifier si on peut passer en HALF_OPEN
                if (( current_time - $self._last_failure_time >= $self._recovery_timeout )); then
                    $self._state="HALF_OPEN"
                    echo "[CIRCUIT] Passage en HALF_OPEN - test de récupération"
                else
                    echo "[CIRCUIT] OPEN - appel rejeté (timeout: $(( $self._recovery_timeout - (current_time - $self._last_failure_time) ))s)"
                    return 1
                fi
                ;;
        esac
        
        # Tentative d'appel
        if eval "$service_call"; then
            $self._on_success
        else
            $self._on_failure
            return 1
        fi
    }
    
    $self._on_success() {
        $self._failure_count=0
        
        if [[ "$self._state" == "HALF_OPEN" ]]; then
            $self._state="CLOSED"
            echo "[CIRCUIT] Récupération réussie - retour à CLOSED"
        fi
    }
    
    $self._on_failure() {
        (( $self._failure_count++ ))
        $self._last_failure_time=$(date +%s)
        
        if (( $self._failure_count >= $self._failure_threshold )); then
            $self._state="OPEN"
            echo "[CIRCUIT] Passage en OPEN après $self._failure_count échecs"
        else
            echo "[CIRCUIT] Échec $self._failure_count/$self._failure_threshold"
        fi
    }
    
    $self.get_state() {
        echo "$self._state"
    }
}

# Service simulé (peut échouer)
unreliable_service() {
    local failure_rate="${UNRELIABLE_FAILURE_RATE:-0.3}"
    local random=$(awk 'BEGIN{srand(); print rand()}')
    
    if (( $(echo "$random < $failure_rate" | bc -l) )); then
        echo "[SERVICE] Échec simulé"
        return 1
    else
        echo "[SERVICE] Succès"
        return 0
    fi
}

# Démonstration
echo "=== Pattern Circuit Breaker ==="

CircuitBreaker "circuit_breaker" 3 10  # 3 échecs max, timeout 10s

echo "--- Test avec service défaillant ---"
UNRELIABLE_FAILURE_RATE=0.8  # 80% de chance d'échec

for i in {1..8}; do
    echo "Appel $i:"
    circuit_breaker.call "unreliable_service"
    sleep 1
done

echo "--- Attente de récupération ---"
echo "Attente de 11 secondes pour permettre la récupération..."
sleep 11

echo "--- Test de récupération ---"
UNRELIABLE_FAILURE_RATE=0.1  # Maintenant 10% d'échec seulement

for i in {1..3}; do
    echo "Appel $i:"
    circuit_breaker.call "unreliable_service"
    sleep 1
done
```

### 5.2 Pattern Retry avec Backoff (Réessai avec recul)

Implémentation d'un mécanisme de réessai intelligent avec délai croissant.

```bash
#!/bin/bash

# Pattern Retry avec Exponential Backoff

RetryWithBackoff() {
    local self="$1"
    local max_attempts="${2:-3}"
    local base_delay="${3:-1}"
    local max_delay="${4:-60}"
    
    $self.execute() {
        local command="$1"
        local attempt=1
        
        while (( attempt <= max_attempts )); do
            echo "[RETRY] Tentative $attempt/$max_attempts"
            
            if eval "$command"; then
                echo "[RETRY] Succès à la tentative $attempt"
                return 0
            fi
            
            if (( attempt < max_attempts )); then
                local delay=$(( base_delay * (2 ** (attempt - 1)) ))
                
                # Limitation du délai maximum
                if (( delay > max_delay )); then
                    delay="$max_delay"
                fi
                
                # Ajout de jitter pour éviter la synchronisation
                local jitter=$(( RANDOM % delay / 4 ))
                local actual_delay=$(( delay + jitter ))
                
                echo "[RETRY] Échec, attente de ${actual_delay}s avant nouvelle tentative"
                sleep "$actual_delay"
            fi
            
            ((attempt++))
        done
        
        echo "[RETRY] Échec définitif après $max_attempts tentatives"
        return 1
    }
}

# Service avec taux de réussite variable
flaky_service() {
    local attempt="$1"
    # Réussit seulement à la 3ème tentative ou plus
    if (( attempt >= 3 )); then
        echo "[SERVICE] Succès à la tentative $attempt"
        return 0
    else
        echo "[SERVICE] Échec à la tentative $attempt"
        return 1
    fi
}

# Démonstration
echo "=== Pattern Retry avec Backoff ==="

RetryWithBackoff "retry_handler" 5 1 30

# Test avec service qui échoue puis réussit
echo "--- Test service instable ---"
attempt=1
retry_handler.execute "
    flaky_service \$((attempt++))
    exit \$?
"
```

## Conclusion : L'architecture au service de la robustesse

Les patterns de programmation apportent une structure éprouvée à vos scripts Bash, transformant du code procédural en architectures maintenables et extensibles. Comme les grandes cathédrales gothiques qui ont résisté aux siècles grâce à leurs arcs-boutants et leurs voûtes, vos scripts gagneront en robustesse grâce à ces patterns.

**Points clés à retenir :**

1. **Strategy** permet de rendre vos algorithmes interchangeables et extensibles
2. **Factory** centralise et valide la création d'objets complexes
3. **Observer** crée des systèmes de notification découplés et maintenables
4. **Command** encapsule les actions pour permettre undo/redo et queuing
5. **Decorator** étend les fonctionnalités de manière modulaire
6. **Circuit Breaker** protège contre les cascades de pannes
7. **Retry avec Backoff** gère élégamment les opérations transitoirement défaillantes

Dans le chapitre suivant, nous explorerons les techniques de métaprogrammation en Bash, pour créer des scripts qui s'écrivent eux-mêmes et s'adaptent dynamiquement à leur environnement.

---

**Exercice pratique :** Implémentez un système de sauvegarde utilisant plusieurs patterns :
- **Strategy** pour différents types de sauvegarde (complète, incrémentielle, compressée)
- **Observer** pour notifier l'administrateur des progrès et erreurs
- **Command** pour permettre l'annulation et la reprise des sauvegardes
- **Circuit Breaker** pour gérer les échecs de connexion réseau

**Réflexion :** Comment adapteriez-vous ces patterns pour créer un framework de test unitaire complet en Bash ?
