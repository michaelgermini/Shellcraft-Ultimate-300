# Chapitre 185 - Sécurité des APIs et bonnes pratiques

> "La sécurité des APIs n'est pas une fonctionnalité optionnelle - c'est la fondation même de la confiance dans l'écosystème numérique moderne." - Expert en sécurité des APIs

## Introduction : APIs sécurisées comme impératif moderne

Dans un monde où les APIs sont les interfaces entre tous les systèmes, leur sécurité devient critique. Des vulnérabilités dans les APIs peuvent compromettre des millions d'utilisateurs et des milliards de dollars de valeur économique. Dans ce chapitre, nous explorerons les menaces courantes, les mécanismes de sécurité avancés, et les meilleures pratiques pour construire et consommer des APIs de manière sécurisée.

De l'authentification robuste aux patterns de sécurité avancés, nous couvrirons l'ensemble du spectre de la sécurité API.

## Section 1 : Menaces et vulnérabilités courantes des APIs

### 1.1 OWASP API Security Top 10

```bash
#!/bin/bash
# Outil d'audit de sécurité des APIs basé sur OWASP Top 10

# Configuration
TARGET_API="${TARGET_API:-https://api.example.com}"
API_TOKEN="${API_TOKEN:-}"
LOG_FILE="api_security_audit.log"

# Fonction de logging
log() {
    local level="$1"
    shift
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

# Fonction d'appel API sécurisé
secure_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    local expected_status="${4:-200}"
    
    local headers=(-H "User-Agent: APISecurityScanner/1.0")
    
    if [[ -n "$API_TOKEN" ]]; then
        headers+=(-H "Authorization: Bearer $API_TOKEN")
    fi
    
    if [[ -n "$data" ]]; then
        headers+=(-H "Content-Type: application/json")
    fi
    
    local response=$(curl -s -w "\nHTTP_STATUS:%{http_code}\nRESPONSE_TIME:%{time_total}\n" \
        "${headers[@]}" \
        -X "$method" \
        ${data:+-d "$data"} \
        "$TARGET_API$endpoint")
    
    local http_status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
    local response_time=$(echo "$response" | grep "RESPONSE_TIME:" | cut -d: -f2)
    local content=$(echo "$response" | sed '/HTTP_STATUS:/d; /RESPONSE_TIME:/d')
    
    echo "{\"status\": $http_status, \"response_time\": $response_time, \"content\": $content}"
}

# Test 1: Injection d'objets (API1:2023)
test_broken_object_level_authorization() {
    log "INFO" "=== Test API1:2023 - Broken Object Level Authorization ==="
    
    local vulnerabilities=()
    
    # Tentative d'accès à des ressources d'autres utilisateurs
    for user_id in {1..5}; do
        local response=$(secure_api_call "GET" "/users/$user_id/profile")
        local status=$(echo "$response" | jq -r '.status')
        
        if [[ "$status" == "200" ]]; then
            log "WARNING" "Accès non autorisé possible à l'utilisateur $user_id"
            vulnerabilities+=("BOLA: Accès à /users/$user_id/profile")
        fi
        
        # Test avec ID négatif
        response=$(secure_api_call "GET" "/users/-$user_id/profile")
        status=$(echo "$response" | jq -r '.status')
        
        if [[ "$status" == "200" ]]; then
            log "CRITICAL" "Accès avec ID négatif réussi: /users/-$user_id/profile"
            vulnerabilities+=("BOLA: ID négatif accepté")
        fi
    done
    
    echo "${#vulnerabilities[@]} vulnérabilités BOLA détectées"
    printf '%s\n' "${vulnerabilities[@]}"
}

# Test 2: Authentification défaillante (API2:2023)
test_broken_authentication() {
    log "INFO" "=== Test API2:2023 - Broken Authentication ==="
    
    local vulnerabilities=()
    
    # Test de tokens JWT expirés
    local expired_token="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyMzkwMjJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
    
    local response=$(curl -s -H "Authorization: Bearer $expired_token" -w "HTTP_STATUS:%{http_code}" "$TARGET_API/secure")
    local status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
    
    if [[ "$status" == "200" ]]; then
        log "CRITICAL" "Token JWT expiré accepté"
        vulnerabilities+=("Broken Auth: Token expiré accepté")
    fi
    
    # Test de tokens malformés
    local malformed_tokens=(
        "not-a-jwt"
        "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9"
        "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIn0"
    )
    
    for token in "${malformed_tokens[@]}"; do
        response=$(curl -s -H "Authorization: Bearer $token" -w "HTTP_STATUS:%{http_code}" "$TARGET_API/secure")
        status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
        
        if [[ "$status" == "200" ]]; then
            log "CRITICAL" "Token malformé accepté: $token"
            vulnerabilities+=("Broken Auth: Token malformé accepté")
        fi
    done
    
    echo "${#vulnerabilities[@]} vulnérabilités d'authentification détectées"
    printf '%s\n' "${vulnerabilities[@]}"
}

# Test 3: Exposition excessive de données (API3:2023)
test_excessive_data_exposure() {
    log "INFO" "=== Test API3:2023 - Excessive Data Exposure ==="
    
    local vulnerabilities=()
    
    # Test des endpoints retournant potentiellement trop de données
    local endpoints=("/users" "/orders" "/transactions" "/logs")
    
    for endpoint in "${endpoints[@]}"; do
        local response=$(secure_api_call "GET" "$endpoint?limit=1000")
        local content=$(echo "$response" | jq -r '.content')
        
        # Vérification de la présence de données sensibles
        if echo "$content" | jq -e '.[] | select(.password or .ssn or .credit_card)' > /dev/null 2>&1; then
            log "CRITICAL" "Données sensibles exposées dans $endpoint"
            vulnerabilities+=("Excessive Data: Données sensibles dans $endpoint")
        fi
        
        # Vérification de la taille des réponses
        local size=$(echo "$content" | wc -c)
        if [[ $size -gt 1000000 ]]; then  # 1MB
            log "WARNING" "Réponse excessive dans $endpoint: $size bytes"
            vulnerabilities+=("Excessive Data: Réponse volumineuse dans $endpoint")
        fi
    done
    
    echo "${#vulnerabilities[@]} vulnérabilités de fuite de données détectées"
    printf '%s\n' "${vulnerabilities[@]}"
}

# Test 4: Manque de limitation des ressources (API4:2023)
test_unrestricted_resource_consumption() {
    log "INFO" "=== Test API4:2023 - Unrestricted Resource Consumption ==="
    
    local vulnerabilities=()
    
    # Test de déni de service par volumétrie
    local large_payload=""
    for i in {1..1000}; do
        large_payload+='{"data": "test"}'
    done
    
    local start_time=$(date +%s)
    local response=$(secure_api_call "POST" "/bulk" "[$large_payload]" "201")
    local end_time=$(date +%s)
    local duration=$((end_time - start_time))
    
    if [[ $duration -gt 30 ]]; then
        log "WARNING" "Traitement lent de gros payload: ${duration}s"
        vulnerabilities+=("Resource Consumption: Traitement lent de gros payload")
    fi
    
    # Test de limitation de débit
    log "INFO" "Test de limitation de débit..."
    local success_count=0
    local total_requests=50
    
    for i in {1..50}; do
        response=$(secure_api_call "GET" "/rate-limited")
        local status=$(echo "$response" | jq -r '.status')
        
        if [[ "$status" == "200" ]]; then
            success_count=$((success_count + 1))
        elif [[ "$status" == "429" ]]; then
            log "INFO" "Rate limiting détecté après $i requêtes"
            break
        fi
        
        sleep 0.1  # Petit délai pour éviter la surcharge
    done
    
    if [[ $success_count -eq $total_requests ]]; then
        log "WARNING" "Aucune limitation de débit détectée"
        vulnerabilities+=("Resource Consumption: Pas de rate limiting")
    fi
    
    echo "${#vulnerabilities[@]} vulnérabilités de consommation de ressources détectées"
    printf '%s\n' "${vulnerabilities[@]}"
}

# Test 5: Autorisation défaillante (API5:2023)
test_broken_function_level_authorization() {
    log "INFO" "=== Test API5:2023 - Broken Function Level Authorization ==="
    
    local vulnerabilities=()
    
    # Test d'accès à des fonctions administrateur
    local admin_endpoints=("/admin/users" "/admin/config" "/admin/logs" "/superuser/settings")
    
    for endpoint in "${admin_endpoints[@]}"; do
        local response=$(secure_api_call "GET" "$endpoint")
        local status=$(echo "$response" | jq -r '.status')
        
        if [[ "$status" == "200" ]]; then
            log "CRITICAL" "Accès non autorisé à une fonction admin: $endpoint"
            vulnerabilities+=("Broken Function Auth: Accès à $endpoint")
        elif [[ "$status" == "403" ]]; then
            log "INFO" "Accès correctement refusé pour $endpoint"
        fi
    done
    
    # Test de privilege escalation via paramètres
    local response=$(secure_api_call "GET" "/users/123?role=admin")
    local status=$(echo "$response" | jq -r '.status')
    
    if [[ "$status" == "200" ]]; then
        local content=$(echo "$response" | jq -r '.content')
        if echo "$content" | jq -e '.role == "admin"' > /dev/null 2>&1; then
            log "CRITICAL" "Privilege escalation via paramètre réussi"
            vulnerabilities+=("Broken Function Auth: Privilege escalation via paramètre")
        fi
    fi
    
    echo "${#vulnerabilities[@]} vulnérabilités d'autorisation détectées"
    printf '%s\n' "${vulnerabilities[@]}"
}

# Test 6: Mass Assignment (API6:2023)
test_mass_assignment() {
    log "INFO" "=== Test API6:2023 - Mass Assignment ==="
    
    local vulnerabilities=()
    
    # Test d'injection de propriétés non autorisées
    local malicious_payload='{
        "name": "Test User",
        "email": "test@example.com",
        "role": "user",
        "is_admin": true,
        "salary": 100000,
        "internal_notes": "This should not be settable"
    }'
    
    local response=$(secure_api_call "POST" "/users" "$malicious_payload" "201")
    local status=$(echo "$response" | jq -r '.status')
    
    if [[ "$status" == "201" ]]; then
        local content=$(echo "$response" | jq -r '.content')
        
        # Vérification si les propriétés sensibles ont été acceptées
        if echo "$content" | jq -e '.is_admin == true or .salary or .internal_notes' > /dev/null 2>&1; then
            log "CRITICAL" "Mass assignment réussi - propriétés sensibles acceptées"
            vulnerabilities+=("Mass Assignment: Propriétés sensibles assignées")
        fi
    fi
    
    echo "${#vulnerabilities[@]} vulnérabilités de mass assignment détectées"
    printf '%s\n' "${vulnerabilities[@]}"
}

# Fonction principale d'audit de sécurité
run_api_security_audit() {
    log "INFO" "=== DÉMARRAGE DE L'AUDIT DE SÉCURITÉ API ==="
    log "INFO" "API cible: $TARGET_API"
    
    local total_vulnerabilities=0
    
    # Exécution des tests OWASP
    test_broken_object_level_authorization
    total_vulnerabilities+=$?
    
    test_broken_authentication
    total_vulnerabilities+=$?
    
    test_excessive_data_exposure
    total_vulnerabilities+=$?
    
    test_unrestricted_resource_consumption
    total_vulnerabilities+=$?
    
    test_broken_function_level_authorization
    total_vulnerabilities+=$?
    
    test_mass_assignment
    total_vulnerabilities+=$?
    
    log "INFO" "=== AUDIT TERMINÉ ==="
    log "INFO" "Total des vulnérabilités détectées: $total_vulnerabilities"
    
    # Rapport de synthèse
    echo "=== RAPPORT D'AUDIT DE SÉCURITÉ API ===" > "api_security_report_$(date +%Y%m%d_%H%M%S).txt"
    echo "API auditée: $TARGET_API" >> "api_security_report_$(date +%Y%m%d_%H%M%S).txt"
    echo "Date: $(date)" >> "api_security_report_$(date +%Y%m%d_%H%M%S).txt"
    echo "Vulnérabilités détectées: $total_vulnerabilities" >> "api_security_report_$(date +%Y%m%d_%H%M%S).txt"
    echo "" >> "api_security_report_$(date +%Y%m%d_%H%M%S).txt"
    echo "Consultez $LOG_FILE pour le détail complet." >> "api_security_report_$(date +%Y%m%d_%H%M%S).txt"
}

# Exécution de l'audit
run_api_security_audit
```

