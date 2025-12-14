# Chapitre 197 - Web scraping

## Table des matières
- [Introduction](#introduction)
- [Scraping de base](#scraping-de-base)
- [Extraction de données](#extraction-de-données)
- [Gestion des sessions](#gestion-des-sessions)
- [Conclusion](#conclusion)

## Introduction

Le web scraping avec cURL permet d'extraire des données de sites web pour l'analyse et l'automatisation.

## Scraping de base

**Scraping simple** :
```bash
#!/bin/bash
# Web scraping de base

scrape_page() {
    local url="$1"
    local output_file="${2:-output.html}"
    
    curl -s "$url" > "$output_file"
    echo "Page sauvegardée: $output_file"
}

# Scraping avec headers
scrape_with_headers() {
    local url="$1"
    
    curl -s \
         -H "User-Agent: Mozilla/5.0" \
         -H "Accept: text/html" \
         "$url"
}
```

## Extraction de données

**Extraction avec outils** :
```bash
#!/bin/bash
# Extraction de données

extract_links() {
    local html_file="$1"
    
    # Extraire tous les liens
    grep -oP 'href="\K[^"]+' "$html_file"
}

extract_text() {
    local html_file="$1"
    local selector="${2:-p}"
    
    # Utiliser html-xml-utils ou pup si disponible
    if command -v pup &> /dev/null; then
        pup "$selector text{}" < "$html_file"
    else
        # Fallback avec grep/sed
        grep -oP "<$selector[^>]*>.*?</$selector>" "$html_file" | \
        sed 's/<[^>]*>//g'
    fi
}
```

## Gestion des sessions

**Scraping avec session** :
```bash
#!/bin/bash
# Scraping avec session

scrape_with_session() {
    local base_url="$1"
    local session_file=".scraping_session"
    
    # Première requête pour obtenir les cookies
    curl -c "$session_file" -s "$base_url" > /dev/null
    
    # Requêtes suivantes avec les cookies
    curl -b "$session_file" -s "${base_url}/protected"
}
```

## Conclusion

Le web scraping avec cURL permet d'automatiser l'extraction de données depuis le web.

