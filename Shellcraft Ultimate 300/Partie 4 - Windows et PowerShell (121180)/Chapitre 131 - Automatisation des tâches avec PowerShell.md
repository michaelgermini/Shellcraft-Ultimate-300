# Chapitre 131 - Automatisation des tâches avec PowerShell

> "L'automatisation n'est pas seulement une optimisation technique, c'est une transformation culturelle qui libère l'intelligence humaine pour des tâches plus créatives et stratégiques." - Adapté de concepts DevOps

## Introduction : PowerShell comme automateur universel

PowerShell transcende son rôle d'interface de commande pour devenir un puissant moteur d'automatisation. En combinant la puissance de .NET, l'accès aux API Windows, et des capacités de scripting avancées, PowerShell permet d'automatiser virtuellement toutes les tâches d'administration système, de déploiement, et de gestion opérationnelle.

Dans ce chapitre, nous explorerons les techniques d'automatisation avancées, les scheduled tasks, les workflows, et l'intégration avec d'autres systèmes.

## Section 1 : Scheduled Tasks et automatisation temporelle

### 1.1 Scheduled Tasks avec PowerShell

**Création et gestion des tâches planifiées :**
```powershell
# Création d'une tâche planifiée simple
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-File C:\Scripts\Backup.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "02:00"
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount

Register-ScheduledTask -TaskName "DailyBackup" -Action $action -Trigger $trigger -Principal $principal -Description "Sauvegarde quotidienne des données"

# Tâche avec conditions et paramètres avancés
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-NoProfile -ExecutionPolicy Bypass -File C:\Scripts\Maintenance.ps1 -Environment Production"
$trigger = New-ScheduledTaskTrigger -Weekly -WeeksInterval 1 -DaysOfWeek Monday -At "03:00"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries $false -DontStopIfGoingOnBatteries $true -StartWhenAvailable $true
$principal = New-ScheduledTaskPrincipal -UserId "Administrator" -RunLevel Highest

Register-ScheduledTask -TaskName "WeeklyMaintenance" -Action $action -Trigger $trigger -Settings $settings -Principal $principal

# Tâche déclenchée par un événement
$eventTrigger = New-ScheduledTaskTrigger -AtStartup
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-Command Write-Host 'System started at: (Get-Date)'"
Register-ScheduledTask -TaskName "StartupNotification" -Action $action -Trigger $eventTrigger

# Gestion des tâches planifiées
Get-ScheduledTask | Where-Object {$_.State -eq "Ready"} | Format-Table TaskName, State, LastRunTime, NextRunTime

# Modification d'une tâche existante
$task = Get-ScheduledTask -TaskName "DailyBackup"
$task.Triggers[0].StartBoundary = "2023-12-01T02:00:00"
$task | Set-ScheduledTask

# Export/Import de tâches
Export-ScheduledTask -TaskName "DailyBackup" | Out-File "C:\Tasks\DailyBackup.xml"
Register-ScheduledTask -Xml (Get-Content "C:\Tasks\DailyBackup.xml" | Out-String) -TaskName "DailyBackup" -User "Administrator"

# Désactivation temporaire
Disable-ScheduledTask -TaskName "DailyBackup"

# Suppression
Unregister-ScheduledTask -TaskName "DailyBackup" -Confirm:$false
```

### 1.2 Automatisation conditionnelle et intelligente

**Tâches avec logique conditionnelle :**
```powershell
# Script de sauvegarde intelligente avec vérifications
param(
    [string]$Environment = "Production",
    [switch]$Force
)

# Fonction de vérification des prérequis
function Test-Prerequisites {
    $checks = @{}
    
    # Vérification de l'espace disque
    $disk = Get-WmiObject Win32_LogicalDisk -Filter "DeviceID='C:'"
    $freeSpaceGB = [math]::Round($disk.FreeSpace / 1GB, 2)
    $checks.DiskSpace = $freeSpaceGB -gt 10
    
    # Vérification des processus critiques
    $criticalProcesses = @("sqlserver", "iis")
    $checks.CriticalProcesses = $true
    foreach ($process in $criticalProcesses) {
        if (-not (Get-Process -Name $process -ErrorAction SilentlyContinue)) {
            $checks.CriticalProcesses = $false
            break
        }
    }
    
    # Vérification de la connectivité réseau
    $checks.NetworkConnectivity = Test-Connection -ComputerName "backup-server" -Count 1 -Quiet
    
    return $checks
}

# Fonction de sauvegarde principale
function Invoke-Backup {
    param([string]$Type = "Full")
    
    $timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
    $backupPath = "D:\Backups\$Environment\$timestamp"
    
    New-Item -ItemType Directory -Path $backupPath -Force
    
    Write-Host "Starting $Type backup for $Environment environment..." -ForegroundColor Green
    
    try {
        switch ($Type) {
            "Full" {
                # Sauvegarde complète
                $databases = Get-SqlDatabase -ServerInstance "localhost"
                foreach ($db in $databases) {
                    Backup-SqlDatabase -ServerInstance "localhost" -Database $db.Name -BackupFile "$backupPath\$($db.Name)_full.bak"
                }
                
                # Sauvegarde des fichiers
                robocopy "C:\Data" "$backupPath\Data" /MIR /R:3 /W:10 /LOG:"$backupPath\backup.log"
            }
            "Incremental" {
                # Sauvegarde incrémentielle (simplifiée)
                $lastBackup = Get-ChildItem "D:\Backups\$Environment\*" -Directory | Sort-Object LastWriteTime -Descending | Select-Object -First 1
                if ($lastBackup) {
                    robocopy "C:\Data" "$backupPath\Data" /MIR /R:3 /W:10 /LOG:"$backupPath\backup.log" /XC /XN /XO
                }
            }
        }
        
        Write-Host "Backup completed successfully" -ForegroundColor Green
        return $true
        
    } catch {
        Write-Error "Backup failed: $_"
        return $false
    }
}

# Fonction de notification
function Send-Notification {
    param(
        [string]$Subject,
        [string]$Body,
        [string]$Priority = "Normal"
    )
    
    # Envoi d'email (nécessite configuration SMTP)
    $smtpServer = "smtp.company.com"
    $from = "backup@company.com"
    $to = "admin@company.com"
    
    Send-MailMessage -From $from -To $to -Subject $Subject -Body $Body -SmtpServer $smtpServer -Priority $Priority
}

# Logique principale
$checks = Test-Prerequisites

Write-Host "Prerequisites check:" -ForegroundColor Yellow
$checks.GetEnumerator() | ForEach-Object {
    $status = if ($_.Value) { "✓" } else { "✗" }
    Write-Host "  $status $($_.Key): $($_.Value)" -ForegroundColor (if ($_.Value) { "Green" } else { "Red" })
}

$allChecksPassed = ($checks.Values | Where-Object { $_ -eq $false }).Count -eq 0

if ($allChecksPassed -or $Force) {
    $backupResult = Invoke-Backup -Type "Full"
    
    if ($backupResult) {
        Send-Notification -Subject "Backup Success: $Environment" -Body "The backup completed successfully at $(Get-Date)"
    } else {
        Send-Notification -Subject "Backup Failed: $Environment" -Body "The backup failed. Please check the logs." -Priority "High"
    }
} else {
    Write-Warning "Prerequisites not met. Skipping backup unless -Force is specified."
    Send-Notification -Subject "Backup Skipped: $Environment" -Body "Prerequisites check failed. Backup was skipped."
}
```

