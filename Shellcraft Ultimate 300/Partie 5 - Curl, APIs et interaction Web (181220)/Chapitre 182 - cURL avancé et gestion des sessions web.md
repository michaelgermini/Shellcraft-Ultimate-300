# Chapitre 182 - cURL avancé et gestion des sessions web

> "cURL n'est pas seulement un outil pour télécharger des fichiers - c'est un protocole d'échange universel qui transforme le shell en client web complet, capable de dialoguer avec n'importe quel serveur sur Internet." - Daniel Stenberg, créateur de cURL

## Introduction : Au-delà des requêtes simples

Si le chapitre précédent nous a familiarisés avec les bases du protocole HTTP et des APIs REST, nous plongeons maintenant dans les fonctionnalités avancées de cURL qui en font l'outil de référence pour les interactions web complexes. Sessions persistantes, téléchargements parallèles, gestion des cookies, et automatisation de workflows complets deviennent possibles grâce à ces capacités avancées.

Dans ce chapitre, nous explorerons comment cURL transcende son rôle d'outil simple pour devenir un véritable orchestrateur d'interactions web, capable de reproduire des comportements de navigateurs et d'automatiser des processus métier complexes.

## Section 1 : Gestion des sessions et cookies

### 1.1 Persistance des sessions

```bash
# Création d'un fichier cookie pour persister les sessions
COOKIE_JAR="session_cookies.txt"

# Première requête - récupération des cookies
curl -c "$COOKIE_JAR" -b "$COOKIE_JAR" \
  -X GET "https://example.com/login" \
  -v

# Connexion avec envoi des credentials
curl -c "$COOKIE_JAR" -b "$COOKIE_JAR" \
  -X POST "https://example.com/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=myuser&password=mypass" \
  -L  # Suivre les redirections

# Accès aux pages protégées avec les cookies
curl -c "$COOKIE_JAR" -b "$COOKIE_JAR" \
  -X GET "https://example.com/dashboard" \
  -o dashboard.html

# Inspection des cookies stockés
cat "$COOKIE_JAR"
# Format: domain.com	TRUE	/	FALSE	1640995200	session_id	abc123...

# Nettoyage
rm "$COOKIE_JAR"
```

### 1.2 Simulation d'un navigateur complet

