# Chapitre 218 - Rate limiting et throttling

## Table des matières
- [Introduction](#introduction)
- [Respect du rate limiting](#respect-du-rate-limiting)
- [Implémentation côté client](#implémentation-côté-client)
- [Conclusion](#conclusion)

## Introduction

Le rate limiting et le throttling contrôlent la fréquence des requêtes pour protéger les APIs.

## Respect du rate limiting

**Détection et respect** :
```bash
#!/bin/bash
# Respect du rate limiting

rate_limited_request() {
    local url="$1"
    
    local response=$(curl -s -w "\n%{http_code}" "$url")
    local http_code=$(echo "$response" | tail -n1)
    local body=$(echo "$response" | head -n-1)
    
    if [ "$http_code" -eq 429 ]; then
        local retry_after=$(echo "$body" | jq -r '.retry_after // 60')
        echo "Rate limit atteint, attente ${retry_after}s"
        sleep "$retry_after"
        rate_limited_request "$url"
    else
        echo "$body"
    fi
}
```

## Implémentation côté client

**Rate limiter côté client** :
```bash
#!/bin/bash
# Rate limiter côté client

RATE_LIMIT="${RATE_LIMIT:-60}"  # Requêtes par minute
REQUEST_LOG=".request_times"

rate_limit_check() {
    local current_time=$(date +%s)
    local cutoff=$((current_time - 60))
    
    # Compter les requêtes dans la dernière minute
    local count=$(awk -v cutoff="$cutoff" '$1 > cutoff {count++} END {print count+0}' "$REQUEST_LOG" 2>/dev/null || echo 0)
    
    if [ "$count" -ge "$RATE_LIMIT" ]; then
        local oldest=$(head -n1 "$REQUEST_LOG" | awk '{print $1}')
        local wait_time=$((60 - (current_time - oldest)))
        sleep "$wait_time"
    fi
    
    echo "$current_time" >> "$REQUEST_LOG"
}
```

## Conclusion

Le respect du rate limiting est essentiel pour maintenir de bonnes relations avec les fournisseurs d'API.