### 1.3 Workflows PowerShell (PowerShell Workflow)

**Création de workflows pour l'automatisation complexe :**
```powershell
# Définition d'un workflow pour le déploiement d'application
workflow Deploy-Application {
    param(
        [string]$ApplicationName,
        [string]$Version,
        [string[]]$TargetServers,
        [PSCredential]$Credential
    )
    
    # Séquence d'activités avec parallélisation
    parallel {
        # Préparation en parallèle sur tous les serveurs
        foreach -parallel ($server in $TargetServers) {
            InlineScript {
                $session = New-PSSession -ComputerName $using:server -Credential $using:Credential
                
                Invoke-Command -Session $session -ScriptBlock {
                    param($appName, $version)
                    
                    # Création des répertoires
                    New-Item -ItemType Directory -Path "C:\Applications\$appName\$version" -Force
                    
                    # Téléchargement de l'application
                    Invoke-WebRequest -Uri "https://artifacts.company.com/$appName/$version/app.zip" -OutFile "C:\Temp\app.zip"
                    
                    # Extraction
                    Expand-Archive -Path "C:\Temp\app.zip" -DestinationPath "C:\Applications\$appName\$version"
                    
                    Write-Host "Preparation completed on $env:COMPUTERNAME"
                    
                } -ArgumentList $using:ApplicationName, $using:Version
                
                Remove-PSSession -Session $session
            }
        }
    }
    
    # Validation séquentielle
    foreach ($server in $TargetServers) {
        $validationResult = InlineScript {
            $session = New-PSSession -ComputerName $using:server -Credential $using:Credential
            
            $result = Invoke-Command -Session $session -ScriptBlock {
                param($appName, $version)
                
                # Validation de l'installation
                $appPath = "C:\Applications\$appName\$version"
                $configFile = "$appPath\config.json"
                
                $validation = @{
                    PathExists = Test-Path $appPath
                    ConfigExists = Test-Path $configFile
                    ServiceStopped = (Get-Service -Name $appName -ErrorAction SilentlyContinue).Status -eq "Stopped"
                }
                
                return $validation
                
            } -ArgumentList $using:ApplicationName, $using:Version
            
            Remove-PSSession -Session $session
            return $result
        }
        
        if (-not ($validationResult.PathExists -and $validationResult.ConfigExists)) {
            throw "Validation failed on $server"
        }
        
        Write-Host "Validation passed on $server"
    }
    
    # Activation en séquence (pour éviter la surcharge)
    foreach ($server in $TargetServers) {
        InlineScript {
            $session = New-PSSession -ComputerName $using:server -Credential $using:Credential
            
            Invoke-Command -Session $session -ScriptBlock {
                param($appName, $version)
                
                # Configuration du service
                $serviceName = $appName
                $binaryPath = "C:\Applications\$appName\$version\$appName.exe"
                
                if (Get-Service -Name $serviceName -ErrorAction SilentlyContinue) {
                    Stop-Service -Name $serviceName
                    sc.exe delete $serviceName
                }
                
                # Création du service
                New-Service -Name $serviceName -BinaryPathName $binaryPath -DisplayName "$appName v$version" -StartupType Automatic
                
                # Démarrage
                Start-Service -Name $serviceName
                
                Write-Host "Service started on $env:COMPUTERNAME"
                
            } -ArgumentList $using:ApplicationName, $using:Version
            
            Remove-PSSession -Session $session
        }
        
        # Pause entre les activations
        Start-Sleep -Seconds 30
    }
    
    # Test final
    parallel {
        foreach -parallel ($server in $TargetServers) {
            InlineScript {
                $testResult = Invoke-WebRequest -Uri "http://$using:server/health" -TimeoutSec 10 -ErrorAction SilentlyContinue
                return @{
                    Server = $using:server
                    StatusCode = $testResult.StatusCode
                    Success = $testResult.StatusCode -eq 200
                }
            }
        }
    }
}

# Utilisation du workflow
$credential = Get-Credential
$servers = @("web01", "web02", "web03")

Deploy-Application -ApplicationName "MyApp" -Version "2.1.0" -TargetServers $servers -Credential $credential
```

## Section 2 : Automatisation des processus métier

### 2.1 Orchestration de tâches complexes