### 1.2 Injection et attaques courantes

```bash
#!/bin/bash
# Tests d'injection et attaques sur APIs

# Configuration
TARGET_API="${TARGET_API:-https://api.example.com}"

# Test d'injection SQL
test_sql_injection() {
    log "INFO" "=== Test d'injection SQL ==="
    
    local sql_payloads=(
        "' OR '1'='1"
        "'; DROP TABLE users; --"
        "' UNION SELECT username, password FROM users; --"
        "admin' --"
        "1' OR '1' = '1"
    )
    
    for payload in "${sql_payloads[@]}"; do
        log "DEBUG" "Test SQLi: $payload"
        
        local response=$(curl -s \
            -H "Content-Type: application/x-www-form-urlencoded" \
            -d "username=$payload&password=test" \
            -w "HTTP_STATUS:%{http_code}" \
            "$TARGET_API/login")
        
        local status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
        
        if [[ "$status" == "200" ]]; then
            log "CRITICAL" "Injection SQL potentiellement réussie avec: $payload"
        fi
    done
}

# Test d'injection NoSQL
test_nosql_injection() {
    log "INFO" "=== Test d'injection NoSQL ==="
    
    local nosql_payloads=(
        '{"$gt": ""}'
        '{"$where": "this.password.length > 0"}'
        '{"username": {"$ne": null}, "password": {"$ne": null}}'
    )
    
    for payload in "${nosql_payloads[@]}"; do
        log "DEBUG" "Test NoSQLi: $payload"
        
        local response=$(curl -s \
            -H "Content-Type: application/json" \
            -d "{\"username\": $payload, \"password\": \"test\"}" \
            -w "HTTP_STATUS:%{http_code}" \
            "$TARGET_API/login")
        
        local status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
        
        if [[ "$status" == "200" ]]; then
            log "CRITICAL" "Injection NoSQL potentiellement réussie avec: $payload"
        fi
    done
}

# Test d'injection de commandes (Command Injection)
test_command_injection() {
    log "INFO" "=== Test d'injection de commandes ==="
    
    local cmd_payloads=(
        "; cat /etc/passwd"
        "| cat /etc/passwd"
        "$(cat /etc/passwd)"
        "`cat /etc/passwd`"
        "; rm -rf /"
    )
    
    for payload in "${cmd_payloads[@]}"; do
        log "DEBUG" "Test Command Injection: $payload"
        
        local response=$(curl -s \
            -d "command=echo hello$payload" \
            -w "HTTP_STATUS:%{http_code}" \
            "$TARGET_API/execute")
        
        local status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
        local content=$(echo "$response" | sed '/HTTP_STATUS:/d')
        
        # Vérification de signes d'injection réussie
        if echo "$content" | grep -q "root:" || [[ "$status" == "500" ]]; then
            log "CRITICAL" "Injection de commande potentiellement réussie avec: $payload"
        fi
    done
}

# Test de traversal de répertoire (Path Traversal)
test_path_traversal() {
    log "INFO" "=== Test de traversal de répertoire ==="
    
    local traversal_payloads=(
        "../../../etc/passwd"
        "..\\..\\..\\windows\\system32\\config\\sam"
        "....//....//....//etc/passwd"
        "%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd"
        ".../...//.../...//.../...//etc/passwd"
    )
    
    for payload in "${traversal_payloads[@]}"; do
        log "DEBUG" "Test Path Traversal: $payload"
        
        local response=$(curl -s \
            -w "HTTP_STATUS:%{http_code}" \
            "$TARGET_API/files?path=$payload")
        
        local status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
        local content=$(echo "$response" | sed '/HTTP_STATUS:/d')
        
        # Vérification d'accès à des fichiers sensibles
        if echo "$content" | grep -q "root:" || [[ "$status" == "200" && $(echo "$content" | wc -c) -gt 100 ]]; then
            log "CRITICAL" "Path traversal potentiellement réussi avec: $payload"
        fi
    done
}

# Test d'injection XXE (XML External Entity)
test_xxe_injection() {
    log "INFO" "=== Test d'injection XXE ==="
    
    local xxe_payload='<?xml version="1.0"?>
<!DOCTYPE foo [
<!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<user>
    <name>&xxe;</name>
</user>'
    
    log "DEBUG" "Test XXE injection"
    
    local response=$(curl -s \
        -H "Content-Type: application/xml" \
        -d "$xxe_payload" \
        -w "HTTP_STATUS:%{http_code}" \
        "$TARGET_API/xml-parser")
    
    local status=$(echo "$response" | grep "HTTP_STATUS:" | cut -d: -f2)
    local content=$(echo "$response" | sed '/HTTP_STATUS:/d')
    
    if echo "$content" | grep -q "root:"; then
        log "CRITICAL" "Injection XXE réussie - contenu de /etc/passwd détecté"
    fi
}

# Fonction principale de test des injections
run_injection_tests() {
    log "INFO" "=== DÉMARRAGE DES TESTS D'INJECTION ==="
    
    test_sql_injection
    test_nosql_injection
    test_command_injection
    test_path_traversal
    test_xxe_injection
    
    log "INFO" "=== TESTS D'INJECTION TERMINÉS ==="
}

# Exécution des tests
run_injection_tests
```

