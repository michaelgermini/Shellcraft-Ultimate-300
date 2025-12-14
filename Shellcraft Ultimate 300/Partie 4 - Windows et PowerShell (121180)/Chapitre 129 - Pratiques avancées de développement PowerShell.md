# Chapitre 129 - Pratiques avancées de développement PowerShell

> "Développer avec PowerShell n'est pas seulement écrire du code - c'est créer des solutions maintenables, testables, et évolutives qui transcendent les simples scripts." - Expert en développement PowerShell

## Introduction : De l'artisanat à l'ingénierie logicielle

Si les chapitres précédents nous ont familiarisés avec la syntaxe et les fonctionnalités de PowerShell, nous abordons maintenant les pratiques qui transforment l'écriture de scripts en véritable développement logiciel. Tests unitaires, gestion des versions, intégration continue, et architecture modulaire deviennent les piliers d'une approche professionnelle du développement PowerShell.

Dans ce chapitre, nous explorerons les méthodologies et outils qui permettent de créer des solutions PowerShell robustes, maintenables, et évolutives.

## Section 1 : Architecture modulaire et design patterns

### 1.1 Structure d'un module PowerShell

```powershell
# Structure recommandée d'un module
# MyModule/
# ├── MyModule.psm1          # Module principal
# ├── MyModule.psd1          # Manifest du module
# ├── Public/                # Fonctions publiques
# │   ├── Get-Something.ps1
# │   └── Set-Something.ps1
# ├── Private/               # Fonctions privées
# │   ├── Convert-Data.ps1
# │   └── Validate-Input.ps1
# ├── Classes/               # Classes PowerShell
# │   └── MyClass.ps1
# ├── Tests/                 # Tests unitaires
# │   └── MyModule.Tests.ps1
# └── en-US/                 # Ressources de localisation
#     └── MyModule-help.xml

# Manifest du module (MyModule.psd1)
@{
    RootModule = 'MyModule.psm1'
    ModuleVersion = '1.0.0'
    GUID = '12345678-1234-1234-1234-123456789012'
    Author = 'Votre Nom'
    CompanyName = 'Votre Société'
    Copyright = '(c) 2024 Votre Société. Tous droits réservés.'
    Description = 'Module PowerShell pour [description]'
    PowerShellVersion = '5.1'
    FunctionsToExport = @('Get-Something', 'Set-Something', 'New-Something')
    CmdletsToExport = @()
    VariablesToExport = @()
    AliasesToExport = @()
    PrivateData = @{
        PSData = @{
            Tags = @('PowerShell', 'Module', 'Example')
            LicenseUri = 'https://github.com/username/MyModule/blob/master/LICENSE'
            ProjectUri = 'https://github.com/username/MyModule'
            ReleaseNotes = 'Version initiale'
        }
    }
}

# Module principal (MyModule.psm1)
# Import des fonctions privées
Get-ChildItem -Path "$PSScriptRoot\Private" -Filter "*.ps1" | ForEach-Object {
    . $_.FullName
}

# Import des classes
Get-ChildItem -Path "$PSScriptRoot\Classes" -Filter "*.ps1" | ForEach-Object {
    . $_.FullName
}

# Import des fonctions publiques
Get-ChildItem -Path "$PSScriptRoot\Public" -Filter "*.ps1" | ForEach-Object {
    . $_.FullName
}

# Export des fonctions publiques
Export-ModuleMember -Function (Get-ChildItem -Path "$PSScriptRoot\Public" -Filter "*.ps1" | ForEach-Object { $_.BaseName })
```

### 1.2 Design patterns pour PowerShell

