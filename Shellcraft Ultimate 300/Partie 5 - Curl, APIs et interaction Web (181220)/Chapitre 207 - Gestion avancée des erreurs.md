# Chapitre 207 - Gestion avancée des erreurs

## Table des matières
- [Introduction](#introduction)
- [Détection d'erreurs](#détection-derreurs)
- [Retry automatique](#retry-automatique)
- [Logging des erreurs](#logging-des-erreurs)
- [Conclusion](#conclusion)

## Introduction

Une gestion avancée des erreurs garantit la robustesse de vos scripts en gérant tous les cas d'erreur possibles.

## Détection d'erreurs

**Détection complète** :
```bash
#!/bin/bash
# Détection d'erreurs

check_curl_error() {
    local exit_code=$?
    local http_code=$1
    
    if [ $exit_code -ne 0 ]; then
        echo "Erreur curl: $exit_code" >&2
        return $exit_code
    fi
    
    if [ "$http_code" -ge 400 ]; then
        echo "Erreur HTTP: $http_code" >&2
        return 1
    fi
    
    return 0
}
```

## Retry automatique

**Retry avec backoff** :
```bash
#!/bin/bash
# Retry avec backoff exponentiel

retry_with_backoff() {
    local max_attempts="${1:-3}"
    local delay="${2:-2}"
    shift 2
    
    local attempt=1
    while [ $attempt -le $max_attempts ]; do
        if curl "$@"; then
            return 0
        fi
        
        sleep $((delay * attempt))
        attempt=$((attempt + 1))
    done
    
    return 1
}
```

## Logging des erreurs

**Logging structuré** :
```bash
#!/bin/bash
# Logging des erreurs

log_error() {
    local error="$1"
    local context="$2"
    
    echo "[$(date -Iseconds)] ERROR: $error | Context: $context" >> error.log
}
```

## Conclusion

Une gestion avancée des erreurs garantit la fiabilité de vos scripts en production.

