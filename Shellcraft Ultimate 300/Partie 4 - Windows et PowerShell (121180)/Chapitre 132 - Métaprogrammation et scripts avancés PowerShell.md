# Chapitre 132 - Métaprogrammation et scripts avancés PowerShell

> "La métaprogrammation est comme écrire du code qui écrit du code. C'est l'art de créer des programmes qui se programment eux-mêmes, ouvrant des possibilités infinies d'automatisation et d'adaptation." - Citation inspirée des concepts de métaprogrammation

## Introduction : PowerShell comme langage de métaprogrammation

La métaprogrammation représente l'étape ultime de maîtrise de PowerShell. Au-delà des cmdlets et scripts traditionnels, PowerShell offre des capacités de métaprogrammation permettant de créer du code dynamique, d'étendre le langage lui-même, et de construire des abstractions de haut niveau.

Dans ce chapitre, nous explorerons les techniques avancées de génération de code, de manipulation d'objets dynamiques, et de création de DSL (Domain-Specific Languages) en PowerShell.

## Section 1 : Génération dynamique de code

### 1.1 ScriptBlock et Invoke-Expression

**Création et manipulation de blocs de script :**
```powershell
# Création de ScriptBlocks dynamiques
$scriptTemplate = @'
param(
    [string]$ServerName,
    [string]$ServiceName
)

Write-Host "Vérification du service $ServiceName sur $ServerName"
$service = Get-Service -ComputerName $ServerName -Name $ServiceName

if ($service.Status -eq 'Running') {
    Write-Host "✓ Service $ServiceName fonctionne correctement" -ForegroundColor Green
} else {
    Write-Warning "⚠ Service $ServiceName est arrêté"
    # Tentative de redémarrage automatique
    try {
        Start-Service -InputObject $service
        Write-Host "✓ Service redémarré automatiquement" -ForegroundColor Green
    } catch {
        Write-Error "✗ Impossible de redémarrer le service: $_"
    }
}

return $service.Status
'@

# Conversion en ScriptBlock
$checkServiceScript = [scriptblock]::Create($scriptTemplate)

# Exécution du script dynamique
$result = & $checkServiceScript -ServerName "localhost" -ServiceName "wuauserv"

# Génération de code avec Invoke-Expression
function New-DynamicFunction {
    param(
        [string]$FunctionName,
        [string[]]$Parameters,
        [string]$Body
    )

    $paramList = $Parameters -join ', '
    $functionCode = @"
function $FunctionName {
    param($paramList)
    $Body
}
"@

    # Création de la fonction dans la portée globale
    Invoke-Expression $functionCode

    Write-Host "Fonction $FunctionName créée dynamiquement"
}

# Exemple d'utilisation
New-DynamicFunction -FunctionName "Test-CustomService" -Parameters @('ComputerName', 'ServiceName') -Body @'
    Write-Host "Test personnalisé du service $ServiceName sur $ComputerName"
    $result = Get-Service -ComputerName $ComputerName -Name $ServiceName
    return $result.Status
'@

# Utilisation de la fonction créée
Test-CustomService -ComputerName "localhost" -ServiceName "WinRM"

# Génération de code basée sur des templates
function New-ServiceMonitor {
    param(
        [string[]]$ServiceNames,
        [string]$ComputerName = $env:COMPUTERNAME
    )

    $monitorCode = @"
# Service Monitor généré automatiquement
# Généré le: $(Get-Date)
# Services surveillés: $($ServiceNames -join ', ')
# Serveur: $ComputerName

function Get-ServiceHealth {
    param([string]`$Computer = '$ComputerName')

    `$services = @()
"@

    foreach ($service in $ServiceNames) {
        $monitorCode += @"

    # Vérification du service $service
    try {
        `$svc = Get-Service -ComputerName `$Computer -Name '$service' -ErrorAction Stop
        `$services += [PSCustomObject]@{
            Name = '$service'
            Status = `$svc.Status
            StartType = `$svc.StartType
            Health = if (`$svc.Status -eq 'Running') { 'Healthy' } else { 'Unhealthy' }
        }
    } catch {
        `$services += [PSCustomObject]@{
            Name = '$service'
            Status = 'NotFound'
            StartType = 'Unknown'
            Health = 'Error'
        }
    }
"@
    }

    $monitorCode += @"

    return `$services
}

# Fonction d'alerte automatique
function Send-ServiceAlert {
    param(
        [Parameter(Mandatory=`$true)]
        [string]`$Computer,

        [Parameter(Mandatory=`$true)]
        [PSCustomObject[]]`$UnhealthyServices
    )

    `$subject = "Alerte Services - $Computer"
    `$body = @"
Services défaillants détectés sur $Computer :

$(`$UnhealthyServices | ForEach-Object { "- `$(`$_.Name): `$(`$_.Status)" } | Out-String)

Vérifiez immédiatement l'état du serveur.
"@

    # Envoi d'email (configuration SMTP requise)
    try {
        Send-MailMessage -To "admin@company.com" -Subject `$subject -Body `$body -SmtpServer "smtp.company.com"
        Write-Host "Alerte envoyée pour $($UnhealthyServices.Count) services défaillants"
    } catch {
        Write-Error "Échec de l'envoi d'alerte: $_"
    }
}
"@

    # Création du fichier de monitoring
    $fileName = "ServiceMonitor_$($ComputerName -replace '[^a-zA-Z0-9]', '_').ps1"
    $monitorCode | Out-File -FilePath $fileName -Encoding UTF8

    Write-Host "Fichier de monitoring créé: $fileName"
    return $fileName
}

# Génération d'un monitor pour les services critiques
$monitorFile = New-ServiceMonitor -ServiceNames @("wuauserv", "WinRM", "W32Time", "Spooler")
```

### 1.2 AST (Abstract Syntax Tree) et analyse de code

**Manipulation du code PowerShell via l'AST :**
```powershell
# Analyse de code PowerShell avec l'AST
function Analyze-PowerShellCode {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Code
    )

    try {
        # Parsing du code en AST
        $ast = [System.Management.Automation.Language.Parser]::ParseInput($Code, [ref]$null, [ref]$null)

        $analysis = @{
            Functions = @()
            Variables = @()
            Commands = @()
            Parameters = @()
            Errors = @()
        }

        # Analyse des fonctions
        $functions = $ast.FindAll({ $args[0] -is [System.Management.Automation.Language.FunctionDefinitionAst] }, $true)
        foreach ($func in $functions) {
            $analysis.Functions += @{
                Name = $func.Name
                Parameters = $func.Parameters | ForEach-Object { $_.Name.VariablePath.UserPath }
                Body = $func.Body.Extent.Text
                Line = $func.Extent.StartLineNumber
            }
        }

        # Analyse des variables
        $variables = $ast.FindAll({ $args[0] -is [System.Management.Automation.Language.VariableExpressionAst] }, $true)
        $analysis.Variables = $variables | ForEach-Object { $_.VariablePath.UserPath } | Select-Object -Unique

        # Analyse des commandes
        $commands = $ast.FindAll({ $args[0] -is [System.Management.Automation.Language.CommandAst] }, $true)
        $analysis.Commands = $commands | ForEach-Object { $_.CommandElements[0].Value } | Select-Object -Unique

        # Analyse des paramètres
        $parameters = $ast.FindAll({ $args[0] -is [System.Management.Automation.Language.ParameterAst] }, $true)
        $analysis.Parameters = $parameters | ForEach-Object { $_.Name.VariablePath.UserPath } | Select-Object -Unique

        return $analysis

    } catch {
        return @{
            Functions = @()
            Variables = @()
            Commands = @()
            Parameters = @()
            Errors = @($_.Exception.Message)
        }
    }
}

