# Chapitre 221 - Introduction à Git et au contrôle de version

> "Git n'est pas seulement un système de contrôle de version - c'est une machine à explorer le temps du code, permettant aux développeurs de voyager dans l'histoire de leur création et de collaborer à travers les dimensions." - Prophète Git

## Introduction : Git comme fondation du développement moderne

Dans l'écosystème DevOps, Git représente bien plus qu'un simple système de contrôle de version. C'est l'infrastructure fondamentale qui permet la collaboration à grande échelle, l'automatisation des déploiements, et la traçabilité complète des changements. Maîtriser Git, c'est maîtriser l'art de la collaboration technique et de la gestion des connaissances collectives.

Dans ce chapitre inaugural de notre exploration Git/Docker/DevOps, nous poserons les bases de la philosophie Git, de ses concepts fondamentaux, et de son intégration dans les workflows modernes de développement.

## Section 1 : Philosophie et concepts fondamentaux de Git

### 1.1 Git vs autres systèmes de contrôle de version

**Systèmes centralisés traditionnels (SVN, CVS) :**
- Architecture client-serveur stricte
- Dépôt unique sur un serveur central
- Travail hors ligne limité
- Historique séquentiel

**Git - Distributed Version Control :**
- Architecture distribuée
- Chaque clone est un dépôt complet
- Travail hors ligne natif
- Historique non linéaire (DAG - Directed Acyclic Graph)
- Performance locale exceptionnelle

```bash
# Comparaison des workflows
echo "=== WORKFLOW SVN (Centralisé) ==="
echo "1. Checkout depuis serveur central"
echo "2. Modifications locales"
echo "3. Commit vers serveur (nécessite connexion)"
echo "4. Conflits résolus sur serveur"
echo ""

echo "=== WORKFLOW GIT (Distribué) ==="
echo "1. Clone du dépôt complet"
echo "2. Modifications locales + commits locaux"
echo "3. Push vers remote quand prêt"
echo "4. Pull/merge pour synchroniser"
echo "5. Conflits résolus localement"
```

### 1.2 Le modèle d'objets de Git

Git repose sur quatre types d'objets fondamentaux :

**1. Blobs (Binary Large Objects) :**
- Contenu des fichiers
- Identifiés par hash SHA-1
- Immuables

```bash
# Création d'un blob
echo "Hello World" | git hash-object -w --stdin
# Retourne: ce013625030ba8dba906f756967f9e9ca394464a

# Inspection d'un blob
git cat-file -p ce013625030ba8dba906f756967f9e9ca394464a
```

**2. Trees :**
- Représentent l'état d'un répertoire
- Contiennent références vers blobs et autres trees
- Hiérarchie des fichiers

```bash
# Création d'un tree
git write-tree
# Retourne hash du tree racine

# Inspection d'un tree
git cat-file -p <tree-hash>
```

**3. Commits :**
- Point dans le temps du projet
- Référence vers un tree
- Métadonnées (auteur, date, message)
- Liens vers commits parents

```bash
# Structure d'un commit
git cat-file -p HEAD
# tree d8263ee9861d19c29c6ba8d4a7e1d0a8bcec3f0b
# parent 4e3d49c6c724d5c9b0c8b8b8b8b8b8b8b8b8b8b
# author John Doe <john@example.com> 1640995200 +0000
# committer John Doe <john@example.com> 1640995200 +0000
#
# Initial commit
```

**4. References (Refs) :**
- Pointeurs vers commits
- Branches, tags, HEAD
- Stored in `.git/refs/`

### 1.3 Le staging area (Index)

L'index est l'espace intermédiaire entre le working directory et le repository :

```bash
# États des fichiers dans Git
echo "=== ÉTATS DES FICHIERS GIT ==="
echo "1. Untracked: Fichiers nouveaux, pas dans l'index"
echo "2. Modified: Fichiers modifiés depuis dernier commit"
echo "3. Staged: Fichiers dans l'index, prêts pour commit"
echo "4. Committed: Fichiers sauvegardés dans l'historique"
echo ""

# Vérification du statut
git status --porcelain

# Staging sélectif
echo "Modification 1" > file1.txt
echo "Modification 2" > file2.txt

git add file1.txt  # Stage seulement file1
git status

# Staging partiel (hunks)
echo "Ligne 1" >> file3.txt
echo "Ligne 2" >> file3.txt
echo "Ligne 3" >> file3.txt

git add -p file3.txt  # Staging interactif
```

