# Chapitre 244 - CI/CD basique

## Table des matières
- [Introduction](#introduction)
- [Pipeline CI de base](#pipeline-ci-de-base)
- [Pipeline CD de base](#pipeline-cd-de-base)
- [Intégration Git](#intégration-git)
- [Conclusion](#conclusion)

## Introduction

CI/CD (Continuous Integration / Continuous Deployment) automatise le processus de développement, de test, et de déploiement. Ce chapitre présente les concepts de base.

## Pipeline CI de base

**GitHub Actions basique** :
```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: |
          npm install
          npm test
  
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build
        run: |
          docker build -t myapp:latest .
```

## Pipeline CD de base

**Déploiement automatique** :
```yaml
# .github/workflows/cd.yml
name: CD Pipeline

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy
        run: |
          ./scripts/deploy.sh
```

## Intégration Git

**Scripts Git intégrés** :
```bash
#!/bin/bash
# Intégration Git dans CI/CD

# Hook pre-push
pre_push_hook() {
    # Exécuter les tests avant push
    if ! npm test; then
        echo "Tests échoués. Push annulé."
        exit 1
    fi
}
```

## Conclusion

CI/CD automatise le cycle de développement et réduit les erreurs humaines.

