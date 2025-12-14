# Chapitre 227 - Hooks Git et automatisation

## Table des matières
- [Introduction](#introduction)
- [Hooks Git de base](#hooks-git-de-base)
- [Hooks avancés](#hooks-avancés)
- [Automatisation avec hooks](#automatisation-avec-hooks)
- [Hooks côté serveur](#hooks-côté-serveur)
- [Conclusion](#conclusion)

## Introduction

Les hooks Git permettent d'exécuter automatiquement des scripts à des moments clés du cycle de vie Git. Ils sont essentiels pour automatiser les vérifications, les tests, et les déploiements.

Imaginez les hooks Git comme des gardiens automatiques : ils surveillent chaque action Git et peuvent intervenir pour valider, transformer, ou notifier selon vos règles.

## Hooks Git de base

### Hooks côté client

**Hooks pré-commit** :
```bash
#!/bin/bash
# .git/hooks/pre-commit
# Hook exécuté avant chaque commit

set -e

echo "=== Pre-commit checks ==="

# Vérifier la syntaxe Bash
echo "Vérification de la syntaxe..."
find . -name "*.sh" -not -path "./.git/*" | while read -r file; do
    if ! bash -n "$file"; then
        echo "Erreur de syntaxe dans $file"
        exit 1
    fi
done

# Exécuter les tests
if [ -d "tests" ]; then
    echo "Exécution des tests..."
    if ! bash tests/run_tests.sh; then
        echo "Les tests ont échoué"
        exit 1
    fi
fi

# Vérifier avec ShellCheck si disponible
if command -v shellcheck &> /dev/null; then
    echo "Vérification ShellCheck..."
    find . -name "*.sh" -not -path "./.git/*" | while read -r file; do
        shellcheck "$file" || exit 1
    done
fi

echo "✓ Pre-commit checks passés"
exit 0
```

**Hook commit-msg** :
```bash
#!/bin/bash
# .git/hooks/commit-msg
# Valide le format du message de commit

commit_msg_file="$1"
commit_msg=$(cat "$commit_msg_file")

# Format: type(scope): description
# Types: feat, fix, docs, style, refactor, test, chore

if ! echo "$commit_msg" | grep -qE '^(feat|fix|docs|style|refactor|test|chore)(\(.+\))?: .+'; then
    echo "Format de commit invalide"
    echo "Format attendu: type(scope): description"
    echo "Types: feat, fix, docs, style, refactor, test, chore"
    exit 1
fi

exit 0
```

## Hooks avancés

### Pre-push hook

**Hook pre-push** :
```bash
#!/bin/bash
# .git/hooks/pre-push
# Exécuté avant chaque push

set -e

echo "=== Pre-push checks ==="

# Vérifier que tous les tests passent
if [ -d "tests" ]; then
    echo "Exécution de la suite de tests complète..."
    bash tests/run_all_tests.sh || {
        echo "Les tests ont échoué. Push annulé."
        exit 1
    }
fi

# Vérifier la couverture de code
if command -v coverage &> /dev/null; then
    echo "Vérification de la couverture..."
    coverage run tests/run_all_tests.sh
    coverage report --fail-under=80 || {
        echo "Couverture insuffisante. Push annulé."
        exit 1
    }
fi

echo "✓ Pre-push checks passés"
exit 0
```

### Post-commit hook

**Hook post-commit** :
```bash
#!/bin/bash
# .git/hooks/post-commit
# Exécuté après chaque commit

# Envoyer une notification
if command -v notify-send &> /dev/null; then
    notify-send "Git Commit" "Commit créé: $(git log -1 --pretty=format:'%s')"
fi

# Logger le commit
echo "$(date): $(git log -1 --pretty=format:'%h - %s')" >> ~/.git_commits.log
```

## Automatisation avec hooks

### Déploiement automatique

**Hook post-receive (serveur)** :
```bash
#!/bin/bash
# .git/hooks/post-receive
# Déploiement automatique après push

set -e

DEPLOY_DIR="/var/www/app"
GIT_DIR="$(pwd)"

echo "=== Déploiement automatique ==="

# Vérifier la branche
while read oldrev newrev refname; do
    branch=$(git rev-parse --symbolic --abbrev-ref $refname)
    
    if [ "$branch" == "main" ]; then
        echo "Déploiement de la branche main..."
        
        # Créer un work tree temporaire
        TEMP_DIR=$(mktemp -d)
        git worktree add "$TEMP_DIR" "$branch"
        
        # Exécuter les tests
        cd "$TEMP_DIR"
        if [ -f "tests/run_tests.sh" ]; then
            bash tests/run_tests.sh || {
                echo "Tests échoués. Déploiement annulé."
                git worktree remove "$TEMP_DIR"
                exit 1
            }
        fi
        
        # Déployer
        rsync -av --exclude='.git' "$TEMP_DIR/" "$DEPLOY_DIR/"
        
        # Nettoyer
        git worktree remove "$TEMP_DIR"
        
        # Redémarrer les services si nécessaire
        if [ -f "$DEPLOY_DIR/restart.sh" ]; then
            bash "$DEPLOY_DIR/restart.sh"
        fi
        
        echo "✓ Déploiement terminé"
    fi
done
```

## Hooks côté serveur

### Validation côté serveur

**Hook update (serveur)** :
```bash
#!/bin/bash
# .git/hooks/update
# Valide les pushes avant acceptation

refname="$1"
oldrev="$2"
newrev="$3"

# Empêcher la suppression de la branche main
if [ "$refname" == "refs/heads/main" ] && [ "$newrev" == "0000000000000000000000000000000000000000" ]; then
    echo "Erreur: La branche main ne peut pas être supprimée"
    exit 1
fi

# Vérifier la taille des fichiers
git diff --name-only "$oldrev" "$newrev" | while read -r file; do
    size=$(git cat-file -s "$newrev:$file" 2>/dev/null || echo "0")
    if [ "$size" -gt 104857600 ]; then  # 100MB
        echo "Erreur: Le fichier $file est trop volumineux (>100MB)"
        exit 1
    fi
done

exit 0
```

## Conclusion

Les hooks Git sont des outils puissants pour automatiser et valider votre workflow Git. Utilisez-les pour garantir la qualité du code, automatiser les déploiements, et maintenir la cohérence de votre projet.

Dans le chapitre suivant, nous explorerons Docker Compose et les multi-conteneurs, découvrant comment orchestrer des applications complexes avec plusieurs conteneurs.

