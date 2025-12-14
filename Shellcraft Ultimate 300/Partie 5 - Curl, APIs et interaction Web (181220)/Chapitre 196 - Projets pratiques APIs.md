# Chapitre 196 - Projets pratiques APIs

## Table des matières
- [Introduction](#introduction)
- [Projet 1 : Client API complet](#projet-1--client-api-complet)
- [Projet 2 : Synchronisation de données](#projet-2--synchronisation-de-données)
- [Projet 3 : Dashboard API](#projet-3--dashboard-api)
- [Conclusion](#conclusion)

## Introduction

Ces projets pratiques démontrent l'utilisation réelle des APIs avec cURL dans des scénarios concrets et production-ready.

## Projet 1 : Client API complet

**Client API réutilisable** :
```bash
#!/bin/bash
# Client API complet

set -euo pipefail

API_BASE_URL="${API_BASE_URL:-https://api.example.com}"
CONFIG_FILE="${HOME}/.api_client_config"

# Charger la configuration
load_config() {
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
    fi
}

# Authentification
authenticate() {
    local username="$1"
    local password="$2"
    
    local response=$(curl -s -X POST "${API_BASE_URL}/auth/login" \
        -H "Content-Type: application/json" \
        -d "{\"username\":\"$username\",\"password\":\"$password\"}")
    
    local token=$(echo "$response" | jq -r '.token')
    echo "API_TOKEN=$token" > "$CONFIG_FILE"
    echo "✓ Authentifié"
}

# CRUD complet
create_resource() {
    local endpoint="$1"
    local data="$2"
    
    curl -s -X POST "${API_BASE_URL}${endpoint}" \
        -H "Authorization: Bearer $API_TOKEN" \
        -H "Content-Type: application/json" \
        -d "$data"
}

get_resource() {
    local endpoint="$1"
    local id="${2:-}"
    
    local url="${API_BASE_URL}${endpoint}"
    [ -n "$id" ] && url="${url}/${id}"
    
    curl -s -X GET "$url" \
        -H "Authorization: Bearer $API_TOKEN"
}

update_resource() {
    local endpoint="$1"
    local id="$2"
    local data="$3"
    
    curl -s -X PUT "${API_BASE_URL}${endpoint}/${id}" \
        -H "Authorization: Bearer $API_TOKEN" \
        -H "Content-Type: application/json" \
        -d "$data"
}

delete_resource() {
    local endpoint="$1"
    local id="$2"
    
    curl -s -X DELETE "${API_BASE_URL}${endpoint}/${id}" \
        -H "Authorization: Bearer $API_TOKEN"
}
```

## Projet 2 : Synchronisation de données

**Synchronisation automatique** :
```bash
#!/bin/bash
# Synchronisation de données API

sync_data() {
    local local_db="local_data.json"
    local api_endpoint="/data"
    
    # Récupérer les données de l'API
    local api_data=$(get_resource "$api_endpoint")
    
    # Comparer avec les données locales
    if [ -f "$local_db" ]; then
        local diff=$(diff <(echo "$api_data" | jq -S .) <(cat "$local_db" | jq -S .))
        
        if [ -n "$diff" ]; then
            echo "Différences détectées, synchronisation..."
            echo "$api_data" > "$local_db"
            echo "✓ Synchronisé"
        else
            echo "✓ Déjà synchronisé"
        fi
    else
        echo "$api_data" > "$local_db"
        echo "✓ Données initiales téléchargées"
    fi
}
```

## Projet 3 : Dashboard API

**Dashboard en ligne de commande** :
```bash
#!/bin/bash
# Dashboard API

show_dashboard() {
    clear
    echo "=== Dashboard API ==="
    echo
    
    # Statistiques utilisateurs
    local users=$(get_resource "/users" | jq '. | length')
    echo "Utilisateurs: $users"
    
    # Statistiques commandes
    local orders=$(get_resource "/orders" | jq '. | length')
    echo "Commandes: $orders"
    
    # Dernières activités
    echo
    echo "=== Dernières activités ==="
    get_resource "/activities" | jq -r '.[:5][] | "\(.timestamp) - \(.action)"'
}
```

## Conclusion

Ces projets pratiques démontrent l'utilisation réelle des APIs avec cURL dans des scénarios de production.

