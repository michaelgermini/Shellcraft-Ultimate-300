# Chapitre 82 - Métaprogrammation et génération de code

> "Le code qui s'écrit lui-même est comme un serpent qui se mord la queue : élégant, puissant, et légèrement effrayant." - Proverbe de programmeur

## Introduction : Quand le code devient son propre créateur

Imaginez un monde où vos scripts Bash s'adaptent automatiquement à leur environnement, génèrent leurs propres fonctions selon les besoins, et s'optimisent eux-mêmes. C'est le domaine de la métaprogrammation : le code qui pense, s'analyse, et se modifie lui-même.

Dans ce chapitre, nous allons explorer les techniques les plus avancées de Bash pour créer du code dynamique : génération de fonctions à la volée, introspection du code en cours d'exécution, évaluation sécurisée d'expressions, et même des systèmes capables de s'étendre eux-mêmes. Vous apprendrez à transformer vos scripts statiques en programmes vivants et adaptatifs.

## Section 1 : Les bases de l'évaluation dynamique (eval)

### 1.1 Comprendre eval : l'évaluation à runtime

`eval` permet d'exécuter du code Bash généré dynamiquement sous forme de chaîne de caractères.

**Analogies pédagogiques :**
- Comme un cuisinier qui écrit sa propre recette pendant qu'il cuisine
- Comme un musicien qui compose sa mélodie en la jouant

```bash
#!/bin/bash

# Les bases d'eval
echo "=== Les bases d'eval ==="

# Exemple simple : construction dynamique de commande
operation="echo"
message="Hello World"
command="$operation '$message'"

echo "Commande construite: $command"
eval "$command"

# Génération de variables dynamiques
echo
echo "=== Génération de variables dynamiques ==="

for i in {1..3}; do
    var_name="variable_$i"
    eval "$var_name='valeur $i'"
done

# Affichage des variables créées
echo "Variables créées:"
for i in {1..3}; do
    var_name="variable_$i"
    eval "echo '$var_name = '\$$var_name"
done

# Exécution conditionnelle
echo
echo "=== Exécution conditionnelle ==="

conditions=("true" "false" "(( 1 + 1 == 2 ))" "[[ -f /etc/passwd ]]")

for condition in "${conditions[@]}"; do
    echo "Test de: $condition"
    if eval "$condition"; then
        echo "  → Vrai"
    else
        echo "  → Faux"
    fi
done
```

### 1.2 Pièges et sécurisation d'eval

`eval` est puissant mais dangereux. Apprenez à l'utiliser en sécurité :

```bash
#!/bin/bash

# Sécurisation d'eval
echo "=== Sécurisation d'eval ==="

# Fonction d'évaluation sécurisée
safe_eval() {
    local code="$1"
    
    # Liste blanche des commandes autorisées
    local allowed_commands="echo printf date seq"
    local allowed_patterns="^[a-zA-Z_][a-zA-Z0-9_]*=|^echo |^printf |^date |^seq "
    
    # Vérifications de sécurité
    if [[ -z "$code" ]]; then
        echo "Erreur: code vide" >&2
        return 1
    fi
    
    # Vérification des patterns dangereux
    if [[ "$code" =~ [\;\&\`\$\(\)\<\>\|] ]]; then
        echo "Erreur: caractères dangereux détectés" >&2
        return 1
    fi
    
    # Vérification de la liste blanche
    local first_word=$(echo "$code" | awk '{print $1}')
    if [[ ! " $allowed_commands " =~ " $first_word " ]]; then
        echo "Erreur: commande non autorisée: $first_word" >&2
        return 1
    fi
    
    echo "Exécution sécurisée: $code"
    eval "$code"
}

# Tests de sécurité
echo "--- Tests de sécurité ---"

# Code sûr
safe_eval "echo 'Hello Safe World'"
safe_eval "date +%Y-%m-%d"
safe_eval "seq 1 3"

echo
echo "--- Code dangereux (rejeté) ---"

# Code dangereux (sera rejeté)
safe_eval "rm -rf /"  # Commande dangereuse
safe_eval "echo 'test'; rm file"  # Injection
safe_eval "eval echo exploit"  # Auto-référence dangereuse
```

