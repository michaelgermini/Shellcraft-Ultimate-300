# Chapitre 187 - Headers et authentification

## Table des matières
- [Introduction](#introduction)
- [Gestion des headers](#gestion-des-headers)
- [Authentification de base](#authentification-de-base)
- [Authentification par token](#authentification-par-token)
- [OAuth et JWT](#oauth-et-jwt)
- [Conclusion](#conclusion)

## Introduction

Les headers HTTP permettent de personnaliser les requêtes et l'authentification sécurise l'accès aux APIs. Ce chapitre couvre toutes les méthodes d'authentification courantes.

## Gestion des headers

**Headers personnalisés** :
```bash
#!/bin/bash
# Gestion des headers avec cURL

# Headers multiples
custom_headers() {
    curl -H "Accept: application/json" \
         -H "Content-Type: application/json" \
         -H "User-Agent: MyApp/1.0" \
         https://api.example.com/data
}

# Headers depuis un fichier
headers_from_file() {
    curl -H @headers.txt https://api.example.com/data
}

# Voir les headers de réponse
show_response_headers() {
    curl -v https://api.example.com/data
}
```

## Authentification de base

**Basic Auth** :
```bash
#!/bin/bash
# Authentification de base

# Basic Auth simple
basic_auth() {
    curl -u "username:password" https://api.example.com/data
}

# Basic Auth avec variable
basic_auth_var() {
    local username="$1"
    local password="$2"
    curl -u "$username:$password" https://api.example.com/data
}

# Basic Auth encodé
basic_auth_encoded() {
    local credentials=$(echo -n "username:password" | base64)
    curl -H "Authorization: Basic $credentials" https://api.example.com/data
}
```

## Authentification par token

**Token Bearer** :
```bash
#!/bin/bash
# Authentification par token

# Bearer token
bearer_token() {
    curl -H "Authorization: Bearer $TOKEN" https://api.example.com/data
}

# Token depuis un fichier
token_from_file() {
    local token=$(cat .token)
    curl -H "Authorization: Bearer $token" https://api.example.com/data
}
```

## OAuth et JWT

**OAuth 2.0** :
```bash
#!/bin/bash
# OAuth 2.0 avec cURL

# Obtenir un token OAuth
get_oauth_token() {
    curl -X POST https://oauth.example.com/token \
         -H "Content-Type: application/x-www-form-urlencoded" \
         -d "grant_type=client_credentials" \
         -d "client_id=$CLIENT_ID" \
         -d "client_secret=$CLIENT_SECRET"
}

# Utiliser le token OAuth
use_oauth_token() {
    local token=$(get_oauth_token | jq -r '.access_token')
    curl -H "Authorization: Bearer $token" https://api.example.com/data
}
```

## Conclusion

Les headers et l'authentification sont essentiels pour sécuriser et personnaliser vos interactions avec les APIs.

Dans le chapitre suivant, nous explorerons les cookies et les sessions web.

