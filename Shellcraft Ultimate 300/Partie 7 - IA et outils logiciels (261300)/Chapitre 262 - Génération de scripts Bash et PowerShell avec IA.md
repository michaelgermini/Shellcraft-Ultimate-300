# Chapitre 262 - Génération de scripts Bash et PowerShell avec IA

## Table des matières
- [Introduction](#introduction)
- [Principes de génération de code avec IA](#principes-de-génération-de-code-avec-ia)
- [Génération de scripts Bash](#génération-de-scripts-bash)
- [Génération de scripts PowerShell](#génération-de-scripts-powershell)
- [Optimisation et amélioration de scripts existants](#optimisation-et-amélioration-de-scripts-existants)
- [Génération de scripts multi-plateforme](#génération-de-scripts-multi-plateforme)
- [Bonnes pratiques et validation](#bonnes-pratiques-et-validation)
- [Conclusion](#conclusion)

## Introduction

La génération automatique de scripts avec l'intelligence artificielle transforme la façon dont nous créons des outils d'automatisation. Au lieu d'écrire manuellement chaque ligne de code, nous pouvons décrire nos besoins en langage naturel et laisser l'IA créer le script approprié.

Imaginez la génération de scripts IA comme un architecte qui transforme vos idées en plans détaillés : vous décrivez ce que vous voulez construire, et l'IA produit les plans techniques complets, prêts à être exécutés.

## Principes de génération de code avec IA

### Processus de génération

**Étapes typiques** :
1. **Description du besoin** : Exprimer l'objectif en langage naturel
2. **Analyse contextuelle** : L'IA comprend le contexte et les contraintes
3. **Génération du code** : Création du script approprié
4. **Validation** : Vérification de la syntaxe et de la logique
5. **Optimisation** : Amélioration du code généré

### Prompts efficaces

**Structure d'un bon prompt** :
```
[Contexte] + [Objectif] + [Contraintes] + [Format de sortie]
```

**Exemples de prompts** :
```bash
# Prompt basique
"Génère un script bash pour sauvegarder un répertoire"

# Prompt amélioré
"Génère un script bash robuste qui :
- Sauvegarde le répertoire /home/user vers /backup
- Crée une archive tar.gz avec horodatage
- Gère les erreurs proprement
- Affiche une barre de progression
- Envoie un email en cas de succès ou d'échec"

# Prompt avec contraintes spécifiques
"Génère un script bash qui :
- Fonctionne sur Ubuntu 20.04+
- Utilise uniquement des commandes POSIX
- Inclut la gestion d'erreurs avec set -euo pipefail
- Documente chaque section avec des commentaires"
```

## Génération de scripts Bash

### Script de génération automatique

**Générateur de scripts Bash avec IA** :
```bash
#!/bin/bash
# Script pour générer des scripts Bash avec IA

generate_bash_script() {
    local description="$1"
    local output_file="${2:-generated_script.sh}"
    local api_key="${OPENAI_API_KEY}"
    
    if [ -z "$api_key" ]; then
        echo "Erreur: OPENAI_API_KEY non définie"
        return 1
    fi
    
    local prompt="Génère un script bash complet et robuste avec les caractéristiques suivantes:
- Utilise 'set -euo pipefail' pour la gestion d'erreurs
- Inclut des commentaires détaillés
- Gère les erreurs proprement avec des messages clairs
- Suit les meilleures pratiques bash
- Inclut une fonction d'aide (--help)

Description du script: $description

Génère uniquement le code bash, sans explications supplémentaires."

    echo "Génération du script..."
    
    local response=$(curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $api_key" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 2000,
            \"temperature\": 0.2
        }")
    
    local script_content=$(echo "$response" | jq -r '.choices[0].message.content' 2>/dev/null)
    
    if [ -n "$script_content" ]; then
        # Nettoyer le contenu (enlever les blocs de code markdown si présents)
        script_content=$(echo "$script_content" | sed 's/```bash//g' | sed 's/```//g' | sed 's/^```$//g')
        
        # Ajouter le shebang si absent
        if ! echo "$script_content" | grep -q "^#!/bin/bash"; then
            script_content="#!/bin/bash\n\n$script_content"
        fi
        
        echo -e "$script_content" > "$output_file"
        chmod +x "$output_file"
        
        echo "Script généré: $output_file"
        echo "Lignes de code: $(wc -l < "$output_file")"
        return 0
    else
        echo "Erreur lors de la génération"
        return 1
    fi
}

# Utilisation
generate_bash_script \
    "Script de backup avec rotation automatique des anciennes sauvegardes" \
    "backup_script.sh"
```

### Exemples de génération

**Script de monitoring système** :
```bash
# Génération
generate_bash_script \
    "Script qui surveille l'utilisation CPU, mémoire et disque toutes les 5 minutes et envoie une alerte si les seuils sont dépassés (CPU > 80%, RAM > 90%, Disk > 85%)" \
    "system_monitor.sh"

# Script généré typique :
#!/bin/bash
set -euo pipefail

CPU_THRESHOLD=80
RAM_THRESHOLD=90
DISK_THRESHOLD=85
ALERT_EMAIL="admin@example.com"
INTERVAL=300  # 5 minutes

check_cpu() {
    local cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d'.' -f1)
    if [ "$cpu_usage" -gt "$CPU_THRESHOLD" ]; then
        echo "ALERTE CPU: ${cpu_usage}% (seuil: ${CPU_THRESHOLD}%)" | \
            mail -s "Alerte système - CPU" "$ALERT_EMAIL"
    fi
}

check_ram() {
    local ram_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
    if [ "$ram_usage" -gt "$RAM_THRESHOLD" ]; then
        echo "ALERTE RAM: ${ram_usage}% (seuil: ${RAM_THRESHOLD}%)" | \
            mail -s "Alerte système - RAM" "$ALERT_EMAIL"
    fi
}

check_disk() {
    local disk_usage=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
    if [ "$disk_usage" -gt "$DISK_THRESHOLD" ]; then
        echo "ALERTE DISQUE: ${disk_usage}% (seuil: ${DISK_THRESHOLD}%)" | \
            mail -s "Alerte système - Disque" "$ALERT_EMAIL"
    fi
}

while true; do
    check_cpu
    check_ram
    check_disk
    sleep "$INTERVAL"
done
```

**Script de déploiement** :
```bash
generate_bash_script \
    "Script de déploiement qui :
- Clone un dépôt Git depuis une branche spécifiée
- Exécute les tests
- Si les tests passent, déploie l'application
- Crée une sauvegarde avant déploiement
- Envoie une notification de succès ou d'échec" \
    "deploy.sh"
```

## Génération de scripts PowerShell

### Générateur PowerShell

**Script de génération PowerShell** :
```powershell
# Fonction de génération de scripts PowerShell avec IA

function New-AIScript {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Description,
        
        [Parameter(Mandatory=$false)]
        [string]$OutputFile = "generated_script.ps1",
        
        [Parameter(Mandatory=$false)]
        [string]$ApiKey = $env:OPENAI_API_KEY
    )
    
    if (-not $ApiKey) {
        Write-Error "OPENAI_API_KEY non définie"
        return
    }
    
    $prompt = @"
Génère un script PowerShell complet et robuste avec les caractéristiques suivantes:
- Utilise 'Set-StrictMode -Version Latest'
- Gère les erreurs avec try-catch
- Inclut des commentaires détaillés
- Suit les conventions PowerShell (verb-noun)
- Inclut une fonction d'aide (Get-Help)
- Utilise des paramètres typés et validés

Description du script: $Description

Génère uniquement le code PowerShell, sans explications supplémentaires.
"@
    
    Write-Host "Génération du script..." -ForegroundColor Yellow
    
    $body = @{
        model = "gpt-4"
        messages = @(
            @{
                role = "user"
                content = $prompt
            }
        )
        max_tokens = 2000
        temperature = 0.2
    } | ConvertTo-Json -Depth 10
    
    try {
        $response = Invoke-RestMethod -Uri "https://api.openai.com/v1/chat/completions" `
            -Method Post `
            -Headers @{
                "Content-Type" = "application/json"
                "Authorization" = "Bearer $ApiKey"
            } `
            -Body $body
        
        $scriptContent = $response.choices[0].message.content
        
        # Nettoyer le contenu
        $scriptContent = $scriptContent -replace '```powershell', '' -replace '```', ''
        
        # Ajouter le shebang si absent
        if ($scriptContent -notmatch '^#') {
            $scriptContent = "# PowerShell Script`n`n$scriptContent"
        }
        
        $scriptContent | Out-File -FilePath $OutputFile -Encoding UTF8
        
        Write-Host "Script généré: $OutputFile" -ForegroundColor Green
        Write-Host "Lignes de code: $((Get-Content $OutputFile | Measure-Object -Line).Lines)" -ForegroundColor Cyan
        
        return $OutputFile
    }
    catch {
        Write-Error "Erreur lors de la génération: $_"
        return $null
    }
}

# Utilisation
New-AIScript `
    -Description "Script qui liste tous les processus utilisant plus de 500MB de mémoire et les arrête après confirmation" `
    -OutputFile "kill_heavy_processes.ps1"
```

### Exemples PowerShell

**Script de gestion d'utilisateurs** :
```powershell
New-AIScript `
    -Description "Script PowerShell qui :
- Crée un nouvel utilisateur Active Directory
- Configure les permissions
- Crée le répertoire home
- Envoie un email de bienvenue avec les credentials temporaires" `
    -OutputFile "create_user.ps1"
```

**Script de monitoring Windows** :
```powershell
New-AIScript `
    -Description "Script qui surveille les services Windows et redémarre automatiquement ceux qui sont arrêtés, avec journalisation des événements" `
    -OutputFile "service_monitor.ps1"
```

## Optimisation et amélioration de scripts existants

### Optimisation automatique

**Script d'optimisation Bash** :
```bash
#!/bin/bash
# Optimise un script bash existant avec IA

optimize_bash_script() {
    local script_file="$1"
    local output_file="${2:-${script_file}.optimized}"
    
    if [ ! -f "$script_file" ]; then
        echo "Erreur: Fichier introuvable: $script_file"
        return 1
    fi
    
    local script_content=$(cat "$script_file")
    
    local prompt="Optimise ce script bash pour :
- Performance (réduire les appels système, utiliser des commandes efficaces)
- Lisibilité (améliorer les noms de variables, ajouter des commentaires)
- Robustesse (améliorer la gestion d'erreurs)
- Maintenabilité (modulariser si nécessaire)

Script actuel:
\`\`\`bash
$script_content
\`\`\`

Génère uniquement le script optimisé, sans explications."

    # Appel à l'API (similaire à generate_bash_script)
    # ...
}

# Utilisation
optimize_bash_script "mon_script.sh" "mon_script_optimized.sh"
```

### Amélioration incrémentale

**Fonction d'amélioration ciblée** :
```bash
improve_script_section() {
    local script_file="$1"
    local line_start="$2"
    local line_end="$3"
    local improvement_request="$4"
    
    local section=$(sed -n "${line_start},${line_end}p" "$script_file")
    
    local prompt="Améliore cette section de script bash selon: $improvement_request

Section actuelle:
\`\`\`bash
$section
\`\`\`

Génère uniquement la section améliorée."

    # Génération et remplacement
    # ...
}
```

## Génération de scripts multi-plateforme

### Scripts adaptatifs

**Générateur multi-plateforme** :
```bash
generate_cross_platform_script() {
    local description="$1"
    local platforms="${2:-linux,windows,macos}"
    
    local prompt="Génère un script qui fonctionne sur: $platforms

Le script doit :
- Détecter automatiquement le système d'exploitation
- Utiliser les commandes appropriées pour chaque plateforme
- Gérer les différences entre systèmes
- Inclure des fallbacks pour la compatibilité

Description: $description

Génère un script unique qui fonctionne sur toutes les plateformes spécifiées."

    # Génération avec détection de plateforme
    # ...
}
```

**Exemple généré** :
```bash
#!/bin/bash
# Script multi-plateforme généré

detect_os() {
    case "$(uname -s)" in
        Linux*)     echo "linux" ;;
        Darwin*)    echo "macos" ;;
        CYGWIN*)    echo "windows" ;;
        MINGW*)     echo "windows" ;;
        *)          echo "unknown" ;;
    esac
}

OS=$(detect_os)

case "$OS" in
    linux)
        PACKAGE_MANAGER="apt-get"
        SERVICE_CMD="systemctl"
        ;;
    macos)
        PACKAGE_MANAGER="brew"
        SERVICE_CMD="launchctl"
        ;;
    windows)
        # Utiliser WSL ou PowerShell selon le contexte
        PACKAGE_MANAGER="choco"
        SERVICE_CMD="sc"
        ;;
esac

# Code commun qui s'adapte selon l'OS
# ...
```

## Bonnes pratiques et validation

### Validation automatique

**Script de validation** :
```bash
validate_generated_script() {
    local script_file="$1"
    
    echo "Validation de: $script_file"
    
    # Vérification syntaxique
    if bash -n "$script_file" 2>/dev/null; then
        echo "✓ Syntaxe valide"
    else
        echo "✗ Erreur de syntaxe"
        bash -n "$script_file"
        return 1
    fi
    
    # Vérification des bonnes pratiques
    local issues=0
    
    # Vérifier set -euo pipefail
    if ! grep -q "set -euo pipefail" "$script_file"; then
        echo "⚠ Script n'utilise pas 'set -euo pipefail'"
        ((issues++))
    fi
    
    # Vérifier la gestion d'erreurs
    if ! grep -qE "(trap|set -e|error|fail)" "$script_file"; then
        echo "⚠ Gestion d'erreurs potentiellement insuffisante"
        ((issues++))
    fi
    
    # Vérifier les commentaires
    local comment_ratio=$(grep -c "^[[:space:]]*#" "$script_file" || echo 0)
    local total_lines=$(wc -l < "$script_file")
    local ratio=$((comment_ratio * 100 / total_lines))
    
    if [ "$ratio" -lt 10 ]; then
        echo "⚠ Peu de commentaires ($ratio%)"
        ((issues++))
    fi
    
    if [ $issues -eq 0 ]; then
        echo "✓ Script conforme aux bonnes pratiques"
        return 0
    else
        echo "⚠ $issues problème(s) détecté(s)"
        return 1
    fi
}
```

### Tests automatiques

**Génération avec tests** :
```bash
generate_with_tests() {
    local description="$1"
    local script_file="${2:-script.sh}"
    local test_file="${script_file%.sh}_test.sh"
    
    # Générer le script principal
    generate_bash_script "$description" "$script_file"
    
    # Générer les tests
    local test_prompt="Génère des tests bash (utilisant shunit2 ou bats) pour ce script: $script_file

Les tests doivent couvrir :
- Cas de succès
- Cas d'erreur
- Cas limites
- Validation des paramètres"

    # Génération des tests
    # ...
}
```

## Conclusion

La génération de scripts avec l'IA révolutionne la création d'outils d'automatisation, permettant de transformer rapidement des idées en code fonctionnel. Cependant, cette technologie nécessite une utilisation prudente : validation systématique, tests approfondis, et compréhension du code généré restent essentiels.

Comme un assistant de développement expérimenté, l'IA peut générer du code rapidement, mais c'est à l'humain de valider, optimiser et maintenir ce code pour garantir sa qualité et sa sécurité.

Dans le chapitre suivant, nous explorerons l'optimisation et le débogage de scripts avec l'IA, découvrant comment utiliser l'intelligence artificielle pour améliorer et corriger le code existant.

