# Chapitre 51 - Scripting modulaires et bibliothèques

## Table des matières
- [Introduction](#introduction)
- [Principes de la modularité](#principes-de-la-modularité)
- [Architecture modulaire](#architecture-modulaire)
- [Création de bibliothèques](#création-de-bibliothèques)
- [Structure d'un module](#structure-dun-module)
- [Chargement dynamique](#chargement-dynamique)
- [Gestion des dépendances](#gestion-des-dépendances)
- [Namespacing et isolation](#namespacing-et-isolation)
- [Versioning des modules](#versioning-des-modules)
- [Tests et validation](#tests-et-validation)
- [Documentation des modules](#documentation-des-modules)
- [Distribution et déploiement](#distribution-et-déploiement)
- [Gestionnaire de packages](#gestionnaire-de-packages)
- [Maintenance et évolution](#maintenance-et-évolution)
- [Bonnes pratiques](#bonnes-pratiques)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Le scripting modulaire constitue l'évolution naturelle des scripts simples vers des architectures complexes et maintenables. Au lieu d'écrire des monolithes, il s'agit de créer des bibliothèques réutilisables qui peuvent être assemblées comme des Lego pour construire des solutions robustes.

Imaginez le scripting modulaire comme une boîte à outils sophistiquée : chaque outil (module) est spécialisé dans une tâche précise, peut être combiné avec d'autres outils, et peut être remplacé ou amélioré indépendamment sans affecter l'ensemble du système. Cette approche transforme le code dupliqué en composants réutilisables, facilite les tests, et permet une évolution progressive du système.

## Principes de la modularité

### Concepts fondamentaux

**Principes essentiels** :
```bash
#!/bin/bash
# Principes de la modularité

# 1. Séparation des responsabilités
#    - Chaque module a une responsabilité unique
#    - Interface claire et bien définie

# 2. Réutilisabilité
#    - Code écrit une fois, utilisé partout
#    - Pas de duplication

# 3. Testabilité
#    - Modules testables indépendamment
#    - Tests unitaires et d'intégration

# 4. Maintenabilité
#    - Modifications isolées
#    - Impact limité aux dépendances

# 5. Évolutivité
#    - Ajout de nouvelles fonctionnalités facile
#    - Remplaçable sans casser le système
```

**Avantages** :
- Réduction de la duplication de code
- Facilité de test
- Maintenance simplifiée
- Collaboration améliorée
- Réutilisation maximale

## Architecture modulaire

### Organisation des fichiers

**Structure recommandée** :
```bash
#!/bin/bash
# Structure d'un projet modulaire

project_root/
├── lib/                    # Bibliothèques
│   ├── logging.sh         # Module de logging
│   ├── network.sh         # Module réseau
│   ├── database.sh        # Module base de données
│   └── utils.sh           # Utilitaires généraux
├── modules/               # Modules spécifiques
│   ├── backup/
│   │   ├── backup.sh
│   │   └── restore.sh
│   └── monitoring/
│       ├── health.sh
│       └── metrics.sh
├── bin/                   # Scripts exécutables
│   ├── main.sh
│   └── helper.sh
├── tests/                 # Tests
│   ├── test_logging.sh
│   └── test_network.sh
├── docs/                  # Documentation
│   └── modules.md
└── config/                # Configuration
    └── defaults.conf
```

## Création de bibliothèques

### Bibliothèque de base

**Exemple de bibliothèque** :
```bash
#!/usr/bin/env bash
# lib/logging.sh - Bibliothèque de logging

# Variables du module
declare -g LOG_LEVEL="${LOG_LEVEL:-INFO}"
declare -g LOG_FILE="${LOG_FILE:-/var/log/app.log}"
declare -g LOG_FORMAT="${LOG_FORMAT:-standard}"

# Niveaux de log
declare -r LOG_DEBUG=0
declare -r LOG_INFO=1
declare -r LOG_WARN=2
declare -r LOG_ERROR=3
declare -r LOG_CRITICAL=4

# Fonction principale de logging
log() {
    local level="$1"
    shift
    local message="$*"
    
    local level_num
    case "$level" in
        DEBUG)    level_num=$LOG_DEBUG ;;
        INFO)     level_num=$LOG_INFO ;;
        WARN)     level_num=$LOG_WARN ;;
        ERROR)    level_num=$LOG_ERROR ;;
        CRITICAL) level_num=$LOG_CRITICAL ;;
        *)        level_num=$LOG_INFO ;;
    esac
    
    # Vérifier si le niveau est suffisant
    if ! _log_should_log "$level_num"; then
        return 0
    fi
    
    # Formater le message
    local formatted_msg
    formatted_msg=$(_log_format_message "$level" "$message")
    
    # Écrire dans le fichier
    echo "$formatted_msg" >> "$LOG_FILE"
    
    # Écrire sur stderr si niveau élevé
    if [ "$level_num" -ge $LOG_ERROR ]; then
        echo "$formatted_msg" >&2
    fi
}

# Fonction privée pour vérifier le niveau
_log_should_log() {
    local level_num="$1"
    local config_level_num
    
    case "$LOG_LEVEL" in
        DEBUG)    config_level_num=$LOG_DEBUG ;;
        INFO)     config_level_num=$LOG_INFO ;;
        WARN)     config_level_num=$LOG_WARN ;;
        ERROR)    config_level_num=$LOG_ERROR ;;
        CRITICAL) config_level_num=$LOG_CRITICAL ;;
        *)        config_level_num=$LOG_INFO ;;
    esac
    
    [ "$level_num" -ge "$config_level_num" ]
}

# Fonction privée pour formater le message
_log_format_message() {
    local level="$1"
    local message="$2"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local hostname=$(hostname)
    local script_name="${0##*/}"
    local pid=$$
    
    case "$LOG_FORMAT" in
        json)
            echo "{\"timestamp\":\"$timestamp\",\"hostname\":\"$hostname\",\"level\":\"$level\",\"script\":\"$script_name\",\"pid\":$pid,\"message\":\"$message\"}"
            ;;
        syslog)
            echo "$timestamp $hostname $script_name[$pid]: [$level] $message"
            ;;
        *)
            echo "[$timestamp] [$level] [$hostname] $script_name[$pid]: $message"
            ;;
    esac
}

# Fonctions de convenance
log_debug() { log DEBUG "$@"; }
log_info() { log INFO "$@"; }
log_warn() { log WARN "$@"; }
log_error() { log ERROR "$@"; }
log_critical() { log CRITICAL "$@"; }
```

## Structure d'un module

### Template de module

**Template complet** :
```bash
#!/usr/bin/env bash
# lib/template.sh - Template de module

# ============================================================================
# Module: Template
# Description: Template pour créer de nouveaux modules
# Version: 1.0.0
# Author: Your Name
# ============================================================================

# Variables du module (préfixées par le nom du module)
declare -g TEMPLATE_CONFIG="${TEMPLATE_CONFIG:-/etc/template.conf}"
declare -g TEMPLATE_DEBUG="${TEMPLATE_DEBUG:-false}"

# Constantes du module
declare -r TEMPLATE_VERSION="1.0.0"
declare -r TEMPLATE_DEFAULT_VALUE="default"

# Fonction d'initialisation
template_init() {
    local config_file="${1:-$TEMPLATE_CONFIG}"
    
    if [ ! -f "$config_file" ]; then
        log_error "Fichier de configuration non trouvé: $config_file"
        return 1
    fi
    
    # Charger la configuration
    source "$config_file" || {
        log_error "Erreur lors du chargement de la configuration"
        return 1
    }
    
    log_info "Module template initialisé"
    return 0
}

# Fonction principale publique
template_do_something() {
    local input="$1"
    
    # Validation
    if [ -z "$input" ]; then
        log_error "Paramètre manquant"
        return 1
    fi
    
    # Traitement
    local result
    result=$(_template_process "$input")
    
    echo "$result"
    return 0
}

# Fonction privée (préfixée par _)
_template_process() {
    local input="$1"
    
    # Logique interne
    echo "Processed: $input"
}

# Fonction de nettoyage
template_cleanup() {
    log_info "Nettoyage du module template"
    # Nettoyage des ressources
}

# Auto-initialisation si nécessaire
# template_init
```

## Chargement dynamique

### Système de chargement

**Chargeur de modules** :
```bash
#!/bin/bash
# lib/loader.sh - Chargeur de modules

# Variables globales
declare -gA _LOADED_MODULES=()
declare -ga _MODULE_SEARCH_PATHS=(
    "$HOME/.local/lib/bash"
    "/usr/local/lib/bash"
    "/opt/bash-libs"
    "./lib"
    "./modules"
)

# Charger un module
load_module() {
    local module_name="$1"
    local version="${2:-latest}"
    
    # Vérifier si déjà chargé
    if [[ -v _LOADED_MODULES["$module_name"] ]]; then
        log_debug "Module $module_name déjà chargé"
        return 0
    fi
    
    # Chercher le module
    local module_path
    module_path=$(_find_module "$module_name" "$version")
    
    if [ -z "$module_path" ]; then
        log_error "Module non trouvé: $module_name (version: $version)"
        return 1
    fi
    
    # Charger le module
    if source "$module_path"; then
        _LOADED_MODULES["$module_name"]="$module_path"
        log_info "Module chargé: $module_name ($module_path)"
        return 0
    else
        log_error "Erreur lors du chargement du module: $module_name"
        return 1
    fi
}

# Trouver un module
_find_module() {
    local module_name="$1"
    local version="$2"
    
    for search_path in "${_MODULE_SEARCH_PATHS[@]}"; do
        # Chercher avec version
        if [ "$version" != "latest" ]; then
            local versioned_path="$search_path/$module_name/$version.sh"
            if [ -f "$versioned_path" ]; then
                echo "$versioned_path"
                return 0
            fi
        fi
        
        # Chercher sans version
        local default_path="$search_path/$module_name.sh"
        if [ -f "$default_path" ]; then
            echo "$default_path"
            return 0
        fi
        
        # Chercher dans sous-répertoire
        local subdir_path="$search_path/$module_name/$module_name.sh"
        if [ -f "$subdir_path" ]; then
            echo "$subdir_path"
            return 0
        fi
    done
    
    return 1
}

# Décharger un module
unload_module() {
    local module_name="$1"
    
    if [[ -v _LOADED_MODULES["$module_name"] ]]; then
        # Appeler la fonction de nettoyage si elle existe
        local cleanup_func="${module_name}_cleanup"
        if declare -f "$cleanup_func" >/dev/null 2>&1; then
            $cleanup_func
        fi
        
        unset _LOADED_MODULES["$module_name"]
        log_info "Module déchargé: $module_name"
        return 0
    fi
    
    log_warn "Module non chargé: $module_name"
    return 1
}

# Lister les modules chargés
list_loaded_modules() {
    echo "Modules chargés:"
    for module in "${!_LOADED_MODULES[@]}"; do
        echo "  - $module: ${_LOADED_MODULES[$module]}"
    done
}
```

## Gestion des dépendances

### Système de dépendances

**Gestionnaire de dépendances** :
```bash
#!/bin/bash
# lib/dependency_manager.sh - Gestionnaire de dépendances

# Déclaration des dépendances
declare -gA _MODULE_DEPENDENCIES=(
    ["web"]="network logging"
    ["database"]="logging"
    ["api"]="web database cache"
    ["cache"]="logging"
    ["monitoring"]="logging network"
)

# Charger avec dépendances
load_with_dependencies() {
    local module_name="$1"
    local version="${2:-latest}"
    
    # Vérifier si déjà chargé
    if [[ -v _LOADED_MODULES["$module_name"] ]]; then
        return 0
    fi
    
    # Charger les dépendances d'abord
    if [[ -v _MODULE_DEPENDENCIES["$module_name"] ]]; then
        local deps="${_MODULE_DEPENDENCIES[$module_name]}"
        for dep in $deps; do
            if ! load_with_dependencies "$dep" "$version"; then
                log_error "Échec du chargement de la dépendance: $dep"
                return 1
            fi
        done
    fi
    
    # Charger le module principal
    if load_module "$module_name" "$version"; then
        log_info "Module $module_name chargé avec ses dépendances"
        return 0
    else
        log_error "Échec du chargement du module: $module_name"
        return 1
    fi
}

# Vérifier les dépendances
check_dependencies() {
    local module_name="$1"
    local missing_deps=()
    
    if [[ -v _MODULE_DEPENDENCIES["$module_name"] ]]; then
        local deps="${_MODULE_DEPENDENCIES[$module_name]}"
        for dep in $deps; do
            if [[ ! -v _LOADED_MODULES["$dep"] ]]; then
                missing_deps+=("$dep")
            fi
        done
    fi
    
    if [ ${#missing_deps[@]} -gt 0 ]; then
        log_error "Dépendances manquantes pour $module_name: ${missing_deps[*]}"
        return 1
    fi
    
    return 0
}
```

## Namespacing et isolation

### Préfixes et namespaces

**Système de namespacing** :
```bash
#!/bin/bash
# Système de namespacing

# Préfixe pour toutes les fonctions du module
MODULE_PREFIX="mymodule"

# Créer une fonction avec namespace
create_namespaced_function() {
    local func_name="$1"
    local func_body="$2"
    
    eval "${MODULE_PREFIX}_${func_name}() {
        $func_body
    }"
}

# Utilisation
create_namespaced_function "init" 'echo "Initialisation"'
create_namespaced_function "process" 'echo "Traitement"'

# Appel
mymodule_init
mymodule_process
```

**Isolation avec sous-shell** :
```bash
#!/bin/bash
# Isolation avec sous-shell

module_execute() {
    local module_name="$1"
    shift
    
    # Exécuter dans un sous-shell pour isolation
    (
        # Charger le module
        source "lib/$module_name.sh"
        
        # Exécuter la commande
        "$@"
    )
}
```

## Versioning des modules

### Gestion des versions

**Système de versioning** :
```bash
#!/bin/bash
# lib/versioning.sh - Gestion des versions

# Format: MAJOR.MINOR.PATCH
parse_version() {
    local version="$1"
    IFS='.' read -ra VERSION_PARTS <<< "$version"
    
    VERSION_MAJOR="${VERSION_PARTS[0]}"
    VERSION_MINOR="${VERSION_PARTS[1]}"
    VERSION_PATCH="${VERSION_PARTS[2]}"
}

# Comparer des versions
compare_versions() {
    local v1="$1"
    local v2="$2"
    
    parse_version "$v1"
    local v1_major=$VERSION_MAJOR
    local v1_minor=$VERSION_MINOR
    local v1_patch=$VERSION_PATCH
    
    parse_version "$v2"
    local v2_major=$VERSION_MAJOR
    local v2_minor=$VERSION_MINOR
    local v2_patch=$VERSION_PATCH
    
    if [ "$v1_major" -lt "$v2_major" ]; then
        echo -1
    elif [ "$v1_major" -gt "$v2_major" ]; then
        echo 1
    elif [ "$v1_minor" -lt "$v2_minor" ]; then
        echo -1
    elif [ "$v1_minor" -gt "$v2_minor" ]; then
        echo 1
    elif [ "$v1_patch" -lt "$v2_patch" ]; then
        echo -1
    elif [ "$v1_patch" -gt "$v2_patch" ]; then
        echo 1
    else
        echo 0
    fi
}

# Charger une version spécifique
load_module_version() {
    local module_name="$1"
    local required_version="$2"
    
    # Chercher la version compatible
    local module_path
    module_path=$(_find_module_version "$module_name" "$required_version")
    
    if [ -n "$module_path" ]; then
        source "$module_path"
    else
        log_error "Version $required_version de $module_name non trouvée"
        return 1
    fi
}
```

## Tests et validation

### Framework de tests

**Système de tests** :
```bash
#!/bin/bash
# lib/test_framework.sh - Framework de tests

# Compteurs
declare -i TEST_PASSED=0
declare -i TEST_FAILED=0
declare -i TEST_TOTAL=0

# Exécuter un test
test_case() {
    local test_name="$1"
    shift
    local test_command="$*"
    
    TEST_TOTAL=$((TEST_TOTAL + 1))
    
    echo -n "Test: $test_name ... "
    
    if eval "$test_command" >/dev/null 2>&1; then
        echo "PASS"
        TEST_PASSED=$((TEST_PASSED + 1))
        return 0
    else
        echo "FAIL"
        TEST_FAILED=$((TEST_FAILED + 1))
        return 1
    fi
}

# Assertion
assert_equal() {
    local expected="$1"
    local actual="$2"
    
    if [ "$expected" = "$actual" ]; then
        return 0
    else
        echo "Assertion failed: expected '$expected', got '$actual'"
        return 1
    fi
}

# Afficher le résumé
test_summary() {
    echo ""
    echo "=== Résumé des tests ==="
    echo "Total: $TEST_TOTAL"
    echo "Passés: $TEST_PASSED"
    echo "Échoués: $TEST_FAILED"
    
    if [ $TEST_FAILED -eq 0 ]; then
        echo "Tous les tests sont passés!"
        return 0
    else
        echo "Certains tests ont échoué"
        return 1
    fi
}
```

**Exemple de test** :
```bash
#!/bin/bash
# tests/test_logging.sh

source lib/test_framework.sh
source lib/logging.sh

# Tests
test_case "Log level INFO" 'log_info "Test message"'
test_case "Log file exists" '[ -f "$LOG_FILE" ]'
test_case "Log format standard" '[ "$LOG_FORMAT" = "standard" ]'

# Afficher le résumé
test_summary
```

## Documentation des modules

### Documentation intégrée

**Système de documentation** :
```bash
#!/bin/bash
# lib/doc_generator.sh - Générateur de documentation

# Extraire la documentation d'un module
extract_module_docs() {
    local module_file="$1"
    
    echo "# Documentation du module: $(basename "$module_file" .sh)"
    echo ""
    
    # Extraire les commentaires de documentation
    grep -E '^#\s+(Description|Usage|Example|Parameters):' "$module_file" | \
        sed 's/^#\s*//'
    
    echo ""
    echo "## Fonctions"
    echo ""
    
    # Lister les fonctions publiques
    grep -E '^[a-zA-Z_][a-zA-Z0-9_]*\(\)' "$module_file" | \
        sed 's/()//' | \
        while read func; do
            echo "### $func"
            # Extraire les commentaires de la fonction
            grep -A 5 "^$func()" "$module_file" | grep '^#' | sed 's/^#\s*//'
            echo ""
        done
}
```

## Distribution et déploiement

### Package de distribution

**Créer un package** :
```bash
#!/bin/bash
# scripts/create_package.sh - Créer un package

create_package() {
    local package_name="$1"
    local version="$2"
    local package_dir="packages/${package_name}-${version}"
    
    mkdir -p "$package_dir"
    
    # Copier les fichiers
    cp -r lib/ "$package_dir/"
    cp -r modules/ "$package_dir/" 2>/dev/null || true
    cp -r bin/ "$package_dir/" 2>/dev/null || true
    cp README.md "$package_dir/" 2>/dev/null || true
    
    # Créer le script d'installation
    cat > "$package_dir/install.sh" << 'EOF'
#!/bin/bash
INSTALL_DIR="${INSTALL_DIR:-/usr/local/lib/bash}"

mkdir -p "$INSTALL_DIR"
cp -r lib/* "$INSTALL_DIR/"
echo "Package installé dans $INSTALL_DIR"
EOF
    
    chmod +x "$package_dir/install.sh"
    
    # Créer l'archive
    tar -czf "packages/${package_name}-${version}.tar.gz" -C packages "${package_name}-${version}"
    
    echo "Package créé: packages/${package_name}-${version}.tar.gz"
}
```

## Gestionnaire de packages

### Système de gestion

**Gestionnaire simple** :
```bash
#!/bin/bash
# lib/package_manager.sh - Gestionnaire de packages

install_package() {
    local package_file="$1"
    local install_dir="${2:-/usr/local/lib/bash}"
    
    if [ ! -f "$package_file" ]; then
        log_error "Fichier de package non trouvé: $package_file"
        return 1
    fi
    
    # Extraire
    local temp_dir=$(mktemp -d)
    tar -xzf "$package_file" -C "$temp_dir"
    
    # Installer
    if [ -f "$temp_dir"/*/install.sh ]; then
        INSTALL_DIR="$install_dir" bash "$temp_dir"/*/install.sh
    else
        mkdir -p "$install_dir"
        cp -r "$temp_dir"/*/lib/* "$install_dir/"
    fi
    
    rm -rf "$temp_dir"
    log_info "Package installé dans $install_dir"
}

list_installed_packages() {
    local install_dir="${1:-/usr/local/lib/bash}"
    
    echo "Packages installés dans $install_dir:"
    ls -1 "$install_dir" | while read module; do
        if [ -f "$install_dir/$module" ]; then
            local version=$(grep -E '^declare -r.*VERSION=' "$install_dir/$module" | head -1 | sed 's/.*VERSION="\(.*\)".*/\1/')
            echo "  - $module (version: ${version:-unknown})"
        fi
    done
}
```

## Maintenance et évolution

### Migration de versions

**Script de migration** :
```bash
#!/bin/bash
# scripts/migrate.sh - Migration entre versions

migrate_module() {
    local module_name="$1"
    local from_version="$2"
    local to_version="$3"
    
    log_info "Migration de $module_name de $from_version vers $to_version"
    
    # Sauvegarder la configuration
    backup_config "$module_name"
    
    # Charger la nouvelle version
    load_module_version "$module_name" "$to_version"
    
    # Exécuter les scripts de migration
    if [ -f "migrations/${module_name}_${from_version}_to_${to_version}.sh" ]; then
        source "migrations/${module_name}_${from_version}_to_${to_version}.sh"
    fi
    
    log_info "Migration terminée"
}
```

## Bonnes pratiques

### Recommandations

**Bonnes pratiques** :
```bash
#!/bin/bash
# Bonnes pratiques pour les modules

# 1. Préfixer toutes les fonctions publiques
#    - mymodule_function_name()

# 2. Préfixer les fonctions privées par _
#    - _mymodule_internal_function()

# 3. Préfixer les variables globales
#    - MYMODULE_CONFIG

# 4. Documenter toutes les fonctions publiques
#    - Commentaires avant chaque fonction

# 5. Gérer les erreurs proprement
#    - Codes de retour cohérents
#    - Messages d'erreur clairs

# 6. Tests pour chaque module
#    - Tests unitaires
#    - Tests d'intégration

# 7. Versioning
#    - Numérotation sémantique
#    - Changelog

# 8. Isolation
#    - Pas de variables globales sauf nécessaires
#    - Fonctions privées pour l'implémentation
```

## Scripts d'automatisation

### Gestionnaire complet

**Gestionnaire de modules** :
```bash
#!/bin/bash
# bin/module_manager.sh - Gestionnaire complet de modules

set -euo pipefail

MODULE_MANAGER_HOME="/opt/module-manager"
MODULE_LIB_DIR="${MODULE_MANAGER_HOME}/lib"

# Charger les utilitaires
source "$MODULE_MANAGER_HOME/lib/loader.sh"
source "$MODULE_MANAGER_HOME/lib/versioning.sh"

# Fonction principale
main() {
    local action="$1"
    shift
    
    case "$action" in
        install)
            install_module "$@"
            ;;
        remove)
            remove_module "$@"
            ;;
        list)
            list_modules "$@"
            ;;
        update)
            update_module "$@"
            ;;
        test)
            test_module "$@"
            ;;
        *)
            echo "Usage: $0 {install|remove|list|update|test}"
            exit 1
            ;;
    esac
}

install_module() {
    local module_name="$1"
    local version="${2:-latest}"
    
    log_info "Installation du module: $module_name (version: $version)"
    
    if load_with_dependencies "$module_name" "$version"; then
        log_info "Module installé avec succès"
    else
        log_error "Échec de l'installation"
        exit 1
    fi
}

# Utilisation
# module_manager.sh install logging 1.0.0
# module_manager.sh list
# module_manager.sh update logging 2.0.0
```

## Conclusion

Le scripting modulaire transforme les scripts simples en architectures complexes et maintenables. En créant des bibliothèques réutilisables, en gérant les dépendances proprement, et en suivant les bonnes pratiques, vous pouvez construire des systèmes qui évoluent facilement et qui sont faciles à maintenir.

Un système bien modulaire utilise des interfaces claires, une documentation complète, et des tests exhaustifs. La compréhension approfondie de ces mécanismes permet de créer des bibliothèques qui peuvent être réutilisées dans de multiples projets et qui facilitent la collaboration entre développeurs.

Dans le chapitre suivant, nous explorerons la gestion des processus zombies, découvrant comment identifier, comprendre, et résoudre ces processus orphelins qui peuvent consommer des ressources système.
