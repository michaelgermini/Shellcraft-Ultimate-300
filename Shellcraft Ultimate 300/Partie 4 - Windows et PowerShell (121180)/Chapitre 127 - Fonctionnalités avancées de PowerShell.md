# Chapitre 127 - Fonctionnalités avancées de PowerShell

> "PowerShell n'est pas seulement un shell - c'est un environnement d'exécution complet où la programmation asynchrone, les workflows, et la configuration déclarative coexistent harmonieusement." - Architecte PowerShell

## Introduction : Au-delà du scripting de base

Si les chapitres précédents nous ont familiarisés avec la syntaxe et les cmdlets de base de PowerShell, nous abordons maintenant ses fonctionnalités avancées qui en font un outil de niveau entreprise. Jobs en arrière-plan, workflows, Desired State Configuration (DSC), et bien d'autres caractéristiques transforment PowerShell d'un simple automate en une plateforme complète d'orchestration.

Dans ce chapitre, nous explorerons ces fonctionnalités avancées qui permettent de construire des solutions robustes, évolutives et maintenables.

## Section 1 : Jobs et exécution asynchrone

### 1.1 Concepts des jobs PowerShell

Les jobs permettent d'exécuter des commandes en arrière-plan, libérant la console pour d'autres tâches :

```powershell
# Création d'un job simple
$job = Start-Job -ScriptBlock { 
    Get-Process | Where-Object CPU -gt 10 
}

# État du job
Get-Job

# Récupération des résultats
Receive-Job -Job $job -Wait

# Nettoyage
Remove-Job -Job $job
```

### 1.2 Gestion avancée des jobs

```powershell
# Jobs nommés
Start-Job -Name "SystemMonitor" -ScriptBlock {
    while ($true) {
        Get-Counter '\Processor(_Total)\% Processor Time' | Export-Csv -Path "C:\Logs\cpu_$((Get-Date).ToString('yyyyMMdd_HHmmss')).csv" -NoTypeInformation
        Start-Sleep -Seconds 60
    }
}

# Jobs avec paramètres
$jobParams = @{
    Name = "BackupJob"
    ScriptBlock = {
        param($SourcePath, $DestinationPath)
        Copy-Item -Path $SourcePath -Destination $DestinationPath -Recurse
    }
    ArgumentList = @("C:\Data", "D:\Backup")
}

Start-Job @jobParams

# Surveillance des jobs
$runningJobs = Get-Job | Where-Object State -eq 'Running'
Write-Host "Jobs en cours: $($runningJobs.Count)"

# Gestion des jobs en échec
$failedJobs = Get-Job | Where-Object State -eq 'Failed'
foreach ($job in $failedJobs) {
    Write-Warning "Job échoué: $($job.Name)"
    Write-Warning "Erreur: $(Receive-Job -Job $job 2>&1)"
}

# Nettoyage automatique des jobs terminés
Get-Job | Where-Object State -in @('Completed', 'Failed') | Remove-Job
```

### 1.3 Jobs planifiés et tâches

```powershell
# Création d'une tâche planifiée
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-File C:\Scripts\DailyReport.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "06:00"
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries

Register-ScheduledTask -TaskName "DailySystemReport" -Action $action -Trigger $trigger -Settings $settings -User "SYSTEM" -RunLevel Highest

# Tâches planifiées avec conditions
$trigger = New-ScheduledTaskTrigger -AtLogOn
$principal = New-ScheduledTaskPrincipal -UserId "DOMAIN\User" -LogonType InteractiveToken
Register-ScheduledTask -TaskName "UserLogonScript" -Action $action -Trigger $trigger -Principal $principal

# Gestion des tâches planifiées
Get-ScheduledTask | Where-Object State -eq 'Ready' | Select-Object TaskName, State, LastRunTime, NextRunTime

# Exécution manuelle
Start-ScheduledTask -TaskName "DailySystemReport"

# Modification
Set-ScheduledTask -TaskName "DailySystemReport" -Trigger $newTrigger

# Désactivation
Disable-ScheduledTask -TaskName "DailySystemReport"
```

