# Chapitre 245 - Automatisation de pipelines

## Table des matières
- [Introduction](#introduction)
- [Pipelines complexes](#pipelines-complexes)
- [Orchestration](#orchestration)
- [Gestion d'erreurs](#gestion-derreurs)
- [Conclusion](#conclusion)

## Introduction

L'automatisation de pipelines permet de créer des workflows complexes qui gèrent automatiquement le build, les tests, et le déploiement.

## Pipelines complexes

**Pipeline multi-étapes** :
```yaml
# .github/workflows/complex-pipeline.yml
name: Complex Pipeline

on:
  push:
    branches: [ main ]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Lint
        run: npm run lint
  
  test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Test
        run: npm test
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build
        run: docker build -t myapp:${{ github.sha }} .
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: ./scripts/deploy.sh
```

## Orchestration

**Script d'orchestration** :
```bash
#!/bin/bash
# Orchestration de pipeline

orchestrate_pipeline() {
    local branch="${1:-main}"
    
    # Lint
    echo "1. Linting..."
    npm run lint || exit 1
    
    # Test
    echo "2. Testing..."
    npm test || exit 1
    
    # Build
    echo "3. Building..."
    docker build -t myapp:latest . || exit 1
    
    # Deploy
    echo "4. Deploying..."
    ./scripts/deploy.sh || exit 1
    
    echo "✓ Pipeline terminé"
}
```

## Gestion d'erreurs

**Gestion d'erreurs robuste** :
```bash
#!/bin/bash
# Pipeline avec gestion d'erreurs

set -euo pipefail

pipeline_with_error_handling() {
    trap 'handle_error $?' EXIT
    
    # Étapes du pipeline
    step1 || return 1
    step2 || return 1
    step3 || return 1
}

handle_error() {
    local exit_code=$1
    echo "Erreur dans le pipeline (code: $exit_code)"
    # Nettoyage et notifications
}
```

## Conclusion

L'automatisation de pipelines crée des workflows fiables et reproductibles.

