# Chapitre 134 - Gestion avancée des erreurs et exceptions PowerShell

> "La gestion d'erreurs n'est pas seulement la prévention des crashes, c'est l'art de créer des systèmes résilients qui apprennent de leurs échecs et s'améliorent continuellement." - Citation inspirée des principes de résilience logicielle

## Introduction : Les erreurs comme opportunités d'apprentissage

La gestion avancée des erreurs PowerShell transcende la simple instruction `try-catch` pour embrasser une philosophie de résilience systémique. En traitant les erreurs comme des signaux précieux sur l'état du système, les scripts PowerShell peuvent devenir auto-adaptatifs, auto-guérisseurs et auto-optimisants.

Dans ce chapitre, nous explorerons les mécanismes sophistiqués de gestion d'erreurs, les patterns de récupération, et les stratégies de résilience avancées.

## Section 1 : Architecture de gestion d'erreurs

### 1.1 Modèle d'erreur PowerShell étendu

**Compréhension profonde des objets d'erreur :**
```powershell
# Analyse détaillée des objets d'erreur PowerShell
function Analyze-ErrorObject {
    param([System.Management.Automation.ErrorRecord]$ErrorRecord)

    $analysis = @{
        ExceptionType = $ErrorRecord.Exception.GetType().FullName
        ExceptionMessage = $ErrorRecord.Exception.Message
        ErrorId = $ErrorRecord.FullyQualifiedErrorId
        Category = $ErrorRecord.CategoryInfo.Category
        Target = $ErrorRecord.CategoryInfo.TargetName
        ScriptStackTrace = $ErrorRecord.ScriptStackTrace
        PipelinePosition = $ErrorRecord.InvocationInfo.PipelinePosition
        ScriptLineNumber = $ErrorRecord.InvocationInfo.ScriptLineNumber
        ScriptName = $ErrorRecord.InvocationInfo.ScriptName
        CommandOrigin = $ErrorRecord.InvocationInfo.CommandOrigin
        HistoryId = $ErrorRecord.InvocationInfo.HistoryId
    }

    # Analyse de la pile d'appels
    $callStack = [System.Diagnostics.StackTrace]::new($true)
    $analysis.CallStack = for ($i = 0; $i -lt $callStack.FrameCount; $i++) {
        $frame = $callStack.GetFrame($i)
        if ($frame.GetMethod().Name -ne "Analyze-ErrorObject") {
            @{
                Method = $frame.GetMethod().Name
                FileName = $frame.GetFileName()
                LineNumber = $frame.GetFileLineNumber()
                ColumnNumber = $frame.GetFileColumnNumber()
            }
        }
    }

    # Classification de l'erreur
    $analysis.Classification = Get-ErrorClassification -ErrorRecord $ErrorRecord

    # Suggestions de récupération
    $analysis.RecoverySuggestions = Get-RecoverySuggestions -ErrorRecord $ErrorRecord

    return $analysis
}

function Get-ErrorClassification {
    param([System.Management.Automation.ErrorRecord]$ErrorRecord)

    $classification = @{
        Severity = "Unknown"
        Category = "Generic"
        IsTransient = $false
        IsRecoverable = $false
        RequiresUserIntervention = $false
    }

    # Classification basée sur le type d'exception
    switch ($ErrorRecord.Exception.GetType().Name) {
        "IOException" {
            $classification.Severity = "High"
            $classification.Category = "I/O"
            $classification.IsTransient = $true
            $classification.IsRecoverable = $true
        }
        "UnauthorizedAccessException" {
            $classification.Severity = "Critical"
            $classification.Category = "Security"
            $classification.IsTransient = $false
            $classification.IsRecoverable = $false
            $classification.RequiresUserIntervention = $true
        }
        "WebException" {
            $classification.Severity = if ($ErrorRecord.Exception.Response.StatusCode -eq 429) { "Medium" } else { "High" }
            $classification.Category = "Network"
            $classification.IsTransient = $true
            $classification.IsRecoverable = $true
        }
        "SqlException" {
            $classification.Severity = "High"
            $classification.Category = "Database"
            $classification.IsTransient = $true
            $classification.IsRecoverable = $true
        }
        default {
            # Classification basée sur l'ErrorId
            switch -Regex ($ErrorRecord.FullyQualifiedErrorId) {
                "CommandNotFoundException" {
                    $classification.Severity = "Low"
                    $classification.Category = "Configuration"
                    $classification.IsRecoverable = $false
                }
                "ParameterBindingException" {
                    $classification.Severity = "Medium"
                    $classification.Category = "InputValidation"
                    $classification.IsRecoverable = $false
                }
                "RuntimeException" {
                    $classification.Severity = "High"
                    $classification.Category = "Runtime"
                    $classification.IsRecoverable = $true
                }
            }
        }
    }

    return $classification
}

function Get-RecoverySuggestions {
    param([System.Management.Automation.ErrorRecord]$ErrorRecord)

    $suggestions = @()

    switch ($ErrorRecord.Exception.GetType().Name) {
        "IOException" {
            $suggestions += "Vérifier les permissions d'accès au fichier/répertoire"
            $suggestions += "Vérifier si le chemin existe"
            $suggestions += "Vérifier l'espace disque disponible"
            $suggestions += "Implémenter une logique de retry avec backoff"
        }
        "WebException" {
            if ($ErrorRecord.Exception.Response.StatusCode -eq 429) {
                $suggestions += "Implémenter un rate limiting côté client"
                $suggestions += "Utiliser un circuit breaker pattern"
            } elseif ($ErrorRecord.Exception.Response.StatusCode -ge 500) {
                $suggestions += "Vérifier le statut du service distant"
                $suggestions += "Implémenter une logique de fallback"
            } else {
                $suggestions += "Valider les paramètres de la requête"
                $suggestions += "Vérifier l'authentification/autorisation"
            }
        }
        "SqlException" {
            $suggestions += "Vérifier la connectivité à la base de données"
            $suggestions += "Valider la chaîne de connexion"
            $suggestions += "Vérifier les timeouts de connexion"
            $suggestions += "Implémenter des retry avec jitter"
        }
    }

    # Suggestions génériques
    if ($ErrorRecord.ScriptStackTrace) {
        $suggestions += "Ajouter une validation d'entrée plus stricte"
        $suggestions += "Implémenter des tests unitaires pour ce chemin de code"
    }

    return $suggestions
}

# Exemple d'utilisation de l'analyse d'erreur
try {
    # Simulation d'une erreur
    Get-Content "C:\Inexistent\File.txt" -ErrorAction Stop
} catch {
    $errorAnalysis = Analyze-ErrorObject -ErrorRecord $_

    Write-Host "=== ANALYSE D'ERREUR ===" -ForegroundColor Red
    Write-Host "Type d'exception: $($errorAnalysis.ExceptionType)"
    Write-Host "Message: $($errorAnalysis.ExceptionMessage)"
    Write-Host "Catégorie: $($errorAnalysis.Classification.Category)"
    Write-Host "Sévérité: $($errorAnalysis.Classification.Severity)"
    Write-Host "Récupérable: $($errorAnalysis.Classification.IsRecoverable)"

    Write-Host "`nSuggestions de récupération:" -ForegroundColor Yellow
    foreach ($suggestion in $errorAnalysis.RecoverySuggestions) {
        Write-Host "  - $suggestion"
    }

    Write-Host "`nPile d'appels:" -ForegroundColor Cyan
    foreach ($frame in $errorAnalysis.CallStack) {
        Write-Host "  $($frame.Method) - $($frame.FileName):$($frame.LineNumber)"
    }
}
```

### 1.2 Gestion d'erreurs hiérarchisée

**Système de gestion d'erreurs multiniveaux :**
```powershell
# Architecture de gestion d'erreurs hiérarchisée
class ErrorHandler {
    [System.Collections.Generic.List[scriptblock]]$GlobalHandlers
    [System.Collections.Generic.Dictionary[string, scriptblock]]$SpecificHandlers
    [System.Collections.Generic.List[PSCustomObject]]$ErrorHistory
    [hashtable]$ErrorMetrics

