# Chapitre 133 - Debugging et profilage avancé PowerShell

> "Le debugging n'est pas seulement corriger les bugs, c'est comprendre en profondeur comment votre code interagit avec le système, révélant des insights qui améliorent non seulement la correction, mais aussi la conception future." - Citation inspirée des pratiques de debugging avancé

## Introduction : Debugging comme exploration systémique

Le debugging avancé PowerShell transcende la simple correction d'erreurs pour devenir une exploration méthodique du comportement du code, des performances du système, et des interactions complexes. En maîtrisant les outils de debugging, de profilage, et d'analyse de trace, les développeurs PowerShell peuvent identifier des problèmes subtils, optimiser les performances, et améliorer la robustesse des scripts.

Dans ce chapitre, nous explorerons les techniques avancées de debugging, les outils de profilage, et les stratégies d'analyse de performance.

## Section 1 : Debugging avancé avec PowerShell

### 1.1 Utilisation du debugger PowerShell intégré

**Configuration et utilisation avancée du debugger :**
```powershell
# Configuration du debugger
$DebugPreference = "Continue"  # Afficher tous les messages debug
$VerbosePreference = "Continue"  # Afficher les messages verbeux
$WarningPreference = "Continue"  # Afficher les avertissements
$ErrorActionPreference = "Stop"  # Stopper sur les erreurs

# Fonction avec debugging intégré
function Debug-ComplexFunction {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true)]
        [string]$InputPath,

        [Parameter(Mandatory=$true)]
        [string]$OutputPath,

        [switch]$Force
    )

    Write-Verbose "Début de Debug-ComplexFunction"
    Write-Debug "Paramètres: InputPath=$InputPath, OutputPath=$OutputPath, Force=$Force"

    # Point d'arrêt conditionnel
    if ($InputPath -notmatch '\.txt$') {
        Write-Warning "Le fichier d'entrée n'est pas un fichier texte"
        if (-not $Force) {
            throw "Extension de fichier invalide. Utilisez -Force pour forcer."
        }
    }

    # Validation des chemins
    if (-not (Test-Path $InputPath)) {
        Write-Error "Fichier d'entrée introuvable: $InputPath" -ErrorAction Stop
    }

    $inputInfo = Get-Item $InputPath
    Write-Debug "Informations fichier d'entrée: Taille=$($inputInfo.Length) octets, Modifié=$($inputInfo.LastWriteTime)"

    # Traitement du fichier
    try {
        Write-Verbose "Lecture du fichier d'entrée"
        $content = Get-Content $InputPath -Raw
        Write-Debug "Contenu chargé: $($content.Length) caractères"

        # Traitement complexe simulé
        Write-Verbose "Traitement du contenu"
        $processedContent = $content.ToUpper()

        # Validation du résultat
        if ($processedContent.Length -ne $content.Length) {
            Write-Warning "La longueur du contenu a changé pendant le traitement"
        }

        Write-Verbose "Écriture du fichier de sortie"
        $processedContent | Out-File $OutputPath -Encoding UTF8

        Write-Verbose "Fonction terminée avec succès"
        Write-Debug "Fichier de sortie créé: $OutputPath"

    } catch {
        Write-Error "Erreur pendant le traitement: $_" -ErrorAction Stop
    }
}

# Utilisation avec debugging
$DebugPreference = "Continue"
Debug-ComplexFunction -InputPath "C:\Temp\input.txt" -OutputPath "C:\Temp\output.txt" -Verbose -Debug

# Utilisation du debugger interactif
Set-PSBreakpoint -Command "Debug-ComplexFunction" -Action {
    Write-Host "Point d'arrêt atteint dans Debug-ComplexFunction" -ForegroundColor Yellow
    # Ici, on peut inspecter les variables, modifier l'état, etc.
}

# Points d'arrêt sur variables
Set-PSBreakpoint -Variable "content" -Mode Write -Action {
    Write-Host "Variable 'content' modifiée: $($content.Length) caractères" -ForegroundColor Cyan
}

# Points d'arrêt sur erreurs
Set-PSBreakpoint -Variable "Error" -Mode Write -Action {
    Write-Host "Erreur détectée: $($Error[-1])" -ForegroundColor Red
}

# Session de debugging interactive
function Enter-DebugSession {
    param([scriptblock]$ScriptBlock)

    try {
        # Configuration des points d'arrêt
        $breakpoints = @(
            @{ Line = 10; Script = $ScriptBlock.File },
            @{ Variable = "result"; Mode = "Write" }
        )

        foreach ($bp in $breakpoints) {
            if ($bp.Line) {
                Set-PSBreakpoint -Script $bp.Script -Line $bp.Line
            } elseif ($bp.Variable) {
                Set-PSBreakpoint -Variable $bp.Variable -Mode $bp.Mode
            }
        }

        # Exécution avec debugging
        & $ScriptBlock

    } finally {
        # Nettoyage des points d'arrêt
        Get-PSBreakpoint | Remove-PSBreakpoint
    }
}

# Exemple d'utilisation
$debugScript = {
    $result = 0
    for ($i = 1; $i -le 10; $i++) {
        $result += $i
        Write-Debug "Itération $i, résultat partiel: $result"
    }
    return $result
}

Enter-DebugSession -ScriptBlock $debugScript
```

### 1.2 Debugging à distance et multi-processus

**Techniques de debugging distribué :**
```powershell
# Fonction de debugging à distance
function Debug-RemoteSession {
    param(
        [Parameter(Mandatory=$true)]
        [string]$ComputerName,

        [Parameter(Mandatory=$true)]
        [PSCredential]$Credential,

        [Parameter(Mandatory=$true)]
        [scriptblock]$DebugScript
    )

    Write-Host "Démarrage du debugging à distance sur $ComputerName" -ForegroundColor Cyan

    # Création de la session distante
    $session = New-PSSession -ComputerName $ComputerName -Credential $Credential

    try {
        # Configuration du debugging distant
        Invoke-Command -Session $session -ScriptBlock {
            # Configuration des préférences de debug
            $global:DebugPreference = "Continue"
            $global:VerbosePreference = "Continue"

            # Fonction de logging distant
            function Write-RemoteLog {
                param([string]$Message, [string]$Level = "INFO")
                $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
                $logMessage = "[$timestamp] [$Level] [REMOTE] $Message"
                Write-Host $logMessage -ForegroundColor (switch ($Level) {
                    "ERROR" { "Red" }
                    "WARN" { "Yellow" }
                    "DEBUG" { "Gray" }
                    default { "White" }
                })
            }
        }

        # Transfert et exécution du script de debug
        $remoteScript = @"
try {
    Write-RemoteLog "Début de l'exécution distante"

    # Script à déboguer
    $DebugScript

    Write-RemoteLog "Exécution distante terminée"
} catch {
    Write-RemoteLog "Erreur distante: `$_" "ERROR"
    throw
}
"@

        # Exécution avec capture des erreurs
        $result = Invoke-Command -Session $session -ScriptBlock ([scriptblock]::Create($remoteScript)) -ErrorVariable remoteError

        if ($remoteError) {
            Write-Warning "Erreurs distantes détectées:"
            $remoteError | ForEach-Object { Write-Host "  $_" -ForegroundColor Red }
        }

        return $result

    } finally {
        # Nettoyage de la session
        Remove-PSSession -Session $session
        Write-Host "Session de debugging distant terminée" -ForegroundColor Green
    }
}

