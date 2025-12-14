# Chapitre 190 - Traitement JSON et XML

## Table des matières
- [Introduction](#introduction)
- [Traitement JSON](#traitement-json)
- [Traitement XML](#traitement-xml)
- [Conversion entre formats](#conversion-entre-formats)
- [Conclusion](#conclusion)

## Introduction

Le traitement de JSON et XML est essentiel pour travailler avec les APIs modernes. Ce chapitre couvre l'extraction, la transformation, et la manipulation de ces formats.

## Traitement JSON

**Manipulation JSON avec jq** :
```bash
#!/bin/bash
# Traitement JSON

# Extraire des valeurs
extract_json_value() {
    local json="$1"
    local key="$2"
    echo "$json" | jq -r ".$key"
}

# Filtrer des données
filter_json() {
    local json="$1"
    local filter="$2"
    echo "$json" | jq "$filter"
}

# Transformer JSON
transform_json() {
    local json="$1"
    echo "$json" | jq '{
        id: .id,
        name: .name | ascii_upcase,
        email: .email
    }'
}
```

## Traitement XML

**Manipulation XML** :
```bash
#!/bin/bash
# Traitement XML

# Extraire avec xmllint
extract_xml_value() {
    local xml="$1"
    local xpath="$2"
    echo "$xml" | xmllint --xpath "$xpath" -
}

# Valider XML
validate_xml() {
    local xml_file="$1"
    xmllint --noout "$xml_file"
}
```

## Conversion entre formats

**Conversion JSON/XML** :
```bash
#!/bin/bash
# Conversion entre formats

# JSON vers XML (avec outils Python)
json_to_xml() {
    python3 -c "
import json, xml.etree.ElementTree as ET
data = json.loads('$1')
root = ET.Element('root')
# Conversion...
"
}
```

## Conclusion

Le traitement de JSON et XML est essentiel pour l'intégration avec les APIs modernes.

