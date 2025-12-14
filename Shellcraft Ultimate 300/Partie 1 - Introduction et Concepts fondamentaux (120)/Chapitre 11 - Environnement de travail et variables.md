# Chapitre 11 - Environnement de travail et variables

## Table des matières
- [Introduction](#introduction)
- [Qu'est-ce que l'environnement de travail ?](#quest-ce-que-lenvironnement-de-travail-)
- [Variables d'environnement : définition et portée](#variables-denvironnement--définition-et-portée)
- [Variables système essentielles](#variables-système-essentielles)
- [Variables utilisateur personnalisées](#variables-utilisateur-personnalisées)
- [Gestion des chemins (PATH)](#gestion-des-chemins-path)
- [Variables locales vs globales](#variables-locales-vs-globales)
- [Exportation et héritage](#exportation-et-héritage)
- [Commandes de manipulation des variables](#commandes-de-manipulation-des-variables)
- [Persistance des variables](#persistance-des-variables)
- [Débogage et inspection](#débogage-et-inspection)
- [Bonnes pratiques](#bonnes-pratiques)
- [Conclusion](#conclusion)

## Introduction

L'environnement de travail représente l'ensemble des paramètres et variables qui définissent le contexte d'exécution de vos commandes. Comme un environnement physique influence notre comportement et nos capacités, l'environnement shell détermine quelles commandes sont disponibles, comment elles se comportent, et quelles informations elles peuvent utiliser.

Imaginez l'environnement shell comme le cockpit d'un avion : les variables sont les instruments de bord, les chemins sont les cartes de navigation, et les paramètres système sont les conditions météorologiques. Un pilote maîtrisant parfaitement son cockpit peut voler avec précision et sécurité, quelles que soient les conditions.

## Qu'est-ce que l'environnement de travail ?

### Définition conceptuelle

**L'environnement de travail** (environment) est un ensemble de paires clé-valeur qui influencent le comportement des processus. Chaque processus hérite de l'environnement de son parent, créant une chaîne de transmission d'informations.

### Analogie avec la biologie

Considérez l'environnement comme l'ADN d'un processus :
- **Variables d'environnement** = Gènes exprimés
- **Héritage** = Transmission génétique
- **Mutation** = Modification de variables
- **Expression** = Utilisation des valeurs

### Portée et visibilité

**Trois niveaux de portée** :
1. **Global** : Visible par tous les processus
2. **Session** : Limité à la session shell courante
3. **Local** : Restreint à la fonction ou commande

## Variables d'environnement : définition et portée

### Syntaxe de base

**Définition simple** :
```bash
# Syntaxe générale
NOM_DE_VARIABLE=valeur

# Exemples concrets
NOM="Alice Dupont"
AGE=25
CHEMIN=/home/user/documents
```

**Règles de nommage** :
- Lettres majuscules par convention
- Caractères alphanumériques et underscore
- Pas d'espaces autour du `=`
- Sensibilité à la casse

### Types de variables

**Par leur origine** :
- **Système** : Définies par le système d'exploitation
- **Utilisateur** : Définies par l'utilisateur
- **Shell** : Spécifiques au shell utilisé
- **Programme** : Définies par les applications

**Par leur portée** :
- **Globales** : Disponibles partout
- **Locales** : Limitées au scope courant
- **Temporaires** : Durée de vie limitée

## Variables système essentielles

### Variables utilisateur et identité

**USER et LOGNAME** :
```bash
echo "Utilisateur: $USER"
echo "Login: $LOGNAME"
# Output: Utilisateur: alice
```

**HOME** :
```bash
echo "Répertoire personnel: $HOME"
cd "$HOME"  # Équivalent à cd ~
```

**UID et GID** :
```bash
echo "User ID: $UID"
echo "Group ID: $GID"
# Output: User ID: 1000
```

### Variables de localisation

**LANG et LC_*** :
```bash
echo "Langue système: $LANG"
echo "Format des nombres: $LC_NUMERIC"
echo "Format de la monnaie: $LC_MONETARY"
echo "Format de la date: $LC_TIME"
```

**TZ (fuseau horaire)** :
```bash
echo "Fuseau horaire: $TZ"
date  # Utilise TZ pour l'affichage
```

### Variables système générales

**HOSTNAME** :
```bash
echo "Nom de l'hôte: $HOSTNAME"
hostname  # Commande équivalente
```

**SHELL** :
```bash
echo "Shell par défaut: $SHELL"
echo "Shell courant: $0"
```

## Variables utilisateur personnalisées

### Création de variables personnalisées

**Variables de projet** :
```bash
# Configuration de projet
PROJET_NOM="MonApplication"
PROJET_VERSION="1.2.3"
PROJET_AUTEUR="Alice Dupont"
PROJET_EMAIL="alice@example.com"
```

**Chemins personnalisés** :
```bash
# Répertoires importants
WORKSPACE="$HOME/workspace"
PROJECTS="$WORKSPACE/projects"
TOOLS="$WORKSPACE/tools"
BACKUPS="$WORKSPACE/backups"
```

**Configuration d'outils** :
```bash
# Éditeur préféré
EDITOR="code --wait"  # VS Code
VISUAL="$EDITOR"

# Navigateur
BROWSER="firefox"

# Pager pour la visualisation
PAGER="less -R"
```

### Variables complexes

**Variables multi-lignes** :
```bash
# Configuration complexe
CONFIG="
[database]
host=localhost
port=5432
name=myapp_db

[server]
host=0.0.0.0
port=8000
workers=4
"
```

**Tableaux dans les variables** :
```bash
# Liste de serveurs (Bash 4+)
SERVERS=("web1.example.com" "web2.example.com" "db.example.com")

# Accès aux éléments
echo "Premier serveur: ${SERVERS[0]}"
echo "Tous les serveurs: ${SERVERS[*]}"
```

## Gestion des chemins (PATH)

### Structure du PATH

**Anatomie d'une variable PATH** :
```bash
echo "$PATH"
# Output: /usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin

# Format: répertoire1:réppertoire2:réppertoire3
```

**Ordre d'importance** :
1. **Chemins système** (`/usr/bin`, `/bin`)
2. **Chemins locaux** (`/usr/local/bin`)
3. **Chemins utilisateur** (`$HOME/bin`, `$HOME/.local/bin`)

### Manipulation du PATH

**Ajout de chemins** :
```bash
# Ajout temporaire (session)
export PATH="/usr/local/go/bin:$PATH"

# Ajout permanent (dans ~/.bashrc)
echo 'export PATH="/usr/local/go/bin:$PATH"' >> ~/.bashrc
```

**Gestion de chemins utilisateur** :
```bash
# Créer répertoire personnel pour binaires
mkdir -p "$HOME/bin"
mkdir -p "$HOME/.local/bin"

# Ajouter au PATH
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"
```

**Vérification et débogage** :
```bash
# Vérifier si un chemin est dans PATH
echo "$PATH" | tr ':' '\n' | grep "/usr/local/bin"

# Lister les exécutables disponibles
which ls
whereis ls
type ls
```

### Chemins spécialisés

**PATH pour scripts personnels** :
```bash
# Structure recommandée
~/.local/bin/     # Scripts personnels
~/bin/           # Raccourcis personnels
~/scripts/       # Scripts de projet

# Configuration
export PATH="$HOME/.local/bin:$HOME/bin:$HOME/scripts:$PATH"
```

**PATH conditionnel** :
```bash
# N'ajouter que si le répertoire existe
[ -d "$HOME/bin" ] && export PATH="$HOME/bin:$PATH"
[ -d "/opt/node/bin" ] && export PATH="/opt/node/bin:$PATH"
```

## Variables locales vs globales

### Variables locales

**Définition et utilisation** :
```bash
# Variable locale (visible seulement dans ce shell)
LOCAL_VAR="valeur locale"
echo "$LOCAL_VAR"

# Dans un sous-shell, elle n'existe pas
(bash -c 'echo "Sous-shell: $LOCAL_VAR"')  # Vide
```

**Variables dans les fonctions** :
```bash
ma_fonction() {
    local variable_locale="visible seulement ici"
    echo "$variable_locale"
}

ma_fonction
echo "$variable_locale"  # Non définie en dehors
```

### Variables globales (exportées)

**Exportation explicite** :
```bash
# Définition et exportation
GLOBAL_VAR="valeur globale"
export GLOBAL_VAR

# Ou en une seule commande
export GLOBAL_VAR2="valeur globale 2"
```

**Héritage par les processus enfants** :
```bash
export PARENT_VAR="du parent"

# Le processus enfant hérite
bash -c 'echo "Enfant: $PARENT_VAR"'

# Même pour les commandes externes
env | grep PARENT_VAR
```

## Exportation et héritage

### Mécanismes d'exportation

**export commande** :
```bash
# Exportation d'une variable existante
EXISTING_VAR="valeur"
export EXISTING_VAR

# Création et exportation simultanée
export NEW_VAR="nouvelle valeur"
```

**declare -x** :
```bash
# Equivalent à export
declare -x DECLARED_VAR="valeur déclarée"
```

### Héritage inter-processus

**Chaîne d'héritage** :
```bash
# Shell parent
export LEVEL1="niveau 1"

# Shell enfant
(export LEVEL2="niveau 2"
 # Sous-shell
 (export LEVEL3="niveau 3"
  env | grep LEVEL
 )
)
```

**Visualisation de l'héritage** :
```bash
# Arbre des processus avec variables
pstree -p $$
ps -eo pid,ppid,cmd | head -10
```

## Commandes de manipulation des variables

### Affichage des variables

**echo et printf** :
```bash
# Affichage simple
echo "$VARIABLE"

# Affichage avec formatage
printf "Valeur: %s\n" "$VARIABLE"
```

**env et printenv** :
```bash
# Toutes les variables exportées
env

# Variable spécifique
printenv PATH
env | grep USER
```

### set et declare

**set : variables du shell** :
```bash
# Toutes les variables (locales + globales)
set | grep VARIABLE

# Options du shell
set -o  # Afficher les options
set +o  # Format alternatif
```

**declare : inspection détaillée** :
```bash
# Type et attributs des variables
declare -p VARIABLE

# Toutes les variables avec attributs
declare -p
```

### unset : suppression

**Suppression de variables** :
```bash
# Supprimer une variable
unset VARIABLE

# Supprimer une fonction
unset -f ma_fonction
```

## Persistance des variables

### Fichiers de configuration

**~/.profile (shell agnostique)** :
```bash
# ~/.profile
export EDITOR="vim"
export PATH="$HOME/bin:$PATH"
```

**~/.bashrc (Bash spécifique)** :
```bash
# ~/.bashrc
export HISTCONTROL=ignoredups
export PS1='\u@\h:\w\$ '
```

**~/.zshrc (Zsh spécifique)** :
```bash
# ~/.zshrc
export ZSH_THEME="robbyrussell"
plugins=(git docker)
```

### Chargement conditionnel

**Par système d'exploitation** :
```bash
# ~/.bashrc
case "$(uname -s)" in
    Linux*)     export OS="Linux";;
    Darwin*)    export OS="macOS";;
    CYGWIN*)    export OS="Cygwin";;
    MINGW*)     export OS="MinGW";;
    *)          export OS="Unknown";;
esac
```

**Par présence de commandes** :
```bash
# Si Docker est installé
command -v docker >/dev/null 2>&1 && export DOCKER_HOST="unix:///var/run/docker.sock"
```

### Variables dynamiques

**Variables calculées** :
```bash
# Date dynamique
export BACKUP_DATE=$(date +%Y%m%d)

# Chemin dynamique
export CURRENT_PROJECT="$HOME/projects/$(basename "$PWD")"
```

## Débogage et inspection

### Outils d'inspection

**Visualisation complète** :
```bash
# Toutes les variables exportées
env | sort

# Variables commençant par USER
env | grep ^USER

# Valeurs vides
env | grep '^[^=]*=$'
```

**Différences entre environnements** :
```bash
# Comparer avec sous-shell
env > /tmp/parent.env
(bash -c 'env') > /tmp/child.env
diff /tmp/parent.env /tmp/child.env
```

### Debugging avancé

**Mode verbose pour les variables** :
```bash
# Tracer l'expansion
set -x
echo "PATH: $PATH"
echo "HOME: $HOME"
set +x
```

**Validation des variables** :
```bash
# Vérifier si définie
[ -n "${VARIABLE:-}" ] && echo "Définie" || echo "Non définie"

# Valeur par défaut
echo "${VARIABLE:-valeur_par_defaut}"

# Vérifier et définir si manquant
: "${REQUIRED_VAR:?Variable requise manquante}"
```

## Bonnes pratiques

### Conventions de nommage

**Standards de nommage** :
```bash
# Préfixes pour éviter les conflits
MYAPP_CONFIG_FILE="/etc/myapp.conf"
MYAPP_LOG_LEVEL="INFO"
MYAPP_DATABASE_URL="postgresql://..."

# Constantes en majuscules
readonly PI=3.14159
readonly CONFIG_PATH="/etc/myapp"
```

### Sécurité des variables

**Éviter l'injection** :
```bash
# DANGER - injection possible
eval "$USER_INPUT"

# SÉCURISÉ - validation stricte
case "$USER_INPUT" in
    option1|option2|option3) eval "$USER_INPUT";;
    *) echo "Option invalide";;
esac
```

**Variables sensibles** :
```bash
# Ne pas exporter les mots de passe
PASSWORD="secret"
# Utiliser des fichiers ou des variables locales uniquement
```

### Maintenance

**Documentation des variables** :
```bash
# ~/.bashrc
# Configuration personnelle d'Alice Dupont
# Créé: 2024-01-15
# Dernière modification: 2024-01-20

# Variables d'environnement essentielles
export EDITOR="code --wait"  # VS Code avec attente
export BROWSER="firefox"     # Navigateur par défaut
export PAGER="less -R"       # Pager avec couleurs
```

**Nettoyage périodique** :
```bash
# Lister les variables personnalisées
env | grep -E '^(MYAPP_|PROJECT_|CUSTOM_)' | sort

# Sauvegarde des variables importantes
env | grep -E '^(EDITOR|BROWSER|PATH|LANG)' > ~/.env_backup
```

## Conclusion

L'environnement de travail représente l'interface invisible mais cruciale entre l'utilisateur et le système. Les variables d'environnement permettent de personnaliser le comportement des commandes, de transmettre des informations aux processus, et de créer des workflows adaptés à nos besoins spécifiques.

Maîtriser la gestion des variables - leur définition, leur portée, leur persistance - transforme un environnement générique en un espace de travail personnalisé et efficace. Cette maîtrise permet non seulement d'optimiser notre productivité quotidienne, mais aussi de créer des scripts robustes et portables.

Dans les chapitres suivants, nous explorerons les permissions système, qui contrôlent non seulement l'accès aux fichiers, mais aussi définissent les limites de ce que chaque processus peut accomplir dans l'environnement que nous venons de configurer.