# Debugging de services Windows à distance
function Debug-RemoteService {
    param(
        [Parameter(Mandatory=$true)]
        [string]$ComputerName,

        [Parameter(Mandatory=$true)]
        [string]$ServiceName,

        [PSCredential]$Credential
    )

    $debugScript = {
        param($serviceName)

        Write-RemoteLog "Analyse du service: $serviceName"

        $service = Get-Service -Name $serviceName -ErrorAction SilentlyContinue
        if (-not $service) {
            throw "Service '$serviceName' introuvable"
        }

        Write-RemoteLog "Service trouvé: $($service.DisplayName), Status: $($service.Status)"

        # Analyse des dépendances
        $dependencies = Get-Service -Name $serviceName -RequiredServices
        Write-RemoteLog "Dépendances: $($dependencies.Name -join ', ')"

        # Vérification du processus associé
        if ($service.Status -eq 'Running') {
            $process = Get-Process -Name "*$serviceName*" -ErrorAction SilentlyContinue | Select-Object -First 1
            if ($process) {
                Write-RemoteLog "Processus associé: PID=$($process.Id), CPU=$($process.CPU), Memory=$([math]::Round($process.WorkingSet / 1MB, 2)) MB"
            }
        }

        # Analyse des logs d'événements
        $events = Get-WinEvent -LogName System -MaxEvents 10 -ErrorAction SilentlyContinue |
            Where-Object { $_.Message -match $serviceName }

        if ($events) {
            Write-RemoteLog "Événements récents ($($events.Count) trouvés):"
            $events | ForEach-Object {
                Write-RemoteLog "  $($_.TimeCreated): $($_.LevelDisplayName) - $($_.Message.Substring(0, 100))"
            }
        }

        return @{
            ServiceName = $service.Name
            DisplayName = $service.DisplayName
            Status = $service.Status
            Dependencies = $dependencies.Name
            EventCount = $events.Count
        }
    }

    return Debug-RemoteSession -ComputerName $ComputerName -Credential $Credential -ScriptBlock $debugScript -ArgumentList $ServiceName
}

# Utilisation du debugging à distance
$cred = Get-Credential
$result = Debug-RemoteService -ComputerName "remote-server" -ServiceName "wuauserv" -Credential $cred
$result

# Debugging multi-session (plusieurs serveurs simultanément)
function Debug-MultiSession {
    param(
        [Parameter(Mandatory=$true)]
        [string[]]$ComputerNames,

        [Parameter(Mandatory=$true)]
        [PSCredential]$Credential,

        [Parameter(Mandatory=$true)]
        [scriptblock]$DebugScript
    )

    Write-Host "Démarrage du debugging multi-session sur $($ComputerNames.Count) serveurs" -ForegroundColor Cyan

    # Création des sessions en parallèle
    $sessions = $ComputerNames | ForEach-Object {
        $computer = $_
        try {
            New-PSSession -ComputerName $computer -Credential $Credential -ErrorAction Stop
        } catch {
            Write-Warning "Impossible de créer une session sur $computer`: $_"
            $null
        }
    } | Where-Object { $_ -ne $null }

    Write-Host "$($sessions.Count) sessions créées sur $($ComputerNames.Count) serveurs demandés"

    try {
        # Exécution parallèle sur toutes les sessions
        $jobs = Invoke-Command -Session $sessions -ScriptBlock $DebugScript -AsJob

        # Attente et récupération des résultats
        $results = $jobs | Wait-Job | Receive-Job

        # Agrégation des résultats
        $aggregatedResults = @{}
        foreach ($result in $results) {
            if ($result -and $result.PSComputerName) {
                $aggregatedResults[$result.PSComputerName] = $result
            }
        }

        return $aggregatedResults

    } finally {
        # Nettoyage
        $sessions | Remove-PSSession
        Get-Job | Remove-Job
        Write-Host "Sessions multi-debugging nettoyées" -ForegroundColor Green
    }
}

# Exemple de debugging multi-session
$servers = @("server1", "server2", "server3")
$cred = Get-Credential

$multiDebugScript = {
    $computerName = $env:COMPUTERNAME
    $uptime = (Get-CimInstance Win32_OperatingSystem).LastBootUpTime
    $cpu = Get-WmiObject Win32_Processor | Measure-Object -Property LoadPercentage -Average

    return [PSCustomObject]@{
        ComputerName = $computerName
        Uptime = (Get-Date) - $uptime
        CPUUsage = $cpu.Average
        Timestamp = Get-Date
    }
}

$multiResults = Debug-MultiSession -ComputerNames $servers -Credential $cred -ScriptBlock $multiDebugScript

Write-Host "=== RÉSULTATS MULTI-SESSION ==="
foreach ($server in $servers) {
    if ($multiResults.ContainsKey($server)) {
        $result = $multiResults[$server]
        Write-Host "$server`: Uptime=$($result.Uptime.Days)j $($result.Uptime.Hours)h, CPU=$($result.CPUUsage)%"
    } else {
        Write-Host "$server`: Échec de connexion" -ForegroundColor Red
    }
}
```

### 1.3 Analyse des traces et logs avancés

**Techniques d'analyse de traces :**
```powershell
# Système d'analyse de traces avancé
class TraceAnalyzer {
    [System.Collections.Generic.List[PSCustomObject]]$Traces
    [hashtable]$Patterns
    [System.Collections.Generic.Dictionary[string, int]]$Counters

    TraceAnalyzer() {
        $this.Traces = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.Counters = [System.Collections.Generic.Dictionary[string, int]]::new()

        # Patterns de détection d'anomalies
        $this.Patterns = @{
            ErrorSpikes = @{
                Pattern = "Exception|Error|Failed"
                Threshold = 5
                TimeWindow = [TimeSpan]::FromMinutes(10)
            }
            PerformanceDegradation = @{
                Pattern = "timeout|slow"
                Threshold = 3
                TimeWindow = [TimeSpan]::FromMinutes(5)
            }
            ResourceExhaustion = @{
                Pattern = "OutOfMemory|DiskFull|ConnectionPool"
                Threshold = 1
                TimeWindow = [TimeSpan]::FromHours(1)
            }
        }
    }

    [void]AddTrace([PSCustomObject]$trace) {
        # Validation de la trace
        if (-not $trace.Timestamp -or -not $trace.Message) {
            Write-Warning "Trace invalide ignorée"
            return
        }

        $this.Traces.Add($trace)

        # Mise à jour des compteurs
        $key = "$($trace.Level):$($trace.Source)"
        if (-not $this.Counters.ContainsKey($key)) {
            $this.Counters[$key] = 0
        }
        $this.Counters[$key]++
    }

    [PSCustomObject[]]GetTraces([DateTime]$startTime, [DateTime]$endTime) {
        return $this.Traces | Where-Object {
            $_.Timestamp -ge $startTime -and $_.Timestamp -le $endTime
        }
    }

