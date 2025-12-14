# Chapitre 86 - Sécurité des scripts shell et durcissement

> "La sécurité n'est pas un produit, mais un processus." - Bruce Schneier

## Introduction : De l'artisanat à la forteresse numérique

Imaginez-vous architecte d'une banque suisse : chaque porte, chaque serrure, chaque système de surveillance doit être infaillible. Les scripts Bash en production sont ces banques : ils contiennent des données sensibles, exécutent des opérations critiques, et doivent résister aux attaques. Dans ce chapitre, nous allons transformer vos scripts de simples outils en forteresses imprenables.

Nous explorerons les techniques de sécurité avancées : validation d'entrée impitoyable, pratiques de codage sécurisé, évaluation des vulnérabilités, et stratégies de durcissement pour créer des scripts dignes de confiance.

## Section 1 : Validation d'entrée et assainissement

### 1.1 Les dangers de l'entrée utilisateur

L'entrée utilisateur est le vecteur d'attaque numéro un :

```bash
#!/bin/bash

# Les dangers de l'entrée utilisateur
echo "=== Les dangers de l'entrée utilisateur ==="

# Fonction démontrant les vulnérabilités classiques
demonstrate_vulnerabilities() {
    echo "--- Démonstration des vulnérabilités ---"
    
    # Vulnérabilité 1: Injection de commande
    vulnerable_command_injection() {
        local user_input="$1"
        echo "Exécution de: ls $user_input"
        
        # DANGEREUX: injection directe
        eval "ls $user_input"
    }
    
    # Vulnérabilité 2: Inclusion de fichier
    vulnerable_file_inclusion() {
        local filename="$1"
        echo "Lecture de: $filename"
        
        # DANGEREUX: inclusion arbitraire
        cat "$filename"
    }
    
    # Vulnérabilité 3: Dépassement de tampon (simulé)
    vulnerable_buffer_overflow() {
        local input="$1"
        
        # Simulation de limitation insuffisante
        if (( ${#input} > 10 )); then
            echo "Input trop long: ${#input} caractères"
            return 1
        fi
        
        echo "Input accepté: $input"
    }
    
    echo "Test d'injection de commande:"
    echo "Entrée malicieuse: '; rm -f /tmp/test.txt; echo'"
    vulnerable_command_injection "'; rm -f /tmp/test.txt; echo 'HACKED'"
    
    echo
    echo "Test d'inclusion de fichier:"
    echo "Fichier malicieux: /etc/passwd"
    vulnerable_file_inclusion "/etc/passwd"
    
    echo
    echo "Test de dépassement de tampon:"
    vulnerable_buffer_overflow "Ceci est une entrée très très très très longue qui dépasse largement la limite de 10 caractères"
}

# Fonction de validation sécurisée
secure_input_validation() {
    local input="$1"
    local type="${2:-string}"
    local max_length="${3:-100}"
    
    echo "--- Validation sécurisée: $input (type: $type, max: $max_length) ---"
    
    # 1. Vérification de longueur
    if (( ${#input} > max_length )); then
        echo "❌ Longueur excessive: ${#input} > $max_length"
        return 1
    fi
    
    # 2. Validation selon le type
    case "$type" in
        integer)
            if ! [[ "$input" =~ ^-?[0-9]+$ ]]; then
                echo "❌ Pas un entier valide"
                return 1
            fi
            # Vérification des limites
            if (( input < -1000 || input > 1000 )); then
                echo "❌ Entier hors limites [-1000, 1000]"
                return 1
            fi
            ;;
        float)
            if ! [[ "$input" =~ ^-?[0-9]*\.?[0-9]+$ ]]; then
                echo "❌ Pas un nombre flottant valide"
                return 1
            fi
            ;;
        email)
            if ! [[ "$input" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
                echo "❌ Format d'email invalide"
                return 1
            fi
            ;;
        path)
            # Vérification des caractères dangereux
            if [[ "$input" =~ [\`\$\(\)\<\>\|\;] ]]; then
                echo "❌ Caractères dangereux dans le chemin"
                return 1
            fi
            # Vérification des chemins absolus seulement
            if [[ "$input" != /* ]]; then
                echo "❌ Chemin relatif non autorisé"
                return 1
            fi
            ;;
        string)
            # Filtrage des caractères de contrôle
            if [[ "$input" =~ [[:cntrl:]] ]]; then
                echo "❌ Caractères de contrôle interdits"
                return 1
            fi
            ;;
        *)
            echo "❌ Type de validation inconnu: $type"
            return 1
            ;;
    esac
    
    echo "✓ Validation réussie"
    return 0
}

# Fonction d'assainissement d'entrée
sanitize_input() {
    local input="$1"
    local type="${2:-string}"
    
    echo "--- Assainissement: '$input' (type: $type) ---"
    
    case "$type" in
        sql)
            # Échappement SQL basique (pour démonstration)
            local sanitized="${input//\'/\'\'}"  # Échapper les quotes
            sanitized="${sanitized//;/}"        # Supprimer les points-virgules
            echo "Assaini SQL: '$sanitized'"
            ;;
        html)
            # Échappement HTML basique
            local sanitized="$input"
            sanitized="${sanitized//&/&amp;}"
            sanitized="${sanitized//</&lt;}"
            sanitized="${sanitized//>/&gt;}"
            sanitized="${sanitized//\"/&quot;}"
            echo "Assaini HTML: '$sanitized'"
            ;;
        shell)
            # Échappement shell
            local sanitized="$input"
            sanitized="${sanitized//\\/\\\\}"   # Échapper les backslashes
            sanitized="${sanitized//\'/\\\'}"   # Échapper les quotes simples
            sanitized="${sanitized//\"/\\\"}"   # Échapper les quotes doubles
            sanitized="${sanitized//\`/\\\`}"   # Échapper les backticks
            echo "Assaini shell: '$sanitized'"
            ;;
        filename)
            # Nettoyage pour noms de fichiers
            local sanitized="${input//[^a-zA-Z0-9._-]/_}"
            # Supprimer les espaces au début/fin
            sanitized="${sanitized#"${sanitized%%[![:space:]]*}"}"
            sanitized="${sanitized%"${sanitized##*[![:space:]]}"}"
            echo "Assaini filename: '$sanitized'"
            ;;
    esac
}

# Tests de sécurité
echo "--- Tests de vulnérabilités (ÉDUCATIFS) ---"
demonstrate_vulnerabilities

echo
echo "--- Tests de validation sécurisée ---"

# Tests de validation
secure_input_validation "42" "integer" 10
secure_input_validation "3.14" "float" 10
secure_input_validation "user@example.com" "email" 50
secure_input_validation "/etc/passwd" "path" 50
secure_input_validation "Hello World!" "string" 50

echo
echo "--- Tests d'assainissement ---"

# Tests d'assainissement
sanitize_input "'; DROP TABLE users; --" "sql"
sanitize_input "<script>alert('XSS')</script>" "html"
sanitize_input 'rm -rf /; echo "Hacked"' "shell"
sanitize_input "file with spaces & special chars!.txt" "filename"
```

