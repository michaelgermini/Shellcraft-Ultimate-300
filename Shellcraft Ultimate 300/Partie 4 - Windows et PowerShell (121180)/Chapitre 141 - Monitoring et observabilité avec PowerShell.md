# Chapitre 141 - Monitoring et observabilité avec PowerShell

> "L'observabilité n'est pas seulement la surveillance des systèmes, c'est l'art de comprendre leur comportement à travers le prisme des données, révélant les patterns cachés qui transforment la réactivité en proactivité." - Citation inspirée des principes d'observabilité moderne

## Introduction : PowerShell comme observateur omniscient

PowerShell transcende son rôle d'outil d'administration pour devenir l'observateur ultime des systèmes complexes. En intégrant les métriques, les logs, les traces, et l'analyse comportementale, PowerShell crée des systèmes d'observabilité d'une profondeur et d'une intelligence sans précédent.

Dans ce chapitre, nous explorerons les frameworks de monitoring avancés, l'analyse de logs intelligente, et les tableaux de bord d'observabilité.

## Section 1 : Framework de monitoring avancé

### 1.1 Collecte et traitement des métriques

**Système de métriques multi-sources avec analyse temps réel :**
```powershell
# Framework avancé de collecte et traitement des métriques
class MetricsCollector {
    [System.Collections.Generic.Dictionary[string, PSObject]]$Metrics
    [System.Collections.Generic.List[PSCustomObject]]$MetricHistory
    [hashtable]$Collectors
    [System.Collections.Generic.Queue[PSCustomObject]]$AlertQueue
    [hashtable]$Thresholds
    [scriptblock]$AlertHandler

    MetricsCollector() {
        $this.Metrics = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.MetricHistory = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.Collectors = @{}
        $this.AlertQueue = [System.Collections.Generic.Queue[PSCustomObject]]::new()

        $this.Thresholds = @{
            CPU_Usage = @{ Warning = 70; Critical = 90 }
            Memory_Usage = @{ Warning = 80; Critical = 95 }
            Disk_Usage = @{ Warning = 85; Critical = 95 }
            Network_Latency = @{ Warning = 100; Critical = 500 }
        }

        $this.AlertHandler = {
            param($alert)
            Write-Host "ALERT: $($alert.Message)" -ForegroundColor (switch ($alert.Severity) {
                "Critical" { "Red" }
                "Warning" { "Yellow" }
                "Info" { "Blue" }
                default { "White" }
            })
        }

        $this.InitializeCollectors()
    }

    hidden [void]InitializeCollectors() {
        # Collecteur système
        $this.Collectors.System = {
            $cpu = Get-Counter '\Processor(_Total)\% Processor Time' -ErrorAction SilentlyContinue
            $memory = Get-Counter '\Memory\% Committed Bytes In Use' -ErrorAction SilentlyContinue
            $disk = Get-WmiObject Win32_LogicalDisk -Filter "DeviceID='C:'"

            return @{
                Timestamp = Get-Date
                CPU_Usage = if ($cpu) { $cpu.CounterSamples[0].CookedValue } else { 0 }
                Memory_Usage = if ($memory) { $memory.CounterSamples[0].CookedValue } else { 0 }
                Disk_Usage = [math]::Round(($disk.Size - $disk.FreeSpace) / $disk.Size * 100, 2)
                Disk_Free_GB = [math]::Round($disk.FreeSpace / 1GB, 2)
            }
        }

        # Collecteur réseau
        $this.Collectors.Network = {
            $connections = Get-NetTCPConnection -State Established | Measure-Object
            $adapters = Get-NetAdapter | Where-Object { $_.Status -eq "Up" }

            $networkMetrics = @{
                Timestamp = Get-Date
                ActiveConnections = $connections.Count
                NetworkAdapters = @()
            }

            foreach ($adapter in $adapters) {
                $stats = Get-NetAdapterStatistics -Name $adapter.Name
                $networkMetrics.NetworkAdapters += @{
                    Name = $adapter.Name
                    BytesSent = $stats.SentBytes
                    BytesReceived = $stats.ReceivedBytes
                    PacketsSent = $stats.SentUnicastPackets
                    PacketsReceived = $stats.ReceivedUnicastPackets
                }
            }

            return $networkMetrics
        }

        # Collecteur processus
        $this.Collectors.Processes = {
            $processes = Get-Process | Where-Object { $_.CPU -gt 0 } | Sort-Object CPU -Descending | Select-Object -First 10

            return @{
                Timestamp = Get-Date
                TopProcesses = $processes | ForEach-Object {
                    @{
                        Name = $_.ProcessName
                        Id = $_.Id
                        CPU = [math]::Round($_.CPU, 2)
                        MemoryMB = [math]::Round($_.WorkingSet / 1MB, 2)
                        Threads = $_.Threads.Count
                    }
                }
                TotalProcesses = (Get-Process).Count
            }
        }

        # Collecteur services
        $this.Collectors.Services = {
            $services = Get-Service
            $running = ($services | Where-Object { $_.Status -eq "Running" }).Count
            $stopped = ($services | Where-Object { $_.Status -eq "Stopped" }).Count

            return @{
                Timestamp = Get-Date
                TotalServices = $services.Count
                RunningServices = $running
                StoppedServices = $stopped
                ServiceHealth = [math]::Round(($running / $services.Count) * 100, 2)
            }
        }
    }

    [void]CollectMetrics() {
        Write-Host "Collecting metrics from all sources..." -ForegroundColor Gray

        foreach ($collectorName in $this.Collectors.Keys) {
            try {
                $metrics = & $this.Collectors[$collectorName]

                # Stockage des métriques
                $this.Metrics[$collectorName] = $metrics

                # Historique
                $this.MetricHistory.Add([PSCustomObject]@{
                    Collector = $collectorName
                    Metrics = $metrics
                    CollectedAt = Get-Date
                })

                # Vérification des seuils
                $this.CheckThresholds($collectorName, $metrics)

            } catch {
                Write-Warning "Failed to collect metrics from $collectorName`: $_"
            }
        }

        # Nettoyage de l'historique (garder seulement les dernières 1000 entrées)
        if ($this.MetricHistory.Count -gt 1000) {
            $this.MetricHistory = $this.MetricHistory | Select-Object -Last 500
        }
    }

    hidden [void]CheckThresholds([string]$collectorName, [hashtable]$metrics) {
        foreach ($metricName in $metrics.Keys) {
            if ($this.Thresholds.ContainsKey($metricName)) {
                $thresholds = $this.Thresholds[$metricName]
                $value = $metrics[$metricName]

                if ($value -is [double] -or $value -is [int]) {
                    $severity = $null
                    $message = $null

                    if ($value -ge $thresholds.Critical) {
                        $severity = "Critical"
                        $message = "$metricName is at critical level: $value"
                    } elseif ($value -ge $thresholds.Warning) {
                        $severity = "Warning"
                        $message = "$metricName is at warning level: $value"
                    }

                    if ($severity) {
                        $alert = [PSCustomObject]@{
                            Id = [guid]::NewGuid().ToString()
                            Timestamp = Get-Date
                            Collector = $collectorName
                            Metric = $metricName
                            Value = $value
                            Threshold = $thresholds.($severity.ToLower())
                            Severity = $severity
                            Message = $message
                        }

                        $this.AlertQueue.Enqueue($alert)
                    }
                }
            }
        }
    }

    [void]ProcessAlerts() {
        while ($this.AlertQueue.Count -gt 0) {
            $alert = $this.AlertQueue.Dequeue()
            & $this.AlertHandler $alert
        }
    }

    [PSCustomObject[]]GetMetrics([string]$collector = $null, [int]$hours = 1) {
        $cutoff = (Get-Date).AddHours(-$hours)

        $history = $this.MetricHistory | Where-Object { $_.CollectedAt -ge $cutoff }

        if ($collector) {
            $history = $history | Where-Object { $_.Collector -eq $collector }
        }

        return $history
    }

    [hashtable]GetCurrentMetrics() {
        return $this.Metrics.Clone()
    }

    [hashtable]CalculateTrends([string]$metricName, [int]$hours = 24) {
        $metrics = $this.GetMetrics($null, $hours)

        $metricValues = @()
        foreach ($entry in $metrics) {
            if ($entry.Metrics.ContainsKey($metricName)) {
                $metricValues += @{
                    Value = $entry.Metrics[$metricName]
                    Timestamp = $entry.CollectedAt
                }
            }
        }

        if ($metricValues.Count -lt 2) {
            return @{ Trend = "InsufficientData"; Change = 0; Direction = "Unknown" }
        }

        # Calcul de la tendance
        $firstValue = $metricValues[0].Value
        $lastValue = $metricValues[-1].Value
        $change = $lastValue - $firstValue
        $changePercent = if ($firstValue -ne 0) { [math]::Round(($change / $firstValue) * 100, 2) } else { 0 }

        $direction = if ($change -gt 0) { "Increasing" } elseif ($change -lt 0) { "Decreasing" } else { "Stable" }

        # Calcul de la volatilité
        $values = $metricValues | ForEach-Object { $_.Value }
        $average = ($values | Measure-Object -Average).Average
        $variance = ($values | ForEach-Object { [math]::Pow($_ - $average, 2) } | Measure-Object -Sum).Sum / $values.Count
        $volatility = [math]::Round([math]::Sqrt($variance), 2)

        return @{
            Trend = "$direction by $changePercent%"
            Change = $change
            ChangePercent = $changePercent
            Direction = $direction
            Volatility = $volatility
            Average = [math]::Round($average, 2)
            Min = ($values | Measure-Object -Minimum).Minimum
            Max = ($values | Measure-Object -Maximum).Maximum
            SampleCount = $values.Count
        }
    }

    [PSCustomObject[]]GetAlerts([int]$hours = 24) {
        $cutoff = (Get-Date).AddHours(-$hours)

        # Simulation d'historique d'alertes (dans un vrai système, ceci serait stocké)
        return $this.MetricHistory | Where-Object { $_.CollectedAt -ge $cutoff } | ForEach-Object {
            # Créer des alertes fictives basées sur les métriques
            $alerts = @()

            if ($_.Metrics.ContainsKey("CPU_Usage") -and $_.Metrics.CPU_Usage -gt 80) {
                $alerts += [PSCustomObject]@{
                    Timestamp = $_.CollectedAt
                    Metric = "CPU_Usage"
                    Value = $_.Metrics.CPU_Usage
                    Severity = if ($_.Metrics.CPU_Usage -gt 90) { "Critical" } else { "Warning" }
                    Message = "High CPU usage detected"
                }
            }

            $alerts
        } | Where-Object { $_ } | Sort-Object Timestamp -Descending
    }

    [void]AddCustomCollector([string]$name, [scriptblock]$collector) {
        $this.Collectors[$name] = $collector
        Write-Host "Custom collector '$name' added" -ForegroundColor Green
    }

    [void]SetThreshold([string]$metricName, [hashtable]$thresholds) {
        $this.Thresholds[$metricName] = $thresholds
        Write-Host "Thresholds updated for metric: $metricName" -ForegroundColor Green
    }

    [string]GenerateMetricsReport() {
        $currentMetrics = $this.GetCurrentMetrics()
        $alerts = $this.GetAlerts(1)

        $report = @"
METRICS COLLECTOR REPORT
========================
Generated: $(Get-Date)

CURRENT METRICS
===============
$($currentMetrics.Keys | ForEach-Object {
    $collector = $_
    $metrics = $currentMetrics[$collector]
    "$collector`:`n$($metrics.GetEnumerator() | Where-Object { $_.Key -ne 'Timestamp' } | ForEach-Object { "  $($_.Key): $($_.Value)" } | Out-String)"
} | Out-String)

TRENDS (Last 24h)
=================
CPU Usage: $($this.CalculateTrends('CPU_Usage').Trend)
Memory Usage: $($this.CalculateTrends('Memory_Usage').Trend)
Disk Usage: $($this.CalculateTrends('Disk_Usage').Trend)

ALERTS (Last Hour)
==================
$($alerts | Select-Object -First 5 | ForEach-Object { "$($_.Timestamp): [$($_.Severity)] $($_.Message) (Value: $($_.Value))" } | Out-String)

COLLECTORS STATUS
=================
$($this.Collectors.Keys | ForEach-Object { "• $_" } | Out-String)
"@

        return $report
    }

    [void]StartMonitoring([int]$intervalSeconds = 60) {
        Write-Host "Starting metrics monitoring (interval: $intervalSeconds seconds)..." -ForegroundColor Green

        $job = Start-Job -ScriptBlock {
            param($collector, $interval)

            while ($true) {
                $collector.CollectMetrics()
                $collector.ProcessAlerts()
                Start-Sleep -Seconds $interval
            }
        } -ArgumentList $this, $intervalSeconds

        Write-Host "Monitoring started. Job ID: $($job.Id)" -ForegroundColor Green
    }
}