    [hashtable]AnalyzePatterns() {
        $analysis = @{}

        foreach ($patternName in $this.Patterns.Keys) {
            $pattern = $this.Patterns[$patternName]
            $analysis[$patternName] = $this.AnalyzePattern($patternName, $pattern)
        }

        return $analysis
    }

    [PSCustomObject]AnalyzePattern([string]$patternName, [hashtable]$pattern) {
        $now = Get-Date
        $windowStart = $now - $pattern.TimeWindow

        $matchingTraces = $this.Traces | Where-Object {
            $_.Timestamp -ge $windowStart -and
            $_.Message -match $pattern.Pattern
        }

        $result = [PSCustomObject]@{
            PatternName = $patternName
            MatchCount = $matchingTraces.Count
            Threshold = $pattern.Threshold
            TimeWindow = $pattern.TimeWindow
            IsAnomaly = $matchingTraces.Count -ge $pattern.Threshold
            LastOccurrence = if ($matchingTraces) { ($matchingTraces | Sort-Object Timestamp -Descending | Select-Object -First 1).Timestamp } else { $null }
            RecentTraces = $matchingTraces | Select-Object -Last 5
        }

        return $result
    }

    [PSCustomObject[]]FindCorrelations([string]$primaryPattern, [string[]]$relatedPatterns) {
        $correlations = @()

        $primaryTraces = $this.Traces | Where-Object { $_.Message -match $primaryPattern }

        foreach ($primaryTrace in $primaryTraces) {
            $correlation = [PSCustomObject]@{
                PrimaryTrace = $primaryTrace
                RelatedTraces = @()
                CorrelationStrength = 0
            }

            # Recherche de traces liées dans une fenêtre temporelle
            $timeWindow = [TimeSpan]::FromMinutes(5)
            $windowStart = $primaryTrace.Timestamp - $timeWindow
            $windowEnd = $primaryTrace.Timestamp + $timeWindow

            foreach ($relatedPattern in $relatedPatterns) {
                $relatedTraces = $this.Traces | Where-Object {
                    $_.Timestamp -ge $windowStart -and
                    $_.Timestamp -le $windowEnd -and
                    $_.Message -match $relatedPattern -and
                    $_ -ne $primaryTrace
                }

                if ($relatedTraces) {
                    $correlation.RelatedTraces += $relatedTraces
                    $correlation.CorrelationStrength += $relatedTraces.Count
                }
            }

            if ($correlation.RelatedTraces.Count -gt 0) {
                $correlations += $correlation
            }
        }

        return $correlations | Sort-Object CorrelationStrength -Descending
    }

    [string]GenerateReport() {
        $report = @"
RAPPORT D'ANALYSE DE TRACES
Généré le: $(Get-Date)

STATISTIQUES GÉNÉRALES
=====================
Total des traces: $($this.Traces.Count)
Période couverte: $(if ($this.Traces.Count -gt 0) { "$($this.Traces[0].Timestamp) - $($this.Traces[-1].Timestamp)" } else { "Aucune trace" })

COMPTEURS PAR TYPE
==================
"@

        foreach ($counter in $this.Counters.GetEnumerator() | Sort-Object Value -Descending) {
            $report += "$($counter.Key): $($counter.Value)`n"
        }

        $report += @"

ANALYSE DES PATTERNS
====================
"@

        $patternAnalysis = $this.AnalyzePatterns()
        foreach ($patternName in $patternAnalysis.Keys) {
            $analysis = $patternAnalysis[$patternName]
            $status = if ($analysis.IsAnomaly) { "ANOMALIE DÉTECTÉE" } else { "Normal" }
            $report += "$patternName`: $status ($($analysis.MatchCount)/$($analysis.Threshold) dans $($analysis.TimeWindow))`n"
        }

        return $report
    }
}

# Fonction de collecte de traces PowerShell
function Start-TraceCollection {
    param(
        [Parameter(Mandatory=$true)]
        [TraceAnalyzer]$Analyzer,

        [int]$DurationMinutes = 60
    )

    Write-Host "Démarrage de la collecte de traces ($DurationMinutes minutes)" -ForegroundColor Cyan

    $startTime = Get-Date
    $endTime = $startTime.AddMinutes($DurationMinutes)

    # Hook pour capturer les commandes PowerShell
    $originalCommandLookup = $ExecutionContext.InvokeCommand.GetCommand

    $ExecutionContext.InvokeCommand.GetCommand = {
        param([string]$commandName, [System.Management.Automation.CommandTypes]$commandType, [object]$args)

        $trace = [PSCustomObject]@{
            Timestamp = Get-Date
            Level = "DEBUG"
            Source = "PowerShell"
            Message = "Commande exécutée: $commandName"
            CommandType = $commandType
            Arguments = $args
        }

        $Analyzer.AddTrace($trace)

        # Appel de la fonction originale
        & $originalCommandLookup $commandName $commandType $args
    }

    # Collecte des erreurs
    $global:OriginalErrorAction = $ErrorActionPreference
    $ErrorActionPreference = "Continue"

    $global:ErrorHandler = $null
    $global:ErrorHandler = Register-ObjectEvent -InputObject $global:Error -EventName "Changed" -Action {
        $error = $global:Error[-1]
        if ($error) {
            $trace = [PSCustomObject]@{
                Timestamp = Get-Date
                Level = "ERROR"
                Source = "PowerShell"
                Message = "Erreur: $($error.Exception.Message)"
                Category = $error.CategoryInfo.Category
                Target = $error.CategoryInfo.TargetName
            }

            $Analyzer.AddTrace($trace)
        }
    }

    # Attente de la durée spécifiée
    while ((Get-Date) -lt $endTime) {
        Start-Sleep -Seconds 10

        # Trace périodique
        $trace = [PSCustomObject]@{
            Timestamp = Get-Date
            Level = "INFO"
            Source = "Collector"
            Message = "Collecte active - Traces collectées: $($Analyzer.Traces.Count)"
        }

        $Analyzer.AddTrace($trace)
    }

    # Nettoyage
    $ErrorActionPreference = $global:OriginalErrorAction
    Unregister-Event -SourceIdentifier $global:ErrorHandler.Name -ErrorAction SilentlyContinue

    Write-Host "Collecte de traces terminée. Total: $($Analyzer.Traces.Count) traces" -ForegroundColor Green
}

# Exemple d'utilisation du système de traces
$analyzer = [TraceAnalyzer]::new()

# Démarrage de la collecte en arrière-plan
$collectionJob = Start-Job -ScriptBlock {
    param($analyzer)

    # Simulation de génération de traces
    for ($i = 1; $i -le 20; $i++) {
        $trace = [PSCustomObject]@{
            Timestamp = Get-Date
            Level = if ($i % 5 -eq 0) { "ERROR" } elseif ($i % 3 -eq 0) { "WARN" } else { "INFO" }
            Source = "TestScript"
            Message = "Trace de test numéro $i - $(if ($i % 5 -eq 0) { 'Erreur simulée' } elseif ($i % 3 -eq 0) { 'Avertissement' } else { 'Information normale' })"
        }

        $analyzer.AddTrace($trace)
        Start-Sleep -Milliseconds 500
    }

} -ArgumentList $analyzer

# Attente de la fin de la collecte
Wait-Job $collectionJob
Receive-Job $collectionJob
Remove-Job $collectionJob