    ErrorHandler() {
        $this.GlobalHandlers = [System.Collections.Generic.List[scriptblock]]::new()
        $this.SpecificHandlers = [System.Collections.Generic.Dictionary[string, scriptblock]]::new()
        $this.ErrorHistory = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.ErrorMetrics = @{
            TotalErrors = 0
            ErrorsByType = @{}
            ErrorsByHour = @{}
            RecoveryAttempts = 0
            SuccessfulRecoveries = 0
        }
    }

    [void]RegisterGlobalHandler([scriptblock]$handler) {
        $this.GlobalHandlers.Add($handler)
    }

    [void]RegisterSpecificHandler([string]$errorType, [scriptblock]$handler) {
        $this.SpecificHandlers[$errorType] = $handler
    }

    [PSCustomObject]HandleError([System.Management.Automation.ErrorRecord]$error, [hashtable]$context = @{}) {
        $this.ErrorMetrics.TotalErrors++

        # Enregistrement de l'erreur
        $errorRecord = [PSCustomObject]@{
            Timestamp = Get-Date
            Error = $error
            Context = $context
            Handled = $false
            Recovered = $false
            RecoveryAttempts = 0
        }

        $this.ErrorHistory.Add($errorRecord)

        # Mise à jour des métriques
        $errorType = $error.Exception.GetType().Name
        if (-not $this.ErrorMetrics.ErrorsByType.ContainsKey($errorType)) {
            $this.ErrorMetrics.ErrorsByType[$errorType] = 0
        }
        $this.ErrorMetrics.ErrorsByType[$errorType]++

        $hourKey = (Get-Date).ToString("yyyy-MM-dd-HH")
        if (-not $this.ErrorMetrics.ErrorsByHour.ContainsKey($hourKey)) {
            $this.ErrorMetrics.ErrorsByHour[$hourKey] = 0
        }
        $this.ErrorMetrics.ErrorsByHour[$hourKey]++

        # Tentative de gestion spécifique
        if ($this.SpecificHandlers.ContainsKey($errorType)) {
            try {
                $result = & $this.SpecificHandlers[$errorType] $error $context
                $errorRecord.Handled = $true
                $errorRecord.Recovered = $result.Recovered
                $errorRecord.RecoveryAttempts = $result.Attempts

                if ($result.Recovered) {
                    $this.ErrorMetrics.SuccessfulRecoveries++
                }

                return $result
            } catch {
                Write-Warning "Erreur dans le gestionnaire spécifique pour $errorType`: $_"
            }
        }

        # Gestion globale
        foreach ($handler in $this.GlobalHandlers) {
            try {
                $result = & $handler $error $context
                if ($result.Handled) {
                    $errorRecord.Handled = $true
                    $errorRecord.Recovered = $result.Recovered
                    return $result
                }
            } catch {
                Write-Warning "Erreur dans le gestionnaire global: $_"
            }
        }

        # Gestion par défaut
        Write-Error "Erreur non gérée: $($error.Exception.Message)" -ErrorAction Continue
        return [PSCustomObject]@{
            Handled = $false
            Recovered = $false
            Attempts = 0
            Message = "Erreur non gérée"
        }
    }

    [hashtable]GetErrorMetrics() {
        return $this.ErrorMetrics.Clone()
    }

    [PSCustomObject[]]GetRecentErrors([int]$count = 10) {
        return $this.ErrorHistory | Sort-Object Timestamp -Descending | Select-Object -First $count
    }

    [void]ClearErrorHistory() {
        $this.ErrorHistory.Clear()
    }
}

# Gestionnaire d'erreurs spécialisé pour les opérations réseau
class NetworkErrorHandler : ErrorHandler {
    NetworkErrorHandler() : base() {
        # Gestionnaire spécifique pour les erreurs réseau
        $this.RegisterSpecificHandler("WebException", {
            param($error, $context)

            $response = $error.Exception.Response
            $statusCode = $response.StatusCode.value__

            switch ($statusCode) {
                429 { # Rate limiting
                    $retryAfter = $response.Headers["Retry-After"]
                    if ($retryAfter) {
                        Write-Host "Rate limit atteint. Retry after: $retryAfter seconds"
                        Start-Sleep -Seconds $retryAfter
                        return @{ Handled = $true; Recovered = $true; Attempts = 1 }
                    }
                }
                {$_ -ge 500} { # Erreurs serveur
                    Write-Host "Erreur serveur ($statusCode). Tentative de retry..."
                    Start-Sleep -Seconds 5
                    return @{ Handled = $true; Recovered = $true; Attempts = 1 }
                }
                404 {
                    Write-Warning "Ressource non trouvée: $($context.Url)"
                    return @{ Handled = $true; Recovered = $false; Attempts = 0 }
                }
            }

            return @{ Handled = $false; Recovered = $false; Attempts = 0 }
        })

        # Gestionnaire spécifique pour les erreurs DNS
        $this.RegisterSpecificHandler("System.Net.Sockets.SocketException", {
            param($error, $context)

            if ($error.Exception.ErrorCode -eq 11001) { # DNS resolution failed
                Write-Warning "Résolution DNS échouée pour $($context.Host)"
                # Tentative de retry avec un autre DNS
                return @{ Handled = $true; Recovered = $true; Attempts = 1 }
            }

            return @{ Handled = $false; Recovered = $false; Attempts = 0 }
        })
    }
}

# Gestionnaire global d'erreurs
$global:ErrorHandler = [NetworkErrorHandler]::new()

# Ajout d'un gestionnaire global
$global:ErrorHandler.RegisterGlobalHandler({
    param($error, $context)

    # Logging structuré
    $logEntry = @{
        Timestamp = Get-Date
        Level = "ERROR"
        Message = $error.Exception.Message
        Source = $error.InvocationInfo.ScriptName
        Line = $error.InvocationInfo.ScriptLineNumber
        Context = $context
    }

    # Écriture dans un fichier de log
    $logEntry | ConvertTo-Json | Out-File "C:\Logs\error_log.json" -Append

    # Notification si erreur critique
    if ($error.Exception.Message -match "critical|fatal") {
        Send-MailMessage -To "admin@company.com" -Subject "Erreur critique détectée" -Body ($logEntry | ConvertTo-Json -Depth 3)
    }

    return @{ Handled = $true; Recovered = $false; Attempts = 0 }
})

