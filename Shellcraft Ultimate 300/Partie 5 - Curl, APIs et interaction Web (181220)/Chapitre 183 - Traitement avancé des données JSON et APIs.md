# Chapitre 183 - Traitement avancé des données JSON et APIs

> "JSON n'est pas seulement un format de données - c'est le langage universel qui permet aux APIs de communiquer, et jq est l'interprète qui donne du sens à cette conversation." - Expert en traitement de données

## Introduction : JSON comme lingua franca des APIs

JSON (JavaScript Object Notation) est devenu le format standard d'échange de données dans les APIs modernes. Avec jq comme outil de traitement spécialisé, le shell devient capable de manipuler, transformer et analyser des données structurées complexes avec une élégance surprenante. Dans ce chapitre, nous explorerons les techniques avancées de traitement JSON, combinant jq avec d'autres outils pour créer des pipelines de traitement de données complets.

De la transformation de données à l'analyse statistique, nous découvrirons comment le shell peut devenir un véritable processeur de données JSON haute performance.

## Section 1 : Maîtrise de jq pour le traitement JSON

### 1.1 Les bases essentielles de jq

```bash
# Installation de jq (si nécessaire)
# Ubuntu/Debian
sudo apt-get install jq

# CentOS/RHEL
sudo yum install jq

# macOS
brew install jq

# Windows (via Chocolatey)
choco install jq

# Test de l'installation
echo '{"test": "Hello jq!"}' | jq '.'

# Formatage et coloration syntaxique
curl -s "https://api.github.com/users/octocat" | jq '.'
```

### 1.2 Requêtes et filtres avancés

```bash
# Données de test
cat > sample_data.json << 'EOF'
{
  "company": "TechCorp",
  "employees": [
    {
      "id": 1,
      "name": "Alice Johnson",
      "department": "Engineering",
      "salary": 75000,
      "skills": ["Python", "JavaScript", "Docker"],
      "projects": [
        {"name": "WebApp", "status": "completed"},
        {"name": "MobileApp", "status": "in_progress"}
      ]
    },
    {
      "id": 2,
      "name": "Bob Smith",
      "department": "Engineering",
      "salary": 80000,
      "skills": ["Java", "Spring", "Kubernetes"],
      "projects": [
        {"name": "Microservices", "status": "completed"}
      ]
    },
    {
      "id": 3,
      "name": "Carol Davis",
      "department": "Marketing",
      "salary": 65000,
      "skills": ["SEO", "Content Marketing", "Analytics"],
      "projects": []
    }
  ],
  "departments": {
    "Engineering": {
      "budget": 500000,
      "headcount": 2
    },
    "Marketing": {
      "budget": 200000,
      "headcount": 1
    }
  }
}
EOF

# Requêtes de base
jq '.company' sample_data.json
jq '.employees | length' sample_data.json
jq '.employees[0]' sample_data.json

# Filtrage et sélection
jq '.employees[] | select(.department == "Engineering")' sample_data.json
jq '.employees[] | select(.salary > 70000) | {name, salary}' sample_data.json

# Tris et agrégations
jq '.employees | sort_by(.salary) | reverse | .[:2]' sample_data.json
jq '.employees | map(.salary) | add / length' sample_data.json  # Salaire moyen

# Manipulation de tableaux
jq '.employees | map(.skills) | flatten | unique' sample_data.json  # Toutes les compétences uniques
jq '.employees[] | {name, project_count: (.projects | length)}' sample_data.json

# Requêtes imbriquées
jq '.employees[] | select(.department == "Engineering") | .projects[] | select(.status == "completed") | .name' sample_data.json

# Transformations complexes
jq '{
  company: .company,
  summary: {
    total_employees: (.employees | length),
    departments: (.employees | group_by(.department) | map({(.[0].department): length})),
    avg_salary: (.employees | map(.salary) | add / length),
    skills_distribution: (.employees | map(.skills) | flatten | group_by(.) | map({(.[0]): length}))
  }
}' sample_data.json
```

