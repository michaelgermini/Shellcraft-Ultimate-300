# Chapitre 191 - Scripts curl avancés

## Table des matières
- [Introduction](#introduction)
- [Scripts réutilisables](#scripts-réutilisables)
- [Gestion d'erreurs avancée](#gestion-derreurs-avancée)
- [Optimisation des performances](#optimisation-des-performances)
- [Conclusion](#conclusion)

## Introduction

Les scripts curl avancés permettent d'automatiser des interactions complexes avec les APIs, gérer les erreurs, et optimiser les performances.

## Scripts réutilisables

**Bibliothèque de fonctions curl** :
```bash
#!/bin/bash
# Bibliothèque de fonctions curl réutilisables

API_BASE_URL="${API_BASE_URL:-https://api.example.com}"
TOKEN_FILE="${TOKEN_FILE:-.token}"

# Fonction générique pour les requêtes
api_request() {
    local method="$1"
    local endpoint="$2"
    local data="${3:-}"
    local headers="${4:-}"
    
    local url="${API_BASE_URL}${endpoint}"
    local curl_cmd="curl -s -X $method"
    
    # Ajouter l'authentification
    if [ -f "$TOKEN_FILE" ]; then
        curl_cmd="$curl_cmd -H \"Authorization: Bearer $(cat $TOKEN_FILE)\""
    fi
    
    # Ajouter les headers personnalisés
    if [ -n "$headers" ]; then
        curl_cmd="$curl_cmd $headers"
    fi
    
    # Ajouter les données
    if [ -n "$data" ]; then
        curl_cmd="$curl_cmd -d '$data'"
    fi
    
    # Ajouter l'URL
    curl_cmd="$curl_cmd \"$url\""
    
    eval "$curl_cmd"
}

# Wrapper pour GET
api_get() {
    api_request "GET" "$1" "" "$2"
}

# Wrapper pour POST
api_post() {
    api_request "POST" "$1" "$2" "$3"
}

# Wrapper pour PUT
api_put() {
    api_request "PUT" "$1" "$2" "$3"
}

# Wrapper pour DELETE
api_delete() {
    api_request "DELETE" "$1" "" "$2"
}
```

## Gestion d'erreurs avancée

**Gestion d'erreurs robuste** :
```bash
#!/bin/bash
# Gestion d'erreurs avancée

api_request_safe() {
    local method="$1"
    local endpoint="$2"
    local data="${3:-}"
    
    local response=$(api_request "$method" "$endpoint" "$data" 2>&1)
    local exit_code=$?
    
    if [ $exit_code -ne 0 ]; then
        echo "Erreur curl: $response" >&2
        return $exit_code
    fi
    
    # Vérifier le code HTTP
    local http_code=$(echo "$response" | grep -oP 'HTTP/\d\.\d \K\d+')
    
    if [ "$http_code" -ge 400 ]; then
        echo "Erreur HTTP $http_code: $response" >&2
        return 1
    fi
    
    echo "$response"
    return 0
}

# Retry automatique
api_request_with_retry() {
    local max_attempts="${MAX_RETRIES:-3}"
    local attempt=1
    
    while [ $attempt -le $max_attempts ]; do
        if api_request_safe "$@"; then
            return 0
        fi
        
        attempt=$((attempt + 1))
        sleep $((attempt * 2))  # Backoff exponentiel
    done
    
    return 1
}
```

## Optimisation des performances

**Requêtes optimisées** :
```bash
#!/bin/bash
# Optimisation des performances

# Compression
api_request_compressed() {
    curl -s \
         -H "Accept-Encoding: gzip, deflate" \
         --compressed \
         "$@"
}

# Connexion persistante
api_request_persistent() {
    curl -s \
         --keepalive-time 60 \
         "$@"
}

# Timeout configurable
api_request_with_timeout() {
    local timeout="${1:-30}"
    shift
    curl -s --max-time "$timeout" "$@"
}
```

## Conclusion

Les scripts curl avancés permettent d'automatiser efficacement les interactions avec les APIs tout en gérant les erreurs et optimisant les performances.