**Création d'un orchestrateur de tâches :**
```powershell
# Orchestrateur de tâches avec dépendances
class TaskOrchestrator {
    [hashtable]$Tasks = @{}
    [System.Collections.Generic.List[string]]$ExecutionOrder = @()
    [hashtable]$Results = @{}
    
    [void]AddTask([string]$taskName, [scriptblock]$action, [string[]]$dependencies = @()) {
        $this.Tasks[$taskName] = @{
            Action = $action
            Dependencies = $dependencies
            State = "Pending"
            Result = $null
        }
    }
    
    [bool]ExecuteTask([string]$taskName) {
        $task = $this.Tasks[$taskName]
        
        if ($task.State -eq "Completed") {
            return $true
        }
        
        # Vérification des dépendances
        foreach ($dep in $task.Dependencies) {
            if (-not $this.ExecuteTask($dep)) {
                Write-Error "Dependency $dep failed for task $taskName"
                $task.State = "Failed"
                return $false
            }
        }
        
        try {
            Write-Host "Executing task: $taskName" -ForegroundColor Yellow
            $task.State = "Running"
            
            # Exécution de l'action
            $result = & $task.Action
            $task.Result = $result
            $task.State = "Completed"
            
            Write-Host "Task $taskName completed successfully" -ForegroundColor Green
            return $true
            
        } catch {
            Write-Error "Task $taskName failed: $_"
            $task.State = "Failed"
            return $false
        }
    }
    
    [void]ExecuteAll() {
        $this.ExecutionOrder = $this.GetExecutionOrder()
        
        foreach ($taskName in $this.ExecutionOrder) {
            if (-not $this.ExecuteTask($taskName)) {
                Write-Error "Execution stopped due to task failure: $taskName"
                break
            }
        }
    }
    
    [string[]]GetExecutionOrder() {
        # Algorithme de tri topologique simple
        $visited = @{}
        $order = @()
        
        function Visit-Task {
            param([string]$taskName)
            
            if ($visited.ContainsKey($taskName)) {
                return
            }
            
            $visited[$taskName] = $true
            
            foreach ($dep in $this.Tasks[$taskName].Dependencies) {
                Visit-Task $dep
            }
            
            $order += $taskName
        }
        
        foreach ($taskName in $this.Tasks.Keys) {
            Visit-Task $taskName
        }
        
        return $order
    }
    
    [hashtable]GetResults() {
        $results = @{}
        foreach ($taskName in $this.Tasks.Keys) {
            $results[$taskName] = @{
                State = $this.Tasks[$taskName].State
                Result = $this.Tasks[$taskName].Result
            }
        }
        return $results
    }
}

# Utilisation de l'orchestrateur pour un déploiement
$orchestrator = [TaskOrchestrator]::new()

# Définition des tâches avec dépendances
$orchestrator.AddTask("CheckPrerequisites", {
    # Vérification des prérequis
    $checks = @{
        DiskSpace = (Get-WmiObject Win32_LogicalDisk -Filter "DeviceID='C:'").FreeSpace / 1GB -gt 5
        NetworkConnectivity = Test-Connection -ComputerName "artifact-repo" -Count 1 -Quiet
        AdminRights = ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    }
    
    $allPassed = ($checks.Values | Where-Object { $_ -eq $false }).Count -eq 0
    
    if (-not $allPassed) {
        throw "Prerequisites check failed: $($checks | ConvertTo-Json)"
    }
    
    return $checks
})

$orchestrator.AddTask("DownloadArtifacts", {
    param($version = "latest")
    
    $artifactUrl = "https://artifacts.company.com/myapp/$version/app.zip"
    $localPath = "C:\Temp\app.zip"
    
    Invoke-WebRequest -Uri $artifactUrl -OutFile $localPath
    
    # Vérification de l'intégrité
    $hash = Get-FileHash $localPath -Algorithm SHA256
    Write-Host "Downloaded artifact SHA256: $($hash.Hash)"
    
    return @{
        Path = $localPath
        Hash = $hash.Hash
        Size = (Get-Item $localPath).Length
    }
}, @("CheckPrerequisites"))

$orchestrator.AddTask("BackupCurrentVersion", {
    $backupPath = "C:\Backup\$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss')"
    
    if (Test-Path "C:\Applications\MyApp") {
        Copy-Item "C:\Applications\MyApp" $backupPath -Recurse
        Write-Host "Backup created at: $backupPath"
        
        return @{
            Path = $backupPath
            Size = (Get-ChildItem $backupPath -Recurse | Measure-Object -Property Length -Sum).Sum
        }
    } else {
        Write-Host "No existing installation found, skipping backup"
        return @{ Skipped = $true }
    }
}, @("CheckPrerequisites"))

$orchestrator.AddTask("StopServices", {
    $services = @("MyApp", "MyApp-Dependencies")
    
    foreach ($service in $services) {
        if (Get-Service -Name $service -ErrorAction SilentlyContinue) {
            Stop-Service -Name $service
            Write-Host "Service $service stopped"
        }
    }
    
    return @{ StoppedServices = $services }
}, @("BackupCurrentVersion"))

$orchestrator.AddTask("DeployApplication", {
    param($artifact)
    
    $deployPath = "C:\Applications\MyApp"
    
    # Création du répertoire de déploiement
    if (Test-Path $deployPath) {
        Remove-Item $deployPath -Recurse -Force
    }
    
    New-Item -ItemType Directory -Path $deployPath -Force
    
    # Extraction de l'artefact
    Expand-Archive -Path $artifact.Path -DestinationPath $deployPath
    
    # Configuration
    $config = Get-Content "$deployPath\config.template.json" | ConvertFrom-Json
    $config.database.host = $env:DATABASE_HOST
    $config.database.password = $env:DATABASE_PASSWORD
    $config | ConvertTo-Json | Set-Content "$deployPath\config.json"
    
    Write-Host "Application deployed to: $deployPath"
    
    return @{
        DeployPath = $deployPath
        Configured = $true
        Version = (Get-Content "$deployPath\version.txt")
    }
}, @("DownloadArtifacts", "StopServices"))

$orchestrator.AddTask("StartServices", {
    $services = @("MyApp-Dependencies", "MyApp")
    
    foreach ($service in $services) {
        if (Get-Service -Name $service -ErrorAction SilentlyContinue) {
            Start-Service -Name $service
            Write-Host "Service $service started"
        }
    }
    
    return @{ StartedServices = $services }
}, @("DeployApplication"))

$orchestrator.AddTask("RunTests", {
    # Tests post-déploiement
    $testResults = @()
    
    # Test de santé
    try {
        $response = Invoke-WebRequest -Uri "http://localhost:8080/health" -TimeoutSec 30
        $testResults += @{
            Test = "HealthCheck"
            Result = $response.StatusCode -eq 200
            Details = "Status: $($response.StatusCode)"
        }
    } catch {
        $testResults += @{
            Test = "HealthCheck"
            Result = $false
            Details = "Error: $_"
        }
    }
    
    # Test de connectivité base de données
    try {
        $connection = New-Object System.Data.SqlClient.SqlConnection
        $connection.ConnectionString = "Server=localhost;Database=MyApp;Integrated Security=True;"
        $connection.Open()
        $connection.Close()
        $testResults += @{
            Test = "DatabaseConnectivity"
            Result = $true
            Details = "Connection successful"
        }
    } catch {
        $testResults += @{
            Test = "DatabaseConnectivity"
            Result = $false
            Details = "Error: $_"
        }
    }
    
    $passedTests = ($testResults | Where-Object { $_.Result -eq $true }).Count
    $totalTests = $testResults.Count
    
    Write-Host "Tests completed: $passedTests/$totalTests passed"
    
    if ($passedTests -lt $totalTests) {
        throw "Some tests failed: $($testResults | Where-Object { $_.Result -eq $false } | ConvertTo-Json)"
    }
    
    return @{
        Results = $testResults
        Passed = $passedTests
        Total = $totalTests
    }
}, @("StartServices"))

# Exécution de l'orchestration
$orchestrator.ExecuteAll()

# Affichage des résultats
$results = $orchestrator.GetResults()
Write-Host "`n=== EXECUTION RESULTS ===" -ForegroundColor Cyan
foreach ($taskName in $orchestrator.ExecutionOrder) {
    $result = $results[$taskName]
    $color = switch ($result.State) {
        "Completed" { "Green" }
        "Failed" { "Red" }
        default { "Yellow" }
    }
    Write-Host "$taskName : $($result.State)" -ForegroundColor $color
}
```

### 2.2 Automatisation des rapports et notifications

**Système de reporting automatisé :**
```powershell
# Module de reporting automatisé
class AutomatedReporter {
    [string]$ReportPath
    [hashtable]$ReportData
    [string]$SMTPServer
    [string[]]$Recipients
    
    AutomatedReporter([string]$reportPath, [string]$smtpServer, [string[]]$recipients) {
        $this.ReportPath = $reportPath
        $this.ReportData = @{}
        $this.SMTPServer = $smtpServer
        $this.Recipients = $recipients
        
        # Création du répertoire de rapports
        New-Item -ItemType Directory -Path $this.ReportPath -Force | Out-Null
    }
    