## Section 2 : Génération dynamique de fonctions

### 2.1 Création de fonctions à la volée

Bash permet de créer des fonctions dont le code est généré dynamiquement :

```bash
#!/bin/bash

# Génération dynamique de fonctions
echo "=== Génération dynamique de fonctions ==="

# Générateur de fonctions getter/setter
generate_accessor() {
    local var_name="$1"
    local var_type="${2:-string}"  # string, int, bool
    
    # Fonction getter
    eval "
${var_name}_get() {
    echo \"\$$var_name\"
}
"
    
    # Fonction setter avec validation
    case "$var_type" in
        int)
            validation='[[ "$1" =~ ^[0-9]+$ ]] || { echo "Erreur: valeur entière attendue" >&2; return 1; }'
            ;;
        bool)
            validation='[[ "$1" =~ ^(true|false)$ ]] || { echo "Erreur: true/false attendu" >&2; return 1; }'
            ;;
        string)
            validation='true'  # Pas de validation spéciale
            ;;
    esac
    
    eval "
${var_name}_set() {
    $validation
    $var_name=\"\$1\"
}
"
}

# Génération des accesseurs
generate_accessor "username" "string"
generate_accessor "age" "int"
generate_accessor "is_admin" "bool"

# Test des fonctions générées
echo "--- Test des accesseurs générés ---"

username_set "alice"
echo "Username: $(username_get)"

age_set "25"
echo "Age: $(age_get)"

is_admin_set "true"
echo "Admin: $(is_admin_get)"

# Test des validations
echo
echo "--- Test des validations ---"
age_set "not_a_number"  # Devrait échouer
is_admin_set "maybe"     # Devrait échouer
```

### 2.2 Factory de fonctions avec paramètres

Création de familles de fonctions paramétrées :

```bash
#!/bin/bash

# Factory de fonctions mathématiques
echo "=== Factory de fonctions mathématiques ==="

create_math_function() {
    local func_name="$1"
    local operation="$2"
    local description="$3"
    
    # Génération de la fonction
    eval "
$func_name() {
    local result
    
    # Validation des paramètres
    for arg in \"\$@\"; do
        if ! [[ \"\$arg\" =~ ^-?[0-9]*\.?[0-9]+$ ]]; then
            echo \"Erreur: paramètre non numérique: \$arg\" >&2
            return 1
        fi
    done
    
    case '$operation' in
        sum)
            result=0
            for arg in \"\$@\"; do
                result=\$(echo \"\$result + \$arg\" | bc -l)
            done
            ;;
        product)
            result=1
            for arg in \"\$@\"; do
                result=\$(echo \"\$result * \$arg\" | bc -l)
            done
            ;;
        average)
            local sum=0 count=0
            for arg in \"\$@\"; do
                sum=\$(echo \"\$sum + \$arg\" | bc -l)
                ((count++))
            done
            result=\$(echo \"scale=2; \$sum / \$count\" | bc -l)
            ;;
        max)
            result=\"\$1\"
            for arg in \"\$@\"; do
                if (( \$(echo \"\$arg > \$result\" | bc -l) )); then
                    result=\"\$arg\"
                fi
            done
            ;;
        min)
            result=\"\$1\"
            for arg in \"\$@\"; do
                if (( \$(echo \"\$arg < \$result\" | bc -l) )); then
                    result=\"\$arg\"
                fi
            done
            ;;
    esac
    
    echo \"\$result\"
}

# Fonction d'aide
${func_name}_help() {
    echo \"$description\"
    echo \"Usage: $func_name <nombre1> [nombre2] ...\"
}
"
}

# Création des fonctions mathématiques
create_math_function "calculate_sum" "sum" "Calcule la somme des nombres"
create_math_function "calculate_product" "product" "Calcule le produit des nombres"
create_math_function "calculate_average" "average" "Calcule la moyenne des nombres"
create_math_function "find_max" "max" "Trouve le maximum des nombres"
create_math_function "find_min" "min" "Trouve le minimum des nombres"

# Tests
echo "--- Tests des fonctions générées ---"

echo "Somme 1 2 3 4 5: $(calculate_sum 1 2 3 4 5)"
echo "Produit 2 3 4: $(calculate_product 2 3 4)"
echo "Moyenne 10 20 30: $(calculate_average 10 20 30)"
echo "Maximum 5 2 8 1: $(find_max 5 2 8 1)"
echo "Minimum 5 2 8 1: $(find_min 5 2 8 1)"

echo
echo "--- Aide ---"
calculate_sum_help
```

