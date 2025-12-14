# Chapitre 181 - Introduction aux APIs Web et protocoles HTTP

> "Les APIs sont les nouveaux shells - elles permettent d'interagir avec le monde numérique comme les shells le font avec les systèmes d'exploitation." - Architecte d'APIs

## Introduction : Le web comme système d'exploitation distribué

Dans cette nouvelle ère où les applications sont distribuées à travers le cloud et interconnectées via des APIs, le shell devient l'interface ultime pour orchestrer ces interactions complexes. Que ce soit pour récupérer des données météorologiques, automatiser des déploiements cloud, ou interagir avec des services d'IA, la maîtrise des protocoles web et des APIs REST devient aussi essentielle que celle des commandes système traditionnelles.

Dans ce chapitre inaugural de notre exploration des interactions web, nous poserons les bases des protocoles HTTP, des architectures REST, et des outils qui transforment le shell en client web universel.

## Section 1 : Fondamentaux du protocole HTTP

### 1.1 Anatomie d'une requête HTTP

```bash
# Requête HTTP de base avec curl
curl -X GET "https://httpbin.org/get" \
  -H "User-Agent: Shellcraft/1.0" \
  -H "Accept: application/json" \
  -v

# Équivalent avec wget
wget --method=GET \
     --header="User-Agent: Shellcraft/1.0" \
     --header="Accept: application/json" \
     https://httpbin.org/get \
     -O response.json
```

```powershell
# Requête HTTP avec PowerShell
Invoke-WebRequest -Uri "https://httpbin.org/get" `
    -Method GET `
    -Headers @{ "User-Agent" = "Shellcraft/1.0"; "Accept" = "application/json" } `
    -Verbose

# Avec .NET directement pour plus de contrôle
$request = [System.Net.WebRequest]::Create("https://httpbin.org/get")
$request.Method = "GET"
$request.UserAgent = "Shellcraft/1.0"
$request.Accept = "application/json"

$response = $request.GetResponse()
$reader = New-Object System.IO.StreamReader($response.GetResponseStream())
$content = $reader.ReadToEnd()
$reader.Close()
$response.Close()

$content | ConvertFrom-Json
```

### 1.2 Méthodes HTTP et leur sémantique

```bash
# GET - Récupération de ressources
curl -X GET "https://api.github.com/user" \
  -H "Authorization: token YOUR_TOKEN"

# POST - Création de ressources
curl -X POST "https://api.example.com/users" \
  -H "Content-Type: application/json" \
  -d '{"name": "John Doe", "email": "john@example.com"}'

# PUT - Mise à jour complète
curl -X PUT "https://api.example.com/users/123" \
  -H "Content-Type: application/json" \
  -d '{"name": "John Smith", "email": "john.smith@example.com"}'

# PATCH - Mise à jour partielle
curl -X PATCH "https://api.example.com/users/123" \
  -H "Content-Type: application/json" \
  -d '{"email": "john.smith@example.com"}'

# DELETE - Suppression
curl -X DELETE "https://api.example.com/users/123"

# HEAD - Métadonnées uniquement
curl -I "https://api.github.com/user"

# OPTIONS - Capacités du serveur
curl -X OPTIONS "https://api.example.com/users" \
  -H "Origin: https://myapp.com" \
  -v
```

### 1.3 Codes de statut HTTP

```bash
# Script pour tester différents codes de statut
#!/bin/bash

BASE_URL="https://httpbin.org/status"

# Codes 2xx - Succès
echo "=== Codes 2xx - Succès ==="
for code in 200 201 202 204; do
    echo "Code $code:"
    curl -s -o /dev/null -w "  Status: %{http_code}\n" "$BASE_URL/$code"
done

# Codes 3xx - Redirections
echo -e "\n=== Codes 3xx - Redirections ==="
for code in 301 302 307 308; do
    echo "Code $code:"
    curl -s -o /dev/null -w "  Status: %{http_code}\n" "$BASE_URL/$code"
done

# Codes 4xx - Erreurs client
echo -e "\n=== Codes 4xx - Erreurs client ==="
for code in 400 401 403 404 429; do
    echo "Code $code:"
    curl -s -o /dev/null -w "  Status: %{http_code}\n" "$BASE_URL/$code"
done

# Codes 5xx - Erreurs serveur
echo -e "\n=== Codes 5xx - Erreurs serveur ==="
for code in 500 502 503 504; do
    echo "Code $code:"
    curl -s -o /dev/null -w "  Status: %{http_code}\n" "$BASE_URL/$code"