```bash
#!/bin/bash
# Script de simulation de navigation web

# Configuration
TARGET_SITE="https://example.com"
USERNAME="testuser"
PASSWORD="testpass"
USER_AGENT="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.124 Safari/537.36"
COOKIE_FILE="browser_cookies.txt"

# Fonction de logging
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*" >&2
}

# Fonction pour faire une requête avec gestion d'erreurs
web_request() {
    local method="$1"
    local url="$2"
    local data="$3"
    local referer="$4"
    
    local headers=(-H "User-Agent: $USER_AGENT")
    
    if [[ -n "$referer" ]]; then
        headers+=(-H "Referer: $referer")
    fi
    
    local curl_cmd=(curl -s -c "$COOKIE_FILE" -b "$COOKIE_FILE" -L -w "HTTP_STATUS:%{http_code};TOTAL_TIME:%{time_total};SIZE_DOWNLOAD:%{size_download}\n")
    
    if [[ "$method" == "POST" ]]; then
        curl_cmd+=(-X POST -d "$data")
    fi
    
    curl_cmd+=("${headers[@]}" "$url")
    
    log "Requête: $method $url"
    local response="$("${curl_cmd[@]}")"
    local http_info="${response##*$'\n'}"
    local content="${response%$'\n'*}"
    
    # Extraction des métriques
    local status_code=$(echo "$http_info" | cut -d';' -f1 | cut -d':' -f2)
    local total_time=$(echo "$http_info" | cut -d';' -f2 | cut -d':' -f2)
    local download_size=$(echo "$http_info" | cut -d';' -f3 | cut -d':' -f2)
    
    log "Status: $status_code, Time: ${total_time}s, Size: ${download_size} bytes"
    
    if [[ "$status_code" -ge 400 ]]; then
        log "Erreur HTTP: $status_code"
        return 1
    fi
    
    echo "$content"
}

# Simulation d'une session de navigation
log "=== Simulation de navigation web ==="

# 1. Accès à la page d'accueil
log "1. Accès à la page d'accueil"
home_page=$(web_request "GET" "$TARGET_SITE")
if [[ $? -ne 0 ]]; then
    log "Erreur: Impossible d'accéder à la page d'accueil"
    exit 1
fi

# Extraction du token CSRF si nécessaire
csrf_token=$(echo "$home_page" | grep -o 'csrf_token.*value="[^"]*"' | sed 's/.*value="//;s/".*//')
if [[ -n "$csrf_token" ]]; then
    log "Token CSRF trouvé: $csrf_token"
fi

# 2. Tentative de connexion
log "2. Tentative de connexion"
login_data="username=$USERNAME&password=$PASSWORD"
if [[ -n "$csrf_token" ]]; then
    login_data="$login_data&csrf_token=$csrf_token"
fi

login_response=$(web_request "POST" "$TARGET_SITE/login" "$login_data" "$TARGET_SITE")
if [[ $? -ne 0 ]]; then
    log "Erreur de connexion"
    exit 1
fi

# Vérifier si la connexion a réussi (recherche d'indicateurs)
if echo "$login_response" | grep -q "logout\|dashboard\|Welcome"; then
    log "✓ Connexion réussie"
else
    log "✗ Échec de la connexion"
    exit 1
fi

# 3. Accès à une page protégée
log "3. Accès au tableau de bord"
dashboard=$(web_request "GET" "$TARGET_SITE/dashboard" "" "$TARGET_SITE/login")
if [[ $? -eq 0 ]]; then
    log "✓ Accès au tableau de bord réussi"
    
    # Sauvegarde du contenu pour analyse
    echo "$dashboard" > "dashboard_$(date +%Y%m%d_%H%M%S).html"
else
    log "✗ Échec d'accès au tableau de bord"
fi

# 4. Déconnexion
log "4. Déconnexion"
logout_response=$(web_request "GET" "$TARGET_SITE/logout" "" "$TARGET_SITE/dashboard")
if [[ $? -eq 0 ]]; then
    log "✓ Déconnexion réussie"
else
    log "✗ Erreur lors de la déconnexion"
fi

# Nettoyage
rm -f "$COOKIE_FILE"

log "=== Session terminée ==="
```

### 1.3 Gestion avancée des cookies

```bash
# Manipulation manuelle des cookies
# Format Netscape: domain.com	TRUE	/	FALSE	1640995200	session_id	value

# Création d'un cookie personnalisé
echo -e "example.com\tTRUE\t/\tFALSE\t$(($(date +%s) + 86400))\tsession_id\tcustom_session_123" > custom_cookie.txt

# Utilisation du cookie personnalisé
curl -b custom_cookie.txt -c response_cookies.txt \
  -X GET "https://example.com/protected" \
  -v

# Modification de cookies existants
# Script pour modifier la valeur d'un cookie
modify_cookie() {
    local cookie_file="$1"
    local cookie_name="$2"
    local new_value="$3"
    
    if [[ ! -f "$cookie_file" ]]; then
        echo "Fichier cookie non trouvé: $cookie_file"
        return 1
    fi
    
    # Utilisation de sed pour modifier la valeur du cookie
    sed -i "s/\($cookie_name\t[^\t]*\t[^\t]*\t[^\t]*\t[^\t]*\t[^\t]*\t\)[^\t]*/\1$new_value/" "$cookie_file"
}

# Exemple: modifier la valeur du cookie de session
modify_cookie "session_cookies.txt" "session_id" "hacked_session_456"

# Inspection détaillée des cookies
inspect_cookies() {
    local cookie_file="$1"
    
    echo "=== Inspection des cookies: $cookie_file ==="
    
    if [[ ! -s "$cookie_file" ]]; then
        echo "Fichier cookie vide ou inexistant"
        return 1
    fi
    
    while IFS=$'\t' read -r domain flag path secure expiry name value; do
        # Ignorer les lignes de commentaires
        [[ "$domain" =~ ^# ]] && continue
        
        echo "Cookie: $name"
        echo "  Domaine: $domain"
        echo "  Chemin: $path"
        echo "  Sécurisé: $secure"
        echo "  Expiration: $(date -d "@$expiry" 2>/dev/null || echo 'Session')"
        echo "  Valeur: $value"
        echo
    done < "$cookie_file"
}

inspect_cookies "session_cookies.txt"

# Rotation automatique des sessions
rotate_session() {
    local base_url="$1"
    local username="$2"
    local password="$3"
    local cookie_file="rotated_session.txt"
    
    echo "Rotation de session pour $username sur $base_url"
    
    # Supprimer l'ancienne session
    rm -f "$cookie_file"
    
    # Créer une nouvelle session
    local login_response=$(curl -s -c "$cookie_file" -b "$cookie_file" \
        -X POST "$base_url/login" \
        -d "username=$username&password=$password")
    
    if echo "$login_response" | grep -q "success\|Welcome"; then
        echo "✓ Nouvelle session créée"
        echo "Cookies sauvegardés dans: $cookie_file"
        return 0
    else
        echo "✗ Échec de création de session"
        rm -f "$cookie_file"
        return 1
    fi
}

# Nettoyage
rm -f *.txt
```