    [void]CollectSystemMetrics() {
        Write-Host "Collecting system metrics..."
        
        $cpu = Get-WmiObject Win32_Processor | Measure-Object -Property LoadPercentage -Average
        $memory = Get-WmiObject Win32_OperatingSystem
        $disk = Get-WmiObject Win32_LogicalDisk -Filter "DeviceID='C:'"
        
        $this.ReportData.SystemMetrics = @{
            Timestamp = Get-Date
            CPU_Usage_Percent = $cpu.Average
            Memory_Usage_Percent = [math]::Round(($memory.TotalVisibleMemorySize - $memory.FreePhysicalMemory) / $memory.TotalVisibleMemorySize * 100, 2)
            Disk_Usage_Percent = [math]::Round(($disk.Size - $disk.FreeSpace) / $disk.Size * 100, 2)
            Uptime_Days = [math]::Round((Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime).TotalDays, 1)
        }
    }
    
    [void]CollectApplicationMetrics() {
        Write-Host "Collecting application metrics..."
        
        $apps = @("MyApp", "Database", "WebServer")
        $appMetrics = @{}
        
        foreach ($app in $apps) {
            $service = Get-Service -Name $app -ErrorAction SilentlyContinue
            $process = Get-Process -Name $app -ErrorAction SilentlyContinue
            
            $appMetrics[$app] = @{
                Service_Status = if ($service) { $service.Status } else { "NotFound" }
                Process_Count = if ($process) { $process.Count } else { 0 }
                Memory_MB = if ($process) { [math]::Round(($process | Measure-Object -Property WorkingSet -Sum).Sum / 1MB, 2) } else { 0 }
                CPU_Percent = if ($process) { [math]::Round(($process | Measure-Object -Property CPU -Sum).Sum, 2) } else { 0 }
            }
        }
        
        $this.ReportData.ApplicationMetrics = $appMetrics
    }
    
    [void]CollectSecurityMetrics() {
        Write-Host "Collecting security metrics..."
        
        # Événements de sécurité récents
        $securityEvents = Get-WinEvent -LogName Security -MaxEvents 100 -ErrorAction SilentlyContinue | 
            Where-Object { $_.TimeCreated -gt (Get-Date).AddHours(-24) }
        
        $failedLogins = ($securityEvents | Where-Object { $_.Id -eq 4625 }).Count
        $successfulLogins = ($securityEvents | Where-Object { $_.Id -eq 4624 }).Count
        
        # Mises à jour de sécurité
        $updates = Get-HotFix | Where-Object { $_.Description -like "*Security*" -and $_.InstalledOn -gt (Get-Date).AddDays(-30) }
        
        $this.ReportData.SecurityMetrics = @{
            Failed_Logins_24h = $failedLogins
            Successful_Logins_24h = $successfulLogins
            Security_Updates_30d = $updates.Count
            Last_Security_Update = if ($updates) { ($updates | Sort-Object InstalledOn -Descending | Select-Object -First 1).InstalledOn } else { $null }
        }
    }
    
    [void]CollectBusinessMetrics() {
        Write-Host "Collecting business metrics..."
        
        # Métriques métier (exemples)
        # Dans un vrai scénario, ces données viendraient d'une base de données métier
        
        $this.ReportData.BusinessMetrics = @{
            Active_Users = Get-Random -Minimum 1000 -Maximum 1500
            Transactions_24h = Get-Random -Minimum 5000 -Maximum 8000
            Revenue_24h = Get-Random -Minimum 50000 -Maximum 100000
            Error_Rate_Percent = [math]::Round((Get-Random -Minimum 0.1 -Maximum 2.0), 2)
        }
    }
    