```powershell
# Pattern Factory pour la création d'objets
class LoggerFactory {
    static [Logger] CreateLogger([string]$type) {
        switch ($type) {
            "File" { return [FileLogger]::new() }
            "Console" { return [ConsoleLogger]::new() }
            "Database" { return [DatabaseLogger]::new() }
            default { throw "Type de logger non supporté: $type" }
        }
    }
}

# Pattern Singleton pour les ressources partagées
class ConfigurationManager {
    static [ConfigurationManager] $_instance = $null
    hidden [hashtable] $_config = @{}
    
    ConfigurationManager() {
        # Constructeur privé
    }
    
    static [ConfigurationManager] GetInstance() {
        if ($null -eq [ConfigurationManager]::_instance) {
            [ConfigurationManager]::_instance = [ConfigurationManager]::new()
        }
        return [ConfigurationManager]::_instance
    }
    
    [void] LoadConfiguration([string]$path) {
        if (Test-Path $path) {
            $this._config = Get-Content $path | ConvertFrom-Json
        }
    }
    
    [object] GetValue([string]$key) {
        return $this._config[$key]
    }
}

# Pattern Strategy pour les algorithmes interchangeables
class BackupStrategy {
    [void] ExecuteBackup([string]$source, [string]$destination) {
        throw "Méthode abstraite - doit être implémentée"
    }
}

class FullBackupStrategy : BackupStrategy {
    [void] ExecuteBackup([string]$source, [string]$destination) {
        Write-Host "Exécution d'une sauvegarde complète"
        Copy-Item -Path $source -Destination $destination -Recurse -Force
    }
}

class IncrementalBackupStrategy : BackupStrategy {
    [void] ExecuteBackup([string]$source, [string]$destination) {
        Write-Host "Exécution d'une sauvegarde incrémentielle"
        # Logique de sauvegarde incrémentielle
        $lastBackup = Get-Item "$destination\last_backup.txt" -ErrorAction SilentlyContinue
        if ($lastBackup) {
            $changedFiles = Get-ChildItem -Path $source -Recurse | Where-Object { $_.LastWriteTime -gt $lastBackup.LastWriteTime }
            $changedFiles | Copy-Item -Destination $destination
        } else {
            Copy-Item -Path $source -Destination $destination -Recurse -Force
        }
    }
}

class BackupManager {
    [BackupStrategy] $Strategy
    
    BackupManager([BackupStrategy]$strategy) {
        $this.Strategy = $strategy
    }
    
    [void] PerformBackup([string]$source, [string]$destination) {
        $this.Strategy.ExecuteBackup($source, $destination)
    }
}

# Utilisation des patterns
$config = [ConfigurationManager]::GetInstance()
$config.LoadConfiguration("C:\Config\app.json")

$logger = [LoggerFactory]::CreateLogger("File")

$backupManager = [BackupManager]::new([FullBackupStrategy]::new())
$backupManager.PerformBackup("C:\Data", "D:\Backup")
```

## Section 2 : Tests et qualité du code

### 2.1 Framework de test Pester