### 2.3 Fonctions auto-modifiantes

Des fonctions capables de s'étendre elles-mêmes :

```bash
#!/bin/bash

# Fonctions auto-modifiantes
echo "=== Fonctions auto-modifiantes ==="

# Cache intelligent pour fonctions coûteuses
create_cached_function() {
    local func_name="$1"
    local actual_func="$2"
    
    # Génération de la fonction avec cache
    eval "
$func_name() {
    # Clé de cache basée sur les arguments
    local cache_key=\"\$(echo \"\$*\" | md5sum | cut -d' ' -f1)\"
    local cache_file=\"/tmp/cache_${func_name}_\${cache_key}\"
    
    # Vérification du cache
    if [[ -f \"\$cache_file\" ]]; then
        cat \"\$cache_file\"
        return 0
    fi
    
    # Exécution réelle et mise en cache
    $actual_func \"\$@\" | tee \"\$cache_file\"
}

# Méthode pour vider le cache
${func_name}_clear_cache() {
    rm -f /tmp/cache_${func_name}_*
    echo \"Cache vidé pour $func_name\"
}

# Méthode pour afficher les statistiques du cache
${func_name}_cache_stats() {
    local cache_files=\$(ls /tmp/cache_${func_name}_* 2>/dev/null | wc -l)
    local cache_size=\$(du -ch /tmp/cache_${func_name}_* 2>/dev/null | tail -1 | cut -f1)
    echo \"Fonction: $func_name\"
    echo \"Entrées en cache: \$cache_files\"
    echo \"Taille du cache: \${cache_size:-0}\"
}
"
}

# Fonction coûteuse simulée
slow_computation() {
    local n="$1"
    echo "Calcul coûteux pour n=$n..." >&2
    sleep 1  # Simulation du coût
    echo $((n * n))
}

# Création de la fonction avec cache
create_cached_function "fast_computation" "slow_computation"

# Tests
echo "--- Tests du cache ---"

echo "Premier appel (calcul):"
result1=$(fast_computation 5)

echo "Deuxième appel (cache):"
result2=$(fast_computation 5)

echo "Troisième appel (cache):"
result3=$(fast_computation 5)

echo "Appel avec autre paramètre:"
result4=$(fast_computation 10)

echo
echo "--- Statistiques ---"
fast_computation_cache_stats

echo
echo "--- Vidage du cache ---"
fast_computation_clear_cache
fast_computation_cache_stats
```

## Section 3 : Introspection et analyse du code

### 3.1 Inspection des fonctions et variables

Bash offre des outils pour inspecter le code en cours d'exécution :