# Analyse des traces
Write-Host "=== ANALYSE DES TRACES ==="
$patternAnalysis = $analyzer.AnalyzePatterns()
foreach ($pattern in $patternAnalysis.Keys) {
    $result = $patternAnalysis[$pattern]
    if ($result.IsAnomaly) {
        Write-Host "⚠️  ANOMALIE: $pattern - $($result.MatchCount) occurrences" -ForegroundColor Red
    } else {
        Write-Host "✓ $pattern - $($result.MatchCount) occurrences" -ForegroundColor Green
    }
}

# Recherche de corrélations
$correlations = $analyzer.FindCorrelations("Error", @("timeout", "failed", "exception"))
if ($correlations) {
    Write-Host "`n=== CORRÉLATIONS TROUVÉES ==="
    foreach ($correlation in $correlations | Select-Object -First 3) {
        Write-Host "Erreur: $($correlation.PrimaryTrace.Message)"
        Write-Host "  Corrélation: $($correlation.RelatedTraces.Count) traces liées"
        Write-Host ""
    }
}

# Génération du rapport
$report = $analyzer.GenerateReport()
$report | Out-File -FilePath "TraceAnalysisReport.txt" -Encoding UTF8
Write-Host "Rapport généré: TraceAnalysisReport.txt"
```

## Section 2 : Profilage et optimisation des performances

### 2.1 Outils de profilage PowerShell

**Profilage détaillé des scripts :**
```powershell
# Module de profilage avancé
class PowerShellProfiler {
    [System.Collections.Generic.Dictionary[string, System.Diagnostics.Stopwatch]]$Timers
    [System.Collections.Generic.Dictionary[string, System.Collections.Generic.List[double]]]$Metrics
    [System.Collections.Generic.List[PSCustomObject]]$CallStack
    [bool]$IsProfiling

    PowerShellProfiler() {
        $this.Timers = [System.Collections.Generic.Dictionary[string, System.Diagnostics.Stopwatch]]::new()
        $this.Metrics = [System.Collections.Generic.Dictionary[string, System.Collections.Generic.List[double]]]::new()
        $this.CallStack = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.IsProfiling = $false
    }

    [void]StartProfiling() {
        $this.IsProfiling = $true
        Write-Host "Profilage PowerShell démarré" -ForegroundColor Green
    }

    [void]StopProfiling() {
        $this.IsProfiling = $false
        $this.StopAllTimers()
        Write-Host "Profilage PowerShell arrêté" -ForegroundColor Yellow
    }

    [void]StartTimer([string]$name) {
        if (-not $this.IsProfiling) { return }

        if (-not $this.Timers.ContainsKey($name)) {
            $this.Timers[$name] = [System.Diagnostics.Stopwatch]::new()
        }

        $this.Timers[$name].Start()

        $callInfo = [PSCustomObject]@{
            Name = $name
            StartTime = Get-Date
            CallStack = (Get-PSCallStack | Select-Object -Skip 1)
        }

        $this.CallStack.Add($callInfo)
    }

    [void]StopTimer([string]$name) {
        if (-not $this.IsProfiling -or -not $this.Timers.ContainsKey($name)) { return }

        $this.Timers[$name].Stop()

        $duration = $this.Timers[$name].Elapsed.TotalMilliseconds

        if (-not $this.Metrics.ContainsKey($name)) {
            $this.Metrics[$name] = [System.Collections.Generic.List[double]]::new()
        }

        $this.Metrics[$name].Add($duration)

        # Mise à jour du call stack
        $callInfo = $this.CallStack | Where-Object { $_.Name -eq $name } | Select-Object -Last 1
        if ($callInfo) {
            $callInfo | Add-Member -MemberType NoteProperty -Name "Duration" -Value $duration
            $callInfo | Add-Member -MemberType NoteProperty -Name "EndTime" -Value (Get-Date)
        }
    }

    [void]StopAllTimers() {
        foreach ($timerName in $this.Timers.Keys) {
            $this.StopTimer($timerName)
        }
    }

    [hashtable]GetMetrics() {
        $results = @{}

        foreach ($metricName in $this.Metrics.Keys) {
            $values = $this.Metrics[$metricName]
            $results[$metricName] = @{
                Count = $values.Count
                Average = [math]::Round(($values | Measure-Object -Average).Average, 2)
                Min = [math]::Round(($values | Measure-Object -Minimum).Minimum, 2)
                Max = [math]::Round(($values | Measure-Object -Maximum).Maximum, 2)
                Total = [math]::Round(($values | Measure-Object -Sum).Sum, 2)
            }
        }

        return $results
    }

    [PSCustomObject[]]GetCallStack() {
        return $this.CallStack.ToArray()
    }

    [void]ProfileCommand([scriptblock]$command, [string]$name = "Command") {
        $this.StartTimer($name)
        try {
            & $command
        } finally {
            $this.StopTimer($name)
        }
    }

    [string]GenerateProfileReport() {
        $metrics = $this.GetMetrics()
        $callStack = $this.GetCallStack()

        $report = @"
RAPPORT DE PROFILAGE POWERSHELL
================================
Généré le: $(Get-Date)

MÉTRIQUES DE PERFORMANCE
========================
"@

        foreach ($metricName in $metrics.Keys) {
            $metric = $metrics[$metricName]
            $report += @"

$metricName
  Exécutions: $($metric.Count)
  Moyenne: $($metric.Average) ms
  Minimum: $($metric.Min) ms
  Maximum: $($metric.Max) ms
  Total: $($metric.Total) ms
"@
        }

        $report += @"

ANALYSE DE LA PILE D'APPELS
===========================
"@

        $groupedCalls = $callStack | Group-Object Name
        foreach ($group in $groupedCalls) {
            $avgDuration = [math]::Round(($group.Group | Measure-Object Duration -Average).Average, 2)
            $report += "$($group.Name): $($group.Count) appels, durée moyenne: ${avgDuration} ms`n"
        }

        $report += @"

RECOMMANDATIONS D'OPTIMISATION
==============================
"@

        # Analyse automatique pour recommandations
        $slowOperations = $metrics.Keys | Where-Object {
            $metrics[$_].Average -gt 1000  # Plus d'1 seconde
        }

        if ($slowOperations) {
            $report += "Opérations lentes détectées (> 1000ms):`n"
            foreach ($op in $slowOperations) {
                $report += "  - $op ($($metrics[$op].Average)ms moyenne)`n"
            }
            $report += "`nConsidérer l'optimisation ou la mise en cache.`n"
        }

        $frequentOperations = $metrics.Keys | Where-Object {
            $metrics[$_].Count -gt 100  # Plus de 100 appels
        }

        if ($frequentOperations) {
            $report += "`nOpérations fréquentes détectées (> 100 appels):`n"
            foreach ($op in $frequentOperations) {
                $report += "  - $op ($($metrics[$op].Count) appels)`n"
            }
            $report += "`nConsidérer la vectorisation ou l'optimisation algorithmique.`n"
        }

        return $report
    }
}