```powershell
# Installation de Pester
Install-Module -Name Pester -Force -SkipPublisherCheck

# Structure d'un fichier de test
# Tests pour le module MyModule
Describe "MyModule Tests" {
    BeforeAll {
        # Configuration initiale
        Import-Module "$PSScriptRoot\..\MyModule.psm1" -Force
        $testData = @{
            ValidInput = "test"
            InvalidInput = $null
        }
    }
    
    AfterAll {
        # Nettoyage
        Remove-Module MyModule -ErrorAction SilentlyContinue
    }
    
    Context "Get-Something Function" {
        It "Returns something when given valid input" {
            $result = Get-Something -InputData $testData.ValidInput
            $result | Should -Not -BeNullOrEmpty
            $result | Should -BeOfType [string]
        }
        
        It "Throws exception for invalid input" {
            { Get-Something -InputData $testData.InvalidInput } | Should -Throw
        }
        
        It "Processes multiple items correctly" {
            $items = @("item1", "item2", "item3")
            $results = Get-Something -InputData $items
            $results.Count | Should -Be 3
        }
    }
    
    Context "Set-Something Function" {
        BeforeEach {
            # Configuration pour chaque test
            $tempFile = New-TemporaryFile
        }
        
        AfterEach {
            # Nettoyage après chaque test
            Remove-Item $tempFile -ErrorAction SilentlyContinue
        }
        
        It "Creates output file when processing data" {
            Set-Something -InputData "test data" -OutputPath $tempFile.FullName
            $tempFile | Should -Exist
            Get-Content $tempFile.FullName | Should -Contain "test data"
        }
        
        It "Validates input parameters" {
            { Set-Something -InputData "" -OutputPath "invalid:path" } | Should -Throw
        }
    }
    
    Context "Error Handling" {
        It "Handles network timeouts gracefully" {
            Mock Invoke-WebRequest { throw "Timeout" }
            
            { Get-Something -UseNetwork $true } | Should -Throw "Network timeout"
        }
        
        It "Retries failed operations" {
            $callCount = 0
            Mock Invoke-WebRequest { 
                $callCount++
                if ($callCount -lt 3) { throw "Temporary failure" }
                return @{ StatusCode = 200 }
            }
            
            { Get-Something -RetryOnFailure $true } | Should -Not -Throw
            $callCount | Should -Be 3
        }
    }
}

# Tests d'intégration
Describe "Integration Tests" {
    It "End-to-end workflow works correctly" {
        # Test du workflow complet
        $inputData = Get-Something -Source "database"
        $processedData = Set-Something -InputData $inputData -Transform $true
        $result = Send-Something -Data $processedData -Destination "api"
        
        $result.Success | Should -Be $true
        $result.StatusCode | Should -Be 200
    }
}

# Exécution des tests
Invoke-Pester -Path ".\Tests" -OutputFormat NUnitXml -OutputFile "TestResults.xml"
```

### 2.2 Mocks et stubs pour les tests

```powershell
# Tests avec mocks pour les dépendances externes
Describe "External Dependencies Tests" {
    BeforeAll {
        Import-Module "$PSScriptRoot\..\MyModule.psm1" -Force
    }
    
    Context "Database Operations" {
        It "Handles database connection failures" {
            Mock Connect-Database { throw "Connection failed" }
            
            { Get-DataFromDatabase -Server "unreachable" } | Should -Throw "Connection failed"
        }
        
        It "Returns cached data when database is unavailable" {
            Mock Connect-Database { throw "Connection failed" }
            Mock Get-CachedData { return @("cached", "data") }
            
            $result = Get-DataFromDatabase -UseCache $true
            $result | Should -Be @("cached", "data")
        }
        
        It "Verifies database calls are made correctly" {
            Mock Invoke-SqlQuery { return @("result1", "result2") } -ParameterFilter {
                $Query -like "*SELECT*" -and $Server -eq "production"
            }
            
            Get-DataFromDatabase -Server "production"
            
            Assert-MockCalled Invoke-SqlQuery -Exactly 1 -ParameterFilter {
                $Server -eq "production"
            }
        }
    }
    
    Context "API Calls" {
        It "Retries on HTTP 500 errors" {
            Mock Invoke-RestMethod {
                param($Uri, $Method)
                if ($Method -eq "GET" -and $Uri -eq "https://api.example.com/data") {
                    # Simuler une erreur 500 puis succès
                    $script:callCount++
                    if ($script:callCount -lt 3) {
                        throw [System.Net.WebException]::new("Internal Server Error", $null, [System.Net.WebExceptionStatus]::ProtocolError, $null)
                    }
                    return @{ status = "success"; data = @("item1", "item2") }
                }
            }
            
            $script:callCount = 0
            
            $result = Get-DataFromAPI -Uri "https://api.example.com/data" -RetryCount 3
            
            $result.status | Should -Be "success"
            $script:callCount | Should -Be 3
        }
    }
}

# Tests de performance
Describe "Performance Tests" {
    It "Processes 1000 items within time limit" {
        $items = 1..1000
        
        $time = Measure-Command {
            $result = Process-LargeDataset -Items $items
        }
        
        $time.TotalSeconds | Should -BeLessThan 5
        $result.Count | Should -Be 1000
    }
    
    It "Memory usage stays within limits" {
        $initialMemory = [GC]::GetTotalMemory($false)
        
        Process-LargeDataset -Items (1..10000)
        
        [GC]::Collect()
        $finalMemory = [GC]::GetTotalMemory($true)
        $memoryUsed = $finalMemory - $initialMemory
        
        $memoryUsed | Should -BeLessThan 50MB
    }
}
```