```bash
#!/bin/bash

# Introspection du code
echo "=== Introspection du code ==="

# Fonction d'inspection des fonctions
inspect_function() {
    local func_name="$1"
    
    echo "=== Inspection de la fonction: $func_name ==="
    
    # Vérification de l'existence
    if ! declare -f "$func_name" >/dev/null; then
        echo "Fonction inexistante: $func_name"
        return 1
    fi
    
    # Informations générales
    echo "Nom: $func_name"
    
    # Code source
    echo "Code source:"
    declare -f "$func_name" | sed '1d;2d' | sed 's/^/  /'
    
    # Analyse du code
    local line_count=$(declare -f "$func_name" | wc -l)
    local char_count=$(declare -f "$func_name" | wc -c)
    
    echo "Statistiques:"
    echo "  Lignes: $line_count"
    echo "  Caractères: $char_count"
}

# Fonction d'inspection des variables
inspect_variable() {
    local var_name="$1"
    
    echo "=== Inspection de la variable: $var_name ==="
    
    # Vérification de l'existence
    if ! declare -p "$var_name" >/dev/null 2>&1; then
        echo "Variable inexistante: $var_name"
        return 1
    fi
    
    # Type et valeur
    local var_info=$(declare -p "$var_name")
    echo "Déclaration: $var_info"
    
    # Analyse détaillée
    if [[ "$var_info" =~ declare\ -a ]]; then
        echo "Type: Array indexé"
        eval "echo \"Taille: \${#$var_name[@]}\"" 
        eval "echo \"Contenu: \${$var_name[*]}\"" 
    elif [[ "$var_info" =~ declare\ -A ]]; then
        echo "Type: Array associatif"
        eval "echo \"Taille: \${#$var_name[@]}\"" 
        eval "echo \"Clés: \${!$var_name[@]}\"" 
    elif [[ "$var_info" =~ declare\ -i ]]; then
        echo "Type: Entier"
        eval "echo \"Valeur: \${$var_name}\"" 
    else
        echo "Type: Chaîne"
        eval "echo \"Longueur: \${#$var_name}\"" 
        eval "echo \"Valeur: '\${$var_name}'\"" 
    fi
}

# Fonctions de test
test_function() {
    local param="$1"
    echo "Ceci est une fonction de test"
    echo "Paramètre: $param"
    return 0
}

# Variables de test
string_var="Hello World"
int_var=42
array_var=(1 2 3 4 5)
declare -A assoc_array=(["cle1"]="valeur1" ["cle2"]="valeur2")

# Inspections
inspect_function "test_function"
echo
inspect_variable "string_var"
echo
inspect_variable "int_var"
echo
inspect_variable "array_var"
echo
inspect_variable "assoc_array"
```

### 3.2 Analyse dynamique du code source

Inspection et modification du code source lui-même :

```bash
#!/bin/bash

# Analyse dynamique du code source
echo "=== Analyse dynamique du code source ==="

# Extracteur de dépendances
analyze_dependencies() {
    local script_file="$1"
    
    echo "=== Analyse des dépendances: $script_file ==="
    
    if [[ ! -f "$script_file" ]]; then
        echo "Fichier inexistant: $script_file"
        return 1
    fi
    
    # Commandes externes utilisées
    echo "Commandes externes:"
    grep -E '^[^#]*\b(awk|sed|grep|curl|wget|ssh|scp|rsync|tar|gzip|mysql|psql)\b' "$script_file" | 
        sed 's/^/  /' || true
    
    # Fonctions définies
    echo "Fonctions définies:"
    grep -E '^[^#]*[a-zA-Z_][a-zA-Z0-9_]*\(\)' "$script_file" | 
        sed 's/().*//' | sed 's/^/  /' || true
    
    # Variables globales
    echo "Variables globales potentielles:"
    grep -E '^[^#]*[A-Z_][A-Z0-9_]*=' "$script_file" | 
        cut -d= -f1 | sed 's/^/  /' || true
    
    # Includes/fichiers sourcés
    echo "Fichiers sourcés:"
    grep -E '\bsource\s|\.\s' "$script_file" | 
        sed 's/^/  /' || true
}

# Optimiseur de code simple
optimize_script() {
    local script_file="$1"
    local output_file="${2:-${script_file}.optimized}"
    
    echo "=== Optimisation du script: $script_file ==="
    
    # Lecture du script
    local content=$(cat "$script_file")
    
    # Optimisations simples
    # Suppression des commentaires vides
    content=$(echo "$content" | sed '/^[[:space:]]*#[[:space:]]*$/d')
    
    # Suppression des espaces en fin de ligne
    content=$(echo "$content" | sed 's/[[:space:]]*$//')
    
    # Regroupement des redirections simples
    content=$(echo "$content" | sed 's/> *\/dev\/null 2>&1/\&>\/dev\/null/g')
    
    # Écriture du script optimisé
    echo "$content" > "$output_file"
    
    local original_size=$(stat -f%z "$script_file" 2>/dev/null || stat -c%s "$script_file")
    local optimized_size=$(stat -f%z "$output_file" 2>/dev/null || stat -c%s "$output_file")
    
    echo "Script original: $original_size octets"
    echo "Script optimisé: $optimized_size octets"
    echo "Réduction: $((original_size - optimized_size)) octets"
    echo "Fichier optimisé: $output_file"
}

# Script de test
cat > /tmp/test_script.sh << 'EOF'
#!/bin/bash

# Script de test pour l'analyse

# Variables globales
DATABASE_HOST="localhost"
MAX_CONNECTIONS=100

# Fonction principale
main() {
    echo "Connexion à $DATABASE_HOST"
    process_data
}

# Fonction de traitement
process_data() {
    # Utilisation d'outils externes
    awk '{print $1}' /etc/passwd > /tmp/output.txt
    grep "root" /tmp/output.txt
    curl -s https://api.example.com/data > /dev/null 2>&1
}

# Appel principal
main
EOF

chmod +x /tmp/test_script.sh

# Analyses
analyze_dependencies "/tmp/test_script.sh"
echo
optimize_script "/tmp/test_script.sh"

# Nettoyage
rm -f /tmp/test_script.sh /tmp/output.txt
```