## Section 2 : Mécanismes de sécurité avancés

### 2.1 OAuth 2.0 et OpenID Connect

```bash
#!/bin/bash
# Implémentation client OAuth 2.0 sécurisé

# Configuration OAuth
OAUTH_AUTH_URL="https://auth.example.com/oauth/authorize"
OAUTH_TOKEN_URL="https://auth.example.com/oauth/token"
CLIENT_ID="secure_client_id"
CLIENT_SECRET="secure_client_secret"
REDIRECT_URI="http://localhost:8080/callback"
SCOPE="read write"

# Fonction de génération de state pour prévention CSRF
generate_state() {
    openssl rand -hex 16
}

# Fonction d'encodage URL
url_encode() {
    local string="$1"
    printf '%s' "$string" | jq -s -R -r @uri
}

# Flux d'autorisation OAuth 2.0 (Authorization Code)
oauth_authorization_code_flow() {
    log "INFO" "=== Flux Authorization Code OAuth 2.0 ==="
    
    local state=$(generate_state)
    
    # Construction de l'URL d'autorisation
    local auth_url="$OAUTH_AUTH_URL?response_type=code&client_id=$CLIENT_ID&redirect_uri=$(url_encode "$REDIRECT_URI")&scope=$(url_encode "$SCOPE")&state=$state"
    
    log "INFO" "Ouvrez cette URL dans votre navigateur:"
    log "INFO" "$auth_url"
    
    # Simulation de réception du code d'autorisation
    echo -n "Entrez le code d'autorisation reçu: "
    read -r auth_code
    
    echo -n "Entrez la valeur state reçue: "
    read -r received_state
    
    # Vérification du state pour prévention CSRF
    if [[ "$received_state" != "$state" ]]; then
        log "ERROR" "Échec de vérification state - possible attaque CSRF"
        return 1
    fi
    
    # Échange du code contre un token
    local token_response=$(curl -s \
        -X POST \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -d "grant_type=authorization_code&code=$auth_code&redirect_uri=$(url_encode "$REDIRECT_URI")&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET" \
        "$OAUTH_TOKEN_URL")
    
    # Validation et stockage du token
    if echo "$token_response" | jq -e '.access_token' > /dev/null 2>&1; then
        local access_token=$(echo "$token_response" | jq -r '.access_token')
        local token_type=$(echo "$token_response" | jq -r '.token_type')
        local expires_in=$(echo "$token_response" | jq -r '.expires_in')
        
        log "SUCCESS" "Token OAuth obtenu: $token_type"
        
        # Stockage sécurisé du token
        echo "{\"access_token\": \"$access_token\", \"token_type\": \"$token_type\", \"expires_at\": $(($(date +%s) + expires_in))}" > ~/.oauth_token
        
        return 0
    else
        log "ERROR" "Échec d'obtention du token OAuth"
        return 1
    fi
}

# Flux Client Credentials (pour applications serveur)
oauth_client_credentials_flow() {
    log "INFO" "=== Flux Client Credentials OAuth 2.0 ==="
    
    local token_response=$(curl -s \
        -X POST \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -u "$CLIENT_ID:$CLIENT_SECRET" \
        -d "grant_type=client_credentials&scope=$(url_encode "$SCOPE")" \
        "$OAUTH_TOKEN_URL")
    
    if echo "$token_response" | jq -e '.access_token' > /dev/null 2>&1; then
        local access_token=$(echo "$token_response" | jq -r '.access_token')
        local expires_in=$(echo "$token_response" | jq -r '.expires_in // 3600')
        
        # Stockage du token
        echo "{\"access_token\": \"$access_token\", \"expires_at\": $(($(date +%s) + expires_in))}" > ~/.oauth_token
        
        log "SUCCESS" "Token Client Credentials obtenu"
        return 0
    else
        log "ERROR" "Échec d'obtention du token Client Credentials"
        return 1
    fi
}

# Fonction de récupération du token valide
get_valid_token() {
    local token_file="${1:-$HOME/.oauth_token}"
    
    if [[ ! -f "$token_file" ]]; then
        log "WARNING" "Aucun token stocké trouvé"
        return 1
    fi
    
    local token_data=$(cat "$token_file")
    local expires_at=$(echo "$token_data" | jq -r '.expires_at')
    local current_time=$(date +%s)
    
    # Vérification d'expiration avec marge de 5 minutes
    if [[ $current_time -gt $((expires_at - 300)) ]]; then
        log "WARNING" "Token expiré ou expirant bientôt"
        return 1
    fi
    
    echo "$token_data" | jq -r '.access_token'
}

# Fonction d'appel API avec token OAuth
oauth_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local token=$(get_valid_token)
    
    if [[ -z "$token" ]]; then
        log "ERROR" "Aucun token valide disponible"
        return 1
    fi
    
    local token_data=$(cat ~/.oauth_token)
    local token_type=$(echo "$token_data" | jq -r '.token_type // "Bearer"')
    
    curl -s \
        -X "$method" \
        -H "Authorization: $token_type $token" \
        ${data:+-H "Content-Type: application/json" -d "$data"} \
        "$endpoint"
}

# Implémentation OpenID Connect
openid_connect_flow() {
    log "INFO" "=== Flux OpenID Connect ==="
    
    # Découverte des endpoints OIDC
    local discovery_url="https://auth.example.com/.well-known/openid-configuration"
    local oidc_config=$(curl -s "$discovery_url")
    
    local auth_endpoint=$(echo "$oidc_config" | jq -r '.authorization_endpoint')
    local token_endpoint=$(echo "$oidc_config" | jq -r '.token_endpoint')
    local userinfo_endpoint=$(echo "$oidc_config" | jq -r '.userinfo_endpoint')
    
    log "INFO" "Endpoints OIDC découverts"
    
    # Flux d'autorisation avec nonce pour prévention replay
    local nonce=$(openssl rand -hex 16)
    local state=$(generate_state)
    
    local auth_url="$auth_endpoint?response_type=code&client_id=$CLIENT_ID&redirect_uri=$(url_encode "$REDIRECT_URI")&scope=$(url_encode "openid profile email")&state=$state&nonce=$nonce"
    
    log "INFO" "URL d'autorisation OIDC:"
    log "INFO" "$auth_url"
    
    # Simulation de réception du code
    echo -n "Entrez le code d'autorisation OIDC: "
    read -r auth_code
    
    # Échange du code contre des tokens
    local token_response=$(curl -s \
        -X POST \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -d "grant_type=authorization_code&code=$auth_code&redirect_uri=$(url_encode "$REDIRECT_URI")&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&nonce=$nonce" \
        "$token_endpoint")
    
    if echo "$token_response" | jq -e '.id_token' > /dev/null 2>&1; then
        local id_token=$(echo "$token_response" | jq -r '.id_token')
        local access_token=$(echo "$token_response" | jq -r '.access_token')
        
        # Validation du JWT ID Token (simplifié)
        local header=$(echo "$id_token" | cut -d. -f1 | base64 -d 2>/dev/null)
        local payload=$(echo "$id_token" | cut -d. -f2 | base64 -d 2>/dev/null)
        
        log "SUCCESS" "ID Token OIDC obtenu et décodé"
        
        # Récupération des informations utilisateur
        local userinfo=$(curl -s -H "Authorization: Bearer $access_token" "$userinfo_endpoint")
        
        echo "Informations utilisateur OIDC:"
        echo "$userinfo" | jq '.'
        
        return 0
    else
        log "ERROR" "Échec du flux OIDC"
        return 1
    fi
}

# Menu principal
case "${1:-}" in
    "auth-code")
        oauth_authorization_code_flow
        ;;
    "client-credentials")
        oauth_client_credentials_flow
        ;;
    "oidc")
        openid_connect_flow
        ;;
    "call")
        oauth_api_call "GET" "${2:-https://api.example.com/user}"
        ;;
    *)
        echo "Usage: $0 {auth-code|client-credentials|oidc|call [url]}"
        ;;
esac
```