### 1.2 Frameworks de validation avancés

Systèmes de validation structurés et réutilisables :

```bash
#!/bin/bash

# Frameworks de validation avancés
echo "=== Frameworks de validation avancés ==="

# Framework de validation objet
ValidationFramework() {
    local self="$1"
    
    # Règles de validation
    declare -A $self._rules
    declare -a $self._errors
    
    $self.add_rule() {
        local field="$1"
        local rule="$2"
        local message="$3"
        
        $self._rules["$field"]="$rule"
        $self._rules["${field}_message"]="$message"
    }
    
    $self.validate() {
        local -n data_ref="$1"  # Passage par référence
        $self._errors=()
        
        local valid=true
        
        for field in "${!data_ref[@]}"; do
            local value="${data_ref[$field]}"
            local rule="${$self._rules[$field]}"
            
            if [[ -n "$rule" ]]; then
                if ! $self._check_rule "$value" "$rule"; then
                    local message="${$self._rules[${field}_message]}"
                    $self._errors+=("${field}: ${message:-Règle '$rule' violée}")
                    valid=false
                fi
            fi
        done
        
        return $(( ! valid ))
    }
    
    $self._check_rule() {
        local value="$1"
        local rule="$2"
        
        case "$rule" in
            required)
                [[ -n "$value" ]]
                ;;
            email)
                [[ "$value" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
                ;;
            integer)
                [[ "$value" =~ ^-?[0-9]+$ ]]
                ;;
            positive_integer)
                [[ "$value" =~ ^[0-9]+$ ]] && (( value > 0 ))
                ;;
            alphanumeric)
                [[ "$value" =~ ^[a-zA-Z0-9]+$ ]]
                ;;
            path_exists)
                [[ -e "$value" ]]
                ;;
            max_length:*)
                local max_len="${rule#*:}"
                (( ${#value} <= max_len ))
                ;;
            min_length:*)
                local min_len="${rule#*:}"
                (( ${#value} >= min_len ))
                ;;
            range:*)
                local min max
                IFS='-' read min max <<< "${rule#*:}"
                [[ "$value" =~ ^-?[0-9]+$ ]] && (( value >= min && value <= max ))
                ;;
            *)
                echo "Règle inconnue: $rule" >&2
                false
                ;;
        esac
    }
    
    $self.get_errors() {
        printf '%s\n' "${$self._errors[@]}"
    }
    
    $self.has_errors() {
        (( ${#$self._errors[@]} > 0 ))
    }
}

# Fonction de validation de formulaire utilisateur
validate_user_form() {
    echo "--- Validation de formulaire utilisateur ---"
    
    # Création du validateur
    ValidationFramework "user_validator"
    
    # Ajout des règles
    user_validator.add_rule "username" "required" "Nom d'utilisateur requis"
    user_validator.add_rule "username" "alphanumeric" "Nom d'utilisateur doit être alphanumérique"
    user_validator.add_rule "username" "min_length:3" "Nom d'utilisateur trop court (min 3)"
    user_validator.add_rule "username" "max_length:20" "Nom d'utilisateur trop long (max 20)"
    
    user_validator.add_rule "email" "required" "Email requis"
    user_validator.add_rule "email" "email" "Format d'email invalide"
    
    user_validator.add_rule "age" "required" "Âge requis"
    user_validator.add_rule "age" "integer" "Âge doit être un nombre entier"
    user_validator.add_rule "age" "range:13-120" "Âge doit être entre 13 et 120 ans"
    
    user_validator.add_rule "config_file" "path_exists" "Fichier de configuration inexistant"
    
    # Données de test
    declare -A user_data=(
        ["username"]="john_doe_123"
        ["email"]="john.doe@example.com"
        ["age"]="25"
        ["config_file"]="/etc/passwd"
    )
    
    echo "Données à valider:"
    for key in "${!user_data[@]}"; do
        echo "  $key: '${user_data[$key]}'"
    done
    
    echo
    echo "Validation:"
    
    if user_validator.validate user_data; then
        echo "✓ Toutes les validations réussies"
    else
        echo "❌ Erreurs de validation:"
        user_validator.get_errors | sed 's/^/  /'
    fi
}

# Fonction de validation de configuration système
validate_system_config() {
    echo "--- Validation de configuration système ---"
    
    ValidationFramework "config_validator"
    
    # Règles pour la configuration
    config_validator.add_rule "port" "required" "Port requis"
    config_validator.add_rule "port" "positive_integer" "Port doit être un entier positif"
    config_validator.add_rule "port" "range:1024-65535" "Port doit être entre 1024 et 65535"
    
    config_validator.add_rule "max_connections" "integer" "Max connections doit être un entier"
    config_validator.add_rule "max_connections" "range:1-10000" "Max connections entre 1 et 10000"
    
    config_validator.add_rule "log_level" "required" "Niveau de log requis"
    
    # Données de configuration
    declare -A config_data=(
        ["port"]="8080"
        ["max_connections"]="100"
        ["log_level"]="INFO"
        ["timeout"]="30"
    )
    
    echo "Configuration à valider:"
    for key in "${!config_data[@]}"; do
        echo "  $key: '${config_data[$key]}'"
    done
    
    echo
    echo "Validation:"
    
    if config_validator.validate config_data; then
        echo "✓ Configuration valide"
    else
        echo "❌ Erreurs de configuration:"
        config_validator.get_errors | sed 's/^/  /'
    fi
}

# Fonction de validation avec transformation
validate_and_transform() {
    echo "--- Validation avec transformation ---"
    
    ValidationFramework "transform_validator"
    
    # Données d'entrée "brutes"
    declare -A raw_data=(
        ["name"]="  john DOE  "
        ["email"]="John.Doe@EXAMPLE.COM"
        ["phone"]="(555) 123-4567"
        ["date"]="2024/01/15"
    )
    
    echo "Données brutes:"
    for key in "${!raw_data[@]}"; do
        echo "  $key: '${raw_data[$key]}'"
    done
    
    # Transformation et validation
    declare -A clean_data
    
    # Name: trim et capitalize
    clean_data["name"]="$(echo "${raw_data[name]}" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//' | awk '{for(i=1;i<=NF;i++) $i=toupper(substr($i,1,1)) tolower(substr($i,2));}1')"
    
    # Email: lowercase
    clean_data["email"]="$(echo "${raw_data[email]}" | tr '[:upper:]' '[:lower:]')"
    
    # Phone: digits only
    clean_data["phone"]="$(echo "${raw_data[phone]}" | sed 's/[^0-9]//g')"
    
    # Date: standardize format
    clean_data["date"]="$(echo "${raw_data[date]}" | sed 's/\//-/g')"
    
    echo
    echo "Données transformées:"
    for key in "${!clean_data[@]}"; do
        echo "  $key: '${clean_data[$key]}'"
    done
    
    # Validation des données transformées
    transform_validator.add_rule "name" "required" "Nom requis"
    transform_validator.add_rule "name" "min_length:2" "Nom trop court"
    transform_validator.add_rule "email" "email" "Email invalide"
    transform_validator.add_rule "phone" "required" "Téléphone requis"
    transform_validator.add_rule "phone" "min_length:10" "Numéro trop court"
    
    echo
    echo "Validation des données transformées:"
    
    if transform_validator.validate clean_data; then
        echo "✓ Données valides après transformation"
    else
        echo "❌ Erreurs après transformation:"
        transform_validator.get_errors | sed 's/^/  /'
    fi
}

# Tests du framework
validate_user_form

echo
validate_system_config

echo
validate_and_transform
```