# Fonction avec gestion d'erreurs avancée
function Invoke-NetworkOperation {
    param(
        [string]$Url,
        [hashtable]$Options = @{}
    )

    $context = @{
        Url = $Url
        Operation = "HTTP_GET"
        Timestamp = Get-Date
    }

    try {
        $response = Invoke-WebRequest -Uri $Url @Options -ErrorAction Stop
        return $response
    } catch {
        $result = $global:ErrorHandler.HandleError($_ , $context)

        if (-not $result.Handled) {
            throw
        }

        if ($result.Recovered) {
            # Retry de l'opération
            try {
                $response = Invoke-WebRequest -Uri $Url @Options -ErrorAction Stop
                return $response
            } catch {
                throw "Échec du retry: $($_.Exception.Message)"
            }
        } else {
            throw "Erreur non récupérable: $($_.Exception.Message)"
        }
    }
}

# Exemple d'utilisation
try {
    # Simulation d'erreurs réseau
    Invoke-NetworkOperation -Url "http://nonexistent.domain"
} catch {
    Write-Host "Erreur finale: $($_.Exception.Message)" -ForegroundColor Red
}

# Analyse des métriques d'erreur
$metrics = $global:ErrorHandler.GetErrorMetrics()
Write-Host "`n=== MÉTRIQUES D'ERREUR ===" -ForegroundColor Cyan
Write-Host "Erreurs totales: $($metrics.TotalErrors)"
Write-Host "Récupérations réussies: $($metrics.SuccessfulRecoveries)"

Write-Host "`nErreurs par type:"
foreach ($type in $metrics.ErrorsByType.Keys) {
    Write-Host "  $type : $($metrics.ErrorsByType[$type])"
}

# Historique récent des erreurs
$recentErrors = $global:ErrorHandler.GetRecentErrors(5)
Write-Host "`n=== ERREURS RÉCENTES ==="
foreach ($error in $recentErrors) {
    Write-Host "$($error.Timestamp): $($error.Error.Exception.Message)"
}
```

## Section 2 : Patterns de récupération et résilience

### 2.1 Circuit Breaker Pattern

**Implémentation avancée du pattern Circuit Breaker :**
```powershell
# Implémentation avancée du pattern Circuit Breaker
class CircuitBreaker {
    [string]$Name
    [CircuitBreakerState]$State
    [int]$FailureThreshold
    [int]$SuccessThreshold
    [TimeSpan]$Timeout
    [int]$CurrentFailures
    [int]$CurrentSuccesses
    [DateTime]$LastFailureTime
    [scriptblock]$Operation
    [hashtable]$Metrics

    CircuitBreaker([string]$name, [scriptblock]$operation, [int]$failureThreshold = 5, [TimeSpan]$timeout = [TimeSpan]::FromSeconds(60)) {
        $this.Name = $name
        $this.Operation = $operation
        $this.FailureThreshold = $failureThreshold
        $this.SuccessThreshold = $failureThreshold  # Même seuil pour la remise en service
        $this.Timeout = $timeout
        $this.State = [CircuitBreakerState]::Closed
        $this.CurrentFailures = 0
        $this.CurrentSuccesses = 0
        $this.Metrics = @{
            TotalRequests = 0
            SuccessfulRequests = 0
            FailedRequests = 0
            CircuitOpens = 0
            CircuitCloses = 0
            LastStateChange = Get-Date
        }
    }

    [PSCustomObject]Execute() {
        $this.Metrics.TotalRequests++

        switch ($this.State) {
            ([CircuitBreakerState]::Closed) {
                return $this.ExecuteClosed()
            }
            ([CircuitBreakerState]::Open) {
                return $this.ExecuteOpen()
            }
            ([CircuitBreakerState]::HalfOpen) {
                return $this.ExecuteHalfOpen()
            }
        }
    }

    hidden [PSCustomObject]ExecuteClosed() {
        try {
            $result = & $this.Operation
            $this.OnSuccess()
            return [PSCustomObject]@{
                Success = $true
                Result = $result
                CircuitState = $this.State
            }
        } catch {
            $this.OnFailure()
            throw
        }
    }

    hidden [PSCustomObject]ExecuteOpen() {
        # Vérification du timeout
        if ((Get-Date) - $this.LastFailureTime -gt $this.Timeout) {
            $this.State = [CircuitBreakerState]::HalfOpen
            $this.CurrentSuccesses = 0
            Write-Host "Circuit Breaker '$($this.Name)': Passage en Half-Open" -ForegroundColor Yellow
            return $this.ExecuteHalfOpen()
        }

        throw [CircuitBreakerException]::new("Circuit Breaker is OPEN", $this.Name)
    }

    hidden [PSCustomObject]ExecuteHalfOpen() {
        try {
            $result = & $this.Operation
            $this.OnHalfOpenSuccess()
            return [PSCustomObject]@{
                Success = $true
                Result = $result
                CircuitState = $this.State
            }
        } catch {
            $this.OnHalfOpenFailure()
            throw
        }
    }

    hidden [void]OnSuccess() {
        $this.Metrics.SuccessfulRequests++
        $this.CurrentFailures = 0
    }

    hidden [void]OnFailure() {
        $this.Metrics.FailedRequests++
        $this.CurrentFailures++
        $this.LastFailureTime = Get-Date

        if ($this.CurrentFailures >= $this.FailureThreshold) {
            $this.State = [CircuitBreakerState]::Open
            $this.Metrics.CircuitOpens++
            $this.Metrics.LastStateChange = Get-Date
            Write-Host "Circuit Breaker '$($this.Name)': OUVERT après $($this.CurrentFailures) échecs" -ForegroundColor Red
        }
    }

    hidden [void]OnHalfOpenSuccess() {
        $this.Metrics.SuccessfulRequests++
        $this.CurrentSuccesses++

        if ($this.CurrentSuccesses >= $this.SuccessThreshold) {
            $this.State = [CircuitBreakerState]::Closed
            $this.Metrics.CircuitCloses++
            $this.Metrics.LastStateChange = Get-Date
            Write-Host "Circuit Breaker '$($this.Name)': FERMÉ après $($this.CurrentSuccesses) succès" -ForegroundColor Green
        }
    }

    hidden [void]OnHalfOpenFailure() {
        $this.State = [CircuitBreakerState]::Open
        $this.CurrentFailures++
        $this.LastFailureTime = Get-Date
        $this.Metrics.FailedRequests++
        Write-Host "Circuit Breaker '$($this.Name)': Retour à OPEN après échec en Half-Open" -ForegroundColor Red
    }

    [hashtable]GetMetrics() {
        return $this.Metrics.Clone()
    }