done
```

```powershell
# Fonction PowerShell pour analyser les codes HTTP
function Get-HttpStatusInfo {
    param([int]$StatusCode)
    
    $statusCategories = @{
        100 = "Continue"
        101 = "Switching Protocols"
        200 = "OK"
        201 = "Created"
        202 = "Accepted"
        204 = "No Content"
        301 = "Moved Permanently"
        302 = "Found"
        304 = "Not Modified"
        400 = "Bad Request"
        401 = "Unauthorized"
        403 = "Forbidden"
        404 = "Not Found"
        405 = "Method Not Allowed"
        429 = "Too Many Requests"
        500 = "Internal Server Error"
        502 = "Bad Gateway"
        503 = "Service Unavailable"
        504 = "Gateway Timeout"
    }
    
    $category = [math]::Floor($StatusCode / 100) * 100
    $description = $statusCategories[$StatusCode]
    
    if (-not $description) {
        $description = switch ($category) {
            100 { "Information" }
            200 { "Success" }
            300 { "Redirection" }
            400 { "Client Error" }
            500 { "Server Error" }
            default { "Unknown" }
        }
    }
    
    [PSCustomObject]@{
        StatusCode = $StatusCode
        Category = $category
        Description = $description
        IsSuccess = $category -eq 200
        IsError = $category -ge 400
        IsClientError = $category -eq 400
        IsServerError = $category -eq 500
    }
}

# Tester plusieurs codes
200, 201, 301, 404, 500 | ForEach-Object {
    Get-HttpStatusInfo -StatusCode $_
} | Format-Table -AutoSize
```

## Section 2 : Authentification et sécurité web

### 2.1 Authentification de base (Basic Auth)

```bash
# Authentification Basic avec curl
curl -X GET "https://api.example.com/secure" \
  -u "username:password"

# Ou explicitement
curl -X GET "https://api.example.com/secure" \
  -H "Authorization: Basic $(echo -n 'username:password' | base64)"

# Avec un fichier de credentials
echo "username:password" > credentials.txt
curl -X GET "https://api.example.com/secure" \
  --user "$(cat credentials.txt)"

# Nettoyer
rm credentials.txt
```

```powershell
# Authentification Basic avec PowerShell
$username = "apiuser"
$password = "securepassword123"
$base64Auth = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("$username`:$password"))

Invoke-WebRequest -Uri "https://api.example.com/secure" `
    -Headers @{ "Authorization" = "Basic $base64Auth" }

# Avec PSCredential
$credential = Get-Credential -UserName "apiuser" -Message "API Credentials"
$base64Auth = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes("$($credential.UserName)`:$($credential.GetNetworkCredential().Password)"))

# Authentification Digest (plus complexe)
# Nécessite souvent une bibliothèque spécialisée
```

### 2.2 Tokens et API Keys

```bash
# Authentification par token (Bearer)
TOKEN="your_jwt_token_here"
curl -X GET "https://api.example.com/data" \
  -H "Authorization: Bearer $TOKEN"

# API Key dans l'en-tête
API_KEY="your_api_key"
curl -X GET "https://api.example.com/data" \
  -H "X-API-Key: $API_KEY"

# API Key en paramètre de requête
curl -X GET "https://api.example.com/data?api_key=$API_KEY"

# Authentification OAuth 2.0 - Flux d'autorisation
# 1. Obtenir le code d'autorisation
AUTH_URL="https://auth.example.com/oauth/authorize"
CLIENT_ID="your_client_id"
REDIRECT_URI="http://localhost:8080/callback"

curl -X GET "$AUTH_URL?client_id=$CLIENT_ID&response_type=code&redirect_uri=$REDIRECT_URI" \
  -L  # Suivre les redirections

# 2. Échanger le code contre un token
TOKEN_URL="https://auth.example.com/oauth/token"
CLIENT_SECRET="your_client_secret"
AUTH_CODE="code_from_callback"

curl -X POST "$TOKEN_URL" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=authorization_code&client_id=$CLIENT_ID&client_secret=$CLIENT_SECRET&code=$AUTH_CODE&redirect_uri=$REDIRECT_URI"
```

```powershell
# Gestion des tokens JWT
class JwtTokenManager {
    [string]$TokenEndpoint
    [string]$ClientId
    [string]$ClientSecret
    hidden [string]$_accessToken
    hidden [DateTime]$_tokenExpiry
    
    JwtTokenManager([string]$endpoint, [string]$clientId, [string]$clientSecret) {
        $this.TokenEndpoint = $endpoint
        $this.ClientId = $clientId
        $this.ClientSecret = $clientSecret
    }
    