### 1.4 Threading et parallélisation

```powershell
# Exécution parallèle avec ForEach-Object -Parallel (PowerShell 7+)
1..10 | ForEach-Object -Parallel {
    $result = Invoke-WebRequest -Uri "https://httpbin.org/delay/1" -TimeoutSec 10
    Write-Host "Requête $_ terminée: $($result.StatusCode)"
} -ThrottleLimit 3

# Runspaces pour parallélisation avancée
$runspacePool = [runspacefactory]::CreateRunspacePool(1, 5)
$runspacePool.Open()

$runspaces = @()

# Création des runspaces
1..10 | ForEach-Object {
    $runspace = [powershell]::Create()
    $runspace.RunspacePool = $runspacePool
    
    [void]$runspace.AddScript({
        param($id)
        Start-Sleep -Seconds (Get-Random -Minimum 1 -Maximum 5)
        return "Tâche $id terminée à $(Get-Date)"
    }).AddArgument($_)
    
    $runspaces += [PSCustomObject]@{
        Runspace = $runspace
        Handle = $runspace.BeginInvoke()
        Id = $_
    }
}

# Collecte des résultats
$results = @()
foreach ($rs in $runspaces) {
    $result = $rs.Runspace.EndInvoke($rs.Handle)
    $results += $result
    $rs.Runspace.Dispose()
}

$runspacePool.Close()
$runspacePool.Dispose()

Write-Host "Résultats:"
$results | ForEach-Object { Write-Host $_ }
```

## Section 2 : Workflows PowerShell

### 2.1 Concepts des workflows

Les workflows offrent une exécution fiable et interruptible pour les tâches longues :

```powershell
# Définition d'un workflow simple
workflow Test-Workflow {
    Write-Output "Début du workflow"
    Start-Sleep -Seconds 2
    Write-Output "Milieu du workflow"
    Start-Sleep -Seconds 2
    Write-Output "Fin du workflow"
}

# Exécution
Test-Workflow

# Workflow avec paramètres
workflow Process-Data {
    param([string[]]$Data)
    
    foreach -parallel ($item in $Data) {
        InlineScript {
            Write-Output "Traitement de $using:item"
            Start-Sleep -Seconds 1
        }
    }
}

# Exécution parallèle
Process-Data -Data @("Item1", "Item2", "Item3", "Item4")
```

### 2.2 Workflows avancés et gestion d'erreurs

```powershell
# Workflow avec gestion d'erreurs
workflow Robust-Workflow {
    param([string]$ComputerName)
    
    try {
        # Étape 1: Test de connectivité
        $reachable = InlineScript {
            Test-Connection -ComputerName $using:ComputerName -Count 1 -Quiet
        }
        
        if (-not $reachable) {
            throw "Ordinateur $ComputerName non accessible"
        }
        
        # Étape 2: Collecte d'informations
        $systemInfo = InlineScript {
            Get-WmiObject -Class Win32_OperatingSystem -ComputerName $using:ComputerName
        }
        
        # Étape 3: Traitement des données
        $processedData = InlineScript {
            [PSCustomObject]@{
                ComputerName = $using:ComputerName
                OSVersion = $using:systemInfo.Caption
                LastBootTime = $using:systemInfo.LastBootUpTime
                Timestamp = Get-Date
            }
        }
        
        return $processedData
        
    } catch {
        Write-Error "Erreur dans le workflow: $($_.Exception.Message)"
        return $null
    }
}

# Exécution avec gestion d'erreurs
$result = Robust-Workflow -ComputerName "RemoteServer"
if ($result) {
    Write-Host "Workflow réussi pour $($result.ComputerName)"
} else {
    Write-Host "Workflow échoué"
}
```

### 2.3 Workflows pour l'administration système