    [void]ForceOpen() {
        $this.State = [CircuitBreakerState]::Open
        $this.LastFailureTime = Get-Date
        $this.Metrics.LastStateChange = Get-Date
        Write-Host "Circuit Breaker '$($this.Name)': Forcé à OPEN" -ForegroundColor Yellow
    }

    [void]ForceClose() {
        $this.State = [CircuitBreakerState]::Closed
        $this.CurrentFailures = 0
        $this.Metrics.LastStateChange = Get-Date
        Write-Host "Circuit Breaker '$($this.Name)': Forcé à CLOSED" -ForegroundColor Yellow
    }

    [string]ToString() {
        return "CircuitBreaker '$($this.Name)': $($this.State) (Échecs: $($this.CurrentFailures)/$($this.FailureThreshold))"
    }
}

# Énumération pour les états du circuit breaker
enum CircuitBreakerState {
    Closed
    Open
    HalfOpen
}

# Exception personnalisée pour le circuit breaker
class CircuitBreakerException : Exception {
    [string]$CircuitBreakerName

    CircuitBreakerException([string]$message, [string]$circuitBreakerName) : base($message) {
        $this.CircuitBreakerName = $circuitBreakerName
    }
}

# Gestionnaire de circuit breakers
class CircuitBreakerManager {
    [System.Collections.Generic.Dictionary[string, CircuitBreaker]]$CircuitBreakers
    [hashtable]$GlobalMetrics

    CircuitBreakerManager() {
        $this.CircuitBreakers = [System.Collections.Generic.Dictionary[string, CircuitBreaker]]::new()
        $this.GlobalMetrics = @{
            TotalCircuits = 0
            OpenCircuits = 0
            HalfOpenCircuits = 0
            ClosedCircuits = 0
        }
    }

    [CircuitBreaker]CreateCircuitBreaker([string]$name, [scriptblock]$operation, [hashtable]$options = @{}) {
        $failureThreshold = $options.FailureThreshold ?? 5
        $timeout = $options.Timeout ?? [TimeSpan]::FromSeconds(60)

        $circuit = [CircuitBreaker]::new($name, $operation, $failureThreshold, $timeout)
        $this.CircuitBreakers[$name] = $circuit
        $this.GlobalMetrics.TotalCircuits++

        return $circuit
    }

    [PSCustomObject]ExecuteCircuitBreaker([string]$name) {
        if (-not $this.CircuitBreakers.ContainsKey($name)) {
            throw "Circuit Breaker '$name' n'existe pas"
        }

        return $this.CircuitBreakers[$name].Execute()
    }

    [void]UpdateGlobalMetrics() {
        $this.GlobalMetrics.OpenCircuits = ($this.CircuitBreakers.Values | Where-Object { $_.State -eq [CircuitBreakerState]::Open }).Count
        $this.GlobalMetrics.HalfOpenCircuits = ($this.CircuitBreakers.Values | Where-Object { $_.State -eq [CircuitBreakerState]::HalfOpen }).Count
        $this.GlobalMetrics.ClosedCircuits = ($this.CircuitBreakers.Values | Where-Object { $_.State -eq [CircuitBreakerState]::Closed }).Count
    }

    [hashtable]GetGlobalMetrics() {
        $this.UpdateGlobalMetrics()
        return $this.GlobalMetrics.Clone()
    }

    [CircuitBreaker[]]GetOpenCircuits() {
        return $this.CircuitBreakers.Values | Where-Object { $_.State -eq [CircuitBreakerState]::Open }
    }

    [void]ResetAllCircuits() {
        foreach ($circuit in $this.CircuitBreakers.Values) {
            $circuit.ForceClose()
        }
        Write-Host "Tous les circuits ont été remis à zéro" -ForegroundColor Green
    }
}

# Utilisation du système de circuit breakers
$circuitManager = [CircuitBreakerManager]::new()

# Création de circuits pour différents services
$webServiceCircuit = $circuitManager.CreateCircuitBreaker("WebService", {
    try {
        $response = Invoke-WebRequest -Uri "http://api.example.com/data" -TimeoutSec 10
        return $response.Content
    } catch {
        throw "Erreur appel web service: $($_.Exception.Message)"
    }
}, @{
    FailureThreshold = 3
    Timeout = [TimeSpan]::FromSeconds(30)
})

$dbCircuit = $circuitManager.CreateCircuitBreaker("Database", {
    try {
        $connection = New-Object System.Data.SqlClient.SqlConnection
        $connection.ConnectionString = "Server=localhost;Database=Test;Integrated Security=True;"
        $connection.Open()
        $connection.Close()
        return "Connexion réussie"
    } catch {
        throw "Erreur connexion base de données: $($_.Exception.Message)"
    }
})

# Test du système avec des échecs simulés
Write-Host "=== TEST DU SYSTÈME DE CIRCUIT BREAKERS ===" -ForegroundColor Cyan

for ($i = 1; $i -le 10; $i++) {
    Write-Host "`nTentative $i :" -ForegroundColor Yellow

    try {
        $result = $circuitManager.ExecuteCircuitBreaker("WebService")
        Write-Host "  ✓ WebService: $($result.Success)" -ForegroundColor Green
    } catch [CircuitBreakerException] {
        Write-Host "  ✗ WebService: Circuit OPEN - $($_.Exception.Message)" -ForegroundColor Red
    } catch {
        Write-Host "  ✗ WebService: Erreur - $($_.Exception.Message)" -ForegroundColor Red
    }

    try {
        $result = $circuitManager.ExecuteCircuitBreaker("Database")
        Write-Host "  ✓ Database: $($result.Success)" -ForegroundColor Green
    } catch [CircuitBreakerException] {
        Write-Host "  ✗ Database: Circuit OPEN" -ForegroundColor Red
    } catch {
        Write-Host "  ✗ Database: Erreur - $($_.Exception.Message)" -ForegroundColor Red
    }

    # Simulation d'un délai entre les tentatives
    Start-Sleep -Milliseconds 500
}

# Métriques finales
$globalMetrics = $circuitManager.GetGlobalMetrics()
Write-Host "`n=== MÉTRIQUES FINALES ===" -ForegroundColor Cyan
Write-Host "Circuits totaux: $($globalMetrics.TotalCircuits)"
Write-Host "Circuits ouverts: $($globalMetrics.OpenCircuits)"
Write-Host "Circuits half-open: $($globalMetrics.HalfOpenCircuits)"
Write-Host "Circuits fermés: $($globalMetrics.ClosedCircuits)"

# Détails des circuits ouverts
$openCircuits = $circuitManager.GetOpenCircuits()
if ($openCircuits) {
    Write-Host "`nCircuits ouverts:" -ForegroundColor Red
    foreach ($circuit in $openCircuits) {
        Write-Host "  - $($circuit.ToString())"
    }
}
```

### 2.2 Retry Pattern avec backoff exponentiel

**Implémentation sophistiquée du pattern Retry :**
```powershell
# Pattern Retry avancé avec backoff exponentiel et jitter
class RetryPolicy {
    [int]$MaxAttempts
    [TimeSpan]$InitialDelay
    [double]$BackoffMultiplier
    [TimeSpan]$MaxDelay
    [bool]$UseJitter
    [scriptblock]$ShouldRetryCondition
    [System.Collections.Generic.List[PSCustomObject]]$Attempts