    [void]GenerateHTMLReport() {
        Write-Host "Generating HTML report..."
        
        $timestamp = Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
        $reportFile = Join-Path $this.ReportPath "system_report_$timestamp.html"
        
        $html = @"
<!DOCTYPE html>
<html>
<head>
    <title>System Report - $(Get-Date -Format "yyyy-MM-dd")</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .section { margin-bottom: 30px; border: 1px solid #ddd; padding: 15px; border-radius: 5px; }
        .metric { display: flex; justify-content: space-between; margin: 5px 0; }
        .value { font-weight: bold; }
        .good { color: green; }
        .warning { color: orange; }
        .critical { color: red; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <h1>System Report - $(Get-Date -Format "yyyy-MM-dd HH:mm:ss")</h1>
    
    <div class="section">
        <h2>System Metrics</h2>
        <div class="metric">
            <span>CPU Usage:</span>
            <span class="value $(if ($this.ReportData.SystemMetrics.CPU_Usage_Percent -gt 80) { "critical" } elseif ($this.ReportData.SystemMetrics.CPU_Usage_Percent -gt 60) { "warning" } else { "good" })">
                $($this.ReportData.SystemMetrics.CPU_Usage_Percent)%
            </span>
        </div>
        <div class="metric">
            <span>Memory Usage:</span>
            <span class="value $(if ($this.ReportData.SystemMetrics.Memory_Usage_Percent -gt 90) { "critical" } elseif ($this.ReportData.SystemMetrics.Memory_Usage_Percent -gt 75) { "warning" } else { "good" })">
                $($this.ReportData.SystemMetrics.Memory_Usage_Percent)%
            </span>
        </div>
        <div class="metric">
            <span>Disk Usage:</span>
            <span class="value $(if ($this.ReportData.SystemMetrics.Disk_Usage_Percent -gt 95) { "critical" } elseif ($this.ReportData.SystemMetrics.Disk_Usage_Percent -gt 85) { "warning" } else { "good" })">
                $($this.ReportData.SystemMetrics.Disk_Usage_Percent)%
            </span>
        </div>
        <div class="metric">
            <span>System Uptime:</span>
            <span class="value">$($this.ReportData.SystemMetrics.Uptime_Days) days</span>
        </div>
    </div>
    
    <div class="section">
        <h2>Application Status</h2>
        <table>
            <tr><th>Application</th><th>Service Status</th><th>Process Count</th><th>Memory (MB)</th><th>CPU (%)</th></tr>
"@

        foreach ($app in $this.ReportData.ApplicationMetrics.Keys) {
            $metrics = $this.ReportData.ApplicationMetrics[$app]
            $statusClass = if ($metrics.Service_Status -eq "Running") { "good" } else { "critical" }
            
            $html += "<tr><td>$app</td><td class='$statusClass'>$($metrics.Service_Status)</td><td>$($metrics.Process_Count)</td><td>$($metrics.Memory_MB)</td><td>$($metrics.CPU_Percent)</td></tr>"
        }

        $html += @"
        </table>
    </div>
    
    <div class="section">
        <h2>Security Overview</h2>
        <div class="metric">
            <span>Failed Logins (24h):</span>
            <span class="value $(if ($this.ReportData.SecurityMetrics.Failed_Logins_24h -gt 10) { "critical" } elseif ($this.ReportData.SecurityMetrics.Failed_Logins_24h -gt 5) { "warning" } else { "good" })">
                $($this.ReportData.SecurityMetrics.Failed_Logins_24h)
            </span>
        </div>
        <div class="metric">
            <span>Successful Logins (24h):</span>
            <span class="value">$($this.ReportData.SecurityMetrics.Successful_Logins_24h)</span>
        </div>
        <div class="metric">
            <span>Security Updates (30d):</span>
            <span class="value">$($this.ReportData.SecurityMetrics.Security_Updates_30d)</span>
        </div>
    </div>
    
    <div class="section">
        <h2>Business Metrics</h2>
        <div class="metric">
            <span>Active Users:</span>
            <span class="value">$($this.ReportData.BusinessMetrics.Active_Users)</span>
        </div>
        <div class="metric">
            <span>Transactions (24h):</span>
            <span class="value">$($this.ReportData.BusinessMetrics.Transactions_24h)</span>
        </div>
        <div class="metric">
            <span>Revenue (24h):</span>
            <span class="value">`$$($this.ReportData.BusinessMetrics.Revenue_24h.ToString("N0"))</span>
        </div>
        <div class="metric">
            <span>Error Rate:</span>
            <span class="value $(if ($this.ReportData.BusinessMetrics.Error_Rate_Percent -gt 1.0) { "critical" } elseif ($this.ReportData.BusinessMetrics.Error_Rate_Percent -gt 0.5) { "warning" } else { "good" })">
                $($this.ReportData.BusinessMetrics.Error_Rate_Percent)%
            </span>
        </div>
    </div>
</body>
</html>
"@

        $html | Out-File -FilePath $reportFile -Encoding UTF8
        Write-Host "Report generated: $reportFile"
        
        $this.ReportData.ReportFile = $reportFile
    }
    
    [void]SendReport() {
        Write-Host "Sending report via email..."
        
        $timestamp = Get-Date -Format "yyyy-MM-dd"
        $subject = "System Report - $timestamp"
        
        $body = @"
Daily System Report

System Health Summary:
- CPU Usage: $($this.ReportData.SystemMetrics.CPU_Usage_Percent)%
- Memory Usage: $($this.ReportData.SystemMetrics.Memory_Usage_Percent)%
- Disk Usage: $($this.ReportData.SystemMetrics.Disk_Usage_Percent)%

Security Events (24h):
- Failed Logins: $($this.ReportData.SecurityMetrics.Failed_Logins_24h)
- Successful Logins: $($this.ReportData.SecurityMetrics.Successful_Logins_24h)

Business Metrics:
- Active Users: $($this.ReportData.BusinessMetrics.Active_Users)
- Transactions: $($this.ReportData.BusinessMetrics.Transactions_24h)
- Revenue: `$$($this.ReportData.BusinessMetrics.Revenue_24h.ToString("N0"))

Full report attached.

Generated at: $(Get-Date -Format "yyyy-MM-dd HH:mm:ss")
"@

        # Envoi de l'email avec pièce jointe
        Send-MailMessage -From "reports@company.com" -To $this.Recipients -Subject $subject -Body $body -SmtpServer $this.SMTPServer -Attachments $this.ReportData.ReportFile
        
        Write-Host "Report sent to: $($this.Recipients -join ', ')"
    }
    
    [void]RunDailyReport() {
        Write-Host "=== DAILY SYSTEM REPORT ===" -ForegroundColor Cyan
        
        $this.CollectSystemMetrics()
        $this.CollectApplicationMetrics()
        $this.CollectSecurityMetrics()
        $this.CollectBusinessMetrics()
        $this.GenerateHTMLReport()
        $this.SendReport()
        
        Write-Host "Daily report completed" -ForegroundColor Green
    }
}

# Utilisation du système de reporting
$reporter = [AutomatedReporter]::new(
    "C:\Reports",
    "smtp.company.com",
    @("admin@company.com", "ops@company.com")
)

# Exécution du rapport quotidien
$reporter.RunDailyReport()

# Programmation via Scheduled Task
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-Command & { . C:\Scripts\AutomatedReporter.ps1; `$reporter.RunDailyReport() }"
$trigger = New-ScheduledTaskTrigger -Daily -At "08:00"
Register-ScheduledTask -TaskName "DailySystemReport" -Action $action -Trigger $trigger -Principal (New-ScheduledTaskPrincipal -UserId "SYSTEM" -LogonType ServiceAccount)
```

## Section 3 : Intégration avec d'autres systèmes

### 3.1 Automatisation avec APIs REST

**Client PowerShell pour APIs :**
```powershell
# Module PowerShell pour l'intégration API
class APIClient {
    [string]$BaseUrl
    [string]$ApiKey
    [hashtable]$DefaultHeaders
    [int]$TimeoutSeconds = 30
    
    APIClient([string]$baseUrl, [string]$apiKey = "") {
        $this.BaseUrl = $baseUrl.TrimEnd('/')
        $this.ApiKey = $apiKey
        $this.DefaultHeaders = @{
            "Content-Type" = "application/json"
            "User-Agent" = "PowerShell-API-Client/1.0"
        }
        
        if ($apiKey) {
            $this.DefaultHeaders["Authorization"] = "Bearer $apiKey"
        }
    }
    
    [PSCustomObject]InvokeAPI([string]$method, [string]$endpoint, [hashtable]$body = $null, [hashtable]$queryParams = $null) {
        $url = $this.BaseUrl + $endpoint
        
        # Ajout des paramètres de requête
        if ($queryParams -and $queryParams.Count -gt 0) {
            $queryString = ($queryParams.GetEnumerator() | ForEach-Object { "$($_.Key)=$( [System.Web.HttpUtility]::UrlEncode($_.Value) )" }) -join "&"
            $url += "?$queryString"
        }
        
        $params = @{
            Uri = $url
            Method = $method
            Headers = $this.DefaultHeaders
            TimeoutSec = $this.TimeoutSeconds
            UseBasicParsing = $true
        }
        
        # Ajout du corps de la requête
        if ($body) {
            $params.Body = $body | ConvertTo-Json -Depth 10
        }
        
        try {
            $response = Invoke-WebRequest @params
            
            return [PSCustomObject]@{
                Success = $true
                StatusCode = $response.StatusCode
                Headers = $response.Headers
                Content = if ($response.Content) { $response.Content | ConvertFrom-Json } else { $null }
                RawContent = $response.Content
            }
            
        } catch {
            $webException = $_.Exception
            
            return [PSCustomObject]@{
                Success = $false
                StatusCode = $webException.Response.StatusCode.value__
                ErrorMessage = $webException.Message
                Content = if ($webException.Response) { 
                    try { 
                        $webException.Response.GetResponseStream() | ConvertFrom-Json 
                    } catch { 
                        $null 
                    }
                } else { 
                    $null 
                }
            }
        }
    }
    
    [PSCustomObject]Get([string]$endpoint, [hashtable]$queryParams = $null) {
        return $this.InvokeAPI("GET", $endpoint, $null, $queryParams)
    }
    
    [PSCustomObject]Post([string]$endpoint, [hashtable]$body = $null) {
        return $this.InvokeAPI("POST", $endpoint, $body)
    }
    
    [PSCustomObject]Put([string]$endpoint, [hashtable]$body = $null) {
        return $this.InvokeAPI("PUT", $endpoint, $body)
    }
    
    [PSCustomObject]Delete([string]$endpoint) {
        return $this.InvokeAPI("DELETE", $endpoint)
    }
}

# Automatisation avec l'API GitHub
class GitHubAutomation {
    [APIClient]$Client
    
    GitHubAutomation([string]$token) {
        $this.Client = [APIClient]::new("https://api.github.com", $token)
        $this.Client.DefaultHeaders["Accept"] = "application/vnd.github.v3+json"
    }
    
    [PSCustomObject]GetRepository([string]$owner, [string]$repo) {
        return $this.Client.Get("/repos/$owner/$repo")
    }
    
    [PSCustomObject]CreateIssue([string]$owner, [string]$repo, [string]$title, [string]$body, [string[]]$labels = @()) {
        $issueData = @{
            title = $title
            body = $body
        }
        
        if ($labels) {
            $issueData.labels = $labels
        }
        
        return $this.Client.Post("/repos/$owner/$repo/issues", $issueData)
    }
    
    [PSCustomObject]CreatePullRequest([string]$owner, [string]$repo, [string]$title, [string]$head, [string]$base, [string]$body) {
        $prData = @{
            title = $title
            head = $head
            base = $base
            body = $body
        }
        
        return $this.Client.Post("/repos/$owner/$repo/pulls", $prData)
    }
    
    [PSCustomObject]GetWorkflowRuns([string]$owner, [string]$repo, [string]$workflowName) {
        $queryParams = @{
            workflow = $workflowName
            per_page = 10
        }
        
        return $this.Client.Get("/repos/$owner/$repo/actions/runs", $queryParams)
    }
}

# Automatisation avec l'API Azure DevOps
class AzureDevOpsAutomation {
    [APIClient]$Client
    [string]$Organization
    [string]$Project
    
    AzureDevOpsAutomation([string]$organization, [string]$project, [string]$pat) {
        $this.Client = [APIClient]::new("https://dev.azure.com/$organization/$project", "")
        $auth = [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$pat"))
        $this.Client.DefaultHeaders["Authorization"] = "Basic $auth"
        $this.Client.DefaultHeaders["Accept"] = "application/json;api-version=6.0"
        
        $this.Organization = $organization
        $this.Project = $project
    }
    
    [PSCustomObject]CreateWorkItem([string]$type, [string]$title, [string]$description) {
        $workItemData = @(
            @{
                op = "add"
                path = "/fields/System.Title"
                value = $title
            },
            @{
                op = "add"
                path = "/fields/System.Description"
                value = $description
            }
        )
        
        return $this.Client.Post("/_apis/wit/workitems/`$$type", $workItemData)
    }
    
    [PSCustomObject]TriggerBuild([int]$buildDefinitionId, [hashtable]$parameters = $null) {
        $buildData = @{
            definition = @{
                id = $buildDefinitionId
            }
        }
        
        if ($parameters) {
            $buildData.parameters = $parameters | ConvertTo-Json -Compress
        }
        
        return $this.Client.Post("/_apis/build/builds", $buildData)
    }
    
    [PSCustomObject]GetBuildStatus([int]$buildId) {
        return $this.Client.Get("/_apis/build/builds/$buildId")
    }
}

# Exemple d'automatisation intégrée
function Invoke-ReleaseAutomation {
    param(
        [string]$GitHubToken,
        [string]$AzureDevOpsPAT,
        [string]$Organization,
        [string]$Project,
        [string]$Repository
    )
    
    Write-Host "Starting release automation..." -ForegroundColor Green
    
    # Initialisation des clients API
    $github = [GitHubAutomation]::new($GitHubToken)
    $ado = [AzureDevOpsAutomation]::new($Organization, $Project, $AzureDevOpsPAT)
    
    try {
        # 1. Vérification de l'état du repository
        Write-Host "Checking repository status..."
        $repoStatus = $github.GetRepository($Organization, $Repository)
        
        if (-not $repoStatus.Success) {
            throw "Failed to get repository status: $($repoStatus.ErrorMessage)"
        }
        
        Write-Host "Repository: $($repoStatus.Content.full_name)"
        Write-Host "Default branch: $($repoStatus.Content.default_branch)"
        
        # 2. Création d'un tag de release
        $version = "v2.1.0"
        $tagTitle = "Release $version"
        
        Write-Host "Creating release tag..."
        # Simulation de création de tag (nécessiterait git operations)
        Write-Host "Tag $version created"
        
        # 3. Déclenchement du build de release
        Write-Host "Triggering release build..."
        $buildResult = $ado.TriggerBuild(42, @{ Version = $version })
        
        if (-not $buildResult.Success) {
            throw "Failed to trigger build: $($buildResult.ErrorMessage)"
        }
        
        $buildId = $buildResult.Content.id
        Write-Host "Build triggered with ID: $buildId"
        
        # 4. Monitoring du build
        Write-Host "Monitoring build progress..."
        do {
            Start-Sleep -Seconds 30
            $buildStatus = $ado.GetBuildStatus($buildId)
            
            if (-not $buildStatus.Success) {
                throw "Failed to get build status: $($buildStatus.ErrorMessage)"
            }
            
            $status = $buildStatus.Content.status
            Write-Host "Build status: $status"
            
        } while ($status -notin @("completed", "cancelled", "failed"))
        
        if ($status -ne "completed") {
            throw "Build failed with status: $status"
        }
        
        Write-Host "Build completed successfully" -ForegroundColor Green
        
        # 5. Création de la release sur GitHub
        Write-Host "Creating GitHub release..."
        $releaseBody = @"
## Release $version

### Changes
- Automated release process
- Improved performance
- Bug fixes

### Build Information
- Build ID: $buildId
- Build Status: $status
- Repository: $($repoStatus.Content.full_name)

Generated automatically by PowerShell automation.
"@
        
        $releaseResult = $github.CreatePullRequest(
            $Organization,
            $Repository,
            $tagTitle,
            "release/$version",
            $repoStatus.Content.default_branch,
            $releaseBody
        )
        
        if ($releaseResult.Success) {
            Write-Host "Release created successfully" -ForegroundColor Green
        } else {
            Write-Warning "Failed to create release: $($releaseResult.ErrorMessage)"
        }
        
        # 6. Notification des équipes
        Write-Host "Sending notifications..."
        $notificationBody = "Release $version has been successfully deployed. Check the release notes for details."
        
        # Ici, on pourrait envoyer des emails, Slack, Teams, etc.
        Write-Host "Notifications sent"
        
        Write-Host "Release automation completed successfully!" -ForegroundColor Green
        
    } catch {
        Write-Error "Release automation failed: $_"
        
        # Création d'un ticket d'incident
        $issueResult = $github.CreateIssue(
            $Organization,
            $Repository,
            "Release Automation Failed",
            "Release automation failed with error: $_",
            @("bug", "automation", "urgent")
        )
        
        if ($issueResult.Success) {
            Write-Host "Incident ticket created: $($issueResult.Content.html_url)"
        }
        
        throw
    }
}

# Utilisation de l'automatisation
$GitHubToken = $env:GITHUB_TOKEN
$AzureDevOpsPAT = $env:AZURE_DEVOPS_PAT

Invoke-ReleaseAutomation -GitHubToken $GitHubToken -AzureDevOpsPAT $AzureDevOpsPAT -Organization "myorg" -Project "myproject" -Repository "myrepo"
```

### 3.2 Automatisation des bases de données

**Gestion automatisée des bases de données :**
```powershell
# Module d'automatisation des bases de données
class DatabaseAutomation {
    [string]$Server
    [string]$Database
    [PSCredential]$Credential
    
