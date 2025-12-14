# Chapitre 73 - Conditions et boucles complexes

## Table des matières
- [Introduction](#introduction)
- [Structures conditionnelles avancées](#structures-conditionnelles-avancées)
- [Boucles imbriquées et optimisation](#boucles-imbriquées-et-optimisation)
- [Contrôle de flux avancé](#contrôle-de-flux-avancé)
- [Gestion des erreurs dans les structures](#gestion-des-erreurs-dans-les-structures)
- [Optimisation des performances](#optimisation-des-performances)
- [Patterns de programmation courants](#patterns-de-programmation-courants)
- [Débogage et bonnes pratiques](#débogage-et-bonnes-pratiques)
- [Conclusion](#conclusion)

## Introduction

Les conditions et boucles complexes constituent l'épine dorsale de la logique programmatique en Bash. Au-delà des simples `if` et `for`, Bash offre des structures sophistiquées qui permettent de créer des scripts adaptatifs, capables de prendre des décisions intelligentes et d'itérer efficacement sur des données complexes.

Imaginez les structures de contrôle comme le système nerveux d'un organisme vivant : les conditions sont les synapses qui décident quoi faire, tandis que les boucles sont les circuits qui répètent les actions essentielles. Une programmation efficace combine ces éléments pour créer des comportements émergents complexes.

## Structures conditionnelles avancées

### Conditions imbriquées et composées

**Opérateurs logiques avancés** :
```bash
# Conditions composées avec &&
if [[ -f "$config_file" ]] && [[ -r "$config_file" ]] && [[ -s "$config_file" ]]; then
    echo "Fichier de configuration valide et accessible"
    source "$config_file"
else
    echo "Erreur: Configuration invalide ou inaccessible"
    exit 1
fi

# Conditions avec || et priorités
if [[ "$USER" == "root" ]] || [[ "$EUID" -eq 0 ]] || id -u >/dev/null 2>&1; then
    echo "Privilèges administrateur détectés"
    ADMIN=true
else
    echo "Utilisateur standard"
    ADMIN=false
fi
```

**Conditions avec parenthèses pour le groupement** :
```bash
# Groupement logique explicite
age=25
pays="FR"
if [[ ( $age -ge 18 && $age -le 65 ) && ( "$pays" == "FR" || "$pays" == "BE" ) ]]; then
    echo "Éligible au programme européen"
fi

# Conditions complexes avec négation
if [[ ! ( "$role" == "admin" || "$role" == "moderator" ) && $active == true ]]; then
    echo "Accès utilisateur standard"
fi
```

### Opérateurs conditionnels étendus

**Tests arithmétiques avancés** :
```bash
# Comparaisons multiples
score=85
if (( score >= 90 )); then
    grade="A"
    bonus=100
elif (( score >= 80 )); then
    grade="B"
    bonus=50
elif (( score >= 70 )); then
    grade="C"
    bonus=25
else
    grade="F"
    bonus=0
fi

echo "Note: $grade, Bonus: $bonus€"
```

**Tests sur les chaînes avec expressions régulières** :
```bash
# Validation de format avec regex
valider_email() {
    local email="$1"
    if [[ "$email" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]; then
        return 0
    else
        return 1
    fi
}

# Utilisation dans une condition
read -p "Email: " user_email
if valider_email "$user_email"; then
    echo "Email valide, envoi du message..."
    envoyer_notification "$user_email"
else
    echo "Format d'email invalide"
    exit 1
fi
```

**Conditions basées sur des motifs de fichiers** :
```bash
# Vérification de type de fichier par extension
verifier_fichier() {
    local fichier="$1"
    case "${fichier##*.}" in
        jpg|jpeg|png|gif)
            echo "Image: $fichier"
            traiter_image "$fichier"
            ;;
        txt|md|rst)
            echo "Document texte: $fichier"
            traiter_texte "$fichier"
            ;;
        pdf)
            echo "PDF: $fichier"
            traiter_pdf "$fichier"
            ;;
        *)
            echo "Type inconnu: $fichier"
            return 1
            ;;
    esac
}
```

### Structures conditionnelles avec case

**Case avec motifs avancés** :
```bash
# Traitement d'options de ligne de commande
traiter_option() {
    case "$1" in
        -h|--help)
            afficher_aide
            exit 0
            ;;
        -v|--verbose)
            VERBOSE=true
            ;;
        -q|--quiet)
            QUIET=true
            ;;
        -f|--file)
            if [[ -z "$2" ]]; then
                echo "Erreur: -f nécessite un argument" >&2
                exit 1
            fi
            FICHIER="$2"
            shift
            ;;
        --debug=*)
            DEBUG_LEVEL="${1#*=}"
            ;;
        *)
            echo "Option inconnue: $1" >&2
            echo "Utilisez -h pour l'aide" >&2
            exit 1
            ;;
    esac
}

# Boucle de traitement des arguments
while [[ $# -gt 0 ]]; do
    traiter_option "$1"
    shift
done
```

**Case avec expressions régulières (Bash 4.0+)** :
```bash
# Classification de versions
classifier_version() {
    local version="$1"
    case "$version" in
        [0-9].[0-9][0-9].[0-9][0-9][0-9])
            echo "Version de développement"
            ;;
        [0-9].[0-9][0-9].[0-9][0-9])
            echo "Version bêta"
            ;;
        [0-9].[0-9].[0-9])
            echo "Version stable"
            ;;
        *)
            echo "Format de version non reconnu"
            return 1
            ;;
    esac
}
```

## Boucles imbriquées et optimisation

### Boucles for avec itération complexe

**Itération sur des tableaux multidimensionnels** :
```bash
# Matrice de serveurs par environnement
declare -A serveurs=(
    ["prod"]="web01 web02 db01 cache01"
    ["staging"]="web-staging01 db-staging01"
    ["dev"]="web-dev01"
)

# Boucle imbriquée pour maintenance
for environnement in "${!serveurs[@]}"; do
    echo "=== Maintenance $environnement ==="
    for serveur in ${serveurs[$environnement]}; do
        echo "Vérification de $serveur..."
        if ping -c 1 -W 1 "$serveur" >/dev/null 2>&1; then
            echo "✓ $serveur accessible"
            # Effectuer la maintenance
            maintenance_serveur "$serveur"
        else
            echo "✗ $serveur inaccessible"
            logger "Serveur $serveur dans $environnement indisponible"
        fi
    done
    echo
done
```

**Boucles avec contrôle de flux avancé** :
```bash
# Recherche dans des fichiers avec break/continue
rechercher_dans_logs() {
    local motif="$1"
    local max_resultats="${2:-10}"
    local compteur=0

    for fichier in /var/log/*.log; do
        if [[ ! -r "$fichier" ]]; then
            echo "Avertissement: $fichier non lisible" >&2
            continue
        fi

        while IFS= read -r ligne; do
            if (( compteur >= max_resultats )); then
                break 2  # Sort de la boucle while ET for
            fi

            if [[ "$ligne" == *"$motif"* ]]; then
                echo "$fichier: $ligne"
                ((compteur++))
            fi
        done < "$fichier"
    done

    echo "Total trouvé: $compteur résultats"
}
```

### Boucles while avec conditions complexes

**Boucles de surveillance avec timeout** :
```bash
# Attendre qu'un service soit disponible
attendre_service() {
    local service="$1"
    local timeout="${2:-30}"
    local compteur=0

    echo "Attente du service $service..."

    while ! curl -s --max-time 1 "$service" >/dev/null 2>&1; do
        if (( compteur >= timeout )); then
            echo "Timeout: $service non disponible après $timeout secondes"
            return 1
        fi

        echo -n "."
        sleep 1
        ((compteur++))
    done

    echo " ✓ $service disponible"
    return 0
}

# Utilisation
if attendre_service "http://localhost:8080/api/health"; then
    echo "Service prêt, démarrage des tests..."
    executer_tests
fi
```

**Boucles avec lecture de flux multiples** :
```bash
# Traiter deux fichiers en parallèle
traiter_parallele() {
    local fichier1="$1"
    local fichier2="$2"

    # Vérifier que les fichiers existent
    [[ -f "$fichier1" ]] || { echo "Fichier 1 introuvable: $fichier1"; return 1; }
    [[ -f "$fichier2" ]] || { echo "Fichier 2 introuvable: $fichier2"; return 1; }

    # Lecture parallèle avec file descriptors
    exec 3<"$fichier1"
    exec 4<"$fichier2"

    local ligne1 ligne2 numero_ligne=1

    while true; do
        read -r ligne1 <&3 || break
        read -r ligne2 <&4 || break

        echo "Ligne $numero_ligne:"
        echo "  Fichier1: $ligne1"
        echo "  Fichier2: $ligne2"

        # Traitement personnalisé des paires de lignes
        comparer_lignes "$ligne1" "$ligne2"

        ((numero_ligne++))
    done

    exec 3<&-
    exec 4<&-
}
```

### Optimisation des boucles

**Réduction du nombre d'opérations coûteuses** :
```bash
# Mauvaise approche : appel système à chaque itération
for fichier in *.txt; do
    if [[ -f "$fichier" ]] && [[ -r "$fichier" ]] && wc -l < "$fichier" -gt 10; then
        traiter_fichier "$fichier"
    fi
done

# Bonne approche : calcul préalable
echo "Préparation de la liste des fichiers..."
declare -a fichiers_valides=()

for fichier in *.txt; do
    # Vérifications groupées
    if [[ -f "$fichier" ]] && [[ -r "$fichier" ]]; then
        nb_lignes=$(wc -l < "$fichier")
        if (( nb_lignes > 10 )); then
            fichiers_valides+=("$fichier")
        fi
    fi
done

echo "Traitement de ${#fichiers_valides[@]} fichiers..."
for fichier in "${fichiers_valides[@]}"; do
    traiter_fichier "$fichier"
done
```

**Utilisation de commandes externes optimisées** :
```bash
# Traitement inefficace
total=0
for fichier in *.log; do
    if [[ -f "$fichier" ]]; then
        lignes=$(wc -l < "$fichier")
        ((total += lignes))
    fi
done
echo "Total: $total lignes"

# Traitement optimisé avec xargs
total=$(find . -name "*.log" -type f -print0 | xargs -0 wc -l | tail -1 | awk '{print $1}')
echo "Total: $total lignes"
```

## Contrôle de flux avancé

### Instructions break et continue avec niveaux

**Break et continue dans boucles imbriquées** :
```bash
# Recherche dans une matrice
chercher_dans_matrice() {
    local valeur="$1"
    local -a matrice=(
        "1 2 3 4"
        "5 6 7 8"
        "9 10 11 12"
    )

    local i j
    for ((i=0; i<${#matrice[@]}; i++)); do
        for j in ${matrice[$i]}; do
            if [[ "$j" == "$valeur" ]]; then
                echo "Trouvé $valeur à la position [$i,$j]"
                return 0
            fi
        done
    done

    echo "Valeur $valeur non trouvée"
    return 1
}

# Break avec niveau spécifique
traiter_donnees() {
    while read -r ligne; do
        case "$ligne" in
            START*)
                echo "Début de section: $ligne"
                while read -r sous_ligne; do
                    case "$sous_ligne" in
                        END*)
                            echo "Fin de section"
                            break 2  # Sort des deux boucles
                            ;;
                        ERROR*)
                            echo "Erreur détectée: $sous_ligne"
                            break 2
                            ;;
                        *)
                            traiter_sous_ligne "$sous_ligne"
                            ;;
                    esac
                done
                ;;
            *)
                echo "Ligne ignorée: $ligne"
                ;;
        esac
    done < "$fichier_donnees"
}
```

### Gestion des signaux dans les boucles

**Boucles interrompibles proprement** :
```bash
# Gestionnaire de signaux
cleanup() {
    echo -e "\nInterruption détectée, nettoyage..."
    rm -f /tmp/temp_*.tmp
    exit 1
}

trap cleanup INT TERM

# Boucle principale avec vérification périodique
echo "Traitement en cours (Ctrl+C pour arrêter)..."
while true; do
    # Vérification d'arrêt propre
    if [[ -f "/tmp/stop_processing" ]]; then
        echo "Signal d'arrêt reçu"
        break
    fi

    # Traitement d'une unité de travail
    traiter_unite_suivante

    # Pause courte pour éviter la surcharge CPU
    sleep 0.1
done

# Nettoyage final
cleanup
```

**Timeout sur les opérations longues** :
```bash
# Fonction avec timeout
executer_avec_timeout() {
    local commande="$1"
    local timeout="${2:-30}"

    # Lancer en arrière-plan
    eval "$commande" &
    local pid=$!

    # Attendre avec timeout
    local compteur=0
    while kill -0 $pid 2>/dev/null; do
        if (( compteur >= timeout )); then
            echo "Timeout: arrêt forcé du processus $pid"
            kill -TERM $pid 2>/dev/null
            sleep 1
            kill -KILL $pid 2>/dev/null
            return 1
        fi
        sleep 1
        ((compteur++))
    done

    # Récupérer le code de sortie
    wait $pid
    return $?
}
```

## Gestion des erreurs dans les structures

### Gestion d'erreurs avec set -e et pièges

**Mode strict avec gestion fine des erreurs** :
```bash
# Configuration du mode strict
set -euo pipefail

# Fonction de gestion d'erreurs
erreur_handler() {
    local ligne="$1"
    local commande="$2"
    echo "Erreur ligne $ligne: $commande" >&2
    echo "Code de sortie: $?" >&2

    # Nettoyage d'urgence
    cleanup_temp_files
    exit 1
}

trap 'erreur_handler $LINENO "$BASH_COMMAND"' ERR

# Script principal avec gestion d'erreurs sélective
creer_sauvegarde() {
    local source="$1"
    local destination="$2"

    # Cette commande peut échouer sans arrêter le script
    mkdir -p "$destination" 2>/dev/null || true

    # Ces commandes doivent réussir
    if ! cp -r "$source" "$destination/"; then
        echo "Erreur: Échec de la copie" >&2
        return 1
    fi

    if ! tar -czf "${destination}.tar.gz" -C "$destination" .; then
        echo "Erreur: Échec de l'archivage" >&2
        return 1
    fi

    return 0
}

# Utilisation avec gestion d'erreur
if creer_sauvegarde "/important" "/backup"; then
    echo "Sauvegarde réussie"
else
    echo "Échec de la sauvegarde"
    exit 1
fi
```

### Validation de données avec assert

**Fonction assert pour validation** :
```bash
# Fonction d'assertion
assert() {
    local condition="$1"
    local message="${2:-Assertion échouée}"

    if ! eval "$condition"; then
        echo "ASSERTION ÉCHOUÉE: $message" >&2
        echo "Condition: $condition" >&2
        echo "Ligne: $LINENO" >&2
        return 1
    fi
}

# Fonction de validation d'utilisateur
valider_utilisateur() {
    local username="$1"
    local uid="$2"

    # Assertions multiples
    assert '[[ -n "$username" ]]' "Nom d'utilisateur requis"
    assert '[[ "$username" =~ ^[a-zA-Z][a-zA-Z0-9_-]*$ ]]' "Format de nom invalide"
    assert '(( uid >= 1000 ))' "UID doit être >= 1000 pour les utilisateurs"
    assert '! id "$username" >/dev/null 2>&1' "Utilisateur $username existe déjà"

    return 0
}

# Utilisation
if valider_utilisateur "john_doe" "1500"; then
    echo "Validation réussie, création de l'utilisateur..."
    useradd -u 1500 john_doe
else
    echo "Validation échouée"
    exit 1
fi
```

## Optimisation des performances

### Cache et mémoïsation

**Cache des résultats coûteux** :
```bash
# Cache pour les calculs coûteux
declare -A cache_resultats
declare -A cache_timestamps

calculer_cout() {
    local parametre="$1"
    local ttl="${2:-3600}"  # TTL par défaut 1h

    # Vérifier le cache
    if [[ -v cache_resultats[$parametre] ]] && [[ -v cache_timestamps[$parametre] ]]; then
        local age=$(( $(date +%s) - ${cache_timestamps[$parametre]} ))
        if (( age < ttl )); then
            echo "${cache_resultats[$parametre]}"
            return 0
        fi
    fi

    # Calcul réel (simulé)
    echo "Calcul coûteux pour $parametre..." >&2
    sleep 2  # Simulation d'un calcul long
    local resultat=$(( RANDOM % 1000 ))

    # Mettre en cache
    cache_resultats[$parametre]=$resultat
    cache_timestamps[$parametre]=$(date +%s)

    echo "$resultat"
}

# Utilisation avec cache automatique
for i in {1..5}; do
    echo "Résultat pour $i: $(calculer_cout "$i" 30)"
done
```

### Parallélisation des traitements

**Exécution parallèle avec contrôle** :
```bash
# Fonction de traitement parallèle
traiter_en_parallele() {
    local max_parallele="${1:-4}"
    local -a pids=()
    local compteur=0

    while IFS= read -r element; do
        # Lancer le traitement en arrière-plan
        traiter_element "$element" &
        pids+=($!)

        ((compteur++))

        # Attendre si on atteint la limite
        if (( ${#pids[@]} >= max_parallele )); then
            # Attendre que le premier processus se termine
            wait "${pids[0]}"
            pids=("${pids[@]:1}")
        fi
    done

    # Attendre que tous les processus restants se terminent
    for pid in "${pids[@]}"; do
        wait "$pid"
    done

    echo "Traités $compteur éléments"
}

# Utilisation
cat liste_elements.txt | traiter_en_parallele 8
```

## Patterns de programmation courants

### Pattern Strategy pour les algorithmes

**Sélection d'algorithmes selon les critères** :
```bash
# Stratégies de tri
trier_tableau() {
    local -a tableau=("$@")
    local taille=${#tableau[@]}

    if (( taille < 10 )); then
        # Petit tableau: tri à bulles simple
        trier_bulles "${tableau[@]}"
    elif (( taille < 1000 )); then
        # Tableau moyen: quicksort
        quicksort "${tableau[@]}"
    else
        # Grand tableau: tri externe
        tri_externe "${tableau[@]}"
    fi
}

# Pattern Factory pour les objets
creer_logger() {
    local type="$1"
    local destination="$2"

    case "$type" in
        file)
            logger_fichier "$destination"
            ;;
        syslog)
            logger_syslog "$destination"
            ;;
        journald)
            logger_journald "$destination"
            ;;
        *)
            echo "Type de logger inconnu: $type" >&2
            return 1
            ;;
    esac
}
```

### Pattern Observer pour les événements

**Système d'événements simple** :
```bash
# Observers enregistrés
declare -a observers_pre_traitement=()
declare -a observers_post_traitement=()

# Enregistrement d'observeurs
ajouter_observer() {
    local moment="$1"
    local fonction="$2"

    case "$moment" in
        pre)
            observers_pre_traitement+=("$fonction")
            ;;
        post)
            observers_post_traitement+=("$fonction")
            ;;
        *)
            echo "Moment invalide: $moment" >&2
            return 1
            ;;
    esac
}

# Notification des observers
notifier_observers() {
    local moment="$1"
    shift
    local -a observers

    case "$moment" in
        pre)
            observers=("${observers_pre_traitement[@]}")
            ;;
        post)
            observers=("${observers_post_traitement[@]}")
            ;;
    esac

    for observer in "${observers[@]}"; do
        if [[ $(type -t "$observer") == "function" ]]; then
            "$observer" "$@"
        fi
    done
}

# Utilisation
logger_pre() { logger "Pré-traitement: $*" ; }
logger_post() { logger "Post-traitement: $*" ; }

ajouter_observer "pre" "logger_pre"
ajouter_observer "post" "logger_post"

# Dans le traitement principal
notifier_observers "pre" "début traitement"
traiter_donnees "$@"
notifier_observers "post" "fin traitement"
```

## Débogage et bonnes pratiques

### Débogage des structures conditionnelles

**Trace des conditions avec set -x** :
```bash
# Fonction de débogage des conditions
debug_condition() {
    local condition="$1"
    local message="$2"

    if [[ "$DEBUG" == "true" ]]; then
        echo "DEBUG: Test de $message"
        echo "DEBUG: Condition: $condition"
        set -x
    fi

    if eval "$condition"; then
        [[ "$DEBUG" == "true" ]] && set +x
        return 0
    else
        [[ "$DEBUG" == "true" ]] && set +x
        return 1
    fi
}

# Utilisation
if debug_condition '[[ -f "$config" ]]' "existence du fichier de config"; then
    echo "Configuration trouvée"
fi
```

**Validation des boucles** :
```bash
# Fonction de validation de boucle
valider_boucle() {
    local type="$1"
    local condition="$2"
    local message="$3"

    case "$type" in
        while|until)
            if ! eval "$condition" 2>/dev/null; then
                echo "AVERTISSEMENT: Condition de boucle invalide: $condition"
                echo "Message: $message"
                return 1
            fi
            ;;
        for)
            # Validation spécifique pour les boucles for
            ;;
    esac

    return 0
}

# Boucle avec validation
valider_boucle "while" '[[ $compteur -lt 10 ]]' "compteur doit être numérique"
while [[ $compteur -lt 10 ]]; do
    echo "Itération $compteur"
    ((compteur++))
done
```

### Bonnes pratiques de structure

**Lisibilité et maintenance** :
```bash
# Mauvaise pratique: tout dans une seule ligne
if [[ $a -gt 5 ]] && [[ $b == "test" ]] || [[ $c -lt 10 ]]; then echo "complexe"; fi

# Bonne pratique: structure claire
seuil_atteint=$(( a > 5 ))
test_mode=$(( b == "test" ))
limite_depasse=$(( c < 10 ))

if [[ $seuil_atteint && ( $test_mode || $limite_depasse ) ]]; then
    echo "Condition complexe mais lisible"
fi
```

**Gestion des ressources dans les boucles** :
```bash
# Pattern RAII (Resource Acquisition Is Initialization)
traiter_fichier_avec_ressources() {
    local fichier="$1"
    local temp_file
    local db_connection

    # Acquisition des ressources
    temp_file=$(mktemp)
    db_connection=$(ouvrir_connexion_base)

    # Gestionnaire de nettoyage
    cleanup() {
        rm -f "$temp_file"
        fermer_connexion_base "$db_connection"
    }

    trap cleanup EXIT

    # Traitement principal
    while IFS= read -r ligne; do
        # Utilisation des ressources
        traiter_ligne "$ligne" "$temp_file" "$db_connection"
    done < "$fichier"

    # Le nettoyage se fait automatiquement via trap
}
```

## Conclusion

Les conditions et boucles complexes transforment Bash d'un simple interprète de commandes en un langage de programmation complet. En maîtrisant les structures conditionnelles avancées, les boucles imbriquées, et les patterns de contrôle de flux, vous pouvez créer des scripts qui s'adaptent intelligemment aux situations complexes et traitent efficacement des volumes importants de données.

Comme un chef d'orchestre qui coordonne harmonieusement les différents instruments, le programmeur Bash expérimenté utilise chaque structure de contrôle à bon escient, créant des symphonies de traitement de données qui résolvent des problèmes complexes avec élégance et efficacité.

Dans le chapitre suivant, nous explorerons les fonctions et scripts modulaires, découvrant comment décomposer les programmes complexes en composants réutilisables et maintenables.