### 2.3 Analyse statique du code

```powershell
# Installation de PSScriptAnalyzer
Install-Module -Name PSScriptAnalyzer -Force

# Analyse de la qualité du code
$analysis = Invoke-ScriptAnalyzer -Path "C:\Scripts\MyScript.ps1"

# Afficher les problèmes
$analysis | Where-Object Severity -eq "Error" | Format-Table -AutoSize
$analysis | Where-Object Severity -eq "Warning" | Format-Table -AutoSize

# Règles personnalisées
$customRules = @'
using System;
using System.Management.Automation;
using Microsoft.Windows.PowerShell.ScriptAnalyzer.Generic;

[Cmdlet("Test", "CustomRule")]
public class AvoidWriteHostRule : IRule
{
    public string RuleName => "AvoidWriteHost";
    public string Description => "Évite l'utilisation de Write-Host";
    public string Severity => "Warning";
    
    public IEnumerable<DiagnosticRecord> AnalyzeScript(Ast ast, string fileName)
    {
        var diagnostics = new List<DiagnosticRecord>();
        
        // Recherche des appels Write-Host
        var writeHostCalls = ast.FindAll(x => 
            x is CommandAst cmd && 
            cmd.GetCommandName() == "Write-Host", 
            true);
            
        foreach (var call in writeHostCalls)
        {
            diagnostics.Add(new DiagnosticRecord(
                "Utilisation de Write-Host détectée. Préférez Write-Output.",
                call.Extent,
                fileName,
                RuleName,
                Severity));
        }
        
        return diagnostics;
    }
}
'@

# Compiler et utiliser la règle personnalisée
Add-Type -TypeDefinition $customRules -ReferencedAssemblies "Microsoft.Windows.PowerShell.ScriptAnalyzer.dll"

# Analyse avec les règles personnalisées
$rules = Get-ScriptAnalyzerRule
$rules += [AvoidWriteHostRule]::new()

Invoke-ScriptAnalyzer -Path "C:\Scripts" -CustomRulePath $rules
```

## Section 3 : Gestion des versions et distribution

### 3.1 Versioning sémantique

