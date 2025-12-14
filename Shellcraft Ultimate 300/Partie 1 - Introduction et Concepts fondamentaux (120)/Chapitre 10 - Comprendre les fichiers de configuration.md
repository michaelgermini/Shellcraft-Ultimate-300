# Chapitre 10 - Comprendre les fichiers de configuration

## Table des matières
- [Introduction](#introduction)
- [Architecture des fichiers de configuration](#architecture-des-fichiers-de-configuration)
- [Fichiers de configuration système](#fichiers-de-configuration-système)
- [Fichiers de configuration utilisateur](#fichiers-de-configuration-utilisateur)
- [Gestion des variables d'environnement](#gestion-des-variables-denvironnement)
- [Configuration des shells](#configuration-des-shells)
- [Dotfiles et gestion de configuration](#dotfiles-et-gestion-de-configuration)
- [Outils de gestion de configuration](#outils-de-gestion-de-configuration)
- [Bonnes pratiques](#bonnes-pratiques)
- [Dépannage des configurations](#dépannage-des-configurations)
- [Conclusion](#conclusion)

## Introduction

Les fichiers de configuration sont le cœur personnalisable du terminal. Ils transforment un environnement générique en un espace de travail adapté à vos besoins, habitudes et préférences. Comprendre leur organisation et leur gestion est essentiel pour créer un environnement terminal efficace et maintenable.

Imaginez les fichiers de configuration comme l'ADN de votre environnement terminal : ils déterminent non seulement l'apparence et le comportement, mais aussi les capacités et les automatismes de votre espace de travail.

## Architecture des fichiers de configuration

### Hiérarchie des configurations

**Niveau système (global)** :
- `/etc/profile` - Configuration système Bash
- `/etc/bash.bashrc` - Configuration interactive Bash
- `/etc/zsh/zshrc` - Configuration Zsh système
- `/etc/environment` - Variables d'environnement globales

**Niveau utilisateur** :
- `~/.profile` - Configuration personnelle (shell agnostique)
- `~/.bashrc` - Configuration Bash personnelle
- `~/.zshrc` - Configuration Zsh personnelle
- `~/.bash_profile` - Alternative à .profile pour macOS

### Ordre de chargement

**Séquence typique Bash** :
1. `/etc/profile` (connexion)
2. `~/.profile` ou `~/.bash_profile`
3. `~/.bashrc` (sessions interactives)
4. `/etc/bash.bashrc`

**Séquence Zsh** :
1. `/etc/zsh/zshenv`
2. `~/.zshenv`
3. `/etc/zsh/zrc` (si connexion)
4. `~/.zshrc`
5. `~/.zlogin` (connexion)

## Fichiers de configuration système

### /etc/profile

**Rôle** : Configuration globale au système
```bash
# /etc/profile
# Variables d'environnement système
export PATH="/usr/local/sbin:/usr/sbin:/sbin:$PATH"
export EDITOR="/usr/bin/vim"

# Configuration umask
umask 022

# Chargement des scripts système
for script in /etc/profile.d/*.sh; do
    [ -r "$script" ] && . "$script"
done
```

### /etc/environment

**Variables d'environnement pures** :
```bash
# /etc/environment
PATH="/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin"
LANG="fr_FR.UTF-8"
LC_ALL="fr_FR.UTF-8"
```

### Scripts profile.d

**Modularité** :
```bash
# /etc/profile.d/java.sh
export JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"
export PATH="$JAVA_HOME/bin:$PATH"
```

## Fichiers de configuration utilisateur

### ~/.profile (shell agnostique)

**Configuration de base** :
```bash
# ~/.profile
# Variables d'environnement personnelles
export EDITOR="vim"
export BROWSER="firefox"
export PAGER="less"

# Chemins personnalisés
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"

# Configuration locale
export LANG="fr_FR.UTF-8"
export LC_COLLATE="C"  # Tri ASCII

# Chargement conditionnel
[ -f ~/.bashrc ] && . ~/.bashrc
```

### ~/.bashrc (Bash spécifique)

**Configuration interactive** :
```bash
# ~/.bashrc
# Options Bash
shopt -s histappend     # Ajouter à l'historique
shopt -s checkwinsize   # Taille fenêtre
shopt -s globstar       # Globbing récursif

# Prompt personnalisé
PS1='\u@\h:\w\$ '

# Alias
alias ll='ls -alF'
alias ..='cd ..'

# Fonctions
mkcd() { mkdir -p "$1" && cd "$1"; }
```

### ~/.zshrc (Zsh spécifique)

**Configuration Zsh moderne** :
```bash
# ~/.zshrc
# Options Zsh
setopt autocd           # cd automatique
setopt correct          # Correction orthographique
setopt share_history    # Historique partagé

# Autocomplétion
autoload -Uz compinit
compinit

# Prompt (avec thème)
autoload -Uz promptinit
promptinit
prompt adam2

# Plugins (avec Oh My Zsh)
plugins=(git docker kubectl)
```

## Gestion des variables d'environnement

### Variables essentielles

**PATH** : Chemins d'exécution
```bash
# Ajouter au début
export PATH="/usr/local/bin:$PATH"

# Ajouter à la fin
export PATH="$PATH:$HOME/bin"

# Vérifier
echo "$PATH" | tr ':' '\n'
```

**Variables locales** :
```bash
export EDITOR="code --wait"
export VISUAL="$EDITOR"
export PAGER="less -R"
export BROWSER="chromium"
```

### Persistance

**Session vs permanente** :
```bash
# Temporaire (session courante)
export TEMP_VAR="valeur"

# Permanente (ajouter à ~/.profile)
echo 'export PERM_VAR="valeur"' >> ~/.profile
```

## Configuration des shells

### Personnalisation du prompt

**Bash simple** :
```bash
# ~/.bashrc
PS1='\[\e[32m\]\u@\h:\[\e[34m\]\w\[\e[0m\]\$ '
# Vert: user@host, Bleu: répertoire, Normal: $
```

**Zsh avec thème** :
```bash
# ~/.zshrc
autoload -Uz promptinit
promptinit

# Thème personnalisé
PROMPT='%F{green}%n@%m%f:%F{blue}%~%f$ '
RPROMPT='%F{yellow}%T%f'  # Heure à droite
```

### Options de complétion

**Bash** :
```bash
# ~/.bashrc
# Ignorer la casse
bind 'set completion-ignore-case on'

# Complétion des variables
bind 'TAB:menu-complete'
```

**Zsh** :
```bash
# ~/.zshrc
# Complétion intelligente
zstyle ':completion:*' menu select
zstyle ':completion:*' matcher-list 'm:{a-z}={A-Za-z}'
```

## Dotfiles et gestion de configuration

### Qu'est-ce qu'un dotfile ?

**Fichiers de configuration cachés** :
```bash
# Liste des dotfiles courants
~/.bashrc ~/.zshrc ~/.profile
~/.vimrc ~/.tmux.conf ~/.gitconfig
~/.ssh/config ~/.config/*
```

### Organisation

**Structure recommandée** :
```bash
~/.dotfiles/
├── bash/
│   ├── .bashrc
│   └── .bash_profile
├── zsh/
│   ├── .zshrc
│   └── .zshenv
├── git/
│   └── .gitconfig
├── vim/
│   └── .vimrc
└── scripts/
    └── install.sh
```

### Gestion avec Git

**Repository dotfiles** :
```bash
# Initialisation
git init ~/.dotfiles
cd ~/.dotfiles

# Ajouter fichiers (sans les cacher)
cp ~/.bashrc bashrc
git add bashrc

# Script de déploiement
#!/bin/bash
# deploy.sh
ln -sf ~/.dotfiles/bashrc ~/.bashrc
ln -sf ~/.dotfiles/zshrc ~/.zshrc
```

## Outils de gestion de configuration

### Chez Moi

**Gestionnaire moderne** :
```bash
# Installation
# Via script d'installation officiel

# Initialisation
chezmoi init

# Ajouter un fichier
chezmoi add ~/.bashrc

# Appliquer
chezmoi apply
```

### GNU Stow

**Gestion par liens symboliques** :
```bash
# Structure
~/.dotfiles/
└── vim/
    └── .vimrc

# Déploiement
cd ~/.dotfiles
stow vim  # Crée ~/.vimrc -> ~/.dotfiles/vim/.vimrc
```

### Ansible

**Automatisation complète** :
```yaml
# playbook.yml
- hosts: localhost
  tasks:
    - name: Install packages
      package:
        name: "{{ item }}"
      loop: "{{ packages }}"

    - name: Configure dotfiles
      copy:
        src: "{{ item.src }}"
        dest: "{{ item.dest }}"
      loop: "{{ dotfiles }}"
```

## Bonnes pratiques

### Modularité

**Séparation des préoccupations** :
```bash
# ~/.bashrc
# Charger des modules séparés
for file in ~/.bashrc.d/*.sh; do
    [ -r "$file" ] && . "$file"
done

# ~/.bashrc.d/aliases.sh
# ~/.bashrc.d/functions.sh
# ~/.bashrc.d/prompt.sh
```

### Documentation

**Commentaires explicites** :
```bash
# ~/.bashrc
# Configuration Bash personnelle
# Créé le: 2024-01-15
# Dernière modification: 2024-01-20

# Options Bash essentielles
# shopt -s: active l'option, -u: désactive
shopt -s histappend  # Ajouter à l'historique existant
shopt -s checkwinsize  # Vérifier la taille de la fenêtre
```

### Tests et validation

**Validation de syntaxe** :
```bash
# Bash
bash -n ~/.bashrc

# Zsh
zsh -n ~/.zshrc
```

**Tests isolés** :
```bash
# Tester dans un shell isolé
bash --rcfile ~/.bashrc
```

## Dépannage des configurations

### Problèmes courants

**Variables non définies** :
```bash
# Debug des variables
echo "PATH=$PATH"
echo "HOME=$HOME"
env | grep -i term  # Variables terminal
```

**Prompt cassé** :
```bash
# Reset du prompt
PS1='$ '

# Vérifier les caractères d'échappement
echo "$PS1" | cat -A
```

**Chargement lent** :
```bash
# Profiler le chargement
time bash -c 'source ~/.bashrc; exit'

# Identifier les goulots
bash -x ~/.bashrc  # Mode debug
```

### Outils de diagnostic

**Vérification de l'environnement** :
```bash
# Variables actives
declare -p | grep -E '^(declare -x|export)'

# Fonctions définies
declare -F

# Alias actifs
alias
```

**Comparaison avec défaut** :
```bash
# Shell vierge
bash --norc --noprofile

# Comparer les configurations
diff <(env) <(bash --norc --noprofile -c 'env')
```

## Conclusion

Les fichiers de configuration transforment un terminal générique en un environnement personnalisé et puissant. Leur compréhension permet non seulement la personnalisation, mais aussi le débogage efficace et la reproductibilité des environnements.

La clé d'une bonne configuration est la modularité : séparer les préoccupations, documenter clairement, et maintenir une approche systématique. Les dotfiles deviennent ainsi un actif précieux, versionnable et partageable.

Dans les prochains chapitres, nous explorerons l'environnement de travail et les variables, puis les notions de base en permissions, pour compléter notre compréhension des fondements du terminal.