    [string] GetAccessToken() {
        # Vérifier si le token est encore valide (avec marge de 5 minutes)
        if ($this._accessToken -and $this._tokenExpiry -gt (Get-Date).AddMinutes(5)) {
            return $this._accessToken
        }
        
        # Obtenir un nouveau token
        $body = @{
            grant_type = "client_credentials"
            client_id = $this.ClientId
            client_secret = $this.ClientSecret
            scope = "api.read api.write"
        }
        
        try {
            $response = Invoke-WebRequest -Uri $this.TokenEndpoint `
                -Method POST `
                -ContentType "application/x-www-form-urlencoded" `
                -Body $body
            
            $tokenData = $response.Content | ConvertFrom-Json
            
            $this._accessToken = $tokenData.access_token
            # Décoder le JWT pour obtenir l'expiration (simplifié)
            $payload = $this._accessToken.Split('.')[1]
            $payloadBytes = [System.Convert]::FromBase64String($payload.PadRight($payload.Length + (4 - $payload.Length % 4) % 4, '='))
            $payloadJson = [System.Text.Encoding]::UTF8.GetString($payloadBytes)
            $jwtPayload = $payloadJson | ConvertFrom-Json
            
            $this._tokenExpiry = [DateTime]::UnixEpoch.AddSeconds($jwtPayload.exp)
            
            return $this._accessToken
            
        } catch {
            throw "Échec de l'obtention du token: $($_.Exception.Message)"
        }
    }
    
    [object] InvokeAuthenticatedRequest([string]$uri, [string]$method = "GET", [object]$body = $null) {
        $token = $this.GetAccessToken()
        
        $headers = @{
            "Authorization" = "Bearer $token"
            "Content-Type" = "application/json"
        }
        
        $params = @{
            Uri = $uri
            Method = $method
            Headers = $headers
        }
        
        if ($body) {
            $params.Body = $body | ConvertTo-Json
        }
        
        return Invoke-WebRequest @params
    }
}

# Utilisation du gestionnaire de tokens
$tokenManager = [JwtTokenManager]::new(
    "https://auth.example.com/oauth/token",
    "my-client-id",
    "my-client-secret"
)

# Faire une requête authentifiée
$response = $tokenManager.InvokeAuthenticatedRequest("https://api.example.com/data")
$data = $response.Content | ConvertFrom-Json
```

### 2.3 Gestion des certificats et HTTPS

```bash
# Vérification des certificats SSL
curl -X GET "https://api.example.com/secure" \
  --cacert /etc/ssl/certs/ca-certificates.crt \
  -v

# Ignorer la vérification SSL (DANGER - seulement pour tests)
curl -X GET "https://self-signed.example.com/api" \
  -k  # --insecure

# Utiliser un certificat client
curl -X GET "https://api.example.com/client-cert-required" \
  --cert client.crt \
  --key client.key \
  --cacert ca.crt

# Vérifier la validité des certificats
openssl s_client -connect api.example.com:443 -servername api.example.com 2>/dev/null | openssl x509 -noout -dates
```

```powershell
# Gestion des certificats avec PowerShell
# Lister les certificats du magasin
Get-ChildItem -Path Cert:\CurrentUser\My | Select-Object Subject, NotAfter, Thumbprint

# Importer un certificat
Import-Certificate -FilePath "C:\Certs\client.pfx" -CertStoreLocation Cert:\CurrentUser\My

# Créer une requête web avec certificat client
$cert = Get-ChildItem -Path Cert:\CurrentUser\My | Where-Object { $_.Subject -like "*Client*" } | Select-Object -First 1

Invoke-WebRequest -Uri "https://api.example.com/client-cert" `
    -Certificate $cert `
    -Method GET

# Validation personnalisée des certificats SSL
function Test-SslCertificate {
    param(
        [string]$Url,
        [switch]$IgnoreCertificateErrors
    )
    
    try {
        $request = [System.Net.WebRequest]::Create($Url)
        $request.Method = "HEAD"
        
        if ($IgnoreCertificateErrors) {
            [System.Net.ServicePointManager]::ServerCertificateValidationCallback = { $true }
        }
        
        $response = $request.GetResponse()
        $certificate = $response.ServicePoint.Certificate
        
        [PSCustomObject]@{
            Url = $Url
            IsValid = $true
            Subject = $certificate.Subject
            Issuer = $certificate.Issuer
            ValidFrom = $certificate.GetEffectiveDateString()
            ValidTo = $certificate.GetExpirationDateString()
            Thumbprint = $certificate.GetCertHashString()
            SerialNumber = $certificate.GetSerialNumberString()
        }
        
        $response.Close()
        
    } catch {
        [PSCustomObject]@{
            Url = $Url
            IsValid = $false
            Error = $_.Exception.Message
        }
    } finally {
        if ($IgnoreCertificateErrors) {
            [System.Net.ServicePointManager]::ServerCertificateValidationCallback = $null
        }
    }
}

# Tester plusieurs URLs
"https://google.com", "https://github.com", "https://self-signed.local" | ForEach-Object {
    Test-SslCertificate -Url $_
} | Format-Table -AutoSize
```

## Section 3 : Architecture REST et design d'APIs

### 3.1 Principes REST

```bash
# Exemple d'API RESTful complète
BASE_URL="https://api.example.com/v1"

# Ressources imbriquées
# GET /users - Liste des utilisateurs
curl -X GET "$BASE_URL/users" \
  -H "Accept: application/json"

# GET /users/123 - Détails d'un utilisateur
curl -X GET "$BASE_URL/users/123" \
  -H "Accept: application/json"

# POST /users - Créer un utilisateur
curl -X POST "$BASE_URL/users" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Doe",
    "email": "john@example.com",
    "department": "IT"
  }'

# PUT /users/123 - Mettre à jour complètement
curl -X PUT "$BASE_URL/users/123" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "John Smith",
    "email": "john.smith@example.com",
    "department": "DevOps"
  }'

# PATCH /users/123 - Mise à jour partielle
curl -X PATCH "$BASE_URL/users/123" \
  -H "Content-Type: application/json" \
  -d '{
    "department": "DevOps"
  }'

# DELETE /users/123 - Supprimer
curl -X DELETE "$BASE_URL/users/123"

# Ressources liées
# GET /users/123/posts - Posts de l'utilisateur
curl -X GET "$BASE_URL/users/123/posts" \
  -H "Accept: application/json"

# Pagination
curl -X GET "$BASE_URL/users?page=2&limit=50" \
  -H "Accept: application/json"

# Filtrage et recherche
curl -X GET "$BASE_URL/users?department=IT&status=active" \
  -H "Accept: application/json"

# Tri
curl -X GET "$BASE_URL/users?sort=name&order=asc" \
  -H "Accept: application/json"
```

### 3.2 Content Types et sérialisation

```bash
# JSON (format le plus courant)
curl -X POST "https://api.example.com/webhook" \
  -H "Content-Type: application/json" \
  -d '{"event": "user.created", "user_id": 123}'

# XML
curl -X POST "https://api.example.com/legacy" \
  -H "Content-Type: application/xml" \
  -d '<user><name>John</name><email>john@example.com</email></user>'

# Form URL-encoded
curl -X POST "https://api.example.com/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=john&password=secret"

# Multipart/form-data (pour l'upload de fichiers)
curl -X POST "https://api.example.com/upload" \
  -F "file=@/path/to/file.txt" \
  -F "description=Document important"

# Accept différents formats
curl -X GET "https://api.example.com/data" \
  -H "Accept: application/xml"

curl -X GET "https://api.example.com/data" \
  -H "Accept: text/csv"
```

```powershell
# Gestion des différents content types
function Invoke-ApiRequest {
    param(
        [string]$Uri,
        [string]$Method = "GET",
        [object]$Body,
        [string]$ContentType = "application/json",
        [hashtable]$Headers = @{}
    )
    
    $params = @{
        Uri = $Uri
        Method = $Method
        Headers = $Headers
    }
    
    # Gestion du corps selon le content type
    if ($Body) {
        switch ($ContentType) {
            "application/json" {
                $params.Body = $Body | ConvertTo-Json -Depth 10
                $params.ContentType = $ContentType
            }
            "application/xml" {
                # Conversion objet vers XML
                $xmlWriter = New-Object System.Xml.XmlTextWriter
                $stringWriter = New-Object System.IO.StringWriter
                $xmlWriter.Formatting = "Indented"
                $xmlWriter.Indentation = 2
                $xmlWriter.IndentChar = " "
                $xmlWriter.WriteStartDocument()
                $xmlWriter.WriteStartElement("root")
                
                function ConvertToXml($obj, $writer) {
                    foreach ($prop in $obj.PSObject.Properties) {
                        $writer.WriteStartElement($prop.Name)
                        if ($prop.Value -is [PSCustomObject] -or $prop.Value -is [hashtable]) {
                            ConvertToXml $prop.Value $writer
                        } else {
                            $writer.WriteString($prop.Value.ToString())
                        }
                        $writer.WriteEndElement()
                    }
                }
                
                ConvertToXml $Body $xmlWriter
                $xmlWriter.WriteEndElement()
                $xmlWriter.WriteEndDocument()
                $xmlWriter.Flush()
                
                $params.Body = $stringWriter.ToString()
                $params.ContentType = $ContentType
            }
            "application/x-www-form-urlencoded" {
                $formData = @()
                foreach ($key in $Body.Keys) {
                    $formData += [System.Web.HttpUtility]::UrlEncode($key) + "=" + [System.Web.HttpUtility]::UrlEncode($Body[$key])
                }
                $params.Body = $formData -join "&"
                $params.ContentType = $ContentType
            }
        }
    }
    
    try {
        $response = Invoke-WebRequest @params
        
        # Parsing de la réponse selon le content type
        $responseContentType = $response.Headers.'Content-Type'
        if ($responseContentType -match "application/json") {
            return $response.Content | ConvertFrom-Json
        } elseif ($responseContentType -match "application/xml") {
            return [xml]$response.Content
        } else {
            return $response.Content
        }
        
    } catch {
        throw "API request failed: $($_.Exception.Message)"
    }
}

# Exemples d'utilisation
# JSON
$userData = Invoke-ApiRequest -Uri "https://api.example.com/users/123" -Method "GET"

# XML
$xmlData = Invoke-ApiRequest -Uri "https://api.example.com/legacy" -Method "GET"

# Form data
$formResult = Invoke-ApiRequest -Uri "https://api.example.com/login" `
    -Method "POST" `
    -Body @{ username = "john"; password = "secret" } `
    -ContentType "application/x-www-form-urlencoded"
```

## Section 4 : Outils et bibliothèques pour les interactions web

### 4.1 cURL : le couteau suisse du web

```bash
# Installation de curl (si nécessaire)
# Ubuntu/Debian
sudo apt-get update && sudo apt-get install curl

# CentOS/RHEL
sudo yum install curl

# macOS (déjà installé)
curl --version

# Fonctionnalités avancées de curl
# Téléchargement avec progression
curl -O -# "https://example.com/large-file.zip"

# Téléchargement en arrière-plan
curl -O "https://example.com/file.zip" &
jobs
fg  # Ramener au premier plan

# Limiter la bande passante
curl --limit-rate 100k -O "https://example.com/file.zip"

# Reprise de téléchargement
curl -C - -O "https://example.com/large-file.zip"

# Tests de performance
curl -w "@curl-format.txt" -o /dev/null -s "https://example.com/api"

# Où curl-format.txt contient :
#      time_namelookup:  %{time_namelookup}\n
#         time_connect:  %{time_connect}\n
#      time_appconnect:  %{time_appconnect}\n
#     time_pretransfer:  %{time_pretransfer}\n
#        time_redirect:  %{time_redirect}\n
#   time_starttransfer:  %{time_starttransfer}\n
#                      ----------\n
#           time_total:  %{time_total}\n

# Utilisation comme client REST
# Script de test d'API complet
#!/bin/bash

API_BASE="https://jsonplaceholder.typicode.com"
AUTH_TOKEN="your_token_here"

# Fonction d'aide pour les appels API
api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    
    local url="${API_BASE}${endpoint}"
    local headers=(-H "Content-Type: application/json")
    
    if [[ -n "$AUTH_TOKEN" ]]; then
        headers+=(-H "Authorization: Bearer $AUTH_TOKEN")
    fi
    
    if [[ -n "$data" ]]; then
        echo "curl -X $method \"${headers[@]}\" -d '$data' \"$url\""
        curl -X "$method" "${headers[@]}" -d "$data" "$url"
    else
        echo "curl -X $method \"${headers[@]}\" \"$url\""
        curl -X "$method" "${headers[@]}" "$url"
    fi
}

echo "=== Test de l'API JSONPlaceholder ==="

# GET - Récupérer des posts
echo -e "\n1. Récupération des posts:"
api_call "GET" "/posts" | head -20

# GET - Post spécifique
echo -e "\n2. Post spécifique:"
api_call "GET" "/posts/1"

# POST - Créer un nouveau post
echo -e "\n3. Création d'un post:"
api_call "POST" "/posts" '{
  "title": "Test Post from Shell",
  "body": "This is a test post created via curl",
  "userId": 1
}'

# PUT - Mettre à jour un post
echo -e "\n4. Mise à jour d'un post:"
api_call "PUT" "/posts/1" '{
  "id": 1,
  "title": "Updated Title",
  "body": "Updated content",
  "userId": 1
}'