    DatabaseAutomation([string]$server, [string]$database, [PSCredential]$credential) {
        $this.Server = $server
        $this.Database = $database
        $this.Credential = $credential
    }
    
    [PSCustomObject]ExecuteQuery([string]$query) {
        try {
            $connectionString = "Server=$($this.Server);Database=$($this.Database);Integrated Security=True;"
            $connection = New-Object System.Data.SqlClient.SqlConnection
            $connection.ConnectionString = $connectionString
            
            $command = $connection.CreateCommand()
            $command.CommandText = $query
            
            $connection.Open()
            
            if ($query.Trim().ToUpper().StartsWith("SELECT")) {
                $adapter = New-Object System.Data.SqlClient.SqlDataAdapter $command
                $dataset = New-Object System.Data.DataSet
                $adapter.Fill($dataset)
                $result = $dataset.Tables[0]
            } else {
                $rowsAffected = $command.ExecuteNonQuery()
                $result = @{ RowsAffected = $rowsAffected }
            }
            
            $connection.Close()
            
            return [PSCustomObject]@{
                Success = $true
                Result = $result
            }
            
        } catch {
            return [PSCustomObject]@{
                Success = $false
                Error = $_.Exception.Message
            }
        }
    }
    
    [void]BackupDatabase([string]$backupPath) {
        Write-Host "Starting database backup..."
        
        $timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
        $backupFile = Join-Path $backupPath "backup_$($this.Database)_$timestamp.bak"
        
        $query = "BACKUP DATABASE [$($this.Database)] TO DISK = '$backupFile' WITH FORMAT, MEDIANAME = 'SQLServerBackups', NAME = 'Full Backup of $($this.Database)';"
        
        $result = $this.ExecuteQuery($query)
        
        if ($result.Success) {
            Write-Host "Database backup completed: $backupFile"
            
            # Compression du backup
            $compressedFile = $backupFile -replace '\.bak$', '.zip'
            Compress-Archive -Path $backupFile -DestinationPath $compressedFile
            Remove-Item $backupFile
            
            Write-Host "Backup compressed: $compressedFile"
        } else {
            throw "Backup failed: $($result.Error)"
        }
    }
    