# Système d'analyse de logs avancé
class LogAnalyzer {
    [System.Collections.Generic.List[PSCustomObject]]$LogEntries
    [hashtable]$LogPatterns
    [System.Collections.Generic.Dictionary[string, int]]$PatternMatches
    [scriptblock]$AnomalyDetector
    [hashtable]$LogLevels

    LogAnalyzer() {
        $this.LogEntries = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.PatternMatches = [System.Collections.Generic.Dictionary[string, int]]::new()
        $this.LogLevels = @{
            "DEBUG" = 0
            "INFO" = 1
            "WARN" = 2
            "ERROR" = 3
            "CRITICAL" = 4
        }

        $this.InitializePatterns()
        $this.AnomalyDetector = $this.DefaultAnomalyDetector()
    }

    hidden [void]InitializePatterns() {
        $this.LogPatterns = @{
            # Patterns d'erreurs
            "Exception" = @{
                Pattern = "exception|error|fail"
                Severity = "ERROR"
                Category = "Error"
            }
            "AuthenticationFailure" = @{
                Pattern = "login.*fail|auth.*fail|unauthorized"
                Severity = "WARN"
                Category = "Security"
            }
            "PerformanceIssue" = @{
                Pattern = "timeout|slow|performance"
                Severity = "WARN"
                Category = "Performance"
            }
            "ResourceExhaustion" = @{
                Pattern = "out.*memory|disk.*full|connection.*pool"
                Severity = "CRITICAL"
                Category = "Resource"
            }
            "NetworkIssue" = @{
                Pattern = "connection.*refus|network.*unreachable|dns.*fail"
                Severity = "ERROR"
                Category = "Network"
            }
        }
    }

    [void]IngestLog([string]$logLine, [string]$source = "Unknown") {
        $entry = [PSCustomObject]@{
            Timestamp = Get-Date
            Source = $source
            RawMessage = $logLine
            ParsedLevel = $null
            Category = $null
            PatternMatches = @()
        }

        # Analyse du log
        $this.AnalyzeLogEntry($entry)

        $this.LogEntries.Add($entry)

        # Limitation de la taille (garder les 10000 dernières entrées)
        if ($this.LogEntries.Count -gt 10000) {
            $this.LogEntries.RemoveAt(0)
        }
    }

