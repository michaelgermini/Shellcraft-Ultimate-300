# Chapitre 194 - Scripts automatisés multi-plateforme

## Table des matières
- [Introduction](#introduction)
- [Détection de plateforme](#détection-de-plateforme)
- [Scripts adaptatifs](#scripts-adaptatifs)
- [Gestion des différences](#gestion-des-différences)
- [Conclusion](#conclusion)

## Introduction

Les scripts automatisés multi-plateforme fonctionnent sur Linux, macOS, et Windows, s'adaptant automatiquement aux différences entre systèmes.

## Détection de plateforme

**Détection automatique** :
```bash
#!/bin/bash
# Détection de plateforme

detect_platform() {
    case "$(uname -s)" in
        Linux*)     echo "linux" ;;
        Darwin*)    echo "macos" ;;
        CYGWIN*|MINGW*|MSYS*) echo "windows" ;;
        *)          echo "unknown" ;;
    esac
}

PLATFORM=$(detect_platform)

# Adapter curl selon la plateforme
get_curl_command() {
    case "$PLATFORM" in
        windows)
            # Windows peut nécessiter des options différentes
            echo "curl.exe"
            ;;
        *)
            echo "curl"
            ;;
    esac
}
```

## Scripts adaptatifs

**Script adaptatif** :
```bash
#!/bin/bash
# Script adaptatif multi-plateforme

# Fonction API adaptative
api_call() {
    local endpoint="$1"
    local method="${2:-GET}"
    local data="${3:-}"
    
    local curl_cmd=$(get_curl_command)
    local url="${API_BASE_URL}${endpoint}"
    
    # Adapter selon la plateforme
    case "$PLATFORM" in
        windows)
            # Windows peut nécessiter des échappements différents
            if [ -n "$data" ]; then
                $curl_cmd -X "$method" -d "$data" "$url"
            else
                $curl_cmd -X "$method" "$url"
            fi
            ;;
        *)
            # Linux/macOS
            if [ -n "$data" ]; then
                curl -X "$method" -d "$data" "$url"
            else
                curl -X "$method" "$url"
            fi
            ;;
    esac
}
```

## Gestion des différences

**Gestion des chemins** :
```bash
#!/bin/bash
# Gestion des chemins multi-plateforme

get_config_dir() {
    case "$PLATFORM" in
        windows)
            echo "$APPDATA/myapp"
            ;;
        macos)
            echo "$HOME/Library/Application Support/myapp"
            ;;
        linux)
            echo "$HOME/.config/myapp"
            ;;
    esac
}

get_temp_dir() {
    case "$PLATFORM" in
        windows)
            echo "$TEMP"
            ;;
        *)
            echo "/tmp"
            ;;
    esac
}
```

## Conclusion

Les scripts multi-plateforme nécessitent une détection et une adaptation automatiques pour fonctionner sur tous les systèmes.