    [void]RestoreDatabase([string]$backupFile, [string]$newDatabaseName = "") {
        if (-not $newDatabaseName) {
            $newDatabaseName = $this.Database
        }
        
        Write-Host "Starting database restore to: $newDatabaseName"
        
        # Mise de la base en mode single user pour la restauration
        $singleUserQuery = "ALTER DATABASE [$newDatabaseName] SET SINGLE_USER WITH ROLLBACK IMMEDIATE;"
        $this.ExecuteQuery($singleUserQuery)
        
        # Restauration
        $restoreQuery = "RESTORE DATABASE [$newDatabaseName] FROM DISK = '$backupFile' WITH REPLACE;"
        $result = $this.ExecuteQuery($restoreQuery)
        
        if ($result.Success) {
            # Remise en mode multi-user
            $multiUserQuery = "ALTER DATABASE [$newDatabaseName] SET MULTI_USER;"
            $this.ExecuteQuery($multiUserQuery)
            
            Write-Host "Database restore completed: $newDatabaseName"
        } else {
            throw "Restore failed: $($result.Error)"
        }
    }
    
    [void]RunMaintenanceTasks() {
        Write-Host "Running database maintenance tasks..."
        
        # Reconstruction des index
        $rebuildIndexesQuery = @"
DECLARE @TableName VARCHAR(255)
DECLARE @sql NVARCHAR(500)
DECLARE @fillfactor INT
SET @fillfactor = 80
DECLARE TableCursor CURSOR FOR
SELECT OBJECT_SCHEMA_NAME([object_id])+'.'+name AS TableName
FROM sys.tables
WHERE name NOT LIKE 'sys%'
OPEN TableCursor
FETCH NEXT FROM TableCursor INTO @TableName
WHILE @@FETCH_STATUS = 0
BEGIN
SET @sql = 'ALTER INDEX ALL ON ' + @TableName + ' REBUILD WITH (FILLFACTOR = ' + CONVERT(VARCHAR(3),@fillfactor) + ')'
EXEC (@sql)
FETCH NEXT FROM TableCursor INTO @TableName
END
CLOSE TableCursor
DEALLOCATE TableCursor
"@
        
        $result = $this.ExecuteQuery($rebuildIndexesQuery)
        if ($result.Success) {
            Write-Host "Index rebuild completed"
        }
        
        # Mise à jour des statistiques
        $updateStatsQuery = "EXEC sp_updatestats;"
        $result = $this.ExecuteQuery($updateStatsQuery)
        if ($result.Success) {
            Write-Host "Statistics update completed"
        }
        
        # Shrink du log
        $shrinkLogQuery = "DBCC SHRINKFILE ($($this.Database)_log, 1);"
        $result = $this.ExecuteQuery($shrinkLogQuery)
        if ($result.Success) {
            Write-Host "Log shrink completed"
        }
    }
    