    hidden [void]AnalyzeLogEntry([PSCustomObject]$entry) {
        $message = $entry.RawMessage.ToLower()

        foreach ($patternName in $this.LogPatterns.Keys) {
            $pattern = $this.LogPatterns[$patternName]

            if ($message -match $pattern.Pattern) {
                $entry.PatternMatches += $patternName
                $entry.Category = $pattern.Category

                # Mise à jour du niveau si plus sévère
                if (-not $entry.ParsedLevel -or $this.LogLevels[$pattern.Severity] -gt $this.LogLevels[$entry.ParsedLevel]) {
                    $entry.ParsedLevel = $pattern.Severity
                }

                # Comptage des patterns
                if (-not $this.PatternMatches.ContainsKey($patternName)) {
                    $this.PatternMatches[$patternName] = 0
                }
                $this.PatternMatches[$patternName]++
            }
        }

        # Niveau par défaut
        if (-not $entry.ParsedLevel) {
            $entry.ParsedLevel = "INFO"
        }
    }

    [PSCustomObject[]]GetLogs([DateTime]$startTime, [DateTime]$endTime, [string]$level = $null, [string]$category = $null) {
        $logs = $this.LogEntries | Where-Object {
            $_.Timestamp -ge $startTime -and $_.Timestamp -le $endTime
        }

        if ($level) {
            $logs = $logs | Where-Object { $_.ParsedLevel -eq $level }
        }

        if ($category) {
            $logs = $logs | Where-Object { $_.Category -eq $category }
        }

        return $logs | Sort-Object Timestamp -Descending
    }

    [hashtable]AnalyzeLogPatterns([TimeSpan]$timeWindow) {
        $cutoff = (Get-Date) - $timeWindow
        $recentLogs = $this.LogEntries | Where-Object { $_.Timestamp -ge $cutoff }

        $analysis = @{
            TotalLogs = $recentLogs.Count
            LogsByLevel = @{}
            LogsByCategory = @{}
            TopPatterns = @()
            Anomalies = @()
            TimeWindow = $timeWindow
        }

        # Répartition par niveau
        $analysis.LogsByLevel = $recentLogs | Group-Object ParsedLevel | ForEach-Object {
            @{ $_.Name = $_.Count }
        }

        # Répartition par catégorie
        $analysis.LogsByCategory = $recentLogs | Group-Object Category | ForEach-Object {
            @{ $_.Name = $_.Count }
        }

        # Patterns les plus fréquents
        $analysis.TopPatterns = $this.PatternMatches.GetEnumerator() | Sort-Object Value -Descending | Select-Object -First 10 | ForEach-Object {
            @{ Pattern = $_.Key; Count = $_.Value }
        }

        # Détection d'anomalies
        $analysis.Anomalies = & $this.AnomalyDetector $recentLogs

        return $analysis
    }

    hidden [scriptblock]DefaultAnomalyDetector() {
        return {
            param($logs)

            $anomalies = @()

            # Anomalie: Pic d'erreurs
            $errorLogs = $logs | Where-Object { $_.ParsedLevel -in @("ERROR", "CRITICAL") }
            if ($errorLogs.Count -gt ($logs.Count * 0.1)) {  # Plus de 10% d'erreurs
                $anomalies += @{
                    Type = "ErrorSpike"
                    Description = "High error rate detected"
                    Severity = "Critical"
                    Evidence = "$($errorLogs.Count) errors out of $($logs.Count) total logs"
                }
            }

            # Anomalie: Nouveaux patterns d'erreur
            $recentPatterns = $logs | Where-Object { $_.Timestamp -gt (Get-Date).AddMinutes(-5) } | Select-Object -ExpandProperty PatternMatches | Select-Object -Unique
            $olderLogs = $logs | Where-Object { $_.Timestamp -le (Get-Date).AddMinutes(-5) -and $_.Timestamp -gt (Get-Date).AddMinutes(-15) }
            $olderPatterns = $olderLogs | Select-Object -ExpandProperty PatternMatches | Select-Object -Unique

            $newPatterns = $recentPatterns | Where-Object { $_ -notin $olderPatterns }
            if ($newPatterns) {
                $anomalies += @{
                    Type = "NewErrorPatterns"
                    Description = "New error patterns detected"
                    Severity = "Warning"
                    Evidence = "New patterns: $($newPatterns -join ', ')"
                }
            }

            return $anomalies
        }
    }

    [PSCustomObject[]]SearchLogs([string]$query, [DateTime]$startTime, [DateTime]$endTime) {
        $logs = $this.GetLogs($startTime, $endTime)

        return $logs | Where-Object {
            $_.RawMessage -match $query -or
            $_.Source -match $query -or
            ($_.PatternMatches -join ' ') -match $query
        }
    }

    [hashtable]GetLogStatistics([TimeSpan]$timeWindow) {
        $analysis = $this.AnalyzeLogPatterns($timeWindow)

        return @{
            TimeWindow = $timeWindow
            TotalLogs = $analysis.TotalLogs
            LogsPerMinute = [math]::Round($analysis.TotalLogs / $timeWindow.TotalMinutes, 2)
            ErrorRate = if ($analysis.TotalLogs -gt 0) {
                [math]::Round(($analysis.LogsByLevel["ERROR"] + $analysis.LogsByLevel["CRITICAL"]) / $analysis.TotalLogs * 100, 2)
            } else { 0 }
            MostCommonCategory = ($analysis.LogsByCategory.GetEnumerator() | Sort-Object Value -Descending | Select-Object -First 1).Key
            AnomaliesDetected = $analysis.Anomalies.Count
            PatternDiversity = $this.PatternMatches.Count
        }
    }

    [void]AddCustomPattern([string]$name, [hashtable]$pattern) {
        $this.LogPatterns[$name] = $pattern
        Write-Host "Custom log pattern '$name' added" -ForegroundColor Green
    }

    [void]SetAnomalyDetector([scriptblock]$detector) {
        $this.AnomalyDetector = $detector
        Write-Host "Custom anomaly detector set" -ForegroundColor Green
    }

    [string]GenerateLogReport([TimeSpan]$timeWindow = [TimeSpan]::FromHours(1)) {
        $stats = $this.GetLogStatistics($timeWindow)
        $analysis = $this.AnalyzeLogPatterns($timeWindow)

        $report = @"
LOG ANALYSIS REPORT
===================
Generated: $(Get-Date)
Time Window: $($timeWindow.TotalMinutes) minutes

STATISTICS
==========
Total Logs: $($stats.TotalLogs)
Logs/Minute: $($stats.LogsPerMinute)
Error Rate: $($stats.ErrorRate)%
Most Common Category: $($stats.MostCommonCategory)
Anomalies Detected: $($stats.AnomaliesDetected)

LOGS BY LEVEL
=============
$($analysis.LogsByLevel.GetEnumerator() | Sort-Object Value -Descending | ForEach-Object { "$($_.Key): $($_.Value)" } | Out-String)

LOGS BY CATEGORY
================
$($analysis.LogsByCategory.GetEnumerator() | Sort-Object Value -Descending | ForEach-Object { "$($_.Key): $($_.Value)" } | Out-String)

TOP PATTERNS
============
$($analysis.TopPatterns | ForEach-Object { "$($_.Pattern): $($_.Count) matches" } | Out-String)

ANOMALIES DETECTED
==================
$($analysis.Anomalies | ForEach-Object { "[$($_.Severity)] $($_.Type): $($_.Description)`n  $($_.Evidence)" } | Out-String)
"@

        return $report
    }
}

