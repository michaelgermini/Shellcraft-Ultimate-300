# Chapitre 189 - Téléchargement multi-fichiers

## Table des matières
- [Introduction](#introduction)
- [Téléchargements parallèles](#téléchargements-parallèles)
- [Téléchargements séquentiels](#téléchargements-séquentiels)
- [Gestion d'erreurs](#gestion-derreurs)
- [Conclusion](#conclusion)

## Introduction

Le téléchargement de plusieurs fichiers nécessite des stratégies efficaces pour gérer la parallélisation et les erreurs.

## Téléchargements parallèles

**Téléchargements en parallèle** :
```bash
#!/bin/bash
# Téléchargements parallèles

download_parallel() {
    local urls=("$@")
    
    for url in "${urls[@]}"; do
        (
            filename=$(basename "$url")
            curl -o "$filename" "$url" && echo "✓ $filename"
        ) &
    done
    
    wait
    echo "Tous les téléchargements terminés"
}
```

## Téléchargements séquentiels

**Téléchargements séquentiels** :
```bash
#!/bin/bash
# Téléchargements séquentiels

download_sequential() {
    local urls=("$@")
    
    for url in "${urls[@]}"; do
        filename=$(basename "$url")
        echo "Téléchargement de $filename..."
        curl -o "$filename" "$url" || echo "✗ Échec: $url"
    done
}
```

## Gestion d'erreurs

**Téléchargement robuste** :
```bash
#!/bin/bash
# Téléchargement avec gestion d'erreurs

download_with_retry() {
    local url="$1"
    local max_attempts="${2:-3}"
    local attempt=1
    
    while [ $attempt -le $max_attempts ]; do
        if curl -f -o "$(basename "$url")" "$url"; then
            return 0
        fi
        attempt=$((attempt + 1))
        sleep 2
    done
    
    return 1
}
```

## Conclusion

Le téléchargement multi-fichiers nécessite une gestion efficace de la parallélisation et des erreurs.

