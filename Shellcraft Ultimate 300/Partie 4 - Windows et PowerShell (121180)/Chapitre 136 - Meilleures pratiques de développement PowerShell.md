# Chapitre 136 - Meilleures pratiques de développement PowerShell

> "L'excellence en développement PowerShell ne vient pas seulement de la maîtrise technique, mais de l'application rigoureuse de principes qui transforment le code en œuvres d'art maintenables, évolutives et élégantes." - Citation inspirée des principes du clean code

## Introduction : L'art du développement PowerShell

Les meilleures pratiques de développement PowerShell transcendent la simple écriture de code fonctionnel pour embrasser une philosophie de qualité, de maintenabilité et d'évolutivité. En appliquant des principes rigoureux de conception, de structure et de qualité de code, les développeurs PowerShell créent des solutions non seulement efficaces mais aussi pérennes dans des environnements complexes.

Dans ce chapitre, nous explorerons les patterns de conception avancés, les standards de codage, et les méthodologies de développement qui élèvent PowerShell au rang d'art.

## Section 1 : Principes de conception et architecture

### 1.1 Séparation des préoccupations et modularité

**Architecture modulaire avancée :**
```powershell
# Framework de développement modulaire PowerShell
class PowerShellModuleFramework {
    [System.Collections.Generic.Dictionary[string, hashtable]]$Modules
    [System.Collections.Generic.Dictionary[string, scriptblock]]$ModuleInitializers
    [hashtable]$GlobalConfiguration
    [System.Collections.Generic.List[string]]$LoadOrder

    PowerShellModuleFramework() {
        $this.Modules = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.ModuleInitializers = [System.Collections.Generic.Dictionary[string, scriptblock]]::new()
        $this.GlobalConfiguration = @{}
        $this.LoadOrder = [System.Collections.Generic.List[string]]::new()
    }

    [void]RegisterModule([string]$moduleName, [hashtable]$moduleDefinition) {
        # Validation de la définition du module
        $requiredKeys = @('Version', 'Description', 'Dependencies', 'Functions', 'Cmdlets')
        foreach ($key in $requiredKeys) {
            if (-not $moduleDefinition.ContainsKey($key)) {
                throw "Module '$moduleName' definition missing required key: $key"
            }
        }

        $moduleDefinition.ModuleName = $moduleName
        $moduleDefinition.Loaded = $false
        $moduleDefinition.LoadTime = $null
        $moduleDefinition.Status = 'Registered'

        $this.Modules[$moduleName] = $moduleDefinition

        Write-Host "Module '$moduleName' v$($moduleDefinition.Version) registered" -ForegroundColor Green
    }

    [void]SetModuleInitializer([string]$moduleName, [scriptblock]$initializer) {
        if (-not $this.Modules.ContainsKey($moduleName)) {
            throw "Module '$moduleName' not registered"
        }

        $this.ModuleInitializers[$moduleName] = $initializer
    }

    [void]SetLoadOrder([string[]]$moduleOrder) {
        $this.LoadOrder = [System.Collections.Generic.List[string]]::new($moduleOrder)

        # Validation de l'ordre de chargement
        $loadedModules = @{}
        foreach ($moduleName in $this.LoadOrder) {
            if (-not $this.Modules.ContainsKey($moduleName)) {
                throw "Module '$moduleName' in load order not registered"
            }

            $module = $this.Modules[$moduleName]
            foreach ($dependency in $module.Dependencies) {
                if (-not $loadedModules.ContainsKey($dependency)) {
                    throw "Dependency '$dependency' for module '$moduleName' not loaded before it"
                }
            }

            $loadedModules[$moduleName] = $true
        }

        Write-Host "Load order validated: $($this.LoadOrder -join ' -> ')" -ForegroundColor Green
    }

    [void]LoadModules() {
        Write-Host "Loading modules in specified order..." -ForegroundColor Cyan

        foreach ($moduleName in $this.LoadOrder) {
            $this.LoadModule($moduleName)
        }

        Write-Host "All modules loaded successfully" -ForegroundColor Green
    }

    hidden [void]LoadModule([string]$moduleName) {
        $module = $this.Modules[$moduleName]

        if ($module.Loaded) {
            Write-Debug "Module '$moduleName' already loaded"
            return
        }

        Write-Host "Loading module: $moduleName" -ForegroundColor Yellow

        # Chargement des dépendances (déjà validé par l'ordre de chargement)
        foreach ($dependency in $module.Dependencies) {
            if (-not $this.Modules[$dependency].Loaded) {
                throw "Dependency '$dependency' not loaded for module '$moduleName'"
            }
        }

        try {
            $startTime = Get-Date

            # Initialisation du module
            if ($this.ModuleInitializers.ContainsKey($moduleName)) {
                & $this.ModuleInitializers[$moduleName]
            }

            # Chargement des fonctions
            foreach ($functionName in $module.Functions.Keys) {
                $functionCode = $module.Functions[$functionName]
                Set-Item -Path "function:global:$functionName" -Value $functionCode
            }

            # Chargement des cmdlets (simulation)
            foreach ($cmdlet in $module.Cmdlets) {
                # Dans un vrai framework, ceci enregistrerait le cmdlet
                Write-Debug "Cmdlet '$cmdlet' would be registered here"
            }

            $loadTime = (Get-Date) - $startTime
            $module.Loaded = $true
            $module.LoadTime = $loadTime
            $module.Status = 'Loaded'

            Write-Host "Module '$moduleName' loaded in $([math]::Round($loadTime.TotalMilliseconds, 2))ms" -ForegroundColor Green

        } catch {
            $module.Status = 'Failed'
            Write-Error "Failed to load module '$moduleName': $_"
            throw
        }
    }

    [hashtable]GetModuleInfo([string]$moduleName) {
        if (-not $this.Modules.ContainsKey($moduleName)) {
            return $null
        }

        $module = $this.Modules[$moduleName]
        return @{
            Name = $module.ModuleName
            Version = $module.Version
            Description = $module.Description
            Dependencies = $module.Dependencies
            Loaded = $module.Loaded
            LoadTime = $module.LoadTime
            Status = $module.Status
            Functions = $module.Functions.Keys
            Cmdlets = $module.Cmdlets
        }
    }

    [hashtable[]]GetAllModulesInfo() {
        $result = @()
        foreach ($moduleName in $this.Modules.Keys) {
            $result += $this.GetModuleInfo($moduleName)
        }
        return $result
    }

    [void]UnloadModule([string]$moduleName) {
        if (-not $this.Modules.ContainsKey($moduleName)) {
            throw "Module '$moduleName' not registered"
        }

        $module = $this.Modules[$moduleName]

        if (-not $module.Loaded) {
            Write-Warning "Module '$moduleName' not loaded"
            return
        }

        # Vérification des dépendances inverses
        $dependentModules = @()
        foreach ($otherModuleName in $this.Modules.Keys) {
            if ($otherModuleName -ne $moduleName -and $this.Modules[$otherModuleName].Dependencies -contains $moduleName) {
                $dependentModules += $otherModuleName
            }
        }

        if ($dependentModules) {
            throw "Cannot unload module '$moduleName': still required by $($dependentModules -join ', ')"
        }

        # Déchargement des fonctions
        foreach ($functionName in $module.Functions.Keys) {
            if (Get-Command $functionName -ErrorAction SilentlyContinue) {
                Remove-Item "function:global:$functionName" -ErrorAction SilentlyContinue
            }
        }

        $module.Loaded = $false
        $module.Status = 'Unloaded'

        Write-Host "Module '$moduleName' unloaded" -ForegroundColor Yellow
    }

    [string]GenerateDependencyGraph() {
        $graph = "digraph ModuleDependencies {`n"

        foreach ($moduleName in $this.Modules.Keys) {
            $module = $this.Modules[$moduleName]

            foreach ($dependency in $module.Dependencies) {
                $graph += "  `"$dependency`" -> `"$moduleName`";`n"
            }
        }

        $graph += "}"

        return $graph
    }
}

# Exemple d'utilisation du framework modulaire
$framework = [PowerShellModuleFramework]::new()

# Définition des modules
$framework.RegisterModule("Core.Logging", @{
    Version = "1.0.0"
    Description = "Core logging functionality"
    Dependencies = @()
    Functions = @{
        'Write-StructuredLog' = {
            param($Message, $Level = "INFO", $Context = @{})
            $logEntry = @{
                Timestamp = Get-Date
                Level = $Level
                Message = $Message
                Context = $Context
            }
            $logEntry | ConvertTo-Json | Out-File "C:\Logs\app.log" -Append
        }
    }
    Cmdlets = @()
})

$framework.RegisterModule("DataAccess.Sql", @{
    Version = "2.1.0"
    Description = "SQL Server data access layer"
    Dependencies = @("Core.Logging")
    Functions = @{
        'Invoke-SqlQuery' = {
            param($Query, $ConnectionString)
            Write-StructuredLog "Executing SQL query" -Context @{ Query = $Query }

            # Simulation d'exécution SQL
            return @{ Result = "Query executed"; RowsAffected = 42 }
        }
    }
    Cmdlets = @("Get-SqlData", "Set-SqlData")
})

$framework.RegisterModule("Web.ApiClient", @{
    Version = "1.5.0"
    Description = "REST API client functionality"
    Dependencies = @("Core.Logging")
    Functions = @{
        'Invoke-RestApi' = {
            param($Uri, $Method = "GET", $Body = $null)
            Write-StructuredLog "Calling REST API" -Context @{ Uri = $Uri; Method = $Method }

            # Simulation d'appel API
            return @{ StatusCode = 200; Content = "API response" }
        }
    }
    Cmdlets = @("Invoke-WebRequestEx", "Send-HttpRequest")
})

$framework.RegisterModule("Business.UserManagement", @{
    Version = "3.0.0"
    Description = "User management business logic"
    Dependencies = @("DataAccess.Sql", "Web.ApiClient", "Core.Logging")
    Functions = @{
        'New-User' = {
            param($UserName, $Email, $Department)
            Write-StructuredLog "Creating new user" -Context @{ UserName = $UserName; Email = $Email }

            # Simulation de création d'utilisateur
            $userId = [guid]::NewGuid()
            Invoke-SqlQuery "INSERT INTO Users..." "Server=localhost;Database=Users"

            return @{ UserId = $userId; Status = "Created" }
        }
    }
    Cmdlets = @("New-ADUserEx", "Set-UserPermissions")
})

# Définition des initialiseurs de modules
$framework.SetModuleInitializer("Core.Logging", {
    Write-StructuredLog "Core.Logging module initialized"
})

$framework.SetModuleInitializer("DataAccess.Sql", {
    Write-StructuredLog "DataAccess.Sql module initialized" -Context @{ DatabaseServer = "localhost" }
})

# Définition de l'ordre de chargement
$framework.SetLoadOrder(@("Core.Logging", "DataAccess.Sql", "Web.ApiClient", "Business.UserManagement"))

# Chargement des modules
$framework.LoadModules()

# Utilisation des modules chargés
$userResult = New-User -UserName "john.doe" -Email "john.doe@company.com" -Department "IT"
Write-Host "User created: $($userResult.UserId)" -ForegroundColor Green

# Informations sur les modules
$moduleInfos = $framework.GetAllModulesInfo()
Write-Host "`n=== LOADED MODULES ===" -ForegroundColor Cyan
foreach ($moduleInfo in $moduleInfos) {
    if ($moduleInfo.Loaded) {
        Write-Host "$($moduleInfo.Name) v$($moduleInfo.Version) - $($moduleInfo.Status)" -ForegroundColor Green
    }
}

# Génération du graphe de dépendances
$dependencyGraph = $framework.GenerateDependencyGraph()
$dependencyGraph | Out-File "ModuleDependencies.dot" -Encoding UTF8
Write-Host "`nDependency graph saved to ModuleDependencies.dot" -ForegroundColor Green

# Déchargement d'un module (démonstration)
try {
    $framework.UnloadModule("Business.UserManagement")
    Write-Host "Successfully unloaded Business.UserManagement" -ForegroundColor Yellow
} catch {
    Write-Warning "Could not unload module: $_"
}
```

### 1.2 Design patterns avancés en PowerShell

**Implémentation de patterns de conception :**
```powershell
# Pattern Strategy - Encapsulation d'algorithmes interchangeables
class CompressionStrategy {
    [string]Compress([string]$data) { throw "Compress method must be implemented" }
    [string]Decompress([string]$data) { throw "Decompress method must be implemented" }
}

class GZipCompression : CompressionStrategy {
    [string]Compress([string]$data) {
        $bytes = [System.Text.Encoding]::UTF8.GetBytes($data)
        $ms = [System.IO.MemoryStream]::new()
        $gzip = [System.IO.Compression.GZipStream]::new($ms, [System.IO.Compression.CompressionMode]::Compress)
        $gzip.Write($bytes, 0, $bytes.Length)
        $gzip.Close()
        return [Convert]::ToBase64String($ms.ToArray())
    }

    [string]Decompress([string]$data) {
        $bytes = [Convert]::FromBase64String($data)
        $ms = [System.IO.MemoryStream]::new($bytes)
        $gzip = [System.IO.Compression.GZipStream]::new($ms, [System.IO.Compression.CompressionMode]::Decompress)
        $sr = [System.IO.StreamReader]::new($gzip)
        return $sr.ReadToEnd()
    }
}

class DeflateCompression : CompressionStrategy {
    [string]Compress([string]$data) {
        $bytes = [System.Text.Encoding]::UTF8.GetBytes($data)
        $ms = [System.IO.MemoryStream]::new()
        $deflate = [System.IO.Compression.DeflateStream]::new($ms, [System.IO.Compression.CompressionMode]::Compress)
        $deflate.Write($bytes, 0, $bytes.Length)
        $deflate.Close()
        return [Convert]::ToBase64String($ms.ToArray())
    }

    [string]Decompress([string]$data) {
        $bytes = [Convert]::FromBase64String($data)
        $ms = [System.IO.MemoryStream]::new($bytes)
        $deflate = [System.IO.Compression.DeflateStream]::new($ms, [System.IO.Compression.CompressionMode]::Decompress)
        $sr = [System.IO.StreamReader]::new($deflate)
        return $sr.ReadToEnd()
    }
}

class DataCompressor {
    [CompressionStrategy]$Strategy

    DataCompressor([CompressionStrategy]$strategy) {
        $this.Strategy = $strategy
    }

    [void]SetStrategy([CompressionStrategy]$strategy) {
        $this.Strategy = $strategy
    }

    [string]Compress([string]$data) {
        return $this.Strategy.Compress($data)
    }

    [string]Decompress([string]$data) {
        return $this.Strategy.Decompress($data)
    }
}

# Pattern Observer - Notification automatique des changements
class IObserver {
    [void]Update([object]$sender, [hashtable]$eventData) { throw "Update method must be implemented" }
}

class Subject {
    [System.Collections.Generic.List[IObserver]]$Observers

    Subject() {
        $this.Observers = [System.Collections.Generic.List[IObserver]]::new()
    }

    [void]Attach([IObserver]$observer) {
        $this.Observers.Add($observer)
    }

    [void]Detach([IObserver]$observer) {
        $this.Observers.Remove($observer)
    }

    [void]Notify([hashtable]$eventData) {
        foreach ($observer in $this.Observers) {
            $observer.Update($this, $eventData)
        }
    }
}

class FileSystemWatcher : Subject {
    [System.IO.FileSystemWatcher]$Watcher

    FileSystemWatcher([string]$path) {
        $this.Watcher = [System.IO.FileSystemWatcher]::new($path)
        $this.Watcher.IncludeSubdirectories = $true
        $this.Watcher.EnableRaisingEvents = $true

        # Enregistrement des événements
        Register-ObjectEvent -InputObject $this.Watcher -EventName "Created" -Action {
            $sender = $args[0]
            $e = $args[1]
            $sender.Notify(@{ EventType = "FileCreated"; FileName = $e.Name; FullPath = $e.FullPath })
        }

        Register-ObjectEvent -InputObject $this.Watcher -EventName "Changed" -Action {
            $sender = $args[0]
            $e = $args[1]
            $sender.Notify(@{ EventType = "FileChanged"; FileName = $e.Name; FullPath = $e.FullPath })
        }

        Register-ObjectEvent -InputObject $this.Watcher -EventName "Deleted" -Action {
            $sender = $args[0]
            $e = $args[1]
            $sender.Notify(@{ EventType = "FileDeleted"; FileName = $e.Name; FullPath = $e.FullPath })
        }
    }
}

class LogObserver : IObserver {
    [string]$LogPath

    LogObserver([string]$logPath) {
        $this.LogPath = $logPath
    }

    [void]Update([object]$sender, [hashtable]$eventData) {
        $logEntry = "$(Get-Date) - $($eventData.EventType): $($eventData.FullPath)"
        $logEntry | Out-File -FilePath $this.LogPath -Append
    }
}

class EmailObserver : IObserver {
    [string]$EmailAddress

    EmailObserver([string]$emailAddress) {
        $this.EmailAddress = $emailAddress
    }

    [void]Update([object]$sender, [hashtable]$eventData) {
        $subject = "File System Event: $($eventData.EventType)"
        $body = "Event: $($eventData.EventType)`nFile: $($eventData.FullPath)`nTime: $(Get-Date)"

        Send-MailMessage -To $this.EmailAddress -Subject $subject -Body $body -SmtpServer "smtp.company.com"
    }
}

# Pattern Factory - Création d'objets spécialisée
class ConnectionFactory {
    static [object]CreateConnection([string]$type, [hashtable]$parameters) {
        switch ($type.ToLower()) {
            "sql" {
                $connectionString = "Server=$($parameters.Server);Database=$($parameters.Database);Integrated Security=True;"
                return [System.Data.SqlClient.SqlConnection]::new($connectionString)
            }
            "oracle" {
                $connectionString = "Data Source=$($parameters.DataSource);User Id=$($parameters.UserId);Password=$($parameters.Password);"
                # Simulation pour Oracle
                return @{ Type = "Oracle"; ConnectionString = $connectionString }
            }
            "mysql" {
                $connectionString = "Server=$($parameters.Server);Database=$($parameters.Database);Uid=$($parameters.User);Pwd=$($parameters.Password);"
                # Simulation pour MySQL
                return @{ Type = "MySQL"; ConnectionString = $connectionString }
            }
            default {
                throw "Unsupported connection type: $type"
            }
        }
    }
}

# Pattern Singleton - Instance unique garantie
class ConfigurationManager {
    static [ConfigurationManager]$Instance = $null
    [hashtable]$Settings
    hidden ConfigurationManager() {
        $this.Settings = @{
            AppName = "PowerShell Application"
            Version = "1.0.0"
            Environment = "Production"
            LogLevel = "INFO"
            MaxRetries = 3
        }
    }

    static [ConfigurationManager]GetInstance() {
        if ($null -eq [ConfigurationManager]::Instance) {
            [ConfigurationManager]::Instance = [ConfigurationManager]::new()
        }
        return [ConfigurationManager]::Instance
    }

    [object]GetSetting([string]$key) {
        return $this.Settings[$key]
    }

    [void]SetSetting([string]$key, [object]$value) {
        $this.Settings[$key] = $value
    }

    [hashtable]GetAllSettings() {
        return $this.Settings.Clone()
    }
}

# Pattern Decorator - Extension fonctionnelle sans modification
class IDataProcessor {
    [object]Process([object]$data) { throw "Process method must be implemented" }
}

class BaseDataProcessor : IDataProcessor {
    [object]Process([object]$data) {
        return $data
    }
}

class LoggingDecorator : IDataProcessor {
    [IDataProcessor]$WrappedProcessor
    [string]$LogPath

    LoggingDecorator([IDataProcessor]$processor, [string]$logPath) {
        $this.WrappedProcessor = $processor
        $this.LogPath = $logPath
    }

    [object]Process([object]$data) {
        $startTime = Get-Date
        Write-Host "Processing data..." -ForegroundColor Gray

        try {
            $result = $this.WrappedProcessor.Process($data)

            $duration = (Get-Date) - $startTime
            $logEntry = "$(Get-Date) - SUCCESS - Duration: $([math]::Round($duration.TotalSeconds, 2))s"
            $logEntry | Out-File -FilePath $this.LogPath -Append

            return $result
        } catch {
            $logEntry = "$(Get-Date) - ERROR - $($_.Exception.Message)"
            $logEntry | Out-File -FilePath $this.LogPath -Append
            throw
        }
    }
}

class ValidationDecorator : IDataProcessor {
    [IDataProcessor]$WrappedProcessor
    [scriptblock]$ValidationRule

    ValidationDecorator([IDataProcessor]$processor, [scriptblock]$validationRule) {
        $this.WrappedProcessor = $processor
        $this.ValidationRule = $validationRule
    }

    [object]Process([object]$data) {
        # Validation d'entrée
        $validationResult = & $this.ValidationRule $data
        if (-not $validationResult.IsValid) {
            throw "Validation failed: $($validationResult.ErrorMessage)"
        }

        $result = $this.WrappedProcessor.Process($data)

        # Validation de sortie
        $outputValidation = & $this.ValidationRule $result
        if (-not $outputValidation.IsValid) {
            throw "Output validation failed: $($outputValidation.ErrorMessage)"
        }

        return $result
    }
}

class CachingDecorator : IDataProcessor {
    [IDataProcessor]$WrappedProcessor
    [System.Collections.Generic.Dictionary[string, object]]$Cache
    [scriptblock]$CacheKeyGenerator

    CachingDecorator([IDataProcessor]$processor, [scriptblock]$cacheKeyGenerator) {
        $this.WrappedProcessor = $processor
        $this.Cache = [System.Collections.Generic.Dictionary[string, object]]::new()
        $this.CacheKeyGenerator = $cacheKeyGenerator
    }

    [object]Process([object]$data) {
        $cacheKey = & $this.CacheKeyGenerator $data

        if ($this.Cache.ContainsKey($cacheKey)) {
            Write-Host "Returning cached result for key: $cacheKey" -ForegroundColor Blue
            return $this.Cache[$cacheKey]
        }

        $result = $this.WrappedProcessor.Process($data)
        $this.Cache[$cacheKey] = $result

        return $result
    }
}

# Exemple d'utilisation des patterns
Write-Host "=== DESIGN PATTERNS DEMONSTRATION ===" -ForegroundColor Cyan

# Strategy Pattern
Write-Host "`n1. Strategy Pattern - Compression:" -ForegroundColor Yellow
$compressor = [DataCompressor]::new([GZipCompression]::new())
$testData = "This is a test string for compression algorithms" * 100
$compressed = $compressor.Compress($testData)
$decompressed = $compressor.Decompress($compressed)
Write-Host "Original length: $($testData.Length), Compressed length: $($compressed.Length)" -ForegroundColor Green

# Changer de stratégie
$compressor.SetStrategy([DeflateCompression]::new())
$compressed2 = $compressor.Compress($testData)
Write-Host "Deflate compressed length: $($compressed2.Length)" -ForegroundColor Green

# Observer Pattern
Write-Host "`n2. Observer Pattern - File System Monitoring:" -ForegroundColor Yellow
$watcher = [FileSystemWatcher]::new("C:\Temp")

$logObserver = [LogObserver]::new("C:\Temp\filesystem.log")
$emailObserver = [EmailObserver]::new("admin@company.com")

$watcher.Attach($logObserver)
$watcher.Attach($emailObserver)

Write-Host "File system monitoring started. Create/modify/delete files in C:\Temp to see notifications." -ForegroundColor Green

# Factory Pattern
Write-Host "`n3. Factory Pattern - Database Connections:" -ForegroundColor Yellow
$sqlConn = [ConnectionFactory]::CreateConnection("sql", @{
    Server = "localhost"
    Database = "MyApp"
})
Write-Host "SQL Connection created: $($sqlConn.ConnectionString)" -ForegroundColor Green

# Singleton Pattern
Write-Host "`n4. Singleton Pattern - Configuration:" -ForegroundColor Yellow
$config1 = [ConfigurationManager]::GetInstance()
$config2 = [ConfigurationManager]::GetInstance()
$config1.SetSetting("TestKey", "TestValue")
Write-Host "Same instance: $($config1 -eq $config2)" -ForegroundColor Green
Write-Host "Configuration value: $($config2.GetSetting('TestKey'))" -ForegroundColor Green

# Decorator Pattern
Write-Host "`n5. Decorator Pattern - Data Processing:" -ForegroundColor Yellow
$baseProcessor = [BaseDataProcessor]::new()

# Ajouter logging
$loggedProcessor = [LoggingDecorator]::new($baseProcessor, "C:\Temp\processing.log")

# Ajouter validation
$validationRule = {
    param($data)
    if ($data -is [string] -and $data.Length -gt 0) {
        return @{ IsValid = $true }
    } else {
        return @{ IsValid = $false; ErrorMessage = "Data must be non-empty string" }
    }
}
$validatedProcessor = [ValidationDecorator]::new($loggedProcessor, $validationRule)

# Ajouter cache
$cacheKeyGen = { param($data) return [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($data)) }
$cachedProcessor = [CachingDecorator]::new($validatedProcessor, $cacheKeyGen)

# Utilisation
$result1 = $cachedProcessor.Process("Hello World")
$result2 = $cachedProcessor.Process("Hello World")  # Doit venir du cache
Write-Host "Processing completed with decorators applied" -ForegroundColor Green

Write-Host "`nDesign patterns demonstration completed!" -ForegroundColor Green
```

## Section 2 : Standards de codage et qualité

### 2.1 Style guide et conventions

**Framework de validation de code PowerShell :**
```powershell
# Analyseur de code PowerShell avec règles de style
class PowerShellCodeAnalyzer {
    [System.Collections.Generic.List[PSCustomObject]]$Rules
    [hashtable]$SeverityLevels
    [System.Collections.Generic.List[PSCustomObject]]$Violations

    PowerShellCodeAnalyzer() {
        $this.Rules = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.SeverityLevels = @{
            "Error" = 4
            "Warning" = 3
            "Info" = 2
            "Verbose" = 1
        }
        $this.Violations = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.InitializeDefaultRules()
    }

    hidden [void]InitializeDefaultRules() {
        # Règle 1: Noms de fonctions en PascalCase
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS001"
            Name = "FunctionNamingConvention"
            Description = "Function names should use PascalCase"
            Severity = "Warning"
            Pattern = 'function\s+([a-zA-Z][a-zA-Z0-9]*)\s*\('
            Validator = {
                param($matches)
                $functionName = $matches[1]
                # Vérifier PascalCase (première lettre majuscule)
                return $functionName -cmatch '^[A-Z][a-zA-Z0-9]*$'
            }
            Message = "Function name '{0}' should use PascalCase"
        })

        # Règle 2: Utilisation de Write-Host interdite
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS002"
            Name = "NoWriteHost"
            Description = "Avoid using Write-Host for production code"
            Severity = "Warning"
            Pattern = '\bWrite-Host\b'
            Validator = { param($matches) return $false }  # Toujours une violation
            Message = "Write-Host usage is discouraged in production code"
        })

        # Règle 3: Paramètres avec validation
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS003"
            Name = "ParameterValidation"
            Description = "Parameters should have validation attributes"
            Severity = "Info"
            Pattern = 'param\s*\(\s*(\$[a-zA-Z][a-zA-Z0-9]*)\s+([^)]*)\)'
            Validator = {
                param($matches)
                $paramDeclaration = $matches[2]
                # Vérifier présence de [Parameter] ou [Validate]
                return $paramDeclaration -match '\[Parameter\]' -or $paramDeclaration -match '\[Validate'
            }
            Message = "Parameter '{0}' should have validation attributes"
        })

        # Règle 4: Gestion d'erreurs appropriée
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS004"
            Name = "ErrorHandling"
            Description = "Functions should have proper error handling"
            Severity = "Warning"
            Pattern = 'function\s+[^{]*\{([^}]*)\}'
            Validator = {
                param($matches)
                $functionBody = $matches[1]
                # Vérifier présence de try-catch ou validation d'erreurs
                return $functionBody -match '\btry\b' -or $functionBody -match '\$ErrorActionPreference' -or $functionBody -match '\bthrow\b'
            }
            Message = "Function should include proper error handling"
        })

        # Règle 5: Commentaires de fonctions
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS005"
            Name = "FunctionDocumentation"
            Description = "Functions should have comment-based help"
            Severity = "Info"
            Pattern = 'function\s+[^{]*\{'
            Validator = {
                param($matches)
                # Cette règle nécessiterait une analyse plus complexe du contexte
                # Pour la démonstration, on retourne toujours true
                return $true
            }
            Message = "Consider adding comment-based help for function"
        })

        # Règle 6: Éviter les variables non initialisées
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS006"
            Name = "VariableInitialization"
            Description = "Variables should be initialized before use"
            Severity = "Warning"
            Pattern = '(\$[a-zA-Z][a-zA-Z0-9]*)\s*='
            Validator = {
                param($matches)
                $variableName = $matches[1]
                # Cette validation nécessiterait une analyse de flux plus complexe
                return $true
            }
            Message = "Variable '{0}' should be initialized before use"
        })

        # Règle 7: Utilisation appropriée des alias
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS007"
            Name = "AliasUsage"
            Description = "Avoid problematic aliases in production code"
            Severity = "Info"
            Pattern = '\b(%|?|foreach|select|sort|group)\b'
            Validator = { param($matches) return $false }  # Violation si alias trouvé
            Message = "Consider using full cmdlet names instead of aliases for better readability"
        })

        # Règle 8: Longueur de ligne raisonnable
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS008"
            Name = "LineLength"
            Description = "Lines should not exceed 120 characters"
            Severity = "Info"
            Pattern = '.*'
            Validator = {
                param($matches, $line)
                return $line.Length -le 120
            }
            Message = "Line exceeds 120 characters ({0} characters)"
        })

        # Règle 9: Utilisation de splatting
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS009"
            Name = "SplattingUsage"
            Description = "Consider using splatting for commands with many parameters"
            Severity = "Info"
            Pattern = '\w+\s+-\w+\s+-\w+\s+-\w+\s+-\w+'
            Validator = { param($matches) return $false }  # Violation si paramètres multiples
            Message = "Consider using splatting (@param) for better readability"
        })

        # Règle 10: Tests unitaires
        $this.Rules.Add([PSCustomObject]@{
            Id = "PS010"
            Name = "UnitTests"
            Description = "Code should have corresponding unit tests"
            Severity = "Info"
            Pattern = 'function\s+[^{]*\{'
            Validator = {
                param($matches)
                # Cette règle nécessiterait de vérifier l'existence de fichiers de test
                return $true
            }
            Message = "Consider adding unit tests for function"
        })
    }

    [void]AnalyzeFile([string]$filePath) {
        if (-not (Test-Path $filePath)) {
            throw "File not found: $filePath"
        }

        $content = Get-Content $filePath -Raw
        $lines = Get-Content $filePath

        Write-Host "Analyzing file: $filePath" -ForegroundColor Cyan

        $lineNumber = 0
        foreach ($line in $lines) {
            $lineNumber++
            $this.AnalyzeLine($line, $lineNumber, $filePath)
        }

        Write-Host "Analysis completed. Found $($this.Violations.Count) violations." -ForegroundColor Green
    }

    hidden [void]AnalyzeLine([string]$line, [int]$lineNumber, [string]$filePath) {
        foreach ($rule in $this.Rules) {
            $matches = [regex]::Matches($line, $rule.Pattern)

            if ($matches.Count -gt 0) {
                foreach ($match in $matches) {
                    $isValid = & $rule.Validator $match.Groups

                    if (-not $isValid) {
                        $violation = [PSCustomObject]@{
                            RuleId = $rule.Id
                            RuleName = $rule.Name
                            Severity = $rule.Severity
                            FilePath = $filePath
                            LineNumber = $lineNumber
                            LineContent = $line.Trim()
                            Message = $rule.Message -f $match.Groups[1].Value
                            Timestamp = Get-Date
                        }

                        $this.Violations.Add($violation)
                    }
                }
            }
        }
    }

    [PSCustomObject[]]GetViolations([string]$severity = $null) {
        $violations = $this.Violations

        if ($severity) {
            $violations = $violations | Where-Object { $_.Severity -eq $severity }
        }

        return $violations | Sort-Object Severity, FilePath, LineNumber
    }

    [hashtable]GetSummary() {
        $violations = $this.Violations

        $summary = @{
            TotalViolations = $violations.Count
            FilesAnalyzed = ($violations | Select-Object -ExpandProperty FilePath -Unique).Count
            ViolationsBySeverity = @{}
            ViolationsByRule = @{}
            MostProblematicFiles = @()
        }

        # Violations par sévérité
        $summary.ViolationsBySeverity = $violations | Group-Object Severity | ForEach-Object {
            @{ $_.Name = $_.Count }
        }

        # Violations par règle
        $summary.ViolationsByRule = $violations | Group-Object RuleName | ForEach-Object {
            @{ $_.Name = $_.Count }
        }

        # Fichiers les plus problématiques
        $summary.MostProblematicFiles = $violations | Group-Object FilePath | Sort-Object Count -Descending | Select-Object -First 5 | ForEach-Object {
            @{ File = $_.Name; Violations = $_.Count }
        }

        return $summary
    }

    [string]GenerateReport() {
        $summary = $this.GetSummary()
        $violations = $this.GetViolations()

        $report = @"
POWERSHELL CODE ANALYSIS REPORT
Generated: $(Get-Date)

SUMMARY
=======
Files Analyzed: $($summary.FilesAnalyzed)
Total Violations: $($summary.TotalViolations)

VIOLATIONS BY SEVERITY
$($summary.ViolationsBySeverity | ForEach-Object { "$($_.Keys[0]): $($_.Values[0])" } | Out-String)

VIOLATIONS BY RULE
$($summary.ViolationsByRule | ForEach-Object { "$($_.Keys[0]): $($_.Values[0])" } | Out-String)

MOST PROBLEMATIC FILES
$($summary.MostProblematicFiles | ForEach-Object { "• $($_.File): $($_.Violations) violations" } | Out-String)

DETAILED VIOLATIONS
==================
"@

        foreach ($violation in $violations | Select-Object -First 20) {
            $report += @"

$($violation.RuleId) - $($violation.RuleName) [$($violation.Severity)]
File: $($violation.FilePath):$($violation.LineNumber)
$($violation.Message)
Line: $($violation.LineContent)
"@
        }

        if ($violations.Count -gt 20) {
            $report += "`n... and $($violations.Count - 20) more violations`n"
        }

        return $report
    }

    [void]ExportResults([string]$outputPath) {
        $results = @{
            Summary = $this.GetSummary()
            Violations = $this.GetViolations()
            GeneratedAt = Get-Date
        }

        $results | ConvertTo-Json -Depth 10 | Out-File -FilePath $outputPath -Encoding UTF8
        Write-Host "Analysis results exported to: $outputPath" -ForegroundColor Green
    }
}