    RetryPolicy([int]$maxAttempts = 3, [TimeSpan]$initialDelay = [TimeSpan]::FromSeconds(1)) {
        $this.MaxAttempts = $maxAttempts
        $this.InitialDelay = $initialDelay
        $this.BackoffMultiplier = 2.0
        $this.MaxDelay = [TimeSpan]::FromMinutes(5)
        $this.UseJitter = $true
        $this.ShouldRetryCondition = { param($exception) $true }  # Retry toutes les erreurs par défaut
        $this.Attempts = [System.Collections.Generic.List[PSCustomObject]]::new()
    }

    [PSCustomObject]Execute([scriptblock]$operation) {
        $attempt = 0
        $lastException = $null

        while ($attempt -lt $this.MaxAttempts) {
            $attempt++
            $startTime = Get-Date

            try {
                $result = & $operation
                $duration = (Get-Date) - $startTime

                # Enregistrement de la tentative réussie
                $this.Attempts.Add([PSCustomObject]@{
                    Attempt = $attempt
                    Success = $true
                    Duration = $duration
                    Exception = $null
                    Timestamp = Get-Date
                })

                return [PSCustomObject]@{
                    Success = $true
                    Result = $result
                    Attempts = $attempt
                    TotalDuration = (Get-Date) - $startTime
                }

            } catch {
                $duration = (Get-Date) - $startTime
                $lastException = $_

                # Enregistrement de la tentative échouée
                $this.Attempts.Add([PSCustomObject]@{
                    Attempt = $attempt
                    Success = $false
                    Duration = $duration
                    Exception = $_.Exception.Message
                    Timestamp = Get-Date
                })

                # Vérification si on doit continuer
                if ($attempt -ge $this.MaxAttempts) {
                    break
                }

                # Vérification de la condition de retry
                if (-not (& $this.ShouldRetryCondition $_.Exception)) {
                    break
                }

                # Calcul du délai avant retry
                $delay = $this.CalculateDelay($attempt)
                Write-Host "Tentative $attempt échouée. Retry dans $($delay.TotalSeconds) secondes..." -ForegroundColor Yellow

                Start-Sleep -Milliseconds $delay.TotalMilliseconds
            }
        }

        # Toutes les tentatives ont échoué
        throw [RetryException]::new("Échec après $($this.MaxAttempts) tentatives", $lastException, $this.Attempts)
    }

    hidden [TimeSpan]CalculateDelay([int]$attempt) {
        # Backoff exponentiel: delay = initialDelay * (backoffMultiplier ^ (attempt - 1))
        $exponentialDelay = $this.InitialDelay.TotalMilliseconds * [Math]::Pow($this.BackoffMultiplier, $attempt - 1)
        $delay = [TimeSpan]::FromMilliseconds($exponentialDelay)

        # Limitation au délai maximum
        if ($delay -gt $this.MaxDelay) {
            $delay = $this.MaxDelay
        }

        # Ajout de jitter si activé
        if ($this.UseJitter) {
            $randomFactor = (Get-Random -Minimum 0.5 -Maximum 1.5)
            $delay = [TimeSpan]::FromMilliseconds($delay.TotalMilliseconds * $randomFactor)
        }

        return $delay
    }

    [RetryPolicy]WithMaxDelay([TimeSpan]$maxDelay) {
        $this.MaxDelay = $maxDelay
        return $this
    }

    [RetryPolicy]WithBackoffMultiplier([double]$multiplier) {
        $this.BackoffMultiplier = $multiplier
        return $this
    }

    [RetryPolicy]WithJitter([bool]$useJitter) {
        $this.UseJitter = $useJitter
        return $this
    }

    [RetryPolicy]WithRetryCondition([scriptblock]$condition) {
        $this.ShouldRetryCondition = $condition
        return $this
    }

    [PSCustomObject[]]GetAttempts() {
        return $this.Attempts.ToArray()
    }
}

# Exception personnalisée pour les retry
class RetryException : Exception {
    [System.Management.Automation.ErrorRecord]$LastError
    [System.Collections.Generic.List[PSCustomObject]]$Attempts

    RetryException([string]$message, [System.Management.Automation.ErrorRecord]$lastError, [System.Collections.Generic.List[PSCustomObject]]$attempts)
        : base($message) {
        $this.LastError = $lastError
        $this.Attempts = $attempts
    }
}

# Politiques de retry prédéfinies
class RetryPolicies {
    static [RetryPolicy]NetworkCall() {
        return [RetryPolicy]::new(5, [TimeSpan]::FromSeconds(1)) |
            ForEach-Object {
                $_.WithMaxDelay([TimeSpan]::FromSeconds(30))
                $_.WithRetryCondition({
                    param($exception)
                    # Retry seulement pour les erreurs réseau temporaires
                    $exception -is [System.Net.WebException] -or
                    $exception -is [System.IO.IOException] -or
                    $exception.Message -match "timeout|connection|network"
                })
            }
    }

    static [RetryPolicy]DatabaseOperation() {
        return [RetryPolicy]::new(3, [TimeSpan]::FromSeconds(2)) |
            ForEach-Object {
                $_.WithMaxDelay([TimeSpan]::FromMinutes(1))
                $_.WithBackoffMultiplier(1.5)
                $_.WithRetryCondition({
                    param($exception)
                    # Retry seulement pour les erreurs de connexion DB
                    $exception -is [System.Data.SqlClient.SqlException] -and
                    ($exception.Number -eq 53 -or $exception.Number -eq 40)  # Connection errors
                })
            }
    }

    static [RetryPolicy]ExternalServiceCall() {
        return [RetryPolicy]::new(4, [TimeSpan]::FromMilliseconds(500)) |
            ForEach-Object {
                $_.WithMaxDelay([TimeSpan]::FromSeconds(10))
                $_.WithRetryCondition({
                    param($exception)
                    # Retry pour les erreurs 5xx et timeouts
                    ($exception -is [System.Net.WebException] -and $exception.Response.StatusCode -ge 500) -or
                    $exception.Message -match "timeout"
                })
            }
    }
}

# Fonctions utilitaires avec retry automatique
function Invoke-WithRetry {
    param(
        [scriptblock]$Operation,
        [RetryPolicy]$Policy = [RetryPolicy]::new()
    )

    try {
        $result = $Policy.Execute($Operation)
        Write-Debug "Opération réussie après $($result.Attempts) tentative(s)"
        return $result.Result
    } catch [RetryException] {
        Write-Error "Échec définitif après $($_.Exception.Attempts.Count) tentatives"
        Write-Error "Dernière erreur: $($_.Exception.LastError.Exception.Message)"

        # Affichage du résumé des tentatives
        Write-Host "Résumé des tentatives:" -ForegroundColor Yellow
        foreach ($attempt in $_.Exception.Attempts) {
            $status = if ($attempt.Success) { "✓" } else { "✗" }
            Write-Host "  $status Tentative $($attempt.Attempt): $([math]::Round($attempt.Duration.TotalSeconds, 2))s"
            if (-not $attempt.Success) {
                Write-Host "    Erreur: $($attempt.Exception)"
            }
        }

        throw
    }
}