### 2.2 JSON Web Tokens (JWT) et signatures

```bash
#!/bin/bash
# Gestion sécurisée des JWT

# Fonction de décodage JWT
decode_jwt() {
    local jwt="$1"
    
    # Décodage du header
    local header=$(echo "$jwt" | cut -d. -f1 | base64 -d 2>/dev/null | jq '.')
    
    # Décodage du payload
    local payload=$(echo "$jwt" | cut -d. -f2 | base64 -d 2>/dev/null | jq '.')
    
    # Signature (non décodée)
    local signature=$(echo "$jwt" | cut -d. -f3)
    
    echo "Header: $header"
    echo "Payload: $payload"
    echo "Signature: $signature"
}

# Fonction de validation JWT
validate_jwt() {
    local jwt="$1"
    local public_key="$2"
    
    # Extraction des composants
    local header_payload=$(echo "$jwt" | cut -d. -f1-2)
    local signature=$(echo "$jwt" | cut -d. -f3)
    
    # Vérification de la signature (nécessite openssl ou jwt lib)
    local expected_signature=$(echo -n "$header_payload" | openssl dgst -sha256 -sign "$public_key" | base64 | tr -d '\n=' | tr '/+' '_-')
    
    if [[ "$signature" == "$expected_signature" ]]; then
        echo "✓ Signature JWT valide"
        return 0
    else
        echo "✗ Signature JWT invalide"
        return 1
    fi
}

# Fonction de création JWT (pour test)
create_jwt() {
    local payload="$1"
    local private_key="$2"
    local algorithm="${3:-RS256}"
    
    # Header
    local header='{"alg":"'$algorithm'","typ":"JWT"}'
    local header_encoded=$(echo -n "$header" | base64 | tr -d '\n=' | tr '/+' '_-')
    
    # Payload
    local payload_encoded=$(echo -n "$payload" | base64 | tr -d '\n=' | tr '/+' '_-')
    
    # Signature
    local data="$header_encoded.$payload_encoded"
    local signature=$(echo -n "$data" | openssl dgst -"$algorithm" -sign "$private_key" | base64 | tr -d '\n=' | tr '/+' '_-')
    
    echo "$data.$signature"
}

# Fonction d'inspection de tokens JWT
inspect_jwt_token() {
    local token="$1"
    
    echo "=== INSPECTION DU TOKEN JWT ==="
    
    # Décodage
    local header=$(echo "$token" | cut -d. -f1)
    local payload=$(echo "$token" | cut -d. -f2)
    
    # Décodage base64url
    decode_base64url() {
        local input="$1"
        local padding=$(( (4 - ${#input} % 4) % 4 ))
        local padded="$input"
        for ((i=0; i<padding; i++)); do
            padded+="="
        done
        
        echo "$padded" | tr '_-' '/+' | base64 -d 2>/dev/null | jq '.'
    }
    
    echo "Header:"
    decode_base64url "$header"
    
    echo -e "\nPayload:"
    decode_base64url "$payload"
    
    # Analyse du payload
    local payload_json=$(decode_base64url "$payload")
    
    echo -e "\n=== ANALYSE DU PAYLOAD ==="
    
    # Expiration
    local exp=$(echo "$payload_json" | jq -r '.exp // empty')
    if [[ -n "$exp" ]]; then
        local exp_date=$(date -d "@$exp" 2>/dev/null || echo "Invalid timestamp")
        local current_time=$(date +%s)
        
        if [[ $current_time -gt $exp ]]; then
            echo "⚠️  Token expiré: $exp_date"
        else
            local remaining=$((exp - current_time))
            echo "✓ Token valide jusqu'au: $exp_date (encore $remaining secondes)"
        fi
    fi
    
    # Issuer
    local iss=$(echo "$payload_json" | jq -r '.iss // empty')
    [[ -n "$iss" ]] && echo "Issuer: $iss"
    
    # Subject
    local sub=$(echo "$payload_json" | jq -r '.sub // empty')
    [[ -n "$sub" ]] && echo "Subject: $sub"
    
    # Audience
    local aud=$(echo "$payload_json" | jq -r '.aud // empty')
    [[ -n "$aud" ]] && echo "Audience: $aud"
    
    # Issued at
    local iat=$(echo "$payload_json" | jq -r '.iat // empty')
    if [[ -n "$iat" ]]; then
        local iat_date=$(date -d "@$iat" 2>/dev/null || echo "Invalid timestamp")
        echo "Issued at: $iat_date"
    fi
}

# Test des fonctions JWT
test_jwt_functions() {
    echo "=== TEST DES FONCTIONS JWT ==="
    
    # Création d'un payload de test
    local test_payload='{
        "iss": "test-issuer",
        "sub": "test-user",
        "aud": "test-audience",
        "exp": '$(( $(date +%s) + 3600 ))',
        "iat": '$(date +%s)',
        "name": "Test User",
        "email": "test@example.com"
    }'
    
    echo "Payload de test:"
    echo "$test_payload" | jq '.'
    
    # Note: Pour une vraie implémentation, il faudrait des clés RSA
    # Ici nous simulons juste l'inspection
    
    # Token JWT de test (expiré)
    local test_token="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE1MTYyMzkwMjJ9.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"
    
    echo -e "\nInspection du token de test:"
    inspect_jwt_token "$test_token"
}

# Exécution des tests
test_jwt_functions
```

