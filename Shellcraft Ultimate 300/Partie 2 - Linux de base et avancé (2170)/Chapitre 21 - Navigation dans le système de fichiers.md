# Chapitre 21 - Navigation dans le système de fichiers

## Table des matières
- [Introduction](#introduction)
- [Structure du système de fichiers Linux](#structure-du-système-de-fichiers-linux)
- [Commandes de navigation essentielles](#commandes-de-navigation-essentielles)
- [Chemins absolus vs chemins relatifs](#chemins-absolus-vs-chemins-relatifs)
- [Raccourcis et navigation intelligente](#raccourcis-et-navigation-intelligente)
- [Exploration des répertoires](#exploration-des-répertoires)
- [Navigation avancée avec pushd/popd](#navigation-avancée-avec-pushdpopd)
- [Marque-pages et téléportation](#marque-pages-et-téléportation)
- [Navigation conditionnelle](#navigation-conditionnelle)
- [Outils de visualisation graphique](#outils-de-visualisation-graphique)
- [Dépannage de la navigation](#dépannage-de-la-navigation)
- [Bonnes pratiques](#bonnes-pratiques)
- [Conclusion](#conclusion)

## Introduction

La navigation dans le système de fichiers constitue l'acte fondamental de l'interaction avec un système Linux. Comme un navigateur web qui vous permet de voyager entre les pages, les commandes de navigation vous permettent d'explorer l'arborescence complexe du système de fichiers, de localiser des fichiers spécifiques, et de vous déplacer efficacement entre différents contextes de travail.

Imaginez le système de fichiers comme une immense bibliothèque : chaque répertoire est une salle, chaque fichier un livre sur les étagères. La navigation efficace vous permet non seulement de trouver rapidement ce que vous cherchez, mais aussi de développer une intuition spatiale qui transforme l'administration système d'une corvée en une exploration fluide.

## Structure du système de fichiers Linux

### Hiérarchie standard du système de fichiers

**La racine (/) et ses enfants principaux** :

```
/ (racine)
├── bin/          # Commandes essentielles
├── boot/         # Fichiers de démarrage
├── dev/          # Périphériques
├── etc/          # Configuration système
├── home/         # Répertoires utilisateurs
├── lib/          # Bibliothèques partagées
├── media/        # Points de montage médias
├── mnt/          # Points de montage temporaires
├── opt/          # Logiciels optionnels
├── proc/         # Informations système
├── root/         # Répertoire de root
├── run/          # Données runtime
├── sbin/         # Commandes système
├── srv/          # Données de services
├── sys/          # Informations matérielles
├── tmp/          # Fichiers temporaires
├── usr/          # Hiérarchie secondaire
└── var/          # Données variables
```

**Répertoires utilisateur (~)** :
```bash
~ (home directory)
├── Desktop/      # Bureau
├── Documents/    # Documents
├── Downloads/    # Téléchargements
├── Music/        # Musique
├── Pictures/     # Images
├── Videos/       # Vidéos
├── .config/      # Configuration d'applications
├── .local/       # Données locales
└── .cache/       # Cache d'applications
```

### Philosophie de l'organisation

**Séparation des préoccupations** :
- **/usr** : Logiciels installés par le système
- **/var** : Données qui changent (logs, bases de données)
- **/tmp** : Fichiers temporaires (nettoyés au reboot)
- **/home** : Données personnelles des utilisateurs

**Liens symboliques stratégiques** :
```bash
# Compatibilité historique
/bin → /usr/bin
/sbin → /usr/sbin

# Organisation moderne
/lib → /usr/lib
```

## Commandes de navigation essentielles

### pwd : Où suis-je ?

**Affichage du répertoire courant** :
```bash
# Commande de base
pwd

# Avec options
pwd -P    # Résoudre les liens symboliques
pwd -L    # Garder les liens symboliques (défaut)

# Exemples
$ pwd
/home/alice

$ cd /usr/bin
$ pwd
/usr/bin

$ cd /bin  # /bin est un lien vers /usr/bin
$ pwd -P   # Montre la vraie destination
/usr/bin
```

**Utilisation en scripting** :
```bash
#!/bin/bash
# Script qui sait où il est exécuté
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
echo "Le script se trouve dans: $SCRIPT_DIR"

# Sauvegarde du répertoire courant
ORIGINAL_DIR="$(pwd)"
# ... faire du travail ...
cd "$ORIGINAL_DIR"  # Retour
```

### cd : Changer de répertoire

**Syntaxe de base** :
```bash
cd [répertoire]

# Exemples courants
cd /home/alice          # Chemin absolu
cd Documents            # Chemin relatif
cd ..                   # Répertoire parent
cd ~                    # Répertoire personnel
cd -                    # Retour au répertoire précédent
```

**Options spéciales** :
```bash
cd              # Vers le répertoire personnel
cd ~            # Même chose explicitement
cd ~/Documents  # Sous-répertoire du home
cd -            # Bascule entre deux répertoires
```

**Comportement intelligent** :
```bash
# Si le répertoire n'existe pas
cd nonexistent
# bash: cd: nonexistent: No such file or directory

# Avec vérification
[ -d "$dir" ] && cd "$dir" || echo "Répertoire inexistant: $dir"
```

### ls : Lister le contenu

**Options essentielles** :
```bash
ls              # Liste simple
ls -l           # Format long (détails)
ls -a           # Inclure les fichiers cachés
ls -h           # Tailles human-readable
ls -t           # Tri par date de modification
ls -r           # Tri inversé
ls -S           # Tri par taille
```

**Combinaisons puissantes** :
```bash
# Vue d'ensemble complète
ls -lah

# Fichiers triés par taille décroissante
ls -lahS

# Fichiers modifiés récemment
ls -lat | head -10

# Comptage des éléments
ls -1 | wc -l
```

**Filtrage avancé** :
```bash
# Seulement les répertoires
ls -d */

# Seulement les fichiers exécutables
ls -l | grep "^-..x"

# Par type de fichier
ls -l *.txt      # Fichiers texte
ls -l *.sh       # Scripts shell
```

## Chemins absolus vs chemins relatifs

### Chemins absolus

**Définition** : Chemin complet depuis la racine
```bash
# Structure complète
/usr/local/bin/python3
/home/alice/Documents/rapport.pdf
/etc/systemd/system/sshd.service

# Avantages
- Indépendant du répertoire courant
- Précis et non ambigu
- Utilisable depuis n'importe où

# Inconvénients
- Verbeux à écrire
- Fragile aux changements de structure
```

### Chemins relatifs

**Définition** : Chemin relatif au répertoire courant
```bash
# Navigation relative
./script.sh          # Fichier dans le répertoire courant
../data/file.txt     # Fichier dans le répertoire parent
../../config.ini    # Deux niveaux plus haut
subdir/nested/file  # Dans un sous-répertoire

# Raccourcis spéciaux
.   # Répertoire courant
..  # Répertoire parent
~   # Répertoire personnel
-   # Répertoire précédent (avec cd)
```

### Résolution de chemins

**Expansion automatique** :
```bash
# Bash résout automatiquement
cd /usr/local
cd bin        # → /usr/local/bin
cd ../../etc  # → /etc

# Vérification de résolution
realpath chemin_relatif
readlink -f chemin_relatif
```

**Calcul mental de chemins** :
```
Position actuelle: /home/alice/projects/web
Cible souhaitée: /home/alice/documents/notes

Chemin relatif: ../../documents/notes
- .. : /home/alice/projects
- .. : /home/alice
- documents/notes : /home/alice/documents/notes
```

## Raccourcis et navigation intelligente

### Variables de répertoire

**Sauvegarde de chemins fréquents** :
```bash
# Dans ~/.bashrc
export PROJECTS="$HOME/projects"
export DOCS="$HOME/Documents"
export SCRIPTS="$HOME/bin"

# Utilisation
cd "$PROJECTS/web-app"
cd "$DOCS/notes"
```

**Variables dynamiques** :
```bash
# Répertoire du script courant
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# Répertoire du projet (remontée automatique)
PROJECT_ROOT="$(cd "$(dirname "${BASH_SOURCE[0]}")/../../../" && pwd)"
```

### Fonctions de navigation

**Fonctions personnalisées** :
```bash
# ~/.bashrc

# Aller au répertoire contenant un fichier
cdf() {
    local file="$1"
    local dir="$(dirname "$file")"
    cd "$dir"
}

# Chercher et aller dans un répertoire
cdfind() {
    local dir="$(find . -type d -name "*$1*" 2>/dev/null | head -1)"
    if [ -n "$dir" ]; then
        cd "$dir"
    else
        echo "Répertoire non trouvé: $1"
    fi
}

# Historique visuel des répertoires
cdh() {
    local dir="$(dirs -v | fzf | sed 's/^[0-9]*[[:space:]]*//')"
    cd "$dir"
}
```

### Tab completion avancée

**Complétion intelligente** :
```bash
# Chemins partiels
cd /usr/l<TAB>  # → /usr/local/
cd ~/Doc<TAB>   # → ~/Documents/

# Variables
cd $PROJ<TAB>   # → $PROJECTS/

# Historique
cd -<TAB>       # Liste des répertoires récents
```

## Exploration des répertoires

### tree : Vue arborescente

**Installation et utilisation** :
```bash
# Installation (Ubuntu/Debian)
sudo apt install tree

# Utilisation de base
tree                    # Arborescence complète
tree -d                 # Seulement les répertoires
tree -L 2               # Profondeur limitée
tree -h                 # Tailles humaines
```

**Options puissantes** :
```bash
# Filtrage
tree -P "*.txt"         # Seulement les .txt
tree -I "*.log"         # Exclure les .log

# Formatage
tree -f                 # Chemins complets
tree -s                 # Tailles des fichiers
tree -D                 # Dates de modification
```

### find avec exécution limitée

**Exploration contrôlée** :
```bash
# Lister récursivement (max profondeur)
find . -maxdepth 2 -type d

# Compter les fichiers par répertoire
find . -type d -exec sh -c 'echo -n "$1: " && find "$1" -maxdepth 1 -type f | wc -l' _ {} \;

# Arborescence simple en shell pur
find . -type d | sed 's|[^/]*/|- |g'
```

### Exploration interactive

**Navigation avec fzf** :
```bash
# ~/.bashrc
cdfzf() {
    local dir="$(find . -type d 2>/dev/null | fzf)"
    if [ -n "$dir" ]; then
        cd "$dir"
    fi
}

# Recherche de fichiers avec aperçu
ff() {
    local file="$(find . -type f 2>/dev/null | fzf --preview 'head -20 {}')"
    if [ -n "$file" ]; then
        ${EDITOR:-vim} "$file"
    fi
}
```

## Navigation avancée avec pushd/popd

### Pile de répertoires

**Principe de fonctionnement** :
```bash
# La pile mémorise les répertoires visités
dirs -v          # Afficher la pile
dirs -c          # Vider la pile

# Exemple d'utilisation
pwd              # /home/alice
pushd /tmp       # Va dans /tmp, mémorise /home/alice
pwd              # /tmp
pushd /var       # Va dans /var, mémorise /tmp
dirs -v          # Affiche la pile:
                 # 0  /var
                 # 1  /tmp
                 # 2  /home/alice
popd             # Retour à /tmp
popd             # Retour à /home/alice
```

### Commandes détaillées

**pushd : Empiler et changer** :
```bash
pushd répertoire         # Va dans répertoire, l'empile
pushd +1                 # Va au répertoire en position 1
pushd                    # Échange les deux premiers de la pile

# Exemples pratiques
pushd ~/projects         # Sauvegarde position actuelle
# ... travail ...
popd                     # Retour automatique
```

**popd : Dépiler et retourner** :
```bash
popd                     # Retour au répertoire précédent
popd +2                  # Enlève le répertoire en position 2

# Gestion d'erreurs
popd 2>/dev/null || echo "Pile vide"
```

### Applications pratiques

**Workflow de développement** :
```bash
# Configuration d'un environnement de dev
pushd ~/projects/myapp   # Projet principal
pushd src/               # Code source
pushd tests/             # Tests

# Navigation rapide
popd                     # Retour aux tests
popd                     # Retour au code source
popd                     # Retour au projet
```

**Script de sauvegarde avec pile** :
```bash
#!/bin/bash
# Script qui sauvegarde sa position
pushd "$HOME" >/dev/null

# Traitement dans différents répertoires
for dir in Documents Pictures projects; do
    if [ -d "$dir" ]; then
        pushd "$dir" >/dev/null
        echo "Sauvegarde de $(pwd)"
        # ... logique de sauvegarde ...
        popd >/dev/null
    fi
done

popd >/dev/null  # Retour à la position initiale
```

## Marque-pages et téléportation

### Variables de répertoire

**Marque-pages permanents** :
```bash
# ~/.bashrc
export WORKSPACE="$HOME/workspace"
export PROJECTS="$WORKSPACE/projects"
export DOTFILES="$WORKSPACE/dotfiles"
export SCRIPTS="$HOME/bin"

# Fonction de téléportation
tp() {
    case "$1" in
        work) cd "$WORKSPACE" ;;
        proj) cd "$PROJECTS" ;;
        dot) cd "$DOTFILES" ;;
        scr) cd "$SCRIPTS" ;;
        *) echo "Destinations: work, proj, dot, scr" ;;
    esac
}

# Utilisation
tp work    # Va dans ~/workspace
tp proj    # Va dans ~/workspace/projects
```

### Système de favoris

**Fichier de configuration** :
```bash
# ~/.nav_favorites
work:$HOME/workspace
proj:$HOME/workspace/projects
docs:$HOME/Documents
pics:$HOME/Pictures
conf:$HOME/.config
```

**Script de navigation par favoris** :
```bash
# ~/.bashrc
goto() {
    local dest="$(grep "^$1:" ~/.nav_favorites | cut -d: -f2-)"
    if [ -n "$dest" ]; then
        cd "$dest" && pwd
    else
        echo "Favori inconnu: $1"
        echo "Favoris disponibles:"
        cut -d: -f1 ~/.nav_favorites
    fi
}

# Liste des favoris
favorites() {
    echo "=== Favoris de navigation ==="
    while IFS=: read -r name path; do
        printf "%-10s %s\n" "$name" "$path"
    done < ~/.nav_favorites
}
```

### Historique intelligent

**Marque-pages automatiques** :
```bash
# ~/.bashrc
# Sauvegarde automatique des répertoires visités
export CD_HISTORY_FILE="$HOME/.cd_history"

cd() {
    builtin cd "$@" && pwd >> "$CD_HISTORY_FILE"
}

# Fonction de recherche dans l'historique
cdhist() {
    local dest="$(tail -20 "$CD_HISTORY_FILE" | sort | uniq | fzf)"
    if [ -n "$dest" ]; then
        cd "$dest"
    fi
}
```

## Navigation conditionnelle

### Tests et conditions

**Navigation sécurisée** :
```bash
# Vérifier avant de changer
smart_cd() {
    local target="$1"
    if [ -d "$target" ]; then
        cd "$target"
    elif [ -f "$target" ]; then
        cd "$(dirname "$target")"
    else
        echo "Cible inexistante: $target"
        return 1
    fi
}

# Avec complétion automatique des erreurs
cd() {
    builtin cd "$@" 2>/dev/null || {
        echo "Erreur: $(pwd) → $1"
        return 1
    }
}
```

### Navigation contextuelle

**Selon l'environnement** :
```bash
# Adapter la navigation selon le système
case "$(uname -s)" in
    Linux)
        export DESKTOP="$HOME/Desktop"
        ;;
    Darwin)
        export DESKTOP="$HOME/Desktop"
        ;;
    CYGWIN*|MINGW*)
        export DESKTOP="$HOME/Desktop"
        ;;
esac

# Navigation selon l'heure
morning_nav() {
    local hour=$(date +%H)
    if [ $hour -lt 12 ]; then
        cd "$PROJECTS"
        echo "🌅 Bonne journée de développement!"
    else
        cd "$DOCS"
        echo "📚 Après-midi d'étude"
    fi
}
```

## Outils de visualisation graphique

### ncdu : Analyseur d'espace disque

**Navigation visuelle de l'espace** :
```bash
# Installation
sudo apt install ncdu

# Analyse interactive
ncdu /home/alice

# Options
ncdu -x /        # Ne pas traverser les systèmes de fichiers
ncdu -q          # Mode silencieux
```

**Navigation dans ncdu** :
```bash
# Contrôles
↑↓ : Navigation
Entrée : Explorer
d : Supprimer
n : Trier par nom
s : Trier par taille
g : Trier par éléments
q : Quitter
```

### midnight commander (mc)

**Navigateur de fichiers en console** :
```bash
# Installation
sudo apt install mc

# Lancement
mc

# Fonctionnalités
- Panneau double (comme Norton Commander)
- Édition intégrée (F4)
- Visualisation (F3)
- Copie/ déplacement intuitifs
```

### ranger : Navigateur moderne

**Navigateur console Python** :
```bash
# Installation
pip install ranger-fm

# Lancement
ranger

# Fonctionnalités avancées
- Prévisualisation automatique
- Onglets multiples
- Marquage en lot
- Actions personnalisables
```

## Dépannage de la navigation

### Problèmes courants

**Répertoire inexistant** :
```bash
cd nonexistent
# bash: cd: nonexistent: No such file or directory

# Vérifications
ls -ld nonexistent  # Vérifier l'existence
file nonexistent    # Type du fichier
```

**Permissions insuffisantes** :
```bash
cd /root
# bash: cd: /root: Permission denied

# Vérifier les permissions
ls -ld /root
# drwx------ 1 root root ... /root

# Solution
sudo cd /root  # Ou utiliser su/sudo
```

**Liens symboliques cassés** :
```bash
cd broken_link
# bash: cd: broken_link: No such file or directory

# Diagnostic
ls -l broken_link
# lrwxrwxrwx 1 user user ... broken_link -> /nonexistent

# Résolution
unlink broken_link
```

### Outils de diagnostic

**Inspection du système de fichiers** :
```bash
# Monter les systèmes de fichiers
mount | grep -E "(ext4|xfs|btrfs)"

# Vérifier l'intégrité
df -h                  # Espace disponible
du -sh * 2>/dev/null   # Taille des répertoires

# Permissions détaillées
ls -la /path/to/problem
stat /path/to/file     # Métadonnées complètes
```

**Réparation de problèmes** :
```bash
# Créer des répertoires manquants
sudo mkdir -p /path/to/directory

# Corriger les permissions
sudo chown -R user:group /path/to/fix

# Recréer des liens cassés
ln -s /real/path /broken/link
```

## Bonnes pratiques

### Organisation personnelle

**Structure recommandée** :
```bash
# Répertoire personnel bien organisé
~/projects/           # Tous les projets
    ├── personal/     # Projets personnels
    ├── work/         # Projets professionnels
    └── open-source/  # Contributions

~/workspace/          # Espace de travail temporaire
~/bin/               # Scripts personnels
~/docs/              # Documentation
~/.config/           # Configurations
```

### Raccourcis efficaces

**Configuration optimale** :
```bash
# ~/.bashrc - Navigation optimisée

# Variables de chemin
export PROJECTS="$HOME/projects"
export WORKSPACE="$HOME/workspace"
export SCRIPTS="$HOME/bin"

# Alias intelligents
alias ..='cd ..'
alias ...='cd ../..'
alias ....='cd ../../..'
alias cd..='cd ..'  # Pour les habitudes Windows

# Fonctions de navigation
cdf() { cd "$(dirname "$1")"; }  # Aller au répertoire d'un fichier
cdr() { cd "$(git rev-parse --show-toplevel 2>/dev/null || pwd)"; }  # Racine Git

# Complétion améliorée
bind 'set mark-symlinked-directories on'
bind 'set completion-ignore-case on'

# Prompt informatif
PS1='\u@\h:\w\$ '
```

### Sécurité et prudence

**Navigation sécurisée** :
```bash
# Toujours vérifier avant d'agir
ls -la /path/before/cd

# Utiliser des chemins absolus dans les scripts
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$SCRIPT_DIR/relative/path"

# Éviter les chemins avec espaces non quotés
cd "/path/with spaces"  # Correct
cd /path/with spaces    # Incorrect
```

### Performance

**Optimisations** :
```bash
# Précharger les chemins fréquents
# (dans ~/.bashrc pour un chargement rapide)
export -a FAST_PATHS=(
    "$HOME"
    "$HOME/projects"
    "$HOME/Documents"
    "/etc"
    "/var/log"
)

# Fonction de navigation rapide
fast_cd() {
    local target="$1"
    for path in "${FAST_PATHS[@]}"; do
        if [[ "$path" == *"$target"* ]]; then
            cd "$path"
            return
        fi
    done
    cd "$target"  # Fallback normal
}
```

## Conclusion

La navigation dans le système de fichiers Linux est bien plus qu'une compétence technique de base ; c'est l'acquisition d'une intuition spatiale qui transforme votre efficacité au terminal. Des commandes simples comme `cd` et `ls` aux techniques avancées comme `pushd/popd` et la navigation conditionnelle, chaque outil contribue à une expérience fluide et productive.

Maîtriser ces techniques revient à développer une mémoire musculaire numérique : au lieu de penser consciemment à chaque commande, vous naviguez instinctivement, permettant à votre attention de se concentrer sur les tâches importantes plutôt que sur la mécanique de base.

Dans le chapitre suivant, nous explorerons la gestion des fichiers et dossiers, apprenant à créer, modifier et organiser vos données de manière efficace et sécurisée.