## Section 2 : Téléchargements parallèles et gestion de files d'attente

### 2.1 Téléchargement parallèle avec xargs

```bash
# Téléchargement parallèle simple
cat urls.txt | xargs -n 1 -P 4 curl -s -O

# Avec gestion d'erreurs et logging
download_parallel() {
    local url_file="$1"
    local max_parallel="${2:-4}"
    local output_dir="${3:-downloads}"
    
    mkdir -p "$output_dir"
    
    # Fonction de téléchargement pour un seul fichier
    download_single() {
        local url="$1"
        local filename=$(basename "$url")
        local output_file="$output_dir/$filename"
        
        echo "[$(date +'%H:%M:%S')] Téléchargement: $filename"
        
        if curl -s -w "%{http_code}" -o "$output_file" "$url" | grep -q "^2"; then
            echo "[$(date +'%H:%M:%S')] ✓ Succès: $filename ($(stat -c%s "$output_file" 2>/dev/null || echo '?') bytes)"
        else
            echo "[$(date +'%H:%M:%S')] ✗ Échec: $filename"
            rm -f "$output_file"
        fi
    }
    
    export -f download_single
    export output_dir
    
    # Téléchargement parallèle avec xargs
    cat "$url_file" | xargs -n 1 -P "$max_parallel" -I {} bash -c 'download_single "$@"' _ {}
}

# Exemple d'utilisation
cat > urls.txt << EOF
https://example.com/file1.zip
https://example.com/file2.zip
https://example.com/file3.zip
https://example.com/file4.zip
https://example.com/file5.zip
EOF

download_parallel "urls.txt" 3 "my_downloads"

# Nettoyage
rm -f urls.txt
```

### 2.2 Gestionnaire de téléchargement avancé