# DELETE - Supprimer un post
echo -e "\n5. Suppression d'un post:"
api_call "DELETE" "/posts/1"

echo -e "\n=== Tests terminés ==="
```

### 4.2 jq : traitement JSON en ligne de commande

```bash
# Installation de jq
# Ubuntu/Debian
sudo apt-get install jq

# CentOS/RHEL
sudo yum install jq

# macOS
brew install jq

# Utilisation de base
curl -s "https://api.github.com/users/octocat" | jq '.'

# Extraction de valeurs
curl -s "https://api.github.com/users/octocat" | jq '.name'
curl -s "https://api.github.com/users/octocat" | jq '.login, .name, .location'

# Travail avec des tableaux
curl -s "https://api.github.com/users/octocat/repos" | jq '.[0:5] | .[] | {name: .name, language: .language, stars: .stargazers_count}'

# Filtres avancés
curl -s "https://api.github.com/search/repositories?q=language:powershell&sort=stars" | jq '.items[] | select(.stargazers_count > 1000) | {name: .name, stars: .stargazers_count, url: .html_url}'

# Modification de JSON
echo '{"name": "John", "age": 30}' | jq '.age = 31'
echo '{"users": [{"name": "John"}, {"name": "Jane"}]}' | jq '.users[] | select(.name == "John")'