### 1.3 Fonctions et expressions avancées

```bash
# Définition de fonctions dans jq
cat > advanced_queries.jq << 'EOF'
# Fonction pour calculer des statistiques
def stats:
  {
    count: length,
    sum: add,
    avg: (add / length),
    min: min,
    max: max
  };

# Fonction pour analyser les employés
def analyze_employee:
  {
    name: .name,
    department: .department,
    salary_level: (
      if .salary > 80000 then "Senior"
      elif .salary > 60000 then "Mid"
      else "Junior"
      end
    ),
    skill_count: (.skills | length),
    project_completion: (
      (.projects | length) as $total |
      (.projects | map(select(.status == "completed")) | length) as $completed |
      if $total > 0 then ($completed / $total * 100) else 0 end
    )
  };

# Fonction pour grouper par département
def group_by_department:
  group_by(.department) | map({
    department: .[0].department,
    employees: map(analyze_employee),
    total_salary: map(.salary) | add,
    avg_salary: (map(.salary) | add / length)
  });

# Fonction récursive pour aplatir des structures imbriquées
def flatten:
  if type == "object" then
    to_entries | map(.value | flatten) | flatten
  elif type == "array" then
    map(flatten) | flatten
  else
    .
  end;
EOF

# Utilisation des fonctions définies
jq -f advanced_queries.jq sample_data.json

# Combinaisons avec des arguments
jq --arg dept "Engineering" '.employees[] | select(.department == $dept)' sample_data.json

# Variables et substitutions
DEPT="Engineering"
jq --arg dept "$DEPT" '.employees[] | select(.department == $dept)' sample_data.json
```

## Section 2 : Intégration avec APIs REST

### 2.1 Pipeline complet : API → traitement → sortie

