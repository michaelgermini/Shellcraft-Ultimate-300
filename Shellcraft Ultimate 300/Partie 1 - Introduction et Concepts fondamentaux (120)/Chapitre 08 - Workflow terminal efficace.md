# Chapitre 08 - Workflow terminal efficace

## Table des matières
- [Introduction](#introduction)
- [Principes d'efficacité](#principes-defficacité)
- [Navigation et manipulation de fichiers](#navigation-et-manipulation-de-fichiers)
- [Recherche et filtrage](#recherche-et-filtrage)
- [Multitâche et sessions](#multitâche-et-sessions)
- [Automatisation des tâches répétitives](#automatisation-des-tâches-répétitives)
- [Gestion de l'historique](#gestion-de-lhistorique)
- [Personnalisation de l'environnement](#personnalisation-de-lenvironnement)
- [Outils de productivité](#outils-de-productivité)
- [Workflows spécialisés](#workflows-spécialisés)
- [Conclusion](#conclusion)

## Introduction

Un workflow terminal efficace n'est pas seulement une question de connaître les commandes ; c'est une approche systématique qui combine outils, raccourcis et automatisation pour maximiser la productivité. Cette efficacité vient d'une compréhension profonde des patterns de travail et de l'optimisation de chaque interaction.

Imaginez le terminal comme un instrument de musique : la connaissance des notes (commandes) est nécessaire, mais la maîtrise vient de la façon dont vous les enchaînez en mélodies productives.

## Principes d'efficacité

### Minimiser la saisie

**Principe** : Chaque frappe économisée est du temps gagné.

**Techniques** :
- Raccourcis clavier au lieu de commandes longues
- Autocomplétion intelligente
- Historique et recherche floue
- Alias pour les commandes fréquentes

### Composer les outils

**Pipeline thinking** :
```bash
# Au lieu de étapes séparées
grep "error" app.log > errors.txt
sort errors.txt > sorted_errors.txt
uniq sorted_errors.txt > unique_errors.txt

# Pipeline efficace
grep "error" app.log | sort | uniq > unique_errors.txt
```

### Prévention des erreurs

**Mode strict** :
```bash
set -euo pipefail  # Échec sur première erreur
```

**Validation** :
```bash
# Vérifier avant d'agir
[ -f "$file" ] && rm "$file" || echo "Fichier inexistant"
```

## Navigation et manipulation de fichiers

### Navigation intelligente

**Raccourcis de répertoire** :
```bash
# Créer des marqueurs
export PROJETS="$HOME/projects"
export DOCS="$HOME/documents"

# Navigation rapide
cd "$PROJETS/web-app"
cd "$DOCS/notes"
```

**Pile de répertoires** :
```bash
# Sauvegarder position
pushd /tmp
# Travail...
popd  # Retour automatique
```

### Manipulation avancée

**Opérations groupées** :
```bash
# Renommer multiple
for file in *.txt; do mv "$file" "${file%.txt}.md"; done

# Traitement conditionnel
find . -name "*.log" -mtime +7 -delete  # Logs > 7 jours
```

**Copies intelligentes** :
```bash
# Copie avec progression
rsync -avh --progress source/ destination/

# Synchronisation bidirectionnelle
unison source/ destination/
```

## Recherche et filtrage

### Recherche de fichiers

**Find patterns optimaux** :
```bash
# Fichiers modifiés récemment
find . -mtime -1 -type f

# Recherche par taille
find . -size +100M -exec ls -lh {} \;

# Combinaisons complexes
find . -name "*.py" -exec grep -l "TODO" {} \;
```

### Recherche dans le contenu

**Ripgrep pour la performance** :
```bash
# Recherche rapide
rg "function_name" --type py

# Recherche avec contexte
rg -C 3 "error" *.log

# Recherche interactive
fzf --bind 'enter:execute(code {})'
```

### Filtrage et traitement

**Chaînes de traitement** :
```bash
# Analyse de logs
tail -f app.log | grep "ERROR" | cut -d' ' -f3 | sort | uniq -c | sort -nr
```

## Multitâche et sessions

### Multiplexage avec tmux

**Sessions persistantes** :
```bash
# Créer session
tmux new -s dev

# Détacher : Ctrl+b d
# Rattacher : tmux a -t dev

# Panneaux multiples
tmux split-window -v  # Split vertical
tmux split-window -h  # Split horizontal
```

**Configuration productive** :
```bash
# .tmux.conf
set -g mouse on
bind-key -n C-t new-window
bind-key -n C-Tab next-window
```

### Gestion des processus

**Contrôle fin** :
```bash
# Jobs en arrière-plan
command &
jobs
fg %1  # Premier plan
bg %1  # Arrière-plan

# Gestion avancée
disown %1  # Détacher du shell
```

**Monitoring** :
```bash
# Processus par utilisateur
ps u "$USER"

# Arbre des processus
pstree -p

# Ressources en temps réel
htop
```

## Automatisation des tâches répétitives

### Scripts quotidiens

**Sauvegarde automatisée** :
```bash
#!/bin/bash
# daily_backup.sh
BACKUP_DIR="/mnt/backup/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"
rsync -a --delete ~/documents/ "$BACKUP_DIR/"
```

**Nettoyage système** :
```bash
#!/bin/bash
# system_clean.sh
# Cache, logs, paquets obsolètes
sudo apt autoremove
sudo apt autoclean
rm -rf ~/.cache/thumbnails/*
find ~/.local/share/Trash -mtime +30 -delete
```

### Cron et planification

**Tâches planifiées** :
```bash
# Éditer crontab
crontab -e

# Exemples
@daily ~/scripts/daily_backup.sh
*/30 * * * * ~/scripts/health_check.sh  # Toutes les 30 min
@reboot ~/scripts/start_services.sh
```

**Systemd timers** :
```bash
# Fichier .timer
[Unit]
Description=Daily backup timer

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

## Gestion de l'historique

### Recherche intelligente

**Ctrl+R interactif** :
```bash
# Recherche dans l'historique
Ctrl+R
(reverse-i-search)`git': git status
```

**Historique avec contexte** :
```bash
# Dernières commandes
history | tail -10

# Recherche floue
history | grep "ssh"
```

### Historique partagé

**Configuration Zsh/Bash** :
```bash
# .zshrc
setopt share_history
setopt hist_ignore_dups
HISTSIZE=10000
SAVEHIST=10000
```

## Personnalisation de l'environnement

### Alias productifs

**Navigation** :
```bash
alias ..='cd ..'
alias ...='cd ../..'
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
```

**Opérations courantes** :
```bash
alias gs='git status'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
alias gl='git log --oneline'
```

### Fonctions avancées

**Navigation intelligente** :
```bash
# Aller au répertoire le plus récemment modifié
recent() {
    cd "$(find . -type d -printf '%T@ %p\n' | sort -n | tail -1 | cut -d' ' -f2-)"
}
```

**Recherche et édition** :
```bash
# Éditer le fichier trouvé
edit() {
    local file
    file=$(fzf) && ${EDITOR:-vim} "$file"
}
```

## Outils de productivité

### Fuzzy finding

**fzf pour tout** :
```bash
# Recherche de fichiers
vim $(find . -name "*.md" | fzf)

# Historique interactif
eval $(history | fzf | sed 's/^[ ]*[0-9]*[ ]*//')
```

### Éditeurs en ligne de commande

**Édition rapide** :
```bash
# sed pour modifications simples
sed -i 's/old/new/g' file.txt

# awk pour traitement
awk '{print $1, $NF}' data.txt
```

### Outils modernes

**tldr pour l'aide** :
```bash
# Au lieu de man ls
tldr ls
```

**cheat.sh pour les exemples** :
```bash
# Exemples pratiques
curl cheat.sh/tar

# Recherche dans la communauté
curl cheat.sh/find~-name
```

## Workflows spécialisés

### Développement

**Workflow Git avancé** :
```bash
# Status et commit intelligent
alias gst='git status --short'
alias gca='git add -A && git commit --amend --no-edit'

# Branches
alias gb='git branch | fzf | xargs git checkout'
```

### Administration système

**Monitoring continu** :
```bash
# Watch avec intervalle
watch -n 5 'ps aux | head -10'

# Alertes
if [ $(df / | tail -1 | awk '{print $5}' | sed 's/%//') -gt 90 ]; then
    echo "Disque presque plein!" | mail -s "Alerte disque" admin@example.com
fi
```

### Analyse de données

**Traitement en flux** :
```bash
# Analyse de logs temps réel
tail -f access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -10
```

## Conclusion

Un workflow terminal efficace combine connaissance technique, automatisation et optimisation ergonomique. Il transforme des tâches complexes en routines fluides, permettant de se concentrer sur l'essentiel plutôt que sur la mécanique.

Les principes clés - composition, automatisation, prévention des erreurs - s'appliquent à tous les domaines : développement, administration système, analyse de données.

L'efficacité ultime vient de l'adaptation personnalisée : chaque développeur développe son propre workflow, optimisé pour ses tâches et ses préférences. Les outils présentés ici ne sont que des points de départ ; l'excellence vient de leur orchestration personnelle.

Dans les chapitres suivants, nous explorerons les outils d'aide rapide et la compréhension des fichiers de configuration, éléments essentiels d'un environnement de travail personnalisé et efficace.