# Scripts jq complexes
cat > process_api_data.jq << 'EOF'
# Script jq pour traiter des données d'API
.items
| map({
    id: .id,
    name: .name,
    status: (if .active then "Active" else "Inactive" end),
    tags: (.tags // []),
    metadata: {
        created: .created_at,
        modified: .updated_at,
        owner: .owner.login
    }
})
| sort_by(.metadata.created)
| reverse
EOF

curl -s "https://api.example.com/data" | jq -f process_api_data.jq
```

### 4.3 httpie : alternative moderne à curl

```bash
# Installation de httpie
# pip install httpie

# Utilisation simple
http GET https://api.github.com/user

# Avec authentification
http GET https://api.github.com/user Authorization:"token YOUR_TOKEN"

# POST avec JSON
http POST https://api.example.com/users name="John Doe" email="john@example.com"

# Upload de fichier
http POST https://api.example.com/upload file@/path/to/file.txt

# Sessions (cookies persistants)
http --session=logged-in POST https://api.example.com/login username=john password=secret
http --session=logged-in GET https://api.example.com/dashboard

# Mode verbose avec timing
http --verbose --timing GET https://api.github.com/user
```

## Section 5 : Patterns d'intégration web

### 5.1 Client API générique

```bash
#!/bin/bash
# Client API générique réutilisable

# Configuration
API_BASE_URL="${API_BASE_URL:-https://api.example.com/v1}"
API_TOKEN="${API_TOKEN:-}"
DEFAULT_TIMEOUT="${DEFAULT_TIMEOUT:-30}"

# Fonction de logging
log() {
    local level="$1"
    shift
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] [$level] $*" >&2
}

