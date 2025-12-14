# Chapitre 79 - Debugging avancé

> "Le debugging est comme être un détective dans un crime où vous êtes aussi le criminel." - Filipe Fortes

## Introduction : L'art du debugging dans le monde des scripts

Imaginez-vous en tant que développeur de logiciels, face à un script complexe qui refuse obstinément de fonctionner comme prévu. C'est comme essayer de réparer une voiture de course avec seulement une lampe de poche et un tournevis, dans le noir complet. Le debugging avancé en Bash est votre boîte à outils complète : des projecteurs puissants, des diagnostics électroniques sophistiqués, et même des outils d'analyse post-mortem.

Dans ce chapitre, nous allons explorer les techniques de debugging les plus avancées pour Bash : des options de traçage détaillées aux analyses de performance, en passant par des stratégies de logging intelligentes et des méthodes d'inspection post-mortem.

## Section 1 : Les options de traçage de base (set -x, -v)

### 1.1 L'option -x : L'exécution pas à pas

L'option `set -x` (ou `set -o xtrace`) est votre microscope pour observer l'exécution ligne par ligne. Elle affiche chaque commande avant son exécution, avec les expansions réalisées.

```bash
#!/bin/bash

set -x  # Active le mode traçage

nom="Alice"
age=28

echo "Bonjour, je m'appelle $nom et j'ai $age ans"
# + echo 'Bonjour, je m'appelle Alice et j'ai 28 ans'

for i in {1..3}; do
    echo "Itération $i"
done
# + for i in '{1..3}'
# + echo 'Itération 1'
# + echo 'Itération 2'
# + echo 'Itération 3'
```

**Analogies pédagogiques :**
- C'est comme regarder un film au ralenti pour comprendre chaque mouvement
- Comparable à un professeur qui explique étape par étape une démonstration mathématique

### 1.2 L'option -v : Le script brut

L'option `set -v` (ou `set -o verbose`) affiche les lignes du script exactement comme elles sont lues, avant toute expansion.

```bash
#!/bin/bash

set -v  # Mode verbose

nom="Alice"
echo "Salut $nom"
# nom="Alice"
# echo "Salut $nom"
# Salut Alice
```

**Différences clés :**
- `-v` : Affiche le code source brut
- `-x` : Affiche les commandes après expansion
- Combinaison : `set -xv` pour voir les deux

### 1.3 Gestion fine du traçage

Vous pouvez activer/désactiver le traçage de manière sélective :

```bash
#!/bin/bash

echo "Cette ligne s'exécute normalement"

set -x  # Activation du traçage
resultat=$(ls -la /tmp | wc -l)
echo "Nombre de fichiers : $resultat"
set +x  # Désactivation du traçage

echo "Retour au mode normal"
```

**Astuce professionnelle :** Utilisez des blocs de traçage pour isoler les sections problématiques sans noyer votre sortie.

## Section 2 : La variable PS4 et la personnalisation du traçage

### 2.1 Comprendre PS4

La variable `PS4` contrôle le prompt affiché par `set -x`. Par défaut, elle vaut `+ `, mais vous pouvez la personnaliser pour des informations plus riches.

```bash
#!/bin/bash

# Format personnalisé : [SCRIPT:LIGNE:FONCTION] COMMANDE
PS4='+ [${BASH_SOURCE:-${0##*/}}:$LINENO:${FUNCNAME[0]:-main}] '

set -x

fonction_test() {
    local var="test"
    echo "Dans la fonction : $var"
}

fonction_test
# + [script_debug.sh:12:main] fonction_test
# + [script_debug.sh:5:fonction_test] local var=test
# + [script_debug.sh:6:fonction_test] echo 'Dans la fonction : test'
```

### 2.2 PS4 avancée avec timestamps et PID

```bash
#!/bin/bash

# Format ultra-détaillé avec timestamp et PID
PS4='+ $(date "+%H:%M:%S") [${BASH_SOURCE:-${0##*/}}:$LINENO:${FUNCNAME[0]:-main}][$$] '

set -x

echo "Début du script"
sleep 1
echo "Fin du script"
```

