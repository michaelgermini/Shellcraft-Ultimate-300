# Chapitre 208 - Cache et mise en cache

## Table des matières
- [Introduction](#introduction)
- [Cache HTTP](#cache-http)
- [Cache local](#cache-local)
- [Invalidation du cache](#invalidation-du-cache)
- [Conclusion](#conclusion)

## Introduction

La mise en cache réduit la charge sur les APIs et améliore les performances en stockant les réponses fréquemment utilisées.

## Cache HTTP

**Respect du cache HTTP** :
```bash
#!/bin/bash
# Cache HTTP

cached_request() {
    local url="$1"
    local cache_file=".cache/$(echo "$url" | md5sum | cut -d' ' -f1)"
    
    if [ -f "$cache_file" ]; then
        local cached_time=$(stat -c %Y "$cache_file")
        local current_time=$(date +%s)
        local age=$((current_time - cached_time))
        
        if [ $age -lt 3600 ]; then  # Cache valide 1 heure
            cat "$cache_file"
            return 0
        fi
    fi
    
    local response=$(curl -s "$url")
    mkdir -p .cache
    echo "$response" > "$cache_file"
    echo "$response"
}
```

## Cache local

**Système de cache local** :
```bash
#!/bin/bash
# Cache local

CACHE_DIR="${CACHE_DIR:-.cache}"
CACHE_TTL="${CACHE_TTL:-3600}"

get_cached() {
    local key="$1"
    local cache_file="${CACHE_DIR}/${key}"
    
    if [ -f "$cache_file" ]; then
        local age=$(($(date +%s) - $(stat -c %Y "$cache_file")))
        if [ $age -lt $CACHE_TTL ]; then
            cat "$cache_file"
            return 0
        fi
    fi
    
    return 1
}

set_cache() {
    local key="$1"
    local value="$2"
    local cache_file="${CACHE_DIR}/${key}"
    
    mkdir -p "$CACHE_DIR"
    echo "$value" > "$cache_file"
}
```

## Invalidation du cache

**Invalidation intelligente** :
```bash
#!/bin/bash
# Invalidation du cache

invalidate_cache() {
    local pattern="$1"
    
    find "$CACHE_DIR" -name "$pattern" -delete
}

clear_all_cache() {
    rm -rf "$CACHE_DIR"/*
}
```

## Conclusion

La mise en cache améliore significativement les performances tout en réduisant la charge sur les APIs.