```powershell
# Workflow de déploiement d'application
workflow Deploy-Application {
    param(
        [string[]]$ComputerNames,
        [string]$ApplicationPath,
        [string]$InstallArgs
    )
    
    foreach -parallel ($computer in $ComputerNames) {
        InlineScript {
            Write-Output "Déploiement vers $using:computer"
            
            # Copie des fichiers
            Copy-Item -Path $using:ApplicationPath -Destination "\\$using:computer\C$\Temp\" -Force
            
            # Installation
            $installProcess = Start-Process -FilePath "msiexec.exe" `
                -ArgumentList "/i `"C:\Temp\$($using:ApplicationPath | Split-Path -Leaf)`" $using:InstallArgs /quiet /norestart" `
                -Wait -PassThru
            
            if ($installProcess.ExitCode -eq 0) {
                Write-Output "Installation réussie sur $using:computer"
            } else {
                Write-Error "Échec d'installation sur $using:computer (Code: $($installProcess.ExitCode))"
            }
        }
    }
}

# Utilisation
Deploy-Application -ComputerNames @("Server1", "Server2", "Server3") `
    -ApplicationPath "C:\Installers\MyApp.msi" `
    -InstallArgs "INSTALLDIR=`"C:\Program Files\MyApp`""
```

### 2.4 Persistance et reprise des workflows

```powershell
# Workflow avec points de contrôle
workflow Long-Running-Workflow {
    param([int]$TotalItems)
    
    $processed = 0
    
    foreach ($i in 1..$TotalItems) {
        # Traitement d'un élément
        InlineScript {
            Write-Output "Traitement de l'élément $using:i"
            Start-Sleep -Milliseconds 500
        }
        
        $processed = $i
        
        # Point de contrôle tous les 10 éléments
        if ($i % 10 -eq 0) {
            Checkpoint-Workflow
            Write-Output "Point de contrôle atteint: $i éléments traités"
        }
    }
    
    return $processed
}

# Exécution avec possibilité de reprise
$job = Long-Running-Workflow -TotalItems 100 -AsJob

# Simulation d'interruption
Start-Sleep -Seconds 5
Suspend-Job -Job $job

# Reprise plus tard
Resume-Job -Job $job
Receive-Job -Job $job -Wait
```

## Section 3 : Desired State Configuration (DSC)

### 3.1 Concepts de DSC

DSC permet de définir et maintenir l'état souhaité des systèmes de manière déclarative :

```powershell
# Configuration DSC simple
Configuration WebServerConfig {
    param(
        [string[]]$ComputerName = 'localhost'
    )
    
    Import-DscResource -ModuleName PSDesiredStateConfiguration
    
    Node $ComputerName {
        # Installation d'IIS
        WindowsFeature IIS {
            Ensure = 'Present'
            Name = 'Web-Server'
        }
        
        # Création d'un répertoire
        File WebsiteContent {
            Ensure = 'Present'
            Type = 'Directory'
            DestinationPath = 'C:\inetpub\wwwroot\MySite'
        }
        
        # Service IIS démarré
        Service IISService {
            Name = 'W3SVC'
            StartupType = 'Automatic'
            State = 'Running'
            DependsOn = '[WindowsFeature]IIS'
        }
    }
}

# Compilation de la configuration
WebServerConfig -ComputerName 'WebServer01'

# Application de la configuration
Start-DscConfiguration -Path .\WebServerConfig -Wait -Verbose
```

### 3.2 Resources DSC personnalisées