**Résultat :**
```
+ 14:23:45 [script.sh:8:main][12345] echo 'Début du script'
Début du script
+ 14:23:46 [script.sh:10:main][12345] echo 'Fin du script'
Fin du script
```

### 2.3 Gestion conditionnelle du traçage

Combinez PS4 avec des conditions pour un debugging intelligent :

```bash
#!/bin/bash

# Traçage seulement si DEBUG est défini
DEBUG="${DEBUG:-false}"
if [[ "$DEBUG" == "true" ]]; then
    PS4='+ [DEBUG:$LINENO] '
    set -x
fi

echo "Script en cours..."
# Si DEBUG=true, chaque ligne sera tracée
```

## Section 3 : Analyse post-mortem et gestion des core dumps

### 3.1 Les traps pour l'analyse post-mortem

Utilisez `trap` pour capturer l'état du script en cas d'erreur :

```bash
#!/bin/bash

# Fonction d'analyse post-mortem
post_mortem() {
    echo "=== ANALYSE POST-MORTEM ===" >&2
    echo "Script: $0" >&2
    echo "Ligne d'erreur: $1" >&2
    echo "Commande échouée: $2" >&2
    echo "PID: $$" >&2
    echo "Variables importantes:" >&2
    echo "  VAR1=$VAR1" >&2
    echo "  VAR2=$VAR2" >&2
    echo "Stack trace:" >&2
    local i=0
    while caller $i; do ((i++)); done >&2
}

# Installation du trap
trap 'post_mortem $LINENO "$BASH_COMMAND"' ERR

set -e  # Arrêt sur erreur

# Script qui va échouer
VAR1="valeur1"
VAR2="valeur2"

echo "Début du traitement"
commande_inexistante  # Cette commande va échouer
echo "Cette ligne ne sera jamais exécutée"
```

### 3.2 Stack trace personnalisé

Pour des analyses plus poussées, créez une fonction de stack trace :

```bash
#!/bin/bash

stack_trace() {
    echo "=== STACK TRACE ===" >&2
    local frame=0
    while caller $frame; do
        local line func file
        read line func file <<< "$(caller $frame)"
        echo "  Frame $frame: $file:$line in $func" >&2
        ((frame++))
    done
}

fonction_niveau3() {
    stack_trace
    exit 1
}

fonction_niveau2() {
    fonction_niveau3
}

fonction_niveau1() {
    fonction_niveau2
}

fonction_niveau1
```

### 3.3 Gestion des signaux pour le debugging

Interceptez différents signaux pour des analyses spécialisées :

```bash
#!/bin/bash

# Gestionnaire pour SIGUSR1 : dump des variables
dump_variables() {
    echo "=== DUMP DES VARIABLES ===" >&2
    declare -p | grep -E '^declare [^=]*=' >&2
}

# Gestionnaire pour SIGUSR2 : stack trace
dump_stack() {
    echo "=== STACK TRACE ===" >&2
    local frame=0
    while caller $frame; do ((frame++)); done >&2
}

# Installation des traps
trap dump_variables USR1
trap dump_stack USR2
trap 'echo "Script interrompu par l utilisateur" >&2; exit 1' INT

echo "Script en cours... Envoyez SIGUSR1 pour dump, SIGUSR2 pour stack"
echo "PID: $$"
sleep 60  # Laissez le temps d'envoyer les signaux
```

**Utilisation :**
```bash
# Dans un autre terminal
kill -USR1 12345  # Dump des variables
kill -USR2 12345  # Stack trace
```

## Section 4 : Profiling et analyse de performance

### 4.1 Mesure du temps d'exécution

Utilisez `time` pour mesurer les performances :

```bash
#!/bin/bash

# Fonction de profiling simple
profile_function() {
    local func_name="$1"
    local start_time=$(date +%s.%N)
    
    # Exécuter la fonction
    "$@"
    
    local end_time=$(date +%s.%N)
    local duration=$(echo "$end_time - $start_time" | bc)
    
    echo "[$func_name] Durée: ${duration}s" >&2
}

# Fonction à profiler
operation_lente() {
    for i in {1..1000}; do
        echo "Traitement $i" > /dev/null
    done
}

profile_function operation_lente
```