# Fonction principale pour les appels API
api_call() {
    local method="$1"
    local endpoint="$2"
    local data="$3"
    local content_type="${4:-application/json}"
    local timeout="${5:-$DEFAULT_TIMEOUT}"
    
    local url="${API_BASE_URL}${endpoint}"
    local headers=(-H "Accept: application/json")
    
    # Ajout du token si disponible
    if [[ -n "$API_TOKEN" ]]; then
        headers+=(-H "Authorization: Bearer $API_TOKEN")
    fi
    
    # Type de contenu pour les requêtes avec corps
    if [[ -n "$data" ]]; then
        headers+=(-H "Content-Type: $content_type")
    fi
    
    # Headers supplémentaires depuis les variables d'environnement
    if [[ -n "$API_HEADERS" ]]; then
        # API_HEADERS="X-Custom: value; X-Another: value2"
        IFS=';' read -ra HEADER_ARRAY <<< "$API_HEADERS"
        for header in "${HEADER_ARRAY[@]}"; do
            headers+=(-H "$header")
        done
    fi
    
    # Construction de la commande curl
    local curl_cmd=(curl -s -w "\nHTTP_STATUS:%{http_code}\n" --max-time "$timeout")
    
    if [[ "$DEBUG" == "true" ]]; then
        curl_cmd+=(-v)
        log "DEBUG" "Commande: ${curl_cmd[*]} -X $method ${headers[*]} ${data:+-d "$data"} $url"
    fi
    
    # Exécution
    local response
    if [[ -n "$data" ]]; then
        response="$("${curl_cmd[@]}" -X "$method" "${headers[@]}" -d "$data" "$url" 2>/dev/null)"
    else
        response="$("${curl_cmd[@]}" -X "$method" "${headers[@]}" "$url" 2>/dev/null)"
    fi
    
    # Séparation du statut et du corps
    local body status
    body="$(echo "$response" | sed '$d')"
    status="$(echo "$response" | tail -1 | cut -d: -f2)"
    
    # Gestion des erreurs
    if [[ "$status" -ge 400 ]]; then
        log "ERROR" "API call failed: $method $endpoint (Status: $status)"
        if [[ "$DEBUG" == "true" ]]; then
            log "ERROR" "Response: $body"
        fi
        return $status
    fi
    
    # Log de succès
    if [[ "$DEBUG" == "true" ]]; then
        log "DEBUG" "API call successful: $method $endpoint (Status: $status)"
    fi
    
    # Retour de la réponse
    echo "$body"
}

