# Chapitre 199 - Tests automatisés d'API

## Table des matières
- [Introduction](#introduction)
- [Tests unitaires](#tests-unitaires)
- [Tests d'intégration](#tests-dintégration)
- [Tests de charge](#tests-de-charge)
- [Conclusion](#conclusion)

## Introduction

Les tests automatisés d'API garantissent la qualité et la fiabilité des services web en validant automatiquement leur comportement.

## Tests unitaires

**Framework de tests simple** :
```bash
#!/bin/bash
# Framework de tests API

test_count=0
pass_count=0
fail_count=0

# Assertion
assert() {
    local description="$1"
    local condition="$2"
    
    test_count=$((test_count + 1))
    
    if eval "$condition"; then
        echo "✓ $description"
        pass_count=$((pass_count + 1))
        return 0
    else
        echo "✗ $description"
        fail_count=$((fail_count + 1))
        return 1
    fi
}

# Test de réponse HTTP
test_http_status() {
    local url="$1"
    local expected_status="$2"
    
    local actual_status=$(curl -s -o /dev/null -w "%{http_code}" "$url")
    
    assert "HTTP status should be $expected_status" \
           "[ \"$actual_status\" -eq \"$expected_status\" ]"
}

# Test de contenu JSON
test_json_contains() {
    local url="$1"
    local key="$2"
    local expected_value="$3"
    
    local response=$(curl -s "$url")
    local actual_value=$(echo "$response" | jq -r ".$key")
    
    assert "JSON should contain $key=$expected_value" \
           "[ \"$actual_value\" == \"$expected_value\" ]"
}

# Résumé des tests
test_summary() {
    echo
    echo "=== Résumé ==="
    echo "Total: $test_count"
    echo "Réussis: $pass_count"
    echo "Échoués: $fail_count"
    
    if [ $fail_count -eq 0 ]; then
        return 0
    else
        return 1
    fi
}
```

## Tests d'intégration

**Tests d'intégration** :
```bash
#!/bin/bash
# Tests d'intégration API

test_user_workflow() {
    # Créer un utilisateur
    local create_response=$(create_resource "/users" '{"name":"Test User","email":"test@example.com"}')
    local user_id=$(echo "$create_response" | jq -r '.id')
    
    # Lire l'utilisateur
    test_http_status "${API_BASE_URL}/users/$user_id" 200
    
    # Mettre à jour l'utilisateur
    update_resource "/users" "$user_id" '{"name":"Updated User"}'
    test_json_contains "${API_BASE_URL}/users/$user_id" "name" "Updated User"
    
    # Supprimer l'utilisateur
    delete_resource "/users" "$user_id"
    test_http_status "${API_BASE_URL}/users/$user_id" 404
}
```

## Tests de charge

**Tests de charge simples** :
```bash
#!/bin/bash
# Tests de charge

load_test() {
    local url="$1"
    local concurrent="${2:-10}"
    local requests="${3:-100}"
    
    echo "Test de charge: $concurrent requêtes concurrentes, $requests requêtes totales"
    
    for i in $(seq 1 $requests); do
        (
            curl -s -o /dev/null -w "%{time_total}\n" "$url"
        ) &
        
        # Limiter la concurrence
        if [ $((i % concurrent)) -eq 0 ]; then
            wait
        fi
    done
    
    wait
    echo "Test terminé"
}
```

## Conclusion

Les tests automatisés d'API garantissent la qualité et la fiabilité des services web.

