# Chapitre 75 - Scripts interactifs

## Table des matières
- [Introduction](#introduction)
- [Entrées utilisateur de base](#entrées-utilisateur-de-base)
- [Validation et nettoyage des entrées](#validation-et-nettoyage-des-entrées)
- [Interfaces utilisateur textuelles](#interfaces-utilisateur-textuelles)
- [Menus et sélections](#menus-et-sélections)
- [Barres de progression et indicateurs](#barres-de-progression-et-indicateurs)
- [Gestion des interruptions](#gestion-des-interruptions)
- [Interfaces avancées avec dialog](#interfaces-avancées-avec-dialog)
- [Conclusion](#conclusion)

## Introduction

Les scripts interactifs transforment les programmes Bash d'outils d'administration en véritables applications conviviales. Au-delà de l'exécution automatique, l'interactivité permet de guider l'utilisateur, de recueillir ses préférences, et d'adapter le comportement du script selon les besoins spécifiques.

Imaginez les scripts interactifs comme des assistants personnels numériques : ils posent les bonnes questions au bon moment, valident les réponses, offrent des choix clairs, et guident l'utilisateur à travers des processus complexes avec patience et précision.

## Entrées utilisateur de base

### Lecture avec read

**Syntaxe de base de read** :
```bash
# Lecture simple
echo "Quel est votre nom ?"
read nom
echo "Bonjour $nom !"

# Lecture avec prompt intégré
read -p "Quel âge avez-vous ? " age
echo "Vous avez $age ans."

# Lecture silencieuse (pour mots de passe)
read -s -p "Mot de passe : " password
echo  # Nouvelle ligne après la saisie silencieuse
echo "Mot de passe saisi (longueur : ${#password})"
```

**Options avancées de read** :
```bash
# Timeout pour éviter les blocages
if read -t 10 -p "Répondez en 10 secondes : " reponse; then
    echo "Réponse : $reponse"
else
    echo "Timeout - réponse par défaut utilisée"
    reponse="defaut"
fi

# Nombre maximum de caractères
read -n 1 -p "Appuyez sur une touche : " touche
echo "Vous avez appuyé sur : $touche"

# Lecture de plusieurs variables
echo "Entrez nom âge ville :"
read nom age ville
echo "Nom: $nom, Âge: $age, Ville: $ville"

# Lecture dans un tableau
echo "Entrez plusieurs valeurs (Ctrl+D pour finir) :"
readarray -t valeurs
echo "Valeurs saisies : ${valeurs[@]}"
```

**Gestion des erreurs de saisie** :
```bash
# Fonction de lecture sécurisée
lire_avec_validation() {
    local prompt="$1"
    local validation_func="$2"
    local max_tentatives="${3:-3}"
    local valeur=""

    for ((i=1; i<=max_tentatives; i++)); do
        read -p "$prompt (tentative $i/$max_tentatives) : " valeur

        if [[ -z "$valeur" ]]; then
            echo "Valeur vide, veuillez réessayer." >&2
            continue
        fi

        if [[ -n "$validation_func" ]] && ! "$validation_func" "$valeur"; then
            echo "Valeur invalide, veuillez réessayer." >&2
            continue
        fi

        echo "$valeur"
        return 0
    done

    echo "Nombre maximum de tentatives atteint." >&2
    return 1
}

# Fonctions de validation
valider_email() {
    [[ "$1" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]
}

valider_age() {
    [[ "$1" =~ ^[0-9]+$ ]] && (( $1 >= 0 && $1 <= 150 ))
}

# Utilisation
if email=$(lire_avec_validation "Email" valider_email); then
    echo "Email valide : $email"
fi

if age=$(lire_avec_validation "Âge" valider_age); then
    echo "Âge valide : $age"
fi
```

## Validation et nettoyage des entrées

### Nettoyage automatique des entrées

**Suppression des espaces et normalisation** :
```bash
# Fonction de nettoyage général
nettoyer_entree() {
    local entree="$1"

    # Supprimer les espaces au début et à la fin
    entree="${entree#"${entree%%[![:space:]]*}"}"
    entree="${entree%"${entree##*[![:space:]]}"}"

    # Remplacer les espaces multiples par un seul
    entree=$(echo "$entree" | tr -s '[:space:]' ' ')

    # Convertir en minuscules si demandé
    if [[ "${2:-}" == "minuscules" ]]; then
        entree=$(echo "$entree" | tr '[:upper:]' '[:lower:]')
    fi

    echo "$entree"
}

# Nettoyage des chemins
nettoyer_chemin() {
    local chemin="$1"

    # Supprimer les espaces autour
    chemin=$(nettoyer_entree "$chemin")

    # Résoudre les chemins relatifs
    if [[ "$chemin" != /* ]]; then
        chemin="$(pwd)/$chemin"
    fi

    # Normaliser le chemin
    chemin=$(realpath "$chemin" 2>/dev/null || echo "$chemin")

    echo "$chemin"
}

# Utilisation
read -p "Entrez un chemin : " chemin_brut
chemin_nettoye=$(nettoyer_chemin "$chemin_brut")
echo "Chemin normalisé : $chemin_nettoye"
```

**Validation de types de données** :
```bash
# Validateur générique avec types
valider_type() {
    local valeur="$1"
    local type="$2"

    case "$type" in
        entier)
            [[ "$valeur" =~ ^-?[0-9]+$ ]]
            ;;
        positif)
            [[ "$valeur" =~ ^[0-9]+$ ]] && (( valeur > 0 ))
            ;;
        decimal)
            [[ "$valeur" =~ ^-?[0-9]*\.?[0-9]+$ ]]
            ;;
        boolean)
            [[ "$valeur" =~ ^(true|false|oui|non|1|0)$ ]]
            ;;
        chemin)
            [[ -e "$valeur" ]]
            ;;
        fichier)
            [[ -f "$valeur" ]]
            ;;
        repertoire)
            [[ -d "$valeur" ]]
            ;;
        email)
            [[ "$valeur" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]
            ;;
        ip)
            [[ "$valeur" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] && {
                local ip=(${valeur//./ })
                [[ ${ip[0]} -le 255 && ${ip[1]} -le 255 && ${ip[2]} -le 255 && ${ip[3]} -le 255 ]]
            }
            ;;
        *)
            echo "Type de validation inconnu: $type" >&2
            return 1
            ;;
    esac
}

# Fonction de saisie avec validation de type
lire_avec_type() {
    local prompt="$1"
    local type="$2"
    local defaut="${3:-}"

    local valeur=""
    while [[ -z "$valeur" ]]; do
        if [[ -n "$defaut" ]]; then
            read -p "$prompt [$defaut] : " valeur
            valeur="${valeur:-$defaut}"
        else
            read -p "$prompt : " valeur
        fi

        if [[ -z "$valeur" ]]; then
            echo "Valeur requise." >&2
            continue
        fi

        if valider_type "$valeur" "$type"; then
            echo "$valeur"
            return 0
        else
            echo "Valeur invalide pour le type '$type'." >&2
            valeur=""
        fi
    done
}

# Utilisation
port=$(lire_avec_type "Port SSH" "positif" "22")
ip=$(lire_avec_type "Adresse IP" "ip")
email=$(lire_avec_type "Email" "email")
```

## Interfaces utilisateur textuelles

### Affichage formaté et couleurs

**Codes d'échappement ANSI** :
```bash
# Définition des couleurs
declare -r RESET='\033[0m'
declare -r BOLD='\033[1m'
declare -r RED='\033[31m'
declare -r GREEN='\033[32m'
declare -r YELLOW='\033[33m'
declare -r BLUE='\033[34m'
declare -r MAGENTA='\033[35m'
declare -r CYAN='\033[36m'

# Styles de texte
declare -r UNDERLINE='\033[4m'
declare -r BLINK='\033[5m'
declare -r REVERSE='\033[7m'

# Fonctions d'affichage coloré
info() {
    echo -e "${GREEN}ℹ${RESET} $1"
}

warning() {
    echo -e "${YELLOW}⚠${RESET} $1"
}

error() {
    echo -e "${RED}✗${RESET} $1"
}

success() {
    echo -e "${GREEN}✓${RESET} $1"
}

title() {
    echo -e "${BOLD}${BLUE}$1${RESET}"
    echo -e "${BLUE}$(printf '%.0s─' {1..${#1}})${RESET}"
}

# Utilisation
title "Installation de l'application"
info "Vérification des prérequis..."
warning "Le répertoire /tmp sera utilisé"
success "Prérequis validés"
```

**Barres de progression simples** :
```bash
# Barre de progression avec caractères
progress_bar() {
    local current="$1"
    local total="$2"
    local width="${3:-50}"
    local message="${4:-Progression}"

    # Calcul du pourcentage
    local percentage=$(( current * 100 / total ))
    local filled=$(( current * width / total ))
    local empty=$(( width - filled ))

    # Construction de la barre
    local bar=$(printf '%.0s█' $(seq 1 $filled))
    bar+=$(printf '%.0s░' $(seq 1 $empty))

    # Affichage
    printf '\r%s [%s] %d%%' "$message" "$bar" "$percentage"

    # Nouvelle ligne à la fin
    if (( current >= total )); then
        echo
    fi
}

# Simulation de progression
echo "Téléchargement en cours..."
for ((i=1; i<=100; i++)); do
    progress_bar "$i" 100 30 "Téléchargement"
    sleep 0.05
done

echo "Téléchargement terminé !"
```

### Tables et affichages structurés

**Affichage de tableaux** :
```bash
# Fonction d'affichage de tableau
afficher_tableau() {
    local -a headers=("$@")
    local -a data=()
    local max_lengths=()

    # Lecture des données depuis stdin
    while IFS= read -r line; do
        IFS='|' read -ra fields <<< "$line"
        data+=("${fields[@]}")

        # Calcul des longueurs maximales
        for ((i=0; i<${#fields[@]}; i++)); do
            local len=${#fields[i]}
            if (( i >= ${#max_lengths[@]} )) || (( len > max_lengths[i] )); then
                max_lengths[i]=$len
            fi
        done
    done

    # S'assurer que les headers ont aussi des longueurs appropriées
    for ((i=0; i<${#headers[@]}; i++)); do
        local len=${#headers[i]}
        if (( len > max_lengths[i] )); then
            max_lengths[i]=$len
        fi
    done

    # Fonction d'affichage d'une ligne
    afficher_ligne() {
        local -a fields=("$@")
        local separator="+"

        for ((i=0; i<${#fields[@]}; i++)); do
            printf "%s─%s─" "$separator" "$(printf '%.0s─' $(seq 1 $((max_lengths[i] + 2))))"
            separator="+"
        done
        echo "$separator"
    }

    afficher_valeurs() {
        local -a fields=("$@")
        local prefix="│"

        for ((i=0; i<${#fields[@]}; i++)); do
            printf "%s %-*s │" "$prefix" "${max_lengths[i]}" "${fields[i]}"
            prefix=""
        done
        echo
    }

    # Affichage du tableau
    afficher_ligne "${headers[@]}"
    afficher_valeurs "${headers[@]}"
    afficher_ligne "${headers[@]}"

    # Affichage des données
    for ((i=0; i<${#data[@]}; i+=${#headers[@]})); do
        afficher_valeurs "${data[@]:i:${#headers[@]}}"
    done

    afficher_ligne "${headers[@]}"
}

# Utilisation
echo "Nom|Âge|Ville
Alice Dupont|28|Paris
Bob Martin|34|Lyon
Charlie Brown|22|Marseille" | afficher_tableau "Nom" "Âge" "Ville"
```

## Menus et sélections

### Menus simples avec select

**Menu de base avec select** :
```bash
# Menu avec select
afficher_menu() {
    local titre="$1"
    shift
    local -a options=("$@")

    echo
    echo "=== $titre ==="
    echo

    select choix in "${options[@]}"; do
        if [[ -n "$choix" ]]; then
            echo "Vous avez choisi : $choix"
            return $REPLY
        else
            echo "Choix invalide. Veuillez réessayer."
        fi
    done
}

# Utilisation
options=("Installer" "Configurer" "Désinstaller" "Quitter")
afficher_menu "Gestionnaire de logiciels" "${options[@]}"
choix=$?

case $choix in
    1) installer ;;
    2) configurer ;;
    3) desinstaller ;;
    4) exit 0 ;;
esac
```

**Menu hiérarchique** :
```bash
# Menu principal
menu_principal() {
    while true; do
        echo
        echo "=== Menu Principal ==="
        echo "1. Gestion des utilisateurs"
        echo "2. Gestion des services"
        echo "3. Sauvegarde"
        echo "4. Monitoring"
        echo "5. Quitter"
        echo

        read -p "Votre choix : " choix

        case $choix in
            1) menu_utilisateurs ;;
            2) menu_services ;;
            3) menu_sauvegarde ;;
            4) menu_monitoring ;;
            5) echo "Au revoir !"; exit 0 ;;
            *) echo "Choix invalide." ;;
        esac
    done
}

# Sous-menu utilisateurs
menu_utilisateurs() {
    while true; do
        echo
        echo "=== Gestion des Utilisateurs ==="
        echo "1. Créer un utilisateur"
        echo "2. Supprimer un utilisateur"
        echo "3. Modifier un utilisateur"
        echo "4. Lister les utilisateurs"
        echo "5. Retour au menu principal"
        echo

        read -p "Votre choix : " choix

        case $choix in
            1) creer_utilisateur ;;
            2) supprimer_utilisateur ;;
            3) modifier_utilisateur ;;
            4) lister_utilisateurs ;;
            5) return ;;
            *) echo "Choix invalide." ;;
        esac
    done
}

# Démarrage
menu_principal
```

### Sélections avancées avec recherche

**Menu avec recherche/filtrage** :
```bash
# Menu avec recherche interactive
menu_avec_recherche() {
    local titre="$1"
    shift
    local -a items=("$@")
    local recherche=""

    while true; do
        clear
        echo "=== $titre ==="
        echo "Tapez pour rechercher (Enter pour valider, Ctrl+C pour quitter)"
        echo "Recherche : $recherche"
        echo

        # Filtrage des éléments
        local -a filtres=()
        for item in "${items[@]}"; do
            if [[ -z "$recherche" ]] || [[ "${item,,}" == *"${recherche,,}"* ]]; then
                filtres+=("$item")
            fi
        done

        # Affichage des éléments filtrés
        if (( ${#filtres[@]} == 0 )); then
            echo "Aucun résultat pour '$recherche'"
        else
            for ((i=0; i<${#filtres[@]}; i++)); do
                echo "$((i+1)). ${filtres[i]}"
            done
        fi

        echo
        read -n1 -p "Numéro ou recherche : " saisie

        if [[ "$saisie" =~ ^[0-9]+$ ]]; then
            # Sélection par numéro
            local index=$((saisie - 1))
            if (( index >= 0 && index < ${#filtres[@]} )); then
                echo -e "\nSélection : ${filtres[index]}"
                return "$index"
            fi
        elif [[ "$saisie" == "" ]]; then
            # Validation de la recherche
            if (( ${#filtres[@]} == 1 )); then
                echo -e "\nSélection : ${filtres[0]}"
                return 0
            fi
        else
            # Ajout à la recherche
            recherche+="$saisie"
        fi
    done
}

# Utilisation
services=("apache2" "nginx" "mysql" "postgresql" "redis" "mongodb")
menu_avec_recherche "Sélection du service" "${services[@]}"
selection=$?
echo "Service sélectionné : ${services[selection]}"
```

## Barres de progression et indicateurs

### Barres de progression avancées

**Barre de progression avec ETA** :
```bash
# Barre de progression avec estimation du temps restant
progress_bar_eta() {
    local current="$1"
    local total="$2"
    local start_time="$3"
    local width="${4:-50}"

    # Calcul du pourcentage
    local percentage=$(( current * 100 / total ))
    local filled=$(( current * width / total ))
    local empty=$(( width - filled ))

    # Construction de la barre
    local bar=""
    bar+=$(printf '%.0s█' $(seq 1 $filled))
    bar+=$(printf '%.0s░' $(seq 1 $empty))

    # Calcul du temps écoulé et estimé
    local current_time=$(date +%s)
    local elapsed=$(( current_time - start_time ))
    local eta="?"

    if (( current > 0 )); then
        local total_estimated=$(( elapsed * total / current ))
        local remaining=$(( total_estimated - elapsed ))

        if (( remaining > 0 )); then
            local remaining_min=$(( remaining / 60 ))
            local remaining_sec=$(( remaining % 60 ))
            eta=$(printf "%02d:%02d" $remaining_min $remaining_sec)
        else
            eta="00:00"
        fi
    fi

    # Affichage
    printf '\r[%s] %d%% ETA: %s' "$bar" "$percentage" "$eta"

    if (( current >= total )); then
        echo
    fi
}

# Utilisation avec simulation
echo "Traitement des fichiers..."
start_time=$(date +%s)
total_files=100

for ((i=1; i<=total_files; i++)); do
    progress_bar_eta "$i" "$total_files" "$start_time" 40

    # Simulation du travail
    sleep 0.1
done

echo "Traitement terminé !"
```

**Indicateurs de chargement multiples** :
```bash
# Indicateurs de chargement animés
spinner() {
    local pid=$1
    local message="$2"
    local -a spinner=('⠋' '⠙' '⠹' '⠸' '⠼' '⠴' '⠦' '⠧' '⠇' '⠏')

    while kill -0 $pid 2>/dev/null; do
        for i in "${spinner[@]}"; do
            printf '\r%s %s' "$i" "$message"
            sleep 0.1
        done
    done

    printf '\r✓ %s\n' "$message"
}

pulse() {
    local pid=$1
    local message="$2"

    while kill -0 $pid 2>/dev/null; do
        printf '\r%s   ' "$message"
        sleep 0.3
        printf '\r%s . ' "$message"
        sleep 0.3
        printf '\r%s ..' "$message"
        sleep 0.3
    done

    printf '\r✓ %s\n' "$message"
}

# Utilisation
long_operation() {
    sleep 5
    echo "Opération terminée"
}

echo "Démarrage d'une longue opération..."
long_operation &
spinner $! "Traitement en cours"

echo "Démarrage d'une autre opération..."
long_operation &
pulse $! "Chargement des données"
```

## Gestion des interruptions

### Gestion des signaux

**Gestion propre des interruptions** :
```bash
# Gestionnaire d'interruption
cleanup() {
    echo -e "\n\nInterruption détectée !"

    # Nettoyage des ressources temporaires
    if [[ -n "${temp_files[*]}" ]]; then
        echo "Suppression des fichiers temporaires..."
        rm -f "${temp_files[@]}"
    fi

    # Arrêt des processus en arrière-plan
    if [[ -n "${bg_pids[*]}" ]]; then
        echo "Arrêt des processus en arrière-plan..."
        kill "${bg_pids[@]}" 2>/dev/null
    fi

    # Sauvegarde de l'état si nécessaire
    if [[ -n "$etat_fichier" ]]; then
        echo "Sauvegarde de l'état..."
        sauvegarder_etat "$etat_fichier"
    fi

    echo "Nettoyage terminé. Au revoir !"
    exit 1
}

# Configuration des gestionnaires de signaux
trap cleanup INT TERM HUP

# Variables de suivi
temp_files=()
bg_pids=()
etat_fichier="/tmp/mon_script.etat"

# Fonction utilitaire pour le suivi
ajouter_temp_file() {
    temp_files+=("$1")
}

ajouter_bg_pid() {
    bg_pids+=("$1")
}

# Script principal
echo "Script interactif (Ctrl+C pour quitter)"
echo "Création de fichiers temporaires..."

# Création de fichiers temporaires
for i in {1..3}; do
    temp_file=$(mktemp)
    ajouter_temp_file "$temp_file"
    echo "Créé : $temp_file"
done

# Lancement de processus en arrière-plan
sleep 30 &
ajouter_bg_pid $!

echo "Processus en cours... Appuyez sur Ctrl+C pour arrêter proprement."

# Boucle principale
while true; do
    read -p "Commande (ou 'quit') : " cmd
    case "$cmd" in
        quit) break ;;
        *) echo "Commande exécutée : $cmd" ;;
    esac
done

# Nettoyage normal
cleanup
```

**Confirmation d'interruption** :
```bash
# Gestionnaire avec confirmation
interrupt_handler() {
    echo -e "\n\nInterruption détectée !"
    read -p "Voulez-vous vraiment quitter ? (o/N) : " -n1 reponse
    echo

    case "${reponse,,}" in
        o|oui|y|yes)
            echo "Arrêt confirmé."
            cleanup
            exit 0
            ;;
        *)
            echo "Reprise du script..."
            return
            ;;
    esac
}

# Configuration du gestionnaire
trap interrupt_handler INT

# Script avec possibilité d'interruption contrôlée
echo "Script avec interruption contrôlée"
echo "Tapez Ctrl+C pour confirmation d'arrêt"

while true; do
    read -p "> " commande
    case "$commande" in
        quit) break ;;
        *) eval "$commande" ;;
    esac
done
```

## Interfaces avancées avec dialog

### Installation et utilisation de base

**Vérification et installation de dialog** :
```bash
# Vérification de l'installation
if ! command -v dialog >/dev/null 2>&1; then
    echo "dialog n'est pas installé."

    # Installation selon le système
    if command -v apt >/dev/null 2>&1; then
        echo "Installation avec apt..."
        sudo apt update && sudo apt install -y dialog
    elif command -v yum >/dev/null 2>&1; then
        echo "Installation avec yum..."
        sudo yum install -y dialog
    else
        echo "Veuillez installer dialog manuellement."
        exit 1
    fi
fi

echo "dialog est prêt à être utilisé."
```

**Boîte de message simple** :
```bash
# Boîte de message
dialog --title "Information" \
       --msgbox "Ceci est un message d'information affiché dans une boîte de dialogue." 10 50

# Avec gestion du code de retour
if dialog --title "Confirmation" \
         --yesno "Voulez-vous continuer ?" 10 50; then
    echo "Utilisateur a répondu OUI"
else
    echo "Utilisateur a répondu NON"
fi
```

### Boîtes de saisie avancées

**Saisie de texte** :
```bash
# Saisie simple
nom=$(dialog --title "Saisie du nom" \
             --inputbox "Entrez votre nom :" 10 50 \
             3>&1 1>&2 2>&3)

# Vérification du code de retour
if [[ $? -eq 0 ]]; then
    echo "Nom saisi : $nom"
else
    echo "Saisie annulée"
fi

# Saisie avec valeur par défaut
email=$(dialog --title "Adresse email" \
               --inputbox "Entrez votre adresse email :" 10 50 "user@example.com" \
               3>&1 1>&2 2>&3)

# Saisie de mot de passe
mot_de_passe=$(dialog --title "Mot de passe" \
                      --passwordbox "Entrez votre mot de passe :" 10 50 \
                      3>&1 1>&2 2>&3)
```

**Menus déroulants** :
```bash
# Menu de sélection
choix=$(dialog --title "Menu principal" \
               --menu "Sélectionnez une option :" 15 50 5 \
               1 "Installer l'application" \
               2 "Configurer les paramètres" \
               3 "Mettre à jour" \
               4 "Désinstaller" \
               5 "Quitter" \
               3>&1 1>&2 2>&3)

case $choix in
    1) installer ;;
    2) configurer ;;
    3) mettre_a_jour ;;
    4) desinstaller ;;
    5) exit 0 ;;
    *) echo "Annulé" ;;
esac

# Liste à choix multiples (checklist)
services_selectionnes=$(dialog --title "Sélection des services" \
                                --checklist "Cochez les services à installer :" 15 50 5 \
                                apache "Serveur web Apache" on \
                                mysql "Base de données MySQL" on \
                                php "Interpréteur PHP" off \
                                nginx "Serveur web Nginx" off \
                                3>&1 1>&2 2>&3)

echo "Services sélectionnés : $services_selectionnes"
```

### Interfaces complexes

**Boîte de progression** :
```bash
# Fonction de progression avec dialog
progress_with_dialog() {
    local total="$1"
    local message="${2:-Traitement en cours}"

    # Création d'un pipe nommé pour la communication
    local pipe_file=$(mktemp)
    rm "$pipe_file"
    mkfifo "$pipe_file"

    # Lancement de dialog en arrière-plan
    dialog --title "Progression" \
           --gauge "$message" 10 50 0 < "$pipe_file" &
    local dialog_pid=$!

    # Simulation de progression
    for ((i=0; i<=total; i++)); do
        # Calcul du pourcentage
        local percentage=$(( i * 100 / total ))

        # Envoi à dialog
        echo "$percentage" > "$pipe_file"

        # Simulation du travail
        sleep 0.1
    done

    # Fermeture
    echo "100" > "$pipe_file"
    sleep 1

    # Nettoyage
    rm -f "$pipe_file"
    kill $dialog_pid 2>/dev/null
}

# Utilisation
progress_with_dialog 100 "Installation des composants"
```

**Formulaires complexes** :
```bash
# Formulaire d'inscription
resultat=$(dialog --title "Formulaire d'inscription" \
                  --form "Remplissez le formulaire :" 15 50 0 \
                  "Nom :" 1 1 "" 1 15 30 0 \
                  "Prénom :" 2 1 "" 2 15 30 0 \
                  "Email :" 3 1 "" 3 15 30 0 \
                  "Téléphone :" 4 1 "" 4 15 30 0 \
                  3>&1 1>&2 2>&3)

# Traitement du résultat
if [[ $? -eq 0 ]]; then
    # Parsing des valeurs (séparées par des sauts de ligne)
    IFS=$'\n' read -d '' -ra valeurs <<< "$resultat"

    nom="${valeurs[0]}"
    prenom="${valeurs[1]}"
    email="${valeurs[2]}"
    telephone="${valeurs[3]}"

    echo "Inscription de $prenom $nom"
    echo "Email : $email"
    echo "Téléphone : $telephone"
else
    echo "Formulaire annulé"
fi
```

## Conclusion

Les scripts interactifs transforment les outils d'administration en véritables applications conviviales. En maîtrisant la lecture d'entrées utilisateur, la validation de données, les interfaces colorées, les menus sophistiqués, et les boîtes de dialogue avancées, vous pouvez créer des scripts qui guident efficacement les utilisateurs à travers des processus complexes.

Comme un bon interface homme-machine, un script interactif bien conçu réduit l'erreur humaine, améliore l'expérience utilisateur, et rend les tâches complexes accessibles à tous les niveaux de compétence.

Dans le chapitre suivant, nous explorerons les pipes, redirections et flux multiples, découvrant comment orchestrer des traitements de données complexes en combinant intelligemment les commandes et les processus.
