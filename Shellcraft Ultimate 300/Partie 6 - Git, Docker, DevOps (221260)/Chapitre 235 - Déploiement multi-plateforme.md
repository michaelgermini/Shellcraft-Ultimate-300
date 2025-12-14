# Chapitre 235 - Déploiement multi-plateforme

## Table des matières
- [Introduction](#introduction)
- [Build multi-architecture](#build-multi-architecture)
- [Déploiement adaptatif](#déploiement-adaptatif)
- [Conclusion](#conclusion)

## Introduction

Le déploiement multi-plateforme permet de déployer vos applications sur différentes architectures (x86_64, ARM, etc.) et systèmes d'exploitation.

## Build multi-architecture

**Buildx multi-platform** :
```bash
#!/bin/bash
# Build multi-architecture

build_multi_arch() {
    local image_name="$1"
    local dockerfile="${2:-Dockerfile}"
    
    docker buildx create --use --name multiarch
    docker buildx build \
        --platform linux/amd64,linux/arm64 \
        -t "$image_name" \
        -f "$dockerfile" \
        --push .
}
```

## Déploiement adaptatif

**Script de déploiement adaptatif** :
```bash
#!/bin/bash
# Déploiement adaptatif selon la plateforme

detect_platform() {
    case "$(uname -m)" in
        x86_64) echo "amd64" ;;
        arm64|aarch64) echo "arm64" ;;
        *) echo "unknown" ;;
    esac
}

deploy_adaptive() {
    local platform=$(detect_platform)
    local image_base="$1"
    
    docker pull "${image_base}:${platform}"
    docker tag "${image_base}:${platform}" "${image_base}:latest"
    docker run -d "${image_base}:latest"
}
```

## Conclusion

Le déploiement multi-plateforme étend la portée de vos applications à différentes architectures.

