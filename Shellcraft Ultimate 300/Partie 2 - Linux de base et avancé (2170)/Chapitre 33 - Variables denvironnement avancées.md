# Chapitre 33 - Variables d'environnement avancées

## Table des matières
- [Introduction](#introduction)
- [Variables d'environnement système avancées](#variables-denvironnement-système-avancées)
- [Gestion avancée des variables](#gestion-avancée-des-variables)
- [Variables dynamiques et calculées](#variables-dynamiques-et-calculées)
- [Persistance et chargement des variables](#persistance-et-chargement-des-variables)
- [Variables spéciales du shell](#variables-spéciales-du-shell)
- [Environnements isolés et virtuels](#environnements-isolés-et-virtuels)
- [Variables d'environnement pour applications](#variables-denvironnement-pour-applications)
- [Débogage et inspection avancée](#débogage-et-inspection-avancée)
- [Optimisation et performance](#optimisation-et-performance)
- [Scripts de gestion d'environnement](#scripts-de-gestion-denvironnement)
- [Conclusion](#conclusion)

## Introduction

Les variables d'environnement constituent le système nerveux central de la configuration dans les systèmes UNIX. Au-delà des simples affectations de valeurs, elles permettent de créer des environnements dynamiques, adaptatifs, et hautement configurables.

Imaginez les variables d'environnement comme les paramètres d'un instrument de musique sophistiqué : chaque réglage influence le comportement global, et une combinaison intelligente de paramètres peut produire des résultats extraordinaires. Dans un système Linux avancé, les variables d'environnement contrôlent non seulement le comportement des applications, mais aussi la sécurité, les performances, et l'intégration entre différents composants système.

## Variables d'environnement système avancées

### Variables système critiques

**Variables de localisation** :
```bash
#!/bin/bash
# Variables de localisation et internationalisation

# LANG : Langue principale
export LANG=fr_FR.UTF-8
export LC_ALL=fr_FR.UTF-8

# Variables LC_* spécifiques
export LC_TIME=fr_FR.UTF-8      # Format de date/heure
export LC_NUMERIC=fr_FR.UTF-8   # Format numérique
export LC_MONETARY=fr_FR.UTF-8  # Format monétaire
export LC_MESSAGES=en_US.UTF-8  # Messages système

# Vérifier les locales disponibles
locale -a

# Générer une locale si nécessaire
# sudo locale-gen fr_FR.UTF-8
```

**Variables de terminal** :
```bash
#!/bin/bash
# Variables de terminal avancées

# TERM : Type de terminal
export TERM=xterm-256color

# Variables de couleur
export COLORTERM=truecolor
export TERM_PROGRAM=iTerm.app  # Sur macOS

# Variables de terminal spécifiques
export TERMINAL=gnome-terminal
export TERM_SESSION_ID="session-$(date +%s)"

# Variables pour tmux/screen
export TMUX_PANE="%0"
export STY="1234.pts-0.hostname"  # Screen
```

**Variables de système** :
```bash
#!/bin/bash
# Variables système avancées

# HOSTNAME et domaine
export HOSTNAME=$(hostname)
export DOMAINNAME=$(domainname 2>/dev/null || echo "local")

# Architecture système
export HOSTTYPE=$(uname -m)
export MACHTYPE=$(uname -m)-$(uname -s)-$(uname -r | cut -d. -f1)

# Informations système
export OSTYPE=$(uname -s | tr '[:upper:]' '[:lower:]')
export UNAME=$(uname -a)

# Variables de version
export SHELL_VERSION="$BASH_VERSION"
export OS_VERSION=$(uname -r)
```

### Variables de sécurité

**Variables de sécurité système** :
```bash
#!/bin/bash
# Variables de sécurité

# Umask par défaut
export UMASK=022

# Variables de sécurité SSH
export SSH_AUTH_SOCK="$HOME/.ssh/agent.sock"
export SSH_AGENT_PID=$(pgrep ssh-agent)

# Variables de sécurité pour applications
export GPG_TTY=$(tty)
export PINENTRY_USER_DATA="USE_CURSES=1"

# Variables de certificats
export SSL_CERT_FILE="/etc/ssl/certs/ca-certificates.crt"
export SSL_CERT_DIR="/etc/ssl/certs"

# Variables de sécurité pour développement
export NODE_ENV="production"  # ou "development"
export PYTHONDONTWRITEBYTECODE=1
export PIP_REQUIRE_VIRTUALENV=true
```

## Gestion avancée des variables

### Manipulation avancée

**Substitution de variables** :
```bash
#!/bin/bash
# Substitution avancée de variables

# Valeur par défaut
echo "${VAR:-default}"        # Utilise default si VAR vide ou non définie
echo "${VAR-default}"         # Utilise default seulement si VAR non définie

# Assignation conditionnelle
${VAR:=default}               # Assigne default si VAR vide
${VAR=default}                # Assigne seulement si VAR non définie

# Vérification d'existence
${VAR:?message}               # Erreur si VAR vide ou non définie
${VAR?message}                # Erreur seulement si VAR non définie

# Suppression de préfixe/suffixe
VAR="/path/to/file.txt"
echo "${VAR#*/}"              # Supprime le plus court préfixe : path/to/file.txt
echo "${VAR##*/}"             # Supprime le plus long préfixe : file.txt
echo "${VAR%/*}"              # Supprime le plus court suffixe : /path/to/file
echo "${VAR%%/*}"             # Supprime le plus long suffixe : (vide)

# Remplacement
VAR="hello world"
echo "${VAR/world/universe}"  # Remplace première occurrence
echo "${VAR//world/universe}" # Remplace toutes les occurrences

# Extraction de sous-chaîne
VAR="abcdefgh"
echo "${VAR:2}"               # À partir de l'index 2 : cdefgh
echo "${VAR:2:3}"             # 3 caractères à partir de l'index 2 : cde
echo "${VAR: -3}"             # 3 derniers caractères : fgh
```

**Variables indirectes** :
```bash
#!/bin/bash
# Références indirectes de variables

# Variable contenant le nom d'une autre variable
VAR_NAME="USERNAME"
eval "echo \$$VAR_NAME"       # Affiche la valeur de $USERNAME

# Syntaxe moderne avec ${!}
VAR_NAME="HOME"
echo "${!VAR_NAME}"           # Affiche la valeur de $HOME

# Tableau de noms de variables
VAR_NAMES=("USER" "HOME" "PATH")
for var_name in "${VAR_NAMES[@]}"; do
    echo "$var_name=${!var_name}"
done
```

### Gestion des tableaux

**Tableaux d'environnement** :
```bash
#!/bin/bash
# Tableaux et variables d'environnement

# Tableau Bash
declare -a MY_ARRAY=("item1" "item2" "item3")

# Convertir tableau en variable d'environnement
export MY_ARRAY_STR=$(IFS=:; echo "${MY_ARRAY[*]}")

# Reconvertir en tableau
IFS=: read -ra RESTORED_ARRAY <<< "$MY_ARRAY_STR"

# Tableaux associatifs (Bash 4+)
declare -A CONFIG=(
    [database_host]="localhost"
    [database_port]="5432"
    [database_name]="mydb"
)

# Exporter comme variables séparées
for key in "${!CONFIG[@]}"; do
    export "DB_${key^^}=${CONFIG[$key]}"
done
```

## Variables dynamiques et calculées

### Variables calculées à la volée

**Variables dynamiques** :
```bash
#!/bin/bash
# Variables calculées dynamiquement

# Fonction pour variable dynamique
get_timestamp() {
    date +%s
}

# Variable mise à jour à chaque accès
TIMESTAMP=$(get_timestamp)

# Variable calculée avec fonction
get_dynamic_path() {
    echo "/data/$(date +%Y/%m/%d)"
}

export DATA_PATH=$(get_dynamic_path)

# Variable avec command substitution
export CURRENT_USER=$(whoami)
export CURRENT_DIR=$(pwd)
export SYSTEM_LOAD=$(uptime | awk -F'load average:' '{print $2}')

# Variable avec calcul arithmétique
export RANDOM_PORT=$((RANDOM % 10000 + 1000))
export DISK_USAGE=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
```

**Variables conditionnelles** :
```bash
#!/bin/bash
# Variables conditionnelles

# Variable selon la plateforme
case "$(uname -s)" in
    Linux*)     export PLATFORM="linux" ;;
    Darwin*)    export PLATFORM="macos" ;;
    CYGWIN*|MINGW*) export PLATFORM="windows" ;;
    *)          export PLATFORM="unknown" ;;
esac

# Variable selon l'environnement
if [ -f "/.dockerenv" ]; then
    export ENVIRONMENT="docker"
elif [ -n "${CI:-}" ]; then
    export ENVIRONMENT="ci"
else
    export ENVIRONMENT="local"
fi

# Variable selon la connexion
if [ -n "${SSH_CONNECTION:-}" ]; then
    export SESSION_TYPE="remote"
else
    export SESSION_TYPE="local"
fi
```

## Persistance et chargement des variables

### Fichiers de configuration

**Ordre de chargement** :
```bash
#!/bin/bash
# Ordre de chargement des fichiers de configuration

# Pour login shells :
# 1. /etc/profile
# 2. ~/.bash_profile (si existe)
# 3. ~/.bash_login (si ~/.bash_profile n'existe pas)
# 4. ~/.profile (si ni ~/.bash_profile ni ~/.bash_login n'existent)

# Pour non-login shells :
# 1. /etc/bash.bashrc (Linux)
# 2. ~/.bashrc

# Vérifier le type de shell
if shopt -q login_shell; then
    echo "Login shell"
else
    echo "Non-login shell"
fi
```

**Gestion organisée des variables** :
```bash
#!/bin/bash
# Organisation des variables d'environnement

# Créer une structure organisée
setup_environment_structure() {
    local env_dir="$HOME/.config/env"
    
    mkdir -p "$env_dir"/{system,user,project,secrets}
    
    # Fichiers par catégorie
    touch "$env_dir/system/path.sh"
    touch "$env_dir/system/locale.sh"
    touch "$env_dir/user/aliases.sh"
    touch "$env_dir/user/functions.sh"
    touch "$env_dir/project/project1.sh"
    touch "$env_dir/secrets/.gitignore"
    
    echo "Structure créée dans $env_dir"
}

# Charger les fichiers par catégorie
load_environment() {
    local env_dir="${ENV_DIR:-$HOME/.config/env}"
    
    # Charger dans l'ordre
    for category in system user project; do
        if [ -d "$env_dir/$category" ]; then
            for file in "$env_dir/$category"/*.sh; do
                [ -f "$file" ] && source "$file"
            done
        fi
    done
}
```

### Variables par projet

**Environnements par projet** :
```bash
#!/bin/bash
# Gestion d'environnements par projet

# Fichier .env dans le projet
load_project_env() {
    local project_dir="${1:-.}"
    local env_file="$project_dir/.env"
    
    if [ -f "$env_file" ]; then
        # Charger les variables (sans export par défaut)
        set -a  # Auto-export
        source "$env_file"
        set +a  # Désactiver auto-export
        
        echo "Variables du projet chargées depuis $env_file"
    else
        echo "Fichier .env non trouvé dans $project_dir"
    fi
}

# Exemple de fichier .env
# DATABASE_URL=postgresql://localhost/mydb
# API_KEY=secret_key_here
# DEBUG=true

# Créer un environnement de projet
create_project_env() {
    local project_name="$1"
    local env_file=".env"
    
    cat > "$env_file" << EOF
# Environnement pour $project_name
PROJECT_NAME=$project_name
PROJECT_DIR=$(pwd)
PROJECT_ENV=development

# Variables personnalisées
# Ajoutez vos variables ici
EOF
    
    echo "Fichier $env_file créé"
}
```

## Variables spéciales du shell

### Variables shell avancées

**Variables de processus** :
```bash
#!/bin/bash
# Variables shell spéciales

# Informations sur le processus courant
echo "PID: $$"
echo "PPID: $PPID"
echo "UID: $UID"
echo "EUID: $EUID"
echo "GID: $GID"
echo "EGID: $EGID"

# Informations sur la commande
echo "Nom du script: $0"
echo "Nombre d'arguments: $#"
echo "Tous les arguments: $@"
echo "Tous les arguments (une chaîne): $*"
echo "Code de retour: $?"

# Options du shell
echo "Options: $-"

# Dernière commande en arrière-plan
echo "PID du dernier job: $!"
```

**Variables de répertoire** :
```bash
#!/bin/bash
# Variables de répertoire

# Répertoire courant
echo "PWD: $PWD"
echo "OLDPWD: $OLDPWD"

# Répertoire home
echo "HOME: $HOME"

# Répertoire temporaire
echo "TMPDIR: ${TMPDIR:-/tmp}"

# Fonction pour navigation
cd_with_history() {
    local target="$1"
    export OLDPWD="$PWD"
    cd "$target"
    echo "Déplacé de $OLDPWD vers $PWD"
}
```

### Variables de contrôle shell

**Options et comportement** :
```bash
#!/bin/bash
# Variables de contrôle shell

# IFS : Internal Field Separator
OLD_IFS="$IFS"
IFS=$'\n'  # Séparateur = nouvelle ligne
# ... traitement ...
IFS="$OLD_IFS"

# Variables de contrôle d'erreur
set -e     # Arrêter sur erreur
set -u     # Erreur si variable non définie
set -o pipefail  # Erreur dans les pipes

# Variables de débogage
export PS4='+ [${BASH_SOURCE}:${LINENO}]: ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x     # Mode debug

# Variables de historique
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTCONTROL=ignoreboth:erasedups
export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S "
```

## Environnements isolés et virtuels

### Environnements virtuels

**Création d'environnements isolés** :
```bash
#!/bin/bash
# Création d'environnement isolé

create_isolated_env() {
    local env_name="$1"
    local env_dir="$HOME/.envs/$env_name"
    
    mkdir -p "$env_dir"/{bin,lib,include}
    
    # Créer un script d'activation
    cat > "$env_dir/activate" << 'EOF'
#!/bin/bash
# Activation de l'environnement

ENV_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Sauvegarder l'environnement original
export _OLD_PATH="$PATH"
export _OLD_PS1="$PS1"

# Modifier le PATH
export PATH="$ENV_DIR/bin:$PATH"

# Modifier le prompt
export PS1="($(basename "$ENV_DIR")) $PS1"

# Variables spécifiques à l'environnement
export ENV_NAME="$(basename "$ENV_DIR")"
export ENV_DIR="$ENV_DIR"
EOF
    
    chmod +x "$env_dir/activate"
    
    # Créer un script de désactivation
    cat > "$env_dir/deactivate" << 'EOF'
#!/bin/bash
# Désactivation de l'environnement

if [ -n "${_OLD_PATH:-}" ]; then
    export PATH="$_OLD_PATH"
    unset _OLD_PATH
fi

if [ -n "${_OLD_PS1:-}" ]; then
    export PS1="$_OLD_PS1"
    unset _OLD_PS1
fi

unset ENV_NAME ENV_DIR
EOF
    
    chmod +x "$env_dir/deactivate"
    
    echo "Environnement créé: $env_dir"
    echo "Activer avec: source $env_dir/activate"
}
```

**Environnements Python virtuels** :
```bash
#!/bin/bash
# Intégration avec venv Python

setup_python_env() {
    local project_dir="$1"
    
    cd "$project_dir"
    
    # Créer venv
    python3 -m venv venv
    
    # Activer
    source venv/bin/activate
    
    # Variables d'environnement Python
    export VIRTUAL_ENV="$project_dir/venv"
    export VIRTUAL_ENV_PROMPT="($(basename "$project_dir"))"
    
    echo "Environnement Python activé"
}
```

## Variables d'environnement pour applications

### Variables pour outils de développement

**Variables pour Git** :
```bash
#!/bin/bash
# Configuration Git via variables

export GIT_AUTHOR_NAME="Alice Developer"
export GIT_AUTHOR_EMAIL="alice@example.com"
export GIT_COMMITTER_NAME="$GIT_AUTHOR_NAME"
export GIT_COMMITTER_EMAIL="$GIT_AUTHOR_EMAIL"

# Éditeur pour Git
export GIT_EDITOR=vim

# Pager pour Git
export GIT_PAGER=less

# Configuration Git globale
export GIT_CONFIG_GLOBAL="$HOME/.gitconfig"
```

**Variables pour Docker** :
```bash
#!/bin/bash
# Configuration Docker

export DOCKER_HOST=unix:///var/run/docker.sock
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH="$HOME/.docker"

# BuildKit pour builds plus rapides
export DOCKER_BUILDKIT=1
export COMPOSE_DOCKER_CLI_BUILD=1
```

**Variables pour Node.js** :
```bash
#!/bin/bash
# Configuration Node.js

export NODE_ENV=production
export NODE_OPTIONS="--max-old-space-size=4096"
export NPM_CONFIG_PREFIX="$HOME/.npm-global"
export PATH="$NPM_CONFIG_PREFIX/bin:$PATH"
```

**Variables pour Python** :
```bash
#!/bin/bash
# Configuration Python

export PYTHONPATH="$HOME/projects:$PYTHONPATH"
export PYTHONDONTWRITEBYTECODE=1
export PYTHONUNBUFFERED=1
export PIP_REQUIRE_VIRTUALENV=true
```

## Débogage et inspection avancée

### Outils d'inspection

**Inspection complète** :
```bash
#!/bin/bash
# Inspection avancée des variables

# Afficher toutes les variables d'environnement
printenv
env

# Afficher avec tri
env | sort

# Rechercher une variable
env | grep -i "path"

# Compter les variables
env | wc -l

# Variables par préfixe
env | grep "^PATH"

# Différence entre set et env
# set : Toutes les variables shell (locales + exportées)
# env : Seulement les variables exportées

# Comparer
diff <(set | sort) <(env | sort)
```

**Scripts de diagnostic** :
```bash
#!/bin/bash
# Diagnostic d'environnement

diagnose_environment() {
    echo "=== Diagnostic d'environnement ==="
    echo ""
    
    echo "=== Variables système ==="
    echo "USER: $USER"
    echo "HOME: $HOME"
    echo "SHELL: $SHELL"
    echo "PATH: $PATH"
    echo ""
    
    echo "=== Variables de localisation ==="
    echo "LANG: ${LANG:-non définie}"
    echo "LC_ALL: ${LC_ALL:-non définie}"
    echo ""
    
    echo "=== Variables de terminal ==="
    echo "TERM: ${TERM:-non définie}"
    echo "COLORTERM: ${COLORTERM:-non définie}"
    echo ""
    
    echo "=== Variables de développement ==="
    echo "EDITOR: ${EDITOR:-non définie}"
    echo "VISUAL: ${VISUAL:-non définie}"
    echo ""
    
    echo "=== Statistiques ==="
    echo "Nombre total de variables: $(env | wc -l)"
    echo "Variables avec PATH: $(env | grep -c PATH)"
}

# Vérifier les variables manquantes
check_required_vars() {
    local required=("HOME" "USER" "PATH" "SHELL")
    local missing=()
    
    for var in "${required[@]}"; do
        if [ -z "${!var:-}" ]; then
            missing+=("$var")
        fi
    done
    
    if [ ${#missing[@]} -gt 0 ]; then
        echo "⚠️  Variables manquantes: ${missing[*]}"
        return 1
    else
        echo "✓ Toutes les variables requises sont définies"
        return 0
    fi
}
```

## Optimisation et performance

### Optimisation du PATH

**Gestion efficace du PATH** :
```bash
#!/bin/bash
# Optimisation du PATH

# Fonction pour ajouter au PATH (sans doublons)
add_to_path() {
    local new_path="$1"
    
    if [[ ":$PATH:" != *":$new_path:"* ]]; then
        export PATH="$new_path:$PATH"
    fi
}

# Fonction pour ajouter à la fin
append_to_path() {
    local new_path="$1"
    
    if [[ ":$PATH:" != *":$new_path:"* ]]; then
        export PATH="$PATH:$new_path"
    fi
}

# Nettoyer les doublons du PATH
clean_path() {
    local cleaned_path
    cleaned_path=$(echo "$PATH" | tr ':' '\n' | awk '!seen[$0]++' | tr '\n' ':' | sed 's/:$//')
    export PATH="$cleaned_path"
}

# Optimiser le PATH (répertoires fréquents en premier)
optimize_path() {
    local frequent_dirs=("/usr/local/bin" "/usr/bin" "/bin")
    local other_path
    
    # Extraire les autres répertoires
    other_path=$(echo "$PATH" | tr ':' '\n' | \
        grep -vE "^($(IFS='|'; echo "${frequent_dirs[*]}"))$" | \
        tr '\n' ':' | sed 's/:$//')
    
    # Reconstruire avec fréquents en premier
    export PATH="$(IFS=':'; echo "${frequent_dirs[*]}"):$other_path"
}
```

### Cache et performance

**Mise en cache des variables** :
```bash
#!/bin/bash
# Cache de variables pour performance

# Cache pour commandes coûteuses
cache_variable() {
    local var_name="$1"
    local cache_file="/tmp/env_cache_${var_name}_$$"
    local cache_age="${2:-3600}"  # 1 heure par défaut
    
    if [ -f "$cache_file" ]; then
        local file_age=$(($(date +%s) - $(stat -c %Y "$cache_file")))
        if [ $file_age -lt $cache_age ]; then
            export "$var_name=$(cat "$cache_file")"
            return 0
        fi
    fi
    
    # Calculer et mettre en cache
    local value=$(eval "$3")
    echo "$value" > "$cache_file"
    export "$var_name=$value"
}

# Utilisation
# cache_variable "SYSTEM_INFO" 3600 "uname -a"
```

## Scripts de gestion d'environnement

### Script complet de gestion

**Gestionnaire d'environnement complet** :
```bash
#!/bin/bash
# Gestionnaire d'environnement complet

set -euo pipefail

ENV_MANAGER_DIR="${ENV_MANAGER_DIR:-$HOME/.env-manager}"

# Initialiser le gestionnaire
init_env_manager() {
    mkdir -p "$ENV_MANAGER_DIR"/{envs,scripts,backups}
    
    cat > "$ENV_MANAGER_DIR/config" << EOF
# Configuration du gestionnaire d'environnement
AUTO_LOAD_PROJECT_ENV=true
BACKUP_ON_CHANGE=true
EOF
    
    echo "Gestionnaire d'environnement initialisé"
}

# Sauvegarder l'environnement actuel
backup_environment() {
    local backup_file="$ENV_MANAGER_DIR/backups/env_$(date +%Y%m%d_%H%M%S).env"
    
    env | sort > "$backup_file"
    echo "Environnement sauvegardé: $backup_file"
}

# Restaurer un environnement
restore_environment() {
    local backup_file="$1"
    
    if [ ! -f "$backup_file" ]; then
        echo "Fichier de backup non trouvé: $backup_file"
        return 1
    fi
    
    # Charger les variables
    set -a
    source "$backup_file"
    set +a
    
    echo "Environnement restauré depuis $backup_file"
}

# Créer un environnement nommé
create_named_env() {
    local env_name="$1"
    local env_file="$ENV_MANAGER_DIR/envs/$env_name.env"
    
    # Sauvegarder l'environnement actuel
    env | sort > "$env_file"
    
    echo "Environnement '$env_name' créé: $env_file"
}

# Charger un environnement nommé
load_named_env() {
    local env_name="$1"
    local env_file="$ENV_MANAGER_DIR/envs/$env_name.env"
    
    if [ ! -f "$env_file" ]; then
        echo "Environnement '$env_name' non trouvé"
        return 1
    fi
    
    # Sauvegarder avant changement
    backup_environment
    
    # Charger le nouvel environnement
    set -a
    source "$env_file"
    set +a
    
    echo "Environnement '$env_name' chargé"
}

# Lister les environnements disponibles
list_environments() {
    echo "=== Environnements disponibles ==="
    ls -1 "$ENV_MANAGER_DIR/envs"/*.env 2>/dev/null | \
    while read -r env_file; do
        echo "  $(basename "$env_file" .env)"
    done
}
```

## Conclusion

La maîtrise avancée des variables d'environnement permet de créer des configurations sophistiquées, des environnements isolés, et des workflows optimisés. En comprenant les mécanismes de substitution, de persistance, et de chargement, vous pouvez créer des systèmes de configuration flexibles et maintenables.

Les variables d'environnement ne sont pas seulement des valeurs statiques, mais des outils dynamiques qui peuvent s'adapter au contexte, calculer des valeurs à la volée, et créer des environnements entièrement personnalisés. Un système bien configuré utilise les variables d'environnement pour créer une expérience utilisateur cohérente et efficace.

Dans le chapitre suivant, nous explorerons la gestion des disques et partitions, découvrant comment administrer efficacement le stockage dans un système Linux.