# Fonction d'analyse de code avec rapport intégré
function Invoke-PowerShellCodeAnalysis {
    param(
        [Parameter(Mandatory=$true)]
        [string[]]$Path,

        [string]$OutputPath = $null,

        [switch]$Recurse,

        [string]$Severity = $null
    )

    $analyzer = [PowerShellCodeAnalyzer]::new()

    foreach ($filePath in $Path) {
        if ((Get-Item $filePath) -is [System.IO.DirectoryInfo]) {
            $files = Get-ChildItem $filePath -Filter "*.ps1" -Recurse:$Recurse
            foreach ($file in $files) {
                $analyzer.AnalyzeFile($file.FullName)
            }
        } else {
            $analyzer.AnalyzeFile($filePath)
        }
    }

    # Génération du rapport
    $report = $analyzer.GenerateReport()
    Write-Host "`n=== POWERSHELL CODE ANALYSIS REPORT ===" -ForegroundColor Cyan
    Write-Host $report

    # Filtrage par sévérité si demandé
    if ($Severity) {
        $filteredViolations = $analyzer.GetViolations($Severity)
        Write-Host "`n=== VIOLATIONS FILTERED BY SEVERITY: $Severity ===" -ForegroundColor Yellow
        $filteredViolations | Format-Table RuleId, RuleName, FilePath, LineNumber, Message -AutoSize
    }

    # Export si demandé
    if ($OutputPath) {
        $analyzer.ExportResults($OutputPath)
    }

    return $analyzer
}

