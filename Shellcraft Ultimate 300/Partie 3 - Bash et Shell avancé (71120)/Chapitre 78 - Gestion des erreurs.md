# Chapitre 78 - Gestion des erreurs

## Table des matières
- [Introduction](#introduction)
- [Types d'erreurs en Bash](#types-derreurs-en-bash)
- [Codes de retour et variables spéciales](#codes-de-retour-et-variables-spéciales)
- [Gestion d'erreurs avec set et trap](#gestion-derreurs-avec-set-et-trap)
- [Fonctions robustes et gestion d'erreurs](#fonctions-robustes-et-gestion-derreurs)
- [Patterns de récupération d'erreurs](#patterns-de-récupération-derreurs)
- [Tests et validation](#tests-et-validation)
- [Débogage avancé](#débogage-avancé)
- [Conclusion](#conclusion)

## Introduction

La gestion des erreurs constitue le pilier de la robustesse et de la fiabilité des scripts Bash. Au-delà de la simple détection des échecs, une bonne gestion d'erreurs permet de créer des scripts qui anticipent les problèmes, récupèrent gracieusement des situations d'erreur, et fournissent des informations utiles pour le débogage et la maintenance.

Imaginez la gestion d'erreurs comme un système immunitaire sophistiqué : elle détecte les anomalies, isole les problèmes, déclenche des réponses appropriées, et apprend des incidents passés pour mieux se défendre à l'avenir.

## Types d'erreurs en Bash

### Erreurs de syntaxe vs erreurs d'exécution

**Erreurs de syntaxe** :
```bash
# Erreur détectée lors du parsing
if [ $variable -eq 5 ]; then  # Erreur si variable vide
    echo "Variable vaut 5"
fi

# Mauvaise syntaxe de fonction
function ma_fonction {
    echo "Hello"
# Manque le }

# Erreur dans les expansions
echo "${array[non_existent]}"  # Variable array non définie
```

**Erreurs d'exécution** :
```bash
# Fichier inexistant
cat fichier_inexistant.txt

# Commande introuvable
commande_qui_nexiste_pas

# Permissions insuffisantes
touch /root/fichier_prive

# Erreur réseau
curl --connect-timeout 1 http://serveur.inaccessible

# Erreur arithmétique
resultat=$(( 10 / 0 ))  # Division par zéro
```

### Erreurs silencieuses vs erreurs bruyantes

**Erreurs silencieuses dangereuses** :
```bash
# Cette commande peut échouer silencieusement
cp source.txt destination.txt || true

# Variable non définie (pas d'erreur)
echo "Utilisateur: $USER_INEXISTANT"

# Commande qui peut réussir partiellement
mkdir -p /tmp/test && touch /root/test  # mkdir réussit, touch échoue

# Redirections qui masquent les erreurs
commande_qui_echoue > /dev/null 2>&1
```

**Erreurs explicites utiles** :
```bash
# Vérification explicite
if ! cp source.txt destination.txt; then
    echo "Erreur: Échec de la copie" >&2
    exit 1
fi

# Mode strict
set -euo pipefail

# Validation des variables
: ${VARIABLE_REQUISE:?"Variable obligatoire manquante"}

# Tests avec assertions
[[ -f "$fichier" ]] || { echo "Fichier requis manquant: $fichier" >&2; exit 1; }
```

## Codes de retour et variables spéciales

### Variable $?

**Utilisation basique de $?** :
```bash
# Code de retour de la dernière commande
ls /tmp
echo "Code de retour: $?"

# Test conditionnel
mkdir /tmp/test_directory
if [[ $? -eq 0 ]]; then
    echo "Répertoire créé avec succès"
else
    echo "Échec de création du répertoire"
fi

# Chaînage de vérifications
cp fichier.txt sauvegarde.txt && [[ $? -eq 0 ]] && echo "Sauvegarde réussie"
```

**Codes de retour spécifiques** :
```bash
# Codes de retour courants
commande_qui_reussit
case $? in
    0)  echo "Succès" ;;
    1)  echo "Erreur générale" ;;
    2)  echo "Erreur d'usage (mauvais arguments)" ;;
    126) echo "Commande trouvée mais non exécutable" ;;
    127) echo "Commande introuvable" ;;
    130) echo "Interruption (Ctrl+C)" ;;
    *)  echo "Code d'erreur inconnu: $?" ;;
esac
```

**Capture et propagation des codes d'erreur** :
```bash
# Fonction qui préserve les codes d'erreur
executer_et_preserve_code() {
    local commande="$1"
    shift

    # Exécution de la commande
    eval "$commande" "$@"

    # Capture et retour du code
    local code_retour=$?
    echo "Commande terminée avec le code: $code_retour" >&2
    return $code_retour
}

# Utilisation
executer_et_preserve_code "ls /inexistant"
echo "Code de retour de la fonction: $?"
```

### Variables spéciales de débogage

**Variables d'information système** :
```bash
# Informations sur l'exécution
echo "PID du script: $$"
echo "PID du parent: $PPID"
echo "Numéro de ligne: $LINENO"
echo "Nom du script: $0"
echo "Arguments: $@"
echo "Nombre d'arguments: $#"

# Fonction appelante (avec BASH_SOURCE et FUNCNAME)
afficher_contexte() {
    echo "Script: ${BASH_SOURCE[0]}"
    echo "Fonction: ${FUNCNAME[0]}"
    echo "Ligne: ${BASH_LINENO[0]}"
}

erreur_detection() {
    echo "Erreur détectée dans:"
    afficher_contexte
    echo "Commande qui a échoué: $BASH_COMMAND"
}

# Gestionnaire d'erreurs
trap erreur_detection ERR
```

## Gestion d'erreurs avec set et trap

### Options set pour le contrôle d'erreurs

**set -e : Arrêt sur erreur** :
```bash
# Sans set -e (comportement par défaut)
echo "Avant l'erreur"
cat fichier_inexistant
echo "Après l'erreur (affiché malgré l'erreur)"

# Avec set -e
set -e
echo "Avant l'erreur"
cat fichier_inexistant
echo "Après l'erreur (jamais affiché)"
```

**set -u : Erreur sur variables non définies** :
```bash
# Sans set -u
echo "Variable non définie: $VARIABLE_INEXISTANTE"

# Avec set -u
set -u
echo "Variable non définie: $VARIABLE_INEXISTANTE"  # Erreur
```

**set -o pipefail : Gestion des pipes** :
```bash
# Sans pipefail
echo "test" | grep "motif_inexistant" | wc -l  # Retourne 0 (wc réussit)

# Avec pipefail
set -o pipefail
echo "test" | grep "motif_inexistant" | wc -l  # Retourne 1 (grep échoue)
```

**Combinaison optimale** :
```bash
#!/bin/bash

# Configuration optimale pour les scripts de production
set -euo pipefail

# Gestionnaire d'erreurs global
trap 'echo "Erreur ligne $LINENO: $BASH_COMMAND" >&2; exit 1' ERR

# Script principal
echo "Début du script"

# Cette ligne va maintenant arrêter le script si elle échoue
cp fichier_source.txt fichier_destination.txt

echo "Fin du script (non atteinte en cas d'erreur)"
```

### Gestionnaire trap avancé

**Trap pour différents signaux** :
```bash
# Gestionnaire complet
cleanup() {
    local signal="$1"
    echo "Signal reçu: $signal" >&2

    # Nettoyage des ressources temporaires
    if [[ -d "/tmp/script_$$" ]]; then
        rm -rf "/tmp/script_$$"
        echo "Répertoire temporaire nettoyé" >&2
    fi

    # Fermeture des descripteurs de fichiers
    exec 3>&- 4>&-

    # Sauvegarde d'état si nécessaire
    if [[ -n "${ETAT_FICHIER:-}" ]]; then
        sauvegarder_etat "$ETAT_FICHIER"
    fi

    # Message final selon le signal
    case "$signal" in
        EXIT) echo "Script terminé normalement" >&2 ;;
        INT)  echo "Script interrompu par l'utilisateur" >&2 ;;
        TERM) echo "Script terminé par le système" >&2 ;;
        *)    echo "Script terminé (signal: $signal)" >&2 ;;
    esac

    exit 1
}

# Installation des gestionnaires
trap 'cleanup EXIT' EXIT
trap 'cleanup INT'  INT
trap 'cleanup TERM' TERM
trap 'cleanup HUP'  HUP

echo "Script démarré (PID: $$)"
echo "Appuyez sur Ctrl+C pour tester l'interruption"

# Simulation de travail
sleep 10

echo "Script terminé normalement"
```

**Trap avec fonction de callback** :
```bash
# Système de callbacks d'erreur
declare -a ERROR_CALLBACKS=()

ajouter_callback_erreur() {
    local callback="$1"
    ERROR_CALLBACKS+=("$callback")
}

executer_callbacks_erreur() {
    local code_erreur="$1"
    local message="$2"

    for callback in "${ERROR_CALLBACKS[@]}"; do
        if type "$callback" >/dev/null 2>&1; then
            "$callback" "$code_erreur" "$message"
        fi
    done
}

# Callbacks prédéfinis
log_erreur() {
    local code="$1"
    local message="$2"
    logger -p "user.err" "Script $0: $message (code: $code)"
}

email_erreur() {
    local code="$1"
    local message="$2"
    echo "Erreur dans $0: $message" | mail -s "Erreur script" admin@example.com
}

# Enregistrement des callbacks
ajouter_callback_erreur log_erreur
ajouter_callback_erreur email_erreur

# Gestionnaire principal
gestionnaire_erreur() {
    local code_erreur=$?
    local ligne=$1
    local commande="$2"

    local message="Erreur ligne $ligne: $commande (code: $code_erreur)"

    echo "$message" >&2
    executer_callbacks_erreur "$code_erreur" "$message"

    exit $code_erreur
}

trap 'gestionnaire_erreur $LINENO "$BASH_COMMAND"' ERR
```

## Fonctions robustes et gestion d'erreurs

### Fonctions avec contrat d'erreur

**Fonction avec gestion d'erreurs complète** :
```bash
# Fonction robuste avec contrat d'erreur défini
creer_sauvegarde() {
    local source="$1"
    local destination="${2:-${source}.bak}"
    local compression="${3:-gzip}"

    # Pré-conditions
    if [[ -z "$source" ]]; then
        echo "Erreur: Source requise" >&2
        return 10  # Code d'erreur spécifique
    fi

    if [[ ! -e "$source" ]]; then
        echo "Erreur: Source inexistante: $source" >&2
        return 11
    fi

    if [[ ! -r "$source" ]]; then
        echo "Erreur: Source non lisible: $source" >&2
        return 12
    fi

    # Vérification de l'espace disque
    local taille_source=$(stat -f%z "$source" 2>/dev/null || stat -c%s "$source" 2>/dev/null)
    local espace_disponible=$(df "$(dirname "$destination")" | tail -1 | awk '{print $4 * 1024}')

    if (( espace_disponible < taille_source * 2 )); then
        echo "Erreur: Espace disque insuffisant" >&2
        return 13
    fi

    # Création de la sauvegarde
    echo "Création de la sauvegarde: $source -> $destination"

    case "$compression" in
        gzip)
            if ! gzip -c "$source" > "${destination}.gz"; then
                echo "Erreur: Échec de la compression gzip" >&2
                return 20
            fi
            ;;
        bzip2)
            if ! bzip2 -c "$source" > "${destination}.bz2"; then
                echo "Erreur: Échec de la compression bzip2" >&2
                return 21
            fi
            ;;
        none)
            if ! cp "$source" "$destination"; then
                echo "Erreur: Échec de la copie" >&2
                return 22
            fi
            ;;
        *)
            echo "Erreur: Méthode de compression inconnue: $compression" >&2
            return 23
            ;;
    esac

    # Post-conditions
    local fichier_final="${destination}.gz"
    case "$compression" in
        bzip2) fichier_final="${destination}.bz2" ;;
        none)  fichier_final="$destination" ;;
    esac

    if [[ ! -f "$fichier_final" ]]; then
        echo "Erreur: Fichier de sauvegarde non créé" >&2
        return 30
    fi

    local taille_destination=$(stat -f%z "$fichier_final" 2>/dev/null || stat -c%s "$fichier_final" 2>/dev/null)
    if (( taille_destination == 0 )); then
        echo "Erreur: Fichier de sauvegarde vide" >&2
        rm -f "$fichier_final"
        return 31
    fi

    echo "Sauvegarde créée avec succès: $fichier_final"
    return 0
}

# Utilisation avec gestion fine des erreurs
if creer_sauvegarde "/etc/passwd" "/tmp/passwd.bak" "gzip"; then
    echo "✓ Sauvegarde réussie"
else
    code_erreur=$?
    case $code_erreur in
        10) echo "✗ Paramètre source manquant" ;;
        11) echo "✗ Source inexistante" ;;
        12) echo "✗ Source non lisible" ;;
        13) echo "✗ Espace disque insuffisant" ;;
        20) echo "✗ Échec compression gzip" ;;
        21) echo "✗ Échec compression bzip2" ;;
        22) echo "✗ Échec copie" ;;
        23) echo "✗ Méthode compression invalide" ;;
        30) echo "✗ Fichier non créé" ;;
        31) echo "✗ Fichier vide" ;;
        *)  echo "✗ Erreur inconnue (code: $code_erreur)" ;;
    esac
    exit 1
fi
```

### Fonctions avec gestion de ressources (RAII)

**Pattern RAII en Bash** :
```bash
# Gestionnaire de ressources avec RAII
with_resource() {
    local resource_type="$1"
    local resource_spec="$2"
    local command="$3"
    shift 3

    local resource_id=""

    # Allocation de la ressource
    case "$resource_type" in
        file)
            resource_id=$(mktemp)
            if [[ ! -f "$resource_id" ]]; then
                echo "Erreur: Impossible de créer le fichier temporaire" >&2
                return 1
            fi
            ;;
        directory)
            resource_id=$(mktemp -d)
            if [[ ! -d "$resource_id" ]]; then
                echo "Erreur: Impossible de créer le répertoire temporaire" >&2
                return 1
            fi
            ;;
        fd)
            # Allocation d'un descripteur de fichier
            for fd in {10..100}; do
                if ! { true >&$fd; } 2>/dev/null; then
                    resource_id=$fd
                    break
                fi
            done
            if [[ -z "$resource_id" ]]; then
                echo "Erreur: Aucun descripteur disponible" >&2
                return 1
            fi
            ;;
        *)
            echo "Erreur: Type de ressource inconnu: $resource_type" >&2
            return 1
            ;;
    esac

    # Exécution de la commande avec la ressource
    (
        # Export de la ressource vers l'environnement
        export RESOURCE_ID="$resource_id"
        export RESOURCE_TYPE="$resource_type"

        # Fonction de nettoyage automatique
        cleanup_resource() {
            case "$RESOURCE_TYPE" in
                file|directory)
                    rm -rf "$RESOURCE_ID"
                    ;;
                fd)
                    eval "exec $RESOURCE_ID>&-"
                    ;;
            esac
        }

        # Installation du nettoyage
        trap cleanup_resource EXIT

        # Exécution de la commande utilisateur
        eval "$command" "$@"
    )
}

# Utilisation
# Avec un fichier temporaire
with_resource file "" '
    echo "Contenu temporaire" > "$RESOURCE_ID"
    wc -l "$RESOURCE_ID"
'

# Avec un répertoire temporaire
with_resource directory "" '
    echo "Fichier dans répertoire temporaire" > "$RESOURCE_ID/test.txt"
    ls -la "$RESOURCE_ID"
'
```

## Patterns de récupération d'erreurs

### Retry automatique avec backoff

**Mécanisme de retry intelligent** :
```bash
# Fonction de retry avec backoff exponentiel
retry_with_backoff() {
    local max_attempts="${1:-3}"
    local base_delay="${2:-1}"
    local max_delay="${3:-60}"
    local backoff_multiplier="${4:-2}"
    local command="$5"
    shift 5

    local attempt=1
    local delay=$base_delay

    while (( attempt <= max_attempts )); do
        echo "Tentative $attempt/$max_attempts..."

        if eval "$command" "$@"; then
            echo "✓ Succès à la tentative $attempt"
            return 0
        fi

        if (( attempt == max_attempts )); then
            echo "✗ Échec définitif après $max_attempts tentatives"
            return 1
        fi

        echo "Échec, prochaine tentative dans ${delay}s..."
        sleep "$delay"

        # Calcul du délai pour la prochaine tentative
        delay=$(( delay * backoff_multiplier ))
        if (( delay > max_delay )); then
            delay=$max_delay
        fi

        # Ajout de jitter pour éviter la synchronisation
        delay=$(( delay + RANDOM % delay / 4 ))

        ((attempt++))
    done
}

# Utilisation
retry_with_backoff 5 2 30 2 "curl --connect-timeout 5 http://unstable-api.com"
```

### Circuit breaker pattern

**Interrupteur automatique** :
```bash
# Implémentation du pattern Circuit Breaker
declare -A CIRCUIT_BREAKERS=()

circuit_breaker() {
    local name="$1"
    local failure_threshold="${2:-5}"
    local recovery_timeout="${3:-60}"
    local command="$4"
    shift 4

    # État du circuit breaker
    local state="${CIRCUIT_BREAKERS[${name}_state]:-closed}"
    local failures="${CIRCUIT_BREAKERS[${name}_failures]:-0}"
    local last_failure="${CIRCUIT_BREAKERS[${name}_last_failure]:-0}"

    case "$state" in
        open)
            # Vérifier si on peut passer en half-open
            local now=$(date +%s)
            if (( now - last_failure >= recovery_timeout )); then
                echo "Circuit breaker '$name': Passage en half-open"
                CIRCUIT_BREAKERS[${name}_state]="half-open"
                state="half-open"
            else
                echo "Circuit breaker '$name': Ouvert (timeout)"
                return 1
            fi
            ;;
    esac

    # Exécution de la commande
    if eval "$command" "$@"; then
        # Succès
        case "$state" in
            half-open)
                echo "Circuit breaker '$name': Fermeture (récupération réussie)"
                CIRCUIT_BREAKERS[${name}_state]="closed"
                CIRCUIT_BREAKERS[${name}_failures]=0
                ;;
            closed)
                CIRCUIT_BREAKERS[${name}_failures]=0
                ;;
        esac
        return 0
    else
        # Échec
        ((failures++))
        CIRCUIT_BREAKERS[${name}_failures]=$failures
        CIRCUIT_BREAKERS[${name}_last_failure]=$(date +%s)

        if (( failures >= failure_threshold )); then
            echo "Circuit breaker '$name': Ouverture (seuil dépassé)"
            CIRCUIT_BREAKERS[${name}_state]="open"
        fi

        return 1
    fi
}

# Utilisation
for i in {1..10}; do
    if circuit_breaker "api_call" 3 30 "curl -s http://api.example.com"; then
        echo "Appel $i réussi"
    else
        echo "Appel $i échoué"
    fi
    sleep 1
done
```

### Gestion d'erreurs en cascade

**Propagation d'erreurs avec contexte** :
```bash
# Pile d'erreurs pour le contexte
declare -a ERROR_STACK=()

push_error_context() {
    local context="$1"
    ERROR_STACK+=("$context")
}

pop_error_context() {
    if (( ${#ERROR_STACK[@]} > 0 )); then
        unset 'ERROR_STACK[${#ERROR_STACK[@]}-1]'
    fi
}

get_error_context() {
    if (( ${#ERROR_STACK[@]} > 0 )); then
        echo "Contexte d'erreur: ${ERROR_STACK[*]}"
    fi
}

# Fonction avec gestion d'erreurs en cascade
fonction_a() {
    push_error_context "fonction_a"

    if ! fonction_b; then
        echo "Erreur dans fonction_a:" >&2
        get_error_context >&2
        pop_error_context
        return 1
    fi

    pop_error_context
    return 0
}

fonction_b() {
    push_error_context "fonction_b"

    if ! commande_qui_peut_echouer; then
        echo "Erreur dans fonction_b:" >&2
        get_error_context >&2
        pop_error_context
        return 1
    fi

    pop_error_context
    return 0
}

# Utilisation
if ! fonction_a; then
    echo "Erreur détectée avec contexte complet"
fi
```

## Tests et validation

### Framework de tests unitaires pour erreurs

**Tests d'erreurs prédéfinis** :
```bash
# Framework de test pour les erreurs
declare -i TESTS_REUSSIS=0
declare -i TESTS_ECHOUES=0

# Fonction d'assertion pour les erreurs
assert_fails() {
    local expected_code="${1:-1}"
    local command="$2"
    shift 2

    eval "$command" "$@"
    local actual_code=$?

    if [[ $actual_code -eq $expected_code ]]; then
        echo "✓ Test réussi: commande a échoué avec le code attendu $expected_code"
        ((TESTS_REUSSIS++))
    else
        echo "✗ Test échoué: attendu code $expected_code, obtenu $actual_code"
        echo "  Commande: $command"
        ((TESTS_ECHOUES++))
    fi
}

assert_succeeds() {
    local command="$1"
    shift

    if eval "$command" "$@"; then
        echo "✓ Test réussi: commande a réussi"
        ((TESTS_REUSSIS++))
    else
        echo "✗ Test échoué: commande a échoué (code: $?)"
        echo "  Commande: $command"
        ((TESTS_ECHOUES++))
    fi
}

assert_output_contains() {
    local expected_pattern="$1"
    local command="$2"
    shift 2

    local output
    if output=$(eval "$command" "$@"); then
        if [[ "$output" =~ $expected_pattern ]]; then
            echo "✓ Test réussi: sortie contient '$expected_pattern'"
            ((TESTS_REUSSIS++))
        else
            echo "✗ Test échoué: '$expected_pattern' non trouvé dans la sortie"
            echo "  Sortie: $output"
            ((TESTS_ECHOUES++))
        fi
    else
        echo "✗ Test échoué: commande a échoué (code: $?)"
        ((TESTS_ECHOUES++))
    fi
}

# Tests des fonctions de gestion d'erreurs
echo "=== Tests de gestion d'erreurs ==="

# Test de fonction robuste
assert_fails 10 "creer_sauvegarde"  # Source manquante
assert_fails 11 "creer_sauvegarde /inexistant"  # Source inexistante
assert_succeeds "creer_sauvegarde /etc/passwd"  # Cas normal

# Test de circuit breaker
assert_fails 1 "circuit_breaker test_cb 1 1 false"  # Doit échouer
# Après échec, le circuit devrait être ouvert
assert_fails 1 "circuit_breaker test_cb 1 1 true"   # Doit échouer car ouvert

# Résumé des tests
echo
echo "=== Résumé des tests ==="
echo "Réussis: $TESTS_REUSSIS"
echo "Échoués: $TESTS_ECHOUES"
echo "Total: $((TESTS_REUSSIS + TESTS_ECHOUES))"
```

### Tests d'intégration avec simulation d'erreurs

**Simulation d'environnements défaillants** :
```bash
# Fonction de simulation d'erreurs
simulate_failure() {
    local failure_type="$1"
    local command="$2"
    shift 2

    case "$failure_type" in
        network)
            # Simulation de panne réseau
            export http_proxy="http://nonexistent:3128"
            eval "$command" "$@"
            unset http_proxy
            ;;
        disk_full)
            # Simulation de disque plein
            local temp_dir=$(mktemp -d)
            mount -t tmpfs -o size=1K tmpfs "$temp_dir"
            (
                cd "$temp_dir"
                eval "$command" "$@"
            )
            umount "$temp_dir"
            rmdir "$temp_dir"
            ;;
        permission)
            # Simulation de problèmes de permissions
            if [[ $EUID -eq 0 ]]; then
                # Si root, tester avec un utilisateur normal
                su nobody -c "$command $@"
            else
                # Si pas root, utiliser sudo si disponible
                if command -v sudo >/dev/null; then
                    sudo -u nobody "$command" "$@"
                else
                    eval "$command" "$@"
                fi
            fi
            ;;
        *)
            eval "$command" "$@"
            ;;
    esac
}

# Tests avec simulation d'erreurs
echo "=== Tests avec simulation d'erreurs ==="

# Test réseau
echo "Test réseau:"
if simulate_failure network "curl -s --max-time 5 http://httpbin.org/ip"; then
    echo "✓ Réseau fonctionne"
else
    echo "✗ Problème réseau simulé"
fi

# Test espace disque
echo "Test espace disque:"
if simulate_failure disk_full "dd if=/dev/zero of=test.dat bs=1M count=10"; then
    echo "✓ Espace disque suffisant"
else
    echo "✗ Espace disque insuffisant (simulé)"
fi
```

## Débogage avancé

### Mode debug avec traçage complet

**Configuration de débogage étendue** :
```bash
# Fonction de débogage avancé
enable_debug_mode() {
    local level="${1:-basic}"

    case "$level" in
        basic)
            set -x  # Trace des commandes
            ;;
        verbose)
            set -xv  # Trace étendue
            PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
            ;;
        full)
            set -xv
            PS4='+(${BASH_SOURCE}:${LINENO}): ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'

            # Trap pour les signaux de débogage
            trap 'echo "Signal DEBUG: $BASH_COMMAND (ligne $LINENO)" >&2' DEBUG

            # Fonction de logging automatique
            log_debug() {
                echo "DEBUG: $*" >&2
            }

            # Surcharge des commandes importantes
            alias cp='log_debug "cp $*" && cp'
            alias mv='log_debug "mv $*" && mv'
            alias rm='log_debug "rm $*" && rm'
            ;;
    esac

    echo "Mode debug activé: $level" >&2
}

disable_debug_mode() {
    set +x
    set +v
    trap - DEBUG 2>/dev/null || true
    unalias cp mv rm 2>/dev/null || true
    echo "Mode debug désactivé" >&2
}

# Utilisation
enable_debug_mode full

# Script avec débogage
cp fichier1.txt fichier2.txt
mv fichier2.txt backup/
rm fichier1.txt

disable_debug_mode
```

### Analyse postmortem des erreurs

**Collecte d'informations de débogage** :
```bash
# Fonction de postmortem
postmortem_analysis() {
    local exit_code="$1"
    local script_name="${2:-$0}"
    local output_file="${3:-postmortem_$(date +%Y%m%d_%H%M%S).log}"

    {
        echo "=== ANALYSE POSTMORTEM ==="
        echo "Date: $(date)"
        echo "Script: $script_name"
        echo "Code de sortie: $exit_code"
        echo "Utilisateur: $(id)"
        echo "Répertoire de travail: $(pwd)"
        echo "PID: $$"
        echo

        echo "=== ENVIRONNEMENT ==="
        echo "Bash version: $BASH_VERSION"
        echo "Options shell: $-"
        echo "PATH: $PATH"
        echo

        echo "=== VARIABLES SYSTÈME ==="
        echo "UPTIME: $(uptime)"
        echo "CHARGE: $(cat /proc/loadavg 2>/dev/null || echo 'N/A')"
        echo "MÉMOIRE: $(free -h 2>/dev/null || echo 'N/A')"
        echo "DISQUE: $(df -h . 2>/dev/null || echo 'N/A')"
        echo

        echo "=== DERNIÈRES COMMANDES ==="
        history 10 2>/dev/null || echo "Historique non disponible"
        echo

        echo "=== FICHIERS OUVERTS ==="
        ls -la /proc/$$/fd/ 2>/dev/null || echo "Informations non disponibles"
        echo

        echo "=== STACK TRACE ==="
        local i
        for ((i=0; i<${#FUNCNAME[@]}; i++)); do
            echo "  ${FUNCNAME[$i]}() in ${BASH_SOURCE[$i]}:${BASH_LINENO[$i]}"
        done

    } > "$output_file"

    echo "Analyse postmortem sauvegardée: $output_file" >&2
}

# Gestionnaire d'erreur avec postmortem
fatal_error() {
    local exit_code=$?
    local ligne=$1
    local commande="$2"

    echo "ERREUR FATALE ligne $ligne: $commande" >&2
    echo "Code de sortie: $exit_code" >&2

    # Analyse postmortem
    postmortem_analysis "$exit_code" "$0"

    exit $exit_code
}

trap 'fatal_error $LINENO "$BASH_COMMAND"' ERR
```

### Profilage des performances

**Mesure des performances avec gestion d'erreurs** :
```bash
# Fonction de profiling avec gestion d'erreurs
profile_command() {
    local command="$1"
    local output_file="${2:-profile_$(date +%Y%m%d_%H%M%S).log}"
    shift 2

    local start_time=$(date +%s.%N 2>/dev/null || date +%s)
    local start_mem=$(ps -o rss= $$ 2>/dev/null || echo "0")

    # Redirections pour capturer stdout/stderr séparément
    exec 3>&1 4>&2

    {
        echo "=== PROFILAGE DE COMMANDE ==="
        echo "Commande: $command"
        echo "Arguments: $@"
        echo "Début: $(date)"
        echo

        # Exécution avec timing
        if time eval "$command" "$@" 2>&4; then
            local exit_code=0
            echo "Résultat: SUCCÈS"
        else
            local exit_code=$?
            echo "Résultat: ÉCHEC (code: $exit_code)"
        fi

        local end_time=$(date +%s.%N 2>/dev/null || date +%s)
        local end_mem=$(ps -o rss= $$ 2>/dev/null || echo "0")

        echo
        echo "=== MÉTRIQUES ==="
        echo "Code de sortie: $exit_code"

        # Calcul du temps écoulé
        if command -v bc >/dev/null; then
            local duration=$(echo "$end_time - $start_time" | bc -l 2>/dev/null || echo "N/A")
            echo "Temps écoulé: ${duration}s"
        fi

        # Calcul de la mémoire utilisée
        local mem_diff=$(( end_mem - start_mem ))
        echo "Mémoire utilisée: ${mem_diff}KB"

        echo "Fin: $(date)"

    } > "$output_file" 2>&1

    # Restauration des descripteurs
    exec 3>&- 4>&-

    echo "Profilage terminé: $output_file" >&2
    return $exit_code
}

# Utilisation
profile_command "sleep 2 && echo 'Test terminé'" profile_test.log
```

## Conclusion

La gestion d'erreurs en Bash transforme les scripts fragiles en programmes robustes capables de fonctionner de manière fiable dans des environnements hostiles. En maîtrisant les codes de retour, les options set, les gestionnaires trap, et les patterns de récupération, vous créez des scripts qui non seulement détectent les erreurs, mais les anticipent et les récupèrent gracieusement.

Comme un pilote expérimenté qui sait gérer les situations d'urgence, le programmeur Bash compétent transforme les erreurs en occasions d'apprentissage et d'amélioration continue du système.

Dans le chapitre suivant, nous explorerons les techniques de débogage avancé, découvrant comment inspecter, analyser et résoudre les problèmes complexes dans les scripts Bash.