# Fonctions d'aide pour les méthodes HTTP courantes
api_get() {
    local endpoint="$1"
    api_call "GET" "$endpoint"
}

api_post() {
    local endpoint="$1"
    local data="$2"
    api_call "POST" "$endpoint" "$data"
}

api_put() {
    local endpoint="$1"
    local data="$2"
    api_call "PUT" "$endpoint" "$data"
}

api_patch() {
    local endpoint="$1"
    local data="$2"
    api_call "PATCH" "$endpoint" "$data"
}

api_delete() {
    local endpoint="$1"
    api_call "DELETE" "$endpoint"
}

# Exemples d'utilisation
if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
    # Configuration depuis l'environnement
    export API_BASE_URL="https://jsonplaceholder.typicode.com"
    
    echo "=== Test du client API générique ==="
    
    # GET
    echo "1. Récupération d'un post:"
    api_get "/posts/1" | jq '.'
    
    # POST
    echo -e "\n2. Création d'un post:"
    api_post "/posts" '{
        "title": "Test from Shell",
        "body": "This is a test post",
        "userId": 1
    }' | jq '.'
    
    # PUT
    echo -e "\n3. Mise à jour d'un post:"
    api_put "/posts/1" '{
        "id": 1,
        "title": "Updated Title",
        "body": "Updated content",
        "userId": 1
    }' | jq '.'
    
    echo -e "\n=== Tests terminés ==="
fi
```

### 5.2 Gestionnaire d'APIs multi-services

```powershell
# Classe pour gérer plusieurs APIs
class ApiManager {
    [hashtable]$ApiConfigurations
    hidden [hashtable]$_tokens
    
    ApiManager() {
        $this.ApiConfigurations = @{}
        $this._tokens = @{}
    }
    
    [void] AddApiConfiguration([string]$name, [hashtable]$config) {
        $this.ApiConfigurations[$name] = $config
    }
    
    [object] InvokeApiCall([string]$apiName, [string]$method, [string]$endpoint, [object]$body = $null, [hashtable]$queryParams = @{}) {
        if (-not $this.ApiConfigurations.ContainsKey($apiName)) {
            throw "Configuration API non trouvée: $apiName"
        }
        
        $config = $this.ApiConfigurations[$apiName]
        $baseUrl = $config.BaseUrl
        $authType = $config.AuthType
        
        # Construction de l'URL
        $url = "$baseUrl$endpoint"
        if ($queryParams.Count -gt 0) {
            $queryString = ($queryParams.Keys | ForEach-Object { "$_=$($queryParams[$_])" }) -join "&"
            $url = "$url`?$queryString"
        }
        
        # Préparation des headers
        $headers = @{}
        if ($config.DefaultHeaders) {
            $headers = $config.DefaultHeaders.Clone()
        }
        
        # Gestion de l'authentification
        switch ($authType) {
            "Bearer" {
                $token = $this.GetOrRefreshToken($apiName)
                $headers["Authorization"] = "Bearer $token"
            }
            "Basic" {
                $credentials = "$($config.Username):$($config.Password)"
                $encodedCredentials = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($credentials))
                $headers["Authorization"] = "Basic $encodedCredentials"
            }
            "ApiKey" {
                $headers[$config.ApiKeyHeader] = $config.ApiKey
            }
        }
        
        # Exécution de la requête
        $params = @{
            Uri = $url
            Method = $method
            Headers = $headers
        }
        
        if ($body) {
            if ($body -is [string]) {
                $params.Body = $body
            } else {
                $params.Body = $body | ConvertTo-Json -Depth 10
                $params.ContentType = "application/json"
            }
        }
        