## Section 3 : Bonnes pratiques de sécurité API

### 3.1 Défense en profondeur

```bash
#!/bin/bash
# Framework de bonnes pratiques de sécurité API

# Configuration
API_BASE_URL="${API_BASE_URL:-https://api.example.com}"
SECURITY_LOG="api_security.log"
MAX_REQUESTS_PER_MINUTE=60
RATE_LIMIT_WINDOW=60

# Fonction de logging sécurisé
secure_log() {
    local level="$1"
    local message="$2"
    local sensitive_data="$3"
    
    # Sanitisation des données sensibles
    local sanitized_message="$message"
    if [[ -n "$sensitive_data" ]]; then
        sanitized_message=$(echo "$message" | sed 's/'"$sensitive_data"'/[REDACTED]/g')
    fi
    
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $sanitized_message" >> "$SECURITY_LOG"
}

# Fonction de validation d'entrée
validate_input() {
    local input="$1"
    local type="$2"
    local max_length="${3:-1000}"
    
    # Vérification de longueur
    if [[ ${#input} -gt $max_length ]]; then
        secure_log "WARNING" "Input too long: ${#input} characters (max: $max_length)"
        return 1
    fi
    
    case "$type" in
        "email")
            if ! echo "$input" | grep -qE '^[^@]+@[^@]+\.[^@]+$'; then
                secure_log "WARNING" "Invalid email format: $input"
                return 1
            fi
            ;;
        "url")
            if ! echo "$input" | grep -qE '^https?://'; then
                secure_log "WARNING" "Invalid URL format: $input"
                return 1
            fi
            ;;
        "alphanumeric")
            if ! echo "$input" | grep -qE '^[a-zA-Z0-9_]+$'; then
                secure_log "WARNING" "Invalid alphanumeric input: $input"
                return 1
            fi
            ;;
    esac
    
    return 0
}

# Fonction de limitation de débit
rate_limit_check() {
    local client_ip="$1"
    local endpoint="$2"
    
    local rate_limit_key="${client_ip}_${endpoint}"
    local current_time=$(date +%s)
    local window_start=$((current_time - RATE_LIMIT_WINDOW))
    
    # Comptage des requêtes récentes (simulation avec fichier)
    local request_log="/tmp/rate_limit_${rate_limit_key}"
    
    # Nettoyage des anciennes entrées
    if [[ -f "$request_log" ]]; then
        awk -v window_start="$window_start" '$1 >= window_start {print}' "$request_log" > "${request_log}.tmp"
        mv "${request_log}.tmp" "$request_log"
    fi
    
    # Comptage des requêtes dans la fenêtre
    local request_count=$(wc -l < "$request_log" 2>/dev/null || echo 0)
    
    if [[ $request_count -ge $MAX_REQUESTS_PER_MINUTE ]]; then
        secure_log "WARNING" "Rate limit exceeded for $client_ip on $endpoint ($request_count requests)"
        return 1
    fi
    
    # Enregistrement de la requête
    echo "$current_time" >> "$request_log"
    
    return 0
}

# Fonction d'appel API sécurisé
secure_api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    local client_ip="${4:-127.0.0.1}"
    
    # Vérification de limitation de débit
    if ! rate_limit_check "$client_ip" "$endpoint"; then
        echo '{"error": "Rate limit exceeded", "status": 429}' | jq '.'
        return 429
    fi
    
    # Validation des données d'entrée
    if [[ -n "$data" ]]; then
        # Validation JSON
        if ! echo "$data" | jq empty 2>/dev/null; then
            secure_log "WARNING" "Invalid JSON data received from $client_ip"
            echo '{"error": "Invalid JSON", "status": 400}' | jq '.'
            return 400
        fi
        
        # Validation de taille
        if [[ ${#data} -gt 10000 ]]; then
            secure_log "WARNING" "Request too large from $client_ip: ${#data} characters"
            echo '{"error": "Request too large", "status": 413}' | jq '.'
            return 413
        fi
    fi
    
    # Log de la requête (sans données sensibles)
    secure_log "INFO" "$method $endpoint from $client_ip"
    
    # Simulation de traitement
    local response='{"status": "success", "data": "processed"}'
    
    # Log de la réponse
    secure_log "INFO" "Response: $(echo "$response" | jq -r '.status')"
    
    echo "$response" | jq '.'
    return 200
}

# Fonction de chiffrement des données sensibles
encrypt_sensitive_data() {
    local data="$1"
    local key="${2:-$(openssl rand -hex 32)}"
    
    echo "$data" | openssl enc -aes-256-cbc -salt -pass "pass:$key" | base64 -w 0
}

decrypt_sensitive_data() {
    local encrypted_data="$1"
    local key="$2"
    
    echo "$encrypted_data" | base64 -d | openssl enc -d -aes-256-cbc -pass "pass:$key"
}

# Fonction de rotation des secrets
rotate_api_keys() {
    local old_key="$1"
    local new_key="${2:-$(openssl rand -hex 32)}"
    
    secure_log "INFO" "Rotation des clés API - Ancienne clé: ${old_key:0:8}..."
    
    # Mise à jour de la configuration
    sed -i "s/$old_key/$new_key/" /etc/api/config
    
    # Notification (simulée)
    secure_log "INFO" "Nouvelle clé API générée: ${new_key:0:8}..."
    
    echo "Clé API rotated successfully"
}

# Fonction d'audit de sécurité
security_audit() {
    local audit_date=$(date +%Y%m%d_%H%M%S)
    local audit_file="security_audit_$audit_date.txt"
    
    {
        echo "=== AUDIT DE SÉCURITÉ API ==="
        echo "Date: $(date)"
        echo
        
        echo "=== LOGS DE SÉCURITÉ ==="
        if [[ -f "$SECURITY_LOG" ]]; then
            tail -50 "$SECURITY_LOG"
        else
            echo "Aucun log de sécurité trouvé"
        fi
        echo
        
        echo "=== STATISTIQUES DE LIMITATION DE DÉBIT ==="
        find /tmp -name "rate_limit_*" -type f 2>/dev/null | while read -r file; do
            local count=$(wc -l < "$file")
            local client=$(basename "$file" | sed 's/rate_limit_//')
            echo "$client: $count requêtes"
        done
        echo
        
        echo "=== RECOMMANDATIONS ==="
        echo "1. Examiner régulièrement les logs de sécurité"
        echo "2. Ajuster les limites de débit selon l'usage"
        echo "3. Implémenter une rotation régulière des clés API"
        echo "4. Mettre à jour régulièrement les certificats SSL"
        echo "5. Activer la journalisation détaillée en production"
        
    } > "$audit_file"
    
    secure_log "INFO" "Audit de sécurité généré: $audit_file"
    echo "Audit completed: $audit_file"
}

# Fonction de réponse d'urgence
emergency_response() {
    local threat_type="$1"
    
    secure_log "CRITICAL" "RÉPONSE D'URGENCE ACTIVÉE - Menace: $threat_type"
    
    case "$threat_type" in
        "ddos")
            # Activation du mode défense DDoS
            MAX_REQUESTS_PER_MINUTE=10
            secure_log "CRITICAL" "Mode anti-DDoS activé - Limite réduite à $MAX_REQUESTS_PER_MINUTE req/min"
            ;;
            
        "breach")
            # Révocation immédiate des tokens
            rm -f ~/.oauth_token
            rotate_api_keys "$(grep API_KEY /etc/api/config | cut -d= -f2)"
            secure_log "CRITICAL" "Tokens et clés API révoqués"
            ;;
            
        "intrusion")
            # Isolation du système
            secure_log "CRITICAL" "Mode d'isolation activé - Accès restreint"
            ;;
    esac
    
    # Notification d'urgence (simulée)
    echo "ALERTE URGENCE: $threat_type détecté - Réponse automatique activée" | mail -s "API Security Alert" admin@example.com 2>/dev/null || true
}

# Menu principal
case "${1:-}" in
    "call")
        secure_api_call "${2:-GET}" "${3:-/test}" "${4:-}" "${5:-127.0.0.1}"
        ;;
    "audit")
        security_audit
        ;;
    "emergency")
        emergency_response "${2:-unknown}"
        ;;
    "rotate-keys")
        rotate_api_keys "${2:-old_key}"
        ;;
    *)
        echo "Usage: $0 {call [method] [endpoint] [data] [ip]|audit|emergency [threat]|rotate-keys [old_key]}"
        echo
        echo "Exemples:"
        echo "  $0 call POST /users '{\"name\":\"John\"}' 192.168.1.100"
        echo "  $0 audit"
        echo "  $0 emergency ddos"
        ;;
esac
```