# Fonction de correction automatique de certaines violations
function Repair-PowerShellCode {
    param(
        [Parameter(Mandatory=$true)]
        [string]$FilePath,

        [switch]$BackupOriginal
    )

    if (-not (Test-Path $FilePath)) {
        throw "File not found: $FilePath"
    }

    $content = Get-Content $FilePath -Raw

    if ($BackupOriginal) {
        Copy-Item $FilePath "$FilePath.backup" -Force
        Write-Host "Original file backed up to: $FilePath.backup" -ForegroundColor Yellow
    }

    # Corrections automatiques
    $corrections = @(
        # Remplacer Write-Host par Write-Output dans certains cas
        @{
            Pattern = '\bWrite-Host\s+([^;]+);'
            Replacement = 'Write-Output $1;'
            Description = "Replace Write-Host with Write-Output"
        },
        # Ajouter [CmdletBinding()] aux fonctions avancées
        @{
            Pattern = 'function\s+(\w+)\s*\(\s*param\s*\('
            Replacement = '[CmdletBinding()]`nfunction $1 {`n    param('
            Description = "Add CmdletBinding to advanced functions"
        }
    )

    $modified = $content
    $correctionsApplied = 0

    foreach ($correction in $corrections) {
        $matches = [regex]::Matches($modified, $correction.Pattern)
        if ($matches.Count -gt 0) {
            $modified = [regex]::Replace($modified, $correction.Pattern, $correction.Replacement)
            $correctionsApplied += $matches.Count
            Write-Host "Applied correction: $($correction.Description) ($($matches.Count) instances)" -ForegroundColor Green
        }
    }

    if ($correctionsApplied -gt 0) {
        $modified | Out-File -FilePath $FilePath -Encoding UTF8
        Write-Host "Applied $correctionsApplied automatic corrections to $FilePath" -ForegroundColor Green
    } else {
        Write-Host "No automatic corrections could be applied to $FilePath" -ForegroundColor Yellow
    }

    return $correctionsApplied
}