# Démonstration du système de monitoring
Write-Host "=== POWERSHELL MONITORING & OBSERVABILITY DEMO ===" -ForegroundColor Cyan

# Initialisation des systèmes
$metricsCollector = [MetricsCollector]::new()
$logAnalyzer = [LogAnalyzer]::new()

# Collecte initiale des métriques
$metricsCollector.CollectMetrics()

# Traitement des alertes
$metricsCollector.ProcessAlerts()

# Injection de logs de test
$testLogs = @(
    "2023-12-01 10:00:00 INFO Application started successfully",
    "2023-12-01 10:01:00 ERROR Database connection failed: timeout",
    "2023-12-01 10:02:00 WARN High memory usage detected: 85%",
    "2023-12-01 10:03:00 ERROR Authentication failed for user: admin",
    "2023-12-01 10:04:00 INFO User login successful",
    "2023-12-01 10:05:00 CRITICAL Out of memory exception",
    "2023-12-01 10:06:00 ERROR Network connection refused",
    "2023-12-01 10:07:00 WARN Slow query detected: 5.2 seconds"
)

foreach ($log in $testLogs) {
    $logAnalyzer.IngestLog($log, "TestApplication")
}

# Analyse des logs
$logAnalysis = $logAnalyzer.AnalyzeLogPatterns([TimeSpan]::FromHours(1))

# Rapport de métriques
$metricsReport = $metricsCollector.GenerateMetricsReport()
Write-Host "`n$metricsReport"

# Rapport d'analyse des logs
$logReport = $logAnalyzer.GenerateLogReport()
Write-Host "`n$logReport"

# Recherche dans les logs
$searchResults = $logAnalyzer.SearchLogs("error", (Get-Date).AddHours(-1), (Get-Date))
Write-Host "`n=== ERROR LOGS FOUND ===" -ForegroundColor Red
$searchResults | ForEach-Object { Write-Host "$($_.Timestamp): $($_.RawMessage)" }

# Tendances des métriques
$cpuTrend = $metricsCollector.CalculateTrends("CPU_Usage")
Write-Host "`n=== METRICS TRENDS ===" -ForegroundColor Yellow
Write-Host "CPU Usage Trend: $($cpuTrend.Trend)"
Write-Host "Memory Usage Trend: $($metricsCollector.CalculateTrends('Memory_Usage').Trend)"

Write-Host "`nMonitoring and observability demonstration completed!" -ForegroundColor Green
```

### 1.2 Tableaux de bord et visualisation

**Système de tableaux de bord interactifs :**
```powershell
# Framework de tableaux de bord interactifs
class DashboardFramework {
    [System.Collections.Generic.Dictionary[string, PSObject]]$Dashboards
    [hashtable]$DataSources
    [System.Collections.Generic.List[PSCustomObject]]$Widgets
    [scriptblock]$RenderEngine

    DashboardFramework() {
        $this.Dashboards = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.DataSources = @{}
        $this.Widgets = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.RenderEngine = {
            param($dashboard)

            Write-Host "=== $($dashboard.Title.ToUpper()) ===" -ForegroundColor Cyan
            Write-Host "Generated: $(Get-Date)" -ForegroundColor Gray
            Write-Host ""

            foreach ($widget in $dashboard.Widgets) {
                & $this.RenderWidget $widget
                Write-Host ""
            }
        }
    }

    [PSObject]CreateDashboard([string]$name, [string]$title, [hashtable]$config = @{}) {
        $dashboard = [PSCustomObject]@{
            Name = $name
            Title = $title
            Description = $config.Description ?? ""
            RefreshInterval = $config.RefreshInterval ?? 300
            Widgets = [System.Collections.Generic.List[PSCustomObject]]::new()
            Layout = $config.Layout ?? "Vertical"
            Created = Get-Date
        }

        $this.Dashboards[$name] = $dashboard
        return $dashboard
    }

    [void]AddWidget([string]$dashboardName, [hashtable]$widgetConfig) {
        if (-not $this.Dashboards.ContainsKey($dashboardName)) {
            throw "Dashboard '$dashboardName' not found"
        }

        $widget = [PSCustomObject]@{
            Id = [guid]::NewGuid().ToString()
            Type = $widgetConfig.Type ?? "Text"
            Title = $widgetConfig.Title ?? "Untitled Widget"
            DataSource = $widgetConfig.DataSource
            Config = $widgetConfig.Config ?? @{}
            Position = $widgetConfig.Position ?? $this.Dashboards[$dashboardName].Widgets.Count
            Size = $widgetConfig.Size ?? "Medium"
        }

        $this.Dashboards[$dashboardName].Widgets.Add($widget)
    }

    [void]AddDataSource([string]$name, [scriptblock]$dataProvider) {
        $this.DataSources[$name] = $dataProvider
    }

    [PSCustomObject]GetDashboardData([string]$dashboardName) {
        if (-not $this.Dashboards.ContainsKey($dashboardName)) {
            throw "Dashboard '$dashboardName' not found"
        }

        $dashboard = $this.Dashboards[$dashboardName]

        # Collecte des données pour tous les widgets
        $dashboardData = [PSCustomObject]@{
            Dashboard = $dashboard
            WidgetData = @{}
            GeneratedAt = Get-Date
        }

        foreach ($widget in $dashboard.Widgets) {
            if ($widget.DataSource -and $this.DataSources.ContainsKey($widget.DataSource)) {
                try {
                    $data = & $this.DataSources[$widget.DataSource]
                    $dashboardData.WidgetData[$widget.Id] = $data
                } catch {
                    $dashboardData.WidgetData[$widget.Id] = @{ Error = $_.Exception.Message }
                }
            }
        }

        return $dashboardData
    }

    [void]RenderDashboard([string]$dashboardName) {
        $dashboardData = $this.GetDashboardData($dashboardName)
        & $this.RenderEngine $dashboardData
    }

    hidden [void]RenderWidget([PSCustomObject]$widget, [hashtable]$data) {
        Write-Host "[$($widget.Title)]" -ForegroundColor Yellow

        switch ($widget.Type) {
            "Metric" {
                $this.RenderMetricWidget($widget, $data)
            }
            "Chart" {
                $this.RenderChartWidget($widget, $data)
            }
            "Table" {
                $this.RenderTableWidget($widget, $data)
            }
            "Text" {
                $this.RenderTextWidget($widget, $data)
            }
            "Alert" {
                $this.RenderAlertWidget($widget, $data)
            }
            default {
                Write-Host "Unknown widget type: $($widget.Type)" -ForegroundColor Red
            }
        }
    }

    hidden [void]RenderMetricWidget([PSCustomObject]$widget, [hashtable]$data) {
        if ($data.Error) {
            Write-Host "Error: $($data.Error)" -ForegroundColor Red
            return
        }

        $value = $data.Value ?? 0
        $unit = $widget.Config.Unit ?? ""
        $thresholds = $widget.Config.Thresholds ?? @{}

        $color = "White"
        if ($thresholds.Critical -and $value -ge $thresholds.Critical) {
            $color = "Red"
        } elseif ($thresholds.Warning -and $value -ge $thresholds.Warning) {
            $color = "Yellow"
        } elseif ($thresholds.Good -and $value -le $thresholds.Good) {
            $color = "Green"
        }

        Write-Host "$value $unit" -ForegroundColor $color

        if ($widget.Config.ShowTrend) {
            $trend = $data.Trend ?? "Unknown"
            Write-Host "Trend: $trend" -ForegroundColor Gray
        }
    }