### 4.2 Profiling avec PS4 et timestamps

Intégrez le profiling directement dans le traçage :

```bash
#!/bin/bash

# PS4 avec timestamp pour profiling
PS4='+ $(date +%s.%N) [$LINENO] '

exec 3>&2  # Sauvegarde stderr
exec 2> /tmp/profile.log  # Redirection du traçage

set -x

fonction_a_profiler() {
    echo "Début traitement"
    sleep 0.1
    echo "Milieu traitement"
    sleep 0.1
    echo "Fin traitement"
}

fonction_a_profiler

set +x
exec 2>&3  # Restauration stderr

echo "=== ANALYSE DU PROFIL ==="
awk '{
    split($1, time, "+")
    if (NR == 1) { prev_time = time[2] }
    else {
        duration = time[2] - prev_time
        print "Durée: " duration "s - " $0
        prev_time = time[2]
    }
}' /tmp/profile.log
```

### 4.3 Outil de profiling intégré

Créez un système de profiling complet :

```bash
#!/bin/bash

declare -A PROFILE_START
declare -A PROFILE_COUNT
declare -A PROFILE_TOTAL

profile_start() {
    local label="$1"
    PROFILE_START["$label"]=$(date +%s.%N)
    ((PROFILE_COUNT["$label"]++))
}

profile_end() {
    local label="$1"
    local end_time=$(date +%s.%N)
    local start_time="${PROFILE_START["$label"]}"
    
    if [[ -n "$start_time" ]]; then
        local duration=$(echo "$end_time - $start_time" | bc)
        PROFILE_TOTAL["$label"]=$(echo "${PROFILE_TOTAL["$label"]:0} + $duration" | bc)
        unset PROFILE_START["$label"]
    fi
}

profile_report() {
    echo "=== RAPPORT DE PROFILING ==="
    printf "%-20s %-8s %-12s %-12s\n" "Fonction" "Appels" "Total(s)" "Moyen(s)"
    printf "%-20s %-8s %-12s %-12s\n" "--------" "------" "--------" "--------"
    
    for label in "${!PROFILE_TOTAL[@]}"; do
        local total="${PROFILE_TOTAL["$label"]}"
        local count="${PROFILE_COUNT["$label"]}"
        local avg=$(echo "scale=4; $total / $count" | bc)
        printf "%-20s %-8s %-12s %-12s\n" "$label" "$count" "$total" "$avg"
    done
}

# Utilisation
operation1() {
    profile_start "operation1"
    sleep 0.1
    profile_end "operation1"
}

operation2() {
    profile_start "operation2"
    sleep 0.05
    operation1  # Imbrication possible
    profile_end "operation2"
}

# Exécution
for i in {1..3}; do operation1; done
for i in {1..2}; do operation2; done

profile_report
```

## Section 5 : Command aliasing pour le logging avancé

### 5.1 Surcharge de commandes avec des alias

Créez des alias pour tracer automatiquement certaines commandes :

```bash
#!/bin/bash

# Alias pour tracer les commandes critiques
alias rm='rm -v'  # rm verbeux
alias cp='cp -v'  # cp verbeux
alias mv='mv -v'  # mv verbeux

# Alias avec logging personnalisé
log_command() {
    local cmd="$1"
    shift
    echo "[$(date)] Exécution: $cmd $*" >&2
    "$cmd" "$@"
    local status=$?
    echo "[$(date)] Statut: $status" >&2
    return $status
}

alias rm='log_command rm'
alias cp='log_command cp'

echo "Test des alias avec logging"
touch test1.txt test2.txt
cp test1.txt test3.txt
rm test1.txt
```

### 5.2 Surcharge intelligente avec context

Alias qui s'adaptent au contexte :