# Exemple d'utilisation de l'analyseur de code
Write-Host "=== POWERSHELL CODE ANALYSIS DEMO ===" -ForegroundColor Cyan

# Création d'un fichier de test avec des violations
$testFile = @"
function testFunction {
    param($input)
    Write-Host "Processing $input"
    $result = Get-Process | Where-Object { $_.Name -like "*$input*" } | Select-Object -First 5
    return $result
}

function anotherFunction($param1, $param2, $param3, $param4, $param5) {
    Write-Host "This function has many parameters"
    $veryLongVariableNameThatExceedsReasonableLengthAndMakesCodeHardToRead = "test"
    return $param1 + $param2 + $param3 + $param4 + $param5
}
"@

$testFile | Out-File -FilePath "TestCode.ps1" -Encoding UTF8

# Analyse du fichier
$analyzer = Invoke-PowerShellCodeAnalysis -Path "TestCode.ps1" -OutputPath "CodeAnalysisResults.json"

# Affichage des violations par sévérité
Write-Host "`n=== VIOLATIONS BY SEVERITY ===" -ForegroundColor Yellow
$analyzer.GetViolations() | Group-Object Severity | ForEach-Object {
    Write-Host "$($_.Name): $($_.Count) violations"
}

# Correction automatique
Write-Host "`n=== AUTOMATIC CODE REPAIR ===" -ForegroundColor Yellow
Repair-PowerShellCode -FilePath "TestCode.ps1" -BackupOriginal