    hidden [void]RenderChartWidget([PSCustomObject]$widget, [hashtable]$data) {
        if ($data.Error) {
            Write-Host "Error: $($data.Error)" -ForegroundColor Red
            return
        }

        $values = $data.Values ?? @()
        $maxValue = ($values | Measure-Object -Maximum).Maximum ?? 100
        $chartWidth = $widget.Config.Width ?? 40

        Write-Host "Chart: $($widget.Config.ChartType ?? 'Bar')" -ForegroundColor Gray

        for ($i = 0; $i -lt $values.Count; $i++) {
            $value = $values[$i]
            $barLength = [math]::Round(($value / $maxValue) * $chartWidth)
            $bar = "█" * $barLength
            $label = $data.Labels[$i] ?? "Item $i"

            Write-Host ("{0,-15}: {1} ({2})" -f $label, $bar, $value) -ForegroundColor Blue
        }
    }

    hidden [void]RenderTableWidget([PSCustomObject]$widget, [hashtable]$data) {
        if ($data.Error) {
            Write-Host "Error: $($data.Error)" -ForegroundColor Red
            return
        }

        $items = $data.Items ?? @()

        if ($items.Count -eq 0) {
            Write-Host "No data available" -ForegroundColor Gray
            return
        }

        # Déterminer les colonnes
        $columns = $items[0].PSObject.Properties.Name

        # En-tête
        $header = $columns -join " | "
        Write-Host $header -ForegroundColor Cyan
        Write-Host ("-" * $header.Length) -ForegroundColor Gray

        # Données
        foreach ($item in $items) {
            $row = $columns | ForEach-Object { $item.$_ } | Join-String -Separator " | "
            Write-Host $row
        }
    }

    hidden [void]RenderTextWidget([PSCustomObject]$widget, [hashtable]$data) {
        $text = $data.Text ?? "No content"
        $color = $widget.Config.Color ?? "White"

        Write-Host $text -ForegroundColor $color
    }

    hidden [void]RenderAlertWidget([PSCustomObject]$widget, [hashtable]$data) {
        $alerts = $data.Alerts ?? @()

        if ($alerts.Count -eq 0) {
            Write-Host "✓ No active alerts" -ForegroundColor Green
            return
        }

        foreach ($alert in $alerts) {
            $color = switch ($alert.Severity) {
                "Critical" { "Red" }
                "Warning" { "Yellow" }
                "Info" { "Blue" }
                default { "White" }
            }

            Write-Host "⚠️  $($alert.Message)" -ForegroundColor $color
        }
    }

    [void]StartAutoRefresh([string]$dashboardName, [int]$intervalSeconds = 60) {
        Write-Host "Starting auto-refresh for dashboard '$dashboardName' (every $intervalSeconds seconds)" -ForegroundColor Green

        $job = Start-Job -ScriptBlock {
            param($framework, $dashboard, $interval)

            while ($true) {
                Clear-Host
                $framework.RenderDashboard($dashboard)
                Start-Sleep -Seconds $interval
            }
        } -ArgumentList $this, $dashboardName, $intervalSeconds

        Write-Host "Auto-refresh started. Job ID: $($job.Id)" -ForegroundColor Green
    }

    [void]ExportDashboard([string]$dashboardName, [string]$format = "JSON") {
        $dashboardData = $this.GetDashboardData($dashboardName)

        switch ($format.ToUpper()) {
            "JSON" {
                $dashboardData | ConvertTo-Json -Depth 10 | Out-File -FilePath "$dashboardName`_dashboard.json" -Encoding UTF8
            }
            "HTML" {
                $this.ExportToHtml($dashboardName, $dashboardData)
            }
            "XML" {
                $dashboardData | ConvertTo-Xml | Out-File -FilePath "$dashboardName`_dashboard.xml" -Encoding UTF8
            }
        }

        Write-Host "Dashboard exported to $dashboardName`_dashboard.$($format.ToLower())" -ForegroundColor Green
    }