# Exemples d'utilisation des politiques de retry
Write-Host "=== EXEMPLES DE POLITIQUES DE RETRY ===" -ForegroundColor Cyan

# 1. Appel réseau avec retry
Write-Host "`n1. Test d'appel réseau avec retry:" -ForegroundColor Yellow
$networkPolicy = [RetryPolicies]::NetworkCall()

$networkResult = Invoke-WithRetry -Operation {
    # Simulation d'un appel réseau qui peut échouer
    if ((Get-Random -Maximum 10) -lt 7) {  # 70% de chance d'échec
        throw [System.Net.WebException]::new("Connection timeout", [System.Net.WebExceptionStatus]::Timeout)
    }
    return "Données récupérées du réseau"
} -Policy $networkPolicy

Write-Host "Résultat: $networkResult" -ForegroundColor Green

# 2. Opération base de données avec retry
Write-Host "`n2. Test d'opération base de données avec retry:" -ForegroundColor Yellow
$dbPolicy = [RetryPolicies]::DatabaseOperation()

$dbResult = Invoke-WithRetry -Operation {
    # Simulation d'une opération DB qui peut échouer
    if ((Get-Random -Maximum 10) -lt 4) {  # 40% de chance d'échec
        $sqlException = New-Object System.Data.SqlClient.SqlException
        $sqlException | Add-Member -MemberType NoteProperty -Name "Number" -Value 53  # Connection error
        throw $sqlException
    }
    return "Données mises à jour en base"
} -Policy $dbPolicy

Write-Host "Résultat: $dbResult" -ForegroundColor Green

# 3. Appel de service externe avec retry et jitter
Write-Host "`n3. Test d'appel de service externe avec jitter:" -ForegroundColor Yellow
$servicePolicy = [RetryPolicies]::ExternalServiceCall()

$serviceResult = Invoke-WithRetry -Operation {
    # Simulation d'un appel de service externe
    $random = Get-Random -Maximum 10
    if ($random -lt 5) {  # 50% de chance d'échec
        $webException = [System.Net.WebException]::new("Service temporarily unavailable")
        $webException | Add-Member -MemberType NoteProperty -Name "Response" -Value @{
            StatusCode = 503
        }
        throw $webException
    }
    return "Service appelé avec succès"
} -Policy $servicePolicy

Write-Host "Résultat: $serviceResult" -ForegroundColor Green

# 4. Retry personnalisé
Write-Host "`n4. Test de retry personnalisé:" -ForegroundColor Yellow
$customPolicy = [RetryPolicy]::new(3, [TimeSpan]::FromSeconds(0.5))
$customPolicy.WithBackoffMultiplier(3.0).WithJitter($true)

$customResult = Invoke-WithRetry -Operation {
    $attempt = $global:CustomAttemptCount++
    if ($attempt -lt 2) {  # Échec des 2 premières tentatives
        throw "Erreur temporaire (tentative $attempt)"
    }
    return "Opération réussie à la tentative $($attempt + 1)"
} -Policy $customPolicy

Write-Host "Résultat: $customResult" -ForegroundColor Green

# Analyse des performances de retry
Write-Host "`n=== ANALYSE DES PERFORMANCES DE RETRY ===" -ForegroundColor Cyan

$allAttempts = @()
$allAttempts += $networkPolicy.GetAttempts()
$allAttempts += $dbPolicy.GetAttempts()
$allAttempts += $servicePolicy.GetAttempts()
$allAttempts += $customPolicy.GetAttempts()

$successfulAttempts = $allAttempts | Where-Object { $_.Success }
$failedAttempts = $allAttempts | Where-Object { -not $_.Success }

Write-Host "Total des tentatives: $($allAttempts.Count)"
Write-Host "Tentatives réussies: $($successfulAttempts.Count)"
Write-Host "Tentatives échouées: $($failedAttempts.Count)"

if ($successfulAttempts) {
    $avgDuration = ($successfulAttempts | Measure-Object Duration.TotalSeconds -Average).Average
    Write-Host "Durée moyenne des succès: $([math]::Round($avgDuration, 2)) secondes"
}

if ($failedAttempts) {
    Write-Host "Répartition des échecs par type:"
    $failedAttempts | Group-Object Exception | ForEach-Object {
        Write-Host "  $($_.Name): $($_.Count) occurrence(s)"
    }
}
```

## Section 3 : Systèmes d'auto-guérison et apprentissage

### 3.1 Auto-remédiation basée sur les patterns

**Système d'auto-remédiation intelligent :**
```powershell
# Système d'auto-remédiation basé sur l'apprentissage des erreurs
class AutoRemediationSystem {
    [System.Collections.Generic.Dictionary[string, hashtable]]$ErrorPatterns
    [System.Collections.Generic.Dictionary[string, scriptblock]]$RemediationActions
    [System.Collections.Generic.List[PSCustomObject]]$RemediationHistory
    [hashtable]$SuccessMetrics

    AutoRemediationSystem() {
        $this.ErrorPatterns = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.RemediationActions = [System.Collections.Generic.Dictionary[string, scriptblock]]::new()
        $this.RemediationHistory = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.SuccessMetrics = @{
            TotalRemediations = 0
            SuccessfulRemediations = 0
            FailedRemediations = 0
            AverageResolutionTime = 0
        }
    }

    [void]RegisterErrorPattern([string]$patternName, [hashtable]$patternDefinition) {
        $this.ErrorPatterns[$patternName] = $patternDefinition
    }

    [void]RegisterRemediationAction([string]$patternName, [scriptblock]$action) {
        $this.RemediationActions[$patternName] = $action
    }

    [PSCustomObject]AnalyzeAndRemediate([System.Management.Automation.ErrorRecord]$error, [hashtable]$context = @{}) {
        $startTime = Get-Date

        # Recherche du pattern correspondant
        $matchedPattern = $this.MatchErrorPattern($error, $context)

        if (-not $matchedPattern) {
            return [PSCustomObject]@{
                PatternMatched = $null
                RemediationAttempted = $false
                Success = $false
                Duration = (Get-Date) - $startTime
                Message = "Aucun pattern de remédiation trouvé"
            }
        }

        # Tentative de remédiation
        $this.SuccessMetrics.TotalRemediations++
        $remediationResult = $this.ExecuteRemediation($matchedPattern, $error, $context)
        $duration = (Get-Date) - $startTime

        # Enregistrement de l'historique
        $historyEntry = [PSCustomObject]@{
            Timestamp = Get-Date
            Error = $error.Exception.Message
            Pattern = $matchedPattern
            RemediationAttempted = $true
            Success = $remediationResult.Success
            Duration = $duration
            Context = $context
            Details = $remediationResult.Details
        }

        $this.RemediationHistory.Add($historyEntry)

        # Mise à jour des métriques
        if ($remediationResult.Success) {
            $this.SuccessMetrics.SuccessfulRemediations++
        } else {
            $this.SuccessMetrics.FailedRemediations++
        }

        # Mise à jour du temps de résolution moyen
        $totalTime = ($this.RemediationHistory | Measure-Object Duration.TotalSeconds -Sum).Sum
        $this.SuccessMetrics.AverageResolutionTime = $totalTime / $this.RemediationHistory.Count

        # Apprentissage: ajustement des patterns basé sur le succès/échec
        $this.LearnFromOutcome($matchedPattern, $remediationResult.Success)

        return [PSCustomObject]@{
            PatternMatched = $matchedPattern
            RemediationAttempted = $true
            Success = $remediationResult.Success
            Duration = $duration
            Message = $remediationResult.Message
            Details = $remediationResult.Details
        }
    }