# Réanalyse après correction
Write-Host "`n=== RE-ANALYSIS AFTER REPAIR ===" -ForegroundColor Yellow
$analyzer2 = Invoke-PowerShellCodeAnalysis -Path "TestCode.ps1"

Write-Host "`nCode analysis demonstration completed!" -ForegroundColor Green

# Nettoyage des fichiers de test
Remove-Item "TestCode.ps1" -ErrorAction SilentlyContinue
Remove-Item "TestCode.ps1.backup" -ErrorAction SilentlyContinue
Remove-Item "CodeAnalysisResults.json" -ErrorAction SilentlyContinue
```

### 2.2 Tests unitaires et TDD en PowerShell

**Framework de tests unitaires complet :**
```powershell
# Framework de tests unitaires PowerShell avancé
class PowerShellTestFramework {
    [System.Collections.Generic.List[PSCustomObject]]$TestSuites
    [System.Collections.Generic.List[PSCustomObject]]$TestResults
    [hashtable]$GlobalSetup
    [hashtable]$GlobalTeardown
    [scriptblock]$TestDiscoveryPattern

    PowerShellTestFramework() {
        $this.TestSuites = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.TestResults = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.GlobalSetup = @{}
        $this.GlobalTeardown = @{}
        $this.TestDiscoveryPattern = { param($file) $file.Name -match '\.Tests\.ps1$' }
    }