    hidden [void]ExportToHtml([string]$dashboardName, [PSCustomObject]$dashboardData) {
        $html = @"
<!DOCTYPE html>
<html>
<head>
    <title>$($dashboardData.Dashboard.Title)</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; background-color: #f5f5f5; }
        .dashboard { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
        .widget { margin: 20px 0; padding: 15px; border: 1px solid #ddd; border-radius: 5px; }
        .widget-title { font-weight: bold; color: #333; margin-bottom: 10px; }
        .metric { font-size: 2em; font-weight: bold; color: #007acc; }
        .alert { color: #d9534f; }
        .warning { color: #f0ad4e; }
        .success { color: #5cb85c; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <div class="dashboard">
        <h1>$($dashboardData.Dashboard.Title)</h1>
        <p>$($dashboardData.Dashboard.Description)</p>
        <p><small>Generated: $($dashboardData.GeneratedAt)</small></p>
"@

        foreach ($widget in $dashboardData.Dashboard.Widgets) {
            $data = $dashboardData.WidgetData[$widget.Id]

            $html += @"
        <div class="widget">
            <div class="widget-title">$($widget.Title)</div>
"@

            switch ($widget.Type) {
                "Metric" {
                    $html += "<div class=`"metric`">$($data.Value) $($widget.Config.Unit)</div>"
                }
                "Table" {
                    $html += "<table>"
                    if ($data.Items -and $data.Items.Count -gt 0) {
                        $columns = $data.Items[0].PSObject.Properties.Name
                        $html += "<tr>" + ($columns | ForEach-Object { "<th>$_</th>" }) + "</tr>"
                        foreach ($item in $data.Items) {
                            $html += "<tr>" + ($columns | ForEach-Object { "<td>$($item.$_)</td>" }) + "</tr>"
                        }
                    }
                    $html += "</table>"
                }
                "Text" {
                    $html += "<p>$($data.Text)</p>"
                }
            }

            $html += "</div>"
        }

        $html += @"
    </div>
</body>
</html>
"@

        $html | Out-File -FilePath "$dashboardName`_dashboard.html" -Encoding UTF8
    }
}

# Création d'un tableau de bord complet
$dashboard = [DashboardFramework]::new()

# Sources de données
$dashboard.AddDataSource("SystemMetrics", {
    $cpu = Get-Counter '\Processor(_Total)\% Processor Time' -ErrorAction SilentlyContinue
    $memory = Get-Counter '\Memory\% Committed Bytes In Use' -ErrorAction SilentlyContinue

    return @{
        CPU_Value = if ($cpu) { [math]::Round($cpu.CounterSamples[0].CookedValue, 1) } else { 0 }
        Memory_Value = if ($memory) { [math]::Round($memory.CounterSamples[0].CookedValue, 1) } else { 0 }
        CPU_Trend = "Stable"
        Memory_Trend = "Stable"
    }
})

$dashboard.AddDataSource("ServiceStatus", {
    $services = Get-Service
    $running = ($services | Where-Object { $_.Status -eq "Running" }).Count
    $total = $services.Count

    return @{
        Items = @(
            @{ Service = "Total Services"; Count = $total; Status = "Info" }
            @{ Service = "Running Services"; Count = $running; Status = "Success" }
            @{ Service = "Stopped Services"; Count = ($total - $running); Status = "Warning" }
        )
    }
})

$dashboard.AddDataSource("TopProcesses", {
    $processes = Get-Process | Sort-Object CPU -Descending | Select-Object -First 5

    return @{
        Items = $processes | ForEach-Object {
            @{
                Name = $_.ProcessName
                CPU = [math]::Round($_.CPU, 1)
                Memory_MB = [math]::Round($_.WorkingSet / 1MB, 1)
            }
        }
    }
})

$dashboard.AddDataSource("RecentAlerts", {
    # Simulation d'alertes récentes
    return @{
        Alerts = @(
            @{ Message = "High CPU usage detected"; Severity = "Warning"; Timestamp = (Get-Date).AddMinutes(-5) }
            @{ Message = "Service restart required"; Severity = "Info"; Timestamp = (Get-Date).AddMinutes(-10) }
        )
    }
})

# Création du tableau de bord
$mainDashboard = $dashboard.CreateDashboard("SystemOverview", "System Monitoring Dashboard", @{
    Description = "Real-time monitoring of system health and performance"
    RefreshInterval = 30
})

# Ajout des widgets
$dashboard.AddWidget("SystemOverview", @{
    Type = "Metric"
    Title = "CPU Usage"
    DataSource = "SystemMetrics"
    Config = @{
        Unit = "%"
        Thresholds = @{ Warning = 70; Critical = 90 }
        ShowTrend = $true
    }
})

$dashboard.AddWidget("SystemOverview", @{
    Type = "Metric"
    Title = "Memory Usage"
    DataSource = "SystemMetrics"
    Config = @{
        Unit = "%"
        Thresholds = @{ Warning = 80; Critical = 95 }
        ShowTrend = $true
    }
})

$dashboard.AddWidget("SystemOverview", @{
    Type = "Table"
    Title = "Service Status"
    DataSource = "ServiceStatus"
})

$dashboard.AddWidget("SystemOverview", @{
    Type = "Table"
    Title = "Top Processes"
    DataSource = "TopProcesses"
})

$dashboard.AddWidget("SystemOverview", @{
    Type = "Alert"
    Title = "Recent Alerts"
    DataSource = "RecentAlerts"
})

$dashboard.AddWidget("SystemOverview", @{
    Type = "Text"
    Title = "System Summary"
    DataSource = $null
    Config = @{
        Text = "System is running normally with all critical services operational."
        Color = "Green"
    }
})

# Rendu du tableau de bord
$dashboard.RenderDashboard("SystemOverview")

# Export du tableau de bord
$dashboard.ExportDashboard("SystemOverview", "HTML")

Write-Host "`nDashboard demonstration completed!" -ForegroundColor Green
Write-Host "HTML dashboard exported to: SystemOverview_dashboard.html" -ForegroundColor Green
```

## Section 2 : Tracing et distributed tracing

### 2.1 Système de tracing distribué

**Implémentation d'un système de tracing avancé :**
```powershell
# Framework de tracing distribué PowerShell
class DistributedTracer {
    [System.Collections.Generic.List[PSCustomObject]]$Traces
    [System.Collections.Generic.Dictionary[string, PSObject]]$ActiveSpans
    [string]$ServiceName
    [hashtable]$TraceConfig
    [scriptblock]$Exporter

    DistributedTracer([string]$serviceName) {
        $this.ServiceName = $serviceName
        $this.Traces = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.ActiveSpans = [System.Collections.Generic.Dictionary[string, PSObject]]::new()

        $this.TraceConfig = @{
            SampleRate = 1.0  # 100% sampling
            MaxTraceDuration = [TimeSpan]::FromMinutes(5)
            MaxSpansPerTrace = 1000
            ExportInterval = [TimeSpan]::FromSeconds(30)
        }

        $this.Exporter = {
            param($traces)
            # Export par défaut vers console
            foreach ($trace in $traces) {
                Write-Host "Trace exported: $($trace.TraceId)" -ForegroundColor Gray
            }
        }
    }

    [PSObject]StartTrace([string]$operationName, [hashtable]$tags = @{}) {
        $traceId = [guid]::NewGuid().ToString()
        $spanId = [guid]::NewGuid().ToString()

        $trace = [PSCustomObject]@{
            TraceId = $traceId
            RootSpanId = $spanId
            ServiceName = $this.ServiceName
            OperationName = $operationName
            StartTime = Get-Date
            EndTime = $null
            Duration = $null
            Status = "Running"
            Tags = $tags
            Spans = [System.Collections.Generic.List[PSObject]]::new()
            Error = $null
        }

        # Span racine
        $rootSpan = [PSCustomObject]@{
            SpanId = $spanId
            ParentSpanId = $null
            OperationName = $operationName
            StartTime = Get-Date
            EndTime = $null
            Duration = $null
            Tags = $tags
            Events = [System.Collections.Generic.List[PSCustomObject]]::new()
            Status = "Running"
        }

        $trace.Spans.Add($rootSpan)
        $this.ActiveSpans[$spanId] = $rootSpan
        $this.Traces.Add($trace)

        return [PSCustomObject]@{
            TraceId = $traceId
            SpanId = $spanId
            Trace = $trace
            Span = $rootSpan
        }
    }

    [PSObject]StartSpan([string]$traceId, [string]$operationName, [string]$parentSpanId = $null, [hashtable]$tags = @{}) {
        $spanId = [guid]::NewGuid().ToString()

        $span = [PSCustomObject]@{
            SpanId = $spanId
            ParentSpanId = $parentSpanId
            OperationName = $operationName
            StartTime = Get-Date
            EndTime = $null
            Duration = $null
            Tags = $tags
            Events = [System.Collections.Generic.List[PSCustomObject]]::new()
            Status = "Running"
        }

        $this.ActiveSpans[$spanId] = $span

        # Ajout au trace
        $trace = $this.Traces | Where-Object { $_.TraceId -eq $traceId } | Select-Object -First 1
        if ($trace) {
            $trace.Spans.Add($span)
        }

        return $span
    }

    [void]AddSpanEvent([string]$spanId, [string]$eventName, [hashtable]$attributes = @{}) {
        if (-not $this.ActiveSpans.ContainsKey($spanId)) {
            Write-Warning "Span '$spanId' not found"
            return
        }

        $span = $this.ActiveSpans[$spanId]

        $event = [PSCustomObject]@{
            Name = $eventName
            Timestamp = Get-Date
            Attributes = $attributes
        }

        $span.Events.Add($event)
    }

    [void]SetSpanTag([string]$spanId, [string]$key, $value) {
        if (-not $this.ActiveSpans.ContainsKey($spanId)) {
            Write-Warning "Span '$spanId' not found"
            return
        }

        $span = $this.ActiveSpans[$spanId]
        $span.Tags[$key] = $value
    }

    [void]EndSpan([string]$spanId, [string]$status = "Completed", [string]$error = $null) {
        if (-not $this.ActiveSpans.ContainsKey($spanId)) {
            Write-Warning "Span '$spanId' not found"
            return
        }

        $span = $this.ActiveSpans[$spanId]
        $span.EndTime = Get-Date
        $span.Duration = $span.EndTime - $span.StartTime
        $span.Status = $status

        if ($error) {
            $span.Tags["error"] = $true
            $span.Tags["error.message"] = $error
        }

        # Vérification de la limite de spans par trace
        $trace = $this.Traces | Where-Object { $_.Spans -contains $span } | Select-Object -First 1
        if ($trace -and $trace.Spans.Count -ge $this.TraceConfig.MaxSpansPerTrace) {
            Write-Warning "Trace $($trace.TraceId) reached maximum spans limit"
        }
    }

    [void]EndTrace([string]$traceId, [string]$status = "Completed", [string]$error = $null) {
        $trace = $this.Traces | Where-Object { $_.TraceId -eq $traceId } | Select-Object -First 1

        if (-not $trace) {
            Write-Warning "Trace '$traceId' not found"
            return
        }

        $trace.EndTime = Get-Date
        $trace.Duration = $trace.EndTime - $trace.StartTime
        $trace.Status = $status

        if ($error) {
            $trace.Error = $error
        }

        # Fermeture de tous les spans actifs de ce trace
        $activeSpans = $trace.Spans | Where-Object { $_.Status -eq "Running" }
        foreach ($span in $activeSpans) {
            $this.EndSpan($span.SpanId, "AutoClosed")
        }

        # Vérification de la durée maximale
        if ($trace.Duration -gt $this.TraceConfig.MaxTraceDuration) {
            Write-Warning "Trace $($trace.TraceId) exceeded maximum duration"
        }

        # Export du trace si nécessaire
        if ((Get-Random) -le $this.TraceConfig.SampleRate) {
            & $this.Exporter @($trace)
        }
    }

    [PSCustomObject[]]GetActiveTraces() {
        return $this.Traces | Where-Object { $_.Status -eq "Running" }
    }

    [PSCustomObject[]]GetTraces([DateTime]$startTime, [DateTime]$endTime, [string]$status = $null) {
        $traces = $this.Traces | Where-Object {
            $_.StartTime -ge $startTime -and $_.StartTime -le $endTime
        }

        if ($status) {
            $traces = $traces | Where-Object { $_.Status -eq $status }
        }

        return $traces | Sort-Object StartTime -Descending
    }

    [hashtable]AnalyzeTracePerformance([TimeSpan]$timeWindow) {
        $cutoff = (Get-Date) - $timeWindow
        $recentTraces = $this.Traces | Where-Object { $_.StartTime -ge $cutoff }

        $analysis = @{
            TimeWindow = $timeWindow
            TotalTraces = $recentTraces.Count
            AverageDuration = $null
            LongestTrace = $null
            ShortestTrace = $null
            ErrorRate = 0
            Throughput = 0
            TopOperations = @()
            SpanCountDistribution = @()
        }

        if ($recentTraces.Count -gt 0) {
            # Durée moyenne
            $totalDuration = ($recentTraces | Where-Object { $_.Duration } | ForEach-Object { $_.Duration.TotalSeconds } | Measure-Object -Sum).Sum
            $completedTraces = $recentTraces | Where-Object { $_.Duration }
            $analysis.AverageDuration = [TimeSpan]::FromSeconds($totalDuration / $completedTraces.Count)

            # Trace la plus longue/courte
            $analysis.LongestTrace = $completedTraces | Sort-Object { $_.Duration.TotalSeconds } -Descending | Select-Object -First 1
            $analysis.ShortestTrace = $completedTraces | Sort-Object { $_.Duration.TotalSeconds } | Select-Object -First 1

            # Taux d'erreur
            $errorTraces = $recentTraces | Where-Object { $_.Status -eq "Error" -or $_.Error }
            $analysis.ErrorRate = [math]::Round(($errorTraces.Count / $recentTraces.Count) * 100, 2)

            # Débit (traces par seconde)
            $analysis.Throughput = [math]::Round($recentTraces.Count / $timeWindow.TotalSeconds, 2)

            # Opérations les plus fréquentes
            $analysis.TopOperations = $recentTraces | Group-Object OperationName | Sort-Object Count -Descending | Select-Object -First 10 | ForEach-Object {
                @{ Operation = $_.Name; Count = $_.Count; AvgDuration = [math]::Round(($_.Group | Where-Object { $_.Duration } | ForEach-Object { $_.Duration.TotalSeconds } | Measure-Object -Average).Average, 2) }
            }

            # Distribution du nombre de spans
            $analysis.SpanCountDistribution = $recentTraces | Group-Object { $_.Spans.Count } | Sort-Object Name | ForEach-Object {
                @{ SpanCount = [int]$_.Name; TraceCount = $_.Count }
            }
        }

        return $analysis
    }

    [void]SetExporter([scriptblock]$exporter) {
        $this.Exporter = $exporter
    }

    [void]ConfigureTracing([hashtable]$config) {
        foreach ($key in $config.Keys) {
            if ($this.TraceConfig.ContainsKey($key)) {
                $this.TraceConfig[$key] = $config[$key]
            }
        }
    }

    [string]GenerateTraceReport([TimeSpan]$timeWindow = [TimeSpan]::FromHours(1)) {
        $analysis = $this.AnalyzeTracePerformance($timeWindow)

        $report = @"
DISTRIBUTED TRACING REPORT
==========================
Service: $($this.ServiceName)
Generated: $(Get-Date)
Time Window: $($timeWindow.TotalMinutes) minutes

PERFORMANCE METRICS
===================
Total Traces: $($analysis.TotalTraces)
Throughput: $($analysis.Throughput) traces/second
Error Rate: $($analysis.ErrorRate)%

$($analysis.AverageDuration ? "Average Duration: $([math]::Round($analysis.AverageDuration.TotalSeconds, 2)) seconds" : "No completed traces")

TRACE STATISTICS
================
$($analysis.LongestTrace ? "Longest Trace: $($analysis.LongestTrace.OperationName) ($([math]::Round($analysis.LongestTrace.Duration.TotalSeconds, 2))s)" : "No trace data")
$($analysis.ShortestTrace ? "Shortest Trace: $($analysis.ShortestTrace.OperationName) ($([math]::Round($analysis.ShortestTrace.Duration.TotalSeconds, 2))s)" : "No trace data")

TOP OPERATIONS
==============
$($analysis.TopOperations | ForEach-Object { "$($_.Operation): $($_.Count) executions, avg $($_.AvgDuration)s" } | Out-String)

SPAN DISTRIBUTION
=================
$($analysis.SpanCountDistribution | ForEach-Object { "$($_.SpanCount) spans: $($_.TraceCount) traces" } | Out-String)
"@

        return $report
    }

    [void]StartPeriodicExport() {
        Write-Host "Starting periodic trace export..." -ForegroundColor Green

        $job = Start-Job -ScriptBlock {
            param($tracer)

            while ($true) {
                $tracesToExport = $tracer.Traces | Where-Object { $_.Status -ne "Running" -and -not $_.Exported }
                if ($tracesToExport) {
                    & $tracer.Exporter $tracesToExport
                    foreach ($trace in $tracesToExport) {
                        $trace | Add-Member -MemberType NoteProperty -Name "Exported" -Value $true -Force
                    }
                }

                Start-Sleep -Seconds $tracer.TraceConfig.ExportInterval.TotalSeconds
            }
        } -ArgumentList $this

        Write-Host "Periodic export started. Job ID: $($job.Id)" -ForegroundColor Green
    }
}

# Fonctions utilitaires pour le tracing
function Start-TracedOperation {
    param(
        [Parameter(Mandatory=$true)]
        [string]$OperationName,

        [Parameter(Mandatory=$true)]
        [scriptblock]$Operation,

        [hashtable]$Tags = @(),

        [switch]$AutoEnd
    )

    if (-not $global:DistributedTracer) {
        Write-Warning "Distributed tracer not initialized. Use Initialize-Tracing first."
        return & $Operation
    }

    $traceContext = $global:DistributedTracer.StartTrace($OperationName, $Tags)

    try {
        $result = & $Operation
        $global:DistributedTracer.EndTrace($traceContext.TraceId, "Completed")
        return $result
    } catch {
        $global:DistributedTracer.EndTrace($traceContext.TraceId, "Error", $_.Exception.Message)
        throw
    }
}

function Start-TracedSpan {
    param(
        [Parameter(Mandatory=$true)]
        [string]$TraceId,

        [Parameter(Mandatory=$true)]
        [string]$OperationName,

        [Parameter(Mandatory=$true)]
        [scriptblock]$Operation,

        [hashtable]$Tags = @(),

        [string]$ParentSpanId = $null
    )

    if (-not $global:DistributedTracer) {
        return & $Operation
    }

    $span = $global:DistributedTracer.StartSpan($TraceId, $OperationName, $ParentSpanId, $Tags)

    try {
        $result = & $Operation
        $global:DistributedTracer.EndSpan($span.SpanId, "Completed")
        return $result
    } catch {
        $global:DistributedTracer.EndSpan($span.SpanId, "Error", $_.Exception.Message)
        throw
    }
}

function Add-TracingEvent {
    param(
        [Parameter(Mandatory=$true)]
        [string]$SpanId,

        [Parameter(Mandatory=$true)]
        [string]$EventName,

        [hashtable]$Attributes = @()
    )

    if ($global:DistributedTracer) {
        $global:DistributedTracer.AddSpanEvent($SpanId, $EventName, $Attributes)
    }
}

function Initialize-Tracing {
    param(
        [Parameter(Mandatory=$true)]
        [string]$ServiceName,

        [hashtable]$Configuration = @()
    )

    $global:DistributedTracer = [DistributedTracer]::new($ServiceName)

    if ($Configuration) {
        $global:DistributedTracer.ConfigureTracing($Configuration)
    }

    # Exporter vers fichier JSON
    $global:DistributedTracer.SetExporter({
        param($traces)

        foreach ($trace in $traces) {
            $fileName = "trace_$($trace.TraceId).json"
            $trace | ConvertTo-Json -Depth 10 | Out-File -FilePath $fileName -Encoding UTF8
            Write-Host "Trace exported to: $fileName" -ForegroundColor Gray
        }
    })

    Write-Host "Distributed tracing initialized for service: $ServiceName" -ForegroundColor Green
}

# Démonstration du système de tracing distribué
Write-Host "=== DISTRIBUTED TRACING DEMONSTRATION ===" -ForegroundColor Cyan

# Initialisation du tracing
Initialize-Tracing -ServiceName "PowerShellService" -Configuration @{
    SampleRate = 1.0
    MaxTraceDuration = [TimeSpan]::FromMinutes(10)
}

# Simulation d'opérations tracées
Start-TracedOperation -OperationName "ProcessUserRequest" -Tags @{ UserId = "12345"; Endpoint = "/api/users" } -ScriptBlock {
    Write-Host "Processing user request..." -ForegroundColor Blue

    # Sous-opération tracée
    Start-TracedSpan -TraceId $traceContext.TraceId -OperationName "ValidateInput" -ScriptBlock {
        Add-TracingEvent -SpanId $span.SpanId -EventName "InputValidationStarted" -Attributes @{ InputSize = 1024 }
        Start-Sleep -Milliseconds 100  # Simulation
        Add-TracingEvent -SpanId $span.SpanId -EventName "InputValidationCompleted" -Attributes @{ IsValid = $true }
    }

    Start-TracedSpan -TraceId $traceContext.TraceId -OperationName "DatabaseQuery" -ScriptBlock {
        Add-TracingEvent -SpanId $span.SpanId -EventName "QueryStarted" -Attributes @{ Query = "SELECT * FROM users" }
        Start-Sleep -Milliseconds 200  # Simulation
        Add-TracingEvent -SpanId $span.SpanId -EventName "QueryCompleted" -Attributes @{ RowCount = 1 }
    }

    Start-TracedSpan -TraceId $traceContext.TraceId -OperationName "SendResponse" -ScriptBlock {
        Add-TracingEvent -SpanId $span.SpanId -EventName "ResponseStarted" -Attributes @{ ResponseSize = 512 }
        Start-Sleep -Milliseconds 50  # Simulation
        Add-TracingEvent -SpanId $span.SpanId -EventName "ResponseCompleted" -Attributes @{ StatusCode = 200 }
    }
}

# Simulation d'une opération avec erreur
Start-TracedOperation -OperationName "FailedOperation" -Tags @{ Component = "Database" } -ScriptBlock {
    Write-Host "Starting operation that will fail..." -ForegroundColor Yellow

    Start-TracedSpan -TraceId $traceContext.TraceId -OperationName "RiskyOperation" -ScriptBlock {
        Add-TracingEvent -SpanId $span.SpanId -EventName "OperationStarted"
        Start-Sleep -Milliseconds 150

        # Simulation d'une erreur
        throw "Database connection timeout"
    }
}

# Analyse des performances
$performanceAnalysis = $global:DistributedTracer.AnalyzeTracePerformance([TimeSpan]::FromHours(1))

Write-Host "`n=== TRACING PERFORMANCE ANALYSIS ===" -ForegroundColor Yellow
Write-Host "Total traces: $($performanceAnalysis.TotalTraces)"
Write-Host "Error rate: $($performanceAnalysis.ErrorRate)%"
Write-Host "Throughput: $($performanceAnalysis.Throughput) traces/second"

if ($performanceAnalysis.AverageDuration) {
    Write-Host "Average trace duration: $([math]::Round($performanceAnalysis.AverageDuration.TotalSeconds, 2)) seconds"
}

# Rapport de tracing
$traceReport = $global:DistributedTracer.GenerateTraceReport()
Write-Host "`n$traceReport"

# Démarrage de l'export périodique
$global:DistributedTracer.StartPeriodicExport()

Write-Host "`nDistributed tracing demonstration completed!" -ForegroundColor Green
Write-Host "Trace files have been exported to the current directory." -ForegroundColor Green
```

## Conclusion : PowerShell comme observateur omniprésent

PowerShell transcende son rôle d'outil d'administration pour devenir l'observateur ultime des systèmes complexes. En intégrant métriques temps réel, analyse de logs intelligente, tableaux de bord interactifs, et tracing distribué, PowerShell crée une couche d'observabilité d'une profondeur et d'une intelligence sans précédent.

Dans le prochain chapitre, nous explorerons les intégrations avec l'Intelligence Artificielle et le Machine Learning, ouvrant PowerShell aux possibilités de l'automatisation cognitive.

---

**Exercice pratique :** Créez une plateforme d'observabilité complète avec PowerShell qui inclut :
1. Un système de collecte de métriques multi-sources avec alertes intelligentes
2. Un analyseur de logs avec détection d'anomalies et patterns personnalisés
3. Des tableaux de bord interactifs avec export HTML/JSON
4. Un système de tracing distribué avec analyse de performance
5. Des rapports automatiques et monitoring en temps réel

**Challenge avancé :** Développez une plateforme d'observabilité qui :
- Prédit les pannes système avec l'analyse prédictive
- Fournit des recommandations d'optimisation automatiques
- Intègre le monitoring des applications conteneurisées
- Supporte l'observabilité multi-cloud
- Implémente l'analyse de traces pour la détection de bottlenecks

**Réflexion :** Comment l'observabilité transforme-t-elle la gestion des systèmes ? L'observation passive suffit-elle, ou devons-nous tendre vers l'observabilité proactive et prescriptive ? Quelle est la place de PowerShell dans l'écosystème des outils d'observabilité modernes comme Prometheus, Jaeger, et ELK ?
