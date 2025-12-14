# Chapitre 204 - Projets pratiques avancés

## Table des matières
- [Introduction](#introduction)
- [Projet 1 : Système de synchronisation](#projet-1--système-de-synchronisation)
- [Projet 2 : API Gateway](#projet-2--api-gateway)
- [Projet 3 : Monitoring distribué](#projet-3--monitoring-distribué)
- [Conclusion](#conclusion)

## Introduction

Ces projets avancés combinent toutes les techniques apprises pour créer des systèmes complets et production-ready.

## Projet 1 : Système de synchronisation

**Synchronisation bidirectionnelle** :
```bash
#!/bin/bash
# Système de synchronisation

sync_bidirectional() {
    local local_data="local.json"
    local api_endpoint="/data"
    
    # Récupérer les données
    local api_data=$(get_resource "$api_endpoint")
    local local_data_content=$(cat "$local_data" 2>/dev/null || echo "{}")
    
    # Détecter les conflits
    detect_conflicts "$api_data" "$local_data_content"
    
    # Synchroniser
    merge_data "$api_data" "$local_data_content"
}
```

## Projet 2 : API Gateway

**Gateway simple** :
```bash
#!/bin/bash
# API Gateway

gateway_request() {
    local endpoint="$1"
    local method="${2:-GET}"
    
    # Routage
    case "$endpoint" in
        /users*)
            forward_to_service "user-service" "$endpoint" "$method"
            ;;
        /orders*)
            forward_to_service "order-service" "$endpoint" "$method"
            ;;
        *)
            echo "404 Not Found"
            return 1
            ;;
    esac
}
```

## Projet 3 : Monitoring distribué

**Monitoring multi-services** :
```bash
#!/bin/bash
# Monitoring distribué

monitor_services() {
    local services=("api1" "api2" "api3")
    
    for service in "${services[@]}"; do
        (
            check_service "$service"
        ) &
    done
    
    wait
    generate_aggregated_report
}
```

## Conclusion

Ces projets avancés démontrent l'application réelle de toutes les techniques apprises.

