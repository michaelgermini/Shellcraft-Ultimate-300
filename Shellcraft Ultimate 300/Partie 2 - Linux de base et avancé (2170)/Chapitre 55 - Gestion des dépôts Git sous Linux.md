# Chapitre 55 - Gestion des dépôts Git sous Linux

## Table des matières
- [Introduction](#introduction)
- [Installation et configuration de Git](#installation-et-configuration-de-git)
- [Workflows Git avancés](#workflows-git-avancés)
- [Intégration avec les outils système](#intégration-avec-les-outils-système)
- [Gestion des gros dépôts](#gestion-des-gros-dépôts)
- [Sécurité et bonnes pratiques](#sécurité-et-bonnes-pratiques)
- [Automation et hooks](#automation-et-hooks)
- [Outils complémentaires](#outils-complémentaires)
- [Conclusion](#conclusion)

## Introduction

Git constitue bien plus qu'un simple système de contrôle de version sous Linux ; c'est un écosystème complet qui s'intègre profondément avec les workflows de développement et d'administration système. Maîtriser Git permet de gérer efficacement les configurations, les scripts, et même les infrastructures as code.

Imaginez Git comme une machine à remonter le temps sophistiquée : elle enregistre chaque changement, permet de revenir à n'importe quel état passé, de créer des réalités alternatives (branches), et de fusionner intelligemment les différentes lignes temporelles.

## Installation et configuration de Git

### Installation

**Installation sur différentes distributions** :
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install git

# Fedora/CentOS/RHEL
sudo dnf install git
# ou
sudo yum install git

# Arch Linux
sudo pacman -S git

# openSUSE
sudo zypper install git

# Vérifier l'installation
git --version
```

**Installation depuis les sources** :
```bash
# Dépendances
sudo apt install libcurl4-openssl-dev libexpat1-dev gettext libz-dev libssl-dev

# Télécharger et compiler
cd /tmp
wget https://github.com/git/git/archive/v2.42.0.tar.gz
tar -xzf v2.42.0.tar.gz
cd git-2.42.0
make configure
./configure --prefix=/usr/local
make all
sudo make install
```

### Configuration initiale

**Configuration globale** :
```bash
# Identité
git config --global user.name "Votre Nom"
git config --global user.email "votre.email@example.com"

# Éditeur par défaut
git config --global core.editor "vim"
# ou
git config --global core.editor "code --wait"  # VS Code

# Vérifier la configuration
git config --list
git config --global --list
```

**Configuration système** :
```bash
# Configuration pour tous les utilisateurs
sudo git config --system user.name "System User"
sudo git config --system core.editor "vim"

# Fichier de configuration système
/etc/gitconfig
```

**Configuration locale (par dépôt)** :
```bash
# Dans un dépôt spécifique
cd /path/to/repo
git config user.name "Nom pour ce projet"
git config user.email "email@project.com"

# Fichier de configuration locale
.git/config
```

### Configuration avancée

**Options utiles** :
```bash
# Coloration de la sortie
git config --global color.ui auto

# Alias pratiques
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.unstage 'reset HEAD --'
git config --global alias.last 'log -1 HEAD'
git config --global alias.visual '!gitk'

# Gestion des fins de ligne
git config --global core.autocrlf input        # Linux/macOS
git config --global core.autocrlf true         # Windows

# Gestion des espaces
git config --global core.whitespace trailing-space,space-before-tab

# Taille maximale des fichiers
git config --global core.bigFileThreshold 50m

# Compression
git config --global core.compression 9
```

**Configuration SSH** :
```bash
# Générer une clé SSH pour Git
ssh-keygen -t ed25519 -C "git@example.com" -f ~/.ssh/id_ed25519_git

# Ajouter à l'agent SSH
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_git

# Configurer Git pour utiliser cette clé
# ~/.ssh/config
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_git
```

## Workflows Git avancés

### Modèles de workflow

**Git Flow** :
```bash
# Structure des branches
main          # Production
develop       # Développement
feature/*     # Nouvelles fonctionnalités
release/*     # Préparation de release
hotfix/*      # Corrections urgentes

# Créer une feature
git checkout -b feature/nouvelle-fonctionnalite develop
# ... développement ...
git checkout develop
git merge --no-ff feature/nouvelle-fonctionnalite
git branch -d feature/nouvelle-fonctionnalite
```

**GitHub Flow** :
```bash
# Branche principale + branches de feature
main          # Toujours déployable
feature/*     # Branches de développement

# Workflow simple
git checkout -b feature/mon-feature main
# ... développement ...
git push origin feature/mon-feature
# Pull Request sur GitHub
git checkout main
git merge feature/mon-feature
```

**GitLab Flow** :
```bash
# Avec environnements
main → staging → production

# Branches de feature vers main
# main vers staging (automatique)
# staging vers production (manuel)
```

### Gestion avancée des branches

**Branches de suivi** :
```bash
# Créer une branche qui suit une branche distante
git checkout -b local-branch origin/remote-branch
git checkout --track origin/remote-branch

# Configurer le suivi après création
git branch --set-upstream-to=origin/remote-branch local-branch
```

**Rebase interactif** :
```bash
# Réorganiser les commits
git rebase -i HEAD~3

# Options dans l'éditeur:
# pick : garder le commit
# reword : modifier le message
# edit : modifier le commit
# squash : fusionner avec le précédent
# drop : supprimer le commit

# Réécrire l'historique
git rebase -i origin/main
```

**Cherry-pick** :
```bash
# Appliquer un commit spécifique
git cherry-pick commit-hash

# Appliquer plusieurs commits
git cherry-pick commit1 commit2 commit3

# Appliquer une plage
git cherry-pick commit1^..commit2
```

### Gestion des tags

**Création et gestion** :
```bash
# Créer un tag annoté
git tag -a v1.0.0 -m "Version 1.0.0"

# Créer un tag léger
git tag v1.0.0

# Taguer un commit spécifique
git tag -a v1.0.0 commit-hash

# Lister les tags
git tag
git tag -l "v1.*"

# Voir les détails d'un tag
git show v1.0.0

# Pousser les tags
git push origin v1.0.0
git push origin --tags  # Tous les tags
```

**Tags signés** :
```bash
# Créer un tag signé
git tag -s v1.0.0 -m "Version signée"

# Vérifier un tag signé
git tag -v v1.0.0
```

## Intégration avec les outils système

### Git avec les scripts système

**Versionner les configurations** :
```bash
# Créer un dépôt pour les dotfiles
cd ~
git init --bare .dotfiles
alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles --work-tree=$HOME'

# Ajouter des fichiers
dotfiles add ~/.bashrc ~/.vimrc
dotfiles commit -m "Ajouter configurations"

# Ignorer les fichiers non suivis
dotfiles config --local status.showUntrackedFiles no
```

**Gestion de configuration système** :
```bash
# Versionner /etc
sudo git init /etc
cd /etc
sudo git add .
sudo git commit -m "Configuration système initiale"

# Créer un hook pour commits automatiques
# .git/hooks/post-commit
#!/bin/bash
# Sauvegarder automatiquement
```

### Intégration avec cron

**Sauvegardes automatiques** :
```bash
#!/bin/bash
# backup_git_repos.sh

REPOS_DIR="/home/user/repositories"
BACKUP_DIR="/backups/git"

for repo in "$REPOS_DIR"/*; do
    if [ -d "$repo/.git" ]; then
        repo_name=$(basename "$repo")
        echo "Sauvegarde de $repo_name"
        
        # Créer une archive
        cd "$repo"
        git bundle create "${BACKUP_DIR}/${repo_name}.bundle" --all
        
        # Ou cloner vers backup
        git clone --mirror "$repo" "${BACKUP_DIR}/${repo_name}.git"
    fi
done
```

**Cron job** :
```bash
# Crontab
0 2 * * * /usr/local/bin/backup_git_repos.sh
```

### Intégration avec systemd

**Service Git pour synchronisation** :
```bash
# /etc/systemd/system/git-sync.service
[Unit]
Description=Git Repository Synchronization
After=network.target

[Service]
Type=oneshot
User=gituser
ExecStart=/usr/local/bin/sync_git_repos.sh

# Timer
# /etc/systemd/system/git-sync.timer
[Unit]
Description=Git Sync Timer

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

## Gestion des gros dépôts

### Optimisation pour gros dépôts

**Shallow clone** :
```bash
# Cloner seulement les derniers commits
git clone --depth 1 https://github.com/user/repo.git

# Augmenter la profondeur
git fetch --unshallow

# Shallow pour une branche spécifique
git clone --depth 1 --branch main https://github.com/user/repo.git
```

**Partial clone** :
```bash
# Cloner sans objets blob
git clone --filter=blob:none https://github.com/user/repo.git

# Télécharger les blobs à la demande
git checkout branch-name
```

**Git LFS (Large File Storage)** :
```bash
# Installation
sudo apt install git-lfs

# Initialiser dans un dépôt
git lfs install

# Suivre les gros fichiers
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.mp4"

# Voir les fichiers suivis
git lfs ls-files
```

### Nettoyage et maintenance

**Nettoyage du dépôt** :
```bash
# Nettoyer les fichiers non suivis
git clean -n      # Dry run
git clean -f      # Supprimer les fichiers
git clean -fd     # Supprimer fichiers et répertoires

# Nettoyer les références
git remote prune origin

# Nettoyer les objets non référencés
git gc --prune=now

# Nettoyage agressif
git gc --aggressive --prune=now
```

**Réduction de la taille** :
```bash
# Trouver les gros fichiers
git rev-list --objects --all | \
    git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | \
    awk '/^blob/ {print substr($0,6)}' | \
    sort --numeric-sort --key=2 | \
    tail -10

# Supprimer un fichier de l'historique
git filter-branch --force --index-filter \
    "git rm --cached --ignore-unmatch large-file.zip" \
    --prune-empty --tag-name-filter cat -- --all

# Alternative moderne (git-filter-repo)
git filter-repo --path large-file.zip --invert-paths
```

## Sécurité et bonnes pratiques

### Sécurité des dépôts

**Ne jamais commiter les secrets** :
```bash
# .gitignore
*.key
*.pem
*.env
secrets/
config/secrets.yaml

# Vérifier avant de commiter
git diff --cached

# Si déjà commité, supprimer de l'historique
git filter-branch --force --index-filter \
    "git rm --cached --ignore-unmatch secrets.txt" \
    --prune-empty --tag-name-filter cat -- --all
```

**Signatures GPG** :
```bash
# Configurer GPG
gpg --gen-key
gpg --list-secret-keys --keyid-format LONG

# Configurer Git pour signer
git config --global user.signingkey YOUR_KEY_ID
git config --global commit.gpgsign true

# Signer un commit
git commit -S -m "Message signé"

# Signer un tag
git tag -s v1.0.0 -m "Tag signé"
```

**Permissions des dépôts** :
```bash
# Dépôt privé
chmod -R 700 .git

# Dépôt partagé
chmod -R 775 .git
chgrp -R developers .git
```

### Bonnes pratiques

**Messages de commit** :
```bash
# Format conventionnel
git commit -m "feat: ajouter nouvelle fonctionnalité"
git commit -m "fix: corriger bug de connexion"
git commit -m "docs: mettre à jour README"
git commit -m "refactor: réorganiser le code"
git commit -m "test: ajouter tests unitaires"

# Types: feat, fix, docs, style, refactor, test, chore
```

**.gitignore efficace** :
```bash
# .gitignore pour projets Linux
# Fichiers système
.DS_Store
Thumbs.db
*~

# Fichiers de build
build/
dist/
*.o
*.so
*.a

# Fichiers temporaires
*.tmp
*.log
*.swp
*.swo

# Environnements virtuels
venv/
env/
.venv

# IDE
.vscode/
.idea/
*.sublime-*

# Secrets
.env
*.key
*.pem
secrets/
```

## Automation et hooks

### Hooks Git

**Hooks côté client** :
```bash
# .git/hooks/pre-commit
#!/bin/bash
# Vérifier le code avant commit
if ! npm test; then
    echo "Tests échoués, commit annulé"
    exit 1
fi

# .git/hooks/pre-push
#!/bin/bash
# Vérifier avant push
echo "Exécution des tests complets..."
npm run test:full
```

**Hooks côté serveur** :
```bash
# hooks/update (sur le serveur)
#!/bin/bash
# Vérifier les permissions
refname="$1"
oldrev="$2"
newrev="$3"

# Empêcher push sur main
if [ "$refname" = "refs/heads/main" ]; then
    echo "Push direct sur main interdit"
    exit 1
fi
```

**Hooks utiles** :
```bash
# post-commit : Notification
#!/bin/bash
echo "Commit créé: $(git log -1 --pretty=format:'%h - %s')" | \
    mail -s "Nouveau commit" team@example.com

# post-receive : Déploiement automatique
#!/bin/bash
cd /var/www/app
git pull origin main
systemctl restart app
```

### Scripts d'automatisation

**Script de synchronisation** :
```bash
#!/bin/bash
# sync_repos.sh

REPOS=(
    "/home/user/project1"
    "/home/user/project2"
)

for repo in "${REPOS[@]}"; do
    cd "$repo" || continue
    
    echo "Synchronisation de $(basename "$repo")"
    
    # Pull les changements
    git fetch origin
    git pull origin main
    
    # Push les commits locaux
    if [ -n "$(git status -porcelain)" ]; then
        git add .
        git commit -m "Auto-sync: $(date)"
        git push origin main
    fi
done
```

**Script de backup** :
```bash
#!/bin/bash
# backup_git.sh

BACKUP_DIR="/backups/git/$(date +%Y%m%d)"
mkdir -p "$BACKUP_DIR"

find /home -name ".git" -type d | while read gitdir; do
    repodir=$(dirname "$gitdir")
    reponame=$(basename "$repodir")
    
    echo "Backup de $reponame"
    git clone --mirror "$repodir" "${BACKUP_DIR}/${reponame}.git"
done
```

## Outils complémentaires

### Outils en ligne de commande

**tig** : Interface texte pour Git
```bash
# Installation
sudo apt install tig

# Utilisation
tig                    # Vue d'ensemble
tig status            # Statut
tig log               # Historique
tig blame             # Blame
```

**lazygit** : Interface TUI moderne
```bash
# Installation
# Via snap
sudo snap install lazygit

# Via GitHub releases
wget https://github.com/jesseduffield/lazygit/releases/download/v0.40.2/lazygit_0.40.2_Linux_x86_64.tar.gz
tar -xzf lazygit_0.40.2_Linux_x86_64.tar.gz
sudo mv lazygit /usr/local/bin/
```

**git-extras** : Commandes supplémentaires
```bash
# Installation
sudo apt install git-extras

# Commandes utiles
git ignore            # Ajouter à .gitignore
git summary           # Résumé du dépôt
git effort            # Effort par fichier
git line-summary      # Statistiques par ligne
```

### Intégration avec les outils système

**Git avec vim** :
```bash
# Plugins populaires
# vim-fugitive : Interface Git dans Vim
# vim-gitgutter : Afficher les changements
```

**Git avec les scripts** :
```bash
# Fonction pour informations Git dans le prompt
git_info() {
    local branch=$(git branch --show-current 2>/dev/null)
    if [ -n "$branch" ]; then
        local status=$(git status --porcelain 2>/dev/null | wc -l)
        if [ "$status" -gt 0 ]; then
            echo " [$branch*]"
        else
            echo " [$branch]"
        fi
    fi
}

# Utilisation dans PS1
PS1='\u@\h:\w$(git_info)\$ '
```

## Conclusion

Git sous Linux n'est pas seulement un outil de développement, mais un système fondamental pour la gestion de configuration, l'automatisation système, et la collaboration. La maîtrise de Git - des workflows de base aux techniques avancées - est essentielle pour tout administrateur système moderne.

Les techniques apprises ici - gestion de gros dépôts, hooks d'automatisation, intégration système - permettent de créer des workflows efficaces et automatisés. Git devient alors non seulement un outil de versionnement, mais un composant central de l'infrastructure as code et de l'automatisation DevOps.

Dans le prochain chapitre, nous explorerons les projets pratiques Linux, appliquant toutes les compétences acquises dans des scénarios réels et complexes.