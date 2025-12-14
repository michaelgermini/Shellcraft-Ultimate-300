# Chapitre 236 - Projets DevOps pratiques

## Table des matières
- [Introduction](#introduction)
- [Projet 1 : Pipeline CI/CD complet](#projet-1--pipeline-cicd-complet)
- [Projet 2 : Infrastructure as Code](#projet-2--infrastructure-as-code)
- [Projet 3 : Monitoring complet](#projet-3--monitoring-complet)
- [Conclusion](#conclusion)

## Introduction

Ces projets pratiques intègrent tous les concepts DevOps appris pour créer des solutions complètes et production-ready.

## Projet 1 : Pipeline CI/CD complet

**Pipeline GitHub Actions** :
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
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
          ./run_tests.sh
          docker-compose down
  
  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build Docker image
        run: docker build -t myapp:${{ github.sha }} .
      - name: Push to registry
        run: docker push myapp:${{ github.sha }}
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: ./deploy.sh
```

## Projet 2 : Infrastructure as Code

**Terraform complet** :
```hcl
# infrastructure/main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  
  tags = {
    Name = "App Server"
  }
}
```

## Projet 3 : Monitoring complet

**Stack de monitoring** :
```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
  
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
  
  alertmanager:
    image: prom/alertmanager
    ports:
      - "9093:9093"
```

## Conclusion

Ces projets pratiques démontrent l'intégration complète des outils DevOps dans des solutions réelles.