    hidden [string]MatchErrorPattern([System.Management.Automation.ErrorRecord]$error, [hashtable]$context) {
        foreach ($patternName in $this.ErrorPatterns.Keys) {
            $pattern = $this.ErrorPatterns[$patternName]

            $matches = $true

            # Vérification des conditions du pattern
            if ($pattern.ContainsKey('ExceptionType') -and $error.Exception.GetType().Name -notmatch $pattern.ExceptionType) {
                $matches = $false
            }

            if ($pattern.ContainsKey('ErrorMessage') -and $error.Exception.Message -notmatch $pattern.ErrorMessage) {
                $matches = $false
            }

            if ($pattern.ContainsKey('Category') -and $error.CategoryInfo.Category -ne $pattern.Category) {
                $matches = $false
            }

            if ($pattern.ContainsKey('ContextCondition') -and -not (& $pattern.ContextCondition $context)) {
                $matches = $false
            }

            if ($matches) {
                return $patternName
            }
        }

        return $null
    }

    hidden [PSCustomObject]ExecuteRemediation([string]$patternName, [System.Management.Automation.ErrorRecord]$error, [hashtable]$context) {
        if (-not $this.RemediationActions.ContainsKey($patternName)) {
            return [PSCustomObject]@{
                Success = $false
                Message = "Aucune action de remédiation définie pour le pattern $patternName"
                Details = @{}
            }
        }

        $action = $this.RemediationActions[$patternName]

        try {
            $result = & $action $error $context

            return [PSCustomObject]@{
                Success = $result.Success
                Message = $result.Message ?? "Remédiation exécutée"
                Details = $result.Details ?? @{}
            }

        } catch {
            return [PSCustomObject]@{
                Success = $false
                Message = "Échec de la remédiation: $($_.Exception.Message)"
                Details = @{ RemediationError = $_.Exception.Message }
            }
        }
    }

    hidden [void]LearnFromOutcome([string]$patternName, [bool]$success) {
        # Apprentissage simple: ajuster les poids des patterns
        if (-not $this.ErrorPatterns[$patternName].ContainsKey('SuccessCount')) {
            $this.ErrorPatterns[$patternName].SuccessCount = 0
            $this.ErrorPatterns[$patternName].FailureCount = 0
        }

        if ($success) {
            $this.ErrorPatterns[$patternName].SuccessCount++
        } else {
            $this.ErrorPatterns[$patternName].FailureCount++
        }

        # Calcul du taux de succès
        $total = $this.ErrorPatterns[$patternName].SuccessCount + $this.ErrorPatterns[$patternName].FailureCount
        $this.ErrorPatterns[$patternName].SuccessRate = $this.ErrorPatterns[$patternName].SuccessCount / $total
    }

    [hashtable]GetMetrics() {
        return $this.SuccessMetrics.Clone()
    }

    [PSCustomObject[]]GetTopPatterns([int]$count = 5) {
        return $this.ErrorPatterns.Values |
            Where-Object { $_.ContainsKey('SuccessCount') } |
            Sort-Object SuccessRate -Descending |
            Select-Object -First $count
    }

    [string]GenerateReport() {
        $report = @"
RAPPORT D'AUTO-RÉMÉDIATION
==========================
Généré le: $(Get-Date)

MÉTRIQUES GÉNÉRALES
===================
Total des remédiations: $($this.SuccessMetrics.TotalRemediations)
Remédiations réussies: $($this.SuccessMetrics.SuccessfulRemediations)
Remédiations échouées: $($this.SuccessMetrics.FailedRemediations)
Temps de résolution moyen: $([math]::Round($this.SuccessMetrics.AverageResolutionTime, 2)) secondes

TAUX DE SUCCÈS PAR PATTERN
===========================
"@

        foreach ($pattern in $this.GetTopPatterns(10)) {
            $successRate = [math]::Round($pattern.SuccessRate * 100, 1)
            $totalAttempts = $pattern.SuccessCount + $pattern.FailureCount
            $report += "$($pattern.Name ?? 'Unknown'): ${successRate}% ($($pattern.SuccessCount)/$totalAttempts)`n"
        }

        $report += @"

DERNIÈRES REMÉDIATIONS
======================
"@

        $recentRemediations = $this.RemediationHistory | Sort-Object Timestamp -Descending | Select-Object -First 5
        foreach ($remediation in $recentRemediations) {
            $status = if ($remediation.Success) { "✓" } else { "✗" }
            $report += "$status $($remediation.Timestamp): $($remediation.Pattern) - $([math]::Round($remediation.Duration.TotalSeconds, 2))s`n"
        }

        return $report
    }
}

# Configuration du système d'auto-remédiation
$remediationSystem = [AutoRemediationSystem]::new()

# Pattern pour les services arrêtés
$remediationSystem.RegisterErrorPattern("ServiceStopped", @{
    ExceptionType = "InvalidOperationException"
    ErrorMessage = "Cannot.*service.*because.*stopped"
    Category = "ResourceUnavailable"
})

$remediationSystem.RegisterRemediationAction("ServiceStopped", {
    param($error, $context)

    # Extraction du nom du service depuis l'erreur
    if ($error.Exception.Message -match "service '([^']+)'") {
        $serviceName = $Matches[1]

        try {
            Start-Service -Name $serviceName
            return @{
                Success = $true
                Message = "Service $serviceName redémarré"
                Details = @{ ServiceName = $serviceName; Action = "Start" }
            }
        } catch {
            return @{
                Success = $false
                Message = "Échec du redémarrage du service $serviceName`: $($_.Exception.Message)"
                Details = @{ ServiceName = $serviceName; Error = $_.Exception.Message }
            }
        }
    }

    return @{ Success = $false; Message = "Nom du service non trouvé dans l'erreur" }
})

# Pattern pour les erreurs de connexion réseau
$remediationSystem.RegisterErrorPattern("NetworkTimeout", @{
    ExceptionType = "WebException"
    ErrorMessage = "timeout|connection|network"
    ContextCondition = { param($ctx) $ctx.ContainsKey('RetryCount') -and $ctx.RetryCount -lt 3 }
})