```powershell
# Classe pour gérer le versioning
class SemanticVersion {
    [int]$Major
    [int]$Minor
    [int]$Patch
    [string]$PreRelease
    [string]$BuildMetadata
    
    SemanticVersion([string]$versionString) {
        # Parsing de la chaîne de version
        $regex = '^(\d+)\.(\d+)\.(\d+)(?:-([0-9A-Za-z-]+(?:\.[0-9A-Za-z-]+)*))?(?:\+([0-9A-Za-z-]+(?:\.[0-9A-Za-z-]+)*))?$'
        $match = [regex]::Match($versionString, $regex)
        
        if ($match.Success) {
            $this.Major = [int]$match.Groups[1].Value
            $this.Minor = [int]$match.Groups[2].Value
            $this.Patch = [int]$match.Groups[3].Value
            $this.PreRelease = $match.Groups[4].Value
            $this.BuildMetadata = $match.Groups[5].Value
        } else {
            throw "Format de version invalide: $versionString"
        }
    }
    
    [string] ToString() {
        $version = "$($this.Major).$($this.Minor).$($this.Patch)"
        if ($this.PreRelease) { $version += "-$($this.PreRelease)" }
        if ($this.BuildMetadata) { $version += "+$($this.BuildMetadata)" }
        return $version
    }
    
    [int] CompareTo([SemanticVersion]$other) {
        # Comparaison selon les règles semver
        if ($this.Major -ne $other.Major) { return $this.Major - $other.Major }
        if ($this.Minor -ne $other.Minor) { return $this.Minor - $other.Minor }
        if ($this.Patch -ne $other.Patch) { return $this.Patch - $other.Patch }
        
        # Gestion des versions pré-release
        if ($this.PreRelease -and -not $other.PreRelease) { return -1 }
        if (-not $this.PreRelease -and $other.PreRelease) { return 1 }
        if ($this.PreRelease -and $other.PreRelease) {
            return [string]::Compare($this.PreRelease, $other.PreRelease)
        }
        
        return 0
    }
    
    static [SemanticVersion] IncrementMajor([SemanticVersion]$version) {
        return [SemanticVersion]::new("$($version.Major + 1).0.0")
    }
    
    static [SemanticVersion] IncrementMinor([SemanticVersion]$version) {
        return [SemanticVersion]::new("$($version.Major).$($version.Minor + 1).0")
    }
    
    static [SemanticVersion] IncrementPatch([SemanticVersion]$version) {
        return [SemanticVersion]::new("$($version.Major).$($version.Minor).$($version.Patch + 1)")
    }
}

# Utilisation du versioning sémantique
$currentVersion = [SemanticVersion]::new("1.2.3")
$newVersion = [SemanticVersion]::IncrementMinor($currentVersion)
Write-Host "Nouvelle version: $($newVersion.ToString())"
```

### 3.2 Build et déploiement automatisés