        try {
            $response = Invoke-WebRequest @params
            
            # Parsing de la réponse selon le content type attendu
            $contentType = $response.Headers.'Content-Type'
            if ($contentType -match "application/json") {
                return $response.Content | ConvertFrom-Json
            } elseif ($contentType -match "application/xml") {
                return [xml]$response.Content
            } else {
                return $response.Content
            }
            
        } catch {
            $statusCode = $_.Exception.Response.StatusCode.value__
            throw "API call failed: $method $url (Status: $statusCode, Error: $($_.Exception.Message))"
        }
    }
    
    hidden [string] GetOrRefreshToken([string]$apiName) {
        $config = $this.ApiConfigurations[$apiName]
        $tokenKey = "$apiName`_token"
        $expiryKey = "$apiName`_expiry"
        
        # Vérifier si le token est encore valide
        if ($this._tokens.ContainsKey($tokenKey) -and 
            $this._tokens.ContainsKey($expiryKey) -and
            $this._tokens[$expiryKey] -gt (Get-Date).AddMinutes(5)) {
            return $this._tokens[$tokenKey]
        }
        
        # Obtenir un nouveau token
        $tokenUrl = $config.TokenUrl
        $tokenBody = @{
            grant_type = "client_credentials"
            client_id = $config.ClientId
            client_secret = $config.ClientSecret
            scope = $config.Scope -join " "
        }
        
        $tokenResponse = Invoke-WebRequest -Uri $tokenUrl -Method POST -Body $tokenBody -ContentType "application/x-www-form-urlencoded"
        $tokenData = $tokenResponse.Content | ConvertFrom-Json
        
        $this._tokens[$tokenKey] = $tokenData.access_token
        $this._tokens[$expiryKey] = (Get-Date).AddSeconds($tokenData.expires_in)
        
        return $tokenData.access_token
    }
}

# Utilisation du gestionnaire d'APIs
$apiManager = [ApiManager]::new()

# Configuration pour GitHub API
$apiManager.AddApiConfiguration("GitHub", @{
    BaseUrl = "https://api.github.com"
    AuthType = "Bearer"
    TokenUrl = "https://github.com/login/oauth/access_token"
    DefaultHeaders = @{ "Accept" = "application/vnd.github.v3+json" }
})

# Configuration pour une API interne
$apiManager.AddApiConfiguration("InternalAPI", @{
    BaseUrl = "https://internal-api.company.com/v1"
    AuthType = "Basic"
    Username = "apiuser"
    Password = "securepassword"
})

# Configuration pour une API avec clé
$apiManager.AddApiConfiguration("WeatherAPI", @{
    BaseUrl = "https://api.weather.com/v1"
    AuthType = "ApiKey"
    ApiKeyHeader = "X-API-Key"
    ApiKey = "your-weather-api-key"
})

# Utilisation des APIs configurées
try {
    # Appel GitHub API
    $userData = $apiManager.InvokeApiCall("GitHub", "GET", "/user")
    Write-Host "GitHub user: $($userData.login)"
    
    # Appel API météo
    $weatherData = $apiManager.InvokeApiCall("WeatherAPI", "GET", "/current", $null, @{ q = "Paris"; units = "metric" })
    Write-Host "Weather in Paris: $($weatherData.main.temp)°C"
    
    # Appel API interne
    $internalData = $apiManager.InvokeApiCall("InternalAPI", "POST", "/reports/generate", @{
        reportType = "user-activity"
        dateRange = @{
            start = "2024-01-01"
            end = "2024-01-31"
        }
    })
    Write-Host "Report generated: $($internalData.reportId)"
    
} catch {
    Write-Error "API call failed: $($_.Exception.Message)"
}
```

## Conclusion : Le shell comme interface universelle

Les protocoles HTTP et les APIs REST transforment le shell d'un outil d'administration système en une interface universelle capable d'interagir avec n'importe quel service web. Que ce soit pour récupérer des données météorologiques, automatiser des déploiements cloud, ou orchestrer des workflows complexes, la maîtrise de ces protocoles devient essentielle.

Dans le prochain chapitre, nous explorerons l'utilisation avancée de cURL pour les scénarios complexes : téléchargements parallèles, gestion des sessions, et intégration avec des pipelines d'automatisation.

---

**Exercice pratique :** Créez un script de monitoring d'APIs qui :
1. Teste la disponibilité de plusieurs services
2. Mesure les temps de réponse
3. Vérifie les codes de retour HTTP
4. Génère des rapports avec graphiques
5. Envoie des alertes en cas de problème

**Challenge avancé :** Développez un client API générique qui supporte :
- Authentification OAuth 2.0 complète
- Pagination automatique
- Retry avec backoff exponentiel
- Cache intelligent des réponses
- Métriques de performance intégrées

**Réflexion :** Comment les APIs changent-elles notre perception du shell, et quels sont les impacts sur l'architecture des systèmes d'automatisation moderne ?