```bash
#!/bin/bash

# Fonction de logging contextuel
log_with_context() {
    local cmd="$1"
    shift
    local context="${FUNCNAME[1]:-main}"
    local line="${BASH_LINENO[0]}"
    
    echo "[$(date '+%H:%M:%S')] [$context:$line] $cmd $*" >&2
    "$cmd" "$@"
    local status=$?
    echo "[$(date '+%H:%M:%S')] [$context:$line] Exit: $status" >&2
    return $status
}

# Installation des alias
alias curl='log_with_context curl'
alias wget='log_with_context wget'

# Fonction qui utilise les commandes aliasées
telecharger_fichier() {
    local url="$1"
    local dest="$2"
    
    echo "Téléchargement de $url vers $dest"
    curl -s "$url" -o "$dest"
}

telecharger_fichier "https://example.com/file.txt" "/tmp/test.txt"
```

### 5.3 Logging conditionnel basé sur des patterns

Alias qui se déclenchent seulement pour certains patterns :

```bash
#!/bin/bash

# Logging seulement pour les commandes dangereuses
dangerous_commands="rm|mv|cp|chmod|chown"

log_if_dangerous() {
    local cmd="$1"
    
    if [[ "$cmd" =~ ^($dangerous_commands) ]]; then
        echo "[DANGER] $(date): $cmd $*" >&2
    fi
    
    "$cmd" "$@"
}

# Surcharge des commandes dangereuses
for cmd in rm mv cp chmod chown; do
    alias "$cmd"="log_if_dangerous $cmd"
done

echo "Commandes normales (pas de log)"
ls -la /tmp

echo "Commandes dangereuses (avec log)"
rm -f /tmp/test.txt 2>/dev/null || true
chmod 755 /tmp/test.sh 2>/dev/null || true
```

## Section 6 : Techniques avancées de debugging

### 6.1 Debugging des sous-shells

Les sous-shells peuvent être difficiles à debugger. Utilisez des techniques spéciales :

```bash
#!/bin/bash

set -x

# Debugging d'un sous-shell
(
    echo "Début sous-shell"
    var="dans sous-shell"
    echo "Var: $var"
    echo "PID sous-shell: $$"
) &
wait

echo "Retour dans shell parent: $$"
```

### 6.2 Inspection des descripteurs de fichiers

Vérifiez l'état des descripteurs de fichiers pendant le debugging :

```bash
#!/bin/bash

inspect_fd() {
    echo "=== DESCRIPTEURS DE FICHIERS ==="
    for fd in {0..10}; do
        if [[ -e "/proc/$$/fd/$fd" ]]; then
            local target=$(readlink "/proc/$$/fd/$fd")
            echo "FD $fd -> $target"
        fi
    done
}

# Utilisation
exec 3> /tmp/debug.log
exec 4< /tmp/input.txt

inspect_fd

echo "Test écriture FD 3" >&3
echo "Test lecture FD 4:"
read line <&4
echo "Lu: $line"

inspect_fd
```

### 6.3 Debugging des signaux et interruptions

Analysez comment votre script réagit aux interruptions :

```bash
#!/bin/bash

signal_debug() {
    local signal="$1"
    echo "[DEBUG] Signal reçu: $signal (ligne $2, commande: $3)" >&2
    
    # Dump de l'état
    echo "[DEBUG] Variables locales:" >&2
    local var
    for var in "${!@}"; do
        echo "[DEBUG]   $var=${!var}" >&2
    done
}

trap 'signal_debug INT $LINENO "$BASH_COMMAND"' INT
trap 'signal_debug TERM $LINENO "$BASH_COMMAND"' TERM

echo "Script en cours... Appuyez sur Ctrl+C pour debugger"
sleep 10
```

### 6.4 Analyse de la mémoire et des variables

Inspectez l'utilisation mémoire et l'état des variables :

```bash
#!/bin/bash

memory_snapshot() {
    echo "=== SNAPSHOT MÉMOIRE ==="
    
    # Variables scalaires
    echo "Variables scalaires:"
    declare -p | grep -E '^declare -- [^=]*=' | head -5
    
    # Arrays
    echo "Arrays:"
    declare -p | grep -E '^declare -a ' | head -3
    
    # Associative arrays
    echo "Arrays associatifs:"
    declare -p | grep -E '^declare -A ' | head -3
    
    # Mémoire système (si disponible)
    if command -v free >/dev/null 2>&1; then
        echo "Mémoire système:"
        free -h
    fi
}

# Test
declare -a tableau=(1 2 3 4 5)
declare -A assoc=(["cle1"]="val1" ["cle2"]="val2")

memory_snapshot
```

