# Chapitre 192 - APIs REST avancées

## Table des matières
- [Introduction](#introduction)
- [Pagination](#pagination)
- [Filtrage et recherche](#filtrage-et-recherche)
- [Rate limiting](#rate-limiting)
- [Conclusion](#conclusion)

## Introduction

Les APIs REST avancées implémentent des fonctionnalités comme la pagination, le filtrage, et le rate limiting pour gérer efficacement de grandes quantités de données.

## Pagination

**Gestion de la pagination** :
```bash
#!/bin/bash
# Pagination avec APIs REST

fetch_all_pages() {
    local endpoint="$1"
    local page=1
    local all_data="[]"
    
    while true; do
        local response=$(api_get "${endpoint}?page=${page}&limit=100")
        local page_data=$(echo "$response" | jq '.data')
        
        if [ "$(echo "$page_data" | jq 'length')" -eq 0 ]; then
            break
        fi
        
        all_data=$(echo "$all_data" | jq ". + $page_data")
        page=$((page + 1))
    done
    
    echo "$all_data"
}
```

## Filtrage et recherche

**Filtrage avancé** :
```bash
#!/bin/bash
# Filtrage et recherche

search_api() {
    local endpoint="$1"
    local query="$2"
    local filters="${3:-}"
    
    local url="${endpoint}?q=${query}"
    
    if [ -n "$filters" ]; then
        url="${url}&${filters}"
    fi
    
    api_get "$url"
}
```

## Rate limiting

**Gestion du rate limiting** :
```bash
#!/bin/bash
# Gestion du rate limiting

rate_limited_request() {
    local endpoint="$1"
    local rate_limit="${RATE_LIMIT:-60}"  # Requêtes par minute
    
    # Vérifier le dernier appel
    local last_call_file=".last_api_call"
    local current_time=$(date +%s)
    
    if [ -f "$last_call_file" ]; then
        local last_call=$(cat "$last_call_file")
        local time_diff=$((current_time - last_call))
        local min_interval=$((60 / rate_limit))
        
        if [ $time_diff -lt $min_interval ]; then
            sleep $((min_interval - time_diff))
        fi
    fi
    
    echo "$current_time" > "$last_call_file"
    api_get "$endpoint"
}
```

## Conclusion

Les APIs REST avancées nécessitent une gestion sophistiquée de la pagination, du filtrage, et du rate limiting.

