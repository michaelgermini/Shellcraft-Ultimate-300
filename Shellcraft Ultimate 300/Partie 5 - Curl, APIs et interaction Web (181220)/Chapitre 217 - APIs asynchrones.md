# Chapitre 217 - APIs asynchrones

## Table des matières
- [Introduction](#introduction)
- [Requêtes asynchrones](#requêtes-asynchrones)
- [Polling](#polling)
- [Conclusion](#conclusion)

## Introduction

Les APIs asynchrones permettent de gérer des opérations longues sans bloquer les requêtes.

## Requêtes asynchrones

**Gestion asynchrone** :
```bash
#!/bin/bash
# Requêtes asynchrones

async_request() {
    local url="$1"
    
    # Démarrer la tâche
    local job_id=$(curl -s -X POST "$url/tasks" | jq -r '.job_id')
    
    # Polling jusqu'à completion
    while true; do
        local status=$(curl -s "$url/tasks/$job_id" | jq -r '.status')
        
        if [ "$status" == "completed" ]; then
            curl -s "$url/tasks/$job_id/result"
            break
        elif [ "$status" == "failed" ]; then
            echo "Tâche échouée"
            return 1
        fi
        
        sleep 2
    done
}
```

## Polling

**Polling intelligent** :
```bash
#!/bin/bash
# Polling avec backoff

poll_with_backoff() {
    local url="$1"
    local max_attempts="${2:-10}"
    local delay="${3:-2}"
    
    local attempt=1
    while [ $attempt -le $max_attempts ]; do
        local response=$(curl -s "$url")
        
        if [ "$(echo "$response" | jq -r '.ready')" == "true" ]; then
            echo "$response"
            return 0
        fi
        
        sleep $delay
        delay=$((delay * 2))  # Backoff exponentiel
        attempt=$((attempt + 1))
    done
    
    return 1
}
```

## Conclusion

Les APIs asynchrones permettent de gérer efficacement les opérations longues.

