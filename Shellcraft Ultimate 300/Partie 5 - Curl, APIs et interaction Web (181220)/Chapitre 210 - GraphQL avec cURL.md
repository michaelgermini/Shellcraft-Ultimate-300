# Chapitre 210 - GraphQL avec cURL

## Table des matières
- [Introduction](#introduction)
- [Requêtes GraphQL](#requêtes-graphql)
- [Variables GraphQL](#variables-graphql)
- [Mutations GraphQL](#mutations-graphql)
- [Conclusion](#conclusion)

## Introduction

GraphQL est une alternative aux APIs REST qui permet des requêtes plus flexibles et efficaces.

## Requêtes GraphQL

**Requêtes GraphQL de base** :
```bash
#!/bin/bash
# Requêtes GraphQL

graphql_query() {
    local endpoint="$1"
    local query="$2"
    
    curl -X POST "$endpoint" \
         -H "Content-Type: application/json" \
         -d "{\"query\":\"$query\"}"
}

# Exemple
get_users() {
    local query='{
        users {
            id
            name
            email
        }
    }'
    
    graphql_query "https://api.example.com/graphql" "$query"
}
```

## Variables GraphQL

**Requêtes avec variables** :
```bash
#!/bin/bash
# GraphQL avec variables

graphql_with_variables() {
    local endpoint="$1"
    local query="$2"
    local variables="$3"
    
    curl -X POST "$endpoint" \
         -H "Content-Type: application/json" \
         -d "{\"query\":\"$query\",\"variables\":$variables}"
}
```

## Mutations GraphQL

**Mutations** :
```bash
#!/bin/bash
# Mutations GraphQL

graphql_mutation() {
    local endpoint="$1"
    local mutation="$2"
    local variables="$3"
    
    graphql_with_variables "$endpoint" "$mutation" "$variables"
}
```

## Conclusion

GraphQL offre une alternative puissante aux APIs REST avec cURL.