    [void]AddTestSuite([string]$name, [scriptblock]$setup = $null, [scriptblock]$teardown = $null) {
        $suite = [PSCustomObject]@{
            Name = $name
            Setup = $setup
            Teardown = $teardown
            Tests = [System.Collections.Generic.List[PSCustomObject]]::new()
            ExecutionTime = 0
            Status = "Created"
        }

        $this.TestSuites.Add($suite)
    }

    [void]AddTest([string]$suiteName, [string]$testName, [scriptblock]$testBlock, [hashtable]$tags = @{}) {
        $suite = $this.TestSuites | Where-Object { $_.Name -eq $suiteName } | Select-Object -First 1

        if (-not $suite) {
            throw "Test suite '$suiteName' not found"
        }

        $test = [PSCustomObject]@{
            Name = $testName
            TestBlock = $testBlock
            Tags = $tags
            Status = "Pending"
            ExecutionTime = 0
            Error = $null
            Assertions = 0
            Failures = 0
        }

        $suite.Tests.Add($test)
    }

    [void]RunTests([string[]]$suiteNames = $null, [string[]]$tags = $null) {
        Write-Host "=== POWERSHELL TEST EXECUTION ===" -ForegroundColor Cyan
        Write-Host "Started at: $(Get-Date)" -ForegroundColor Gray

        $suitesToRun = if ($suiteNames) {
            $this.TestSuites | Where-Object { $_.Name -in $suiteNames }
        } else {
            $this.TestSuites
        }

        foreach ($suite in $suitesToRun) {
            $this.RunTestSuite($suite, $tags)
        }

        $this.GenerateTestReport()
    }

