# Chapitre 213 - APIs versionnées

## Table des matières
- [Introduction](#introduction)
- [Gestion des versions](#gestion-des-versions)
- [Migration entre versions](#migration-entre-versions)
- [Conclusion](#conclusion)

## Introduction

La gestion des versions d'API est essentielle pour maintenir la compatibilité tout en évoluant.

## Gestion des versions

**Support multi-versions** :
```bash
#!/bin/bash
# Gestion des versions d'API

API_VERSION="${API_VERSION:-v1}"

api_call_versioned() {
    local endpoint="$1"
    local version="${2:-$API_VERSION}"
    
    curl -s "${API_BASE_URL}/${version}${endpoint}"
}
```

## Migration entre versions

**Migration automatique** :
```bash
#!/bin/bash
# Migration entre versions

migrate_to_version() {
    local new_version="$1"
    
    # Tester la nouvelle version
    if test_api_version "$new_version"; then
        export API_VERSION="$new_version"
        echo "Migration vers $new_version réussie"
    else
        echo "Migration échouée, garde de la version actuelle"
    fi
}
```

## Conclusion

La gestion des versions d'API garantit la compatibilité et l'évolutivité.

