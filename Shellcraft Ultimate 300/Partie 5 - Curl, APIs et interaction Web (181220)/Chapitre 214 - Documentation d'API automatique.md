# Chapitre 214 - Documentation d'API automatique

## Table des matières
- [Introduction](#introduction)
- [Génération de documentation](#génération-de-documentation)
- [Tests de documentation](#tests-de-documentation)
- [Conclusion](#conclusion)

## Introduction

La documentation automatique d'API garantit que la documentation reste synchronisée avec l'API réelle.

## Génération de documentation

**Génération depuis OpenAPI** :
```bash
#!/bin/bash
# Génération de documentation

generate_docs() {
    local openapi_spec="$1"
    
    # Générer la documentation avec redoc-cli ou swagger-ui
    if command -v redoc-cli &> /dev/null; then
        redoc-cli bundle "$openapi_spec" -o docs.html
    fi
}
```

## Tests de documentation

**Validation de la documentation** :
```bash
#!/bin/bash
# Tests de documentation

test_documentation() {
    local endpoint="$1"
    local expected_docs="$2"
    
    # Vérifier que l'endpoint correspond à la documentation
    local actual_response=$(curl -s "$endpoint")
    validate_against_spec "$actual_response" "$expected_docs"
}
```

## Conclusion

La documentation automatique maintient la cohérence entre l'API et sa documentation.

