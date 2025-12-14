# Chapitre 202 - Sécurité et best practices

## Table des matières
- [Introduction](#introduction)
- [Gestion sécurisée des secrets](#gestion-sécurisée-des-secrets)
- [Validation des entrées](#validation-des-entrées)
- [Protection contre les attaques](#protection-contre-les-attaques)
- [Conclusion](#conclusion)

## Introduction

La sécurité est essentielle lors de l'interaction avec les APIs. Ce chapitre couvre les meilleures pratiques pour sécuriser vos scripts et interactions.

## Gestion sécurisée des secrets

**Gestion des secrets** :
```bash
#!/bin/bash
# Gestion sécurisée des secrets

# Utiliser des variables d'environnement
load_secrets() {
    if [ -f ".env" ]; then
        set -a
        source .env
        set +a
    fi
}

# Ne jamais logger les secrets
safe_api_call() {
    local endpoint="$1"
    curl -s -H "Authorization: Bearer $API_TOKEN" \
         "${API_BASE_URL}${endpoint}" \
         2>&1 | grep -v "$API_TOKEN"  # Filtrer les tokens des logs
}
```

## Validation des entrées

**Validation robuste** :
```bash
#!/bin/bash
# Validation des entrées

validate_url() {
    local url="$1"
    
    if [[ ! "$url" =~ ^https?:// ]]; then
        echo "URL invalide" >&2
        return 1
    fi
    
    return 0
}

validate_json() {
    local json="$1"
    
    if ! echo "$json" | jq . > /dev/null 2>&1; then
        echo "JSON invalide" >&2
        return 1
    fi
    
    return 0
}
```

## Protection contre les attaques

**Protections de base** :
```bash
#!/bin/bash
# Protection contre les attaques

# Rate limiting
rate_limit() {
    local max_requests="${MAX_REQUESTS:-60}"
    local window="${WINDOW:-60}"  # secondes
    
    # Implémenter un rate limiter simple
    # ...
}

# Validation SSL
validate_ssl() {
    local url="$1"
    
    curl --ssl-no-revoke \
         --cert-status \
         "$url"
}
```

## Conclusion

La sécurité doit être une priorité dans tous les scripts d'interaction avec les APIs.