# Fonctions utilitaires de profilage
function Start-Profiler {
    param([PowerShellProfiler]$Profiler = $null)

    if (-not $Profiler) {
        $global:PSProfiler = [PowerShellProfiler]::new()
    } else {
        $global:PSProfiler = $Profiler
    }

    $global:PSProfiler.StartProfiling()
}

function Stop-Profiler {
    if ($global:PSProfiler) {
        $global:PSProfiler.StopProfiling()

        $report = $global:PSProfiler.GenerateProfileReport()
        Write-Host "=== RAPPORT DE PROFILAGE ===" -ForegroundColor Cyan
        Write-Host $report

        return $global:PSProfiler
    }
}

function Measure-CommandProfiled {
    param(
        [Parameter(Mandatory=$true)]
        [scriptblock]$Command,

        [string]$Name = "AnonymousCommand"
    )

    if ($global:PSProfiler -and $global:PSProfiler.IsProfiling) {
        $global:PSProfiler.ProfileCommand($Command, $Name)
    } else {
        & $Command
    }
}

# Proxy functions pour intercepter les appels
$originalGetService = $function:Get-Service
function Get-Service {
    [CmdletBinding(DefaultParameterSetName='Default', HelpUri='https://go.microsoft.com/fwlink/?LinkID=2096492')]
    param(
        [Parameter(ParameterSetName='Default', Position=0)]
        [string[]]$Name,

        [Parameter(ParameterSetName='Default')]
        [string[]]$DisplayName,

        [Parameter(ParameterSetName='Default')]
        [string[]]$Include,

        [Parameter(ParameterSetName='Default')]
        [string[]]$Exclude,

        [Parameter(ParameterSetName='InputObject', Mandatory=$true, ValueFromPipeline=$true)]
        [psobject]$InputObject,

        [Parameter(ParameterSetName='Default')]
        [Alias('CN','ComputerName')]
        [string[]]$ComputerName,

        [switch]$DependentServices,

        [switch]$RequiredServices,

        [switch]$All
    )

    $timerName = "Get-Service"
    if ($global:PSProfiler -and $global:PSProfiler.IsProfiling) {
        $global:PSProfiler.StartTimer($timerName)
    }

    try {
        # Appel de la fonction originale
        if ($PSBoundParameters.ContainsKey('InputObject')) {
            $result = & $originalGetService @PSBoundParameters
        } else {
            $result = & $originalGetService @PSBoundParameters
        }

        return $result
    } finally {
        if ($global:PSProfiler -and $global:PSProfiler.IsProfiling) {
            $global:PSProfiler.StopTimer($timerName)
        }
    }
}

# Exemple d'utilisation du profilage
Start-Profiler

# Script à profiler
Measure-CommandProfiled -Name "ServiceOperations" -Command {
    $services = Get-Service -Name "*win*" | Where-Object { $_.Status -eq 'Running' }
    foreach ($service in $services) {
        $deps = Get-Service -Name $service.Name -RequiredServices
        Write-Debug "Service $($service.Name) has $($deps.Count) dependencies"
    }
}

Measure-CommandProfiled -Name "ComplexCalculation" -Command {
    $result = 0
    for ($i = 0; $i -lt 10000; $i++) {
        $result += [math]::Sqrt($i) * [math]::Sin($i)
    }
    Write-Host "Complex calculation result: $result"
}

$profiler = Stop-Profiler

# Analyse détaillée
$metrics = $profiler.GetMetrics()
Write-Host "`n=== ANALYSE DÉTAILLÉE ===" -ForegroundColor Yellow
foreach ($metricName in $metrics.Keys) {
    $metric = $metrics[$metricName]
    Write-Host "$metricName`: $($metric.Count) exécutions, moyenne $($metric.Average)ms"
}

# Sauvegarde du rapport
$profiler.GenerateProfileReport() | Out-File -FilePath "PowerShellProfileReport.txt" -Encoding UTF8
Write-Host "Rapport de profilage sauvegardé: PowerShellProfileReport.txt"
```

### 2.2 Analyse de performance mémoire

**Outils d'analyse mémoire avancée :**
```powershell
# Analyseur de mémoire PowerShell
class MemoryAnalyzer {
    [System.Collections.Generic.Dictionary[string, object]]$Snapshots
    [System.Collections.Generic.List[PSCustomObject]]$Leaks

    MemoryAnalyzer() {
        $this.Snapshots = [System.Collections.Generic.Dictionary[string, object]]::new()
        $this.Leaks = [System.Collections.Generic.List[PSCustomObject]]::new()
    }

    [void]TakeSnapshot([string]$name) {
        Write-Host "Capture d'instantané mémoire: $name" -ForegroundColor Cyan

        $snapshot = @{
            Timestamp = Get-Date
            ProcessInfo = Get-Process -Id $PID
            RunspaceInfo = Get-Runspace
            VariableInfo = Get-Variable | Where-Object { $_.Value -is [Array] -or $_.Value -is [hashtable] -or $_.Value.PSObject.Properties.Count -gt 0 }
            ModuleInfo = Get-Module
        }

        $this.Snapshots[$name] = $snapshot

        # Calcul de l'empreinte mémoire
        $memoryUsage = $snapshot.ProcessInfo.WorkingSet
        Write-Host "Empreinte mémoire: $([math]::Round($memoryUsage / 1MB, 2)) MB"
    }

    [hashtable]CompareSnapshots([string]$snapshot1, [string]$snapshot2) {
        if (-not $this.Snapshots.ContainsKey($snapshot1) -or -not $this.Snapshots.ContainsKey($snapshot2)) {
            throw "Instantané(s) introuvable(s)"
        }

        $snap1 = $this.Snapshots[$snapshot1]
        $snap2 = $this.Snapshots[$snapshot2]

        $comparison = @{
            MemoryDelta = $snap2.ProcessInfo.WorkingSet - $snap1.ProcessInfo.WorkingSet
            VariableDelta = ($snap2.VariableInfo | Measure-Object).Count - ($snap1.VariableInfo | Measure-Object).Count
            TimeDelta = $snap2.Timestamp - $snap1.Timestamp
        }

        return $comparison
    }

    [void]DetectMemoryLeaks() {
        Write-Host "Détection des fuites mémoire..." -ForegroundColor Yellow

        # Analyse des variables globales qui grossissent
        $globalVariables = Get-Variable -Scope Global | Where-Object {
            $_.Value -is [Array] -or $_.Value -is [System.Collections.Generic.List[object]]
        }

        foreach ($var in $globalVariables) {
            $itemCount = 0

            if ($var.Value -is [Array]) {
                $itemCount = $var.Value.Count
            } elseif ($var.Value -is [System.Collections.Generic.List[object]]) {
                $itemCount = $var.Value.Count
            }

            if ($itemCount -gt 1000) {  # Seuil arbitraire
                $leak = [PSCustomObject]@{
                    VariableName = $var.Name
                    ItemCount = $itemCount
                    VariableType = $var.Value.GetType().Name
                    DetectedAt = Get-Date
                    Severity = if ($itemCount -gt 10000) { "Critical" } elseif ($itemCount -gt 5000) { "High" } else { "Medium" }
                }

                $this.Leaks.Add($leak)
                Write-Warning "Fuite mémoire potentielle détectée: $($var.Name) ($itemCount éléments)"
            }
        }

        # Analyse des runspaces
        $runspaces = Get-Runspace | Where-Object { $_.RunspaceAvailability -ne 'Available' }
        if ($runspaces) {
            Write-Warning "Runspaces potentiellement bloqués: $($runspaces.Count)"
            foreach ($rs in $runspaces) {
                Write-Host "  Runspace $($rs.Id): $($rs.RunspaceAvailability)" -ForegroundColor Red
            }
        }
    }