```bash
#!/bin/bash
# Gestionnaire de téléchargement avancé avec reprise et parallélisation

# Configuration
MAX_PARALLEL=4
RETRY_ATTEMPTS=3
TIMEOUT=300
USER_AGENT="DownloadManager/1.0"
LOG_FILE="download_manager.log"

# Fonction de logging
log() {
    local level="$1"
    shift
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

# Fonction de téléchargement avec reprise
download_with_resume() {
    local url="$1"
    local output_file="$2"
    local attempt=1
    
    while [[ $attempt -le $RETRY_ATTEMPTS ]]; do
        log "INFO" "Tentative $attempt/$RETRY_ATTEMPTS pour $url"
        
        # Vérifier si le fichier existe partiellement
        local resume_option=""
        if [[ -f "$output_file" ]]; then
            local file_size=$(stat -c%s "$output_file" 2>/dev/null || echo 0)
            if [[ $file_size -gt 0 ]]; then
                resume_option="-C $file_size"
                log "INFO" "Reprise depuis $file_size bytes"
            fi
        fi
        
        # Téléchargement avec timeout et reprise
        local start_time=$(date +%s)
        if curl -s $resume_option \
                --max-time $TIMEOUT \
                --user-agent "$USER_AGENT" \
                -w "%{http_code};%{size_download};%{speed_download}" \
                -o "$output_file" \
                "$url" 2>>"$LOG_FILE" | {
            
            read -r http_code size speed
            local end_time=$(date +%s)
            local duration=$((end_time - start_time))
            
            if [[ "$http_code" =~ ^2 ]]; then
                log "SUCCESS" "Téléchargé: $(basename "$output_file") (${size} bytes, ${speed} B/s, ${duration}s)"
                return 0
            elif [[ "$http_code" == "416" ]]; then
                log "WARNING" "Fichier déjà complet: $(basename "$output_file")"
                return 0
            else
                log "ERROR" "Échec HTTP $http_code pour $(basename "$output_file")"
            fi
        }; then
            return 0
        fi
        
        attempt=$((attempt + 1))
        if [[ $attempt -le $RETRY_ATTEMPTS ]]; then
            local delay=$((attempt * 2))
            log "WARNING" "Nouvelle tentative dans ${delay}s"
            sleep $delay
        fi
    done
    
    log "ERROR" "Échec définitif pour $(basename "$output_file") après $RETRY_ATTEMPTS tentatives"
    return 1
}

# Fonction principale de gestion des téléchargements
download_manager() {
    local url_file="$1"
    local output_dir="${2:-downloads}"
    
    mkdir -p "$output_dir"
    
    # Fonction wrapper pour xargs
    download_wrapper() {
        local url="$1"
        local filename=$(basename "$url" | sed 's/[?].*//' | sed 's/[<>:|*]/_/g')
        local output_file="$output_dir/$filename"
        
        download_with_resume "$url" "$output_file"
    }
    
    export -f download_with_resume
    export -f log
    export -f download_wrapper
    export LOG_FILE
    export TIMEOUT
    export USER_AGENT
    export output_dir
    
    log "INFO" "Démarrage du gestionnaire de téléchargements"
    log "INFO" "Parallélisation: $MAX_PARALLEL, Timeout: ${TIMEOUT}s, Retries: $RETRY_ATTEMPTS"
    
    # Comptage des URLs
    local total_urls=$(wc -l < "$url_file")
    log "INFO" "URLs à traiter: $total_urls"
    
    # Téléchargement parallèle
    local start_time=$(date +%s)
    cat "$url_file" | xargs -n 1 -P "$MAX_PARALLEL" -I {} bash -c 'download_wrapper "$@"' _ {}
    local end_time=$(date +%s)
    
    # Statistiques finales
    local successful=$(find "$output_dir" -type f -size +0 2>/dev/null | wc -l)
    local total_size=$(find "$output_dir" -type f -exec stat -c%s {} \; 2>/dev/null | paste -sd+ | bc 2>/dev/null || echo 0)
    local duration=$((end_time - start_time))
    
    log "INFO" "Téléchargements terminés"
    log "INFO" "Fichiers réussis: $successful/$total_urls"
    log "INFO" "Taille totale: $(numfmt --to=iec-i --suffix=B $total_size 2>/dev/null || echo "${total_size} bytes")"
    log "INFO" "Durée totale: ${duration}s"
    
    if [[ $successful -gt 0 ]]; then
        local avg_speed=$((total_size / duration))
        log "INFO" "Vitesse moyenne: $(numfmt --to=iec-i --suffix=B/s $avg_speed 2>/dev/null || echo "${avg_speed} B/s")"
    fi
}

# Exemple d'utilisation
cat > download_list.txt << EOF
https://releases.ubuntu.com/20.04/ubuntu-20.04.3-desktop-amd64.iso
https://releases.ubuntu.com/20.04/ubuntu-20.04.3-live-server-amd64.iso
https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/debian-11.2.0-amd64-netinst.iso
EOF

# Lancement du gestionnaire
download_manager "download_list.txt" "iso_downloads"

# Nettoyage
rm -f download_list.txt
```

### 2.3 Téléchargement avec limitation de bande passante