```bash
#!/bin/bash
# Pipeline complet de traitement d'APIs

# Configuration
API_BASE="https://jsonplaceholder.typicode.com"
OUTPUT_DIR="api_data"
LOG_FILE="api_pipeline.log"

# Fonction de logging
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# Fonction de récupération de données API
fetch_api_data() {
    local endpoint="$1"
    local output_file="$2"
    
    log "Récupération: $endpoint"
    
    if curl -s -H "Accept: application/json" "$API_BASE$endpoint" | jq '.' > "$output_file" 2>/dev/null; then
        log "✓ Données sauvegardées: $output_file"
        return 0
    else
        log "✗ Échec de récupération: $endpoint"
        return 1
    fi
}

# Fonction de traitement des données utilisateurs
process_users() {
    local input_file="$1"
    local output_file="$2"
    
    log "Traitement des utilisateurs"
    
    jq '{
        total_users: length,
        users: map({
            id: .id,
            name: .name,
            username: .username,
            email: .email,
            company: .company.name,
            city: .address.city,
            website: .website
        }),
        stats: {
            companies: map(.company.name) | unique | length,
            cities: map(.address.city) | unique | length,
            domains: map(.email | split("@")[1]) | unique
        }
    }' "$input_file" > "$output_file"
    
    log "✓ Utilisateurs traités: $output_file"
}

# Fonction de traitement des posts
process_posts() {
    local input_file="$1"
    local output_file="$2"
    
    log "Traitement des posts"
    
    jq '{
        total_posts: length,
        posts_by_user: group_by(.userId) | map({
            user_id: .[0].userId,
            post_count: length,
            titles: map(.title)
        }),
        longest_title: max_by(.title | length) | .title,
        shortest_title: min_by(.title | length) | .title,
        avg_title_length: (map(.title | length) | add / length | round)
    }' "$input_file" > "$output_file"
    
    log "✓ Posts traités: $output_file"
}

# Fonction d'analyse croisée
cross_analyze() {
    local users_file="$1"
    local posts_file="$2"
    local output_file="$3"
    
    log "Analyse croisée utilisateurs/posts"
    
    jq -s '{
        users: .[0].users,
        posts: .[1]
    } | {
        analysis: {
            user_post_distribution: .posts.posts_by_user,
            top_contributors: (.posts.posts_by_user | sort_by(.post_count) | reverse | .[:3]),
            inactive_users: (.users | map(.id) as $user_ids | .posts.posts_by_user | map(.user_id) as $active_users | $user_ids - $active_users | map({user_id: ., status: "inactive"}))
        }
    }' "$users_file" "$posts_file" > "$output_file"
    
    log "✓ Analyse croisée terminée: $output_file"
}

# Fonction de génération de rapport
generate_report() {
    local users_processed="$1"
    local posts_processed="$2"
    local cross_analysis="$3"
    local output_file="$4"
    
    log "Génération du rapport final"
    
    {
        echo "# Rapport d'analyse d'API"
        echo "Généré le: $(date)"
        echo
        
        echo "## Statistiques générales"
        jq -r '"- Utilisateurs totaux: \(.total_users)"' "$users_processed"
        jq -r '"- Posts totaux: \(.total_posts)"' "$posts_processed"
        
        echo
        echo "## Top contributeurs"
        jq -r '.analysis.top_contributors[] | "- User \(.user_id): \(.post_count) posts"' "$cross_analysis"
        
        echo
        echo "## Domaines email"
        jq -r '.stats.domains[] | "- \(.): \(length) utilisateurs"' "$users_processed"
        
        echo
        echo "## Analyse des longueurs de titre"
        jq -r '"- Plus long titre: \(.longest_title | length) caractères"' "$posts_processed"
        jq -r '"- Plus court titre: \(.shortest_title | length) caractères"' "$posts_processed"
        jq -r '"- Longueur moyenne: \(.avg_title_length) caractères"' "$posts_processed"
        
    } > "$output_file"
    
    log "✓ Rapport généré: $output_file"
}

# Pipeline principal
main() {
    mkdir -p "$OUTPUT_DIR"
    
    log "=== DÉMARRAGE DU PIPELINE API ==="
    
    # Étape 1: Récupération des données
    fetch_api_data "/users" "$OUTPUT_DIR/users_raw.json"
    fetch_api_data "/posts" "$OUTPUT_DIR/posts_raw.json"
    
    # Étape 2: Traitement des données
    process_users "$OUTPUT_DIR/users_raw.json" "$OUTPUT_DIR/users_processed.json"
    process_posts "$OUTPUT_DIR/posts_raw.json" "$OUTPUT_DIR/posts_processed.json"
    
    # Étape 3: Analyse croisée
    cross_analyze "$OUTPUT_DIR/users_processed.json" "$OUTPUT_DIR/posts_processed.json" "$OUTPUT_DIR/cross_analysis.json"
    
    # Étape 4: Génération de rapport
    generate_report "$OUTPUT_DIR/users_processed.json" "$OUTPUT_DIR/posts_processed.json" "$OUTPUT_DIR/cross_analysis.json" "api_analysis_report.md"
    
    log "=== PIPELINE TERMINÉ ==="
    log "Fichiers générés dans: $OUTPUT_DIR"
    log "Rapport final: api_analysis_report.md"
}

# Exécution du pipeline
main

# Nettoyage
rm -f sample_data.json advanced_queries.jq
```

### 2.2 Gestion d'erreurs et robustesse

