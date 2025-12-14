# Chapitre 201 - Traitement et transformation de données

## Table des matières
- [Introduction](#introduction)
- [Transformation JSON](#transformation-json)
- [Filtrage et agrégation](#filtrage-et-agrégation)
- [Export de données](#export-de-données)
- [Conclusion](#conclusion)

## Introduction

Le traitement et la transformation de données permettent de manipuler les réponses API pour les adapter à vos besoins spécifiques.

## Transformation JSON

**Transformation avec jq** :
```bash
#!/bin/bash
# Transformation JSON

transform_json() {
    local json="$1"
    local transformation="$2"
    
    echo "$json" | jq "$transformation"
}

# Exemple: Extraire et renommer
extract_and_rename() {
    local json="$1"
    echo "$json" | jq '{
        id: .user_id,
        nom: .name,
        email: .email_address
    }'
}
```

## Filtrage et agrégation

**Filtrage avancé** :
```bash
#!/bin/bash
# Filtrage et agrégation

filter_data() {
    local json="$1"
    local condition="$2"
    
    echo "$json" | jq ".[] | select($condition)"
}

# Agrégation
aggregate_data() {
    local json="$1"
    echo "$json" | jq 'group_by(.category) | map({
        category: .[0].category,
        count: length,
        total: map(.amount) | add
    })'
}
```

## Export de données

**Export multi-formats** :
```bash
#!/bin/bash
# Export de données

export_to_csv() {
    local json="$1"
    local output="$2"
    
    echo "$json" | jq -r '.[] | [.id, .name, .email] | @csv' > "$output"
}

export_to_xml() {
    local json="$1"
    local output="$2"
    
    # Conversion JSON vers XML
    echo "$json" | jq -r 'to_entries[] | "<\(.key)>\(.value)</\(.key)>"'
}
```

## Conclusion

Le traitement et la transformation de données permettent d'adapter les réponses API à vos besoins spécifiques.