```bash
# Téléchargement avec contrôle de bande passante
bandwidth_limited_download() {
    local url="$1"
    local output_file="$2"
    local max_speed_kbps="${3:-1024}"  # 1 Mbps par défaut
    
    # Conversion en bytes par seconde
    local max_speed=$((max_speed_kbps * 1024 / 8))
    
    log "INFO" "Téléchargement limité à ${max_speed_kbps} Kbps ($max_speed B/s)"
    
    curl -s \
        --limit-rate "${max_speed}" \
        -w "Status: %{http_code}, Size: %{size_download} bytes, Speed: %{speed_download} B/s\n" \
        -o "$output_file" \
        "$url"
}

# Téléchargement en arrière-plan avec surveillance
background_download() {
    local url="$1"
    local output_file="$2"
    local log_file="${output_file}.log"
    
    # Lancement en arrière-plan
    curl -s \
        -w "%{time_total};%{size_download};%{speed_download};%{http_code}\n" \
        -o "$output_file" \
        "$url" > "$log_file" &
    
    local pid=$!
    
    echo "Téléchargement lancé en arrière-plan (PID: $pid)"
    echo "Fichier: $output_file"
    echo "Log: $log_file"
    
    # Surveillance
    while kill -0 $pid 2>/dev/null; do
        if [[ -f "$log_file" ]]; then
            local progress=$(tail -1 "$log_file" 2>/dev/null)
            if [[ -n "$progress" ]]; then
                IFS=';' read -r time size speed status <<< "$progress"
                printf "\rProgress: %.1fs, %d bytes, %.0f B/s, Status: %s" "$time" "$size" "$speed" "$status"
            fi
        fi
        sleep 1
    done
    
    echo -e "\nTéléchargement terminé"
    
    # Résultat final
    if [[ -f "$log_file" ]]; then
        local final_stats=$(tail -1 "$log_file")
        IFS=';' read -r time size speed status <<< "$final_stats"
        
        if [[ "$status" =~ ^2 ]]; then
            echo "✓ Succès: ${size} bytes téléchargés en ${time}s"
        else
            echo "✗ Échec: Code HTTP $status"
        fi
    fi
}

# Exemple d'utilisation
background_download "https://example.com/large-file.zip" "download.zip"
```

## Section 3 : Automatisation de workflows web

### 3.1 Scraping web automatisé

```bash
#!/bin/bash
# Outil de scraping web éthique

# Configuration
TARGET_URL="https://example.com/blog"
OUTPUT_DIR="scraped_content"
DELAY=2  # Délai entre les requêtes en secondes
MAX_PAGES=10
USER_AGENT="WebScraper/1.0 (respectful crawler)"

# Fonction de scraping respectueuse
scrape_page() {
    local url="$1"
    local output_file="$2"
    
    log "INFO" "Scraping: $url"
    
    # Requête avec en-têtes respectueux
    local response=$(curl -s \
        -H "User-Agent: $USER_AGENT" \
        -H "Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8" \
        -H "Accept-Language: en-US,en;q=0.5" \
        -H "Accept-Encoding: gzip, deflate" \
        -H "Connection: keep-alive" \
        -H "Upgrade-Insecure-Requests: 1" \
        --compressed \
        --max-time 30 \
        "$url")
    
    if [[ $? -eq 0 ]]; then
        echo "$response" > "$output_file"
        log "SUCCESS" "Contenu sauvegardé: $(basename "$output_file")"
        
        # Extraction des liens pour crawling (simple)
        echo "$response" | grep -o '<a[^>]*href="[^"]*"' | sed 's/.*href="//;s/".*//' | head -5
    else
        log "ERROR" "Échec du scraping: $url"
        return 1
    fi
}

# Fonction principale de crawling
web_crawler() {
    local start_url="$1"
    local max_pages="${2:-10}"
    
    mkdir -p "$OUTPUT_DIR"
    
    local visited_urls=()
    local urls_to_visit=("$start_url")
    local page_count=0
    
    while [[ ${#urls_to_visit[@]} -gt 0 && $page_count -lt $max_pages ]]; do
        local current_url="${urls_to_visit[0]}"
        urls_to_visit=("${urls_to_visit[@]:1}")
        
        # Vérifier si déjà visité
        if [[ " ${visited_urls[*]} " =~ " ${current_url} " ]]; then
            continue
        fi
        
        visited_urls+=("$current_url")
        page_count=$((page_count + 1))
        
        local filename="page_${page_count}.html"
        local output_file="$OUTPUT_DIR/$filename"
        
        if scrape_page "$current_url" "$output_file"; then
            # Extraction des nouveaux liens (version simplifiée)
            local new_links=$(grep -o '<a[^>]*href="[^"]*"' "$output_file" | \
                sed 's/.*href="//;s/".*//' | \
                grep '^https*://' | \
                grep "$TARGET_URL" | \
                head -3)
            
            # Ajout des nouveaux liens à la file d'attente
            while IFS= read -r link; do
                if [[ -n "$link" && ! " ${visited_urls[*]} " =~ " ${link} " && ! " ${urls_to_visit[*]} " =~ " ${link} " ]]; then
                    urls_to_visit+=("$link")
                fi
            done <<< "$new_links"
        fi
        
        # Délai respectueux
        if [[ ${#urls_to_visit[@]} -gt 0 ]]; then
            log "INFO" "Attente de ${DELAY}s avant la prochaine requête..."
            sleep $DELAY
        fi
    done
    
    log "INFO" "Crawling terminé: $page_count pages traitées"
}

# Fonction d'analyse du contenu scrapé
analyze_scraped_content() {
    local content_dir="$1"
    
    echo "=== Analyse du contenu scrapé ==="
    
    local total_files=$(find "$content_dir" -name "*.html" | wc -l)
    local total_size=$(find "$content_dir" -name "*.html" -exec cat {} \; | wc -c)
    
    echo "Fichiers HTML: $total_files"
    echo "Taille totale: $(numfmt --to=iec-i --suffix=B $total_size)"
    
    # Analyse des titres
    echo -e "\nTitres des pages:"
    find "$content_dir" -name "*.html" -exec grep -h '<title>' {} \; | sed 's/<title>//;s/<\/title>//' | head -10
    
    # Analyse des liens
    local total_links=$(find "$content_dir" -name "*.html" -exec grep -h '<a href=' {} \; | wc -l)
    echo "Liens totaux trouvés: $total_links"
    
    # Mots les plus fréquents (simple)
    echo -e "\nMots fréquents:"
    find "$content_dir" -name "*.html" -exec cat {} \; | \
        tr '[:upper:]' '[:lower:]' | \
        grep -o '\b[a-z]\{4,\}\b' | \
        sort | uniq -c | sort -nr | head -10
}

# Exemple d'utilisation
log "INFO" "Démarrage du scraping web"
web_crawler "$TARGET_URL" "$MAX_PAGES"
analyze_scraped_content "$OUTPUT_DIR"

# Nettoyage
# rm -rf "$OUTPUT_DIR"  # Décommenter pour nettoyer
```