## Section 2 : Premiers pas avec Git

### 2.1 Configuration initiale

```bash
# Configuration globale
git config --global user.name "John Doe"
git config --global user.email "john.doe@example.com"

# Configuration locale (par dépôt)
git config user.name "John Doe"
git config user.email "john.doe@company.com"

# Configuration utile
git config --global core.editor "nano"
git config --global merge.tool "vimdiff"
git config --global color.ui auto
git config --global alias.st "status -s"
git config --global alias.ci "commit"
git config --global alias.br "branch"
git config --global alias.co "checkout"

# Configuration pour push par défaut
git config --global push.default simple

# Vérification de la configuration
git config --list --show-origin
```

### 2.2 Création et clonage de dépôts

```bash
# Initialisation d'un nouveau dépôt
mkdir my-project
cd my-project
git init

# Création du premier fichier
echo "# My Project" > README.md
git add README.md
git commit -m "Initial commit"

# Clonage d'un dépôt existant
git clone https://github.com/user/repo.git
git clone git@github.com:user/repo.git  # SSH

# Clonage superficiel (pour gros dépôts)
git clone --depth 1 https://github.com/user/repo.git

# Clonage avec historique limité
git clone --shallow-since="2023-01-01" https://github.com/user/repo.git
```

### 2.3 Workflow de base : Add, Commit, Push

```bash
# Vérification du statut
git status

# Ajout de fichiers
git add .                          # Tout ajouter
git add *.txt                      # Fichiers spécifiques
git add -A                         # Tout (nouveau, modifié, supprimé)

# Commit des changements
git commit -m "Description des changements"

# Commit avec message détaillé
git commit -m "Titre du commit

Description détaillée des changements:
- Changement 1
- Changement 2
- Correction bug XYZ"

# Modification du dernier commit (avant push)
git commit --amend -m "Message corrigé"

# Push vers remote
git push origin main               # Branche main
git push origin feature-branch     # Branche spécifique
git push -u origin main            # Premier push avec tracking

# Pull des changements distants
git pull origin main               # Pull + merge automatique
git fetch origin                   # Seulement récupération
git merge origin/main              # Merge manuel
```

### 2.4 Gestion des branches

```bash
# Lister les branches
git branch -a                      # Toutes les branches
git branch -r                      # Branches distantes
git branch --merged                # Branches mergées
git branch --no-merged             # Branches non mergées

# Création de branches
git branch feature-x               # Création locale
git checkout -b feature-y          # Création + checkout
git push -u origin feature-x       # Push avec tracking

# Navigation entre branches
git checkout main                  # Changer de branche
git switch feature-x               # Git 2.23+
git switch -c new-feature          # Création + switch

# Fusion de branches
git merge feature-x                # Merge dans branche courante
git merge --no-ff feature-x        # Merge avec commit de merge
git rebase main                    # Rebase sur main

# Suppression de branches
git branch -d feature-x            # Suppression si mergée
git branch -D feature-x            # Forcer suppression
git push origin --delete feature-x # Suppression distante
```

## Section 3 : Historique et navigation temporelle

### 3.1 Exploration de l'historique

```bash
# Historique des commits
git log                            # Historique complet
git log --oneline                  # Format compact
git log --graph --decorate         # Avec graphe des branches
git log --stat                     # Avec statistiques des fichiers
git log --author="John"            # Par auteur
git log --since="2023-01-01"       # Depuis une date
git log --grep="bug"               # Recherche dans messages

# Différences
git diff                          # Modifications non stagées
git diff --staged                 # Modifications stagées
git diff HEAD~1                   # Diff avec commit précédent
git diff branch1..branch2         # Diff entre branches

# Blame (qui a modifié quoi)
git blame file.txt                # Attribution par ligne
git blame -L 10,20 file.txt       # Lignes spécifiques
```

### 3.2 Time traveling avec Git