$remediationSystem.RegisterRemediationAction("NetworkTimeout", {
    param($error, $context)

    $retryCount = $context.RetryCount + 1
    $delay = [math]::Min(30, [math]::Pow(2, $retryCount))  # Backoff exponentiel

    Write-Host "Tentative de retry dans $delay secondes (tentative $retryCount)..." -ForegroundColor Yellow
    Start-Sleep -Seconds $delay

    return @{
        Success = $true
        Message = "Retry programmé après délai de $delay secondes"
        Details = @{ RetryCount = $retryCount; Delay = $delay; Action = "Retry" }
    }
})

# Pattern pour les erreurs de quota disque
$remediationSystem.RegisterErrorPattern("DiskFull", @{
    ExceptionType = "IOException"
    ErrorMessage = "disk.*full|no.*space|quota.*exceeded"
})

$remediationSystem.RegisterRemediationAction("DiskFull", {
    param($error, $context)

    # Nettoyage automatique des fichiers temporaires
    try {
        $tempPath = [System.IO.Path]::GetTempPath()
        $oldFiles = Get-ChildItem $tempPath -File | Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) }

        if ($oldFiles) {
            $oldFiles | Remove-Item -Force
            $freedSpace = ($oldFiles | Measure-Object Length -Sum).Sum / 1MB

            return @{
                Success = $true
                Message = "Nettoyage des fichiers temporaires: $([math]::Round($freedSpace, 2)) MB libérés"
                Details = @{ FilesRemoved = $oldFiles.Count; SpaceFreedMB = $freedSpace; Action = "Cleanup" }
            }
        } else {
            return @{
                Success = $false
                Message = "Aucun fichier temporaire ancien trouvé"
                Details = @{ Action = "NoAction" }
            }
        }
    } catch {
        return @{
            Success = $false
            Message = "Échec du nettoyage: $($_.Exception.Message)"
            Details = @{ Error = $_.Exception.Message }
        }
    }
})

# Fonction wrapper avec auto-remédiation
function Invoke-WithAutoRemediation {
    param(
        [Parameter(Mandatory=$true)]
        [scriptblock]$Operation,

        [hashtable]$Context = @{}
    )

    try {
        return & $Operation
    } catch {
        Write-Host "Erreur détectée: $($_.Exception.Message)" -ForegroundColor Red

        $remediationResult = $remediationSystem.AnalyzeAndRemediate($_, $Context)

        Write-Host "Résultat de la remédiation: $($remediationResult.Message)" -ForegroundColor (if ($remediationResult.Success) { "Green" } else { "Yellow" })

        if ($remediationResult.Success) {
            # Retry de l'opération après remédiation
            try {
                Write-Host "Nouvelle tentative de l'opération..." -ForegroundColor Cyan
                return & $Operation
            } catch {
                Write-Error "Échec même après remédiation: $($_.Exception.Message)"
                throw
            }
        } else {
            Write-Error "Remédiation échouée: $($remediationResult.Message)"
            throw
        }
    }
}

# Tests du système d'auto-remédiation
Write-Host "=== TESTS DU SYSTÈME D'AUTO-RÉMÉDIATION ===" -ForegroundColor Cyan

# Test 1: Service arrêté
Write-Host "`n1. Test de redémarrage de service:" -ForegroundColor Yellow
try {
    Invoke-WithAutoRemediation -Operation {
        # Simulation d'une opération qui échoue à cause d'un service arrêté
        throw [InvalidOperationException]::new("Cannot perform operation because the service 'TestService' is stopped")
    } -Context @{ Operation = "TestServiceCheck" }
} catch {
    Write-Host "Test 1 terminé (erreur attendue)" -ForegroundColor Gray
}

# Test 2: Timeout réseau
Write-Host "`n2. Test de retry réseau:" -ForegroundColor Yellow
try {
    Invoke-WithAutoRemediation -Operation {
        # Simulation d'un timeout réseau
        throw [System.Net.WebException]::new("The operation has timed out")
    } -Context @{ Url = "http://test.com"; RetryCount = 0 }
} catch {
    Write-Host "Test 2 terminé (erreur attendue)" -ForegroundColor Gray
}

# Test 3: Espace disque plein
Write-Host "`n3. Test de nettoyage disque:" -ForegroundColor Yellow
try {
    Invoke-WithAutoRemediation -Operation {
        # Simulation d'une erreur disque plein
        throw [System.IO.IOException]::new("There is not enough space on the disk")
    } -Context @{ Path = "C:\Test"; RequiredSpace = 100MB }
} catch {
    Write-Host "Test 3 terminé (erreur attendue)" -ForegroundColor Gray
}

# Rapport final
Write-Host "`n=== RAPPORT FINAL ===" -ForegroundColor Cyan
$metrics = $remediationSystem.GetMetrics()
Write-Host "Remédiations totales: $($metrics.TotalRemediations)"
Write-Host "Taux de succès: $([math]::Round(($metrics.SuccessfulRemediations / $metrics.TotalRemediations) * 100, 1))%"

Write-Host "`nPatterns les plus efficaces:"
$topPatterns = $remediationSystem.GetTopPatterns(3)
foreach ($pattern in $topPatterns) {
    $successRate = [math]::Round($pattern.SuccessRate * 100, 1)
    Write-Host "  $($pattern.Name ?? 'Unknown'): ${successRate}% de succès"
}

# Génération du rapport complet
$fullReport = $remediationSystem.GenerateReport()
$fullReport | Out-File -FilePath "AutoRemediationReport.txt" -Encoding UTF8
Write-Host "`nRapport complet sauvegardé: AutoRemediationReport.txt"
```

## Conclusion : Les erreurs comme catalyseur d'amélioration

La gestion avancée des erreurs PowerShell transforme les échecs en opportunités d'apprentissage et d'amélioration automatique. En implémentant des systèmes de classification d'erreurs, de circuit breakers, de retry intelligents, et d'auto-remédiation, les scripts PowerShell deviennent non seulement plus robustes, mais aussi capables d'évoluer et de s'adapter aux conditions changeantes du système.

Dans le prochain chapitre, nous explorerons les techniques de sécurisation avancées des scripts PowerShell et les meilleures pratiques de développement sécurisé.

---

**Exercice pratique :** Créez un système complet de gestion d'erreurs qui :
1. Classe automatiquement les erreurs par sévérité et type
2. Implémente des circuit breakers pour les services critiques
3. Utilise des politiques de retry avec backoff exponentiel
4. Fournit des rapports détaillés sur les patterns d'erreur
5. Intègre un système d'auto-remédiation apprenant

**Challenge avancé :** Développez une plateforme de résilience qui :
- Prédit les pannes basées sur les patterns d'erreur historiques
- Implémente des stratégies de failover automatique
- Adapte dynamiquement les seuils de retry selon les conditions
- Fournit des dashboards en temps réel de la santé système
- Intègre l'apprentissage machine pour l'optimisation des remédiations

**Réflexion :** Comment la gestion d'erreurs évolue-t-elle avec la complexité des systèmes distribués ? Les approches traditionnelles de try-catch sont-elles suffisantes face aux architectures modernes ? Quand l'auto-remédiation devient-elle essentielle, et quels sont les risques associés à la délégation de décisions aux machines ?