# Exemple d'analyse de code
$sampleCode = @'
function Test-Function {
    param(
        [string]$ComputerName,
        [int]$Timeout = 30
    )

    $service = Get-Service -ComputerName $ComputerName -Name "WinRM"
    Write-Host "Service status: $($service.Status)"

    return $service.Status
}

$result = Test-Function -ComputerName "localhost"
'@

$analysis = Analyze-PowerShellCode -Code $sampleCode
Write-Host "=== ANALYSE DU CODE ==="
Write-Host "Fonctions trouvées: $($analysis.Functions.Count)"
foreach ($func in $analysis.Functions) {
    Write-Host "  - $($func.Name) (ligne $($func.Line))"
    Write-Host "    Paramètres: $($func.Parameters -join ', ')"
}

Write-Host "Variables trouvées: $($analysis.Variables -join ', ')"
Write-Host "Commandes utilisées: $($analysis.Commands -join ', ')"

# Génération de code basée sur l'analyse
function Generate-CodeDocumentation {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Code
    )

    $analysis = Analyze-PowerShellCode -Code $Code

    $doc = @"
# Documentation générée automatiquement
# Généré le: $(Get-Date)

## Fonctions définies

"@

    foreach ($func in $analysis.Functions) {
        $doc += @"

### $($func.Name)
- **Ligne**: $($func.Line)
- **Paramètres**: $($func.Parameters -join ', ')
- **Description**: Fonction générique

"@
    }

    $doc += @"

## Variables utilisées
$($analysis.Variables -join ', ')

## Commandes externes
$($analysis.Commands -join ', ')

## Métriques de complexité
- Nombre de fonctions: $($analysis.Functions.Count)
- Nombre de variables: $($analysis.Variables.Count)
- Nombre de commandes: $($analysis.Commands.Count)
"@

    return $doc
}

# Génération de documentation
$documentation = Generate-CodeDocumentation -Code $sampleCode
$documentation | Out-File -FilePath "CodeDocumentation.md" -Encoding UTF8

# Réécriture automatique de code
function Optimize-PowerShellCode {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Code
    )

    $optimizedCode = $Code

    # Optimisations automatiques
    $optimizations = @(
        # Remplacement de Get-Service suivi de Where-Object
        @{
            Pattern = 'Get-Service.*\|.*Where-Object.*\{\s*\$_\.Status\s*-eq\s*[''\"]([^''\"]+)[''\"]\s*\}'
            Replacement = 'Get-Service -Status $1'
        },
        # Optimisation des pipelines
        @{
            Pattern = '\$var\s*=\s*Get-.*\|\s*Where-Object.*\{\s*\$_\.Name\s*-eq\s*[''\"]([^''\"]+)[''\"]\s*\}'
            Replacement = '$var = Get-* -Name $1'
        }
    )

    foreach ($opt in $optimizations) {
        $optimizedCode = $optimizedCode -replace $opt.Pattern, $opt.Replacement
    }

    return $optimizedCode
}
```

### 1.3 Création de DSL (Domain-Specific Languages)

**Construction d'un DSL pour l'administration système :**
```powershell
# DSL pour l'administration de serveurs
class ServerAdminDSL {
    [hashtable]$Servers = @{}
    [scriptblock]$CurrentScriptBlock
    [string]$CurrentServerName

    # Méthode pour définir un serveur
    [void]Server([string]$name, [scriptblock]$scriptBlock) {
        $this.CurrentServerName = $name
        $this.CurrentScriptBlock = $scriptBlock
        $this.Servers[$name] = @{
            Name = $name
            Configuration = @{}
            Actions = @()
            ScriptBlock = $scriptBlock
        }
    }

    # Méthodes DSL pour la configuration
    [void]IPAddress([string]$ip) {
        $this.Servers[$this.CurrentServerName].Configuration.IPAddress = $ip
    }

    [void]Domain([string]$domain) {
        $this.Servers[$this.CurrentServerName].Configuration.Domain = $domain
    }

    [void]Role([string[]]$roles) {
        $this.Servers[$this.CurrentServerName].Configuration.Roles = $roles
    }

    [void]Backup([string]$schedule, [string]$destination) {
        $this.Servers[$this.CurrentServerName].Actions += @{
            Type = 'Backup'
            Schedule = $schedule
            Destination = $destination
        }
    }

    [void]Monitor([string]$service, [string]$alertEmail) {
        $this.Servers[$this.CurrentServerName].Actions += @{
            Type = 'Monitor'
            Service = $service
            AlertEmail = $alertEmail
        }
    }

    [void]Update([string]$schedule) {
        $this.Servers[$this.CurrentServerName].Actions += @{
            Type = 'Update'
            Schedule = $schedule
        }
    }

    # Exécution du DSL
    [void]Execute() {
        foreach ($serverName in $this.Servers.Keys) {
            $server = $this.Servers[$serverName]
            Write-Host "Configuration du serveur: $serverName" -ForegroundColor Cyan

            # Exécution du scriptblock dans le contexte DSL
            & $server.ScriptBlock

            # Application de la configuration
            $this.ApplyConfiguration($server)
        }
    }

    [void]ApplyConfiguration([hashtable]$server) {
        Write-Host "  Configuration: $($server.Configuration | ConvertTo-Json -Compress)"

        foreach ($action in $server.Actions) {
            Write-Host "  Action: $($action.Type) - $($action | ConvertTo-Json -Compress)" -ForegroundColor Yellow

            # Simulation de l'exécution des actions
            switch ($action.Type) {
                'Backup' {
                    Write-Host "    Planification sauvegarde: $($action.Schedule) vers $($action.Destination)"
                }
                'Monitor' {
                    Write-Host "    Configuration monitoring: $($action.Service) -> $($action.AlertEmail)"
                }
                'Update' {
                    Write-Host "    Planification mises à jour: $($action.Schedule)"
                }
            }
        }
    }

