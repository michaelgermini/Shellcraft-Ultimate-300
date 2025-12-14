# Chapitre 223 - Git branching, merge et rebase

## Table des matières
- [Introduction](#introduction)
- [Branches Git](#branches-git)
- [Merge et stratégies](#merge-et-stratégies)
- [Rebase avancé](#rebase-avancé)
- [Workflows de branching](#workflows-de-branching)
- [Conclusion](#conclusion)

## Introduction

Les branches Git sont au cœur de la collaboration moderne. Comprendre le branching, le merge, et le rebase est essentiel pour travailler efficacement en équipe et maintenir un historique propre.

Imaginez les branches Git comme des lignes temporelles parallèles : chaque branche représente une version alternative de votre projet, et vous pouvez fusionner ces lignes temporelles ou réécrire l'histoire selon vos besoins.

## Branches Git

### Création et gestion de branches

**Commandes essentielles** :
```bash
#!/bin/bash
# Gestion complète des branches Git

# Créer une nouvelle branche
create_branch() {
    local branch_name="$1"
    local base_branch="${2:-main}"
    
    # Créer et basculer sur la nouvelle branche
    git checkout -b "$branch_name" "$base_branch"
    
    echo "Branche $branch_name créée depuis $base_branch"
}

# Lister toutes les branches
list_branches() {
    echo "=== Branches locales ==="
    git branch
    
    echo
    echo "=== Branches distantes ==="
    git branch -r
    
    echo
    echo "=== Toutes les branches ==="
    git branch -a
}

# Supprimer une branche
delete_branch() {
    local branch_name="$1"
    local force="${2:-false}"
    
    if [ "$force" == "true" ]; then
        git branch -D "$branch_name"
    else
        git branch -d "$branch_name"
    fi
}

# Renommer une branche
rename_branch() {
    local old_name="$1"
    local new_name="$2"
    
    git branch -m "$old_name" "$new_name"
    
    # Si la branche était suivie à distance
    if git rev-parse --verify "origin/$old_name" &>/dev/null; then
        git push origin --delete "$old_name"
        git push origin -u "$new_name"
    fi
}
```

## Merge et stratégies

### Stratégies de merge

**Différentes stratégies** :
```bash
#!/bin/bash
# Stratégies de merge Git

# Merge simple (fast-forward si possible)
simple_merge() {
    local source_branch="$1"
    local target_branch="${2:-main}"
    
    git checkout "$target_branch"
    git merge "$source_branch"
}

# Merge avec commit de merge explicite
merge_with_commit() {
    local source_branch="$1"
    local target_branch="${2:-main}"
    
    git checkout "$target_branch"
    git merge --no-ff "$source_branch" -m "Merge $source_branch into $target_branch"
}

# Merge avec squash (combine tous les commits en un)
squash_merge() {
    local source_branch="$1"
    local target_branch="${2:-main}"
    
    git checkout "$target_branch"
    git merge --squash "$source_branch"
    git commit -m "Squash merge: $source_branch"
}

# Résoudre les conflits de merge
resolve_merge_conflicts() {
    echo "=== Résolution de conflits de merge ==="
    
    # Voir les fichiers en conflit
    git diff --name-only --diff-filter=U
    
    # Pour chaque fichier en conflit
    git diff --name-only --diff-filter=U | while read -r file; do
        echo "Conflit dans: $file"
        echo "Éditez le fichier pour résoudre les conflits"
        echo "Puis: git add $file"
    done
    
    # Une fois tous les conflits résolus
    echo "Pour finaliser: git commit"
}
```

## Rebase avancé

### Rebase interactif

**Rebase interactif** :
```bash
#!/bin/bash
# Rebase interactif avancé

# Rebase interactif des 5 derniers commits
interactive_rebase() {
    local commits="${1:-5}"
    
    git rebase -i HEAD~"$commits"
}

# Rebase sur une branche distante
rebase_onto_remote() {
    local branch="${1:-main}"
    
    git fetch origin
    git rebase "origin/$branch"
}

# Rebase avec auto-stash
rebase_with_stash() {
    local target_branch="${1:-main}"
    
    git stash
    git rebase "$target_branch"
    git stash pop
}

# Continuer après résolution de conflit
continue_rebase() {
    # Après avoir résolu les conflits
    git add .
    git rebase --continue
}

# Abandonner un rebase
abort_rebase() {
    git rebase --abort
}
```

## Workflows de branching

### Git Flow

**Workflow Git Flow** :
```bash
#!/bin/bash
# Workflow Git Flow

setup_gitflow() {
    # Branches principales
    git checkout -b develop main
    
    echo "Git Flow configuré"
}

# Créer une feature branch
create_feature() {
    local feature_name="$1"
    
    git checkout develop
    git pull origin develop
    git checkout -b "feature/$feature_name"
    
    echo "Feature branch créée: feature/$feature_name"
}

# Finir une feature
finish_feature() {
    local feature_name="$1"
    
    git checkout develop
    git merge --no-ff "feature/$feature_name"
    git branch -d "feature/$feature_name"
    git push origin develop
}

# Créer une release
create_release() {
    local version="$1"
    
    git checkout develop
    git checkout -b "release/$version"
    
    echo "Release branch créée: release/$version"
}

# Finir une release
finish_release() {
    local version="$1"
    
    git checkout main
    git merge --no-ff "release/$version"
    git tag -a "v$version" -m "Release $version"
    git checkout develop
    git merge --no-ff "release/$version"
    git branch -d "release/$version"
}
```

## Conclusion

Le branching, le merge, et le rebase sont des outils puissants pour gérer l'évolution de votre code. Chaque stratégie a ses avantages : choisissez celle qui correspond à votre workflow et à votre équipe.

Dans le chapitre suivant, nous explorerons les hooks Git et l'automatisation, découvrant comment automatiser des actions lors des événements Git.