## Section 7 : Outils et frameworks de debugging

### 7.1 Framework de debugging modulaire

Créez un framework réutilisable pour vos scripts :

```bash
#!/bin/bash

# Framework de debugging modulaire
DEBUG_LEVEL="${DEBUG_LEVEL:-0}"

debug() {
    local level="$1"
    shift
    
    if (( DEBUG_LEVEL >= level )); then
        echo "[DEBUG:$level] $*" >&2
    fi
}

trace_cmd() {
    local level="$1"
    local cmd="$2"
    shift 2
    
    debug "$level" "Exécution: $cmd $*"
    if (( DEBUG_LEVEL >= level )); then
        set -x
        "$cmd" "$@"
        local status=$?
        set +x
        debug "$level" "Statut: $status"
        return $status
    else
        "$cmd" "$@"
    fi
}

profile_block() {
    local label="$1"
    local level="$2"
    shift 2
    
    local start=$(date +%s.%N)
    debug "$level" "Début bloc: $label"
    
    "$@"
    
    local end=$(date +%s.%N)
    local duration=$(echo "$end - $start" | bc)
    debug "$level" "Fin bloc: $label (durée: ${duration}s)"
}

# Configuration
DEBUG_LEVEL=2  # Niveau de debug

# Utilisation
debug 1 "Message niveau 1"
debug 2 "Message niveau 2"
debug 3 "Message niveau 3 (non affiché)"

trace_cmd 2 echo "Test trace_cmd"

profile_block "test_profiling" 1 sleep 0.1
```

### 7.2 Intégration avec des outils externes

Utilisez des outils externes pour un debugging avancé :

```bash
#!/bin/bash

# Debugging avec strace (si disponible)
debug_with_strace() {
    local cmd="$1"
    shift
    
    if command -v strace >/dev/null 2>&1; then
        echo "[DEBUG] Tracing avec strace: $cmd $*" >&2
        strace -e trace="$cmd" "$cmd" "$@" 2>&1 | 
            grep -E "(write|read|open|close)" >&2
    else
        echo "[DEBUG] strace non disponible, exécution normale" >&2
        "$cmd" "$@"
    fi
}

# Debugging avec ltrace pour les appels de bibliothèque
debug_with_ltrace() {
    local cmd="$1"
    shift
    
    if command -v ltrace >/dev/null 2>&1; then
        echo "[DEBUG] Tracing des bibliothèques: $cmd $*" >&2
        ltrace -e 'printf,puts' "$cmd" "$@" 2>&1 | head -10 >&2
    else
        echo "[DEBUG] ltrace non disponible, exécution normale" >&2
        "$cmd" "$@"
    fi
}

# Test
debug_with_strace ls -la /tmp | head -5
debug_with_ltrace echo "Test ltrace"
```

## Section 8 : Bonnes pratiques et stratégies

### 8.1 Niveaux de debugging progressifs

Implémentez différents niveaux de verbosité :

```bash
#!/bin/bash

# Niveaux de debug
# 0: Aucun debug
# 1: Erreurs seulement
# 2: Warnings + erreurs
# 3: Info + warnings + erreurs
# 4: Debug + tout
# 5: Trace complet

DEBUG_LEVEL="${DEBUG_LEVEL:-0}"

log_error() { (( DEBUG_LEVEL >= 1 )) && echo "[ERROR] $*" >&2; }
log_warn()  { (( DEBUG_LEVEL >= 2 )) && echo "[WARN]  $*" >&2; }
log_info()  { (( DEBUG_LEVEL >= 3 )) && echo "[INFO]  $*" >&2; }
log_debug() { (( DEBUG_LEVEL >= 4 )) && echo "[DEBUG] $*" >&2; }
log_trace() { (( DEBUG_LEVEL >= 5 )) && echo "[TRACE] $*" >&2; }

# Utilisation
log_error "Une erreur critique"
log_warn "Un avertissement"
log_info "Information générale"
log_debug "Détail de debug"
log_trace "Trace très détaillée"
```

