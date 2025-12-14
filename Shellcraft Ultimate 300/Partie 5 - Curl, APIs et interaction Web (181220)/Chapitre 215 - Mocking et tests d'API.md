# Chapitre 215 - Mocking et tests d'API

## Table des matières
- [Introduction](#introduction)
- [Mocking d'API](#mocking-dapi)
- [Tests avec mocks](#tests-avec-mocks)
- [Conclusion](#conclusion)

## Introduction

Le mocking permet de tester vos scripts sans dépendre d'APIs réelles, améliorant la vitesse et la fiabilité des tests.

## Mocking d'API

**Serveur mock simple** :
```bash
#!/bin/bash
# Serveur mock

mock_server() {
    local port="${1:-8080}"
    
    while true; do
        echo -e "HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n{\"mock\":\"data\"}" | \
        nc -l "$port"
    done
}
```

## Tests avec mocks

**Tests avec serveur mock** :
```bash
#!/bin/bash
# Tests avec mocks

test_with_mock() {
    # Démarrer le serveur mock
    mock_server 8080 &
    local mock_pid=$!
    
    # Tester contre le mock
    local response=$(curl -s http://localhost:8080/test)
    
    # Vérifier la réponse
    assert_equals "$response" '{"mock":"data"}'
    
    # Arrêter le mock
    kill $mock_pid
}
```

## Conclusion

Le mocking améliore la qualité des tests en isolant les dépendances externes.

