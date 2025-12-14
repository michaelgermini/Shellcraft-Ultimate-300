# Chapitre 237 - Intégration Git Docker Scripts

## Table des matières
- [Introduction](#introduction)
- [Workflow intégré](#workflow-intégré)
- [Automatisation complète](#automatisation-complète)
- [Scripts de déploiement](#scripts-de-déploiement)
- [Conclusion](#conclusion)

## Introduction

L'intégration de Git, Docker et des scripts crée un workflow DevOps complet où chaque commit peut déclencher un build, des tests, et un déploiement automatique.

Imaginez cette intégration comme une chaîne de production automatisée : Git apporte les changements, Docker les conteneurise, et les scripts orchestrent le tout.

## Workflow intégré

### Pipeline complet

**Script de workflow intégré** :
```bash
#!/bin/bash
# Workflow intégré Git + Docker + Scripts

set -euo pipefail

WORKFLOW_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_NAME="${PROJECT_NAME:-myapp}"
DOCKER_REGISTRY="${DOCKER_REGISTRY:-registry.example.com}"

# Workflow complet
complete_workflow() {
    local branch="${1:-$(git rev-parse --abbrev-ref HEAD)}"
    
    echo "=== Workflow intégré ==="
    echo "Branche: $branch"
    echo
    
    # 1. Vérifier Git
    echo "1. Vérification Git..."
    check_git_status
    
    # 2. Build Docker
    echo "2. Build Docker..."
    build_docker_image "$branch"
    
    # 3. Tests
    echo "3. Exécution des tests..."
    run_tests_in_container
    
    # 4. Push vers registry
    echo "4. Push vers registry..."
    push_to_registry "$branch"
    
    # 5. Déploiement (si main)
    if [ "$branch" == "main" ]; then
        echo "5. Déploiement..."
        deploy_to_production
    fi
    
    echo "✓ Workflow terminé"
}

check_git_status() {
    if [ -n "$(git status --porcelain)" ]; then
        echo "⚠ Changements non commités détectés"
        read -p "Continuer quand même? (y/N): " confirm
        if [[ ! "$confirm" =~ ^[Yy]$ ]]; then
            exit 1
        fi
    fi
    
    git fetch origin
    local commits_behind=$(git rev-list --count HEAD..origin/"$(git rev-parse --abbrev-ref HEAD)")
    if [ "$commits_behind" -gt 0 ]; then
        echo "⚠ Vous êtes en retard de $commits_behind commit(s)"
    fi
}

build_docker_image() {
    local branch="$1"
    local commit_sha=$(git rev-parse --short HEAD)
    local image_tag="${PROJECT_NAME}:${branch}-${commit_sha}"
    local latest_tag="${PROJECT_NAME}:${branch}-latest"
    
    echo "Build de l'image: $image_tag"
    docker build -t "$image_tag" -t "$latest_tag" .
    
    echo "✓ Image construite: $image_tag"
}

run_tests_in_container() {
    local test_image="${PROJECT_NAME}:test"
    
    # Build image de test
    docker build -f Dockerfile.test -t "$test_image" .
    
    # Exécuter les tests
    docker run --rm "$test_image" ./run_tests.sh
    
    echo "✓ Tests passés"
}

push_to_registry() {
    local branch="$1"
    local commit_sha=$(git rev-parse --short HEAD)
    local image_tag="${PROJECT_NAME}:${branch}-${commit_sha}"
    local latest_tag="${PROJECT_NAME}:${branch}-latest"
    
    docker tag "$image_tag" "${DOCKER_REGISTRY}/${image_tag}"
    docker tag "$latest_tag" "${DOCKER_REGISTRY}/${latest_tag}"
    
    docker push "${DOCKER_REGISTRY}/${image_tag}"
    docker push "${DOCKER_REGISTRY}/${latest_tag}"
    
    echo "✓ Images poussées vers $DOCKER_REGISTRY"
}

deploy_to_production() {
    local commit_sha=$(git rev-parse --short HEAD)
    local image="${DOCKER_REGISTRY}/${PROJECT_NAME}:main-${commit_sha}"
    
    echo "Déploiement de $image en production..."
    
    # Mettre à jour docker-compose.yml avec la nouvelle image
    sed -i "s|image:.*|image: $image|" docker-compose.prod.yml
    
    # Déployer
    docker-compose -f docker-compose.prod.yml up -d
    
    # Vérifier la santé
    sleep 10
    if check_health; then
        echo "✓ Déploiement réussi"
    else
        echo "✗ Échec du déploiement, rollback..."
        rollback_deployment
        exit 1
    fi
}
```

## Automatisation complète

### Hook Git pour déclencher le workflow

**Post-commit hook** :
```bash
#!/bin/bash
# .git/hooks/post-commit
# Déclenche le workflow après chaque commit

WORKFLOW_SCRIPT="$(git rev-parse --show-toplevel)/scripts/workflow.sh"

if [ -f "$WORKFLOW_SCRIPT" ]; then
    branch=$(git rev-parse --abbrev-ref HEAD)
    
    # Exécuter le workflow en arrière-plan
    nohup bash "$WORKFLOW_SCRIPT" "$branch" > /tmp/workflow.log 2>&1 &
    
    echo "Workflow démarré en arrière-plan (log: /tmp/workflow.log)"
fi
```

## Scripts de déploiement

### Script de déploiement complet

**Déploiement automatisé** :
```bash
#!/bin/bash
# Script de déploiement complet

deploy() {
    local environment="${1:-staging}"
    local version="${2:-latest}"
    
    echo "=== Déploiement $environment ==="
    
    # Charger la configuration
    load_config "$environment"
    
    # Pull la dernière image
    docker pull "${DOCKER_REGISTRY}/${PROJECT_NAME}:${version}"
    
    # Arrêter les anciens conteneurs
    docker-compose -f "docker-compose.${environment}.yml" down
    
    # Démarrer les nouveaux
    IMAGE_TAG="$version" docker-compose -f "docker-compose.${environment}.yml" up -d
    
    # Vérifier la santé
    wait_for_health
    
    echo "✓ Déploiement $environment terminé"
}

load_config() {
    local env="$1"
    source "config/${env}.env"
}

wait_for_health() {
    local max_attempts=30
    local attempt=0
    
    while [ $attempt -lt $max_attempts ]; do
        if check_health; then
            return 0
        fi
        attempt=$((attempt + 1))
        sleep 2
    done
    
    return 1
}
```

## Conclusion

L'intégration de Git, Docker et des scripts crée un workflow DevOps puissant et automatisé. Chaque commit peut déclencher un cycle complet de build, test, et déploiement.

Dans le chapitre suivant, nous explorerons le déploiement cloud et les stratégies de déploiement modernes.