    hidden [void]RunTestSuite([PSCustomObject]$suite, [string[]]$tags = $null) {
        Write-Host "`nRunning test suite: $($suite.Name)" -ForegroundColor Yellow

        $suite.Status = "Running"
        $suiteStartTime = Get-Date

        # Setup de la suite
        if ($suite.Setup) {
            try {
                & $suite.Setup
            } catch {
                Write-Error "Suite setup failed for '$($suite.Name)': $_"
                $suite.Status = "Failed"
                return
            }
        }

        # Filtrage des tests par tags
        $testsToRun = if ($tags) {
            $suite.Tests | Where-Object {
                $testTags = $_.Tags.Keys
                $tagsMatch = $true
                foreach ($tag in $tags) {
                    if ($tag -notin $testTags) {
                        $tagsMatch = $false
                        break
                    }
                }
                $tagsMatch
            }
        } else {
            $suite.Tests
        }

        Write-Host "Running $($testsToRun.Count) tests..." -ForegroundColor Gray

        foreach ($test in $testsToRun) {
            $this.RunTest($test, $suite)
        }

        # Teardown de la suite
        if ($suite.Teardown) {
            try {
                & $suite.Teardown
            } catch {
                Write-Warning "Suite teardown failed for '$($suite.Name)': $_"
            }
        }

        $suite.ExecutionTime = ((Get-Date) - $suiteStartTime).TotalSeconds
        $suite.Status = "Completed"

        Write-Host "Suite '$($suite.Name)' completed in $([math]::Round($suite.ExecutionTime, 2))s" -ForegroundColor Green
    }

    hidden [void]RunTest([PSCustomObject]$test, [PSCustomObject]$suite) {
        Write-Host "  Running test: $($test.Name)" -ForegroundColor Gray

        $test.Status = "Running"
        $testStartTime = Get-Date

        try {
            # Exécution du test dans un scope isolé
            $testContext = @{
                SuiteName = $suite.Name
                TestName = $test.Name
                Assertions = 0
                Failures = 0
            }

            # Définir les fonctions d'assertion dans le scope du test
            $assertEqual = {
                param($actual, $expected, $message = "")
                $testContext.Assertions++
                if ($actual -ne $expected) {
                    $testContext.Failures++
                    throw "Assertion failed: Expected '$expected', got '$actual'. $message"
                }
            }

            $assertTrue = {
                param($condition, $message = "")
                $testContext.Assertions++
                if (-not $condition) {
                    $testContext.Failures++
                    throw "Assertion failed: Condition is false. $message"
                }
            }

            $assertNotNull = {
                param($value, $message = "")
                $testContext.Assertions++
                if ($null -eq $value) {
                    $testContext.Failures++
                    throw "Assertion failed: Value is null. $message"
                }
            }

            $assertThrows = {
                param($scriptBlock, $expectedException = $null, $message = "")
                $testContext.Assertions++
                try {
                    & $scriptBlock
                    $testContext.Failures++
                    throw "Assertion failed: Expected exception was not thrown. $message"
                } catch {
                    if ($expectedException -and $_.Exception.GetType().Name -ne $expectedException) {
                        $testContext.Failures++
                        throw "Assertion failed: Expected exception '$expectedException', got '$($_.Exception.GetType().Name)'. $message"
                    }
                }
            }

            # Exécuter le test
            & $test.TestBlock

            $test.Status = if ($testContext.Failures -eq 0) { "Passed" } else { "Failed" }
            $test.Assertions = $testContext.Assertions
            $test.Failures = $testContext.Failures

            if ($test.Status -eq "Passed") {
                Write-Host "    ✓ PASSED ($($test.Assertions) assertions)" -ForegroundColor Green
            } else {
                Write-Host "    ✗ FAILED ($($test.Failures) failures)" -ForegroundColor Red
            }

        } catch {
            $test.Status = "Failed"
            $test.Error = $_.Exception.Message
            Write-Host "    ✗ FAILED: $($_.Exception.Message)" -ForegroundColor Red
        }

        $test.ExecutionTime = ((Get-Date) - $testStartTime).TotalSeconds

        # Enregistrer le résultat
        $result = [PSCustomObject]@{
            SuiteName = $suite.Name
            TestName = $test.Name
            Status = $test.Status
            ExecutionTime = $test.ExecutionTime
            Assertions = $test.Assertions
            Failures = $test.Failures
            Error = $test.Error
            Timestamp = Get-Date
        }

        $this.TestResults.Add($result)
    }

    [void]DiscoverAndRunTests([string]$path, [string[]]$tags = $null) {
        Write-Host "Discovering tests in: $path" -ForegroundColor Cyan

        $testFiles = Get-ChildItem $path -Recurse -Include "*.Tests.ps1"

        foreach ($testFile in $testFiles) {
            Write-Host "Loading test file: $($testFile.Name)" -ForegroundColor Gray
            . $testFile.FullName
        }

        $this.RunTests($null, $tags)
    }

    hidden [void]GenerateTestReport() {
        $totalTests = $this.TestResults.Count
        $passedTests = ($this.TestResults | Where-Object { $_.Status -eq "Passed" }).Count
        $failedTests = ($this.TestResults | Where-Object { $_.Status -eq "Failed" }).Count
        $totalAssertions = ($this.TestResults | Measure-Object Assertions -Sum).Sum
        $totalExecutionTime = ($this.TestResults | Measure-Object ExecutionTime -Sum).Sum

        Write-Host "`n=== TEST EXECUTION SUMMARY ===" -ForegroundColor Cyan
        Write-Host "Total tests: $totalTests"
        Write-Host "Passed: $passedTests" -ForegroundColor Green
        Write-Host "Failed: $failedTests" -ForegroundColor Red
        Write-Host "Success rate: $([math]::Round(($passedTests / $totalTests) * 100, 1))%"
        Write-Host "Total assertions: $totalAssertions"
        Write-Host "Total execution time: $([math]::Round($totalExecutionTime, 2))s"
        Write-Host "Average test time: $([math]::Round($totalExecutionTime / $totalTests, 3))s"

        # Rapport détaillé des échecs
        if ($failedTests -gt 0) {
            Write-Host "`n=== FAILED TESTS ===" -ForegroundColor Red
            $failedResults = $this.TestResults | Where-Object { $_.Status -eq "Failed" }
            foreach ($failed in $failedResults) {
                Write-Host "• $($failed.SuiteName).$($failed.TestName): $($failed.Error)" -ForegroundColor Red
            }
        }

        # Métriques par suite
        Write-Host "`n=== SUITE METRICS ===" -ForegroundColor Yellow
        foreach ($suite in $this.TestSuites) {
            $suiteResults = $this.TestResults | Where-Object { $_.SuiteName -eq $suite.Name }
            $suitePassed = ($suiteResults | Where-Object { $_.Status -eq "Passed" }).Count
            $suiteTotal = $suiteResults.Count

            if ($suiteTotal -gt 0) {
                $suiteRate = [math]::Round(($suitePassed / $suiteTotal) * 100, 1)
                Write-Host "$($suite.Name): $suitePassed/$suiteTotal tests passed ($suiteRate%) - $([math]::Round($suite.ExecutionTime, 2))s"
            }
        }
    }