    [hashtable]AnalyzeMemoryPatterns() {
        $patterns = @{
            LargeObjects = @()
            Collections = @()
            CacheObjects = @()
        }

        # Recherche d'objets volumineux
        $variables = Get-Variable | Where-Object { $null -ne $_.Value }

        foreach ($var in $variables) {
            try {
                $size = 0

                if ($var.Value -is [Array] -or $var.Value -is [System.Collections.ICollection]) {
                    $size = $var.Value.Count
                    $patterns.Collections += @{
                        Variable = $var.Name
                        Count = $size
                        Type = $var.Value.GetType().Name
                    }
                } elseif ($var.Value.PSObject -and $var.Value.PSObject.Properties) {
                    $size = $var.Value.PSObject.Properties.Count
                    if ($size -gt 50) {  # Objets complexes
                        $patterns.LargeObjects += @{
                            Variable = $var.Name
                            PropertyCount = $size
                            Type = $var.Value.GetType().Name
                        }
                    }
                }

                # Détection de caches
                if ($var.Name -match "cache|Cache" -or $var.Value.GetType().Name -match "Dictionary|Hashtable") {
                    $cacheSize = 0
                    if ($var.Value -is [hashtable] -or $var.Value -is [System.Collections.IDictionary]) {
                        $cacheSize = $var.Value.Count
                    }

                    if ($cacheSize -gt 100) {
                        $patterns.CacheObjects += @{
                            Variable = $var.Name
                            Size = $cacheSize
                            Type = $var.Value.GetType().Name
                        }
                    }
                }

            } catch {
                # Ignorer les erreurs d'analyse
            }
        }

        return $patterns
    }

    [string]GenerateMemoryReport() {
        $this.DetectMemoryLeaks()
        $patterns = $this.AnalyzeMemoryPatterns()

        $report = @"
RAPPORT D'ANALYSE MÉMOIRE POWERSHELL
=====================================
Généré le: $(Get-Date)

INSTANTANÉS CAPTURÉS
===================
$($this.Snapshots.Count) instantané(s) disponible(s): $($this.Snapshots.Keys -join ', ')

FUITES MÉMOIRE DÉTECTÉES
========================
$($this.Leaks.Count) fuite(s) potentielle(s) trouvée(s)

"@

        foreach ($leak in $this.Leaks) {
            $report += "- $($leak.VariableName): $($leak.ItemCount) éléments ($($leak.Severity))`n"
        }

        $report += @"

PATTERNS MÉMOIRE
================
Collections volumineuses:
$($patterns.Collections | ForEach-Object { "  - $($_.Variable): $($_.Count) éléments ($($_.Type))" } | Out-String)

Objets complexes:
$($patterns.LargeObjects | ForEach-Object { "  - $($_.Variable): $($_.PropertyCount) propriétés ($($_.Type))" } | Out-String)

Caches volumineux:
$($patterns.CacheObjects | ForEach-Object { "  - $($_.Variable): $($_.Size) éléments ($($_.Type))" } | Out-String)

RECOMMANDATIONS
===============
"@

        if ($this.Leaks.Count -gt 0) {
            $report += "- Nettoyer les variables globales volumineuses`n"
            $report += "- Implémenter une gestion appropriée des collections`n"
        }

        if ($patterns.Collections.Count -gt 0) {
            $report += "- Considérer la pagination pour les grandes collections`n"
            $report += "- Implémenter des mécanismes de nettoyage automatique`n"
        }

        if ($patterns.CacheObjects.Count -gt 0) {
            $report += "- Implémenter des politiques d'expiration pour les caches`n"
            $report += "- Surveiller la taille des caches`n"
        }

        $report += @"
- Utiliser [GC]::Collect() pour forcer le garbage collection si nécessaire
- Surveiller l'utilisation mémoire avec Get-Process
- Considérer l'utilisation de streams pour les gros volumes de données
"@

        return $report
    }
}

# Fonctions utilitaires pour l'analyse mémoire
function Start-MemoryAnalysis {
    param([MemoryAnalyzer]$Analyzer = $null)

    if (-not $Analyzer) {
        $global:MemoryAnalyzer = [MemoryAnalyzer]::new()
    } else {
        $global:MemoryAnalyzer = $Analyzer
    }

    Write-Host "Analyse mémoire démarrée" -ForegroundColor Green
}

function Stop-MemoryAnalysis {
    if ($global:MemoryAnalyzer) {
        $report = $global:MemoryAnalyzer.GenerateMemoryReport()
        Write-Host "=== RAPPORT MÉMOIRE ===" -ForegroundColor Cyan
        Write-Host $report

        return $global:MemoryAnalyzer
    }
}

function Get-MemorySnapshot {
    param([string]$Name = "Snapshot_$(Get-Date -Format 'yyyyMMdd_HHmmss')")

    if ($global:MemoryAnalyzer) {
        $global:MemoryAnalyzer.TakeSnapshot($Name)
    } else {
        Write-Warning "Analyseur mémoire non initialisé. Utilisez Start-MemoryAnalysis d'abord."
    }
}

# Exemple d'utilisation de l'analyse mémoire
Start-MemoryAnalysis

# Capture d'instantané initial
Get-MemorySnapshot -Name "Initial"

# Simulation d'une opération qui peut causer des fuites
$largeCollection = [System.Collections.Generic.List[object]]::new()
for ($i = 0; $i -lt 5000; $i++) {
    $largeCollection.Add(@{
        Id = $i
        Name = "Item$i"
        Data = "Some data for item $i with additional content to consume memory"
        Timestamp = Get-Date
    })
}

# Capture d'instantané après l'opération
Get-MemorySnapshot -Name "AfterLargeOperation"

# Création d'un cache qui peut grossir
$global:MyCache = @{}
for ($i = 0; $i -lt 2000; $i++) {
    $global:MyCache["key_$i"] = "value_$i"
}

# Capture finale
Get-MemorySnapshot -Name "Final"

$analyzer = Stop-MemoryAnalysis

# Comparaison des instantanés
if ($analyzer.Snapshots.ContainsKey("Initial") -and $analyzer.Snapshots.ContainsKey("Final")) {
    $comparison = $analyzer.CompareSnapshots("Initial", "Final")
    Write-Host "`n=== COMPARAISON MÉMOIRE ===" -ForegroundColor Yellow
    Write-Host "Delta mémoire: $([math]::Round($comparison.MemoryDelta / 1MB, 2)) MB"
    Write-Host "Delta variables: $($comparison.VariableDelta)"
    Write-Host "Durée: $($comparison.TimeDelta)"
}

