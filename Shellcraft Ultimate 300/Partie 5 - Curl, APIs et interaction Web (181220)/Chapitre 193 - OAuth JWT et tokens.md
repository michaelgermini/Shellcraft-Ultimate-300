# Chapitre 193 - OAuth, JWT et tokens

## Table des matières
- [Introduction](#introduction)
- [OAuth 2.0](#oauth-20)
- [JWT (JSON Web Tokens)](#jwt-json-web-tokens)
- [Gestion des tokens](#gestion-des-tokens)
- [Conclusion](#conclusion)

## Introduction

OAuth, JWT, et les tokens sont les standards modernes d'authentification et d'autorisation pour les APIs sécurisées.

## OAuth 2.0

**Flux OAuth 2.0** :
```bash
#!/bin/bash
# OAuth 2.0 avec cURL

# Authorization Code Flow
oauth_authorize() {
    local client_id="$1"
    local redirect_uri="$2"
    local scope="${3:-read write}"
    
    local auth_url="https://oauth.example.com/authorize?client_id=${client_id}&redirect_uri=${redirect_uri}&response_type=code&scope=${scope}"
    
    echo "Visitez: $auth_url"
    read -p "Entrez le code d'autorisation: " auth_code
    
    echo "$auth_code"
}

# Échanger le code contre un token
oauth_get_token() {
    local client_id="$1"
    local client_secret="$2"
    local auth_code="$3"
    local redirect_uri="$4"
    
    curl -X POST https://oauth.example.com/token \
         -H "Content-Type: application/x-www-form-urlencoded" \
         -d "grant_type=authorization_code" \
         -d "client_id=${client_id}" \
         -d "client_secret=${client_secret}" \
         -d "code=${auth_code}" \
         -d "redirect_uri=${redirect_uri}"
}

# Client Credentials Flow
oauth_client_credentials() {
    local client_id="$1"
    local client_secret="$2"
    
    curl -X POST https://oauth.example.com/token \
         -u "${client_id}:${client_secret}" \
         -d "grant_type=client_credentials"
}
```

## JWT (JSON Web Tokens)

**Gestion JWT** :
```bash
#!/bin/bash
# Gestion JWT

# Décoder un JWT (sans vérification)
decode_jwt() {
    local token="$1"
    
    # Extraire le payload (partie 2)
    local payload=$(echo "$token" | cut -d'.' -f2)
    
    # Ajouter le padding si nécessaire
    local padding=$((4 - ${#payload} % 4))
    if [ $padding -ne 4 ]; then
        payload="${payload}$(printf '%*s' $padding | tr ' ' '=')"
    fi
    
    # Décoder base64
    echo "$payload" | base64 -d | jq .
}

# Vérifier l'expiration
check_jwt_expiry() {
    local token="$1"
    local payload=$(decode_jwt "$token")
    local exp=$(echo "$payload" | jq -r '.exp')
    local current_time=$(date +%s)
    
    if [ "$exp" -lt "$current_time" ]; then
        echo "Token expiré"
        return 1
    fi
    
    echo "Token valide"
    return 0
}
```

## Gestion des tokens

**Gestionnaire de tokens** :
```bash
#!/bin/bash
# Gestionnaire de tokens

TOKEN_FILE=".token"
REFRESH_TOKEN_FILE=".refresh_token"

save_token() {
    local token="$1"
    echo "$token" > "$TOKEN_FILE"
}

get_token() {
    if [ -f "$TOKEN_FILE" ]; then
        cat "$TOKEN_FILE"
    fi
}

refresh_token() {
    local refresh_token=$(cat "$REFRESH_TOKEN_FILE")
    
    local response=$(curl -s -X POST https://oauth.example.com/token \
         -d "grant_type=refresh_token" \
         -d "refresh_token=${refresh_token}")
    
    local new_token=$(echo "$response" | jq -r '.access_token')
    save_token "$new_token"
    
    echo "$new_token"
}

get_valid_token() {
    local token=$(get_token)
    
    if [ -z "$token" ] || ! check_jwt_expiry "$token"; then
        token=$(refresh_token)
    fi
    
    echo "$token"
}
```

## Conclusion

OAuth, JWT, et la gestion des tokens sont essentiels pour sécuriser les interactions avec les APIs modernes.