## Section 4 : Génération de code à partir de templates

### 4.1 Système de templates simple

Création d'un système de génération de code basé sur des templates :

```bash
#!/bin/bash

# Système de templates
echo "=== Système de templates ==="

# Fonction de rendu de template
render_template() {
    local template="$1"
    local output_file="$2"
    shift 2
    
    # Variables passées en arguments (clé=valeur)
    declare -A variables
    for arg in "$@"; do
        local key="${arg%%=*}"
        local value="${arg#*=}"
        variables["$key"]="$value"
    done
    
    # Rendu du template
    local rendered="$template"
    
    for key in "${!variables[@]}"; do
        # Remplacement avec gestion des caractères spéciaux
        rendered=$(echo "$rendered" | sed "s|{{$key}}|${variables[$key]}|g")
    done
    
    # Écriture du résultat
    echo "$rendered" > "$output_file"
    echo "Template rendu: $output_file"
}

# Template pour un script de sauvegarde
backup_template='#!/bin/bash

# Script de sauvegarde généré automatiquement
# Généré le: {{generation_date}}
# Pour: {{client_name}}

BACKUP_NAME="{{backup_name}}"
SOURCE_DIR="{{source_dir}}"
DEST_DIR="{{dest_dir}}"
COMPRESSION="{{compression}}"

log() {
    echo "[$(date +"%Y-%m-%d %H:%M:%S")] $*" >&2
}

main() {
    log "Début de la sauvegarde: $BACKUP_NAME"
    
    # Création du répertoire de destination
    mkdir -p "$DEST_DIR"
    
    # Compression selon le type demandé
    case "$COMPRESSION" in
        gzip)
            tar -czf "$DEST_DIR/$BACKUP_NAME.tar.gz" "$SOURCE_DIR"
            ;;
        bzip2)
            tar -cjf "$DEST_DIR/$BACKUP_NAME.tar.bz2" "$SOURCE_DIR"
            ;;
        xz)
            tar -cJf "$DEST_DIR/$BACKUP_NAME.tar.xz" "$SOURCE_DIR"
            ;;
        *)
            log "Type de compression inconnu: $COMPRESSION"
            exit 1
            ;;
    esac
    
    if [[ $? -eq 0 ]]; then
        log "Sauvegarde réussie: $DEST_DIR/$BACKUP_NAME"
    else
        log "Erreur lors de la sauvegarde"
        exit 1
    fi
}

main "$@"
'

# Génération de scripts personnalisés
echo "--- Génération script sauvegarde site web ---"
render_template "$backup_template" "/tmp/backup_website.sh" \
    "generation_date=$(date)" \
    "client_name=MonSiteWeb" \
    "backup_name=website_$(date +%Y%m%d)" \
    "source_dir=/var/www/html" \
    "dest_dir=/backup/website" \
    "compression=gzip"

echo "--- Génération script sauvegarde base de données ---"
render_template "$backup_template" "/tmp/backup_database.sh" \
    "generation_date=$(date)" \
    "client_name=MaBaseDeDonnées" \
    "backup_name=database_$(date +%Y%m%d)" \
    "source_dir=/var/lib/mysql" \
    "dest_dir=/backup/database" \
    "compression=xz"

# Test des scripts générés
echo "--- Test des scripts générés ---"
chmod +x /tmp/backup_website.sh /tmp/backup_database.sh

echo "Contenu du script website:"
head -10 /tmp/backup_website.sh

# Nettoyage
rm -f /tmp/backup_website.sh /tmp/backup_database.sh
```