## Section 2 : Pratiques de codage sécurisé

### 2.1 Le principe du moindre privilège

Réduire les permissions au minimum nécessaire :

```bash
#!/bin/bash

# Le principe du moindre privilège
echo "=== Le principe du moindre privilège ==="

# Fonction de gestion des privilèges
PrivilegeManager() {
    local self="$1"
    
    $self._original_uid="$EUID"
    $self._original_gid="$EGID"
    declare -a $self._capabilities
    
    $self.drop_privileges() {
        local target_user="${1:-nobody}"
        local target_group="${2:-nogroup}"
        
        echo "Abandon des privilèges (vers $target_user:$target_group)"
        
        # Vérification que nous avons les droits
        if (( EUID == 0 )); then
            # Changement d'utilisateur/groupe
            if ! chown "$target_user:$target_group" "$$" 2>/dev/null; then
                echo "⚠️ Impossible de changer les permissions du processus"
            fi
            
            # Drop des groupes supplémentaires
            if command -v sg >/dev/null 2>&1; then
                sg "$target_group" -c "id" >/dev/null 2>&1
            fi
            
            echo "✓ Privilèges abandonnés"
        else
            echo "⚠️ Pas de privilèges root à abandonner"
        fi
    }
    
    $self.restrict_capabilities() {
        echo "Restriction des capacités système"
        
        # Désactivation des commandes dangereuses
        unset -f sudo su chpasswd usermod
        
        # Restriction du PATH
        export PATH="/bin:/usr/bin:/usr/local/bin"
        
        # Désactivation des variables d'environnement dangereuses
        unset -v LD_PRELOAD LD_LIBRARY_PATH
        
        echo "✓ Capacités restreintes"
    }
    
    $self.validate_operation() {
        local operation="$1"
        local resource="$2"
        
        echo "Validation de l'opération: $operation sur $resource"
        
        case "$operation" in
            read)
                if [[ -r "$resource" ]]; then
                    echo "✓ Permission de lecture accordée"
                    return 0
                fi
                ;;
            write)
                if [[ -w "$resource" ]]; then
                    echo "✓ Permission d'écriture accordée"
                    return 0
                fi
                ;;
            execute)
                if [[ -x "$resource" ]]; then
                    echo "✓ Permission d'exécution accordée"
                    return 0
                fi
                ;;
        esac
        
        echo "❌ Permission refusée"
        return 1
    }
    
    $self.audit_log() {
        local action="$1"
        local details="$2"
        
        local timestamp
        timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        local user
        user=$(whoami)
        local pid
        pid=$$
        
        echo "[$timestamp] [$user:$pid] $action: $details" >> /var/log/script_audit.log
    }
}

# Fonction de bac à sable sécurisé
secure_sandbox() {
    local script_content="$1"
    local allowed_commands="${2:-echo cat grep wc}"
    
    echo "--- Bac à sable sécurisé ---"
    
    # Création d'un environnement isolé
    local sandbox_dir
    sandbox_dir=$(mktemp -d "/tmp/sandbox_XXXXXX")
    local sandbox_script="$sandbox_dir/script.sh"
    
    # Écriture du script dans le sandbox
    echo "#!/bin/bash" > "$sandbox_script"
    echo "export PATH='/bin:/usr/bin'" >> "$sandbox_script"
    echo "$script_content" >> "$sandbox_script"
    
    chmod 755 "$sandbox_script"
    
    echo "Script sandbox créé: $sandbox_script"
    
    # Exécution dans un sous-shell restreint
    (
        # Restrictions
        ulimit -t 10  # CPU time limit: 10 seconds
        ulimit -v 100000  # Virtual memory: 100MB
        ulimit -f 1000   # File size: 1MB
        
        # Variables d'environnement nettoyées
        unset -v SHELL ENV BASH_ENV
        
        # Exécution
        echo "Exécution en mode sandbox..."
        if timeout 30 bash "$sandbox_script" 2>&1; then
            echo "✓ Exécution sandbox réussie"
        else
            echo "❌ Échec ou timeout de l'exécution sandbox"
        fi
    )
    
    # Nettoyage
    rm -rf "$sandbox_dir"
}

# Fonction de vérification d'intégrité
integrity_check() {
    local file="$1"
    local expected_hash="$2"
    
    echo "--- Vérification d'intégrité: $file ---"
    
    if [[ ! -f "$file" ]]; then
        echo "❌ Fichier inexistant"
        return 1
    fi
    
    local actual_hash
    actual_hash=$(sha256sum "$file" 2>/dev/null | cut -d' ' -f1)
    
    if [[ "$actual_hash" == "$expected_hash" ]]; then
        echo "✓ Intégrité vérifiée"
        return 0
    else
        echo "❌ Intégrité compromise"
        echo "  Attendu: $expected_hash"
        echo "  Obtenu:  $actual_hash"
        return 1
    fi
}

# Démonstration des pratiques sécurisées
echo "--- Gestion des privilèges ---"

PrivilegeManager "priv_mgr"

if (( EUID == 0 )); then
    priv_mgr.drop_privileges "nobody" "nogroup"
fi

priv_mgr.restrict_capabilities

echo
echo "--- Validation d'opérations ---"
priv_mgr.validate_operation "read" "/etc/passwd"
priv_mgr.validate_operation "write" "/tmp/test.txt"

echo
echo "--- Test de bac à sable ---"
secure_sandbox "
echo 'Script sandbox en exécution'
echo 'PID:' \$\$
echo 'Utilisateur:' \$(whoami)
echo 'Mémoire disponible:' \$(free -h | grep Mem | awk '{print \$7}')
"

echo
echo "--- Vérification d'intégrité ---"

# Création d'un fichier de test
echo "Contenu de test pour intégrité" > /tmp/integrity_test.txt
test_hash=$(sha256sum /tmp/integrity_test.txt | cut -d' ' -f1)

integrity_check "/tmp/integrity_test.txt" "$test_hash"

# Modification du fichier
echo "Contenu modifié" >> /tmp/integrity_test.txt
integrity_check "/tmp/integrity_test.txt" "$test_hash"

# Nettoyage
rm -f /tmp/integrity_test.txt
```