```bash
# Références temporelles
git show HEAD                     # Commit actuel
git show HEAD~1                   # Commit précédent
git show HEAD~2                   # Il y a 2 commits
git show HEAD@{1}                 # Position précédente de HEAD
git show main~3                   # 3 commits avant main

# Recherche temporelle
git log --until="2 weeks ago"     # Commits jusqu'à il y a 2 semaines
git log --since="2023-01-01"      # Depuis une date
git log --author-date-order       # Tri par date d'auteur

# Checkout temporel
git checkout HEAD~2              # Aller 2 commits en arrière
git checkout main~5              # Aller à 5 commits avant main
git checkout <commit-hash>       # Aller à un commit spécifique

# Retour au présent
git checkout main                # Retour à la branche principale
git switch -                      # Retour à la branche précédente
```

### 3.3 Gestion des références et tags

```bash
# Tags légers
git tag v1.0                     # Tag sur commit actuel
git tag v0.9 HEAD~1              # Tag sur commit spécifique

# Tags annotés (recommandés)
git tag -a v1.0 -m "Version 1.0 stable"
git tag -a v2.0 -m "Version 2.0 avec nouvelles features"

# Lister les tags
git tag                          # Tous les tags
git tag -l "v1.*"                # Tags correspondants

# Push des tags
git push origin v1.0             # Tag spécifique
git push origin --tags           # Tous les tags

# Suppression de tags
git tag -d v1.0                  # Suppression locale
git push origin :refs/tags/v1.0  # Suppression distante
```

## Section 4 : Collaboration et remotes

### 4.1 Gestion des remotes

```bash
# Lister les remotes
git remote -v

# Ajout de remotes
git remote add origin https://github.com/user/repo.git
git remote add upstream https://github.com/original/repo.git

# Récupération depuis remotes
git fetch origin                 # Récupération sans merge
git fetch --all                  # Tous les remotes
git pull origin main             # Fetch + merge
git pull --rebase origin main    # Fetch + rebase

# Push vers remotes
git push origin main             # Push branche main
git push -u origin feature-x     # Push avec tracking
git push origin --all            # Toutes les branches
git push origin --tags           # Tous les tags

# Gestion des remotes
git remote rename origin upstream
git remote remove origin
git remote set-url origin https://github.com/new/repo.git
```

### 4.2 Gestion des conflits de merge

```bash
# Simulation de conflit
# Modifier le même fichier dans deux branches différentes
git checkout -b branch1
echo "Version 1" > conflict.txt
git add conflict.txt
git commit -m "Version 1"

git checkout -b branch2
echo "Version 2" > conflict.txt
git add conflict.txt
git commit -m "Version 2"

# Tentative de merge
git checkout branch1
git merge branch2
# Résultat: CONFLICT (content): Merge conflict in conflict.txt

# Résolution du conflit
git status                       # Voir fichiers en conflit
cat conflict.txt                 # Voir marqueurs de conflit
# <<<<<<< HEAD
# Version 1
# =======
# Version 2
# >>>>>>> branch2

# Éditer le fichier pour résoudre
echo "Version résolue" > conflict.txt
git add conflict.txt
git commit -m "Résoudre conflit de merge"

# Outils de résolution
git mergetool                    # Outil configuré
git merge --abort                # Annuler le merge
```

### 4.3 Workflows collaboratifs

**Workflow GitHub Flow :**
```bash
# 1. Créer une branche pour la feature
git checkout -b feature/new-feature

# 2. Développer et commiter
echo "Nouvelle feature" > feature.txt
git add feature.txt
git commit -m "Ajouter nouvelle feature"

# 3. Push de la branche
git push -u origin feature/new-feature

# 4. Créer une Pull Request (via interface GitHub)

# 5. Après approbation, merge via interface
# Ou merge local:
git checkout main
git pull origin main
git merge feature/new-feature
git push origin main

# 6. Nettoyage
git branch -d feature/new-feature
git push origin --delete feature/new-feature
```

