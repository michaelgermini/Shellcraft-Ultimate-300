# Chapitre 211 - APIs gRPC et REST

## Table des matières
- [Introduction](#introduction)
- [REST : Architecture et principes](#rest--architecture-et-principes)
- [gRPC : Framework RPC moderne](#grpc--framework-rpc-moderne)
- [Comparaison détaillée REST vs gRPC](#comparaison-détaillée-rest-vs-grpc)
- [Utilisation REST avec cURL](#utilisation-rest-avec-curl)
- [Utilisation gRPC avec grpcurl](#utilisation-grpc-avec-grpcurl)
- [Cas d'usage et choix](#cas-dusage-et-choix)
- [Migration et coexistence](#migration-et-coexistence)
- [Performance et optimisation](#performance-et-optimisation)
- [Conclusion](#conclusion)

## Introduction

Les APIs REST et gRPC représentent deux approches fondamentalement différentes pour construire des interfaces de programmation. REST utilise HTTP avec JSON, tandis que gRPC utilise HTTP/2 avec Protocol Buffers. Chaque approche a ses avantages et ses cas d'usage optimaux.

Imaginez REST comme une conversation en langage naturel : flexible, lisible, facile à comprendre, mais parfois verbeuse. gRPC est comme un code morse optimisé : compact, rapide, efficace, mais nécessitant une connaissance préalable du protocole.

## REST : Architecture et principes

### Principes REST

**Caractéristiques fondamentales** :
- **Stateless** : Chaque requête contient toutes les informations nécessaires
- **Ressources** : Les données sont représentées comme des ressources identifiables par des URLs
- **Méthodes HTTP** : GET, POST, PUT, DELETE, PATCH pour les opérations CRUD
- **Représentations** : JSON, XML, HTML comme formats de données
- **Hypermedia** : Liens entre ressources (HATEOAS)

**Architecture REST** :
```bash
#!/bin/bash
# Exemple d'architecture REST

# Ressources identifiables par URL
GET    /api/users          # Liste des utilisateurs
GET    /api/users/123      # Utilisateur spécifique
POST   /api/users          # Créer un utilisateur
PUT    /api/users/123      # Mettre à jour complètement
PATCH  /api/users/123      # Mettre à jour partiellement
DELETE /api/users/123      # Supprimer un utilisateur
```

### Avantages de REST

**Points forts** :
- **Simplicité** : Facile à comprendre et utiliser
- **Flexibilité** : Format JSON lisible et modifiable
- **Caching** : Support natif du cache HTTP
- **Compatibilité** : Fonctionne avec tous les navigateurs et outils
- **Écosystème** : Nombreux outils et bibliothèques disponibles

**Exemple REST simple** :
```bash
#!/bin/bash
# API REST simple avec cURL

# GET - Récupérer des données
get_users() {
    curl -X GET "https://api.example.com/users" \
         -H "Accept: application/json"
}

# POST - Créer une ressource
create_user() {
    local name="$1"
    local email="$2"
    
    curl -X POST "https://api.example.com/users" \
         -H "Content-Type: application/json" \
         -H "Accept: application/json" \
         -d "{
             \"name\": \"$name\",
             \"email\": \"$email\"
         }"
}

# PUT - Mettre à jour complètement
update_user() {
    local user_id="$1"
    local name="$2"
    local email="$3"
    
    curl -X PUT "https://api.example.com/users/$user_id" \
         -H "Content-Type: application/json" \
         -d "{
             \"name\": \"$name\",
             \"email\": \"$email\"
         }"
}

# DELETE - Supprimer
delete_user() {
    local user_id="$1"
    
    curl -X DELETE "https://api.example.com/users/$user_id"
}
```

## gRPC : Framework RPC moderne

### Principes gRPC

**Caractéristiques fondamentales** :
- **Protocol Buffers** : Format de sérialisation binaire efficace
- **HTTP/2** : Multiplexage, compression, streaming
- **Stub générés** : Code client et serveur généré automatiquement
- **Streaming** : Support natif du streaming bidirectionnel
- **Type-safe** : Contrats stricts définis dans les fichiers .proto

**Architecture gRPC** :
```protobuf
// Exemple de fichier .proto
syntax = "proto3";

service UserService {
    rpc GetUser (UserRequest) returns (UserResponse);
    rpc ListUsers (Empty) returns (stream UserResponse);
    rpc CreateUser (CreateUserRequest) returns (UserResponse);
    rpc UpdateUser (UpdateUserRequest) returns (UserResponse);
    rpc DeleteUser (UserRequest) returns (Empty);
}

message UserRequest {
    int32 id = 1;
}

message UserResponse {
    int32 id = 1;
    string name = 2;
    string email = 3;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}
```

### Avantages de gRPC

**Points forts** :
- **Performance** : Format binaire plus rapide que JSON
- **Type safety** : Contrats stricts évitent les erreurs
- **Streaming** : Support natif du streaming bidirectionnel
- **Code génération** : Clients et serveurs générés automatiquement
- **Multiplexage** : Plusieurs requêtes sur une seule connexion HTTP/2

**Exemple gRPC** :
```bash
#!/bin/bash
# gRPC avec grpcurl (outil similaire à cURL pour gRPC)

# Installation de grpcurl
install_grpcurl() {
    # Linux
    if command -v wget >/dev/null 2>&1; then
        wget https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz
        tar -xzf grpcurl_*.tar.gz
        sudo mv grpcurl /usr/local/bin/
    fi
    
    # macOS
    if command -v brew >/dev/null 2>&1; then
        brew install grpcurl
    fi
}

# Lister les services disponibles
list_services() {
    local endpoint="$1"
    
    grpcurl -plaintext "$endpoint" list
}

# Appeler une méthode gRPC
grpc_call() {
    local endpoint="$1"
    local service="$2"
    local method="$3"
    local data="$4"
    
    grpcurl -plaintext \
            -d "$data" \
            "$endpoint" \
            "${service}.${method}"
}

# Exemple : Récupérer un utilisateur
get_user_grpc() {
    local endpoint="${GRPC_ENDPOINT:-localhost:50051}"
    local user_id="${1:-1}"
    
    grpcurl -plaintext \
            -d "{\"id\": $user_id}" \
            "$endpoint" \
            "UserService.GetUser"
}

# Streaming : Liste des utilisateurs
list_users_stream() {
    local endpoint="${GRPC_ENDPOINT:-localhost:50051}"
    
    grpcurl -plaintext \
            "$endpoint" \
            "UserService.ListUsers"
}
```

## Comparaison détaillée REST vs gRPC

### Tableau comparatif

| Caractéristique | REST | gRPC |
|----------------|------|------|
| **Format de données** | JSON (texte) | Protocol Buffers (binaire) |
| **Protocole** | HTTP/1.1 ou HTTP/2 | HTTP/2 uniquement |
| **Taille des messages** | Plus grande (JSON verbeux) | Plus petite (binaire compact) |
| **Vitesse** | Plus lent | Plus rapide |
| **Lisibilité** | Excellente (JSON lisible) | Faible (binaire) |
| **Compatibilité navigateur** | Native | Nécessite un proxy |
| **Streaming** | Limitée (SSE, WebSockets) | Native (bidirectionnel) |
| **Code génération** | Manuelle | Automatique |
| **Type safety** | Faible | Fort |
| **Caching** | Natif HTTP | Limité |
| **Complexité** | Faible | Moyenne à élevée |

### Performance

**Comparaison de performance** :
```bash
#!/bin/bash
# Benchmark REST vs gRPC

benchmark_rest() {
    local endpoint="$1"
    local iterations="${2:-100}"
    
    echo "Benchmark REST ($iterations requêtes)..."
    time for i in $(seq 1 $iterations); do
        curl -s "$endpoint/api/users" > /dev/null
    done
}

benchmark_grpc() {
    local endpoint="$1"
    local iterations="${2:-100}"
    
    echo "Benchmark gRPC ($iterations requêtes)..."
    time for i in $(seq 1 $iterations); do
        grpcurl -plaintext -d '{}' "$endpoint" "UserService.ListUsers" > /dev/null
    done
}

# Comparaison
compare_performance() {
    local rest_endpoint="$1"
    local grpc_endpoint="$2"
    
    echo "=== Comparaison REST vs gRPC ==="
    benchmark_rest "$rest_endpoint"
    echo ""
    benchmark_grpc "$grpc_endpoint"
}
```

### Cas d'usage spécifiques

**Quand utiliser REST** :
- APIs publiques et web
- Intégration avec navigateurs
- Prototypage rapide
- APIs simples CRUD
- Caching important
- Compatibilité maximale requise

**Quand utiliser gRPC** :
- Communication inter-services (microservices)
- APIs internes haute performance
- Streaming de données
- Contrats stricts nécessaires
- Latence minimale requise
- Environnements contrôlés

## Utilisation REST avec cURL

### Requêtes REST complètes

**Script REST complet** :
```bash
#!/bin/bash
# Script REST complet avec cURL

set -euo pipefail

API_BASE_URL="${API_BASE_URL:-https://api.example.com}"
API_KEY="${API_KEY:-}"

# Headers communs
get_headers() {
    local headers=(
        "Content-Type: application/json"
        "Accept: application/json"
    )
    
    if [ -n "$API_KEY" ]; then
        headers+=("Authorization: Bearer $API_KEY")
    fi
    
    printf '%s\n' "${headers[@]}"
}

# GET avec pagination
get_users_paginated() {
    local page="${1:-1}"
    local limit="${2:-10}"
    
    curl -X GET "$API_BASE_URL/users?page=$page&limit=$limit" \
         -H "$(get_headers | head -1)" \
         -H "$(get_headers | tail -1)"
}

# POST avec validation
create_user_rest() {
    local name="$1"
    local email="$2"
    
    local payload=$(cat <<EOF
{
    "name": "$name",
    "email": "$email"
}
EOF
)
    
    curl -X POST "$API_BASE_URL/users" \
         -H "$(get_headers | head -1)" \
         -H "$(get_headers | tail -1)" \
         -d "$payload" \
         -w "\nHTTP Status: %{http_code}\n"
}

# PUT avec gestion d'erreurs
update_user_rest() {
    local user_id="$1"
    local name="$2"
    local email="$3"
    
    local payload=$(cat <<EOF
{
    "name": "$name",
    "email": "$email"
}
EOF
)
    
    local response=$(curl -s -w "\n%{http_code}" \
         -X PUT "$API_BASE_URL/users/$user_id" \
         -H "$(get_headers | head -1)" \
         -H "$(get_headers | tail -1)" \
         -d "$payload")
    
    local http_code=$(echo "$response" | tail -1)
    local body=$(echo "$response" | sed '$d')
    
    if [ "$http_code" -eq 200 ]; then
        echo "$body" | jq .
    else
        echo "Erreur: HTTP $http_code" >&2
        echo "$body" >&2
        return 1
    fi
}

# DELETE avec confirmation
delete_user_rest() {
    local user_id="$1"
    local confirm="${2:-no}"
    
    if [ "$confirm" != "yes" ]; then
        echo "Confirmation requise: delete_user_rest $user_id yes"
        return 1
    fi
    
    curl -X DELETE "$API_BASE_URL/users/$user_id" \
         -H "$(get_headers | tail -1)" \
         -w "\nHTTP Status: %{http_code}\n"
}
```

### Gestion avancée REST

**Gestion des erreurs et retry** :
```bash
#!/bin/bash
# Gestion avancée REST avec retry et backoff

rest_request_with_retry() {
    local method="$1"
    local url="$2"
    local data="${3:-}"
    local max_retries="${4:-3}"
    local retry_delay="${5:-1}"
    
    local attempt=1
    
    while [ $attempt -le $max_retries ]; do
        local response=$(curl -s -w "\n%{http_code}" \
            -X "$method" \
            "$url" \
            -H "Content-Type: application/json" \
            ${data:+-d "$data"})
        
        local http_code=$(echo "$response" | tail -1)
        local body=$(echo "$response" | sed '$d')
        
        if [ "$http_code" -ge 200 ] && [ "$http_code" -lt 300 ]; then
            echo "$body"
            return 0
        elif [ "$http_code" -ge 500 ] && [ $attempt -lt $max_retries ]; then
            echo "Tentative $attempt échouée (HTTP $http_code), retry dans ${retry_delay}s..." >&2
            sleep $retry_delay
            retry_delay=$((retry_delay * 2))  # Exponential backoff
            attempt=$((attempt + 1))
        else
            echo "Erreur HTTP $http_code" >&2
            echo "$body" >&2
            return 1
        fi
    done
    
    return 1
}
```

## Utilisation gRPC avec grpcurl

### Installation et configuration

**Installation grpcurl** :
```bash
#!/bin/bash
# Installation de grpcurl selon la plateforme

install_grpcurl() {
    local platform=$(uname -s | tr '[:upper:]' '[:lower:]')
    local arch=$(uname -m)
    
    case "$platform" in
        linux)
            if [ "$arch" = "x86_64" ]; then
                wget https://github.com/fullstorydev/grpcurl/releases/download/v1.8.7/grpcurl_1.8.7_linux_x86_64.tar.gz
                tar -xzf grpcurl_*.tar.gz
                sudo mv grpcurl /usr/local/bin/
                rm grpcurl_*.tar.gz
            fi
            ;;
        darwin)
            if command -v brew >/dev/null 2>&1; then
                brew install grpcurl
            else
                echo "Installez Homebrew pour macOS"
                return 1
            fi
            ;;
        *)
            echo "Plateforme non supportée: $platform"
            return 1
            ;;
    esac
    
    echo "grpcurl installé avec succès"
    grpcurl --version
}
```

### Requêtes gRPC complètes

**Script gRPC complet** :
```bash
#!/bin/bash
# Script gRPC complet avec grpcurl

set -euo pipefail

GRPC_ENDPOINT="${GRPC_ENDPOINT:-localhost:50051}"
PROTO_DIR="${PROTO_DIR:-./proto}"

# Lister les services
list_grpc_services() {
    grpcurl -plaintext "$GRPC_ENDPOINT" list
}

# Décrire un service
describe_service() {
    local service="$1"
    
    grpcurl -plaintext \
            -import-path "$PROTO_DIR" \
            -proto "${service}.proto" \
            "$GRPC_ENDPOINT" \
            describe "$service"
}

# Appel unary (requête-réponse simple)
grpc_unary_call() {
    local service="$1"
    local method="$2"
    local json_data="$3"
    
    grpcurl -plaintext \
            -d "$json_data" \
            "$GRPC_ENDPOINT" \
            "${service}.${method}"
}

# Appel streaming serveur
grpc_server_stream() {
    local service="$1"
    local method="$2"
    local json_data="$3"
    
    grpcurl -plaintext \
            -d "$json_data" \
            "$GRPC_ENDPOINT" \
            "${service}.${method}" | \
    while IFS= read -r line; do
        echo "Réponse: $line"
        # Traitement de chaque message streamé
    done
}

# Appel streaming client
grpc_client_stream() {
    local service="$1"
    local method="$2"
    local input_file="$3"
    
    # Envoyer plusieurs messages depuis un fichier
    grpcurl -plaintext \
            -d @ \
            "$GRPC_ENDPOINT" \
            "${service}.${method}" < "$input_file"
}

# Appel bidirectionnel
grpc_bidirectional_stream() {
    local service="$1"
    local method="$2"
    
    # Nécessite une implémentation plus complexe
    # Généralement fait avec un client généré
    echo "Streaming bidirectionnel nécessite un client généré"
}

# Exemple complet : Service utilisateur
user_service_example() {
    # Créer un utilisateur
    create_user_grpc() {
        local name="$1"
        local email="$2"
        
        local payload=$(cat <<EOF
{
    "name": "$name",
    "email": "$email"
}
EOF
)
        
        grpc_unary_call "UserService" "CreateUser" "$payload"
    }
    
    # Récupérer un utilisateur
    get_user_grpc() {
        local user_id="$1"
        
        local payload=$(cat <<EOF
{
    "id": $user_id
}
EOF
)
        
        grpc_unary_call "UserService" "GetUser" "$payload"
    }
    
    # Lister les utilisateurs (streaming)
    list_users_grpc() {
        grpc_server_stream "UserService" "ListUsers" "{}"
    }
}
```

### Gestion avancée gRPC

**Gestion des erreurs et métadonnées** :
```bash
#!/bin/bash
# Gestion avancée gRPC

grpc_call_with_metadata() {
    local service="$1"
    local method="$2"
    local json_data="$3"
    local metadata="$4"  # Format: "key1:value1,key2:value2"
    
    local metadata_args=()
    IFS=',' read -ra METADATA_ARRAY <<< "$metadata"
    for item in "${METADATA_ARRAY[@]}"; do
        IFS=':' read -r key value <<< "$item"
        metadata_args+=(-H "$key: $value")
    done
    
    grpcurl -plaintext \
            "${metadata_args[@]}" \
            -d "$json_data" \
            "$GRPC_ENDPOINT" \
            "${service}.${method}"
}

# Appel avec TLS
grpc_call_tls() {
    local service="$1"
    local method="$2"
    local json_data="$3"
    local cert_file="${4:-}"
    
    local tls_args=()
    if [ -n "$cert_file" ]; then
        tls_args=(-cacert "$cert_file")
    fi
    
    grpcurl "${tls_args[@]}" \
            -d "$json_data" \
            "$GRPC_ENDPOINT" \
            "${service}.${method}"
}
```

## Cas d'usage et choix

### Quand choisir REST

**Scénarios REST optimaux** :
```bash
#!/bin/bash
# Exemples de cas d'usage REST

# 1. API publique web
public_web_api() {
    # REST est idéal car :
    # - Compatible navigateur
    # - Facile à déboguer (JSON lisible)
    # - Caching HTTP natif
    # - Documentation simple (OpenAPI/Swagger)
    
    curl "https://api.public-service.com/v1/users"
}

# 2. Prototypage rapide
rapid_prototyping() {
    # REST permet de démarrer rapidement
    # Pas besoin de définir des fichiers .proto
    # JSON flexible pour itérations rapides
    
    curl -X POST "http://localhost:3000/api/test" \
         -H "Content-Type: application/json" \
         -d '{"data": "test"}'
}

# 3. Intégration tierce
third_party_integration() {
    # La plupart des APIs tierces sont REST
    # Format standard et bien supporté
    
    curl "https://api.github.com/users/octocat"
}
```

### Quand choisir gRPC

**Scénarios gRPC optimaux** :
```bash
#!/bin/bash
# Exemples de cas d'usage gRPC

# 1. Communication inter-services (microservices)
microservices_communication() {
    # gRPC est idéal car :
    # - Performance élevée
    # - Contrats stricts
    # - Streaming natif
    # - Code généré automatiquement
    
    grpcurl -plaintext \
            -d '{"service_id": "user-service"}' \
            "service-mesh:50051" \
            "ServiceRegistry.GetService"
}

# 2. APIs internes haute performance
high_performance_internal() {
    # gRPC pour latence minimale
    # Format binaire plus rapide
    # Multiplexage HTTP/2
    
    grpcurl -plaintext \
            -d '{"query": "SELECT * FROM users"}' \
            "database-proxy:50051" \
            "DatabaseService.ExecuteQuery"
}

# 3. Streaming de données
data_streaming() {
    # gRPC supporte le streaming bidirectionnel
    # Idéal pour données temps réel
    
    grpcurl -plaintext \
            "streaming-service:50051" \
            "StreamingService.Subscribe" | \
    while read -r event; do
        process_event "$event"
    done
}
```

## Migration et coexistence

### Migration REST vers gRPC

**Stratégie de migration** :
```bash
#!/bin/bash
# Migration progressive REST vers gRPC

# Phase 1: Coexistence
coexistence_phase() {
    # Maintenir REST pour compatibilité
    # Ajouter gRPC en parallèle
    # Router selon le client
    
    route_request() {
        local client_type="$1"
        local endpoint="$2"
        
        case "$client_type" in
            web|mobile)
                # Utiliser REST
                curl "$endpoint"
                ;;
            internal|microservice)
                # Utiliser gRPC
                grpcurl -plaintext "$endpoint" "Service.Method"
                ;;
        esac
    }
}

# Phase 2: Proxy gRPC vers REST
grpc_to_rest_proxy() {
    # Créer un proxy qui traduit gRPC en REST
    # Permet migration progressive
    
    echo "Implémentation d'un proxy gRPC->REST"
    echo "Utiliser grpc-gateway ou similaire"
}

# Phase 3: Migration complète
full_migration() {
    # Une fois tous les clients migrés
    # Retirer REST, garder uniquement gRPC
    
    echo "Migration complète vers gRPC"
}
```

### Utilisation simultanée

**Architecture hybride** :
```bash
#!/bin/bash
# Architecture hybride REST + gRPC

hybrid_api_architecture() {
    # REST pour APIs publiques
    public_api_rest() {
        curl "https://api.example.com/public/users"
    }
    
    # gRPC pour communication interne
    internal_api_grpc() {
        grpcurl -plaintext \
                "internal-service:50051" \
                "InternalService.Method"
    }
    
    # Gateway qui route selon le contexte
    api_gateway() {
        local request_type="$1"
        
        if [ "$request_type" = "public" ]; then
            public_api_rest
        else
            internal_api_grpc
        fi
    }
}
```

## Performance et optimisation

### Optimisation REST

**Techniques d'optimisation REST** :
```bash
#!/bin/bash
# Optimisation REST

# 1. Compression
rest_with_compression() {
    curl -H "Accept-Encoding: gzip, deflate" \
         --compressed \
         "https://api.example.com/users"
}

# 2. Caching
rest_with_cache() {
    # Utiliser les headers HTTP de cache
    curl -H "If-None-Match: \"etag-value\"" \
         "https://api.example.com/users"
}

# 3. Pagination efficace
rest_pagination() {
    local page=1
    local limit=100
    
    while true; do
        local response=$(curl -s \
            "https://api.example.com/users?page=$page&limit=$limit")
        
        echo "$response" | jq -r '.data[]'
        
        local has_more=$(echo "$response" | jq -r '.has_more')
        if [ "$has_more" != "true" ]; then
            break
        fi
        
        page=$((page + 1))
    done
}

# 4. Requêtes parallèles
rest_parallel() {
    # Utiliser xargs pour paralléliser
    seq 1 10 | xargs -P 5 -I {} \
        curl -s "https://api.example.com/users/{}"
}
```

### Optimisation gRPC

**Techniques d'optimisation gRPC** :
```bash
#!/bin/bash
# Optimisation gRPC

# 1. Compression
grpc_with_compression() {
    grpcurl -plaintext \
            -H "grpc-encoding: gzip" \
            -d '{}' \
            "$GRPC_ENDPOINT" \
            "Service.Method"
}

# 2. Multiplexage HTTP/2
grpc_multiplexing() {
    # HTTP/2 permet plusieurs requêtes simultanées
    # sur une seule connexion TCP
    
    # Exécuter plusieurs requêtes en parallèle
    for i in {1..10}; do
        grpcurl -plaintext \
                -d "{\"id\": $i}" \
                "$GRPC_ENDPOINT" \
                "Service.Method" &
    done
    wait
}

# 3. Streaming efficace
grpc_efficient_streaming() {
    # Utiliser le streaming pour grandes quantités de données
    grpcurl -plaintext \
            "$GRPC_ENDPOINT" \
            "Service.StreamMethod" | \
    while IFS= read -r chunk; do
        # Traiter chaque chunk immédiatement
        process_chunk "$chunk"
    done
}

# 4. Connection pooling
grpc_connection_pool() {
    # Réutiliser les connexions HTTP/2
    # gRPC gère automatiquement le pooling
    # via HTTP/2
    
    echo "gRPC utilise automatiquement le connection pooling HTTP/2"
}
```

## Conclusion

REST et gRPC sont deux approches complémentaires pour construire des APIs modernes. REST excelle dans les APIs publiques, l'intégration web, et les cas d'usage nécessitant flexibilité et compatibilité. gRPC brille dans les communications inter-services, les environnements haute performance, et les applications nécessitant streaming et contrats stricts.

Le choix entre REST et gRPC dépend de vos besoins spécifiques : performance, compatibilité, complexité, et contexte d'utilisation. Dans de nombreux cas, une architecture hybride utilisant les deux approches selon le contexte offre le meilleur des deux mondes.

La maîtrise des deux technologies vous permet de choisir la meilleure solution pour chaque situation, créant des systèmes plus efficaces et maintenables.

Dans le chapitre suivant, nous explorerons le streaming et les Server-Sent Events, découvrant comment gérer les données en temps réel avec cURL et les APIs modernes.