# Sauvegarde du rapport
$analyzer.GenerateMemoryReport() | Out-File -FilePath "MemoryAnalysisReport.txt" -Encoding UTF8
Write-Host "Rapport d'analyse mémoire sauvegardé: MemoryAnalysisReport.txt"
```

## Section 3 : Automatisation du debugging et monitoring

### 3.1 Systèmes de surveillance automatique

**Monitoring proactif des scripts :**
```powershell
# Système de surveillance automatique des scripts PowerShell
class ScriptMonitor {
    [System.Collections.Generic.Dictionary[string, hashtable]]$ScriptStats
    [System.Collections.Generic.List[PSCustomObject]]$Alerts
    [hashtable]$Thresholds
    [scriptblock]$AlertAction

    ScriptMonitor() {
        $this.ScriptStats = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.Alerts = [System.Collections.Generic.List[PSCustomObject]]::new()

        # Seuils par défaut
        $this.Thresholds = @{
            MaxExecutionTime = 300  # 5 minutes
            MaxMemoryUsage = 500MB  # 500 MB
            MaxErrorCount = 5       # 5 erreurs
            MinSuccessRate = 0.95   # 95%
        }

        # Action d'alerte par défaut
        $this.AlertAction = {
            param($alert)
            Write-Host "🚨 ALERT: $($alert.Message)" -ForegroundColor Red
            Write-Host "   Script: $($alert.ScriptName)" -ForegroundColor Yellow
            Write-Host "   Severity: $($alert.Severity)" -ForegroundColor Yellow
            Write-Host "   Timestamp: $($alert.Timestamp)" -ForegroundColor Yellow
        }
    }

    [void]RegisterScript([string]$scriptName, [hashtable]$metadata = @{}) {
        $this.ScriptStats[$scriptName] = @{
            Name = $scriptName
            ExecutionCount = 0
            SuccessCount = 0
            FailureCount = 0
            TotalExecutionTime = 0
            MaxExecutionTime = 0
            MinExecutionTime = [double]::MaxValue
            AverageExecutionTime = 0
            LastExecution = $null
            LastSuccess = $null
            LastFailure = $null
            MemoryPeaks = @()
            Errors = [System.Collections.Generic.List[string]]::new()
            Metadata = $metadata
        }

        Write-Debug "Script enregistré: $scriptName"
    }

    [PSCustomObject]ExecuteMonitored([string]$scriptName, [scriptblock]$scriptBlock) {
        if (-not $this.ScriptStats.ContainsKey($scriptName)) {
            $this.RegisterScript($scriptName)
        }

        $stats = $this.ScriptStats[$scriptName]
        $startTime = Get-Date
        $startMemory = (Get-Process -Id $PID).WorkingSet

        $executionId = [guid]::NewGuid().ToString()

        Write-Debug "Début exécution $scriptName (ID: $executionId)"

        $stats.ExecutionCount++

        try {
            # Exécution du script
            $result = & $scriptBlock

            $executionTime = ((Get-Date) - $startTime).TotalSeconds
            $endMemory = (Get-Process -Id $PID).WorkingSet
            $memoryUsage = $endMemory - $startMemory

            # Mise à jour des statistiques
            $stats.SuccessCount++
            $stats.LastExecution = Get-Date
            $stats.LastSuccess = Get-Date
            $stats.TotalExecutionTime += $executionTime

            if ($executionTime -gt $stats.MaxExecutionTime) {
                $stats.MaxExecutionTime = $executionTime
            }

            if ($executionTime -lt $stats.MinExecutionTime) {
                $stats.MinExecutionTime = $executionTime
            }

            $stats.AverageExecutionTime = $stats.TotalExecutionTime / $stats.ExecutionCount
            $stats.MemoryPeaks += $memoryUsage

            # Vérification des seuils
            $this.CheckThresholds($scriptName, @{
                ExecutionTime = $executionTime
                MemoryUsage = $memoryUsage
            })

            Write-Debug "Exécution réussie $scriptName en $([math]::Round($executionTime, 2))s"

            return [PSCustomObject]@{
                Success = $true
                ExecutionId = $executionId
                ExecutionTime = $executionTime
                MemoryUsage = $memoryUsage
                Result = $result
            }

        } catch {
            $executionTime = ((Get-Date) - $startTime).TotalSeconds

            $stats.FailureCount++
            $stats.LastExecution = Get-Date
            $stats.LastFailure = Get-Date
            $stats.Errors.Add("$((Get-Date)): $($_.Exception.Message)")

            # Alerte d'échec
            $this.CreateAlert($scriptName, "Script execution failed", "Error", @{
                ErrorMessage = $_.Exception.Message
                ExecutionTime = $executionTime
            })

            Write-Error "Échec exécution $scriptName`: $_"

            return [PSCustomObject]@{
                Success = $false
                ExecutionId = $executionId
                ExecutionTime = $executionTime
                Error = $_.Exception.Message
            }
        }
    }

    [void]CheckThresholds([string]$scriptName, [hashtable]$metrics) {
        $stats = $this.ScriptStats[$scriptName]

        # Vérification du temps d'exécution
        if ($metrics.ExecutionTime -gt $this.Thresholds.MaxExecutionTime) {
            $this.CreateAlert($scriptName, "Execution time exceeded threshold", "Warning", @{
                ExecutionTime = $metrics.ExecutionTime
                Threshold = $this.Thresholds.MaxExecutionTime
            })
        }

        # Vérification de l'utilisation mémoire
        if ($metrics.MemoryUsage -gt $this.Thresholds.MaxMemoryUsage) {
            $this.CreateAlert($scriptName, "Memory usage exceeded threshold", "Warning", @{
                MemoryUsage = $metrics.MemoryUsage
                Threshold = $this.Thresholds.MaxMemoryUsage
            })
        }

        # Vérification du taux de succès
        $successRate = $stats.SuccessCount / $stats.ExecutionCount
        if ($successRate -lt $this.Thresholds.MinSuccessRate) {
            $this.CreateAlert($scriptName, "Success rate below threshold", "Critical", @{
                SuccessRate = $successRate
                Threshold = $this.Thresholds.MinSuccessRate
            })
        }

        # Vérification du nombre d'erreurs récentes
        $recentErrors = $stats.Errors | Where-Object { $_ -match ((Get-Date).AddHours(-1).ToString("yyyy-MM-dd HH")) }
        if ($recentErrors.Count -gt $this.Thresholds.MaxErrorCount) {
            $this.CreateAlert($scriptName, "Too many errors in the last hour", "Critical", @{
                ErrorCount = $recentErrors.Count
                Threshold = $this.Thresholds.MaxErrorCount
            })
        }
    }

    [void]CreateAlert([string]$scriptName, [string]$message, [string]$severity, [hashtable]$details = @{}) {
        $alert = [PSCustomObject]@{
            Id = [guid]::NewGuid().ToString()
            Timestamp = Get-Date
            ScriptName = $scriptName
            Message = $message
            Severity = $severity
            Details = $details
        }

        $this.Alerts.Add($alert)

        # Exécution de l'action d'alerte
        & $this.AlertAction $alert
    }

    [PSCustomObject[]]GetRecentAlerts([int]$hours = 24) {
        $cutoff = (Get-Date).AddHours(-$hours)
        return $this.Alerts | Where-Object { $_.Timestamp -gt $cutoff } | Sort-Object Timestamp -Descending
    }