### 4.2 Générateur de code orienté objet

Création automatique de classes avec méthodes :

```bash
#!/bin/bash

# Générateur de code orienté objet
echo "=== Générateur de code orienté objet ==="

# Template de classe Bash
class_template='{{class_name}}() {
    local self="$1"
    
{{properties}}
    
{{methods}}
}

# Constructeur
{{class_name}}_new() {
    local instance="{{class_name}}_instance_$RANDOM"
    {{class_name}} "$instance"
    echo "$instance"
}

# Méthodes statiques
{{static_methods}}
'

# Générateur de classe
generate_class() {
    local class_name="$1"
    shift
    
    # Parsing des arguments
    local properties=""
    local methods=""
    local static_methods=""
    
    while [[ $# -gt 0 ]]; do
        case "$1" in
            --property)
                local prop_name="$2"
                local prop_default="${3:-}"
                properties="${properties}    \$self.$prop_name=\"$prop_default\"\\n"
                shift 3
                ;;
            --method)
                local method_name="$2"
                local method_body="$3"
                methods="${methods}$method_name() {
    $method_body
}
"
                shift 3
                ;;
            --static-method)
                local static_name="$2"
                local static_body="$3"
                static_methods="${static_methods}${static_name}() {
    $static_body
}
"
                shift 3
                ;;
            *)
                shift
                ;;
        esac
    done
    
    # Génération de la classe
    render_template "$class_template" "/tmp/${class_name}.sh" \
        "class_name=$class_name" \
        "properties=$properties" \
        "methods=$methods" \
        "static_methods=$static_methods"
    
    # Sourcing de la classe générée
    source "/tmp/${class_name}.sh"
    echo "Classe générée: $class_name"
}

# Génération d'une classe CompteBancaire
generate_class "CompteBancaire" \
    --property "solde" "0" \
    --property "titulaire" "" \
    --property "numero_compte" "" \
    --method "deposer" '
        local montant="$1"
        if (( $(echo "$montant > 0" | bc -l) )); then
            $self.solde=$(echo "$self.solde + $montant" | bc -l)
            echo "Dépôt de $montant € effectué. Nouveau solde: $self.solde €"
        else
            echo "Erreur: montant invalide" >&2
            return 1
        fi' \
    --method "retirer" '
        local montant="$1"
        if (( $(echo "$self.solde >= $montant" | bc -l) )); then
            $self.solde=$(echo "$self.solde - $montant" | bc -l)
            echo "Retrait de $montant € effectué. Nouveau solde: $self.solde €"
        else
            echo "Erreur: solde insuffisant" >&2
            return 1
        fi' \
    --method "afficher_solde" '
        echo "Solde du compte $self.numero_compte: $self.solde €"'

# Test de la classe générée
echo "--- Test de la classe CompteBancaire ---"

# Création d'instances
compte1=$(CompteBancaire_new)
compte2=$(CompteBancaire_new)

# Configuration
$compte1.titulaire="Alice Dupont"
$compte1.numero_compte="FR1234567890"

$compte2.titulaire="Bob Martin"
$compte2.numero_compte="FR0987654321"

# Opérations
$compte1.deposer 1000
$compte1.retirer 150
$compte1.afficher_solde

$compte2.deposer 500
$compte2.afficher_solde

# Nettoyage
rm -f /tmp/CompteBancaire.sh
```

## Section 5 : Métaprogrammation sécurisée et bonnes pratiques

### 5.1 Sandboxing et sécurité

Techniques pour exécuter du code généré en sécurité :

```bash
#!/bin/bash

# Métaprogrammation sécurisée
echo "=== Métaprogrammation sécurisée ==="

# Fonction d'évaluation dans un sous-shell sandbox
safe_eval_sandbox() {
    local code="$1"
    local timeout="${2:-5}"
    
    # Création d'un environnement restreint
    (
        # Variables d'environnement limitées
        export PATH="/bin:/usr/bin"
        unset BASH_ENV
        
        # Fonctions dangereuses désactivées
        unset -f eval
        unset -f exec
        unset -f source
        
        # Timeout pour éviter les boucles infinies
        timeout "$timeout" bash -c "$code"
    )
}

# Test de sécurité
echo "--- Tests de sécurité ---"

# Code sûr
echo "Test code sûr:"
safe_eval_sandbox "echo 'Hello from sandbox'" 2

# Code potentiellement dangereux (timeout)
echo "Test timeout:"
safe_eval_sandbox "while true; do true; done" 2

# Code avec fonctions interdites
echo "Test fonctions interdites:"
safe_eval_sandbox "eval echo 'tentative d injection'" 2
```

