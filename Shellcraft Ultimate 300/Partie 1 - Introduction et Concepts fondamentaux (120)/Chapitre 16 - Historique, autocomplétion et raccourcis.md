# Chapitre 16 - Historique, autocomplétion et raccourcis

## Table des matières
- [Introduction](#introduction)
- [L'historique des commandes](#lhistorique-des-commandes)
- [Navigation dans l'historique](#navigation-dans-lhistorique)
- [Recherche dans l'historique](#recherche-dans-lhistorique)
- [Autocomplétion intelligente](#autocomplétion-intelligente)
- [Raccourcis clavier essentiels](#raccourcis-clavier-essentiels)
- [Personnalisation de l'autocomplétion](#personnalisation-de-lautocomplétion)
- [Historique partagé et persistant](#historique-partagé-et-persistant)
- [Raccourcis avancés](#raccourcis-avancés)
- [Macros et snippets](#macros-et-snippets)
- [Outils complémentaires](#outils-complémentaires)
- [Conclusion](#conclusion)

## Introduction

L'historique, l'autocomplétion et les raccourcis représentent les mécanismes d'accélération qui transforment un utilisateur novice en expert du terminal. Ces fonctionnalités réduisent drastiquement la frappe répétitive tout en améliorant la précision et la vitesse d'exécution.

Imaginez ces outils comme le système de mémoire et d'anticipation d'un pianiste virtuose : l'historique conserve les mélodies passées, l'autocomplétion suggère les notes suivantes, et les raccourcis permettent d'exécuter des séquences complexes d'un simple geste. Ensemble, ils créent un environnement où la pensée précède l'action, et où l'efficacité devient instinctive.

## L'historique des commandes

### Principe de fonctionnement

**Mécanisme de stockage** :
- Chaque commande saisie est enregistrée dans un fichier d'historique
- Par défaut : `~/.bash_history` (Bash) ou `~/.zsh_history` (Zsh)
- Taille configurable (par défaut 1000 commandes)

**Contenu d'une entrée d'historique** :
```
# Numéro de ligne
# Timestamp (optionnel)
# Commande exécutée
```

### Configuration de l'historique

**Variables de configuration Bash** :
```bash
# ~/.bashrc
HISTSIZE=10000        # Nombre de commandes en mémoire
HISTFILESIZE=20000    # Nombre de commandes dans le fichier
HISTCONTROL=ignoredups:ignorespace  # Contrôler l'historique
HISTIGNORE="ls:cd:exit"  # Commandes à ignorer
```

**Configuration Zsh** :
```bash
# ~/.zshrc
HISTSIZE=10000
SAVEHIST=10000
setopt hist_ignore_dups     # Ignorer les doublons
setopt hist_ignore_space    # Ignorer les commandes commençant par espace
setopt share_history        # Historique partagé entre sessions
```

### Gestion du fichier d'historique

**Visualisation** :
```bash
# Dernières commandes
history | tail -10

# Recherche dans l'historique
history | grep "git"

# Numéros de ligne
history 5  # Dernières 5 commandes avec numéros
```

**Maintenance** :
```bash
# Supprimer une commande spécifique
history -d 1234

# Effacer tout l'historique
history -c
history -w  # Sauvegarder les changements

# Recharger depuis le fichier
history -r
```

## Navigation dans l'historique

### Raccourcis de base

**Flèches directionnelles** :
```bash
# Commande précédente
↑ (flèche haut)

# Commande suivante
↓ (flèche bas)

# Caractère précédent/suivant dans la commande
← → (flèches gauche/droite)
```

**Ctrl + raccourcis** :
```bash
# Début de ligne
Ctrl+A

# Fin de ligne
Ctrl+E

# Supprimer jusqu'à la fin de ligne
Ctrl+K

# Supprimer le mot précédent
Ctrl+W
```

### Édition en ligne

**Mode d'édition** :
```bash
# Insérer/en remplacer
Insert

# Supprimer le caractère sous le curseur
Delete

# Supprimer le caractère avant le curseur
Backspace
```

**Déplacement avancé** :
```bash
# Mot précédent
Alt+B (ou Esc+B)

# Mot suivant
Alt+F (ou Esc+F)

# Supprimer le mot suivant
Alt+D
```

## Recherche dans l'historique

### Recherche incrémentale (Ctrl+R)

**Recherche arrière** :
```bash
# Appuyer sur Ctrl+R
(reverse-i-search)`': 

# Taper les premiers caractères
(reverse-i-search)`git': git status

# Répéter Ctrl+R pour résultats précédents
(reverse-i-search)`git': git log --oneline

# Exécuter : Entrée
# Éditer : flèche droite
# Annuler : Ctrl+C ou Ctrl+G
```

### Recherche avec historique

**Commandes de recherche** :
```bash
# Recherche dans l'historique
history | grep "pattern"

# Dernière commande contenant "pattern"
!pattern

# Dernière commande commençant par "pattern"
!pattern:p  # Juste afficher
!pattern    # Exécuter
```

### Raccourcis d'historique

**Références rapides** :
```bash
# Dernière commande
!!

# Commande par numéro
!123

# Dernière commande avec substitution
!!:s/ancien/nouveau/

# Argument de la dernière commande
!$

# Tous les arguments de la dernière commande
!*
```

## Autocomplétion intelligente

### Autocomplétion de base

**Complétion automatique** :
```bash
# Noms de fichiers/répertoires
ls Doc  # Tab → ls Documents/

# Commandes
git sta  # Tab → git status

# Variables
echo $PAT  # Tab → echo $PATH
```

**Types de complétion** :
- **Fichiers/Dossiers** : chemins locaux et absolus
- **Commandes** : programmes dans PATH
- **Variables** : variables shell et d'environnement
- **Utilisateurs** : noms d'utilisateurs système
- **Hôtes** : noms d'hôtes connus (SSH)

### Autocomplétion programmable

**Fonctionnement interne** :
```bash
# Voir les fonctions de complétion
complete -p  # Lister toutes
complete -p git  # Complétion pour git

# Exemple de complétion personnalisée
_complete_mycommand() {
    COMPREPLY=($(compgen -W "option1 option2 option3" "${COMP_WORDS[1]}"))
}
complete -F _complete_mycommand mycommand
```

### Configuration avancée

**Options d'autocomplétion** :
```bash
# ~/.bashrc ou ~/.zshrc
# Ignorer la casse
bind 'set completion-ignore-case on'

# Traiter les _ et - comme équivalents
bind 'set completion-map-case on'

# Complétion partielle des répertoires
bind 'set mark-symlinked-directories on'

# Menu de complétion (plusieurs choix)
bind 'TAB:menu-complete'
```

## Raccourcis clavier essentiels

### Contrôle des processus

**Gestion des jobs** :
```bash
# Suspendre le processus foreground
Ctrl+Z

# Terminer le processus
Ctrl+C

# Sortie propre (EOF)
Ctrl+D
```

### Édition de ligne

**Déplacements rapides** :
```bash
# Début de ligne
Ctrl+A

# Fin de ligne
Ctrl+E

# Supprimer jusqu'à la fin de ligne
Ctrl+K

# Supprimer depuis le début de ligne
Ctrl+U

# Échanger le caractère actuel avec le précédent
Ctrl+T
```

### Recherche et historique

**Recherche dans l'historique** :
```bash
# Recherche arrière incrémentale
Ctrl+R

# Recherche avant
Ctrl+S  # (peut nécessiter stty -ixon)

# Ligne précédente dans l'historique
Ctrl+P

# Ligne suivante dans l'historique
Ctrl+N
```

## Personnalisation de l'autocomplétion

### Scripts de complétion

**Installation de complétions** :
```bash
# Pour Bash (Ubuntu/Debian)
sudo apt install bash-completion

# Activation
echo 'source /usr/share/bash-completion/bash_completion' >> ~/.bashrc

# Pour des outils spécifiques
curl -o ~/.git-completion.bash \
    https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash
echo 'source ~/.git-completion.bash' >> ~/.bashrc
```

### Complétion personnalisée

**Exemple pour une commande maison** :
```bash
# ~/.bashrc
_complete_myapp() {
    local cur prev opts
    COMPREPLY=()
    cur="${COMP_WORDS[COMP_CWORD]}"
    prev="${COMP_WORDS[COMP_CWORD-1]}"
    
    opts="--help --version --verbose start stop restart"
    
    case "${prev}" in
        start|stop)
            COMPREPLY=( $(compgen -W "service1 service2 service3" -- ${cur}) )
            return 0
            ;;
        *)
            ;;
    esac
    
    COMPREPLY=( $(compgen -W "${opts}" -- ${cur}) )
}

complete -F _complete_myapp myapp
```

### Complétion conditionnelle

**Complétion basée sur le contexte** :
```bash
# Complétion différente selon l'argument
_complete_deploy() {
    case ${COMP_CWORD} in
        1) COMPREPLY=( $(compgen -W "staging production" -- "${COMP_WORDS[1]}") ) ;;
        2) COMPREPLY=( $(compgen -W "web app db" -- "${COMP_WORDS[2]}") ) ;;
        *) COMPREPLY=() ;;
    esac
}
complete -F _complete_deploy deploy
```

## Historique partagé et persistant

### Historique multi-session

**Configuration pour Zsh** :
```bash
# ~/.zshrc
setopt share_history
setopt inc_append_history
setopt extended_history
```

**Configuration pour Bash** :
```bash
# ~/.bashrc
shopt -s histappend
PROMPT_COMMAND="history -a; history -n"
```

### Historique horodaté

**Format étendu** :
```bash
# ~/.zshrc
setopt extended_history

# Format: : <beginning time>:<elapsed seconds>;<command>
```

**Recherche temporelle** :
```bash
# Commandes de la dernière heure
history | grep "$(date -d '1 hour ago' '+%Y-%m-%d %H')"

# Statistiques d'usage
history | awk '{print $4}' | sort | uniq -c | sort -nr | head -10
```

### Nettoyage de l'historique

**Suppression sélective** :
```bash
# Supprimer les commandes sensibles
history | grep "password\|secret" | cut -d' ' -f1 | xargs -I {} history -d {}

# Garder seulement les N dernières commandes
history | head -n 1000 > ~/.bash_history_temp
mv ~/.bash_history_temp ~/.bash_history
```

## Raccourcis avancés

### Macros clavier

**Bind personnalisé** :
```bash
# ~/.bashrc
# Raccourci pour git status
bind '"\C-g\C-s":"git status\n"'

# Raccourci pour ls -la
bind '"\C-l\C-a":"ls -la\n"'
```

### Expansion et substitution

**Expansion d'historique** :
```bash
# Dernière commande
!!

# Avec substitution
!!:s/old/new/

# Éviter l'exécution accidentelle
!!:p  # Juste afficher

# Répéter avec sudo
sudo !!
```

### Raccourcis Zsh spécifiques

**Correction automatique** :
```bash
# ~/.zshrc
setopt correct
setopt correct_all

# Correction personnalisée
alias git='nocorrect git'
```

**Expansion globale** :
```bash
# ~/.zshrc
setopt glob_complete    # Complétion avec globs
setopt extended_glob    # Globs étendus
```

## Macros et snippets

### Snippets personnels

**Fonction avec complétion** :
```bash
# ~/.bashrc
deploy_app() {
    local env=${1:-staging}
    local component=${2:-all}
    
    echo "Déploiement de $component en $env"
    # Logique de déploiement...
}

# Complétion pour deploy_app
_complete_deploy_app() {
    case ${COMP_CWORD} in
        1) COMPREPLY=( $(compgen -W "staging production dev" -- "${COMP_WORDS[1]}") ) ;;
        2) COMPREPLY=( $(compgen -W "web api db all" -- "${COMP_WORDS[2]}") ) ;;
    esac
}
complete -F _complete_deploy_app deploy_app
```

### Templates de commandes

**Script de génération** :
```bash
#!/bin/bash
# new_script.sh - Créer un nouveau script Bash
cat << 'EOF' > "$1"
#!/bin/bash
set -euo pipefail

# Description: TODO

main() {
    echo "Hello World"
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    main "$@"
fi
EOF

chmod +x "$1"
echo "Script $1 créé"
```

## Outils complémentaires

### fzf : Fuzzy finder

**Recherche floue dans l'historique** :
```bash
# ~/.bashrc
[ -f ~/.fzf.bash ] && source ~/.fzf.bash

# Raccourci pour historique fuzzy
bind '"\C-r": "\C-x\C-r"'
```

**Utilisation avancée** :
```bash
# Recherche de fichiers avec aperçu
vim $(find . -name "*.md" | fzf --preview "head -20 {}")

# Historique interactif
eval $(history | fzf | sed 's/^[ ]*[0-9]*[ ]*//')
```

### zsh-autosuggestions

**Suggestions automatiques** :
```bash
# Installation (Zsh)
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.zsh/zsh-autosuggestions
echo 'source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh' >> ~/.zshrc

# Navigation dans les suggestions
# → : accepter
# Ctrl+E : aller à la fin et accepter
```

### tldr et cheat

**Aide rapide intégrée** :
```bash
# ~/.bashrc
# Fonction d'aide intelligente
smart_help() {
    local cmd="$1"
    if command -v "$cmd" >/dev/null 2>&1; then
        tldr "$cmd" 2>/dev/null || echo "=== DESCRIPTION ===" && whatis "$cmd" 2>/dev/null || echo "Commande trouvée mais pas de description"
    else
        echo "=== SUGGESTIONS ==="
        apropos "$cmd" | head -5
    fi
}
alias h='smart_help'
```

## Conclusion

L'historique, l'autocomplétion et les raccourcis transforment l'interaction avec le terminal d'une saisie laborieuse en une danse fluide entre l'intention et l'exécution. Ces mécanismes réduisent la charge cognitive tout en augmentant considérablement la productivité.

La véritable maîtrise vient non seulement de connaître ces raccourcis, mais de les intégrer naturellement dans son flux de travail. Ils deviennent alors des extensions de la pensée, permettant d'exprimer des intentions complexes avec une économie de mouvements.

Dans le chapitre suivant, nous explorerons la syntaxe et les conventions des commandes, établissant les règles grammaticales qui gouvernent la communication avec le système.
