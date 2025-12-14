# Chapitre 265 - Outils de productivité : tmux, fzf, htop

## Table des matières
- [Introduction](#introduction)
- [tmux : multiplexeur de terminal](#tmux--multiplexeur-de-terminal)
- [fzf : fuzzy finder universel](#fzf--fuzzy-finder-universel)
- [htop : moniteur de processus avancé](#htop--moniteur-de-processus-avancé)
- [Intégration avec l'IA](#intégration-avec-lia)
- [Workflows combinés](#workflows-combinés)
- [Configuration avancée](#configuration-avancée)
- [Conclusion](#conclusion)

## Introduction

Les outils de productivité transforment le terminal d'un simple interpréteur de commandes en un environnement de travail puissant et efficace. tmux permet de gérer plusieurs sessions et fenêtres, fzf offre une recherche floue instantanée, et htop fournit une visualisation interactive des processus. Combinés avec l'intelligence artificielle, ces outils créent un environnement de développement et d'administration système sans égal.

Imaginez ces outils comme les extensions de votre cerveau : tmux étend votre mémoire de travail en permettant de garder plusieurs contextes actifs simultanément, fzf étend votre capacité de recherche en trouvant instantanément ce que vous cherchez même si vous ne connaissez pas le nom exact, et htop étend votre perception en vous montrant ce qui se passe réellement dans votre système.

## tmux : multiplexeur de terminal

### Concepts fondamentaux

**Qu'est-ce que tmux ?** :
- Terminal Multiplexer : permet de créer plusieurs sessions de terminal
- Sessions persistantes : continuent même après déconnexion SSH
- Partage de sessions : plusieurs utilisateurs peuvent voir la même session
- Gestion de fenêtres et panneaux : organisation flexible de l'espace

**Avantages** :
- Continuité de travail même après déconnexion
- Organisation efficace de plusieurs tâches
- Partage de sessions pour collaboration
- Scripting et automatisation

### Installation et configuration de base

**Installation** :
```bash
# Linux (Debian/Ubuntu)
sudo apt install tmux

# Linux (Fedora/RHEL)
sudo dnf install tmux

# macOS
brew install tmux

# Arch Linux
sudo pacman -S tmux
```

**Configuration de base** :
```bash
# ~/.tmux.conf
# Préfixe : Ctrl+b (par défaut)
# Changer le préfixe en Ctrl+a (plus accessible)
unbind C-b
set-option -g prefix C-a
bind-key C-a send-prefix

# Activer la souris
set -g mouse on

# Numérotation des fenêtres à partir de 1
set -g base-index 1
setw -g pane-base-index 1

# Rafraîchir la configuration
bind r source-file ~/.tmux.conf \; display "Config reloaded!"

# Couleurs 256
set -g default-terminal "screen-256color"
set -ga terminal-overrides ",xterm-256color:Tc"

# Historique plus long
set -g history-limit 10000

# Mode copie amélioré (compatible vim)
setw -g mode-keys vi
bind-key -T copy-mode-vi v send-keys -X begin-selection
bind-key -T copy-mode-vi y send-keys -X copy-selection-and-cancel
```

### Utilisation de base

**Sessions** :
```bash
# Créer une nouvelle session
tmux new -s nom_session

# Lister les sessions
tmux ls

# Attacher à une session
tmux attach -t nom_session
# ou
tmux a -t nom_session

# Détacher d'une session
# Dans tmux : Ctrl+a d

# Tuer une session
tmux kill-session -t nom_session

# Tuer toutes les sessions
tmux kill-server
```

**Fenêtres** :
```bash
# Dans tmux (préfixe Ctrl+a)
# Créer une nouvelle fenêtre
Ctrl+a c

# Fermer la fenêtre courante
Ctrl+a &

# Naviguer entre fenêtres
Ctrl+a n  # Suivante
Ctrl+a p  # Précédente
Ctrl+a 0-9  # Aller à la fenêtre N

# Renommer la fenêtre courante
Ctrl+a ,

# Lister toutes les fenêtres
Ctrl+a w
```

**Panneaux** :
```bash
# Dans tmux
# Diviser horizontalement
Ctrl+a "

# Diviser verticalement
Ctrl+a %

# Naviguer entre panneaux
Ctrl+a o  # Suivant
Ctrl+a ;  # Précédent
Ctrl+a flèches  # Direction

# Redimensionner
Ctrl+a Ctrl+flèches

# Fermer le panneau
Ctrl+a x

# Basculer en plein écran
Ctrl+a z
```

### Configuration avancée

**Configuration productive** :
```bash
# ~/.tmux.conf
# Raccourcis personnalisés

# Split horizontal avec répertoire courant
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"

# Synchroniser les panneaux (taper dans tous)
bind S setw synchronize-panes

# Navigation vim-like
bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# Redimensionnement vim-like
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# Status bar améliorée
set -g status-position bottom
set -g status-justify left
set -g status-style 'bg=#333333 fg=#5e5e5e'
set -g status-left-length 40
set -g status-left '#[fg=green,bold]#S '
set -g status-right '#[fg=yellow,bold]%Y-%m-%d %H:%M'
set -g status-right-length 50

# Fenêtres actives
setw -g window-status-style 'fg=colour247 bg=colour238'
setw -g window-status-current-style 'fg=colour81 bg=colour238 bold'
setw -g window-status-format ' #I #W '
setw -g window-status-current-format ' #I #W '

# Alertes visuelles
setw -g monitor-activity on
set -g visual-activity on
```

**Scripts tmux** :
```bash
#!/bin/bash
# Créer une session de développement complète

tmux new-session -d -s dev

# Fenêtre 1: Éditeur
tmux rename-window -t dev:0 'editor'
tmux send-keys -t dev:0 'vim' C-m

# Fenêtre 2: Terminal
tmux new-window -t dev:1 -n 'terminal'
tmux send-keys -t dev:1 'cd ~/projects' C-m

# Fenêtre 3: Logs
tmux new-window -t dev:2 -n 'logs'
tmux split-window -h -t dev:2
tmux send-keys -t dev:2.0 'tail -f /var/log/app.log' C-m
tmux send-keys -t dev:2.1 'journalctl -f' C-m

# Fenêtre 4: Git
tmux new-window -t dev:3 -n 'git'
tmux send-keys -t dev:3 'git status' C-m

# Attacher à la session
tmux attach -t dev
```

### Intégration avec scripts

**Scripts automatisés** :
```bash
#!/bin/bash
# Déployer avec tmux

deploy_with_tmux() {
    local project="$1"
    local env="${2:-production}"
    
    # Créer une session de déploiement
    tmux new-session -d -s "deploy-$project"
    
    # Panneau 1: Build
    tmux send-keys -t "deploy-$project" "cd $project && npm run build" C-m
    
    # Panneau 2: Tests
    tmux split-window -v
    tmux send-keys -t "deploy-$project" "cd $project && npm test" C-m
    
    # Panneau 3: Déploiement
    tmux split-window -h
    tmux send-keys -t "deploy-$project" "cd $project && ./deploy.sh $env" C-m
    
    # Attacher pour voir le résultat
    tmux attach -t "deploy-$project"
}
```

## fzf : fuzzy finder universel

### Installation et configuration

**Installation** :
```bash
# Linux (Debian/Ubuntu)
sudo apt install fzf

# macOS
brew install fzf

# Via script d'installation
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

# Arch Linux
sudo pacman -S fzf
```

**Configuration shell** :
```bash
# ~/.bashrc ou ~/.zshrc
# Intégration fzf
[ -f ~/.fzf.bash ] && source ~/.fzf.bash

# Pour Zsh
[ -f ~/.fzf.zsh ] && source ~/.fzf.zsh
```

### Utilisation de base

**Recherche de fichiers** :
```bash
# Recherche interactive de fichiers
fzf

# Recherche dans un répertoire spécifique
fzf --path ~/projects

# Recherche avec prévisualisation
fzf --preview 'head -20 {}'

# Recherche avec plusieurs prévisualisations
fzf --preview 'cat {}' --preview-window=up:40%
```

**Intégration avec les commandes** :
```bash
# Ouvrir un fichier trouvé avec vim
vim $(fzf)

# Éditer un fichier trouvé
code $(fzf)

# Copier un fichier trouvé
cp $(fzf) ~/backup/

# Supprimer des fichiers sélectionnés
rm -i $(fzf -m)
```

### Fonctions personnalisées

**Fonctions utiles** :
```bash
# Recherche de fichiers et édition
fe() {
    local file
    file=$(fzf --query="$1" --select-1 --exit-0)
    [ -n "$file" ] && ${EDITOR:-vim} "$file"
}

# Changement de répertoire
fd() {
    local dir
    dir=$(find ${1:-.} -path '*/\.*' -prune \
                    -o -type d -print 2> /dev/null | fzf +m) &&
    cd "$dir"
}

# Recherche dans l'historique
fh() {
    print -z $(history | fzf +s +m -n2..,.. --tac +m | sed 's/^[0-9 ]*  //')
}

# Recherche de processus
fkill() {
    local pid
    pid=$(ps -ef | sed 1d | fzf -m | awk '{print $2}')
    [ -n "$pid" ] && kill -${1:-9} "$pid"
}

# Recherche Git
fg() {
    local file
    file=$(git ls-files | fzf -m --preview 'git diff {}')
    [ -n "$file" ] && git add "$file"
}
```

**Alias pratiques** :
```bash
# ~/.bashrc ou ~/.zshrc
alias ff='fzf --preview "head -20 {}"'
alias fv='vim $(fzf)'
alias fcd='cd $(find . -type d | fzf)'
alias fkill='ps aux | fzf | awk "{print \$2}" | xargs kill'
```

### Intégration avancée

**Avec Git** :
```bash
# Sélectionner des fichiers Git
git_add() {
    git ls-files -m -o --exclude-standard | fzf -m | xargs git add
}

# Sélectionner une branche
git_checkout() {
    git branch | fzf | xargs git checkout
}

# Voir les différences
git_diff() {
    git diff --name-only | fzf -m --preview 'git diff {}'
}
```

**Avec Docker** :
```bash
# Sélectionner un conteneur
docker_exec() {
    docker ps | fzf | awk '{print $1}' | xargs -I {} docker exec -it {} bash
}

# Sélectionner une image
docker_run() {
    docker images | fzf | awk '{print $1":"$2}' | xargs docker run -it
}
```

**Avec SSH** :
```bash
# Sélectionner un serveur SSH
sshf() {
    local host
    host=$(grep -E "^Host " ~/.ssh/config | awk '{print $2}' | fzf)
    [ -n "$host" ] && ssh "$host"
}
```

### Configuration avancée

**Configuration fzf** :
```bash
# ~/.fzfrc ou dans ~/.bashrc/~/.zshrc
export FZF_DEFAULT_OPTS='
    --height 40%
    --border
    --layout=reverse
    --info=inline
    --preview-window=right:50%
    --bind=ctrl-/:toggle-preview
    --color=fg:#f8f8f2,bg:#282a36,hl:#bd93f9
    --color=fg+:#f8f8f2,bg+:#44475a,hl+:#bd93f9
    --color=info:#ffb86c,prompt:#50fa7b,pointer:#ff79c6
    --color=marker:#ff79c6,spinner:#ffb86c,header:#6272a4
'

# Options par défaut
export FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
export FZF_ALT_C_COMMAND='fd --type d --hidden --follow --exclude .git'
```

## htop : moniteur de processus avancé

### Installation et utilisation

**Installation** :
```bash
# Linux (Debian/Ubuntu)
sudo apt install htop

# Linux (Fedora/RHEL)
sudo dnf install htop

# macOS
brew install htop

# Arch Linux
sudo pacman -S htop
```

**Utilisation de base** :
```bash
# Lancer htop
htop

# Avec options
htop -u username  # Filtrer par utilisateur
htop -p PID       # Filtrer par PID
htop -s COLUMN    # Trier par colonne
```

### Navigation et commandes

**Raccourcis clavier** :
```
F1  : Aide
F2  : Configuration (colonnes, couleurs)
F3  : Recherche de processus
F4  : Filtrer par nom
F5  : Vue en arbre
F6  : Trier par colonne
F7  : Diminuer la priorité (nice)
F8  : Augmenter la priorité (nice)
F9  : Tuer un processus
F10 : Quitter
```

**Tri et filtrage** :
```bash
# Dans htop
# Trier par CPU
F6 puis sélectionner PERCENT_CPU

# Trier par mémoire
F6 puis sélectionner M_RESIDENT

# Filtrer par nom
F4 puis taper le nom

# Recherche
F3 puis taper le terme
```

### Configuration

**Configuration personnalisée** :
```bash
# Créer la configuration
mkdir -p ~/.config/htop
cp /etc/htoprc ~/.config/htop/htoprc

# Éditer la configuration
vim ~/.config/htop/htoprc
```

**Paramètres utiles** :
```bash
# ~/.config/htop/htoprc
# Afficher les threads
hide_threads=0

# Afficher les processus kernel
hide_kernel_threads=0

# Afficher les processus utilisateur
hide_userland_threads=0

# Vue en arbre par défaut
tree_view=1

# Couleurs
color_scheme=0  # 0-6 selon préférence

# Colonnes à afficher
fields=0 48 17 18 38 39 40 2 46 47 49 1
```

### Alternatives et compléments

**btop** (alternative moderne) :
```bash
# Installation
# Via cargo
cargo install btop

# Utilisation
btop
```

**glances** (monitoring complet) :
```bash
# Installation
pip install glances

# Utilisation
glances
```

**bashtop** (monitoring en Bash) :
```bash
# Installation
git clone https://github.com/aristocratos/bashtop.git
cd bashtop
sudo make install

# Utilisation
bashtop
```

## Intégration avec l'IA

### Scripts intelligents avec tmux

**Session intelligente** :
```bash
#!/bin/bash
# Créer une session tmux intelligente avec IA

ai_tmux_session() {
    local project="$1"
    
    # Analyser le projet avec IA
    local analysis=$(ai_analyze_project "$project")
    
    # Créer la session selon l'analyse
    tmux new-session -d -s "$project"
    
    # Fenêtres suggérées par IA
    echo "$analysis" | jq -r '.suggested_windows[]' | \
    while read -r window; do
        tmux new-window -t "$project" -n "$window"
        # Commande suggérée par IA
        local cmd=$(echo "$analysis" | jq -r ".windows[\"$window\"]")
        tmux send-keys -t "$project:$window" "$cmd" C-m
    done
    
    tmux attach -t "$project"
}
```

### Recherche intelligente avec fzf

**Recherche avec contexte IA** :
```bash
#!/bin/bash
# Recherche intelligente de fichiers

ai_find() {
    local query="$1"
    
    # Recherche normale
    local results=$(fzf --query "$query")
    
    # Analyser avec IA pour suggestions
    local suggestions=$(ai_suggest_files "$query" "$results")
    
    # Afficher les suggestions
    echo "$suggestions" | fzf --multi --preview 'cat {}'
}
```

### Monitoring intelligent avec htop

**Alertes intelligentes** :
```bash
#!/bin/bash
# Monitoring intelligent avec htop et IA

intelligent_monitoring() {
    while true; do
        # Collecter les métriques
        local metrics=$(htop -d 1 -n 1 | extract_metrics)
        
        # Analyser avec IA
        local analysis=$(ai_analyze_metrics "$metrics")
        
        # Détecter les problèmes
        if [ "$(echo "$analysis" | jq -r '.has_issues')" == "true" ]; then
            # Alertes intelligentes
            ai_alert "$analysis"
        fi
        
        sleep 60
    done
}
```

## Workflows combinés

### Workflow de développement

**Setup complet** :
```bash
#!/bin/bash
# Workflow de développement avec tmux, fzf, htop

dev_workflow() {
    local project="$1"
    
    # Créer session tmux
    tmux new-session -d -s "dev-$project"
    
    # Fenêtre 1: Éditeur avec fzf
    tmux rename-window -t "dev-$project:0" 'editor'
    tmux send-keys -t "dev-$project:0" 'vim $(fzf)' C-m
    
    # Fenêtre 2: Terminal avec fzf
    tmux new-window -t "dev-$project:1" -n 'terminal'
    tmux send-keys -t "dev-$project:1" 'cd $(find . -type d | fzf)' C-m
    
    # Fenêtre 3: Monitoring avec htop
    tmux new-window -t "dev-$project:2" -n 'monitor'
    tmux send-keys -t "dev-$project:2" 'htop' C-m
    
    # Fenêtre 4: Git avec fzf
    tmux new-window -t "dev-$project:3" -n 'git'
    tmux send-keys -t "dev-$project:3" 'git status' C-m
    
    # Attacher
    tmux attach -t "dev-$project"
}
```

### Workflow d'administration

**Monitoring système** :
```bash
#!/bin/bash
# Workflow d'administration système

admin_workflow() {
    # Créer session admin
    tmux new-session -d -s admin
    
    # Panneau 1: htop
    tmux send-keys -t admin 'htop' C-m
    
    # Panneau 2: Logs avec fzf
    tmux split-window -v
    tmux send-keys -t admin 'tail -f /var/log/syslog | fzf' C-m
    
    # Panneau 3: Commandes fréquentes
    tmux split-window -h
    tmux send-keys -t admin 'history | fzf' C-m
    
    tmux attach -t admin
}
```

## Configuration avancée

### Configuration complète tmux

**Configuration professionnelle** :
```bash
# ~/.tmux.conf
# Configuration complète et productive

# ============================================================================
# OPTIONS DE BASE
# ============================================================================
set -g default-terminal "screen-256color"
set -ga terminal-overrides ",xterm-256color:Tc"
set -g mouse on
set -g history-limit 10000
set -g base-index 1
setw -g pane-base-index 1

# ============================================================================
# PRÉFIXE ET RACCOURCIS
# ============================================================================
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# Rafraîchir config
bind r source-file ~/.tmux.conf \; display "Config reloaded!"

# ============================================================================
# NAVIGATION VIM-LIKE
# ============================================================================
setw -g mode-keys vi
bind-key -T copy-mode-vi v send-keys -X begin-selection
bind-key -T copy-mode-vi y send-keys -X copy-selection-and-cancel

bind h select-pane -L
bind j select-pane -D
bind k select-pane -U
bind l select-pane -R

# ============================================================================
# SPLIT WINDOWS
# ============================================================================
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
unbind '"'
unbind %

# ============================================================================
# REDIMENSIONNEMENT
# ============================================================================
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5

# ============================================================================
# STATUS BAR
# ============================================================================
set -g status-position bottom
set -g status-justify left
set -g status-style 'bg=#333333 fg=#5e5e5e'
set -g status-left-length 40
set -g status-left '#[fg=green,bold]#S '
set -g status-right '#[fg=yellow,bold]%Y-%m-%d %H:%M #[fg=cyan]#(whoami)@#h'
set -g status-right-length 60

setw -g window-status-style 'fg=colour247 bg=colour238'
setw -g window-status-current-style 'fg=colour81 bg=colour238 bold'
setw -g window-status-format ' #I #W '
setw -g window-status-current-format ' #I #W '

# ============================================================================
# PLUGINS (si tmux-plugin-manager installé)
# ============================================================================
# set -g @plugin 'tmux-plugins/tpm'
# set -g @plugin 'tmux-plugins/tmux-sensible'
# set -g @plugin 'tmux-plugins/tmux-resurrect'
# run '~/.tmux/plugins/tpm/tpm'
```

### Configuration complète fzf

**Configuration avancée** :
```bash
# ~/.fzfrc
# Configuration fzf complète

# Options par défaut
export FZF_DEFAULT_OPTS='
    --height 40%
    --border
    --layout=reverse
    --info=inline
    --preview-window=right:50%
    --bind=ctrl-/:toggle-preview
    --bind=ctrl-f:page-down
    --bind=ctrl-b:page-up
    --bind=ctrl-j:down
    --bind=ctrl-k:up
    --color=fg:#f8f8f2,bg:#282a36,hl:#bd93f9
    --color=fg+:#f8f8f2,bg+:#44475a,hl+:#bd93f9
    --color=info:#ffb86c,prompt:#50fa7b,pointer:#ff79c6
    --color=marker:#ff79c6,spinner:#ffb86c,header:#6272a4
'

# Commandes par défaut
export FZF_DEFAULT_COMMAND='fd --type f --hidden --follow --exclude .git'
export FZF_CTRL_T_COMMAND="$FZF_DEFAULT_COMMAND"
export FZF_ALT_C_COMMAND='fd --type d --hidden --follow --exclude .git'

# Prévisualisations avancées
export FZF_CTRL_T_OPTS="
    --preview 'bat --color=always --style=header,grid --line-range :300 {}'
    --bind 'ctrl-/:change-preview-window(down|hidden|)'
    --bind 'ctrl-v:execute(code {})+abort'
"

export FZF_ALT_C_OPTS="
    --preview 'tree -C {} | head -200'
    --bind 'ctrl-/:change-preview-window(down|hidden|)'
"
```

## Conclusion

Les outils de productivité tmux, fzf et htop transforment fondamentalement l'expérience du terminal. tmux offre la persistance et l'organisation, fzf révolutionne la recherche, et htop fournit une visualisation claire des processus système. Combinés avec l'intelligence artificielle, ces outils créent un environnement de travail terminal qui rivalise et dépasse souvent les interfaces graphiques modernes.

La maîtrise de ces outils ne vient pas seulement de leur utilisation individuelle, mais de leur intégration harmonieuse dans des workflows personnalisés qui s'adaptent à vos besoins spécifiques. En investissant du temps dans leur configuration et leur apprentissage, vous multipliez votre productivité et votre efficacité.

Dans le chapitre suivant, nous explorerons l'automatisation intelligente avec le cloud, découvrant comment combiner ces outils de productivité avec les capacités du cloud pour créer des systèmes d'automatisation puissants.