### 5.2 Validation et tests des codes générés

Assurance qualité pour le code généré dynamiquement :

```bash
#!/bin/bash

# Validation des codes générés
echo "=== Validation des codes générés ==="

# Testeur de syntaxe
validate_bash_syntax() {
    local code="$1"
    local temp_file="/tmp/validate_bash_$$.sh"
    
    # Écriture du code dans un fichier temporaire
    echo "#!/bin/bash" > "$temp_file"
    echo "$code" >> "$temp_file"
    
    # Test de syntaxe
    if bash -n "$temp_file" 2>/dev/null; then
        echo "✓ Syntaxe valide"
        rm -f "$temp_file"
        return 0
    else
        echo "✗ Erreurs de syntaxe détectées"
        bash -n "$temp_file"  # Afficher les erreurs
        rm -f "$temp_file"
        return 1
    fi
}

# Analyseur de sécurité
security_audit() {
    local code="$1"
    
    local issues=0
    
    # Vérifications de sécurité
    if echo "$code" | grep -q "rm -rf /"; then
        echo "! ATTENTION: Suppression récursive détectée"
        ((issues++))
    fi
    
    if echo "$code" | grep -q "eval.*\$"; then
        echo "! ATTENTION: eval avec variable détecté"
        ((issues++))
    fi
    
    if echo "$code" | grep -q "exec.*bash"; then
        echo "! ATTENTION: exec bash détecté"
        ((issues++))
    fi
    
    if echo "$code" | grep -q "source.*\$"; then
        echo "! ATTENTION: source dynamique détecté"
        ((issues++))
    fi
    
    if (( issues == 0 )); then
        echo "✓ Aucun problème de sécurité détecté"
        return 0
    else
        echo "! $issues problème(s) de sécurité détecté(s)"
        return 1
    fi
}

# Testeur fonctionnel
functional_test() {
    local code="$1"
    local test_input="$2"
    local expected_output="$3"
    
    local temp_file="/tmp/functional_test_$$.sh"
    
    # Création du script de test
    cat > "$temp_file" << EOF
#!/bin/bash
$code
EOF
    chmod +x "$temp_file"
    
    # Exécution du test
    local actual_output
    if [[ -n "$test_input" ]]; then
        actual_output=$($temp_file <<< "$test_input" 2>/dev/null)
    else
        actual_output=$($temp_file 2>/dev/null)
    fi
    
    rm -f "$temp_file"
    
    if [[ "$actual_output" == "$expected_output" ]]; then
        echo "✓ Test fonctionnel réussi"
        return 0
    else
        echo "✗ Test fonctionnel échoué"
        echo "  Attendu: '$expected_output'"
        echo "  Obtenu:  '$actual_output'"
        return 1
    fi
}

# Suite de tests complète
test_generated_code() {
    local code="$1"
    local description="$2"
    
    echo "=== Test du code généré: $description ==="
    
    # Validation syntaxique
    if ! validate_bash_syntax "$code"; then
        return 1
    fi
    
    # Audit de sécurité
    if ! security_audit "$code"; then
        return 1
    fi
    
    # Test fonctionnel (si applicable)
    if [[ -n "$3" ]]; then
        functional_test "$code" "$3" "$4"
    fi
    
    echo "✓ Code validé avec succès"
}

# Tests
echo "--- Tests de validation ---"

# Code valide
test_generated_code "
echo 'Hello World'
exit 0
" "Echo simple"

# Code avec problème de sécurité
test_generated_code "
eval 'echo dangerous'
rm -rf /
" "Code dangereux"

# Code avec test fonctionnel
test_generated_code "
read input
echo \"Hello \$input\"
" "Code avec input" "Alice" "Hello Alice"
```

