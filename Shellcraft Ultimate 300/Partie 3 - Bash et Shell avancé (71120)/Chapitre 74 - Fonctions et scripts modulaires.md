# Chapitre 74 - Fonctions et scripts modulaires

## Table des matières
- [Introduction](#introduction)
- [Fonctions de base en Bash](#fonctions-de-base-en-bash)
- [Paramètres et arguments avancés](#paramètres-et-arguments-avancés)
- [Portée des variables et environnements](#portée-des-variables-et-environnements)
- [Fonctions récursives et mémoïsation](#fonctions-récursives-et-mémoïsation)
- [Modules et bibliothèques](#modules-et-bibliothèques)
- [Gestion des erreurs dans les fonctions](#gestion-des-erreurs-dans-les-fonctions)
- [Tests et validation des fonctions](#tests-et-validation-des-fonctions)
- [Conclusion](#conclusion)

## Introduction

Les fonctions et la modularité constituent l'épine dorsale de tout programme Bash maintenable et évolutif. Au-delà de simples regroupements de commandes, les fonctions permettent de créer des bibliothèques réutilisables, de structurer le code de manière logique, et de faciliter les tests et la maintenance.

Imaginez les fonctions comme les pièces d'un jeu de construction sophistiqué : chaque pièce a un rôle précis, peut être combinée avec d'autres, et contribue à la construction d'édifices complexes et stables. La modularité transforme le chaos du code linéaire en une architecture élégante et maintenable.

## Fonctions de base en Bash

### Déclaration et syntaxe

**Syntaxe de déclaration** :
```bash
# Fonction avec le mot-clé function
function nom_fonction {
    # Corps de la fonction
    echo "Hello World"
}

# Syntaxe alternative sans mot-clé
nom_fonction() {
    # Corps de la fonction
    echo "Hello World"
}

# Fonction en une ligne
saluer() { echo "Bonjour $1!"; }
```

**Appel de fonctions** :
```bash
# Déclaration
afficher_message() {
    local niveau="$1"
    local message="$2"
    echo "[$niveau] $message"
}

# Appel
afficher_message "INFO" "Démarrage du script"
afficher_message "ERROR" "Fichier introuvable"
```

**Fonctions avec corps complexe** :
```bash
# Fonction multi-lignes
traiter_fichier() {
    local fichier="$1"
    local action="$2"

    # Vérifications préalables
    if [[ ! -f "$fichier" ]]; then
        echo "Erreur: Fichier $fichier n'existe pas" >&2
        return 1
    fi

    # Traitement selon l'action
    case "$action" in
        compter)
            wc -l < "$fichier"
            ;;
        taille)
            du -h "$fichier"
            ;;
        type)
            file "$fichier"
            ;;
        *)
            echo "Action inconnue: $action" >&2
            return 1
            ;;
    esac
}

# Utilisation
traiter_fichier "/etc/passwd" "compter"
traiter_fichier "mon_script.sh" "taille"
```

### Valeurs de retour

**Codes de retour explicites** :
```bash
# Fonction avec gestion d'erreur explicite
valider_fichier() {
    local fichier="$1"

    # Vérification de l'existence
    if [[ ! -f "$fichier" ]]; then
        echo "Fichier inexistant: $fichier" >&2
        return 1  # Erreur
    fi

    # Vérification de la lisibilité
    if [[ ! -r "$fichier" ]]; then
        echo "Fichier non lisible: $fichier" >&2
        return 2  # Erreur différente
    fi

    # Vérification du contenu
    if [[ ! -s "$fichier" ]]; then
        echo "Fichier vide: $fichier" >&2
        return 3  # Encore une erreur différente
    fi

    echo "Fichier valide: $fichier"
    return 0  # Succès
}

# Utilisation avec gestion des codes de retour
if valider_fichier "config.txt"; then
    echo "Validation réussie"
else
    code_retour=$?
    case $code_retour in
        1) echo "Fichier inexistant" ;;
        2) echo "Fichier non lisible" ;;
        3) echo "Fichier vide" ;;
        *) echo "Erreur inconnue: $code_retour" ;;
    esac
fi
```

**Retour de valeurs via echo** :
```bash
# Fonction retournant une valeur
calculer_hash() {
    local fichier="$1"
    local algo="${2:-sha256}"

    if [[ ! -f "$fichier" ]]; then
        echo "ERREUR:FICHIER_INEXISTANT" >&2
        return 1
    fi

    # Calcul du hash
    case "$algo" in
        md5)    hash=$(md5sum "$fichier" | cut -d' ' -f1) ;;
        sha1)   hash=$(sha1sum "$fichier" | cut -d' ' -f1) ;;
        sha256) hash=$(sha256sum "$fichier" | cut -d' ' -f1) ;;
        *)      echo "Algorithme non supporté: $algo" >&2; return 1 ;;
    esac

    echo "$hash"
    return 0
}

# Utilisation
hash_fichier=$(calculer_hash "important.txt" "sha256")
if [[ $? -eq 0 ]]; then
    echo "Hash du fichier: $hash_fichier"
else
    echo "Erreur lors du calcul du hash"
fi
```

## Paramètres et arguments avancés

### Gestion des paramètres positionnels

**Accès aux paramètres spéciaux** :
```bash
# Fonction avec paramètres variables
afficher_parametres() {
    echo "Nombre de paramètres: $#"
    echo "Tous les paramètres: $@"
    echo "Tous comme chaîne: $*"
    echo "Nom du script/fonction: $0"

    # Itération sur les paramètres
    local i=1
    for param in "$@"; do
        echo "Paramètre $i: $param"
        ((i++))
    done
}

# Test
afficher_parametres "un" "deux" "trois mots"
```

**Shift et traitement séquentiel** :
```bash
# Fonction de parsing d'options
traiter_options() {
    local verbose=false
    local output_file=""
    local input_files=()

    while [[ $# -gt 0 ]]; do
        case "$1" in
            -v|--verbose)
                verbose=true
                shift
                ;;
            -o|--output)
                if [[ -z "$2" || "$2" == -* ]]; then
                    echo "Erreur: -o nécessite un argument" >&2
                    return 1
                fi
                output_file="$2"
                shift 2
                ;;
            -*)
                echo "Option inconnue: $1" >&2
                return 1
                ;;
            *)
                # Fichier d'entrée
                input_files+=("$1")
                shift
                ;;
        esac
    done

    # Affichage des résultats
    echo "Verbose: $verbose"
    echo "Fichier de sortie: ${output_file:-stdout}"
    echo "Fichiers d'entrée: ${input_files[*]}"
}

# Utilisation
traiter_options -v -o resultat.txt fichier1.txt fichier2.txt
```

### Paramètres nommés et options longues

**Simulation des paramètres nommés** :
```bash
# Fonction avec paramètres nommés simulés
creer_utilisateur() {
    # Valeurs par défaut
    local username=""
    local uid=""
    local gid=""
    local home_dir=""
    local shell="/bin/bash"

    # Parsing des paramètres nommés
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --username)
                username="$2"
                shift 2
                ;;
            --uid)
                uid="$2"
                shift 2
                ;;
            --gid)
                gid="$2"
                shift 2
                ;;
            --home)
                home_dir="$2"
                shift 2
                ;;
            --shell)
                shell="$2"
                shift 2
                ;;
            *)
                echo "Paramètre inconnu: $1" >&2
                return 1
                ;;
        esac
    done

    # Validation
    if [[ -z "$username" ]]; then
        echo "Erreur: --username requis" >&2
        return 1
    fi

    # Création de l'utilisateur (simulation)
    echo "Création de l'utilisateur:"
    echo "  Username: $username"
    echo "  UID: ${uid:-auto}"
    echo "  GID: ${gid:-auto}"
    echo "  Home: ${home_dir:-/home/$username}"
    echo "  Shell: $shell"
}

# Utilisation
creer_utilisateur --username john --uid 1500 --shell /bin/zsh
```

**Bibliothèque de parsing d'options avancées** :
```bash
# Fonction de parsing générique
parse_options() {
    local -n _options="$1"  # Nameref pour modifier le tableau associatif
    local -a _args=()

    while [[ $# -gt 0 ]]; do
        case "$1" in
            --*=*)
                # Option avec valeur: --key=value
                local key="${1%%=*}"
                local value="${1#*=}"
                _options["${key#--}"]="$value"
                shift
                ;;
            --*)
                # Option booléenne ou avec valeur suivante
                local key="${1#--}"
                if [[ $# -gt 1 && "$2" != --* ]]; then
                    _options["$key"]="$2"
                    shift 2
                else
                    _options["$key"]=true
                    shift
                fi
                ;;
            -*)
                # Option courte (non implémenté complètement ici)
                echo "Options courtes non supportées: $1" >&2
                return 1
                ;;
            *)
                # Argument positionnel
                _args+=("$1")
                shift
                ;;
        esac
    done

    # Retourner les arguments positionnels via variable globale
    ARGS_POSITIONNELS=("${_args[@]}")
}

# Utilisation
declare -A options
parse_options options --verbose --output=result.txt --format=json fichier1.txt

echo "Options parsées:"
for key in "${!options[@]}"; do
    echo "  $key = ${options[$key]}"
done

echo "Arguments positionnels: ${ARGS_POSITIONNELS[@]}"
```

## Portée des variables et environnements

### Variables locales et globales

**Utilisation de local** :
```bash
# Fonction avec variables locales
traiter_donnees() {
    local compteur=0
    local temp_file=$(mktemp)
    local -a resultats=()

    # Traitement...
    while IFS= read -r ligne; do
        # Utilisation des variables locales
        traiter_ligne "$ligne" "$temp_file"
        ((compteur++))
        resultats+=("$ligne")
    done

    echo "Traitées: $compteur lignes"

    # Nettoyage
    rm -f "$temp_file"
}

# Variables globales non affectées
compteur=100
traiter_donnees < donnees.txt
echo "Compteur global toujours: $compteur"
```

**Variables readonly dans les fonctions** :
```bash
# Définition de constantes
definir_constantes() {
    readonly SCRIPT_VERSION="1.2.3"
    readonly CONFIG_DIR="${CONFIG_DIR:-/etc/app}"
    readonly LOG_LEVEL="${LOG_LEVEL:-INFO}"
}

# Appel une fois au début du script
definir_constantes

# Tentative de modification (erreur)
SCRIPT_VERSION="1.2.4"  # Erreur: readonly variable
```

### Environnements isolés

**Sous-shells pour isolation** :
```bash
# Fonction s'exécutant dans un sous-shell
traitement_isole() (
    # Toutes les variables sont locales au sous-shell
    local temp_dir=$(mktemp -d)
    local old_pwd=$PWD

    cd "$temp_dir"

    # Traitement isolé
    echo "Traitement dans: $PWD"
    creer_fichiers_temporaires
    compiler
    tester

    # Nettoyage automatique à la fin du sous-shell
    cd "$old_pwd"
)

# Variables du parent non affectées
var_parent="valeur_originale"
traitement_isole
echo "Variable parent toujours: $var_parent"
```

**Export temporaire d'environnement** :
```bash
# Fonction avec environnement contrôlé
executer_dans_environnement() {
    local -r env_file="$1"
    shift

    # Sauvegarde de l'environnement actuel
    local -A env_backup
    while IFS='=' read -r key value; do
        env_backup["$key"]="$value"
    done < <(env)

    # Chargement du nouvel environnement
    if [[ -f "$env_file" ]]; then
        set -a  # Auto-export des variables
        source "$env_file"
        set +a
    fi

    # Exécution de la commande
    "$@"

    # Restauration de l'environnement (optionnel)
    # Note: Compliqué à implémenter complètement
}

# Utilisation
executer_dans_environnement "env.prod" ./mon_app --verbose
```

## Fonctions récursives et mémoïsation

### Récursion en Bash

**Fonction récursive simple** :
```bash
# Calcul de factorielle
factorielle() {
    local n="$1"

    # Cas de base
    if (( n <= 1 )); then
        echo 1
        return
    fi

    # Cas récursif
    local precedente=$(factorielle $((n-1)))
    echo $((n * precedente))
}

# Utilisation
echo "5! = $(factorielle 5)"
```

**Récursion avec structures de données** :
```bash
# Parcours récursif de répertoire
parcourir_repertoire() {
    local repertoire="$1"
    local prefix="${2:-}"

    # Lister le contenu
    for item in "$repertoire"/*; do
        if [[ -d "$item" ]]; then
            echo "${prefix}📁 $(basename "$item")/"
            # Appel récursif avec indentation
            parcourir_repertoire "$item" "  $prefix"
        else
            echo "${prefix}📄 $(basename "$item")"
        fi
    done
}

# Utilisation
parcourir_repertoire "/home/user/projects"
```

**Limites de la récursion Bash** :
```bash
# Test des limites de récursion
test_recursion() {
    local profondeur="$1"

    if (( profondeur <= 0 )); then
        echo "Profondeur maximale atteinte"
        return
    fi

    # Appel récursif
    test_recursion $((profondeur - 1))
}

# Bash a généralement une limite d'environ 1000 appels récursifs
echo "Test de récursion profonde..."
test_recursion 100  # Devrait fonctionner
# test_recursion 2000  # Risque de dépassement de limite
```

### Mémoïsation et cache

**Cache automatique pour fonctions coûteuses** :
```bash
# Système de mémoïsation générique
declare -A _memo_cache=()
declare -A _memo_timestamps=()

# Fonction mémoïsée
memo() {
    local func_name="$1"
    local ttl="${2:-3600}"  # TTL par défaut 1h
    shift 2

    # Clé de cache basée sur le nom de fonction et les arguments
    local cache_key="$func_name:$(printf '%q ' "$@")"

    # Vérification du cache
    if [[ -v _memo_cache[$cache_key] ]] && [[ -v _memo_timestamps[$cache_key] ]]; then
        local age=$(( $(date +%s) - ${_memo_timestamps[$cache_key]} ))
        if (( age < ttl )); then
            echo "${_memo_cache[$cache_key]}"
            return 0
        fi
    fi

    # Calcul réel en appelant la fonction
    local result
    if result=$("$func_name" "$@"); then
        # Mise en cache
        _memo_cache[$cache_key]="$result"
        _memo_timestamps[$cache_key]=$(date +%s)
        echo "$result"
        return 0
    else
        return 1
    fi
}

# Fonction coûteuse à mémoïser
calcul_complexe() {
    local n="$1"
    echo "Calcul complexe pour $n..." >&2
    sleep 1  # Simulation
    echo $((n * n))
}

# Utilisation
echo "Premier appel: $(memo calcul_complexe 5)"
echo "Deuxième appel (caché): $(memo calcul_complexe 5)"
echo "Nouvelle valeur: $(memo calcul_complexe 10)"
```

## Modules et bibliothèques

### Chargement dynamique de fonctions

**Système de modules simple** :
```bash
# Fonction de chargement de module
charger_module() {
    local module="$1"

    # Chemins de recherche des modules
    local -a chemins=(
        "$HOME/.bash_modules"
        "/usr/local/lib/bash"
        "./modules"
    )

    for chemin in "${chemins[@]}"; do
        local fichier_module="$chemin/${module}.sh"
        if [[ -f "$fichier_module" ]]; then
            echo "Chargement du module: $module" >&2
            source "$fichier_module"
            return 0
        fi
    done

    echo "Module non trouvé: $module" >&2
    return 1
}

# Utilisation
charger_module "logging" || exit 1
charger_module "network" || exit 1

# Les fonctions des modules sont maintenant disponibles
logger "Script démarré"
get_external_ip
```

**Structure d'un module** :
```bash
#!/usr/bin/env bash
# Module: logging.sh

# Variables du module
LOG_LEVEL="${LOG_LEVEL:-INFO}"
LOG_FILE="${LOG_FILE:-/var/log/app.log}"

# Fonctions publiques du module
logger() {
    local level="$1"
    local message="$2"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

    # Vérification du niveau de log
    if ! niveau_suffisant "$level"; then
        return 0
    fi

    # Format du message
    local formatted_msg="[$timestamp] [$level] $message"

    # Écriture dans le fichier
    echo "$formatted_msg" >> "$LOG_FILE"

    # Affichage console si niveau DEBUG ou supérieur
    if [[ "$LOG_LEVEL" == "DEBUG" ]]; then
        echo "$formatted_msg" >&2
    fi
}

# Fonctions privées (préfixées par _)
niveau_suffisant() {
    local level="$1"

    case "$LOG_LEVEL" in
        DEBUG) return 0 ;;
        INFO)  [[ "$level" != "DEBUG" ]] ;;
        WARN)  [[ "$level" =~ (WARN|ERROR) ]] ;;
        ERROR) [[ "$level" == "ERROR" ]] ;;
        *)     return 1 ;;
    esac
}

# Initialisation du module
init_logging() {
    # Création du répertoire de logs si nécessaire
    local log_dir=$(dirname "$LOG_FILE")
    mkdir -p "$log_dir" 2>/dev/null || true

    # Test d'écriture
    if ! echo "[$timestamp] [INFO] Module logging initialisé" >> "$LOG_FILE" 2>/dev/null; then
        echo "Erreur: Impossible d'écrire dans $LOG_FILE" >&2
        return 1
    fi

    logger "INFO" "Module logging initialisé avec succès"
}

# Auto-initialisation
init_logging
```

### Gestionnaire de dépendances

**Système de dépendances entre modules** :
```bash
# Gestionnaire de modules avec dépendances
declare -A _modules_charges=()
declare -A _dependances_modules=(
    ["web"]="network logging"
    ["database"]="logging"
    ["api"]="web database cache"
    ["cache"]="logging"
)

charger_module_avec_dependances() {
    local module="$1"

    # Vérifier si déjà chargé
    if [[ -v _modules_charges[$module] ]]; then
        return 0
    fi

    # Charger les dépendances d'abord
    if [[ -v _dependances_modules[$module] ]]; then
        for dep in ${_dependances_modules[$module]}; do
            charger_module_avec_dependances "$dep"
        done
    fi

    # Charger le module principal
    if charger_module "$module"; then
        _modules_charges[$module]=true
        echo "Module $module chargé avec ses dépendances" >&2
        return 0
    else
        echo "Échec du chargement du module: $module" >&2
        return 1
    fi
}

# Utilisation
charger_module_avec_dependances "api"  # Charge automatiquement web, database, cache, logging
```

## Gestion des erreurs dans les fonctions

### Gestion d'erreurs structurée

**Fonction avec gestion d'erreurs complète** :
```bash
# Types d'erreurs
declare -r ERR_FILE_NOT_FOUND=1
declare -r ERR_PERMISSION_DENIED=2
declare -r ERR_INVALID_FORMAT=3
declare -r ERR_NETWORK_ERROR=4

# Fonction avec gestion d'erreurs
backup_database() {
    local db_name="$1"
    local backup_dir="$2"
    local -n _erreur="$3"  # Paramètre par référence pour l'erreur

    _erreur=""  # Réinitialisation

    # Validation des paramètres
    if [[ -z "$db_name" ]]; then
        _erreur="Nom de base de données requis"
        return $ERR_INVALID_FORMAT
    fi

    if [[ ! -d "$backup_dir" ]]; then
        _erreur="Répertoire de sauvegarde inexistant: $backup_dir"
        return $ERR_FILE_NOT_FOUND
    fi

    if [[ ! -w "$backup_dir" ]]; then
        _erreur="Pas de permission d'écriture: $backup_dir"
        return $ERR_PERMISSION_DENIED
    fi

    # Tentative de sauvegarde
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="$backup_dir/${db_name}_backup_$timestamp.sql"

    if ! mysqldump "$db_name" > "$backup_file" 2>/dev/null; then
        _erreur="Échec de la sauvegarde MySQL"
        return $ERR_NETWORK_ERROR
    fi

    # Compression
    if ! gzip "$backup_file"; then
        _erreur="Échec de la compression"
        rm -f "$backup_file"  # Nettoyage du fichier non compressé
        return $ERR_PERMISSION_DENIED
    fi

    echo "Sauvegarde réussie: ${backup_file}.gz"
    return 0
}

# Utilisation avec gestion d'erreurs
erreur_message=""
if backup_database "myapp" "/backups" erreur_message; then
    echo "✓ Sauvegarde terminée"
else
    code_erreur=$?
    echo "✗ Échec de sauvegarde (code $code_erreur): $erreur_message"

    # Action corrective selon le type d'erreur
    case $code_erreur in
        $ERR_FILE_NOT_FOUND)
            echo "Création du répertoire manquant..."
            mkdir -p "/backups"
            ;;
        $ERR_PERMISSION_DENIED)
            echo "Vérification des permissions..."
            ls -ld "/backups"
            ;;
        *)
            echo "Erreur non récupérable"
            exit 1
            ;;
    esac
fi
```

### Pattern try-catch simulé

**Simulation de try-catch en Bash** :
```bash
# Fonction try-catch simulée
try() {
    local func="$1"
    shift

    # Exécution dans un sous-shell pour capturer les erreurs
    if ( set -e; "$func" "$@" ); then
        return 0
    else
        local code_erreur=$?
        # Gestionnaire d'erreurs personnalisé
        catch "$func" $code_erreur "$@"
        return $code_erreur
    fi
}

catch() {
    local func="$1"
    local code_erreur="$2"
    shift 2

    echo "Exception dans $func (code $code_erreur)" >&2

    case $code_erreur in
        1)  echo "Erreur générique" >&2 ;;
        2)  echo "Fichier introuvable" >&2 ;;
        127) echo "Commande introuvable" >&2 ;;
        *)  echo "Erreur inconnue: $code_erreur" >&2 ;;
    esac
}

# Fonctions pouvant échouer
operation_risquée() {
    local fichier="$1"

    if [[ ! -f "$fichier" ]]; then
        echo "Fichier introuvable: $fichier" >&2
        return 2
    fi

    # Simulation d'une opération risquée
    if (( RANDOM % 3 == 0 )); then
        echo "Échec aléatoire simulé" >&2
        return 1
    fi

    echo "Opération réussie sur $fichier"
}

# Utilisation
try operation_risquée "important.txt"
try operation_risquée "inexistant.txt"
```

## Tests et validation des fonctions

### Framework de test simple

**Tests unitaires pour fonctions** :
```bash
# Framework de test simple
declare -i tests_reussis=0
declare -i tests_echoues=0

# Fonction d'assertion
assert() {
    local condition="$1"
    local message="${2:-Assertion échouée}"

    if eval "$condition"; then
        echo "✓ $message"
        ((tests_reussis++))
    else
        echo "✗ $message"
        ((tests_echoues++))
    fi
}

assert_equals() {
    local attendu="$1"
    local obtenu="$2"
    local message="${3:-Valeur incorrecte}"

    if [[ "$attendu" == "$obtenu" ]]; then
        echo "✓ $message: '$obtenu'"
        ((tests_reussis++))
    else
        echo "✗ $message: attendu '$attendu', obtenu '$obtenu'"
        ((tests_echoues++))
    fi
}

# Fonctions à tester
additionner() {
    echo $(($1 + $2))
}

saluer() {
    local nom="$1"
    local langue="${2:-fr}"

    case "$langue" in
        fr) echo "Bonjour $nom" ;;
        en) echo "Hello $nom" ;;
        es) echo "Hola $nom" ;;
        *)  echo "Salut $nom" ;;
    esac
}

# Tests
echo "=== Tests des fonctions ==="

# Tests de additionner
assert_equals "5" "$(additionner 2 3)" "Addition simple"
assert_equals "0" "$(additionner -1 1)" "Addition avec négatifs"
assert_equals "100" "$(additionner 50 50)" "Addition plus grande"

# Tests de saluer
assert_equals "Bonjour Alice" "$(saluer Alice)" "Salutation française par défaut"
assert_equals "Hello Bob" "$(saluer Bob en)" "Salutation anglaise"
assert_equals "Hola Carlos" "$(saluer Carlos es)" "Salutation espagnole"
assert_equals "Salut Dave" "$(saluer Dave it)" "Salutation inconnue"

# Résumé
echo
echo "=== Résumé des tests ==="
echo "Réussis: $tests_reussis"
echo "Échoués: $tests_echoues"
echo "Total: $((tests_reussis + tests_echoues))"
```

### Tests d'intégration

**Tests de fonctions interdépendantes** :
```bash
# Tests d'intégration pour modules
test_module_logging() {
    echo "=== Tests du module logging ==="

    # Test de chargement
    if charger_module "logging" 2>/dev/null; then
        echo "✓ Module logging chargé"
    else
        echo "✗ Échec du chargement du module logging"
        return 1
    fi

    # Test des fonctions
    local log_file="/tmp/test_log.txt"
    LOG_FILE="$log_file"

    logger "INFO" "Test de logging"
    logger "ERROR" "Test d'erreur"

    # Vérification du contenu
    if [[ -f "$log_file" ]] && grep -q "Test de logging" "$log_file"; then
        echo "✓ Messages de log écrits"
    else
        echo "✗ Messages de log manquants"
        return 1
    fi

    # Nettoyage
    rm -f "$log_file"
    echo "✓ Tests du module logging réussis"
    return 0
}

test_integration_api() {
    echo "=== Tests d'intégration API ==="

    # Simulation de l'API
    api_call() {
        local endpoint="$1"
        case "$endpoint" in
            "/health") echo '{"status": "ok"}' ;;
            "/users")  echo '[{"id": 1, "name": "Alice"}]' ;;
            *)         echo '{"error": "Endpoint inconnu"}' >&2; return 1 ;;
        esac
    }

    # Tests
    local response=$(api_call "/health")
    assert_equals '{"status": "ok"}' "$response" "Endpoint health"

    response=$(api_call "/users")
    if [[ "$response" == *'"name": "Alice"'* ]]; then
        echo "✓ Endpoint users"
    else
        echo "✗ Endpoint users: réponse incorrecte"
    fi
}

# Exécution des tests
test_module_logging
test_integration_api
```

## Conclusion

Les fonctions et la modularité transforment Bash d'un langage de scripting simple en un environnement de programmation structuré et maintenable. En maîtrisant la déclaration de fonctions, la gestion des paramètres, la portée des variables, et les patterns de modularité, vous pouvez créer des bibliothèques réutilisables et des applications complexes.

Comme un architecte qui conçoit des bâtiments modulaires, le programmeur Bash expérimenté construit des systèmes où chaque fonction a un rôle précis, les interactions sont clairement définies, et l'évolution du code devient naturelle et sans risque.

Dans le chapitre suivant, nous explorerons les scripts interactifs, découvrant comment créer des interfaces utilisateur dans le terminal pour rendre vos scripts plus conviviaux et puissants.