```powershell
# Script de build pour un module PowerShell
param(
    [string]$Version = "1.0.0",
    [string]$OutputPath = ".\Build",
    [switch]$PublishToGallery
)

# Configuration du build
$moduleName = "MyModule"
$sourcePath = ".\Source"
$buildConfig = @{
    ModuleName = $moduleName
    Version = $Version
    Author = "Votre Nom"
    Description = "Description du module"
    FunctionsToExport = @()
    Dependencies = @("Pester", "PSScriptAnalyzer")
}

# Fonction de build
function Invoke-Build {
    param([hashtable]$Config)
    
    Write-Host "=== BUILD DU MODULE $($Config.ModuleName) ==="
    
    # Créer le répertoire de build
    $buildPath = Join-Path $OutputPath $Config.Version
    New-Item -ItemType Directory -Path $buildPath -Force | Out-Null
    
    # Copier les fichiers source
    Copy-Item -Path "$sourcePath\*" -Destination $buildPath -Recurse -Force
    
    # Générer le manifest
    $manifestPath = Join-Path $buildPath "$($Config.ModuleName).psd1"
    $manifestContent = @{
        RootModule = "$($Config.ModuleName).psm1"
        ModuleVersion = $Config.Version
        GUID = [guid]::NewGuid().ToString()
        Author = $Config.Author
        Description = $Config.Description
        PowerShellVersion = "5.1"
        FunctionsToExport = $Config.FunctionsToExport
        RequiredModules = $Config.Dependencies
    }
    
    New-ModuleManifest @manifestContent
    
    # Compiler le module
    $moduleContent = Get-ChildItem -Path $sourcePath -Filter "*.ps1" -Recurse | 
        Where-Object { $_.Name -notlike "*.Tests.ps1" } |
        Get-Content
    
    $modulePath = Join-Path $buildPath "$($Config.ModuleName).psm1"
    $moduleContent | Out-File -FilePath $modulePath -Encoding UTF8
    
    Write-Host "✓ Module compilé: $buildPath"
    return $buildPath
}

# Fonction de test
function Invoke-ModuleTests {
    param([string]$ModulePath)
    
    Write-Host "=== TESTS DU MODULE ==="
    
    # Importer Pester
    Import-Module Pester
    
    # Exécuter les tests
    $testResults = Invoke-Pester -Path "$ModulePath\..\Tests" -PassThru -OutputFormat NUnitXml -OutputFile "TestResults.xml"
    
    if ($testResults.FailedCount -gt 0) {
        throw "Échec des tests: $($testResults.FailedCount) tests ont échoué"
    }
    
    Write-Host "✓ Tests réussis: $($testResults.PassedCount) tests passés"
}

# Fonction d'analyse de code
function Invoke-CodeAnalysis {
    param([string]$ModulePath)
    
    Write-Host "=== ANALYSE DU CODE ==="
    
    Import-Module PSScriptAnalyzer
    
    $analysisResults = Invoke-ScriptAnalyzer -Path $ModulePath
    
    $errors = $analysisResults | Where-Object Severity -eq "Error"
    $warnings = $analysisResults | Where-Object Severity -eq "Warning"
    
    if ($errors.Count -gt 0) {
        Write-Error "Erreurs d'analyse détectées: $($errors.Count)"
        $errors | Format-Table -AutoSize
        throw "Échec de l'analyse du code"
    }
    
    if ($warnings.Count -gt 0) {
        Write-Warning "Avertissements d'analyse: $($warnings.Count)"
        $warnings | Format-Table -AutoSize
    }
    
    Write-Host "✓ Analyse du code réussie"
}

# Fonction de publication
function Publish-ModuleToGallery {
    param([string]$ModulePath, [string]$ApiKey)
    
    Write-Host "=== PUBLICATION SUR POWERSHELL GALLERY ==="
    
    # Tester la connexion
    Test-PSGalleryConnection
    
    # Publier le module
    Publish-Module -Path $ModulePath -NuGetApiKey $ApiKey
    
    Write-Host "✓ Module publié sur PowerShell Gallery"
}

# Pipeline de build principal
try {
    # Build
    $modulePath = Invoke-Build -Config $buildConfig
    
    # Tests
    Invoke-ModuleTests -ModulePath $modulePath
    
    # Analyse
    Invoke-CodeAnalysis -ModulePath $modulePath
    
    # Package
    Compress-Archive -Path $modulePath -DestinationPath "$OutputPath\$moduleName-$Version.zip"
    
    # Publication optionnelle
    if ($PublishToGallery) {
        $apiKey = Read-Host "Entrez votre clé API PowerShell Gallery" -AsSecureString
        $apiKeyPlain = [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR($apiKey))
        Publish-ModuleToGallery -ModulePath $modulePath -ApiKey $apiKeyPlain
    }
    
    Write-Host "=== BUILD RÉUSSI ==="
    Write-Host "Module disponible: $modulePath"
    Write-Host "Archive: $OutputPath\$moduleName-$Version.zip"
    
} catch {
    Write-Error "Échec du build: $($_.Exception.Message)"
    exit 1
}
```

### 3.3 Intégration continue avec GitHub Actions

```yaml
# .github/workflows/powershell-ci.yml
name: PowerShell CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: windows-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Setup PowerShell
      uses: actions/setup-powershell@v1
      with:
        pwsh: true
        
    - name: Install dependencies
      shell: pwsh
      run: |
        Install-Module -Name Pester -Force -SkipPublisherCheck
        Install-Module -Name PSScriptAnalyzer -Force -SkipPublisherCheck
        
    - name: Run tests
      shell: pwsh
      run: |
        $results = Invoke-Pester -Path ".\Tests" -PassThru -OutputFormat NUnitXml -OutputFile "TestResults.xml"
        if ($results.FailedCount -gt 0) {
            throw "Tests failed: $($results.FailedCount)"
        }
        
    - name: Run code analysis
      shell: pwsh
      run: |
        $analysis = Invoke-ScriptAnalyzer -Path ".\Source"
        $errors = $analysis | Where-Object Severity -eq "Error"
        if ($errors.Count -gt 0) {
            $errors | Format-Table -AutoSize
            throw "Code analysis failed: $($errors.Count) errors"
        }
        
    - name: Upload test results
      uses: actions/upload-artifact@v3
      with:
        name: test-results
        path: TestResults.xml

  publish:
    needs: test
    runs-on: windows-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Setup PowerShell
      uses: actions/setup-powershell@v1
      with:
        pwsh: true
        
    - name: Build module
      shell: pwsh
      run: |
        .\Build.ps1 -Version "1.0.${{ github.run_number }}"
        
    - name: Publish to PowerShell Gallery
      shell: pwsh
      env:
        NUGET_API_KEY: ${{ secrets.POWERSHELL_GALLERY_API_KEY }}
      run: |
        Publish-Module -Path ".\Build\1.0.${{ github.run_number }}" -NuGetApiKey $env:NUGET_API_KEY
```