### 2.2 Gestion sécurisée des secrets

Protection des informations sensibles :

```bash
#!/bin/bash

# Gestion sécurisée des secrets
echo "=== Gestion sécurisée des secrets ==="

# Gestionnaire de secrets
SecretManager() {
    local self="$1"
    local vault_file="${2:-/tmp/secrets.vault}"
    
    $self._vault="$vault_file"
    declare -A $self._secrets
    
    # Initialisation du coffre-fort
    $self._init_vault() {
        if [[ ! -f "$self._vault" ]]; then
            touch "$self._vault"
            chmod 600 "$self._vault"
        fi
    }
    
    # Chiffrement d'un secret
    $self.encrypt_secret() {
        local plain_text="$1"
        
        # Utilisation de openssl pour le chiffrement
        if command -v openssl >/dev/null 2>&1; then
            echo "$plain_text" | openssl enc -aes-256-cbc -salt -pbkdf2 -pass pass:"$SECRET_KEY" 2>/dev/null | base64 -w 0
        else
            # Fallback simple (non sécurisé pour la production)
            echo "$plain_text" | base64
        fi
    }
    
    # Déchiffrement d'un secret
    $self.decrypt_secret() {
        local encrypted_text="$1"
        
        if command -v openssl >/dev/null 2>&1; then
            echo "$encrypted_text" | base64 -d | openssl enc -d -aes-256-cbc -pbkdf2 -pass pass:"$SECRET_KEY" 2>/dev/null
        else
            echo "$encrypted_text" | base64 -d
        fi
    }
    
    # Stockage d'un secret
    $self.store_secret() {
        local key="$1"
        local value="$2"
        
        $self._init_vault
        
        local encrypted_value
        encrypted_value=$($self.encrypt_secret "$value")
        
        # Stockage dans le fichier
        echo "${key}:${encrypted_value}" >> "$self._vault"
        
        # Mise en cache mémoire (durée de vie limitée)
        $self._secrets["$key"]="$value"
        
        echo "✓ Secret '$key' stocké"
    }
    
    # Récupération d'un secret
    $self.get_secret() {
        local key="$1"
        
        # Vérification du cache
        if [[ -n "${$self._secrets[$key]}" ]]; then
            echo "${$self._secrets[$key]}"
            return 0
        fi
        
        # Recherche dans le fichier
        if [[ -f "$self._vault" ]]; then
            local encrypted_value
            encrypted_value=$(grep "^${key}:" "$self._vault" 2>/dev/null | cut -d: -f2-)
            
            if [[ -n "$encrypted_value" ]]; then
                local decrypted_value
                decrypted_value=$($self.decrypt_secret "$encrypted_value")
                
                # Mise en cache
                $self._secrets["$key"]="$decrypted_value"
                
                echo "$decrypted_value"
                return 0
            fi
        fi
        
        echo "Secret '$key' non trouvé" >&2
        return 1
    }
    
    # Suppression d'un secret
    $self.delete_secret() {
        local key="$1"
        
        # Suppression du cache
        unset $self._secrets["$key"]
        
        # Suppression du fichier
        if [[ -f "$self._vault" ]]; then
            sed -i "/^${key}:/d" "$self._vault"
        fi
        
        echo "✓ Secret '$key' supprimé"
    }
    
    # Rotation des clés (simulation)
    $self.rotate_keys() {
        echo "Rotation des clés de chiffrement..."
        
        local temp_vault="/tmp/temp_vault_$$"
        
        # Recréation de tous les secrets avec la nouvelle clé
        while IFS=: read -r key encrypted_value; do
            local decrypted_value
            decrypted_value=$($self.decrypt_secret "$encrypted_value")
            local new_encrypted
            new_encrypted=$($self.encrypt_secret "$decrypted_value")
            
            echo "${key}:${new_encrypted}" >> "$temp_vault"
        done < "$self._vault"
        
        # Remplacement
        mv "$temp_vault" "$self._vault"
        chmod 600 "$self._vault"
        
        # Vidage du cache
        $self._secrets=()
        
        echo "✓ Clés rotées"
    }
    
    # Audit des accès
    $self.audit_access() {
        local action="$1"
        local key="$2"
        
        local timestamp
        timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        local user
        user=$(whoami)
        local pid
        pid=$$
        
        echo "[$timestamp] [$user:$pid] $action secret: $key" >> /var/log/secret_audit.log
    }
    
    # Nettoyage automatique
    $self.cleanup() {
        rm -f "$self._vault"
        $self._secrets=()
    }
}

# Fonction de génération de mots de passe sécurisés
generate_secure_password() {
    local length="${1:-16}"
    
    # Combinaison de sources d'entropie
    {
        # Données système
        uname -a
        date +%s.%N
        whoami
        pwd
        
        # Aléatoire du système
        if [[ -f /dev/urandom ]]; then
            head -c 32 /dev/urandom | od -An -tx1 | tr -d ' '
        fi
        
        # PID et autres
        echo "$$ $PPID $RANDOM"
    } | sha256sum | cut -c1-$length
}

# Fonction de validation de mot de passe
validate_password_strength() {
    local password="$1"
    
    local score=0
    local feedback=""
    
    # Longueur
    if (( ${#password} >= 12 )); then
        ((score += 2))
    elif (( ${#password} >= 8 )); then
        ((score += 1))
    else
        feedback="${feedback}Mot de passe trop court. "
    fi
    
    # Complexité
    if [[ "$password" =~ [a-z] ]]; then ((score += 1)); else feedback="${feedback}Minuscules manquantes. "; fi
    if [[ "$password" =~ [A-Z] ]]; then ((score += 1)); else feedback="${feedback}Majuscules manquantes. "; fi
    if [[ "$password" =~ [0-9] ]]; then ((score += 1)); else feedback="${feedback}Chiffres manquants. "; fi
    if [[ "$password" =~ [^a-zA-Z0-9] ]]; then ((score += 1)); else feedback="${feedback}Caractères spéciaux manquants. "; fi
    
    # Entropie (simplifiée)
    local unique_chars
    unique_chars=$(echo "$password" | grep -o . | sort | uniq | wc -l)
    if (( unique_chars >= 10 )); then
        ((score += 1))
    fi
    
    # Évaluation
    case "$score" in
        0|1|2|3)
            echo "❌ FAIBLE ($score/7): $feedback"
            return 1
            ;;
        4|5)
            echo "⚠️ MOYEN ($score/7): $feedback"
            return 0
            ;;
        6|7)
            echo "✓ FORT ($score/7)"
            return 0
            ;;
    esac
}

# Démonstration de la gestion des secrets
echo "--- Gestion des secrets ---"

# Clé de chiffrement (en production, utiliser un KMS ou une gestion appropriée)
export SECRET_KEY="ma_cle_secrete_temporaire_pour_demo"

SecretManager "secret_mgr" "/tmp/demo_secrets.vault"

# Stockage de secrets
secret_mgr.store_secret "db_password" "SuperSecretPass123!"
secret_mgr.store_secret "api_key" "sk-1234567890abcdef"
secret_mgr.store_secret "ssh_key" "$(cat ~/.ssh/id_rsa 2>/dev/null || echo 'no_ssh_key')"

# Récupération
echo "Récupération des secrets:"
echo "DB Password: $(secret_mgr.get_secret 'db_password')"
echo "API Key: $(secret_mgr.get_secret 'api_key')"

echo
echo "--- Génération de mots de passe ---"

for i in {1..3}; do
    password=$(generate_secure_password 12)
    echo "Mot de passe $i: $password"
    validate_password_strength "$password"
done

echo
echo "--- Nettoyage ---"
secret_mgr.cleanup

# Nettoyage final
rm -f /tmp/demo_secrets.vault
unset SECRET_KEY
```