### 3.2 Automatisation de formulaires web

```bash
#!/bin/bash
# Automatisation de soumission de formulaires

# Configuration
FORM_URL="https://example.com/contact"
SUCCESS_PATTERN="Thank you|Message sent"
USER_AGENT="FormSubmitter/1.0"

# Fonction de soumission de formulaire
submit_form() {
    local form_data="$1"
    local expected_pattern="$2"
    
    log "INFO" "Soumission du formulaire vers $FORM_URL"
    
    # Récupération de la page du formulaire pour extraire les tokens CSRF
    local form_page=$(curl -s \
        -H "User-Agent: $USER_AGENT" \
        -c cookies.txt -b cookies.txt \
        "$FORM_URL")
    
    # Extraction de token CSRF (exemple générique)
    local csrf_token=$(echo "$form_page" | grep -o 'csrf_token.*value="[^"]*"' | sed 's/.*value="//;s/".*//' | head -1)
    
    if [[ -n "$csrf_token" ]]; then
        form_data="$form_data&csrf_token=$csrf_token"
        log "DEBUG" "Token CSRF ajouté: $csrf_token"
    fi
    
    # Soumission du formulaire
    local response=$(curl -s \
        -H "User-Agent: $USER_AGENT" \
        -H "Referer: $FORM_URL" \
        -c cookies.txt -b cookies.txt \
        -X POST \
        -d "$form_data" \
        -w "HTTP_CODE:%{http_code};RESPONSE_TIME:%{time_total}s\n" \
        "$FORM_URL")
    
    # Analyse de la réponse
    local http_info=$(echo "$response" | tail -1)
    local content=$(echo "$response" | sed '$d')
    
    local http_code=$(echo "$http_info" | cut -d';' -f1 | cut -d':' -f2)
    local response_time=$(echo "$http_info" | cut -d';' -f2 | cut -d':' -f2)
    
    log "INFO" "Code HTTP: $http_code, Temps: $response_time"
    
    if [[ "$http_code" =~ ^2 ]]; then
        if echo "$content" | grep -q "$expected_pattern"; then
            log "SUCCESS" "Formulaire soumis avec succès"
            return 0
        else
            log "WARNING" "Réponse inattendue du serveur"
            log "DEBUG" "Contenu de la réponse: $(echo "$content" | head -5)"
        fi
    else
        log "ERROR" "Échec de soumission (HTTP $http_code)"
        return 1
    fi
}

# Fonction de soumission en masse
bulk_form_submission() {
    local data_file="$1"
    local delay="${2:-1}"
    
    local success_count=0
    local total_count=$(wc -l < "$data_file")
    
    log "INFO" "Soumission en masse: $total_count formulaires"
    
    while IFS='|' read -r name email subject message; do
        # Construction des données du formulaire
        local form_data="name=$name&email=$email&subject=$subject&message=$message"
        
        if submit_form "$form_data" "$SUCCESS_PATTERN"; then
            success_count=$((success_count + 1))
        fi
        
        # Délai entre les soumissions
        if [[ $delay -gt 0 ]]; then
            sleep $delay
        fi
        
        # Affichage de la progression
        echo -ne "\rProgression: $success_count/$total_count"
        
    done < "$data_file"
    
    echo -e "\n"
    log "INFO" "Soumissions terminées: $success_count/$total_count réussies"
}

# Exemple d'utilisation
# Création de données de test
cat > form_data.txt << EOF
John Doe|john@example.com|Question générale|Bonjour, j'aimerais obtenir plus d'informations...
Jane Smith|jane@example.com|Support technique|J'ai un problème avec votre service...
Bob Johnson|bob@example.com|Feedback|Excellent service, continuez comme ça!
EOF

# Soumission des formulaires
bulk_form_submission "form_data.txt" 2

# Nettoyage
rm -f cookies.txt form_data.txt
```