### 5.3 Performance et optimisation

Techniques pour optimiser le code généré dynamiquement :

```bash
#!/bin/bash

# Optimisation du code généré
echo "=== Optimisation du code généré ==="

# Optimiseur de code Bash
optimize_generated_code() {
    local code="$1"
    
    # Optimisations successives
    local optimized="$code"
    
    # 1. Suppression des espaces inutiles
    optimized=$(echo "$optimized" | sed 's/[[:space:]]*$//')
    
    # 2. Regroupement des redirections
    optimized=$(echo "$optimized" | sed 's|> *\/dev\/null 2>&1|\&>\/dev\/null|g')
    
    # 3. Optimisation des conditions
    optimized=$(echo "$optimized" | sed 's/\[\[ \(.*\) \]\]/\1/g')
    
    # 4. Suppression des echo inutiles
    optimized=$(echo "$optimized" | sed '/^echo ""$/d')
    
    # 5. Optimisation des pipes
    # Regrouper les commandes simples
    optimized=$(echo "$optimized" | sed 's/| *cat$/| /g')
    
    echo "$optimized"
}

# Profileur de code généré
profile_generated_code() {
    local code="$1"
    local iterations="${2:-10}"
    
    local temp_file="/tmp/profile_code_$$.sh"
    
    cat > "$temp_file" << EOF
#!/bin/bash
$code
EOF
    chmod +x "$temp_file"
    
    echo "=== Profiling du code généré ==="
    echo "Itérations: $iterations"
    
    # Mesure du temps
    local start_time=$(date +%s.%N)
    for ((i=1; i<=iterations; i++)); do
        $temp_file >/dev/null 2>&1
    done
    local end_time=$(date +%s.%N)
    
    local total_time=$(echo "$end_time - $start_time" | bc)
    local avg_time=$(echo "scale=4; $total_time / $iterations" | bc)
    
    echo "Temps total: ${total_time}s"
    echo "Temps moyen: ${avg_time}s"
    
    rm -f "$temp_file"
}

# Code de test
test_code='
#!/bin/bash
for i in {1..100}; do
    echo "test" > /dev/null 2>&1
done
'

echo "--- Code original ---"
echo "$test_code" | wc -l
echo "$test_code" | wc -c

echo "--- Code optimisé ---"
optimized_code=$(optimize_generated_code "$test_code")
echo "$optimized_code" | wc -l
echo "$optimized_code" | wc -c

echo "--- Profiling ---"
profile_generated_code "$test_code" 5
profile_generated_code "$optimized_code" 5
```

## Conclusion : La programmation au-delà du code

La métaprogrammation en Bash transforme vos scripts de simples automatisations en systèmes intelligents capables de s'adapter, s'optimiser, et même de s'étendre eux-mêmes. Comme un organisme vivant qui évolue pour survivre dans son environnement, vos programmes deviennent des entités dynamiques.

**Points clés à retenir :**

1. **`eval` avec précaution** : L'évaluation dynamique est puissante mais dangereuse - toujours valider et sandboxer
2. **Génération de fonctions** : Créez des familles de fonctions paramétrées pour éviter la duplication
3. **Introspection** : Inspectez et analysez votre code en cours d'exécution pour le déboguer et l'optimiser
4. **Templates de code** : Utilisez des systèmes de templates pour générer du code cohérent et maintenable
5. **Sécurité avant tout** : Validez, auditez, et testez rigoureusement tout code généré dynamiquement

Dans le chapitre suivant, nous explorerons les techniques avancées de performance et d'optimisation, pour que vos scripts Bash non seulement fonctionnent correctement, mais le fassent à la vitesse de l'éclair.

---

**Exercice pratique :** Créez un système de génération automatique de scripts de monitoring qui :
- Analyse un fichier de configuration
- Génère dynamiquement les fonctions de vérification appropriées
- Crée des alertes personnalisées selon les seuils définis
- S'auto-optimise en fonction des performances observées
- Inclut des tests automatiques pour valider le code généré

**Réflexion :** Comment pourriez-vous appliquer ces techniques de métaprogrammation pour créer un langage de domaine spécifique (DSL) en Bash pour automatiser des tâches complexes ?
