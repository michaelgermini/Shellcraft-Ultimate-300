# Chapitre 268 - Analyse de données et scripts multi-plateforme

## Table des matières
- [Introduction](#introduction)
- [Analyse de données avec outils shell](#analyse-de-données-avec-outils-shell)
- [Scripts multi-plateforme](#scripts-multi-plateforme)
- [Traitement de données avec IA](#traitement-de-données-avec-ia)
- [Visualisation de données](#visualisation-de-données)
- [Conclusion](#conclusion)

## Introduction

L'analyse de données dans le terminal combine la puissance des outils shell traditionnels avec les capacités modernes de l'intelligence artificielle. Créer des scripts qui fonctionnent sur plusieurs plateformes tout en analysant efficacement les données est une compétence essentielle pour l'automatisation moderne.

Imaginez l'analyse de données multi-plateforme comme un traducteur universel : il comprend les données dans n'importe quel format, sur n'importe quelle plateforme, et les transforme en insights actionnables, peu importe où elles proviennent.

## Analyse de données avec outils shell

### Analyse avec awk et sed

**Analyse de logs avec awk** :
```bash
#!/bin/bash
# Analyse de logs avec awk

analyze_logs() {
    local log_file="$1"
    
    echo "=== Analyse des logs ==="
    
    # Top 10 des IPs
    echo "Top 10 IPs:"
    awk '{print $1}' "$log_file" | sort | uniq -c | sort -rn | head -10
    
    # Codes de statut HTTP
    echo -e "\nCodes de statut HTTP:"
    awk '{print $9}' "$log_file" | sort | uniq -c | sort -rn
    
    # Pages les plus visitées
    echo -e "\nPages les plus visitées:"
    awk '{print $7}' "$log_file" | sort | uniq -c | sort -rn | head -10
    
    # Analyse temporelle
    echo -e "\nRequêtes par heure:"
    awk '{print $4}' "$log_file" | cut -d: -f2 | sort | uniq -c
}

# Utilisation
analyze_logs "/var/log/nginx/access.log"
```

**Analyse CSV avec awk** :
```bash
#!/bin/bash
# Analyse de fichiers CSV

analyze_csv() {
    local csv_file="$1"
    local column="${2:-1}"
    
    # Statistiques de base
    echo "=== Statistiques CSV ==="
    echo "Nombre de lignes: $(wc -l < "$csv_file")"
    echo "Nombre de colonnes: $(head -1 "$csv_file" | awk -F',' '{print NF}')"
    
    # Analyse d'une colonne spécifique
    echo -e "\nAnalyse de la colonne $column:"
    awk -F',' -v col="$column" 'NR>1 {sum+=$col; count++; if($col>max) max=$col; if(NR==2 || $col<min) min=$col} 
    END {print "Moyenne: " sum/count "\nMax: " max "\nMin: " min}' "$csv_file"
}
```

### Analyse avec jq (JSON)

**Analyse JSON avancée** :
```bash
#!/bin/bash
# Analyse JSON avec jq

analyze_json() {
    local json_file="$1"
    
    echo "=== Analyse JSON ==="
    
    # Structure du JSON
    echo "Structure:"
    jq 'paths(scalars) as $p | {path: $p | map(tostring) | join("."), type: getpath($p) | type}' "$json_file"
    
    # Statistiques sur les arrays
    echo -e "\nArrays:"
    jq '[paths(scalars) | length] | max' "$json_file"
    
    # Extraction de données spécifiques
    echo -e "\nDonnées extraites:"
    jq '.users[] | {name: .name, email: .email}' "$json_file"
}

# Analyse d'API JSON
analyze_api_response() {
    local api_url="$1"
    
    local response=$(curl -s "$api_url")
    
    echo "Nombre d'éléments: $(echo "$response" | jq 'length')"
    echo "Clés disponibles: $(echo "$response" | jq 'keys')"
    
    # Filtrer et transformer
    echo "$response" | jq '.[] | select(.status == "active") | {id, name}'
}
```

## Scripts multi-plateforme

### Détection de plateforme

**Script de détection universel** :
```bash
#!/bin/bash
# Détection de plateforme universelle

detect_platform() {
    local os=""
    local arch=""
    
    # Détection OS
    case "$(uname -s)" in
        Linux*)     os="linux" ;;
        Darwin*)    os="macos" ;;
        CYGWIN*)    os="windows" ;;
        MINGW*)     os="windows" ;;
        MSYS*)      os="windows" ;;
        *)          os="unknown" ;;
    esac
    
    # Détection architecture
    case "$(uname -m)" in
        x86_64)     arch="amd64" ;;
        i386|i686)  arch="386" ;;
        armv7l)     arch="arm" ;;
        aarch64)    arch="arm64" ;;
        *)          arch="unknown" ;;
    esac
    
    echo "${os}_${arch}"
}

# Utilisation
PLATFORM=$(detect_platform)
echo "Plateforme détectée: $PLATFORM"
```

### Scripts adaptatifs

**Script multi-plateforme adaptatif** :
```bash
#!/bin/bash
# Script qui s'adapte à la plateforme

# Détection de plateforme
if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    PACKAGE_MANAGER="apt-get"
    SERVICE_CMD="systemctl"
    USER_ADD="useradd"
elif [[ "$OSTYPE" == "darwin"* ]]; then
    PACKAGE_MANAGER="brew"
    SERVICE_CMD="launchctl"
    USER_ADD="dscl"
elif [[ "$OSTYPE" == "msys" ]] || [[ "$OSTYPE" == "cygwin" ]]; then
    PACKAGE_MANAGER="choco"
    SERVICE_CMD="sc"
    USER_ADD="net user"
fi

# Fonctions adaptatives
install_package() {
    local package="$1"
    
    case "$PACKAGE_MANAGER" in
        apt-get)
            sudo apt-get update && sudo apt-get install -y "$package"
            ;;
        brew)
            brew install "$package"
            ;;
        choco)
            choco install "$package" -y
            ;;
    esac
}

manage_service() {
    local service="$1"
    local action="$2"
    
    case "$SERVICE_CMD" in
        systemctl)
            sudo systemctl "$action" "$service"
            ;;
        launchctl)
            launchctl "$action" "$service"
            ;;
        sc)
            sc "$action" "$service"
            ;;
    esac
}
```

### Gestion de chemins multi-plateforme

**Normalisation de chemins** :
```bash
#!/bin/bash
# Gestion de chemins multi-plateforme

normalize_path() {
    local path="$1"
    
    # Windows paths
    if [[ "$OSTYPE" == "msys" ]] || [[ "$OSTYPE" == "cygwin" ]]; then
        # Convertir C:\path\to\file en /c/path/to/file
        path=$(echo "$path" | sed 's|^\([A-Z]\):|/\1|' | tr '\\' '/')
    fi
    
    echo "$path"
}

get_home_dir() {
    if [[ "$OSTYPE" == "msys" ]] || [[ "$OSTYPE" == "cygwin" ]]; then
        echo "$USERPROFILE"
    else
        echo "$HOME"
    fi
}

get_temp_dir() {
    if [[ "$OSTYPE" == "msys" ]] || [[ "$OSTYPE" == "cygwin" ]]; then
        echo "$TEMP"
    else
        echo "/tmp"
    fi
}
```

## Traitement de données avec IA

### Analyse intelligente de données

**Analyseur de données avec IA** :
```bash
#!/bin/bash
# Analyse de données avec IA

analyze_data_with_ai() {
    local data_file="$1"
    local data_type="${2:-auto}"  # csv, json, log, auto
    
    # Détecter le type si auto
    if [[ "$data_type" == "auto" ]]; then
        if [[ "$data_file" == *.csv ]]; then
            data_type="csv"
        elif [[ "$data_file" == *.json ]]; then
            data_type="json"
        elif [[ "$data_file" == *.log ]]; then
            data_type="log"
        fi
    fi
    
    # Préparer les données pour l'IA
    local data_sample=$(head -100 "$data_file")
    local data_stats=$(get_data_stats "$data_file" "$data_type")
    
    # Analyser avec IA
    local analysis=$(ai_analyze "Analyse ces données et fournis des insights:
Type: $data_type
Statistiques: $data_stats
Échantillon:
$data_sample

Fournis:
1. Patterns identifiés
2. Anomalies potentielles
3. Recommandations
4. Visualisations suggérées")
    
    echo "$analysis"
}

get_data_stats() {
    local file="$1"
    local type="$2"
    
    case "$type" in
        csv)
            echo "Lignes: $(wc -l < "$file"), Colonnes: $(head -1 "$file" | awk -F',' '{print NF}')"
            ;;
        json)
            echo "Éléments: $(jq 'length' "$file" 2>/dev/null || echo "N/A")"
            ;;
        log)
            echo "Lignes: $(wc -l < "$file"), Taille: $(du -h "$file" | cut -f1)"
            ;;
    esac
}
```

### Génération de rapports avec IA

**Générateur de rapports intelligent** :
```bash
#!/bin/bash
# Génération de rapports avec IA

generate_ai_report() {
    local data_file="$1"
    local report_type="${2:-summary}"  # summary, detailed, executive
    
    # Analyser les données
    local analysis=$(analyze_data_with_ai "$data_file")
    
    # Générer le rapport selon le type
    case "$report_type" in
        summary)
            ai_generate "Génère un résumé exécutif (1 page) basé sur cette analyse:
$analysis" > report_summary.md
            ;;
        detailed)
            ai_generate "Génère un rapport détaillé avec graphiques et recommandations basé sur:
$analysis" > report_detailed.md
            ;;
        executive)
            ai_generate "Génère un rapport exécutif avec métriques clés et actions recommandées:
$analysis" > report_executive.md
            ;;
    esac
    
    echo "Rapport généré: report_${report_type}.md"
}
```

## Visualisation de données

### Graphiques ASCII

**Visualisation en terminal** :
```bash
#!/bin/bash
# Graphiques ASCII dans le terminal

draw_bar_chart() {
    local data_file="$1"
    local x_column="${2:-1}"
    local y_column="${3:-2}"
    local width="${4:-50}"
    
    echo "=== Graphique en barres ==="
    
    awk -F',' -v x="$x_column" -v y="$y_column" -v w="$width" '
    NR>1 {
        label = $x
        value = $y
        bars = int((value / max) * w)
        printf "%-20s ", label
        for (i=0; i<bars; i++) printf "█"
        printf " %d\n", value
    }
    NR==2 {max = $y}
    $y > max {max = $y}' "$data_file"
}

draw_line_chart() {
    local data_file="$1"
    local height="${2:-20}"
    
    echo "=== Graphique linéaire ==="
    
    # Calculer les valeurs normalisées
    awk -F',' 'NR>1 {print $2}' "$data_file" | \
    awk -v h="$height" '
    {
        values[NR] = $1
        if ($1 > max) max = $1
    }
    END {
        for (i=1; i<=NR; i++) {
            bars = int((values[i] / max) * h)
            for (j=0; j<bars; j++) printf "█"
            printf " %.2f\n", values[i]
        }
    }'
}
```

### Export vers formats visuels

**Génération de graphiques** :
```bash
#!/bin/bash
# Génération de graphiques avec gnuplot ou alternatives

generate_chart() {
    local data_file="$1"
    local chart_type="${2:-bar}"  # bar, line, pie
    local output_file="${3:-chart.png}"
    
    if command -v gnuplot &> /dev/null; then
        case "$chart_type" in
            bar)
                gnuplot << EOF
set terminal png
set output "$output_file"
set style data histograms
set style fill solid
plot "$data_file" using 2:xtic(1) title "Données"
EOF
                ;;
            line)
                gnuplot << EOF
set terminal png
set output "$output_file"
plot "$data_file" using 1:2 with lines title "Tendance"
EOF
                ;;
        esac
    else
        echo "gnuplot non installé, utilisation de la visualisation ASCII"
        draw_bar_chart "$data_file"
    fi
}
```

## Conclusion

L'analyse de données et les scripts multi-plateforme combinent la puissance des outils shell traditionnels avec la flexibilité nécessaire pour fonctionner sur différents systèmes. En ajoutant l'intelligence artificielle à ce mélange, nous créons des outils d'analyse qui s'adaptent automatiquement aux données et aux plateformes.

Cette approche permet de créer des solutions d'analyse robustes qui fonctionnent partout, analysent efficacement, et fournissent des insights actionnables, peu importe l'environnement.

Dans le chapitre suivant, nous explorerons les workflows avancés multi-plateforme avec IA, découvrant comment orchestrer des processus complexes qui s'adaptent intelligemment aux différents environnements.

