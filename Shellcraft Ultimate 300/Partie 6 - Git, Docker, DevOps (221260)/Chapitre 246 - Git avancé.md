# Chapitre 246 - Git avancé

## Table des matières
- [Introduction](#introduction)
- [Git internals](#git-internals)
- [Références et branches](#références-et-branches)
- [Réécriture d'historique](#réécriture-dhistorique)
- [Conclusion](#conclusion)

## Introduction

Git avancé explore les mécanismes internes de Git, permettant une compréhension profonde et un contrôle total sur votre dépôt.

## Git internals

**Exploration des objets Git** :
```bash
#!/bin/bash
# Exploration des objets Git

# Voir les objets
explore_objects() {
    local commit="$1"
    
    # Hash du commit
    local commit_hash=$(git rev-parse "$commit")
    echo "Commit hash: $commit_hash"
    
    # Voir le contenu
    git cat-file -p "$commit_hash"
    
    # Voir l'arbre
    local tree_hash=$(git cat-file -p "$commit_hash" | grep tree | awk '{print $2}')
    git cat-file -p "$tree_hash"
}

# Explorer les blobs
explore_blobs() {
    git rev-list --objects --all | \
    while read -r hash path; do
        echo "$hash $path"
        git cat-file -t "$hash"
    done
}
```

## Références et branches

**Gestion avancée des références** :
```bash
#!/bin/bash
# Références Git avancées

# Voir toutes les références
list_all_refs() {
    git for-each-ref --format='%(refname) %(objectname) %(subject)'
}

# Créer une référence personnalisée
create_custom_ref() {
    local ref_name="$1"
    local commit="$2"
    
    git update-ref "refs/custom/$ref_name" "$commit"
}

# Supprimer une référence
delete_ref() {
    local ref="$1"
    git update-ref -d "$ref"
}
```

## Réécriture d'historique

**Réécriture avancée** :
```bash
#!/bin/bash
# Réécriture d'historique avancée

# Filter-branch pour modifier l'historique
filter_branch_example() {
    # Changer l'email de tous les commits
    git filter-branch --env-filter '
        if [ "$GIT_AUTHOR_EMAIL" = "old@email.com" ]; then
            export GIT_AUTHOR_EMAIL="new@email.com"
        fi
    ' --tag-name-filter cat -- --all
}

# Réécrire les messages de commit
rewrite_commit_messages() {
    git filter-branch -f --msg-filter '
        sed "s/old message/new message/g"
    ' -- --all
}
```

## Conclusion

Git avancé donne un contrôle total sur votre dépôt et permet des opérations complexes sur l'historique.