## Section 3 : Évaluation des vulnérabilités et audit

### 3.1 Scanner de vulnérabilités intégré

Outil d'analyse automatique des risques :

```bash
#!/bin/bash

# Scanner de vulnérabilités intégré
echo "=== Scanner de vulnérabilités intégré ==="

# Framework d'analyse de sécurité
SecurityScanner() {
    local self="$1"
    
    declare -a $self._vulnerabilities
    declare -A $self._severity_counts
    
    $self._log_vulnerability() {
        local severity="$1"
        local category="$2"
        local description="$3"
        local file="$4"
        local line="$5"
        
        local vuln="$severity|$category|$description|$file|$line"
        $self._vulnerabilities+=("$vuln")
        
        # Comptage par sévérité
        (( $self._severity_counts["$severity"]++ ))
        
        # Affichage immédiat
        local color=""
        case "$severity" in
            CRITICAL) color="\033[31m" ;;  # Rouge
            HIGH) color="\033[35m" ;;     # Magenta
            MEDIUM) color="\033[33m" ;;   # Jaune
            LOW) color="\033[36m" ;;      # Cyan
            INFO) color="\033[32m" ;;     # Vert
        esac
        
        echo -e "${color}[$severity] $category: $description${reset_color}"
        if [[ -n "$file" && -n "$line" ]]; then
            echo -e "${color}  → $file:$line${reset_color}"
        fi
    }
    
    $self.scan_script() {
        local script_file="$1"
        
        echo "=== Analyse de sécurité: $script_file ==="
        
        if [[ ! -f "$script_file" ]]; then
            $self._log_vulnerability "HIGH" "FILE_ACCESS" "Fichier de script inaccessible" "$script_file" ""
            return 1
        fi
        
        # Analyse ligne par ligne
        local line_num=1
        while IFS= read -r line; do
            $self._analyze_line "$line" "$script_file" "$line_num"
            ((line_num++))
        done < "$script_file"
        
        # Analyses globales
        $self._analyze_global "$script_file"
    }
    
    $self._analyze_line() {
        local line="$1"
        local file="$2"
        local line_num="$3"
        
        # Injection de commande
        if [[ "$line" =~ eval.*\$ ]]; then
            $self._log_vulnerability "CRITICAL" "COMMAND_INJECTION" "Utilisation dangereuse d'eval avec variable" "$file" "$line_num"
        fi
        
        if [[ "$line" =~ \$\(.*\$ \) ]]; then
            $self._log_vulnerability "HIGH" "COMMAND_INJECTION" "Substitution de commande avec variable non quotée" "$file" "$line_num"
        fi
        
        # Variables non initialisées
        if [[ "$line" =~ [^a-zA-Z_]\$[a-zA-Z_][a-zA-Z0-9_]*[^a-zA-Z0-9_] ]] && [[ ! "$line" =~ set\ -u ]] && [[ ! "$line" =~ \$\{[a-zA-Z_][a-zA-Z0-9_]*:- ]]; then
            # Vérification simplifiée (à améliorer)
            if [[ "$line" =~ \$\{?[a-zA-Z_][a-zA-Z0-9_]*\}? ]] && [[ ! "$line" =~ \$\{?[a-zA-Z_][a-zA-Z0-9_]*:[-=] ]]; then
                $self._log_vulnerability "MEDIUM" "UNINITIALIZED_VAR" "Variable potentiellement non initialisée" "$file" "$line_num"
            fi
        fi
        
        # Permissions dangereuses
        if [[ "$line" =~ chmod\ 777 ]]; then
            $self._log_vulnerability "HIGH" "PERMISSIONS" "Attribution de permissions 777" "$file" "$line_num"
        fi
        
        # Suppression récursive dangereuse
        if [[ "$line" =~ rm\ -rf\ / ]]; then
            $self._log_vulnerability "CRITICAL" "DELETION" "Suppression récursive de tout le système" "$file" "$line_num"
        fi
        
        # Utilisation de fonctions obsolètes
        if [[ "$line" =~ \$\[.*\] ]]; then
            $self._log_vulnerability "LOW" "DEPRECATED" "Utilisation de \$[...] au lieu de \$(...)" "$file" "$line_num"
        fi
        
        # Inclusion de fichiers non sécurisée
        if [[ "$line" =~ source.*\$ ]]; then
            $self._log_vulnerability "HIGH" "FILE_INCLUSION" "Inclusion de fichier dynamique" "$file" "$line_num"
        fi
        
        # Exposition d'informations sensibles
        if [[ "$line" =~ echo.*password|echo.*secret|echo.*key ]]; then
            $self._log_vulnerability "MEDIUM" "INFO_DISCLOSURE" "Potentielle exposition d'informations sensibles" "$file" "$line_num"
        fi
    }
    
    $self._analyze_global() {
        local file="$1"
        
        # Vérification de set -e
        if ! grep -q "^set -e" "$file" && ! grep -q "^set -o errexit" "$file"; then
            $self._log_vulnerability "MEDIUM" "ERROR_HANDLING" "Pas de 'set -e' pour arrêter sur erreur" "$file" ""
        fi
        
        # Vérification de set -u
        if ! grep -q "^set -u" "$file" && ! grep -q "^set -o nounset" "$file"; then
            $self._log_vulnerability "LOW" "ERROR_HANDLING" "Pas de 'set -u' pour variables non définies" "$file" ""
        fi
        
        # Vérification des fonctions
        local function_count
        function_count=$(grep -c "^[a-zA-Z_][a-zA-Z0-9_]*()" "$file")
        
        if (( function_count > 20 )); then
            $self._log_vulnerability "LOW" "COMPLEXITY" "Script très complexe ($function_count fonctions)" "$file" ""
        fi
        
        # Vérification de la taille
        local line_count
        line_count=$(wc -l < "$file")
        
        if (( line_count > 1000 )); then
            $self._log_vulnerability "INFO" "COMPLEXITY" "Script très long ($line_count lignes)" "$file" ""
        fi
    }
    
    $self.generate_report() {
        local output_file="${1:-security_report.txt}"
        
        echo "=== RAPPORT DE SÉCURITÉ ===" > "$output_file"
        echo "Généré le: $(date)" >> "$output_file"
        echo "Total vulnérabilités: ${#$self._vulnerabilities[@]}" >> "$output_file"
        echo >> "$output_file"
        
        echo "Résumé par sévérité:" >> "$output_file"
        for severity in CRITICAL HIGH MEDIUM LOW INFO; do
            local count="${$self._severity_counts[$severity]:-0}"
            printf "%-8s: %d\n" "$severity" "$count" >> "$output_file"
        done
        
        echo >> "$output_file"
        echo "Détails des vulnérabilités:" >> "$output_file"
        
        for vuln in "${$self._vulnerabilities[@]}"; do
            IFS='|' read -r severity category description file line <<< "$vuln"
            echo "[$severity] $category: $description" >> "$output_file"
            if [[ -n "$file" ]]; then
                echo "  Fichier: $file" >> "$output_file"
                if [[ -n "$line" ]]; then
                    echo "  Ligne: $line" >> "$output_file"
                fi
            fi
            echo >> "$output_file"
        done
        
        echo "✓ Rapport généré: $output_file"
    }
}

# Fonction de test du scanner
test_security_scanner() {
    echo "--- Test du scanner de sécurité ---"
    
    # Création d'un script de test avec des vulnérabilités
    cat > /tmp/vulnerable_script.sh << 'EOF'
#!/bin/bash

# Script avec vulnérabilités pour test

# Pas de set -e ou set -u
user_input="$1"
file_path="$2"

# Injection de commande CRITIQUE
eval "ls $user_input"

# Inclusion de fichier dangereuse
source "$file_path"

# Permissions dangereuses
chmod 777 /tmp/test

# Suppression dangereuse
rm -rf /

# Variable non initialisée
echo "Result: $undefined_variable"

# Ancienne syntaxe
result=$[2 + 2]

# Exposition de secret
echo "Password is: $password"

# Fonction complexe
complex_function() {
    # Beaucoup de code...
    echo "Complex"
}
EOF
    
    SecurityScanner "scanner"
    scanner.scan_script "/tmp/vulnerable_script.sh"
    
    echo
    scanner.generate_report "/tmp/security_report.txt"
    
    echo "Rapport généré:"
    head -20 /tmp/security_report.txt
    
    # Nettoyage
    rm -f /tmp/vulnerable_script.sh /tmp/security_report.txt
}

# Tests du scanner
test_security_scanner
```