```powershell
# Création d'une resource DSC personnalisée
# Fichier MyDscResources.psm1
function Get-TargetResource {
    param(
        [Parameter(Mandatory)]
        [string]$Name,
        
        [Parameter(Mandatory)]
        [string]$Path
    )
    
    $file = Get-Item -Path $Path -ErrorAction SilentlyContinue
    
    return @{
        Name = $Name
        Path = $Path
        Ensure = if ($file) { 'Present' } else { 'Absent' }
        Size = if ($file) { $file.Length } else { 0 }
    }
}

function Set-TargetResource {
    param(
        [Parameter(Mandatory)]
        [string]$Name,
        
        [Parameter(Mandatory)]
        [string]$Path,
        
        [ValidateSet('Present', 'Absent')]
        [string]$Ensure = 'Present',
        
        [string]$Content = ''
    )
    
    if ($Ensure -eq 'Present') {
        if (-not (Test-Path $Path)) {
            New-Item -Path $Path -ItemType File -Force
        }
        Set-Content -Path $Path -Value $Content
    } else {
        if (Test-Path $Path) {
            Remove-Item -Path $Path -Force
        }
    }
}

function Test-TargetResource {
    param(
        [Parameter(Mandatory)]
        [string]$Name,
        
        [Parameter(Mandatory)]
        [string]$Path,
        
        [ValidateSet('Present', 'Absent')]
        [string]$Ensure = 'Present',
        
        [string]$Content = ''
    )
    
    $currentState = Get-TargetResource -Name $Name -Path $Path
    
    if ($Ensure -eq 'Present') {
        return (Test-Path $Path) -and ($currentState.Size -gt 0)
    } else {
        return -not (Test-Path $Path)
    }
}

# Export de la fonction
Export-ModuleMember -Function *-TargetResource
```

### 3.3 Configurations DSC avancées

```powershell
# Configuration avec dépendances et conditions
Configuration AdvancedServerConfig {
    param(
        [string[]]$ComputerName = 'localhost',
        [PSCredential]$DomainCredential
    )
    
    Import-DscResource -ModuleName PSDesiredStateConfiguration, xActiveDirectory, xNetworking
    
    Node $ComputerName {
        # Variables locales
        $domainName = 'contoso.com'
        
        # Installation des features
        WindowsFeature ADDS {
            Ensure = 'Present'
            Name = 'AD-Domain-Services'
        }
        
        WindowsFeature RSATAD {
            Ensure = 'Present'
            Name = 'RSAT-AD-Tools'
            DependsOn = '[WindowsFeature]ADDS'
        }
        
        # Configuration réseau conditionnelle
        if ($ComputerName -ne 'localhost') {
            xIPAddress StaticIP {
                IPAddress = '192.168.1.100'
                InterfaceAlias = 'Ethernet'
                SubnetMask = 24
                AddressFamily = 'IPv4'
            }
        }
        
        # Promotion du contrôleur de domaine
        xADDomainController DomainController {
            DomainName = $domainName
            DomainAdministratorCredential = $DomainCredential
            SafemodeAdministratorPassword = $DomainCredential
            DependsOn = '[WindowsFeature]ADDS'
        }
        
        # Création d'unités d'organisation
        xADOrganizationalUnit ITDept {
            Name = 'IT'
            Path = "DC=$($domainName.Replace('.', ',DC='))"
            Ensure = 'Present'
            DependsOn = '[xADDomainController]DomainController'
        }
        
        # Création d'utilisateurs
        xADUser AdminUser {
            DomainName = $domainName
            UserName = 'admin'
            Password = $DomainCredential
            Ensure = 'Present'
            DependsOn = '[xADOrganizationalUnit]ITDept'
        }
    }
}

# Génération du MOF
$credential = Get-Credential
AdvancedServerConfig -ComputerName 'DC01' -DomainCredential $credential

# Test de la configuration
Test-DscConfiguration -Path .\AdvancedServerConfig

# Application forcée
Start-DscConfiguration -Path .\AdvancedServerConfig -Wait -Force -Verbose
```

### 3.4 Pull Mode et serveur DSC