## Section 4 : Documentation et maintenance

### 4.1 Génération automatique de documentation

```powershell
# Génération de documentation avec platyPS
Install-Module -Name platyPS -Force

# Créer la documentation à partir du code
New-MarkdownHelp -Module MyModule -OutputFolder ".\docs" -WithModulePage -Locale "fr-FR"

# Générer l'aide externe (cab)
New-ExternalHelp -Path ".\docs" -OutputPath ".\en-US" -Force

# Documentation avec des exemples
function Get-User {
    <#
    .SYNOPSIS
        Récupère les informations d'un utilisateur
    
    .DESCRIPTION
        Cette fonction récupère les informations détaillées d'un utilisateur
        dans Active Directory ou un autre système d'identité.
    
    .PARAMETER UserName
        Le nom d'utilisateur à rechercher
    
    .PARAMETER Domain
        Le domaine dans lequel rechercher l'utilisateur
    
    .EXAMPLE
        Get-User -UserName "john.doe"
        
        Récupère les informations de l'utilisateur john.doe dans le domaine actuel
    
    .EXAMPLE
        Get-User -UserName "admin" -Domain "corp.contoso.com"
        
        Récupère les informations de l'administrateur dans le domaine spécifié
    
    .OUTPUTS
        System.Management.Automation.PSCustomObject
        Objet contenant les informations de l'utilisateur
    
    .NOTES
        Version: 1.0
        Auteur: Votre Nom
    #>
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [string]$UserName,
        
        [string]$Domain = $env:USERDOMAIN
    )
    
    process {
        # Logique de récupération de l'utilisateur
        Write-Verbose "Recherche de l'utilisateur $UserName dans $Domain"
        
        # Simulation
        [PSCustomObject]@{
            UserName = $UserName
            Domain = $Domain
            DisplayName = "John Doe"
            Email = "john.doe@$Domain.com"
            Department = "IT"
            Enabled = $true
        }
    }
}

# Mise à jour de la documentation
Update-MarkdownHelp -Path ".\docs\Get-User.md"
```

### 4.2 Métriques et monitoring du code

```powershell
# Analyse des métriques du code
function Get-CodeMetrics {
    param([string]$Path)
    
    $files = Get-ChildItem -Path $Path -Filter "*.ps1" -Recurse
    
    $metrics = foreach ($file in $files) {
        $content = Get-Content $file.FullName -Raw
        $lines = $content -split "`n"
        
        # Compter les lignes de code (non vides, non commentaires)
        $codeLines = $lines | Where-Object { 
            $_.Trim() -and -not $_.Trim().StartsWith("#") 
        } | Measure-Object | Select-Object -ExpandProperty Count
        
        # Compter les fonctions
        $functions = [regex]::Matches($content, 'function\s+\w+').Count
        
        # Compter les classes
        $classes = [regex]::Matches($content, 'class\s+\w+').Count
        
        [PSCustomObject]@{
            FileName = $file.Name
            Path = $file.FullName
            TotalLines = $lines.Count
            CodeLines = $codeLines
            CommentLines = $lines.Count - $codeLines
            Functions = $functions
            Classes = $classes
            Complexity = $codeLines + ($functions * 10) + ($classes * 20)
        }
    }
    
    return $metrics
}