    # Génération de scripts PowerShell à partir du DSL
    [string]GenerateScripts() {
        $scripts = @()

        foreach ($serverName in $this.Servers.Keys) {
            $server = $this.Servers[$serverName]

            $script = @"
# Script généré pour le serveur: $serverName
# Généré le: $(Get-Date)

Write-Host "Configuration du serveur $serverName" -ForegroundColor Green

# Configuration réseau
`$ipAddress = "$($server.Configuration.IPAddress)"
`$domain = "$($server.Configuration.Domain)"

if (`$ipAddress) {
    Write-Host "Configuration IP: `$ipAddress"
    # Code de configuration IP...
}

if (`$domain) {
    Write-Host "Jonction domaine: `$domain"
    # Code de jonction domaine...
}

# Rôles à installer
`$roles = @($($server.Configuration.Roles | ForEach-Object { "'$_'" } | Join-String -Separator ', '))

foreach (`$role in `$roles) {
    Write-Host "Installation du rôle: `$role"
    # Code d'installation des rôles...
}

# Actions planifiées
"@

            foreach ($action in $server.Actions) {
                switch ($action.Type) {
                    'Backup' {
                        $script += @"

# Configuration sauvegarde
`$backupAction = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-Command `"& { Backup-Server -Destination '$($action.Destination)' }`""
`$backupTrigger = New-ScheduledTaskTrigger - $($action.Schedule -replace 'ly', 'Interval 1 -DaysOfWeek ')
Register-ScheduledTask -TaskName "Backup_$serverName" -Action `$backupAction -Trigger `$backupTrigger

"@
                    }
                    'Monitor' {
                        $script += @"

# Configuration monitoring
`$monitorScript = {
    `$service = Get-Service -Name '$($action.Service)' -ErrorAction SilentlyContinue
    if (`$service.Status -ne 'Running') {
        Send-MailMessage -To '$($action.AlertEmail)' -Subject "Service Alert: $($action.Service) on $serverName" -Body "Le service $($action.Service) est arrêté sur $serverName"
    }
}

# Planification du monitoring
`$monitorAction = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-Command `$monitorScript"
`$monitorTrigger = New-ScheduledTaskTrigger -Daily -At "09:00"
Register-ScheduledTask -TaskName "Monitor_$($action.Service)_$serverName" -Action `$monitorAction -Trigger `$monitorTrigger

"@
                    }
                }
            }

            $script += @"

Write-Host "Configuration de $serverName terminée" -ForegroundColor Green
"@

            $scripts += $script
        }

        return $scripts -join "`n`n# ========================================`n`n"
    }
}

# Fonction globale pour utiliser le DSL
function Invoke-ServerAdminDSL {
    param(
        [Parameter(Mandatory=$true)]
        [scriptblock]$Configuration
    )

    $dsl = [ServerAdminDSL]::new()

    # Exécution du scriptblock de configuration dans le contexte du DSL
    & $Configuration

    return $dsl
}

# Exemple d'utilisation du DSL
$serverConfig = Invoke-ServerAdminDSL -Configuration {
    Server "WEB01" {
        IPAddress "192.168.1.10"
        Domain "company.local"
        Role @("Web-Server", "Web-Mgmt-Tools")

        Backup "Weekly -DaysOfWeek Sunday -At '02:00'" "D:\Backups"
        Monitor "W3SVC" "admin@company.com"
        Update "Monthly -Day 1 -At '03:00'"
    }

    Server "DB01" {
        IPAddress "192.168.1.20"
        Domain "company.local"
        Role @("SQL-Server", "SQL-Server-Tools")

        Backup "Daily -At '01:00'" "\\backup\database"
        Monitor "MSSQLSERVER" "dba@company.com"
        Update "Weekly -DaysOfWeek Saturday -At '04:00'"
    }

    Server "DC01" {
        IPAddress "192.168.1.30"
        Domain "company.local"
        Role @("AD-Domain-Services", "DNS")

        Backup "Daily -At '23:00'" "D:\Backups"
        Monitor "DNS" "it-admin@company.com"
        Update "Monthly -Day 15 -At '02:00'"
    }
}

# Exécution de la configuration
$serverConfig.Execute()

# Génération des scripts PowerShell
$generatedScripts = $serverConfig.GenerateScripts()
$generatedScripts | Out-File -FilePath "GeneratedServerScripts.ps1" -Encoding UTF8

Write-Host "Scripts générés sauvegardés dans: GeneratedServerScripts.ps1"
```

## Section 2 : Classes et programmation orientée objet avancée

### 2.1 Classes PowerShell et héritage

**Création de hiérarchies de classes complexes :**
```powershell
# Définition d'une classe de base pour les ressources système
class SystemResource {
    [string]$Name
    [string]$Type
    [datetime]$CreatedDate
    [System.Collections.Generic.List[string]]$Tags
    hidden [hashtable]$Metadata

    SystemResource([string]$name, [string]$type) {
        $this.Name = $name
        $this.Type = $type
        $this.CreatedDate = Get-Date
        $this.Tags = [System.Collections.Generic.List[string]]::new()
        $this.Metadata = @{}
    }

    [void]AddTag([string]$tag) {
        if (-not $this.Tags.Contains($tag)) {
            $this.Tags.Add($tag)
        }
    }

    [void]RemoveTag([string]$tag) {
        $this.Tags.Remove($tag)
    }

    [bool]HasTag([string]$tag) {
        return $this.Tags.Contains($tag)
    }

    [void]SetMetadata([string]$key, [object]$value) {
        $this.Metadata[$key] = $value
    }

    [object]GetMetadata([string]$key) {
        return $this.Metadata[$key]
    }

    [hashtable]GetInfo() {
        return @{
            Name = $this.Name
            Type = $this.Type
            CreatedDate = $this.CreatedDate
            Tags = $this.Tags.ToArray()
            Metadata = $this.Metadata.Clone()
        }
    }

    [string]ToString() {
        return "$($this.Type): $($this.Name)"
    }
}

# Classe pour les services Windows
class WindowsService : SystemResource {
    [string]$ServiceName
    [string]$DisplayName
    [string]$Status
    [string]$StartType
    hidden [object]$ServiceObject

    WindowsService([string]$serviceName) : base($serviceName, "WindowsService") {
        $this.ServiceName = $serviceName
        $this.Refresh()
    }

    [void]Refresh() {
        try {
            $this.ServiceObject = Get-Service -Name $this.ServiceName -ErrorAction Stop
            $this.DisplayName = $this.ServiceObject.DisplayName
            $this.Status = $this.ServiceObject.Status.ToString()
            $this.StartType = $this.ServiceObject.StartType.ToString()
        } catch {
            throw "Service '$($this.ServiceName)' introuvable: $_"
        }
    }

    [void]Start() {
        if ($this.Status -ne "Running") {
            Start-Service -Name $this.ServiceName
            $this.Refresh()
            Write-Host "Service $($this.ServiceName) démarré"
        }
    }

    [void]Stop() {
        if ($this.Status -eq "Running") {
            Stop-Service -Name $this.ServiceName
            $this.Refresh()
            Write-Host "Service $($this.ServiceName) arrêté"
        }
    }

    [void]Restart() {
        Restart-Service -Name $this.ServiceName
        $this.Refresh()
        Write-Host "Service $($this.ServiceName) redémarré"
    }

    [void]SetStartType([string]$startType) {
        $validTypes = @("Automatic", "Manual", "Disabled")
        if ($startType -notin $validTypes) {
            throw "Type de démarrage invalide. Valeurs possibles: $($validTypes -join ', ')"
        }

        Set-Service -Name $this.ServiceName -StartupType $startType
        $this.Refresh()
        Write-Host "Type de démarrage de $($this.ServiceName) défini sur $startType"
    }

    [hashtable]GetDependencies() {
        $service = Get-CimInstance -ClassName Win32_Service -Filter "Name='$($this.ServiceName)'"
        return @{
            DependsOn = $service.DependentServices | ForEach-Object { $_.Name }
            DependedBy = $service.DependentServices | ForEach-Object { $_.Name }
        }
    }

    [hashtable]GetPerformanceMetrics() {
        $process = Get-Process -Name "*$($this.ServiceName)*" -ErrorAction SilentlyContinue | Select-Object -First 1
        if ($process) {
            return @{
                CPU = $process.CPU
                MemoryMB = [math]::Round($process.WorkingSet / 1MB, 2)
                Threads = $process.Threads.Count
                Handles = $process.Handles
            }
        }
        return @{}
    }
}

# Classe pour les processus système
class SystemProcess : SystemResource {
    [int]$ProcessId
    [string]$ProcessName
    [double]$CPUUsage
    [long]$MemoryUsage
    [int]$ThreadCount
    hidden [object]$ProcessObject

    SystemProcess([int]$processId) : base("PID_$processId", "SystemProcess") {
        $this.ProcessId = $processId
        $this.Refresh()
    }

    SystemProcess([string]$processName) : base($processName, "SystemProcess") {
        $this.ProcessName = $processName
        $this.Refresh()
    }

    [void]Refresh() {
        if ($this.ProcessId) {
            $this.ProcessObject = Get-Process -Id $this.ProcessId -ErrorAction SilentlyContinue
        } else {
            $this.ProcessObject = Get-Process -Name $this.ProcessName -ErrorAction SilentlyContinue | Select-Object -First 1
        }

        if ($this.ProcessObject) {
            $this.ProcessId = $this.ProcessObject.Id
            $this.ProcessName = $this.ProcessObject.ProcessName
            $this.CPUUsage = $this.ProcessObject.CPU
            $this.MemoryUsage = $this.ProcessObject.WorkingSet
            $this.ThreadCount = $this.ProcessObject.Threads.Count
        } else {
            throw "Processus introuvable"
        }
    }

    [void]Kill() {
        Stop-Process -Id $this.ProcessId -Force
        Write-Host "Processus $($this.ProcessName) (PID: $($this.ProcessId)) terminé"
    }

    [void]SetPriority([string]$priority) {
        $validPriorities = @("Idle", "BelowNormal", "Normal", "AboveNormal", "High", "RealTime")
        if ($priority -notin $validPriorities) {
            throw "Priorité invalide. Valeurs possibles: $($validPriorities -join ', ')"
        }

        $this.ProcessObject.PriorityClass = $priority
        Write-Host "Priorité du processus $($this.ProcessName) définie sur $priority"
    }

    [hashtable]GetModules() {
        $modules = $this.ProcessObject.Modules | Select-Object -First 10
        return $modules | ForEach-Object { $_.ModuleName }
    }

    [void]Monitor([int]$durationSeconds = 60) {
        Write-Host "Monitoring du processus $($this.ProcessName) pendant $durationSeconds secondes..."

        $startTime = Get-Date
        $cpuSamples = @()
        $memorySamples = @()

        while ((Get-Date) - $startTime).TotalSeconds -lt $durationSeconds) {
            $this.Refresh()
            $cpuSamples += $this.CPUUsage
            $memorySamples += [math]::Round($this.MemoryUsage / 1MB, 2)

            Start-Sleep -Seconds 5
        }

        $stats = @{
            AverageCPU = [math]::Round(($cpuSamples | Measure-Object -Average).Average, 2)
            MaxCPU = [math]::Round(($cpuSamples | Measure-Object -Maximum).Maximum, 2)
            AverageMemoryMB = [math]::Round(($memorySamples | Measure-Object -Average).Average, 2)
            MaxMemoryMB = [math]::Round(($memorySamples | Measure-Object -Maximum).Maximum, 2)
            SampleCount = $cpuSamples.Count
        }

        Write-Host "Statistiques de monitoring:"
        Write-Host "  CPU - Moyenne: $($stats.AverageCPU)%, Max: $($stats.MaxCPU)%"
        Write-Host "  Mémoire - Moyenne: $($stats.AverageMemoryMB) MB, Max: $($stats.MaxMemoryMB) MB"
        Write-Host "  Échantillons: $($stats.SampleCount)"

        return $stats
    }
}

# Classe pour la gestion des ressources système
class SystemResourceManager {
    [System.Collections.Generic.List[SystemResource]]$Resources

    SystemResourceManager() {
        $this.Resources = [System.Collections.Generic.List[SystemResource]]::new()
    }

    [void]AddResource([SystemResource]$resource) {
        $this.Resources.Add($resource)
    }

    [SystemResource[]]GetResourcesByType([string]$type) {
        return $this.Resources | Where-Object { $_.Type -eq $type }
    }

    [SystemResource[]]GetResourcesByTag([string]$tag) {
        return $this.Resources | Where-Object { $_.HasTag($tag) }
    }

    [hashtable]GetSystemOverview() {
        $overview = @{
            TotalResources = $this.Resources.Count
            ResourceTypes = @{}
            TaggedResources = @{}
            RecentResources = @()
        }

        # Comptage par type
        foreach ($resource in $this.Resources) {
            if (-not $overview.ResourceTypes.ContainsKey($resource.Type)) {
                $overview.ResourceTypes[$resource.Type] = 0
            }
            $overview.ResourceTypes[$resource.Type]++
        }

        # Ressources taguées
        foreach ($resource in $this.Resources) {
            foreach ($tag in $resource.Tags) {
                if (-not $overview.TaggedResources.ContainsKey($tag)) {
                    $overview.TaggedResources[$tag] = [System.Collections.Generic.List[SystemResource]]::new()
                }
                $overview.TaggedResources[$tag].Add($resource)
            }
        }

        # Ressources récentes (dernières 24h)
        $yesterday = (Get-Date).AddDays(-1)
        $overview.RecentResources = $this.Resources | Where-Object { $_.CreatedDate -gt $yesterday }

        return $overview
    }

    [void]PerformHealthCheck() {
        Write-Host "=== VÉRIFICATION DE SANTÉ DU SYSTÈME ===" -ForegroundColor Cyan

        $healthy = 0
        $unhealthy = 0

        foreach ($resource in $this.Resources) {
            $isHealthy = $true

            switch ($resource.Type) {
                "WindowsService" {
                    if ($resource.Status -ne "Running") {
                        $isHealthy = $false
                    }
                }
                "SystemProcess" {
                    # Vérification arbitraire - processus utilisant plus de 500MB
                    if ($resource.MemoryUsage -gt 500MB) {
                        $isHealthy = $false
                    }
                }
            }

            if ($isHealthy) {
                Write-Host "✓ $($resource.ToString())" -ForegroundColor Green
                $healthy++
            } else {
                Write-Host "✗ $($resource.ToString())" -ForegroundColor Red
                $unhealthy++
            }
        }

        Write-Host "`nRésumé: $healthy sains, $unhealthy défaillants" -ForegroundColor Yellow
    }

    [void]ExportConfiguration([string]$filePath) {
        $config = @{
            ExportDate = Get-Date
            Resources = @()
        }

        foreach ($resource in $this.Resources) {
            $config.Resources += $resource.GetInfo()
        }

        $config | ConvertTo-Json -Depth 10 | Out-File -FilePath $filePath -Encoding UTF8
        Write-Host "Configuration exportée vers: $filePath"
    }

    [void]ImportConfiguration([string]$filePath) {
        if (-not (Test-Path $filePath)) {
            throw "Fichier de configuration introuvable: $filePath"
        }

        $config = Get-Content $filePath | ConvertFrom-Json

        foreach ($resourceConfig in $config.Resources) {
            $resource = $null

            switch ($resourceConfig.Type) {
                "WindowsService" {
                    $resource = [WindowsService]::new($resourceConfig.Name)
                }
                "SystemProcess" {
                    if ($resourceConfig.Name -match "PID_(\d+)") {
                        $pid = [int]$Matches[1]
                        $resource = [SystemProcess]::new($pid)
                    } else {
                        $resource = [SystemProcess]::new($resourceConfig.Name)
                    }
                }
            }

            if ($resource) {
                # Restaurer les tags
                foreach ($tag in $resourceConfig.Tags) {
                    $resource.AddTag($tag)
                }

                # Restaurer les métadonnées
                foreach ($key in $resourceConfig.Metadata.Keys) {
                    $resource.SetMetadata($key, $resourceConfig.Metadata[$key])
                }

                $this.AddResource($resource)
            }
        }

        Write-Host "Configuration importée depuis: $filePath"
    }
}

# Exemple d'utilisation des classes
$manager = [SystemResourceManager]::new()

# Création de ressources
$webService = [WindowsService]::new("W3SVC")
$webService.AddTag("web")
$webService.AddTag("critical")

$sqlService = [WindowsService]::new("MSSQLSERVER")
$sqlService.AddTag("database")
$sqlService.AddTag("critical")

$chromeProcess = [SystemProcess]::new("chrome")
$chromeProcess.AddTag("browser")

# Ajout au gestionnaire
$manager.AddResource($webService)
$manager.AddResource($sqlService)
$manager.AddResource($chromeProcess)

# Vue d'ensemble du système
$overview = $manager.GetSystemOverview()
Write-Host "=== APERÇU DU SYSTÈME ==="
Write-Host "Ressources totales: $($overview.TotalResources)"
Write-Host "Types de ressources:"
foreach ($type in $overview.ResourceTypes.Keys) {
    Write-Host "  $type : $($overview.ResourceTypes[$type])"
}

# Vérification de santé
$manager.PerformHealthCheck()

# Export/Import de configuration
$manager.ExportConfiguration("SystemResources.json")

$newManager = [SystemResourceManager]::new()
$newManager.ImportConfiguration("SystemResources.json")
```

### 2.2 Interfaces et polymorphisme

**Implémentation de patterns de conception avancés :**
```powershell
# Définition d'interfaces via classes abstraites
class IMonitorable {
    [void]CheckHealth() { throw "Method CheckHealth must be implemented" }
    [hashtable]GetMetrics() { throw "Method GetMetrics must be implemented" }
    [void]Alert([string]$message) { throw "Method Alert must be implemented" }
}

class IConfigurable {
    [void]LoadConfiguration([hashtable]$config) { throw "Method LoadConfiguration must be implemented" }
    [hashtable]GetConfiguration() { throw "Method GetConfiguration must be implemented" }
    [void]SaveConfiguration([string]$path) { throw "Method SaveConfiguration must be implemented" }
}

class ILoggable {
    [void]Log([string]$message, [string]$level = "INFO") { throw "Method Log must be implemented" }
    [string[]]GetLogs([int]$lines = 100) { throw "Method GetLogs must be implemented" }
    [void]ClearLogs() { throw "Method ClearLogs must be implemented" }
}

# Implémentation d'un service monitor avec interfaces multiples
class MonitoredService : WindowsService, IMonitorable, IConfigurable, ILoggable {
    [System.Collections.Generic.List[string]]$Logs
    [hashtable]$Configuration
    [string]$LogFilePath

    MonitoredService([string]$serviceName) : base($serviceName) {
        $this.Logs = [System.Collections.Generic.List[string]]::new()
        $this.Configuration = @{
            HealthCheckInterval = 60
            AlertEmail = "admin@company.com"
            MaxRetries = 3
            LogLevel = "INFO"
        }
        $this.LogFilePath = "$env:TEMP\$serviceName`_monitor.log"
    }

    # Implémentation IMonitorable
    [void]CheckHealth() {
        $this.Log("Vérification de santé du service $($this.ServiceName)")

        try {
            $this.Refresh()

            if ($this.Status -eq "Running") {
                $this.Log("Service $($this.ServiceName) est sain", "INFO")
            } else {
                $this.Log("Service $($this.ServiceName) est arrêté", "WARN")
                $this.Alert("Service $($this.ServiceName) détecté comme arrêté")
            }
        } catch {
            $this.Log("Erreur lors de la vérification de santé: $_", "ERROR")
        }
    }

    [hashtable]GetMetrics() {
        $metrics = @{
            ServiceName = $this.ServiceName
            Status = $this.Status
            StartType = $this.StartType
            Uptime = $null
            RestartCount = 0
        }

        # Calcul de l'uptime (simplifié)
        try {
            $serviceInfo = Get-CimInstance -ClassName Win32_Service -Filter "Name='$($this.ServiceName)'"
            if ($serviceInfo) {
                $metrics.Uptime = (Get-Date) - $serviceInfo.InstallDate
            }
        } catch {
            $this.Log("Impossible de calculer l'uptime: $_", "WARN")
        }

        return $metrics
    }

    [void]Alert([string]$message) {
        $this.Log("ALERTE: $message", "ERROR")

        try {
            $subject = "Alerte Service: $($this.ServiceName)"
            $body = @"
$message

Service: $($this.ServiceName)
Serveur: $($env:COMPUTERNAME)
Horodatage: $(Get-Date)

Vérifiez immédiatement l'état du service.
"@

            Send-MailMessage -To $this.Configuration.AlertEmail -Subject $subject -Body $body -SmtpServer "smtp.company.com"
            $this.Log("Alerte envoyée à $($this.Configuration.AlertEmail)", "INFO")
        } catch {
            $this.Log("Échec de l'envoi d'alerte: $_", "ERROR")
        }
    }

    # Implémentation IConfigurable
    [void]LoadConfiguration([hashtable]$config) {
        foreach ($key in $config.Keys) {
            $this.Configuration[$key] = $config[$key]
        }
        $this.Log("Configuration chargée: $($config.Keys -join ', ')", "INFO")
    }

    [hashtable]GetConfiguration() {
        return $this.Configuration.Clone()
    }

    [void]SaveConfiguration([string]$path) {
        $this.Configuration | ConvertTo-Json | Out-File -FilePath $path -Encoding UTF8
        $this.Log("Configuration sauvegardée dans: $path", "INFO")
    }

    # Implémentation ILoggable
    [void]Log([string]$message, [string]$level = "INFO") {
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
        $logEntry = "[$timestamp] [$level] $message"

        $this.Logs.Add($logEntry)

        # Écriture dans le fichier de log
        $logEntry | Out-File -FilePath $this.LogFilePath -Append -Encoding UTF8

        # Affichage console selon le niveau
        $color = switch ($level) {
            "ERROR" { "Red" }
            "WARN" { "Yellow" }
            "INFO" { "White" }
            "DEBUG" { "Gray" }
            default { "White" }
        }

        Write-Host $logEntry -ForegroundColor $color
    }

    [string[]]GetLogs([int]$lines = 100) {
        if ($this.Logs.Count -le $lines) {
            return $this.Logs.ToArray()
        } else {
            return $this.Logs.GetRange($this.Logs.Count - $lines, $lines).ToArray()
        }
    }

    [void]ClearLogs() {
        $this.Logs.Clear()
        $this.Log("Logs effacés", "INFO")
    }

    # Méthodes supplémentaires spécifiques au service monitoré
    [void]StartMonitoring() {
        $this.Log("Démarrage du monitoring du service $($this.ServiceName)", "INFO")

        # Création d'une tâche planifiée pour les vérifications de santé
        $action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-Command & { `$svc = [MonitoredService]::new('$($this.ServiceName)'); `$svc.CheckHealth() }"
        $trigger = New-ScheduledTaskTrigger -RepetitionInterval (New-TimeSpan -Minutes $this.Configuration.HealthCheckInterval) -RepetitionDuration (New-TimeSpan -Days 365)

        Register-ScheduledTask -TaskName "Monitor_$($this.ServiceName)" -Action $action -Trigger $trigger -Principal (New-ScheduledTaskPrincipal -UserId "SYSTEM")

        $this.Log("Monitoring planifié toutes les $($this.Configuration.HealthCheckInterval) minutes", "INFO")
    }

    [void]StopMonitoring() {
        Unregister-ScheduledTask -TaskName "Monitor_$($this.ServiceName)" -Confirm:$false
        $this.Log("Monitoring arrêté", "INFO")
    }

    [void]RestartWithMonitoring() {
        $this.Restart()
        Start-Sleep -Seconds 5
        $this.CheckHealth()
    }
}

# Fonction utilitaire pour travailler avec des objets implémentant des interfaces
function Invoke-HealthChecks {
    param([IMonitorable[]]$monitorables)

    foreach ($monitorable in $monitorables) {
        Write-Host "Vérification de santé: $($monitorable.GetType().Name)" -ForegroundColor Cyan
        $monitorable.CheckHealth()
        $metrics = $monitorable.GetMetrics()
        Write-Host "Métriques: $($metrics | ConvertTo-Json -Compress)" -ForegroundColor Gray
        Write-Host ""
    }
}

function Backup-Configurations {
    param([IConfigurable[]]$configurables, [string]$backupPath)

    $timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
    $backupDir = Join-Path $backupPath "config_backup_$timestamp"
    New-Item -ItemType Directory -Path $backupDir -Force | Out-Null

    foreach ($configurable in $configurables) {
        $configName = $configurable.GetType().Name
        $configPath = Join-Path $backupDir "$configName.json"
        $configurable.SaveConfiguration($configPath)
    }

    Write-Host "Configurations sauvegardées dans: $backupDir"
}

# Exemple d'utilisation avec polymorphisme
$services = @(
    [MonitoredService]::new("W3SVC"),
    [MonitoredService]::new("MSSQLSERVER")
)

# Configuration personnalisée pour chaque service
$services[0].LoadConfiguration(@{
    HealthCheckInterval = 30
    AlertEmail = "web-admin@company.com"
    LogLevel = "DEBUG"
})

$services[1].LoadConfiguration(@{
    HealthCheckInterval = 60
    AlertEmail = "db-admin@company.com"
    MaxRetries = 5
})

# Utilisation polymorphique des interfaces
Invoke-HealthChecks -monitorables $services

Backup-Configurations -configurables $services -backupPath "C:\Backups"

# Démarrage du monitoring pour tous les services
foreach ($service in $services) {
    $service.StartMonitoring()
}

# Simulation d'un problème et vérification des logs
$services[0].Alert("Test d'alerte manuel")

Write-Host "=== LOGS DU SERVICE W3SVC ==="
$services[0].GetLogs(5) | ForEach-Object { Write-Host $_ }

# Nettoyage
foreach ($service in $services) {
    $service.StopMonitoring()
}
```

## Section 3 : Métaprogrammation système avancée

### 3.1 Manipulation des modules et cmdlets dynamiques

**Création de cmdlets dynamiques et modules :**
```powershell
# Création de cmdlets dynamiques
function New-DynamicCmdlet {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Name,

        [Parameter(Mandatory=$true)]
        [scriptblock]$Implementation,

        [string]$Noun = "Item",
        [string]$Verb = "Get"
    )

    # Définition des paramètres du cmdlet
    $paramBlock = {
        param(
            [Parameter(ValueFromPipeline=$true)]
            [object[]]$InputObject,

            [switch]$PassThru
        )
    }

    # Création du scriptblock du cmdlet
    $cmdletScript = [scriptblock]::Create(@"
$paramBlock

process {
    foreach (`$item in `$InputObject) {
        `$result = & $Implementation `$item
        if (`$PassThru -or `$result) {
            `$result
        }
    }
}
"@)

    # Enregistrement du cmdlet dans la session
    $null = New-Item -Path "function:global:$Name" -Value $cmdletScript -Force

    Write-Host "Cmdlet dynamique '$Name' créé"
}

# Exemple d'utilisation
New-DynamicCmdlet -Name "Get-ServiceStatus" -Implementation {
    param($service)
    $svc = Get-Service -Name $service -ErrorAction SilentlyContinue
    if ($svc) {
        [PSCustomObject]@{
            Name = $svc.Name
            Status = $svc.Status
            DisplayName = $svc.DisplayName
        }
    }
}

# Utilisation du cmdlet créé
Get-ServiceStatus -InputObject "WinRM", "W32Time", "Spooler" | Format-Table

# Création de modules dynamiques
function New-DynamicModule {
    param(
        [Parameter(Mandatory=$true)]
        [string]$ModuleName,

        [Parameter(Mandatory=$true)]
        [hashtable]$Functions,

        [string]$Description = ""
    )

    $modulePath = "$env:TEMP\$ModuleName.psm1"

    # Génération du contenu du module
    $moduleContent = @"
# Module dynamique: $ModuleName
# Généré le: $(Get-Date)
# Description: $Description

"@

    foreach ($functionName in $Functions.Keys) {
        $functionImpl = $Functions[$functionName]

        $moduleContent += @"

function $functionName {
$functionImpl
}

"@
    }

    $moduleContent += @"

# Export des fonctions
Export-ModuleMember -Function $($Functions.Keys -join ', ')

# Initialisation du module
Write-Host "Module $ModuleName chargé" -ForegroundColor Green
"@

    # Création du fichier de module
    $moduleContent | Out-File -FilePath $modulePath -Encoding UTF8

    # Import du module
    Import-Module $modulePath -Force

    Write-Host "Module dynamique '$ModuleName' créé et importé"
    return $modulePath
}

# Création d'un module de monitoring
$monitoringFunctions = @{
    'Start-SystemMonitor' = {
        param([int]$IntervalSeconds = 60)

        $global:MonitoringJob = Start-Job -ScriptBlock {
            param($interval)

            while ($true) {
                $timestamp = Get-Date
                $cpu = Get-WmiObject Win32_Processor | Measure-Object -Property LoadPercentage -Average
                $memory = Get-WmiObject Win32_OperatingSystem

                $metrics = @{
                    Timestamp = $timestamp
                    CPU = $cpu.Average
                    MemoryUsed = $memory.TotalVisibleMemorySize - $memory.FreePhysicalMemory
                    MemoryTotal = $memory.TotalVisibleMemorySize
                }

                # Stockage des métriques (en mémoire pour cet exemple)
                if (-not $global:MonitoringData) {
                    $global:MonitoringData = @()
                }
                $global:MonitoringData += $metrics

                # Limitation à 1000 points
                if ($global:MonitoringData.Count -gt 1000) {
                    $global:MonitoringData = $global:MonitoringData[-500..-1]
                }

                Start-Sleep -Seconds $interval
            }
        } -ArgumentList $IntervalSeconds

        Write-Host "Monitoring système démarré (intervalle: $IntervalSeconds secondes)"
    }

    'Stop-SystemMonitor' = {
        if ($global:MonitoringJob) {
            Stop-Job $global:MonitoringJob
            Remove-Job $global:MonitoringJob
            $global:MonitoringJob = $null
            Write-Host "Monitoring système arrêté"
        } else {
            Write-Warning "Aucun monitoring en cours"
        }
    }

    'Get-SystemMetrics' = {
        param([int]$LastHours = 1)

        if (-not $global:MonitoringData) {
            Write-Warning "Aucune donnée de monitoring disponible"
            return
        }

        $cutoff = (Get-Date).AddHours(-$LastHours)
        $recentData = $global:MonitoringData | Where-Object { $_.Timestamp -gt $cutoff }

        if ($recentData) {
            $stats = @{
                SampleCount = $recentData.Count
                AverageCPU = [math]::Round(($recentData | Measure-Object -Property CPU -Average).Average, 2)
                MaxCPU = [math]::Round(($recentData | Measure-Object -Property CPU -Maximum).Maximum, 2)
                AverageMemoryPercent = [math]::Round((($recentData | Measure-Object -Property MemoryUsed -Average).Average / $recentData[0].MemoryTotal) * 100, 2)
                TimeRange = "$LastHours heure(s)"
            }

            return [PSCustomObject]$stats
        }
    }

    'Show-SystemDashboard' = {
        $metrics = Get-SystemMetrics -LastHours 1

        if ($metrics) {
            Write-Host "=== TABLEAU DE BORD SYSTÈME ===" -ForegroundColor Cyan
            Write-Host "Échantillons (dernière heure): $($metrics.SampleCount)"
            Write-Host "CPU - Moyenne: $($metrics.AverageCPU)%, Max: $($metrics.MaxCPU)%"
            Write-Host "Mémoire - Utilisation moyenne: $($metrics.AverageMemoryPercent)%"
            Write-Host "Période: $($metrics.TimeRange)"
        } else {
            Write-Warning "Aucune métrique disponible"
        }
    }
}

New-DynamicModule -ModuleName "SystemMonitor" -Functions $monitoringFunctions -Description "Module de monitoring système dynamique"

# Utilisation du module créé
Start-SystemMonitor -IntervalSeconds 30
Start-Sleep -Seconds 120  # Attendre quelques métriques
Show-SystemDashboard
Stop-SystemMonitor
```

### 3.2 Manipulation avancée des objets .NET

**Interopérabilité .NET avancée et réflexion :**
```powershell
# Chargement dynamique d'assemblys
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing

# Création d'une interface graphique simple
function New-SimpleGUI {
    param([string]$Title = "PowerShell GUI")

    $form = New-Object System.Windows.Forms.Form
    $form.Text = $Title
    $form.Size = New-Object System.Drawing.Size(400, 300)
    $form.StartPosition = "CenterScreen"

    $label = New-Object System.Windows.Forms.Label
    $label.Location = New-Object System.Drawing.Point(10, 20)
    $label.Size = New-Object System.Drawing.Size(360, 20)
    $label.Text = "Bienvenue dans l'interface PowerShell!"
    $form.Controls.Add($label)

    $button = New-Object System.Windows.Forms.Button
    $button.Location = New-Object System.Drawing.Point(10, 60)
    $button.Size = New-Object System.Drawing.Size(100, 30)
    $button.Text = "Cliquez-moi!"
    $button.Add_Click({
        [System.Windows.Forms.MessageBox]::Show("Vous avez cliqué sur le bouton!", "Information")
    })
    $form.Controls.Add($button)

    $textBox = New-Object System.Windows.Forms.TextBox
    $textBox.Location = New-Object System.Drawing.Point(10, 100)
    $textBox.Size = New-Object System.Drawing.Size(360, 20)
    $form.Controls.Add($textBox)

    $resultButton = New-Object System.Windows.Forms.Button
    $resultButton.Location = New-Object System.Drawing.Point(10, 130)
    $resultButton.Size = New-Object System.Drawing.Size(150, 30)
    $resultButton.Text = "Afficher le texte"
    $resultButton.Add_Click({
        $text = $textBox.Text
        [System.Windows.Forms.MessageBox]::Show("Texte saisi: $text", "Résultat")
    })
    $form.Controls.Add($resultButton)

    return $form
}

# Utilisation de l'interface graphique
$gui = New-SimpleGUI -Title "Démonstration PowerShell GUI"
$gui.ShowDialog()

# Réflexion avancée sur les objets .NET
function Analyze-ObjectReflection {
    param([object]$Object)

    $analysis = @{
        Type = $Object.GetType().FullName
        Assembly = $Object.GetType().Assembly.FullName
        BaseType = $Object.GetType().BaseType
        IsClass = $Object.GetType().IsClass
        IsValueType = $Object.GetType().IsValueType
        IsEnum = $Object.GetType().IsEnum
        Properties = @()
        Methods = @()
        Fields = @()
        Interfaces = @()
    }

    # Analyse des propriétés
    $properties = $Object.GetType().GetProperties()
    foreach ($prop in $properties) {
        $analysis.Properties += @{
            Name = $prop.Name
            Type = $prop.PropertyType.FullName
            CanRead = $prop.CanRead
            CanWrite = $prop.CanWrite
            IsStatic = $prop.GetGetMethod().IsStatic
        }
    }

    # Analyse des méthodes
    $methods = $Object.GetType().GetMethods() | Where-Object { -not $_.IsSpecialName }
    foreach ($method in $methods) {
        $analysis.Methods += @{
            Name = $method.Name
            ReturnType = $method.ReturnType.FullName
            IsStatic = $method.IsStatic
            IsPublic = $method.IsPublic
            Parameters = $method.GetParameters() | ForEach-Object { $_.ParameterType.FullName }
        }
    }

    # Analyse des champs
    $fields = $Object.GetType().GetFields()
    foreach ($field in $fields) {
        $analysis.Fields += @{
            Name = $field.Name
            Type = $field.FieldType.FullName
            IsStatic = $field.IsStatic
            IsPublic = $field.IsPublic
        }
    }

    # Interfaces implémentées
    $analysis.Interfaces = $Object.GetType().GetInterfaces() | ForEach-Object { $_.FullName }

    return $analysis
}

# Analyse d'un objet PowerShell
$service = Get-Service -Name "WinRM"
$reflection = Analyze-ObjectReflection -Object $service

Write-Host "=== ANALYSE RÉFLEXION ==="
Write-Host "Type: $($reflection.Type)"
Write-Host "Assembly: $($reflection.Assembly)"
Write-Host "Propriétés: $($reflection.Properties.Count)"
Write-Host "Méthodes: $($reflection.Methods.Count)"
Write-Host "Champs: $($reflection.Fields.Count)"
Write-Host "Interfaces: $($reflection.Interfaces -join ', ')"

# Utilisation de la réflexion pour l'introspection dynamique
function Invoke-MethodDynamically {
    param(
        [Parameter(Mandatory=$true)]
        [object]$Object,

        [Parameter(Mandatory=$true)]
        [string]$MethodName,

        [object[]]$Arguments = @()
    )

    $method = $Object.GetType().GetMethod($MethodName)
    if ($method) {
        return $method.Invoke($Object, $Arguments)
    } else {
        throw "Méthode '$MethodName' introuvable sur l'objet $($Object.GetType().Name)"
    }
}

function Get-PropertyDynamically {
    param(
        [Parameter(Mandatory=$true)]
        [object]$Object,

        [Parameter(Mandatory=$true)]
        [string]$PropertyName
    )

    $property = $Object.GetType().GetProperty($PropertyName)
    if ($property) {
        return $property.GetValue($Object)
    } else {
        throw "Propriété '$PropertyName' introuvable sur l'objet $($Object.GetType().Name)"
    }
}

function Set-PropertyDynamically {
    param(
        [Parameter(Mandatory=$true)]
        [object]$Object,

        [Parameter(Mandatory=$true)]
        [string]$PropertyName,

        [Parameter(Mandatory=$true)]
        [object]$Value
    )

    $property = $Object.GetType().GetProperty($PropertyName)
    if ($property) {
        $property.SetValue($Object, $Value)
    } else {
        throw "Propriété '$PropertyName' introuvable sur l'objet $($Object.GetType().Name)"
    }
}

# Exemple d'utilisation de la réflexion dynamique
$datetime = Get-Date

# Accès dynamique aux propriétés
$currentHour = Get-PropertyDynamically -Object $datetime -PropertyName "Hour"
Write-Host "Heure actuelle (via réflexion): $currentHour"

# Appel dynamique de méthodes
$tomorrow = Invoke-MethodDynamically -Object $datetime -MethodName "AddDays" -Arguments @(1)
Write-Host "Demain (via réflexion): $tomorrow"

# Création d'objets via réflexion
$arrayListType = [System.Collections.ArrayList]
$arrayList = $arrayListType::new()

# Appel de méthodes sur l'objet créé
Invoke-MethodDynamically -Object $arrayList -MethodName "Add" -Arguments @("Premier élément")
Invoke-MethodDynamically -Object $arrayList -MethodName "Add" -Arguments @("Deuxième élément")

Write-Host "ArrayList créé dynamiquement: $($arrayList -join ', ')"

# Génération de code C# à la volée et compilation
function New-DynamicCSharpClass {
    param(
        [string]$ClassName,
        [string[]]$Properties
    )

    $csharpCode = @"
using System;

public class $ClassName
{
$($Properties | ForEach-Object { "    public string $_ { get; set; }" } | Out-String)
    
    public override string ToString()
    {
        return "$ClassName: $($Properties -join ', ')";
    }
}
"@

    # Ajout du type C# à la session PowerShell
    Add-Type -TypeDefinition $csharpCode -Language CSharp

    Write-Host "Classe C# '$ClassName' compilée dynamiquement"
}

# Création d'une classe C# dynamique
New-DynamicCSharpClass -ClassName "DynamicUser" -Properties @("Name", "Email", "Department")

# Utilisation de la classe créée
$user = New-Object DynamicUser
$user.Name = "John Doe"
$user.Email = "john.doe@company.com"
$user.Department = "IT"

Write-Host "Objet créé dynamiquement: $($user.ToString())"
```

## Conclusion : PowerShell comme langage de métaprogrammation

La métaprogrammation PowerShell ouvre des possibilités extraordinaires pour créer des abstractions de haut niveau, générer du code dynamique, et construire des DSL spécialisés. En combinant les capacités de script PowerShell avec l'écosystème .NET, il devient possible de créer des outils d'automatisation sophistiqués qui s'adaptent aux besoins spécifiques de chaque environnement.

Dans le prochain chapitre, nous explorerons les techniques de debugging et de profilage avancés pour optimiser les performances des scripts PowerShell.

---

**Exercice pratique :** Créez un système de métaprogrammation complet qui :
1. Analyse automatiquement le code PowerShell pour détecter les anti-patterns
2. Génère des tests unitaires basés sur l'analyse AST
3. Crée un DSL pour l'administration de bases de données
4. Implémente un système de plugins extensibles
5. Génère de la documentation automatique à partir du code

**Challenge avancé :** Développez un framework de métaprogrammation PowerShell qui :
- Analyse le code pour détecter les vulnérabilités de sécurité
- Génère automatiquement des remédiations pour les problèmes trouvés
- Crée des abstractions de haut niveau pour les tâches répétitives
- Implémente un système de cache intelligent pour les opérations coûteuses
- Fournit des métriques de performance en temps réel pour les scripts

**Réflexion :** Comment la métaprogrammation change-t-elle fondamentalement notre approche du scripting ? Les techniques avancées comme l'AST et la génération de code dynamique rendent-elles PowerShell plus proche des langages de programmation traditionnels, ou créent-elles une nouvelle catégorie d'outils ?