```powershell
# Configuration pour le mode Pull
[DscLocalConfigurationManager()]
Configuration LCMPullConfig {
    param([string]$ComputerName)
    
    Node $ComputerName {
        Settings {
            RefreshMode = 'Pull'
            ConfigurationID = '8e6d96a7-9c75-4b29-b1b9-0b7c8a3e6c2f'
            DownloadManagerName = 'WebDownloadManager'
            DownloadManagerCustomData = @{
                ServerUrl = 'https://dsc.contoso.com:8080/PSDSCPullServer.svc'
                AllowUnsecureConnection = $false
            }
            RebootNodeIfNeeded = $true
        }
    }
}

# Application de la configuration LCM
LCMPullConfig -ComputerName 'Server01'
Set-DscLocalConfigurationManager -Path .\LCMPullConfig -Verbose

# Publication sur le serveur Pull
Publish-DscConfiguration -Path .\WebServerConfig -ComputerName 'Server01' -Verbose
```

## Section 4 : Classes et programmation orientée objet

### 4.1 Définition de classes (PowerShell 5+)

```powershell
# Définition d'une classe simple
class Employee {
    [string]$FirstName
    [string]$LastName
    [int]$EmployeeId
    [DateTime]$HireDate
    hidden [decimal]$Salary
    
    # Constructeur
    Employee([string]$first, [string]$last, [int]$id) {
        $this.FirstName = $first
        $this.LastName = $last
        $this.EmployeeId = $id
        $this.HireDate = Get-Date
    }
    
    # Méthode
    [string] GetFullName() {
        return "$($this.FirstName) $($this.LastName)"
    }
    
    # Propriété calculée
    [int] YearsOfService() {
        return ((Get-Date) - $this.HireDate).Days / 365
    }
    
    # Méthode statique
    static [Employee] CreateFromCsv([string]$csvLine) {
        $data = $csvLine -split ','
        return [Employee]::new($data[0], $data[1], [int]$data[2])
    }
}

# Utilisation
$emp = [Employee]::new("John", "Doe", 12345)
$emp.FirstName = "Jane"
Write-Host $emp.GetFullName()
Write-Host "Années de service: $($emp.YearsOfService())"

# Héritage
class Manager : Employee {
    [string[]]$DirectReports
    
    Manager([string]$first, [string]$last, [int]$id) : base($first, $last, $id) {
        $this.DirectReports = @()
    }
    
    [void] AddDirectReport([Employee]$employee) {
        $this.DirectReports += $employee.EmployeeId
    }
    
    [int] GetTeamSize() {
        return $this.DirectReports.Count
    }
}

$manager = [Manager]::new("Alice", "Smith", 54321)
$manager.AddDirectReport($emp)
Write-Host "Taille de l'équipe: $($manager.GetTeamSize())"
```

### 4.2 DSC Resources basés sur des classes

```powershell
# Resource DSC basée sur une classe
class CustomFileResource {
    [DscProperty(Key)]
    [string] $Path
    
    [DscProperty(Mandatory)]
    [string] $Content
    
    [DscProperty()]
    [string] $Encoding = 'UTF8'
    
    [DscProperty(NotConfigurable)]
    [string] $CurrentContent
    
    [void] Set() {
        Set-Content -Path $this.Path -Value $this.Content -Encoding $this.Encoding
    }
    
    [bool] Test() {
        if (-not (Test-Path $this.Path)) {
            return $false
        }
        
        $currentContent = Get-Content -Path $this.Path -Raw
        $this.CurrentContent = $currentContent
        
        return $currentContent -eq $this.Content
    }
    
    [CustomFileResource] Get() {
        $current = [CustomFileResource]::new()
        $current.Path = $this.Path
        
        if (Test-Path $this.Path) {
            $current.Content = Get-Content -Path $this.Path -Raw
            $current.CurrentContent = $current.Content
        }
        
        return $current
    }
}

# Utilisation dans une configuration DSC
Configuration FileConfig {
    Import-DscResource -ModuleName MyCustomResources
    
    Node 'localhost' {
        CustomFile MyConfigFile {
            Path = 'C:\Config\app.config'
            Content = @"
<?xml version="1.0"?>
<configuration>
    <appSettings>
        <add key="Environment" value="Production"/>
    </appSettings>
</configuration>
"@
        }
    }
}
```