```bash
#!/bin/bash
# Pipeline API robuste avec gestion d'erreurs

# Configuration
API_BASE="https://jsonplaceholder.typicode.com"
MAX_RETRIES=3
RETRY_DELAY=2
TIMEOUT=30

# Fonction de requête API avec retry
api_request() {
    local endpoint="$1"
    local output_file="$2"
    local attempt=1
    
    while [[ $attempt -le $MAX_RETRIES ]]; do
        log "Tentative $attempt/$MAX_RETRIES pour $endpoint"
        
        if curl -s \
                --max-time $TIMEOUT \
                -H "Accept: application/json" \
                -w "HTTP_CODE:%{http_code}\n" \
                "$API_BASE$endpoint" > "${output_file}.tmp" 2>/dev/null; then
            
            local response=$(cat "${output_file}.tmp")
            local http_code=$(echo "$response" | grep "HTTP_CODE:" | cut -d: -f2)
            local content=$(echo "$response" | sed '/HTTP_CODE:/d')
            
            if [[ "$http_code" =~ ^2 ]]; then
                # Validation JSON
                if echo "$content" | jq empty 2>/dev/null; then
                    echo "$content" > "$output_file"
                    log "✓ Succès: $endpoint (HTTP $http_code)"
                    rm -f "${output_file}.tmp"
                    return 0
                else
                    log "✗ JSON invalide reçu de $endpoint"
                fi
            else
                log "✗ Erreur HTTP $http_code pour $endpoint"
            fi
        else
            log "✗ Échec de connexion pour $endpoint"
        fi
        
        attempt=$((attempt + 1))
        if [[ $attempt -le $MAX_RETRIES ]]; then
            log "Nouvelle tentative dans ${RETRY_DELAY}s..."
            sleep $RETRY_DELAY
        fi
    done
    
    log "✗ Échec définitif pour $endpoint après $MAX_RETRIES tentatives"
    rm -f "${output_file}.tmp"
    return 1
}

# Fonction de validation des données
validate_json_data() {
    local file="$1"
    local schema_check="$2"
    
    if [[ ! -f "$file" ]]; then
        log "✗ Fichier non trouvé: $file"
        return 1
    fi
    
    # Validation JSON de base
    if ! jq empty "$file" 2>/dev/null; then
        log "✗ JSON mal formé dans $file"
        return 1
    fi
    
    # Validation de schéma basique
    if [[ -n "$schema_check" ]]; then
        case "$schema_check" in
            "users")
                if ! jq 'all(.[]; has("id") and has("name") and has("email"))' "$file" | grep -q true; then
                    log "✗ Schéma utilisateurs invalide dans $file"
                    return 1
                fi
                ;;
            "posts")
                if ! jq 'all(.[]; has("id") and has("title") and has("body"))' "$file" | grep -q true; then
                    log "✗ Schéma posts invalide dans $file"
                    return 1
                fi
                ;;
        esac
    fi
    
    log "✓ Validation réussie pour $file"
    return 0
}

# Fonction de traitement avec rollback
process_with_rollback() {
    local input_file="$1"
    local output_file="$2"
    local processor="$3"
    
    # Sauvegarde du fichier de sortie existant
    if [[ -f "$output_file" ]]; then
        cp "$output_file" "${output_file}.backup"
        log "Sauvegarde créée: ${output_file}.backup"
    fi
    
    # Tentative de traitement
    if $processor "$input_file" "$output_file"; then
        log "✓ Traitement réussi: $output_file"
        # Nettoyage de la sauvegarde
        rm -f "${output_file}.backup"
        return 0
    else
        log "✗ Échec du traitement, rollback..."
        # Restauration de la sauvegarde
        if [[ -f "${output_file}.backup" ]]; then
            mv "${output_file}.backup" "$output_file"
            log "✓ Rollback effectué"
        fi
        return 1
    fi
}

# Pipeline robuste
robust_pipeline() {
    local endpoints=("users" "posts" "comments" "albums")
    local failed_endpoints=()
    
    log "=== PIPELINE ROBUSTE - DÉMARRAGE ==="
    
    # Phase 1: Récupération des données
    log "Phase 1: Récupération des données"
    for endpoint in "${endpoints[@]}"; do
        if ! api_request "/$endpoint" "raw_${endpoint}.json"; then
            failed_endpoints+=("$endpoint")
        fi
    done
    
    if [[ ${#failed_endpoints[@]} -gt 0 ]]; then
        log "⚠️ Échec de récupération pour: ${failed_endpoints[*]}"
        if [[ ${#failed_endpoints[@]} -eq ${#endpoints[@]} ]]; then
            log "✗ Tous les endpoints ont échoué, arrêt du pipeline"
            return 1
        fi
    fi
    
    # Phase 2: Validation des données
    log "Phase 2: Validation des données"
    for endpoint in "${endpoints[@]}"; do
        if [[ -f "raw_${endpoint}.json" ]]; then
            if ! validate_json_data "raw_${endpoint}.json" "$endpoint"; then
                log "✗ Validation échouée pour $endpoint"
                failed_endpoints+=("$endpoint")
            fi
        fi
    done
    
    # Phase 3: Traitement avec rollback
    log "Phase 3: Traitement des données"
    for endpoint in "${endpoints[@]}"; do
        if [[ ! " ${failed_endpoints[*]} " =~ " ${endpoint} " ]]; then
            case "$endpoint" in
                "users")
                    process_with_rollback "raw_users.json" "processed_users.json" "process_users_data"
                    ;;
                "posts")
                    process_with_rollback "raw_posts.json" "processed_posts.json" "process_posts_data"
                    ;;
            esac
        fi
    done
    
    # Phase 4: Génération de rapport
    log "Phase 4: Génération de rapport"
    generate_final_report "${endpoints[@]}" > "pipeline_report_$(date +%Y%m%d_%H%M%S).md"
    
    log "=== PIPELINE ROBUSTE - TERMINÉ ==="
}

# Exécution du pipeline robuste
robust_pipeline
```

