# Chapitre 195 - Intégration avec Bash et PowerShell

## Table des matières
- [Introduction](#introduction)
- [Intégration Bash](#intégration-bash)
- [Intégration PowerShell](#intégration-powershell)
- [Scripts hybrides](#scripts-hybrides)
- [Conclusion](#conclusion)

## Introduction

L'intégration de cURL avec Bash et PowerShell permet de créer des scripts puissants qui combinent les capacités de chaque environnement.

## Intégration Bash

**Fonctions Bash pour APIs** :
```bash
#!/bin/bash
# Intégration Bash avec cURL

# Fonction générique API
api_call() {
    local method="$1"
    local endpoint="$2"
    local data="${3:-}"
    
    local response=$(curl -s -X "$method" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $TOKEN" \
        ${data:+-d "$data"} \
        "${API_BASE_URL}${endpoint}")
    
    echo "$response"
}

# Parser JSON avec jq
get_json_value() {
    local json="$1"
    local key="$2"
    echo "$json" | jq -r ".$key"
}

# Exemple d'utilisation
get_user() {
    local user_id="$1"
    local response=$(api_call "GET" "/users/$user_id")
    local name=$(get_json_value "$response" "name")
    echo "Utilisateur: $name"
}
```

## Intégration PowerShell

**Fonctions PowerShell pour APIs** :
```powershell
# Intégration PowerShell avec cURL

function Invoke-ApiCall {
    param(
        [string]$Method = "GET",
        [string]$Endpoint,
        [hashtable]$Data = @{},
        [string]$Token = $env:API_TOKEN
    )
    
    $headers = @{
        "Content-Type" = "application/json"
        "Authorization" = "Bearer $Token"
    }
    
    $uri = "$env:API_BASE_URL$Endpoint"
    
    if ($Data.Count -gt 0) {
        $body = $Data | ConvertTo-Json
        $response = Invoke-RestMethod -Uri $uri -Method $Method -Headers $headers -Body $body
    } else {
        $response = Invoke-RestMethod -Uri $uri -Method $Method -Headers $headers
    }
    
    return $response
}

# Exemple d'utilisation
function Get-User {
    param([string]$UserId)
    
    $response = Invoke-ApiCall -Method "GET" -Endpoint "/users/$UserId"
    return $response
}
```

## Scripts hybrides

**Script fonctionnant sur les deux plateformes** :
```bash
#!/bin/bash
# Script hybride Bash/PowerShell

if [ -n "$PSVersionTable" ]; then
    # PowerShell
    $response = Invoke-ApiCall -Method "GET" -Endpoint "/users"
    Write-Output $response
else
    # Bash
    response=$(api_call "GET" "/users")
    echo "$response"
fi
```

## Conclusion

L'intégration avec Bash et PowerShell permet d'utiliser cURL dans différents environnements selon les besoins.