### 3.2 Monitoring et alerting de sécurité

```bash
#!/bin/bash
# Système de monitoring et alerting de sécurité API

# Configuration
MONITORING_INTERVAL=60  # secondes
ALERT_EMAIL="security@company.com"
LOG_FILE="security_monitoring.log"
THREAT_INTELLIGENCE_API="https://threatintel.example.com/api"

# Fonction de logging
log() {
    local level="$1"
    shift
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $*" | tee -a "$LOG_FILE"
}

# Fonction d'analyse des logs de sécurité
analyze_security_logs() {
    local log_file="$1"
    local analysis_period="${2:-3600}"  # 1 heure par défaut
    
    local cutoff_time=$(($(date +%s) - analysis_period))
    
    echo "=== ANALYSE DES LOGS DE SÉCURITÉ ==="
    echo "Période: dernière $(($analysis_period / 3600)) heure(s)"
    echo
    
    # Statistiques générales
    local total_events=$(grep -c "." "$log_file" 2>/dev/null || echo 0)
    local recent_events=$(awk -v cutoff="$cutoff_time" '$0 ~ /\[[0-9]{4}-[0-9]{2}-[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}\]/ {
        split($1, date_part, "[");
        split(date_part[2], time_part, "]");
        timestamp = mktime(gensub(/[-:]/, " ", "g", time_part[1]));
        if (timestamp >= cutoff) print;
    }' "$log_file" | wc -l)
    
    echo "Événements totaux: $total_events"
    echo "Événements récents: $recent_events"
    echo
    
    # Analyse par niveau de sévérité
    echo "Répartition par sévérité:"
    grep -E '\[(INFO|WARNING|ERROR|CRITICAL)\]' "$log_file" | \
        sed -E 's/.*\[(INFO|WARNING|ERROR|CRITICAL)\].*/\1/' | \
        sort | uniq -c | sort -nr
    
    echo
    
    # Détection d'attaques potentielles
    echo "Détection d'attaques:"
    
    # Scans de ports
    local port_scan_attempts=$(grep -c "Port scan\|Unusual port" "$log_file" 2>/dev/null || echo 0)
    echo "Tentatives de scan de ports: $port_scan_attempts"
    
    # Tentatives d'injection
    local injection_attempts=$(grep -c "SQL injection\|Command injection\|XSS" "$log_file" 2>/dev/null || echo 0)
    echo "Tentatives d'injection: $injection_attempts"
    
    # Échecs d'authentification
    local auth_failures=$(grep -c "Authentication failed\|Invalid credentials" "$log_file" 2>/dev/null || echo 0)
    echo "Échecs d'authentification: $auth_failures"
    
    # Violations de rate limit
    local rate_limit_violations=$(grep -c "Rate limit exceeded" "$log_file" 2>/dev/null || echo 0)
    echo "Violations de rate limit: $rate_limit_violations"
    
    # Adresses IP suspectes
    echo
    echo "Top 10 adresses IP suspectes:"
    grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' "$log_file" | \
        sort | uniq -c | sort -nr | head -10 | \
        awk '{print $2 ": " $1 " événements"}'
}

# Fonction de vérification de l'intelligence sur les menaces
check_threat_intelligence() {
    local ip_address="$1"
    
    if [[ -n "$THREAT_INTELLIGENCE_API" ]]; then
        local response=$(curl -s -H "Authorization: Bearer $THREAT_INTELLIGENCE_TOKEN" \
            "$THREAT_INTELLIGENCE_API/check/$ip_address")
        
        if echo "$response" | jq -e '.malicious' > /dev/null 2>&1; then
            local is_malicious=$(echo "$response" | jq -r '.malicious')
            local threat_level=$(echo "$response" | jq -r '.threat_level')
            
            if [[ "$is_malicious" == "true" ]]; then
                log "CRITICAL" "Adresse IP malveillante détectée: $ip_address (Niveau: $threat_level)"
                return 0
            fi
        fi
    fi
    
    return 1
}

# Fonction d'alerting intelligent
intelligent_alerting() {
    local alert_type="$1"
    local details="$2"
    local severity="${3:-WARNING}"
    
    # Évite les alertes en double (dans les dernières 5 minutes)
    local alert_key="${alert_type}_$(echo "$details" | md5sum | cut -d' ' -f1)"
    local cache_file="/tmp/alert_cache_$alert_key"
    
    if [[ -f "$cache_file" ]]; then
        local last_alert=$(cat "$cache_file")
        local current_time=$(date +%s)
        
        if [[ $((current_time - last_alert)) -lt 300 ]]; then
            # Alerte déjà envoyée récemment
            return
        fi
    fi
    
    # Enregistrement du timestamp de l'alerte
    date +%s > "$cache_file"
    
    # Construction du message d'alerte
    local subject="[$severity] Alerte Sécurité API: $alert_type"
    local body="Type d'alerte: $alert_type
Sévérité: $severity
Détails: $details
Horodatage: $(date)
Système: $(hostname)

Veuillez examiner les logs de sécurité pour plus de détails."

    # Envoi de l'alerte
    if command -v mail >/dev/null 2>&1; then
        echo "$body" | mail -s "$subject" "$ALERT_EMAIL"
        log "INFO" "Alerte $severity envoyée à $ALERT_EMAIL"
    else
        log "WARNING" "Commande mail non disponible - Alerte non envoyée"
    fi
    
    # Escalade pour les alertes critiques
    if [[ "$severity" == "CRITICAL" ]]; then
        # Appel d'un webhook pour escalade
        if [[ -n "$ESCALATION_WEBHOOK" ]]; then
            curl -s -X POST -H "Content-Type: application/json" \
                -d "{\"alert_type\":\"$alert_type\",\"severity\":\"$severity\",\"details\":\"$details\"}" \
                "$ESCALATION_WEBHOOK"
        fi
    fi
}

# Fonction de monitoring en temps réel
real_time_monitoring() {
    log "INFO" "Démarrage du monitoring en temps réel"
    
    # Suivi des métriques
    local metrics_file="/tmp/api_security_metrics"
    
    # Initialisation des compteurs
    echo "{
        \"start_time\": $(date +%s),
        \"total_requests\": 0,
        \"auth_failures\": 0,
        \"rate_limit_hits\": 0,
        \"suspicious_ips\": [],
        \"threats_detected\": 0
    }" > "$metrics_file"
    
    while true; do
        local current_time=$(date +%s)
        
        # Analyse des nouveaux logs
        if [[ -f "$LOG_FILE" ]]; then
            # Comptage des nouvelles alertes depuis la dernière vérification
            local recent_auth_failures=$(tail -n 100 "$LOG_FILE" | grep -c "Authentication failed" 2>/dev/null || echo 0)
            local recent_rate_limits=$(tail -n 100 "$LOG_FILE" | grep -c "Rate limit exceeded" 2>/dev/null || echo 0)
            
            # Mise à jour des métriques
            local metrics=$(cat "$metrics_file" | jq \
                --argjson auth_failures "$recent_auth_failures" \
                --argjson rate_limits "$recent_rate_limits" \
                '.auth_failures += $auth_failures | .rate_limit_hits += $rate_limits')
            
            echo "$metrics" > "$metrics_file"
            
            # Vérifications de seuils
            local total_auth_failures=$(echo "$metrics" | jq -r '.auth_failures')
            local total_rate_limits=$(echo "$metrics" | jq -r '.rate_limit_hits')
            
            if [[ $total_auth_failures -gt 10 ]]; then
                intelligent_alerting "Mass Authentication Failure" "Plus de 10 échecs d'authentification détectés" "CRITICAL"
            fi
            
            if [[ $total_rate_limits -gt 5 ]]; then
                intelligent_alerting "Rate Limit Abuse" "Plus de 5 violations de rate limit détectées" "WARNING"
            fi
        fi
        
        sleep $MONITORING_INTERVAL
    done
}

# Fonction de rapport de sécurité quotidien
daily_security_report() {
    local report_date=$(date +%Y%m%d)
    local report_file="security_report_$report_date.md"
    
    {
        echo "# Rapport de Sécurité API - $report_date"
        echo
        echo "## Résumé Exécutif"
        echo "- Date: $(date)"
        echo "- Période couverte: 24 dernières heures"
        echo
        
        echo "## Métriques de Sécurité"
        
        if [[ -f "/tmp/api_security_metrics" ]]; then
            local metrics=$(cat "/tmp/api_security_metrics")
            echo "- Requêtes totales: $(echo "$metrics" | jq -r '.total_requests')"
            echo "- Échecs d'authentification: $(echo "$metrics" | jq -r '.auth_failures')"
            echo "- Violations de rate limit: $(echo "$metrics" | jq -r '.rate_limit_hits')"
            echo "- Menaces détectées: $(echo "$metrics" | jq -r '.threats_detected')"
        fi
        
        echo
        
        echo "## Analyse des Logs"
        analyze_security_logs "$LOG_FILE" 86400  # 24 heures
        
        echo
        
        echo "## Recommandations"
        echo "1. **Surveillance continue**: Maintenir le monitoring en temps réel"
        echo "2. **Réponse aux incidents**: Automatiser la réponse aux menaces détectées"
        echo "3. **Mises à jour**: Vérifier régulièrement les signatures de menaces"
        echo "4. **Formation**: Former l'équipe aux bonnes pratiques de sécurité"
        echo
        
        echo "## Actions Requises"
        echo "- [ ] Examiner les échecs d'authentification répétés"
        echo "- [ ] Analyser les violations de rate limit"
        echo "- [ ] Mettre à jour les règles de détection"
        echo "- [ ] Archiver les logs de sécurité"
        
    } > "$report_file"
    
    log "INFO" "Rapport de sécurité quotidien généré: $report_file"
    
    # Envoi du rapport
    if [[ -n "$REPORT_EMAIL" ]]; then
        mail -s "Rapport de Sécurité API - $report_date" "$REPORT_EMAIL" < "$report_file"
    fi
}

# Fonction principale
main() {
    case "${1:-}" in
        "monitor")
            real_time_monitoring
            ;;
        "analyze")
            analyze_security_logs "${2:-$LOG_FILE}" "${3:-3600}"
            ;;
        "report")
            daily_security_report
            ;;
        "alert")
            intelligent_alerting "${2:-Test Alert}" "${3:-Test details}" "${4:-INFO}"
            ;;
        *)
            echo "Usage: $0 {monitor|analyze [log_file] [period_seconds]|report|alert [type] [details] [severity]}"
            echo
            echo "Exemples:"
            echo "  $0 monitor              # Monitoring en temps réel"
            echo "  $0 analyze              # Analyse des logs récents"
            echo "  $0 report               # Rapport quotidien"
            echo "  $0 alert 'Test' 'Test alert' WARNING"
            ;;
    esac
}

# Exécution
main "$@"
```

