# Chapitre 241 - Workflow DevOps complet

## Table des matières
- [Introduction](#introduction)
- [Pipeline CI/CD complet](#pipeline-cicd-complet)
- [Infrastructure as Code](#infrastructure-as-code)
- [Monitoring et observabilité](#monitoring-et-observabilité)
- [Conclusion](#conclusion)

## Introduction

Un workflow DevOps complet intègre tous les aspects : développement, tests, build, déploiement, monitoring, et maintenance. Ce chapitre synthétise tous les concepts appris dans un workflow production-ready.

## Pipeline CI/CD complet

**Pipeline GitHub Actions complet** :
```yaml
# .github/workflows/complete-ci-cd.yml
name: Complete CI/CD Pipeline

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
          docker-compose up -d
          ./scripts/run_tests.sh
          docker-compose down
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build and push
        run: |
          docker build -t ${{ secrets.REGISTRY }}/app:${{ github.sha }} .
          docker push ${{ secrets.REGISTRY }}/app:${{ github.sha }}
  
  deploy:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/app app=${{ secrets.REGISTRY }}/app:${{ github.sha }}
```

## Infrastructure as Code

**Terraform + Ansible** :
```bash
#!/bin/bash
# Déploiement infrastructure complète

deploy_infrastructure() {
    # Terraform pour l'infrastructure
    terraform init
    terraform plan
    terraform apply -auto-approve
    
    # Ansible pour la configuration
    ansible-playbook -i inventory.ini playbook.yml
}
```

## Monitoring et observabilité

**Stack complète** :
```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
  
  grafana:
    image: grafana/grafana
  
  loki:
    image: grafana/loki
  
  promtail:
    image: grafana/promtail
```

## Conclusion

Un workflow DevOps complet intègre tous les outils et pratiques pour créer un pipeline de développement à production efficace et fiable.