### 8.2 Debugging en production

Techniques sûres pour le debugging en environnement de production :

```bash
#!/bin/bash

# Debug conditionnel basé sur des variables d'environnement
PRODUCTION="${PRODUCTION:-true}"
DEBUG_FILE="${DEBUG_FILE:-/tmp/script_debug.log}"

safe_debug() {
    local level="$1"
    shift
    
    # En production, seulement les erreurs critiques
    if [[ "$PRODUCTION" == "true" ]]; then
        if (( level <= 1 )); then
            echo "[$(date)] [ERROR] $*" >> "$DEBUG_FILE"
        fi
    else
        # En développement, tout afficher
        echo "[DEBUG:$level] $*" >&2
    fi
}

# Rotation automatique des logs de debug
rotate_debug_log() {
    local max_size=1048576  # 1MB
    
    if [[ -f "$DEBUG_FILE" ]] && (( $(stat -f%z "$DEBUG_FILE" 2>/dev/null || stat -c%s "$DEBUG_FILE") > max_size )); then
        mv "$DEBUG_FILE" "${DEBUG_FILE}.old"
        echo "[$(date)] Rotation du log de debug" >> "${DEBUG_FILE}.old"
    fi
}

# Utilisation
rotate_debug_log

safe_debug 1 "Erreur en production"
safe_debug 2 "Info de debug (pas en prod)"
```

### 8.3 Tests unitaires pour les fonctions de debug

Assurez-vous que vos fonctions de debugging fonctionnent correctement :

```bash
#!/bin/bash

# Tests unitaires pour les fonctions de debug
test_debug_functions() {
    local test_output="/tmp/debug_test.log"
    
    # Test log_error
    log_error "Test error" > "$test_output" 2>&1
    if grep -q "\[ERROR\] Test error" "$test_output"; then
        echo "✓ log_error fonctionne"
    else
        echo "✗ log_error échoue"
    fi
    
    # Test log_debug avec DEBUG_LEVEL=4
    DEBUG_LEVEL=4
    log_debug "Test debug" > "$test_output" 2>&1
    if grep -q "\[DEBUG\] Test debug" "$test_output"; then
        echo "✓ log_debug fonctionne"
    else
        echo "✗ log_debug échoue"
    fi
    
    rm -f "$test_output"
}

# Exécution des tests si demandé
if [[ "${1:-}" == "--test" ]]; then
    test_debug_functions
fi
```

## Conclusion : Maîtriser l'art du debugging

Le debugging avancé en Bash est bien plus qu'une compétence technique : c'est une philosophie de développement. Comme un médecin légiste qui dissèque un mystère, vous devez développer un œil clinique pour identifier les symptômes, poser les bons diagnostics, et appliquer les traitements appropriés.

**Points clés à retenir :**

1. **Les options `set -x` et `set -v`** sont vos premiers alliés pour la visibilité
2. **PS4 personnalisée** transforme le traçage en outil de profiling
3. **Les traps et analyses post-mortem** capturent l'état au moment critique
4. **Le profiling intégré** révèle les goulots d'étranglement
5. **Les alias de logging** automatisent la surveillance des commandes sensibles
6. **Les frameworks modulaires** rendent le debugging réutilisable

Dans le chapitre suivant, nous explorerons les tests et la validation, pour nous assurer que nos scripts ne se contentent pas de fonctionner, mais qu'ils fonctionnent correctement dans tous les scénarios imaginables.

---

**Exercice pratique :** Créez un script de sauvegarde avec debugging intégré qui :
- Trace toutes les opérations de copie
- Profile les temps d'exécution
- Gère les erreurs avec analyse post-mortem
- Fournit différents niveaux de verbosité
- Inclut des tests unitaires pour les fonctions critiques

**Réflexion :** Comment adapteriez-vous ces techniques pour debugger des scripts exécutés sur des serveurs distants via SSH ?