### 3.3 Monitoring d'APIs et alertes

```bash
#!/bin/bash
# Système de monitoring d'APIs avec alertes

# Configuration
API_ENDPOINTS=("https://api1.example.com/health" "https://api2.example.com/status" "https://api3.example.com/ping")
EXPECTED_STATUS="200"
CHECK_INTERVAL=60  # secondes
ALERT_EMAIL="admin@example.com"
LOG_FILE="api_monitor.log"
MAX_FAILURES=3

# Variables globales
declare -A failure_counts
declare -A last_alert_times

# Fonction de logging
log() {
    local level="$1"
    shift
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

# Fonction de vérification d'un endpoint
check_endpoint() {
    local url="$1"
    local start_time=$(date +%s)
    
    local response=$(curl -s \
        -w "HTTP_CODE:%{http_code};RESPONSE_TIME:%{time_total};SIZE:%{size_download}" \
        --max-time 10 \
        "$url")
    
    local end_time=$(date +%s)
    local curl_exit=$?
    
    local http_code=$(echo "$response" | grep -o "HTTP_CODE:[0-9]*" | cut -d: -f2)
    local response_time=$(echo "$response" | grep -o "RESPONSE_TIME:[0-9.]*" | cut -d: -f2)
    local size=$(echo "$response" | grep -o "SIZE:[0-9]*" | cut -d: -f2)
    
    [ -z "$http_code" ] && http_code="000"
    [ -z "$response_time" ] && response_time="0"
    [ -z "$size" ] && size="0"
    
    echo "$url|$http_code|$response_time|$size|$curl_exit|$end_time"
}

# Fonction d'envoi d'alerte
send_alert() {
    local endpoint="$1"
    local issue="$2"
    local details="$3"
    
    local subject="ALERTE API: $endpoint"
    local body="Endpoint: $endpoint
Problème: $issue
Détails: $details
Horodatage: $(date)
Nom d'hôte: $(hostname)"
    
    # Envoi d'email (nécessite mailutils ou similaire)
    if command -v mail >/dev/null 2>&1; then
        echo "$body" | mail -s "$subject" "$ALERT_EMAIL"
        log "INFO" "Alerte email envoyée à $ALERT_EMAIL"
    else
        log "WARNING" "Commande mail non disponible, alerte non envoyée"
    fi
    
    # Mise à jour du timestamp de dernière alerte
    last_alert_times["$endpoint"]=$(date +%s)
}

# Fonction principale de monitoring
monitor_apis() {
    log "INFO" "Démarrage du monitoring d'APIs (${#API_ENDPOINTS[@]} endpoints)"
    
    while true; do
        local current_time=$(date +%s)
        
        for endpoint in "${API_ENDPOINTS[@]}"; do
            log "DEBUG" "Vérification de $endpoint"
            
            local result=$(check_endpoint "$endpoint")
            IFS='|' read -r url http_code response_time size curl_exit timestamp <<< "$result"
            
            if [[ "$http_code" == "$EXPECTED_STATUS" && $curl_exit -eq 0 ]]; then
                # Succès - remise à zéro du compteur d'échecs
                failure_counts["$endpoint"]=0
                log "INFO" "✓ $endpoint - OK (${response_time}s, ${size} bytes)"
            else
                # Échec - incrémentation du compteur
                failure_counts["$endpoint"]=$((failure_counts["$endpoint"] + 1))
                local failure_count=${failure_counts["$endpoint"]}
                
                log "ERROR" "✗ $endpoint - ÉCHEC (HTTP: $http_code, Exit: $curl_exit, Tentatives: $failure_count/$MAX_FAILURES)"
                
                # Vérification du seuil d'alerte
                if [[ $failure_count -ge $MAX_FAILURES ]]; then
                    # Vérification du délai depuis la dernière alerte (au moins 5 minutes)
                    local last_alert=${last_alert_times["$endpoint"]}
                    local time_since_alert=$((current_time - (last_alert:-0)))
                    
                    if [[ $time_since_alert -gt 300 ]]; then
                        send_alert "$endpoint" "Indisponibilité prolongée" "Échecs consécutifs: $failure_count, Dernier code HTTP: $http_code"
                    else
                        log "DEBUG" "Alerte déjà envoyée récemment pour $endpoint"
                    fi
                fi
            fi
        done
        
        log "INFO" "Cycle de vérification terminé, attente de ${CHECK_INTERVAL}s"
        sleep $CHECK_INTERVAL
    done
}

# Fonction de rapport de statut
generate_status_report() {
    echo "=== RAPPORT DE STATUT DES APIs ==="
    echo "Généré le: $(date)"
    echo
    
    for endpoint in "${API_ENDPOINTS[@]}"; do
        local failures=${failure_counts["$endpoint"]}
        local status="OK"
        
        if [[ $failures -gt 0 ]]; then
            status="PROBLÉMATIQUE ($failures échecs)"
        fi
        
        echo "Endpoint: $endpoint"
        echo "Statut: $status"
        echo "Dernière vérification: $(date)"
        echo
    done
}

# Gestion des signaux pour un arrêt propre
trap 'log "INFO" "Arrêt du monitoring demandé par l utilisateur"; generate_status_report; exit 0' INT TERM

# Démarrage du monitoring
monitor_apis
```

## Conclusion : cURL comme orchestrateur web

cURL transcende son rôle d'outil de transfert pour devenir un véritable orchestrateur d'interactions web complexes. Des sessions persistantes aux téléchargements parallèles, en passant par l'automatisation de workflows complets, cURL offre les primitives nécessaires pour construire des solutions d'intégration web robustes et évolutives.

Dans le prochain chapitre, nous explorerons l'utilisation de jq pour le traitement avancé des données JSON, et comment combiner cURL avec d'autres outils pour créer des pipelines de traitement de données web complets.

---

**Exercice pratique :** Créez un système complet de surveillance web qui :
1. Monitor plusieurs sites web et APIs
2. Détecte les changements de contenu
3. Télécharge automatiquement les nouvelles ressources
4. Génère des rapports périodiques
5. Envoie des alertes en cas de problème

**Challenge avancé :** Développez un crawler web respectueux qui :
- Respecte robots.txt
- Limite la fréquence des requêtes
- Évite de surcharger les serveurs
- Gère les sessions et l'authentification
- Stocke les données de manière structurée

**Réflexion :** Comment cURL change-t-il notre approche des interactions web, et quels sont les impacts sur la façon dont nous concevons les systèmes distribués modernes ?