## Section 3 : Performance et optimisation

### 3.1 Optimisation des requêtes jq

```bash
# Benchmarking des expressions jq
benchmark_jq() {
    local json_file="$1"
    local expression="$2"
    local iterations="${3:-10}"
    
    echo "Benchmarking: $expression"
    echo "Fichier: $(basename "$json_file")"
    echo "Itérations: $iterations"
    echo
    
    local total_time=0
    
    for ((i=1; i<=iterations; i++)); do
        local start=$(date +%s.%3N)
        jq "$expression" "$json_file" > /dev/null
        local end=$(date +%s.%3N)
        local duration=$(echo "$end - $start" | bc -l 2>/dev/null || echo "0")
        total_time=$(echo "$total_time + $duration" | bc -l 2>/dev/null || echo "$total_time")
        
        printf "Itération %2d: %.3fs\n" $i $duration
    done
    
    local avg_time=$(echo "scale=3; $total_time / $iterations" | bc -l 2>/dev/null || echo "0")
    echo
    printf "Temps moyen: %.3fs\n" $avg_time
}

# Comparaison d'expressions
echo "=== COMPARAISON DE PERFORMANCES JQ ==="

# Création de données de test volumineuses
jq -n '{
    data: [range(10000) | {
        id: .,
        value: (. * 2),
        category: (["A", "B", "C"][. % 3]),
        nested: {
            prop1: "value\(. % 10)",
            prop2: (. % 100)
        }
    }]
}' > large_dataset.json

echo
echo "Test 1: Comptage simple"
benchmark_jq large_dataset.json '.data | length'

echo
echo "Test 2: Filtrage simple"
benchmark_jq large_dataset.json '.data[] | select(.value > 5000) | .id'

echo
echo "Test 3: Agrégation"
benchmark_jq large_dataset.json '.data | group_by(.category) | map({category: .[0].category, count: length})'

echo
echo "Test 4: Tri"
benchmark_jq large_dataset.json '.data | sort_by(.value) | reverse | .[:10]'

echo
echo "Test 5: Transformation complexe"
benchmark_jq large_dataset.json '.data | map({
    id,
    value,
    category,
    computed: (.value * 1.1),
    nested_sum: (.nested.prop2 + .value)
})'
```

### 3.2 Streaming et traitement en continu