### 3.2 Audit de sécurité automatique

Système de surveillance continue :

```bash
#!/bin/bash

# Audit de sécurité automatique
echo "=== Audit de sécurité automatique ==="

# Système d'audit de sécurité
SecurityAuditor() {
    local self="$1"
    
    $self._audit_log="/var/log/security_audit.log"
    declare -A $self._baseline
    
    # Établissement d'une baseline
    $self.establish_baseline() {
        echo "Établissement de la baseline de sécurité..."
        
        # Comptage des fichiers critiques
        $self._baseline["passwd_entries"]=$(wc -l < /etc/passwd)
        $self._baseline["shadow_entries"]=$(wc -l < /etc/shadow 2>/dev/null || echo "0")
        $self._baseline["sudoers_size"]=$(stat -c%s /etc/sudoers 2>/dev/null || echo "0")
        $self._baseline["cron_jobs"]=$(crontab -l 2>/dev/null | wc -l)
        
        # Permissions des fichiers sensibles
        $self._baseline["passwd_perms"]=$(stat -c%a /etc/passwd)
        $self._baseline["shadow_perms"]=$(stat -c%a /etc/shadow 2>/dev/null || echo "0")
        
        echo "✓ Baseline établie"
    }
    
    # Audit périodique
    $self.audit_system() {
        local timestamp
        timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        
        echo "[$timestamp] === AUDIT DE SÉCURITÉ ===" >> "$self._audit_log"
        
        # Vérification des comptes système
        $self._audit_accounts
        
        # Vérification des permissions
        $self._audit_permissions
        
        # Vérification des processus suspects
        $self._audit_processes
        
        # Vérification des connexions réseau
        $self._audit_network
        
        echo "[$timestamp] Audit terminé" >> "$self._audit_log"
    }
    
    $self._audit_accounts() {
        echo "Audit des comptes..." >> "$self._audit_log"
        
        # Comptes sans mot de passe
        local empty_passwords
        empty_passwords=$(awk -F: '($2 == "") {print $1}' /etc/shadow 2>/dev/null | wc -l)
        
        if (( empty_passwords > 0 )); then
            echo "ALERTE: $empty_passwords comptes sans mot de passe" >> "$self._audit_log"
        fi
        
        # Comptes avec UID 0 (root)
        local root_accounts
        root_accounts=$(awk -F: '($3 == 0) {print $1}' /etc/passwd | wc -l)
        
        if (( root_accounts > 1 )); then
            echo "ALERTE: $root_accounts comptes avec UID 0" >> "$self._audit_log"
        fi
    }
    
    $self._audit_permissions() {
        echo "Audit des permissions..." >> "$self._audit_log"
        
        # Fichiers world-writable
        local world_writable
        world_writable=$(find /etc -type f -perm -002 2>/dev/null | wc -l)
        
        if (( world_writable > 0 )); then
            echo "ALERTE: $world_writable fichiers world-writable dans /etc" >> "$self._audit_log"
        fi
        
        # Fichiers SUID/SGID
        local suid_files
        suid_files=$(find /usr/bin -type f -perm /4000 2>/dev/null | wc -l)
        
        echo "INFO: $suid_files fichiers SUID dans /usr/bin" >> "$self._audit_log"
    }
    
    $self._audit_processes() {
        echo "Audit des processus..." >> "$self._audit_log"
        
        # Processus zombies
        local zombies
        zombies=$(ps aux | awk '{print $8}' | grep -c 'Z')
        
        if (( zombies > 0 )); then
            echo "ALERTE: $zombies processus zombies détectés" >> "$self._audit_log"
        fi
        
        # Processus root suspects
        local root_processes
        root_processes=$(ps -U root -u root u | wc -l)
        
        echo "INFO: $root_processes processus root actifs" >> "$self._audit_log"
    }
    
    $self._audit_network() {
        echo "Audit réseau..." >> "$self._audit_log"
        
        # Ports ouverts
        if command -v ss >/dev/null 2>&1; then
            local listening_ports
            listening_ports=$(ss -tln | grep LISTEN | wc -l)
            
            echo "INFO: $listening_ports ports en écoute" >> "$self._audit_log"
            
            # Ports dangereux ouverts
            if ss -tln | grep -q ':22 '; then
                echo "INFO: SSH actif sur port 22" >> "$self._audit_log"
            fi
        fi
    }
    
    # Génération de rapports
    $self.generate_audit_report() {
        local report_file="${1:-security_audit_report.txt}"
        
        echo "=== RAPPORT D'AUDIT DE SÉCURITÉ ===" > "$report_file"
        echo "Généré le: $(date)" >> "$report_file"
        echo >> "$report_file"
        
        # Résumé des derniers audits
        if [[ -f "$self._audit_log" ]]; then
            echo "DERNIERS ÉVÉNEMENTS:" >> "$report_file"
            tail -20 "$self._audit_log" >> "$report_file"
        fi
        
        echo "✓ Rapport généré: $report_file"
    }
    
    # Surveillance en temps réel
    $self.monitor_realtime() {
        local duration="${1:-60}"
        
        echo "Surveillance en temps réel ($duration secondes)..."
        
        local start_time=$(date +%s)
        local anomalies=0
        
        while (( $(date +%s) - start_time < duration )); do
            # Vérifications rapides
            local current_users
            current_users=$(who | wc -l)
            
            local current_processes
            current_processes=$(ps aux | wc -l)
            
            # Détection d'anomalies simples
            if (( current_processes > 500 )); then
                echo "[$(date)] ALERTE: Nombre élevé de processus ($current_processes)"
                ((anomalies++))
            fi
            
            sleep 5
        done
        
        echo "Surveillance terminée - $anomalies anomalies détectées"
    }
}

# Fonction de durcissement système
harden_system() {
    echo "--- Durcissement du système ---"
    
    local changes=0
    
    # Désactivation de services inutiles (simulation)
    echo "Vérification des services à désactiver..."
    
    # Configuration des permissions
    echo "Configuration des permissions sécurisées..."
    
    # Configuration du firewall (simulation)
    if command -v ufw >/dev/null 2>&1; then
        echo "Configuration d'UFW..."
        # ufw --force enable
        ((changes++))
    fi
    
    # Mise à jour des paquets (simulation)
    echo "Vérification des mises à jour de sécurité..."
    
    # Configuration des logs
    echo "Configuration de l'audit système..."
    
    echo "✓ $changes modifications appliquées"
}

# Tests de l'audit de sécurité
echo "--- Audit de sécurité ---"

SecurityAuditor "auditor"

auditor.establish_baseline
auditor.audit_system

echo "Audit terminé. Consultez: ${auditor._audit_log}"

echo
echo "--- Surveillance temps réel (courte) ---"
auditor.monitor_realtime 10

echo
auditor.generate_audit_report "/tmp/audit_report.txt"

echo "Rapport généré:"
head -10 /tmp/audit_report.txt

echo
harden_system

# Nettoyage
rm -f /tmp/audit_report.txt
```