## Conclusion : La sécurité comme responsabilité collective

La sécurité des APIs ne peut être traitée comme une fonctionnalité optionnelle. Elle doit être intégrée dans chaque aspect du développement et de l'exploitation, depuis la conception initiale jusqu'au monitoring continu. Les menaces évoluent constamment, et notre capacité à les anticiper et à y répondre détermine la confiance que les utilisateurs placent dans nos systèmes.

Dans le prochain chapitre, nous explorerons les tendances émergentes dans l'écosystème API, incluant GraphQL, les APIs asynchrones, et l'avenir de l'intégration applicative.

---

**Exercice pratique :** Créez une suite complète d'outils de sécurité API qui inclut :
1. Scanner automatique de vulnérabilités OWASP Top 10
2. Système de rate limiting intelligent
3. Gestionnaire de secrets chiffrés
4. Monitoring de sécurité en temps réel
5. Génération automatique de rapports de conformité

**Challenge avancé :** Développez un système de réponse automatique aux incidents de sécurité qui :
- Détecte les compromissions en temps réel
- Isole automatiquement les systèmes affectés
- Notifie les équipes appropriées
- Lance des procédures de récupération
- Génère des rapports d'incident post-mortem

**Réflexion :** Comment l'approche "Security by Design" transforme-t-elle notre façon de concevoir et d'implémenter des APIs, et quels sont les impacts sur la productivité des équipes de développement ?