```bash
# Traitement de gros fichiers JSON en streaming
stream_process_large_json() {
    local input_file="$1"
    local chunk_size="${2:-1000}"
    
    log "Traitement en streaming: $input_file (chunks de $chunk_size)"
    
    # Comptage total des éléments
    local total_count=$(jq '.data | length' "$input_file")
    log "Éléments totaux: $total_count"
    
    # Traitement par chunks
    local processed=0
    local chunk_num=1
    
    while [[ $processed -lt $total_count ]]; do
        local end_index=$((processed + chunk_size))
        [[ $end_index -gt $total_count ]] && end_index=$total_count
        
        log "Traitement du chunk $chunk_num: éléments $((processed + 1))-$end_index"
        
        # Extraction du chunk
        jq ".data[$processed:$end_index]" "$input_file" > "chunk_${chunk_num}.json"
        
        # Traitement du chunk
        local chunk_result=$(jq 'map({
            id,
            category,
            processed_value: (.value * 2),
            is_even: (.value % 2 == 0)
        })' "chunk_${chunk_num}.json")
        
        # Sauvegarde des résultats
        echo "$chunk_result" > "processed_chunk_${chunk_num}.json"
        
        # Mise à jour des compteurs
        processed=$end_index
        chunk_num=$((chunk_num + 1))
        
        # Nettoyage du chunk temporaire
        rm "chunk_${chunk_num}.json"
    done
    
    # Fusion des résultats
    jq -s 'flatten' processed_chunk_*.json > "final_result.json"
    
    # Nettoyage
    rm processed_chunk_*.json
    
    log "✓ Traitement terminé: $(jq '. | length' final_result.json) éléments traités"
}

# Traitement de flux JSON (ndjson)
process_ndjson_stream() {
    local input_file="$1"
    local output_file="$2"
    
    log "Traitement de flux NDJSON: $input_file"
    
    # Création d'un fichier NDJSON de test
    if [[ "$input_file" == "create_test" ]]; then
        for i in {1..10000}; do
            echo "{\"id\": $i, \"value\": $((i * 10)), \"timestamp\": \"$(date +%s)\"}"
        done > test_stream.ndjson
        input_file="test_stream.ndjson"
    fi
    
    # Traitement ligne par ligne (streaming)
    local total_processed=0
    local sum_values=0
    
    while IFS= read -r line; do
        # Validation JSON de chaque ligne
        if echo "$line" | jq -e '.id and .value' > /dev/null 2>&1; then
            local id=$(echo "$line" | jq -r '.id')
            local value=$(echo "$line" | jq -r '.value')
            
            # Transformation
            local transformed=$(jq -n \
                --arg id "$id" \
                --argjson value "$value" \
                '{id: $id, original_value: $value, doubled_value: ($value * 2), processed_at: now}')
            
            # Accumulation pour statistiques
            sum_values=$((sum_values + value))
            total_processed=$((total_processed + 1))
            
            # Écriture dans le fichier de sortie (NDJSON)
            echo "$transformed" >> "$output_file"
        else
            log "⚠️ Ligne JSON invalide ignorée: $line"
        fi
    done < "$input_file"
    
    # Statistiques finales
    local avg_value=$((sum_values / total_processed))
    log "✓ Traitement terminé: $total_processed lignes, valeur moyenne: $avg_value"
    
    # Ajout des métadonnées
    local metadata=$(jq -n \
        --arg total "$total_processed" \
        --arg sum "$sum_values" \
        --arg avg "$avg_value" \
        '{_metadata: {total_processed: $total, sum_values: $sum, avg_value: $avg} + now}')
    
    echo "$metadata" >> "$output_file"
}

# Exécution des tests de performance
stream_process_large_json "large_dataset.json" 500
process_ndjson_stream "create_test" "stream_output.ndjson"

# Nettoyage
rm -f large_dataset.json test_stream.ndjson stream_output.ndjson final_result.json
```

### 3.3 Cache et optimisation mémoire