## Conclusion : La sécurité comme état d'esprit

La sécurité des scripts Bash n'est pas un ensemble de règles à suivre mécaniquement : c'est une philosophie de développement qui imprègne chaque ligne de code. Comme un maître d'arts martiaux qui transforme chaque mouvement en défense, vous devez transformer chaque fonction, chaque variable, chaque commande en élément de protection.

**Points clés à retenir :**

1. **Validation impitoyable** : Tout input est coupable jusqu'à preuve du contraire
2. **Principe du moindre privilège** : Réduire les permissions au strict minimum
3. **Gestion sécurisée des secrets** : Protéger les informations sensibles à tout prix
4. **Audit continu** : Surveiller, analyser, et réagir aux menaces
5. **Frameworks de sécurité** : Créer des outils réutilisables pour valider et protéger

Dans le chapitre suivant, nous explorerons les techniques avancées de déploiement et d'automatisation, pour que vos scripts sécurisés puissent conquérir et administrer des infrastructures complexes en toute sécurité.

---

**Exercice pratique :** Créez un framework de sécurité complet incluant :
- Un validateur d'input multi-types avec assainissement
- Un gestionnaire de secrets avec chiffrement et audit
- Un scanner de vulnérabilités pour scripts Bash
- Un système de surveillance temps réel des anomalies
- Des règles de durcissement automatique

**Réflexion :** Comment adapteriez-vous ces techniques de sécurité pour un environnement cloud où les menaces sont distribuées et les périmètres difficiles à définir ?
