# Chapitre 71 - Variables avancées

## Table des matières
- [Introduction](#introduction)
- [Types de variables en Bash](#types-de-variables-en-bash)
- [Variables tableaux](#variables-tableaux)
- [Variables associatives](#variables-associatives)
- [Portée et visibilité des variables](#portée-et-visibilité-des-variables)
- [Variables spéciales et prédéfinies](#variables-spéciales-et-prédéfinies)
- [Manipulation avancée des variables](#manipulation-avancée-des-variables)
- [Débogage des variables](#débogage-des-variables)
- [Conclusion](#conclusion)

## Introduction

Les variables constituent le fondement de la programmation Bash, mais au-delà des simples affectations de chaînes, Bash offre des structures de données sophistiquées qui transforment les scripts en véritables programmes. Maîtriser les variables avancées permet de créer des scripts plus expressifs, maintenables, et performants.

Imaginez les variables avancées comme une palette de couleurs étendue pour un artiste : les variables simples sont comme le noir et blanc, tandis que les tableaux et variables associatives offrent toute la richesse du spectre chromatique, permettant de créer des œuvres complexes et nuancées.

## Types de variables en Bash

### Variables scalaires (simples)

**Déclaration et affectation** :
```bash
# Variables de chaînes
nom="Alice"
age="25"
chemin="/home/alice/documents"

# Variables numériques (stockées comme chaînes)
compteur=0
prix="19.99"

# Variables booléennes (convention)
actif=true
debug=false
```

**Types implicites** :
```bash
# Bash traite tout comme des chaînes
nombre=42
resultat=$((nombre + 8))  # 50

# Concaténation automatique
salutation="Bonjour $nom"  # Bonjour Alice
```

### Variables readonly

**Protection contre la modification** :
```bash
# Déclaration readonly
readonly PI=3.14159
readonly CONFIG_FILE="/etc/app/config"

# Tentative de modification (erreur)
PI=3.14  # bash: PI: readonly variable

# Fonction utile pour la protection
make_readonly() {
    readonly "$1"
}

# Utilisation
make_readonly DATABASE_URL
```

## Variables tableaux

### Tableaux indexés

**Déclaration et initialisation** :
```bash
# Syntaxe déclarative
declare -a utilisateurs

# Initialisation directe
serveurs=("web01" "web02" "db01" "cache01")

# Initialisation avec indices spécifiques
fichiers=([0]="readme.txt" [1]="install.sh" [10]="config.ini")

# Syntaxe compacte
nombres=(1 2 3 4 5 6 7 8 9 10)
```

**Accès aux éléments** :
```bash
# Accès individuel
echo "${serveurs[0]}"    # web01
echo "${serveurs[2]}"    # db01

# Accès à tous les éléments
echo "${serveurs[@]}"    # web01 web02 db01 cache01

# Nombre d'éléments
echo "${#serveurs[@]}"   # 4
echo "${#serveurs}"      # Taille du premier élément (5)

# Dernier élément
echo "${serveurs[-1]}"   # cache01 (Bash 4.1+)
```

**Manipulation des tableaux** :
```bash
# Ajout d'éléments
serveurs+=("backup01")
serveurs[10]="monitoring01"

# Suppression d'éléments
unset 'serveurs[1]'      # Supprime web02

# Tranchage (slicing)
echo "${serveurs[@]:1:2}" # web02 db01 (à partir de l'index 1, 2 éléments)

# Recherche
index=$(echo "${serveurs[@]}" | grep -o "db01" | wc -l)
```

### Tableaux multidimensionnels

**Simulation avec des chaînes délimitées** :
```bash
# Matrice 2D simulée
declare -a matrice
matrice[0]="1,2,3"
matrice[1]="4,5,6"
matrice[2]="7,8,9"

# Accès aux éléments
ligne1="${matrice[1]}"    # "4,5,6"
element="${ligne1%%,*}"   # "4" (premier élément)

# Fonction d'accès générique
get_element() {
    local matrice_name="$1"
    local row="$2"
    local col="$3"

    local ligne="${matrice_name}[$row]"
    local valeur="${!ligne}"

    # Extraire l'élément à la colonne col
    echo "$valeur" | cut -d',' -f$((col+1))
}
```

## Variables associatives

### Déclaration et utilisation de base

**Création de tableaux associatifs** :
```bash
# Déclaration explicite
declare -A utilisateur
declare -A configuration

# Initialisation
utilisateur=(
    [nom]="Alice Dupont"
    [age]="28"
    [email]="alice@example.com"
    [role]="admin"
)

# Accès aux valeurs
echo "${utilisateur[nom]}"    # Alice Dupont
echo "${utilisateur[email]}"  # alice@example.com
```

**Opérations courantes** :
```bash
# Ajout/modification
utilisateur[departement]="IT"
utilisateur[telephone]="01-23-45-67-89"

# Vérification d'existence
if [[ -v utilisateur[salaire] ]]; then
    echo "Salaire défini: ${utilisateur[salaire]}"
else
    echo "Salaire non défini"
fi

# Liste des clés
echo "${!utilisateur[@]}"
# nom age email role departement telephone

# Liste des valeurs
echo "${utilisateur[@]}"
# Alice Dupont 28 alice@example.com admin IT 01-23-45-67-89
```

### Applications pratiques

**Configuration d'application** :
```bash
declare -A config=(
    [db_host]="localhost"
    [db_port]="5432"
    [db_name]="myapp"
    [cache_ttl]="3600"
    [log_level]="INFO"
)

# Fonction de configuration
get_config() {
    local key="$1"
    local default="$2"

    if [[ -v config[$key] ]]; then
        echo "${config[$key]}"
    else
        echo "$default"
    fi
}

# Utilisation
host=$(get_config "db_host" "127.0.0.1")
port=$(get_config "db_port" "3306")
```

**Cache en mémoire** :
```bash
declare -A cache
declare -A cache_timestamps

cache_get() {
    local key="$1"
    local ttl="${2:-3600}"  # TTL par défaut 1h

    # Vérifier si la clé existe et n'est pas expirée
    if [[ -v cache[$key] ]] && [[ -v cache_timestamps[$key] ]]; then
        local age=$(( $(date +%s) - ${cache_timestamps[$key]} ))
        if (( age < ttl )); then
            echo "${cache[$key]}"
            return 0
        fi
    fi
    return 1  # Cache miss
}

cache_set() {
    local key="$1"
    local value="$2"

    cache[$key]="$value"
    cache_timestamps[$key]=$(date +%s)
}
```

## Portée et visibilité des variables

### Variables locales et globales

**Portée globale par défaut** :
```bash
# Variable globale
GLOBAL_VAR="accessible partout"

fonction_test() {
    echo "Dans fonction: $GLOBAL_VAR"
    # Modification affecte la globale
    GLOBAL_VAR="modifiée dans fonction"
}

echo "Avant: $GLOBAL_VAR"
fonction_test
echo "Après: $GLOBAL_VAR"
```

**Variables locales avec local** :
```bash
compteur_global=0

incrementer() {
    local compteur_local=0
    ((compteur_global++))
    ((compteur_local++))

    echo "Global: $compteur_global, Local: $compteur_local"
}

incrementer  # Global: 1, Local: 1
incrementer  # Global: 2, Local: 1
echo "Final global: $compteur_global"
```

**Portée dans les sous-shells** :
```bash
# Variable dans le shell parent
VAR_PARENT="visible"

# Sous-shell (nouvelle instance Bash)
(
    echo "Sous-shell voit: $VAR_PARENT"
    VAR_ENFANT="invisible au parent"
)

echo "Parent ne voit pas: $VAR_ENFANT"
```

### Variables d'environnement

**Export vers l'environnement** :
```bash
# Variable locale uniquement
variable_locale="non exportée"

# Export explicite
export VARIABLE_ENV="visible pour les enfants"

# Combiné
export VAR_COMBINEE="valeur"

# Vérifier les variables exportées
export -p | grep VARIABLE_ENV
```

**Variables d'environnement système** :
```bash
# Variables importantes
echo "Utilisateur: $USER"
echo "Home: $HOME"
echo "Path: $PATH"
echo "Shell: $SHELL"
echo "PID: $$"
echo "PPID: $PPID"
```

## Variables spéciales et prédéfinies

### Variables de position ($1, $2, ...)

**Arguments de script/fonction** :
```bash
#!/bin/bash
# script.sh arg1 arg2 arg3

echo "Script: $0"
echo "Premier argument: $1"
echo "Deuxième argument: $2"
echo "Nombre d'arguments: $#"
echo "Tous les arguments: $@"
echo "Tous comme une chaîne: $*"

# Itération sur les arguments
for arg in "$@"; do
    echo "Argument: $arg"
done
```

**Shift pour traiter séquentiellement** :
```bash
traiter_options() {
    while [[ $# -gt 0 ]]; do
        case "$1" in
            -v|--verbose)
                verbose=true
                shift
                ;;
            -o|--output)
                output_file="$2"
                shift 2  # Consomme l'option et sa valeur
                ;;
            *)
                echo "Option inconnue: $1"
                return 1
                ;;
        esac
    done
}
```

### Variables spéciales de Bash

**Variables de statut et contrôle** :
```bash
# Statut de la dernière commande
echo "Dernière commande: $?"  # 0 = succès, autre = échec

# PID du shell courant
echo "Mon PID: $$"

# PID du dernier job en arrière-plan
sleep 10 &
echo "Job en arrière-plan: $!"

# Options du shell
echo "Options: $-"

# Numéro de ligne (pour débogage)
echo "Ligne: $LINENO"

# Fonction appelante
fonction_a() {
    fonction_b
}

fonction_b() {
    echo "Appelée depuis: ${FUNCNAME[1]}"
}

fonction_a  # Appelée depuis: fonction_a
```

### Variables de tableau spéciales

**$@ vs $* (différence cruciale)** :
```bash
#!/bin/bash

demontrer_arguments() {
    echo "=== Avec \$* ==="
    for arg in $*; do
        echo "[$arg]"
    done

    echo "=== Avec \$@ ==="
    for arg in $@; do
        echo "[$arg]"
    done

    echo "=== Avec \"\$*\" ==="
    for arg in "$*"; do
        echo "[$arg]"
    done

    echo "=== Avec \"\$@\" ==="
    for arg in "$@"; do
        echo "[$arg]"
    done
}

# Test avec arguments contenant des espaces
demontrer_arguments "hello world" "bash scripting" "trois mots"
```

## Manipulation avancée des variables

### Expansion de variables

**Expansion de paramètres** :
```bash
# Valeur par défaut
nom=${NOM_UTILISATEUR:-"Anonyme"}
echo "Bienvenue $nom"

# Valeur par défaut si vide ou non définie
fichier=${FICHIER_CONFIG:-"/etc/default/config"}

# Erreur si non définie
: ${VARIABLE_REQUISE:?"Variable obligatoire manquante"}

# Utilisation de la valeur si définie
prefixe=${ENV_PREFIX:+${ENV_PREFIX}_}
echo "Table: ${prefixe}utilisateurs"
```

**Expansion de sous-chaînes** :
```bash
chemin="/home/alice/documents/rapport.txt"

# Nom de base
basename "${chemin##*/}"  # rapport.txt

# Répertoire
dirname "${chemin%/*}"     # /home/alice/documents

# Extension
extension="${chemin##*.}"  # txt

# Nom sans extension
nom="${chemin%.*}"         # /home/alice/documents/rapport
```

### Transformations de casse

**Opérations sur la casse** :
```bash
texte="Hello World Bash"

# Minuscules
echo "${texte,,}"  # hello world bash

# Majuscules
echo "${texte^^}"  # HELLO WORLD BASH

# Première lettre en majuscule
echo "${texte^}"   # Hello World Bash

# Première lettre de chaque mot
echo "${texte^^[h]}"  # Non standard, utiliser sed/awk
```

### Recherche et remplacement

**Substitution de motifs** :
```bash
url="https://example.com/api/v1/users"

# Remplacement simple
api_url="${url//v1/v2}"  # https://example.com/api/v2/users

# Remplacement préfixe
secure_url="${url/#http/https}"  # https://example.com/api/v1/users

# Remplacement suffixe
backup_file="${url/%.txt/.bak}"  # Si c'était un .txt

# Suppression de préfixe
endpoint="${url#https://*/}"     # api/v1/users

# Suppression de suffixe
base_url="${url%/*}"             # https://example.com/api/v1
```

## Débogage des variables

### Inspection des variables

**Affichage détaillé** :
```bash
# Lister toutes les variables
declare -p

# Variables d'un type spécifique
declare -a  # Tableaux indexés
declare -A  # Tableaux associatifs
declare -i  # Variables entières
declare -r  # Variables readonly

# Fonction d'inspection
inspect_variable() {
    local var_name="$1"

    if declare -p "$var_name" 2>/dev/null; then
        echo "Variable $var_name existe"
        declare -p "$var_name"
    else
        echo "Variable $var_name n'existe pas"
    fi
}
```

**Débogage de tableaux** :
```bash
debug_array() {
    local array_name="$1"

    echo "=== Débogage du tableau: $array_name ==="

    # Vérifier si c'est un tableau
    if [[ "$(declare -p "$array_name" 2>/dev/null)" =~ "declare -a" ]]; then
        echo "Type: Tableau indexé"

        # Récupérer la déclaration
        local decl="$(declare -p "$array_name")"

        # Extraire les éléments (méthode simplifiée)
        eval "local -a temp_array=$decl"
        local temp_array

        echo "Nombre d'éléments: ${#temp_array[@]}"
        echo "Éléments:"
        local i
        for ((i=0; i<${#temp_array[@]}; i++)); do
            echo "  [$i] = '${temp_array[$i]}'"
        done
    else
        echo "Erreur: $array_name n'est pas un tableau"
    fi
}
```

### Validation de variables

**Fonctions de validation** :
```bash
# Validation de type
is_integer() {
    [[ "$1" =~ ^-?[0-9]+$ ]]
}

is_array() {
    [[ "$(declare -p "$1" 2>/dev/null)" =~ "declare -a" ]]
}

is_associative_array() {
    [[ "$(declare -p "$1" 2>/dev/null)" =~ "declare -A" ]]
}

# Validation de contenu
is_valid_email() {
    [[ "$1" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]
}

is_valid_ip() {
    [[ "$1" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]] && {
        local IFS='.' ip=($1)
        [[ ${ip[0]} -le 255 && ${ip[1]} -le 255 && ${ip[2]} -le 255 && ${ip[3]} -le 255 ]]
    }
}

# Utilisation
if is_valid_email "$user_email"; then
    echo "Email valide"
else
    echo "Email invalide"
fi
```

## Conclusion

Les variables avancées de Bash transforment l'écriture de scripts d'une activité simple en un art de la programmation système. Des tableaux aux variables associatives, en passant par la gestion fine de la portée et les expansions sophistiquées, ces fonctionnalités permettent de créer des scripts qui rivalisent en puissance et élégance avec des programmes compilés.

La clé de la maîtrise réside dans la compréhension profonde des mécanismes d'expansion, de portée, et de manipulation. Comme un artisan qui connaît intimement ses outils, le programmeur Bash expérimenté utilise chaque type de variable à bon escient, créant des solutions robustes et maintenables.

Dans le chapitre suivant, nous explorerons les substitutions, expansions et expressions, découvrant comment Bash transforme les variables en véritables moteurs de traitement de données.