```bash
# Système de cache pour les requêtes API
JSON_CACHE_DIR="${HOME}/.api_cache"
mkdir -p "$JSON_CACHE_DIR"

# Fonction de cache intelligent
cached_api_request() {
    local endpoint="$1"
    local cache_duration="${2:-300}"  # 5 minutes par défaut
    
    # Génération de clé de cache basée sur l'endpoint
    local cache_key=$(echo "$endpoint" | md5sum | cut -d' ' -f1)
    local cache_file="$JSON_CACHE_DIR/${cache_key}.json"
    
    # Vérification du cache
    if [[ -f "$cache_file" ]]; then
        local file_age=$(( $(date +%s) - $(stat -c %Y "$cache_file" 2>/dev/null || date +%s) ))
        
        if [[ $file_age -lt $cache_duration ]]; then
            log "✓ Cache hit pour $endpoint (âge: ${file_age}s)"
            cat "$cache_file"
            return 0
        else
            log "Cache expiré pour $endpoint (âge: ${file_age}s)"
        fi
    fi
    
    # Requête API
    log "Requête API: $endpoint"
    local response=$(curl -s -H "Accept: application/json" "$API_BASE$endpoint")
    
    if [[ $? -eq 0 ]]; then
        # Validation JSON
        if echo "$response" | jq empty 2>/dev/null; then
            # Mise en cache
            echo "$response" > "$cache_file"
            log "✓ Cache mis à jour pour $endpoint"
            echo "$response"
            return 0
        else
            log "✗ Réponse JSON invalide pour $endpoint"
            return 1
        fi
    else
        log "✗ Échec de la requête pour $endpoint"
        return 1
    fi
}

# Fonction de préchargement du cache
preload_cache() {
    local endpoints=("$@")
    
    log "Préchargement du cache pour ${#endpoints[@]} endpoints"
    
    for endpoint in "${endpoints[@]}"; do
        if cached_api_request "$endpoint" 3600; then
            log "✓ $endpoint préchargé"
        else
            log "✗ Échec du préchargement pour $endpoint"
        fi
        
        # Petit délai pour éviter de surcharger l'API
        sleep 0.5
    done
}

# Optimisation mémoire pour le traitement de gros JSON
memory_efficient_processing() {
    local input_file="$1"
    local output_file="$2"
    
    log "Traitement optimisé mémoire: $input_file"
    
    # Utilisation de jq en streaming pour éviter de charger tout en mémoire
    jq -c '.data[]' "$input_file" | while IFS= read -r item; do
        # Traitement élément par élément
        local processed_item=$(echo "$item" | jq '{
            id: .id,
            category: .category,
            processed_value: (.value * 1.5),
            timestamp: now
        }')
        
        # Écriture immédiate pour libérer la mémoire
        echo "$processed_item" >> "$output_file"
    done
    
    log "✓ Traitement terminé: $(wc -l < "$output_file") éléments traités"
}

# Exemple d'utilisation
preload_cache "/users" "/posts" "/comments"
cached_api_request "/users"  # Utilisera le cache
memory_efficient_processing "large_dataset.json" "processed_output.json"

# Nettoyage du cache
rm -rf "$JSON_CACHE_DIR"
```

## Conclusion : jq comme processeur de données universel

jq transforme le shell d'un simple exécuteur de commandes en un véritable processeur de données JSON haute performance. Combiné avec les capacités d'APIs de cURL, il crée un écosystème complet pour l'extraction, la transformation et l'analyse de données structurées à l'échelle.

Dans le prochain chapitre, nous explorerons l'automatisation des déploiements et des opérations DevOps avec les outils modernes comme Ansible, Terraform, et les pipelines CI/CD.

---

**Exercice pratique :** Créez un système complet de collecte et d'analyse de données APIs qui :
1. Récupère des données depuis plusieurs APIs publiques
2. Met en cache intelligemment les résultats
3. Effectue des transformations complexes avec jq
4. Génère des rapports statistiques
5. Implémente une surveillance en temps réel

**Challenge avancé :** Développez un processeur de données JSON distribué qui :
- Traite des flux de données en parallèle
- Implémente la reprise sur erreur
- Optimise l'utilisation mémoire
- Supporte des transformations complexes
- Fournit des métriques de performance

**Réflexion :** Comment jq et les outils JSON changent-ils notre approche du traitement de données dans le shell, et quels sont les impacts sur l'architecture des systèmes de données modernes ?