# Rapport de métriques
function New-CodeMetricsReport {
    param([string]$SourcePath, [string]$OutputPath)
    
    $metrics = Get-CodeMetrics -Path $SourcePath
    
    $summary = [PSCustomObject]@{
        TotalFiles = $metrics.Count
        TotalLines = ($metrics | Measure-Object -Property TotalLines -Sum).Sum
        TotalCodeLines = ($metrics | Measure-Object -Property CodeLines -Sum).Sum
        TotalFunctions = ($metrics | Measure-Object -Property Functions -Sum).Sum
        TotalClasses = ($metrics | Measure-Object -Property Classes -Sum).Sum
        AverageComplexity = [math]::Round(($metrics | Measure-Object -Property Complexity -Average).Average, 2)
    }
    
    # Générer le rapport HTML
    $html = @"
<!DOCTYPE html>
<html>
<head>
    <title>PowerShell Code Metrics Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .summary { background-color: #e7f3ff; padding: 10px; margin-bottom: 20px; }
    </style>
</head>
<body>
    <h1>PowerShell Code Metrics Report</h1>
    <div class="summary">
        <h2>Summary</h2>
        <p>Total Files: $($summary.TotalFiles)</p>
        <p>Total Lines: $($summary.TotalLines)</p>
        <p>Code Lines: $($summary.TotalCodeLines)</p>
        <p>Functions: $($summary.TotalFunctions)</p>
        <p>Classes: $($summary.TotalClasses)</p>
        <p>Average Complexity: $($summary.AverageComplexity)</p>
    </div>
    
    <h2>File Details</h2>
    <table>
        <tr>
            <th>File Name</th>
            <th>Total Lines</th>
            <th>Code Lines</th>
            <th>Functions</th>
            <th>Classes</th>
            <th>Complexity</th>
        </tr>
        $($metrics | ForEach-Object {
            "<tr><td>$($_.FileName)</td><td>$($_.TotalLines)</td><td>$($_.CodeLines)</td><td>$($_.Functions)</td><td>$($_.Classes)</td><td>$($_.Complexity)</td></tr>"
        } -join "`n")
    </table>
</body>
</html>
"@
    
    $html | Out-File -FilePath $OutputPath -Encoding UTF8
    Write-Host "Rapport généré: $OutputPath"
}

# Générer le rapport
New-CodeMetricsReport -SourcePath ".\Source" -OutputPath ".\docs\CodeMetrics.html"
```

## Conclusion : L'excellence dans le développement PowerShell

Les pratiques avancées de développement transforment PowerShell d'un langage de scripting en un véritable environnement de développement professionnel. Tests rigoureux, architecture modulaire, intégration continue, et documentation complète deviennent les garants de solutions maintenables et évolutives.

Dans le prochain chapitre, nous conclurons notre exploration de PowerShell avec une réflexion sur l'avenir du shell scripting et les tendances émergentes qui façonneront les prochaines années.

---

**Exercice pratique :** Créez un module PowerShell complet avec :
1. Architecture modulaire (public/privé)
2. Tests unitaires complets avec Pester
3. Analyse statique du code
4. Documentation automatique
5. Pipeline CI/CD
6. Métriques de code

**Challenge avancé :** Développez un framework de test personnalisé pour PowerShell qui :
- Supporte les tests d'intégration
- Fournit des mocks avancés
- Génère des rapports de couverture
- S'intègre avec les outils CI/CD
- Supporte le testing parallèle

**Réflexion :** Comment les pratiques de développement moderne s'appliquent-elles à PowerShell, et quels sont les défis spécifiques à ce langage dans un contexte de développement professionnel ?