    [hashtable]GetTestMetrics() {
        $metrics = @{
            TotalTests = $this.TestResults.Count
            PassedTests = ($this.TestResults | Where-Object { $_.Status -eq "Passed" }).Count
            FailedTests = ($this.TestResults | Where-Object { $_.Status -eq "Failed" }).Count
            TotalAssertions = ($this.TestResults | Measure-Object Assertions -Sum).Sum
            TotalExecutionTime = ($this.TestResults | Measure-Object ExecutionTime -Sum).Sum
            SuitesExecuted = $this.TestSuites.Count
        }

        $metrics.SuccessRate = if ($metrics.TotalTests -gt 0) {
            [math]::Round(($metrics.PassedTests / $metrics.TotalTests) * 100, 2)
        } else { 0 }

        $metrics.AverageTestTime = if ($metrics.TotalTests -gt 0) {
            [math]::Round($metrics.TotalExecutionTime / $metrics.TotalTests, 3)
        } else { 0 }

        return $metrics
    }

    [void]ExportResults([string]$outputPath) {
        $export = @{
            TestSuites = $this.TestSuites
            TestResults = $this.TestResults
            Metrics = $this.GetTestMetrics()
            GeneratedAt = Get-Date
        }

        $export | ConvertTo-Json -Depth 10 | Out-File -FilePath $outputPath -Encoding UTF8
        Write-Host "Test results exported to: $outputPath" -ForegroundColor Green
    }
}

# Fonctions utilitaires pour les tests
function Describe {
    param([string]$name, [scriptblock]$block)
    # Cette fonction serait définie globalement pour la syntaxe BDD
}

function It {
    param([string]$description, [scriptblock]$test)
    # Cette fonction serait définie globalement pour la syntaxe BDD
}

function Assert-Equal {
    param($actual, $expected, $message = "")
    # Fonction d'assertion globale
}

function Assert-True {
    param($condition, $message = "")
    # Fonction d'assertion globale
}

# Exemple d'utilisation du framework de tests
$testFramework = [PowerShellTestFramework]::new()

# Création d'une suite de tests
$testFramework.AddTestSuite("MathUtils", {
    # Setup de la suite
    Write-Host "Setting up MathUtils test suite..."
    . ".\MathUtils.ps1"  # Charger le module à tester
}, {
    # Teardown de la suite
    Write-Host "Cleaning up MathUtils test suite..."
})

# Ajout de tests
$testFramework.AddTest("MathUtils", "Add_PositiveNumbers", {
    $result = Add-Numbers 2 3
    Assert-Equal $result 5 "Addition should work for positive numbers"
}, @{ Category = "Arithmetic"; Priority = "High" })

$testFramework.AddTest("MathUtils", "Add_NegativeNumbers", {
    $result = Add-Numbers (-2) (-3)
    Assert-Equal $result (-5) "Addition should work for negative numbers"
}, @{ Category = "Arithmetic"; Priority = "High" })

$testFramework.AddTest("MathUtils", "Divide_ByZero", {
    Assert-Throws { Divide-Numbers 10 0 } "DivideByZeroException" "Division by zero should throw exception"
}, @{ Category = "ErrorHandling"; Priority = "Critical" })

$testFramework.AddTest("MathUtils", "Factorial_LargeNumber", {
    $result = Get-Factorial 5
    Assert-Equal $result 120 "Factorial of 5 should be 120"
}, @{ Category = "Advanced"; Priority = "Medium" })

# Suite de tests pour les opérations de chaînes
$testFramework.AddTestSuite("StringUtils", {
    Write-Host "Setting up StringUtils test suite..."
}, {
    Write-Host "Cleaning up StringUtils test suite..."
})

$testFramework.AddTest("StringUtils", "Reverse_String", {
    $result = Reverse-String "Hello"
    Assert-Equal $result "olleH" "String reversal should work correctly"
}, @{ Category = "String"; Priority = "Medium" })

$testFramework.AddTest("StringUtils", "IsPalindrome_Valid", {
    $result = Test-Palindrome "radar"
    Assert-True $result "radar should be detected as palindrome"
}, @{ Category = "String"; Priority = "Low" })

# Exécution des tests
$testFramework.RunTests()

# Exécution avec filtrage par tags
Write-Host "`n=== RUNNING HIGH PRIORITY TESTS ONLY ===" -ForegroundColor Yellow
$testFramework.RunTests($null, @("Priority=High"))

# Métriques de test
$metrics = $testFramework.GetTestMetrics()
Write-Host "`n=== FINAL TEST METRICS ===" -ForegroundColor Cyan
Write-Host "Success Rate: $($metrics.SuccessRate)%"
Write-Host "Average Test Time: $($metrics.AverageTestTime)s"
Write-Host "Total Execution Time: $([math]::Round($metrics.TotalExecutionTime, 2))s"

# Export des résultats
$testFramework.ExportResults("TestResults.json")

Write-Host "`nTest framework demonstration completed!" -ForegroundColor Green
```

## Conclusion : L'excellence par la discipline

Les meilleures pratiques de développement PowerShell transforment l'écriture de scripts en une discipline structurée qui produit du code maintenable, testable et évolutif. En appliquant rigoureusement les principes de modularité, les patterns de conception, les standards de codage et les tests unitaires, les développeurs PowerShell créent des solutions qui non seulement fonctionnent mais excellent dans leur conception et leur fiabilité.

Dans le prochain chapitre, nous explorerons PowerShell pour l'administration des systèmes virtualisés et les conteneurs, ouvrant de nouvelles dimensions à l'automatisation infrastructurelle.

---

**Exercice pratique :** Créez une application PowerShell complète qui :
1. Implémente une architecture modulaire avec séparation claire des préoccupations
2. Utilise des patterns de conception avancés (Factory, Observer, Strategy)
3. Respecte strictement les standards de codage PowerShell
4. Inclut une suite complète de tests unitaires avec couverture élevée
5. Fournit des rapports de qualité de code automatiques

**Challenge avancé :** Développez un framework de développement PowerShell qui :
- Implémente une architecture en plugins extensibles
- Fournit des templates de projet avec scaffolding automatique
- Intègre l'analyse statique et les métriques de complexité
- Génère automatiquement la documentation à partir du code
- Supporte le développement piloté par les tests (TDD) de manière native

**Réflexion :** Le développement PowerShell peut-il atteindre le même niveau de rigueur et de qualité que les langages de programmation traditionnels ? Les contraintes du scripting sont-elles compatibles avec les meilleures pratiques du génie logiciel ? Comment adapter les méthodologies agiles au contexte spécifique de l'administration système ?