## Section 5 : Intégration avec .NET

### 5.1 Utilisation des assemblies .NET

```powershell
# Chargement d'assemblies
Add-Type -AssemblyName System.Windows.Forms
Add-Type -AssemblyName System.Drawing
Add-Type -AssemblyName System.Web

# Utilisation des classes .NET
$list = New-Object System.Collections.Generic.List[string]
$list.Add("Item1")
$list.Add("Item2")

# LINQ avec PowerShell
$numbers = 1..100
$evenNumbers = [System.Linq.Enumerable]::Where($numbers, [Func[int,bool]]{ param($n) $n % 2 -eq 0 })
$sum = [System.Linq.Enumerable]::Sum([int[]]$evenNumbers)

# Sérialisation JSON
$user = [PSCustomObject]@{
    Name = "John Doe"
    Age = 30
    Skills = @("PowerShell", "C#", ".NET")
}

$json = [System.Text.Json.JsonSerializer]::Serialize($user, [System.Text.Json.JsonSerializerOptions]::new())
$userFromJson = [System.Text.Json.JsonSerializer]::Deserialize($json, [PSCustomObject])

# Expressions régulières compilées
$pattern = [regex]::new('\b\d{3}-\d{2}-\d{4}\b', [System.Text.RegularExpressions.RegexOptions]::Compiled)
$matches = $pattern.Matches("SSN: 123-45-6789 and 987-65-4321")
```

### 5.2 Création de types personnalisés

```powershell
# Définition d'un type avec Add-Type
Add-Type -TypeDefinition @"
using System;
using System.Collections.Generic;

public class CustomLogger {
    private List<string> logEntries = new List<string>();
    
    public void Log(string message) {
        string entry = $"{DateTime.Now}: {message}";
        logEntries.Add(entry);
        Console.WriteLine(entry);
    }
    
    public string[] GetLogEntries() {
        return logEntries.ToArray();
    }
    
    public void ClearLog() {
        logEntries.Clear();
    }
}
"@ -Language CSharp

# Utilisation du type personnalisé
$logger = New-Object CustomLogger
$logger.Log("Application démarrée")
$logger.Log("Traitement des données")
$entries = $logger.GetLogEntries()
Write-Host "Nombre d'entrées: $($entries.Count)"
```

### 5.3 Interopérabilité COM

```powershell
# Utilisation d'objets COM
$excel = New-Object -ComObject Excel.Application
$excel.Visible = $true

$workbook = $excel.Workbooks.Add()
$worksheet = $workbook.Worksheets.Item(1)

# Écriture de données
$worksheet.Cells.Item(1, 1) = "Nom"
$worksheet.Cells.Item(1, 2) = "Valeur"

$data = @(
    @("CPU", "85%"),
    @("Mémoire", "6.2GB"),
    @("Disque", "45%")
)

for ($i = 0; $i -lt $data.Count; $i++) {
    $worksheet.Cells.Item($i + 2, 1) = $data[$i][0]
    $worksheet.Cells.Item($i + 2, 2) = $data[$i][1]
}

# Sauvegarde
$workbook.SaveAs("C:\Reports\SystemReport.xlsx")
$excel.Quit()

# Nettoyage
[System.Runtime.InteropServices.Marshal]::ReleaseComObject($worksheet) | Out-Null
[System.Runtime.InteropServices.Marshal]::ReleaseComObject($workbook) | Out-Null
[System.Runtime.InteropServices.Marshal]::ReleaseComObject($excel) | Out-Null
[GC]::Collect()
```

## Section 6 : Debugging et profiling avancés

### 6.1 Debugging avancé