    [hashtable]GetScriptStats([string]$scriptName) {
        if (-not $this.ScriptStats.ContainsKey($scriptName)) {
            return $null
        }

        $stats = $this.ScriptStats[$scriptName].Clone()
        $stats.SuccessRate = if ($stats.ExecutionCount -gt 0) { $stats.SuccessCount / $stats.ExecutionCount } else { 0 }
        $stats.AverageMemoryPeak = if ($stats.MemoryPeaks.Count -gt 0) { ($stats.MemoryPeaks | Measure-Object -Average).Average } else { 0 }

        return $stats
    }

    [string]GenerateHealthReport() {
        $totalScripts = $this.ScriptStats.Count
        $totalExecutions = ($this.ScriptStats.Values | Measure-Object ExecutionCount -Sum).Sum
        $totalAlerts = $this.Alerts.Count

        $recentAlerts = $this.GetRecentAlerts(24)

        $report = @"
RAPPORT DE SANTÉ DES SCRIPTS POWERSHELL
========================================
Généré le: $(Get-Date)

STATISTIQUES GÉNÉRALES
=====================
Scripts surveillés: $totalScripts
Exécutions totales: $totalExecutions
Alertes totales: $totalAlerts
Alertes (dernières 24h): $($recentAlerts.Count)

STATISTIQUES PAR SCRIPT
=======================
"@

        foreach ($scriptName in $this.ScriptStats.Keys) {
            $stats = $this.GetScriptStats($scriptName)
            $health = if ($stats.SuccessRate -gt 0.95) { "✓" } elseif ($stats.SuccessRate -gt 0.90) { "⚠" } else { "✗" }

            $report += @"

$scriptName
  $health Taux de succès: $([math]::Round($stats.SuccessRate * 100, 1))%
  Exécutions: $($stats.ExecutionCount)
  Temps moyen: $([math]::Round($stats.AverageExecutionTime, 2))s
  Pic mémoire moyen: $([math]::Round($stats.AverageMemoryPeak / 1MB, 2)) MB
  Dernière exécution: $($stats.LastExecution)
"@
        }

        $report += @"

ALERTES RÉCENTES
================
"@

        foreach ($alert in $recentAlerts | Select-Object -First 10) {
            $icon = switch ($alert.Severity) {
                "Critical" { "🔴" }
                "Error" { "❌" }
                "Warning" { "⚠️" }
                default { "ℹ️" }
            }

            $report += "$icon $($alert.Timestamp): $($alert.ScriptName) - $($alert.Message)`n"
        }

        return $report
    }
}

# Fonctions utilitaires pour le monitoring
function Start-ScriptMonitoring {
    param([ScriptMonitor]$Monitor = $null)

    if (-not $Monitor) {
        $global:ScriptMonitor = [ScriptMonitor]::new()
    } else {
        $global:ScriptMonitor = $Monitor
    }

    Write-Host "Monitoring des scripts démarré" -ForegroundColor Green
}

function Stop-ScriptMonitoring {
    if ($global:ScriptMonitor) {
        $report = $global:ScriptMonitor.GenerateHealthReport()
        Write-Host "=== RAPPORT DE SANTÉ ===" -ForegroundColor Cyan
        Write-Host $report

        return $global:ScriptMonitor
    }
}

function Watch-Script {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Name,

        [Parameter(Mandatory=$true)]
        [scriptblock]$ScriptBlock
    )

    if ($global:ScriptMonitor) {
        return $global:ScriptMonitor.ExecuteMonitored($Name, $ScriptBlock)
    } else {
        Write-Warning "Monitoring non activé. Exécution normale."
        return & $ScriptBlock
    }
}

# Exemple d'utilisation du monitoring automatique
Start-ScriptMonitoring

# Enregistrement de scripts à surveiller
$global:ScriptMonitor.RegisterScript("Backup-Service", @{
    Description = "Sauvegarde des données importantes"
    Owner = "IT Operations"
    Criticality = "High"
})

$global:ScriptMonitor.RegisterScript("Cleanup-Logs", @{
    Description = "Nettoyage des anciens fichiers de log"
    Owner = "System Admin"
    Criticality = "Medium"
})

# Exécution de scripts surveillés
Watch-Script -Name "Backup-Service" -ScriptBlock {
    Write-Host "Démarrage de la sauvegarde..."
    Start-Sleep -Seconds 2  # Simulation

    # Simulation d'une opération qui peut échouer
    if ((Get-Random -Maximum 10) -lt 8) {  # 80% de succès
        Write-Host "Sauvegarde terminée avec succès"
        return "Backup completed"
    } else {
        throw "Erreur de sauvegarde simulée"
    }
}

Watch-Script -Name "Cleanup-Logs" -ScriptBlock {
    Write-Host "Nettoyage des logs..."
    Start-Sleep -Seconds 1

    # Simulation d'une opération lente
    $largeOperation = 0
    for ($i = 0; $i -lt 100000; $i++) {
        $largeOperation += $i
    }

    Write-Host "Nettoyage terminé"
    return "Logs cleaned"
}

# Rapport final
$monitor = Stop-ScriptMonitoring

# Analyse des statistiques
Write-Host "`n=== ANALYSE DES STATISTIQUES ===" -ForegroundColor Yellow
foreach ($scriptName in $monitor.ScriptStats.Keys) {
    $stats = $monitor.GetScriptStats($scriptName)
    Write-Host "$scriptName`: $($stats.ExecutionCount) exécutions, taux de succès $([math]::Round($stats.SuccessRate * 100, 1))%"
}

# Sauvegarde du rapport de santé
$monitor.GenerateHealthReport() | Out-File -FilePath "ScriptHealthReport.txt" -Encoding UTF8
Write-Host "Rapport de santé sauvegardé: ScriptHealthReport.txt"
```

## Conclusion : Debugging comme discipline DevOps

Le debugging avancé PowerShell transcende la simple correction d'erreurs pour devenir une discipline systémique qui améliore la qualité, la performance et la fiabilité des scripts. En combinant profilage, analyse mémoire, tracing distribué et monitoring automatique, les développeurs PowerShell peuvent identifier les problèmes complexes et optimiser leurs solutions de manière proactive.

Dans le prochain chapitre, nous explorerons les techniques de sécurisation avancée des scripts PowerShell et les meilleures pratiques de développement sécurisé.

---

**Exercice pratique :** Créez un système complet de debugging et profilage qui :
1. Profile automatiquement tous les scripts PowerShell exécutés
2. Détecte les fuites mémoire et les goulots d'étranglement
3. Analyse les traces pour identifier les patterns d'erreur
4. Génère des rapports de performance détaillés
5. Implémente des alertes proactives pour les problèmes critiques

**Challenge avancé :** Développez un framework de debugging distribué qui :
- Trace les appels à travers plusieurs serveurs et services
- Corréle automatiquement les erreurs liées
- Fournit des visualisations en temps réel des performances
- Implémente des recommandations d'optimisation automatiques
- S'intègre avec les outils de monitoring existants

**Réflexion :** Comment le debugging évolue-t-il avec la complexité croissante des systèmes ? Le debugging traditionnel suffit-il face aux architectures distribuées modernes, ou nécessite-t-il une approche fondamentalement différente ?