**Workflow Git Flow :**
```bash
# Installation de git-flow
sudo apt-get install git-flow

# Initialisation
git flow init

# Développement d'une feature
git flow feature start my-feature
# Développement...
git flow feature finish my-feature

# Release
git flow release start 1.0.0
# Tests, corrections...
git flow release finish 1.0.0

# Hotfix
git flow hotfix start urgent-fix
# Correction...
git flow hotfix finish urgent-fix
```

## Section 5 : Automatisation et scripting Git

### 5.1 Hooks Git pour automatisation

```bash
# Installation d'un hook pre-commit
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash

echo "=== PRE-COMMIT HOOK ==="

# Vérifications de qualité du code
if command -v pylint >/dev/null 2>&1; then
    echo "Vérification Python avec pylint..."
    python_files=$(git diff --cached --name-only | grep '\.py$')
    if [[ -n "$python_files" ]]; then
        echo "$python_files" | xargs pylint --errors-only
        if [[ $? -ne 0 ]]; then
            echo "❌ Erreurs pylint détectées. Commit refusé."
            exit 1
        fi
    fi
fi

# Vérification des tests
if [[ -f "package.json" ]]; then
    echo "Exécution des tests JavaScript..."
    npm test
    if [[ $? -ne 0 ]]; then
        echo "❌ Tests échoués. Commit refusé."
        exit 1
    fi
fi

# Vérification de la taille des fichiers
git diff --cached --name-only | while read file; do
    size=$(stat -c%s "$file" 2>/dev/null || stat -f%z "$file" 2>/dev/null)
    if [[ $size -gt 100000000 ]]; then  # 100MB
        echo "❌ Fichier trop volumineux: $file (${size} bytes)"
        exit 1
    fi
done

echo "✅ Toutes les vérifications passées"
EOF

chmod +x .git/hooks/pre-commit

# Hook prepare-commit-msg pour formater les messages
cat > .git/hooks/prepare-commit-msg << 'EOF'
#!/bin/bash

# Ajouter automatiquement le numéro de ticket si dans nom de branche
branch_name=$(git symbolic-ref --short HEAD 2>/dev/null)
ticket_number=$(echo "$branch_name" | grep -oE '([A-Z]+-[0-9]+)')

if [[ -n "$ticket_number" ]]; then
    sed -i "1s/^/$ticket_number: /" "$1"
fi
EOF

chmod +x .git/hooks/prepare-commit-msg
```

### 5.2 Scripting Git avancé