```powershell
# Points d'arrêt conditionnels
Set-PSBreakpoint -Script "C:\Scripts\MyScript.ps1" -Line 25 -Action {
    if ($i -gt 10) {
        break
    }
}

# Points d'arrêt sur variables
Set-PSBreakpoint -Variable result -Mode Write

# Debugging de workflows
Debug-Job -Job $workflowJob

# Inspection de la pile d'appels
Get-PSCallStack

# Mode strict pour le debugging
Set-StrictMode -Version Latest

# Validation des paramètres
function Test-Parameter {
    param(
        [Parameter(Mandatory)]
        [ValidateNotNullOrEmpty()]
        [string]$Name,
        
        [ValidateRange(1, 100)]
        [int]$Value,
        
        [ValidateSet("Low", "Medium", "High")]
        [string]$Priority = "Medium"
    )
    
    Write-Host "Paramètres validés: Name=$Name, Value=$Value, Priority=$Priority"
}
```

### 6.2 Profiling des performances

```powershell
# Mesure précise du temps d'exécution
$stopwatch = [System.Diagnostics.Stopwatch]::StartNew()

# Code à mesurer
1..1000 | ForEach-Object { $_ * 2 } | Out-Null

$stopwatch.Stop()
Write-Host "Temps d'exécution: $($stopwatch.ElapsedMilliseconds) ms"

# Profiling avec Trace-Command
Trace-Command -Name ParameterBinding -Expression { Get-Process -Name "powershell" } -PSHost

# Analyse de la mémoire
$before = [GC]::GetTotalMemory($false)
# Code qui consomme de la mémoire
$largeArray = 1..100000
$after = [GC]::GetTotalMemory($false)
$used = $after - $before

Write-Host "Mémoire utilisée: $([math]::Round($used/1MB, 2)) MB"

# Détection des fuites mémoire
function Test-MemoryUsage {
    param([ScriptBlock]$ScriptBlock, [int]$Iterations = 10)
    
    $measurements = @()
    
    for ($i = 1; $i -le $Iterations; $i++) {
        [GC]::Collect()
        $before = [GC]::GetTotalMemory($true)
        
        & $ScriptBlock
        
        [GC]::Collect()
        $after = [GC]::GetTotalMemory($true)
        
        $measurements += $after - $before
    }
    
    $average = ($measurements | Measure-Object -Average).Average
    $max = ($measurements | Measure-Object -Maximum).Maximum
    
    Write-Host "Utilisation mémoire moyenne: $([math]::Round($average/1KB, 2)) KB"
    Write-Host "Utilisation mémoire maximale: $([math]::Round($max/1KB, 2)) KB"
    
    if ($measurements[-1] -gt ($measurements[0] * 1.5)) {
        Write-Warning "Possible fuite mémoire détectée"
    }
}

Test-MemoryUsage -ScriptBlock { $array = 1..10000; $array | Where-Object { $_ % 2 -eq 0 } | Out-Null }
```

## Conclusion : PowerShell comme plateforme d'entreprise

Les fonctionnalités avancées de PowerShell transforment l'environnement de scripting en une véritable plateforme d'orchestration d'entreprise. Jobs, workflows, DSC, et l'intégration profonde avec .NET permettent de construire des solutions robustes et maintenables pour les défis les plus complexes de l'administration système.

Dans le prochain chapitre, nous explorerons l'intégration de PowerShell avec les technologies cloud, notamment Azure et AWS, pour découvrir comment étendre ces capacités à l'infrastructure cloud-native.

---

**Exercice pratique :** Créez une solution complète de déploiement automatisé qui utilise :
1. Jobs pour l'exécution parallèle
2. Workflows pour la gestion d'état
3. DSC pour la configuration déclarative
4. Classes personnalisées pour la logique métier
5. Intégration .NET pour les performances

**Challenge avancé :** Développez un système de monitoring distribué utilisant PowerShell qui :
- Collecte des métriques sur plusieurs serveurs
- Utilise des jobs pour la collecte asynchrone
- Stocke les données dans une base de données
- Génère des rapports automatisés
- Implémente des alertes configurables

**Réflexion :** Comment ces fonctionnalités avancées changent-elles l'approche traditionnelle du scripting, et quels sont les impacts sur l'architecture des solutions d'automatisation ?