    [hashtable]GetHealthMetrics() {
        Write-Host "Collecting database health metrics..."
        
        $metrics = @{}
        
        # Taille de la base
        $sizeQuery = "SELECT SUM(size) * 8 / 1024 AS SizeMB FROM sys.master_files WHERE database_id = DB_ID('$($this.Database)');"
        $result = $this.ExecuteQuery($sizeQuery)
        $metrics.DatabaseSizeMB = if ($result.Success -and $result.Result.Rows.Count -gt 0) { $result.Result.Rows[0].SizeMB } else { 0 }
        
        # Nombre de connexions actives
        $connectionsQuery = "SELECT COUNT(*) as ActiveConnections FROM sys.dm_exec_connections WHERE session_id > 50;"
        $result = $this.ExecuteQuery($connectionsQuery)
        $metrics.ActiveConnections = if ($result.Success -and $result.Result.Rows.Count -gt 0) { $result.Result.Rows[0].ActiveConnections } else { 0 }
        
        # Latence moyenne des requêtes
        $latencyQuery = "SELECT AVG(total_elapsed_time) / 1000 as AvgLatencyMs FROM sys.dm_exec_query_stats WHERE last_execution_time > DATEADD(hour, -1, GETDATE());"
        $result = $this.ExecuteQuery($latencyQuery)
        $metrics.AverageLatencyMs = if ($result.Success -and $result.Result.Rows.Count -gt 0) { $result.Result.Rows[0].AvgLatencyMs } else { 0 }
        
        # Utilisation du cache
        $cacheQuery = "SELECT (SELECT COUNT(*) FROM sys.dm_os_memory_cache_counters WHERE type = 'CACHESTORE_SQLCP') * 100.0 / (SELECT COUNT(*) FROM sys.dm_os_memory_cache_counters) as CacheHitRatio;"
        $result = $this.ExecuteQuery($cacheQuery)
        $metrics.CacheHitRatio = if ($result.Success -and $result.Result.Rows.Count -gt 0) { $result.Result.Rows[0].CacheHitRatio } else { 0 }
        
        return $metrics
    }
}

# Automatisation complète de maintenance de base de données
function Invoke-DatabaseMaintenance {
    param(
        [string]$Server = "localhost",
        [string]$Database = "MyApp",
        [string]$BackupPath = "C:\Database\Backups",
        [switch]$SkipBackup,
        [switch]$SkipRestoreTest
    )
    
    Write-Host "=== DATABASE MAINTENANCE AUTOMATION ===" -ForegroundColor Cyan
    Write-Host "Server: $Server"
    Write-Host "Database: $Database"
    Write-Host "Backup Path: $BackupPath"
    Write-Host ""
    
    # Initialisation
    $dbAutomation = [DatabaseAutomation]::new($Server, $Database, $null)
    
    # Collecte des métriques de santé avant maintenance
    Write-Host "Collecting pre-maintenance health metrics..."
    $preMetrics = $dbAutomation.GetHealthMetrics()
    Write-Host "Database size: $($preMetrics.DatabaseSizeMB) MB"
    Write-Host "Active connections: $($preMetrics.ActiveConnections)"
    Write-Host ""
    
    # Sauvegarde (si demandée)
    if (-not $SkipBackup) {
        try {
            $dbAutomation.BackupDatabase($BackupPath)
        } catch {
            Write-Error "Backup failed, aborting maintenance: $_"
            return
        }
    }
    
    # Exécution des tâches de maintenance
    try {
        $dbAutomation.RunMaintenanceTasks()
    } catch {
        Write-Error "Maintenance tasks failed: $_"
        return
    }
    
    # Test de restauration (si demandé)
    if (-not $SkipRestoreTest -and -not $SkipBackup) {
        Write-Host "Testing restore functionality..."
        try {
            $testDbName = "$Database`_TestRestore"
            $latestBackup = Get-ChildItem $BackupPath -Filter "*.bak" | Sort-Object LastWriteTime -Descending | Select-Object -First 1
            
            if ($latestBackup) {
                $dbAutomation.RestoreDatabase($latestBackup.FullName, $testDbName)
                
                # Nettoyage de la base de test
                $dropQuery = "DROP DATABASE [$testDbName];"
                $dbAutomation.ExecuteQuery($dropQuery)
                
                Write-Host "Restore test successful"
            } else {
                Write-Warning "No backup found for restore test"
            }
        } catch {
            Write-Warning "Restore test failed: $_"
        }
    }
    
    # Collecte des métriques après maintenance
    Write-Host "Collecting post-maintenance health metrics..."
    $postMetrics = $dbAutomation.GetHealthMetrics()
    Write-Host "Maintenance completed successfully"
    Write-Host ""
    
    # Rapport de maintenance
    Write-Host "=== MAINTENANCE REPORT ===" -ForegroundColor Green
    Write-Host "Database: $Database"
    Write-Host "Server: $Server"
    Write-Host "Completed at: $(Get-Date)"
    Write-Host ""
    Write-Host "Performance improvements:"
    Write-Host "  Cache hit ratio: $($postMetrics.CacheHitRatio.ToString('F2'))%"
    Write-Host "  Average latency: $($postMetrics.AverageLatencyMs.ToString('F2')) ms"
    Write-Host ""
    Write-Host "Maintenance tasks completed:"
    Write-Host "  ✓ Index rebuild"
    Write-Host "  ✓ Statistics update"
    Write-Host "  ✓ Log shrink"
    if (-not $SkipBackup) { Write-Host "  ✓ Database backup" }
    if (-not $SkipRestoreTest) { Write-Host "  ✓ Restore test" }
}

# Programmation de la maintenance automatisée
$maintenanceAction = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-Command Invoke-DatabaseMaintenance -Server 'prod-db' -Database 'MyApp' -BackupPath 'D:\Backups'"
$maintenanceTrigger = New-ScheduledTaskTrigger -Weekly -WeeksInterval 1 -DaysOfWeek Sunday -At "02:00"
Register-ScheduledTask -TaskName "WeeklyDatabaseMaintenance" -Action $maintenanceAction -Trigger $maintenanceTrigger -Principal (New-ScheduledTaskPrincipal -UserId "dbadmin" -RunLevel Highest)

# Automatisation des rapports de santé
$healthReportAction = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-Command & { `$db = [DatabaseAutomation]::new('prod-db', 'MyApp', `$null); `$metrics = `$db.GetHealthMetrics(); `$metrics | ConvertTo-Json | Out-File 'C:\Reports\db_health_$(Get-Date -Format yyyyMMdd).json' }"
$healthTrigger = New-ScheduledTaskTrigger -Daily -At "09:00"
Register-ScheduledTask -TaskName "DailyDatabaseHealthReport" -Action $healthReportAction -Trigger $healthTrigger
```

## Conclusion : PowerShell comme moteur d'automatisation universel

PowerShell transcende son rôle initial d'interface de commande pour devenir un puissant moteur d'automatisation capable de gérer des workflows complexes, d'orchestrer des tâches distribuées, et d'intégrer des systèmes hétérogènes.

Dans le prochain chapitre, nous explorerons les techniques avancées de développement PowerShell, incluant les modules, les classes, et les bonnes pratiques de développement.

---

**Exercice pratique :** Créez un système d'automatisation complet qui :
1. Surveille la santé des systèmes toutes les 5 minutes
2. Exécute des sauvegardes automatiques selon des critères définis
3. Génère des rapports quotidiens par email
4. Effectue la maintenance des bases de données hebdomadairement
5. Intègre avec des APIs externes pour l'orchestration

**Challenge avancé :** Développez une plateforme d'automatisation enterprise qui :
- Supporte des workflows multi-étapes avec rollback
- Intègre avec Azure DevOps, GitHub Actions, et Jenkins
- Fournit une interface web pour la gestion des tâches
- Implémente la sécurité et l'audit des automatisations
- S'adapte dynamiquement à la charge du système

**Réflexion :** Comment l'automatisation PowerShell transforme-t-elle les opérations IT ? Quels sont les défis de l'automatisation à grande échelle, et comment les surmonter ? L'automatisation peut-elle devenir trop complexe, et si oui, comment maintenir la simplicité ?