```bash
# Fonction pour analyser l'activité d'un dépôt
analyze_git_activity() {
    local repo_path="${1:-.}"
    local since="${2:-1 month ago}"
    
    cd "$repo_path"
    
    echo "=== ANALYSE D'ACTIVITÉ GIT ==="
    echo "Dépôt: $(pwd)"
    echo "Période: depuis $since"
    echo
    
    # Statistiques générales
    echo "Commits totaux: $(git rev-list --count HEAD)"
    echo "Commits récents: $(git rev-list --count --since="$since" HEAD)"
    echo "Contributeurs actifs: $(git shortlog -sn --since="$since" | wc -l)"
    echo
    
    # Top contributeurs
    echo "Top 5 contributeurs:"
    git shortlog -sn --since="$since" | head -5 | nl
    echo
    
    # Activité par jour
    echo "Activité par jour (derniers 30 jours):"
    git log --since="30 days ago" --pretty=format:"%ad" --date=short | sort | uniq -c | sort -nr | head -10
    echo
    
    # Fichiers les plus modifiés
    echo "Fichiers les plus modifiés:"
    git log --since="$since" --name-only --pretty=format: | sort | uniq -c | sort -nr | head -10
    echo
    
    # Branches actives
    echo "Branches avec commits récents:"
    for branch in $(git branch -r | sed 's|origin/||'); do
        if [[ "$branch" != "HEAD" ]]; then
            commits=$(git rev-list --count --since="$since" "origin/$branch" 2>/dev/null || echo 0)
            if [[ $commits -gt 0 ]]; then
                echo "  $branch: $commits commits"
            fi
        fi
    done
}

# Fonction de nettoyage automatique
git_housekeeping() {
    echo "=== NETTOYAGE GIT ==="
    
    # Suppression des branches mergées localement
    echo "Suppression des branches locales mergées..."
    git branch --merged | grep -v "\*" | grep -v master | grep -v main | xargs -r git branch -d
    
    # Suppression des branches distantes mergées
    echo "Suppression des branches distantes mergées..."
    git fetch -p
    for branch in $(git branch -r --merged | grep -v master | grep -v main | sed 's|origin/||'); do
        git push origin --delete "$branch" 2>/dev/null && echo "  Supprimé: $branch"
    done
    
    # Nettoyage du reflog
    echo "Nettoyage du reflog..."
    git reflog expire --expire=30.days --all
    git gc --prune=30.days --aggressive
    
    echo "✅ Nettoyage terminé"
}

# Fonction de déploiement automatisé
git_deploy() {
    local environment="$1"
    local branch="${2:-main}"
    
    echo "=== DÉPLOIEMENT $environment ==="
    
    # Vérification du statut
    if [[ -n "$(git status --porcelain)" ]]; then
        echo "❌ Working directory pas propre. Commit ou stash les changements."
        return 1
    fi
    
    # Checkout de la branche cible
    git checkout "$branch"
    git pull origin "$branch"
    
    # Tag de déploiement
    local tag_name="${environment}-$(date +%Y%m%d_%H%M%S)"
    git tag "$tag_name"
    git push origin "$tag_name"
    
    echo "✅ Déploiement $environment terminé (tag: $tag_name)"
    
    # Déclenchement du déploiement (exemple avec webhook)
    if [[ -n "$DEPLOY_WEBHOOK_URL" ]]; then
        curl -X POST "$DEPLOY_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -d "{\"environment\": \"$environment\", \"tag\": \"$tag_name\", \"branch\": \"$branch\"}"
    fi
}

# Fonction de monitoring des dépôts
monitor_repositories() {
    local repos_file="$1"
    
    echo "=== MONITORING DES DÉPÔTS ==="
    
    while IFS= read -r repo_url; do
        [[ -z "$repo_url" || "$repo_url" =~ ^# ]] && continue
        
        echo "Vérification: $repo_url"
        
        # Clone temporaire pour analyse
        local temp_dir=$(mktemp -d)
        if git clone --depth 1 "$repo_url" "$temp_dir" 2>/dev/null; then
            cd "$temp_dir"
            
            # Métriques de base
            local commits=$(git rev-list --count HEAD)
            local contributors=$(git shortlog -sn | wc -l)
            local last_commit=$(git log -1 --format=%ci)
            local branch_count=$(git branch -r | wc -l)
            
            echo "  ✅ OK - Commits: $commits, Contributeurs: $contributors, Branches: $branch_count"
            echo "  Dernier commit: $last_commit"
            
            cd - >/dev/null
        else
            echo "  ❌ Échec de clonage"
        fi
        
        rm -rf "$temp_dir"
        echo
    done < "$repos_file"
}

# Exemples d'utilisation
# analyze_git_activity . "2 weeks ago"
# git_housekeeping
# git_deploy production main
# monitor_repositories repos_to_monitor.txt
```

## Conclusion : Git comme langage de collaboration

Git transcende son rôle technique pour devenir le langage fondamental de la collaboration logicielle. Il permet non seulement de versionner le code, mais aussi de capturer l'histoire collaborative des équipes, de faciliter les revues de code, et d'automatiser les processus de développement.

Dans le prochain chapitre, nous explorerons Docker et la conteneurisation, qui révolutionnent la façon dont nous packagons et déployons les applications dans les environnements DevOps modernes.

---

**Exercice pratique :** Créez un workflow Git complet pour une équipe de développement qui inclut :
1. Branches feature avec conventions de nommage
2. Pull requests avec templates
3. Hooks de qualité automatique
4. Scripts de déploiement automatisé
5. Monitoring de l'activité du dépôt

**Challenge avancé :** Développez un système de gestion de versions sémantique intégré à Git qui :
- Analyse automatiquement les commits
- Incrémente les versions selon les conventions semver
- Génère automatiquement les changelogs
- Crée les tags de release
- Intègre avec les outils CI/CD

**Réflexion :** Comment Git transforme-t-il non seulement les pratiques techniques, mais aussi les dynamiques sociales et collaboratives des équipes de développement ?

