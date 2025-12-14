# Chapitre 124 - Administration système avec PowerShell

> "PowerShell a transformé l'administration Windows d'un art obscur en une science accessible, remplaçant des heures de clics par des lignes de code élégantes." - Administrateur système expérimenté

## Introduction : De l'interface graphique au code

L'administration système Windows a longtemps été dominée par des interfaces graphiques complexes et des outils disparates. PowerShell apporte une approche unifiée, scriptable et automatisable à toutes les tâches d'administration. Que ce soit la gestion des processus, services, fichiers, utilisateurs ou la configuration système, PowerShell offre des cmdlets puissants et cohérents.

Dans ce chapitre, nous explorerons comment PowerShell révolutionne l'administration système Windows, en combinant la puissance du pipeline objet avec l'accès direct aux API système.

## Section 1 : Gestion des processus

### 1.1 Inspection et monitoring des processus

```powershell
# Lister tous les processus
Get-Process

# Processus spécifiques
Get-Process -Name "chrome", "firefox", "edge"
Get-Process -Id 1234, 5678

# Informations détaillées
Get-Process | Select-Object Name, Id, CPU, WorkingSet, StartTime | Format-Table -AutoSize

# Processus par utilisateur
Get-Process | Group-Object UserName | Sort-Object Count -Descending

# Processus consommant le plus de ressources
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
Get-Process | Sort-Object WorkingSet -Descending | Select-Object -First 10 |
    Select-Object Name, @{Name='MemoryMB'; Expression={[math]::Round($_.WorkingSet/1MB, 1)}}
```

### 1.2 Contrôle des processus

```powershell
# Arrêter des processus
Stop-Process -Name "notepad" -Confirm:$false
Stop-Process -Id 1234 -Force

# Arrêter tous les processus d'un utilisateur
Get-Process -IncludeUserName | Where-Object UserName -eq "DOMAIN\user" | Stop-Process

# Démarrer des processus
Start-Process -FilePath "notepad.exe"
Start-Process -FilePath "chrome.exe" -ArgumentList "https://example.com"

# Avec différentes options
Start-Process -FilePath "cmd.exe" -ArgumentList "/c ping google.com" -NoNewWindow -Wait

# Contrôle de priorité
Get-Process -Name "chrome" | ForEach-Object {
    $_.PriorityClass = "BelowNormal"
    $_.Refresh()
}

# Limitation des ressources (Windows 10+)
New-JobTrigger -AtStartup
$job = Start-Job -ScriptBlock {
    while ($true) {
        Get-Process | Where-Object CPU -gt 80 | ForEach-Object {
            Stop-Process $_.Id -Force
        }
        Start-Sleep -Seconds 60
    }
}
```

### 1.3 Analyse avancée des processus

```powershell
# Arbre des processus
function Get-ProcessTree {
    param([int]$ProcessId = $PID)
    
    $process = Get-Process -Id $ProcessId -ErrorAction SilentlyContinue
    if (-not $process) { return }
    
    # Informations du processus
    $info = [PSCustomObject]@{
        Id = $process.Id
        Name = $process.Name
        ParentId = $process.ParentId
        CPU = [math]::Round($process.CPU, 2)
        MemoryMB = [math]::Round($process.WorkingSet / 1MB, 1)
        StartTime = $process.StartTime
        Level = 0
    }
    
    # Récupérer les enfants
    $children = Get-Process | Where-Object ParentId -eq $ProcessId
    
    # Résultat
    $info
    
    # Récursion pour les enfants
    foreach ($child in $children) {
        Get-ProcessTree -ProcessId $child.Id | ForEach-Object {
            $_.Level = $info.Level + 1
            $_
        }
    }
}

# Afficher l'arbre des processus
Get-ProcessTree | Format-Table -Property @{
    Name='Indent'; Expression={'  ' * $_.Level}
}, Name, Id, ParentId, CPU, MemoryMB -AutoSize

# Détection de fuites mémoire
function Watch-MemoryLeak {
    param(
        [string]$ProcessName,
        [int]$IntervalSeconds = 60,
        [int]$DurationMinutes = 10
    )
    
    $endTime = (Get-Date).AddMinutes($DurationMinutes)
    $measurements = @()
    
    Write-Host "Surveillance des fuites mémoire pour $ProcessName..."
    
    while ((Get-Date) -lt $endTime) {
        $process = Get-Process -Name $ProcessName -ErrorAction SilentlyContinue
        if ($process) {
            $measurements += [PSCustomObject]@{
                Time = Get-Date
                MemoryMB = [math]::Round($process.WorkingSet / 1MB, 1)
                CPU = [math]::Round($process.CPU, 2)
            }
        }
        Start-Sleep -Seconds $IntervalSeconds
    }
    
    # Analyse des tendances
    if ($measurements.Count -gt 1) {
        $first = $measurements[0]
        $last = $measurements[-1]
        $memoryIncrease = $last.MemoryMB - $first.MemoryMB
        $timeSpan = $last.Time - $first.Time
        
        Write-Host "Analyse terminée:"
        Write-Host "  Durée: $($timeSpan.TotalMinutes) minutes"
        Write-Host "  Mémoire initiale: $($first.MemoryMB) MB"
        Write-Host "  Mémoire finale: $($last.MemoryMB) MB"
        Write-Host "  Augmentation: $memoryIncrease MB"
        Write-Host "  Tendance: $([math]::Round($memoryIncrease / $timeSpan.TotalMinutes, 2)) MB/min"
    }
    
    return $measurements
}

# Surveillance des processus critiques
$criticalProcesses = @("lsass", "winlogon", "services", "csrss")
foreach ($proc in $criticalProcesses) {
    $process = Get-Process -Name $proc -ErrorAction SilentlyContinue
    if (-not $process) {
        Write-Warning "Processus critique $proc non trouvé!"
    }
}
```

## Section 2 : Gestion des services Windows

### 2.1 Inspection des services

```powershell
# Lister tous les services
Get-Service

# Services par statut
Get-Service | Where-Object Status -eq 'Running'
Get-Service | Where-Object Status -eq 'Stopped'

# Services spécifiques
Get-Service -Name "wuauserv", "bits", "winrm"

# Informations détaillées
Get-Service | Select-Object Name, DisplayName, Status, StartType, ServiceType

# Services dépendants
Get-Service -Name "wuauserv" | Select-Object -ExpandProperty DependentServices
Get-Service -Name "wuauserv" | Select-Object -ExpandProperty RequiredServices

# Recherche de services
Get-Service | Where-Object DisplayName -like "*network*"
```

### 2.2 Contrôle des services

```powershell
# Démarrer/arrêter des services
Start-Service -Name "wuauserv"
Stop-Service -Name "wuauserv" -Force

# Redémarrer un service
Restart-Service -Name "spooler"

# Changer le type de démarrage
Set-Service -Name "wuauserv" -StartupType Automatic
Set-Service -Name "wuauserv" -StartupType Disabled

# Configuration avancée
Set-Service -Name "MyService" -DisplayName "Mon Service Personnalisé" -Description "Description détaillée"

# Gestion des dépendances
function Get-ServiceDependencyTree {
    param([string]$ServiceName)
    
    function Get-Dependencies {
        param([string]$Name, [int]$Level = 0)
        
        $service = Get-Service -Name $Name -ErrorAction SilentlyContinue
        if (-not $service) { return }
        
        # Afficher le service avec indentation
        Write-Host ("  " * $Level + $service.Name + " (" + $service.Status + ")")
        
        # Traiter les services requis
        foreach ($dep in $service.RequiredServices) {
            Get-Dependencies -Name $dep.Name -Level ($Level + 1)
        }
    }
    
    Write-Host "Arbre des dépendances pour $ServiceName :"
    Get-Dependencies -ServiceName $ServiceName
}

Get-ServiceDependencyTree -ServiceName "wuauserv"
```

### 2.3 Création et installation de services

```powershell
# Créer un service simple avec sc.exe
sc.exe create "MyService" binPath= "C:\MyApp\MyService.exe" start= auto

# Avec PowerShell (nécessite droits admin)
New-Service -Name "MyPowerShellService" `
    -BinaryPathName "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -ExecutionPolicy Bypass -File C:\Scripts\MyService.ps1" `
    -DisplayName "Mon Service PowerShell" `
    -Description "Service basé sur PowerShell" `
    -StartupType Automatic

# Script de service PowerShell
$serviceScript = @'
# Script de service PowerShell
$ErrorActionPreference = "Stop"

# Fonction principale du service
function Main-ServiceLoop {
    while ($true) {
        try {
            # Logique du service
            Write-EventLog -LogName Application -Source "MyPowerShellService" -EventId 1000 -EntryType Information -Message "Service en cours d'exécution"
            
            # Simulation de travail
            Start-Sleep -Seconds 60
        }
        catch {
            Write-EventLog -LogName Application -Source "MyPowerShellService" -EventId 1001 -EntryType Error -Message "Erreur: $($_.Exception.Message)"
            Start-Sleep -Seconds 30
        }
    }
}

# Enregistrer la source d'événements
if (-not (Get-EventLog -LogName Application -Source "MyPowerShellService" -ErrorAction SilentlyContinue)) {
    New-EventLog -LogName Application -Source "MyPowerShellService"
}

# Démarrer la boucle principale
Main-ServiceLoop
'@

# Sauvegarder le script
$serviceScript | Out-File -FilePath "C:\Scripts\MyService.ps1" -Encoding UTF8
```

## Section 3 : Gestion des fichiers et disques

### 3.1 Gestion avancée des fichiers

```powershell
# Analyse de l'espace disque
Get-WmiObject Win32_LogicalDisk | Where-Object { $_.DriveType -eq 3 } | 
    Select-Object DeviceID, VolumeName, 
        @{Name='SizeGB'; Expression={[math]::Round($_.Size/1GB, 2)}},
        @{Name='FreeGB'; Expression={[math]::Round($_.FreeSpace/1GB, 2)}},
        @{Name='UsedPercent'; Expression={[math]::Round(($_.Size - $_.FreeSpace)/$_.Size * 100, 1)}}

# Nettoyage automatique des fichiers temporaires
function Clear-TempFiles {
    param([int]$DaysOld = 7)
    
    $tempPaths = @(
        "$env:TEMP",
        "$env:TMP",
        "C:\Windows\Temp"
    )
    
    $totalCleaned = 0
    
    foreach ($path in $tempPaths) {
        if (Test-Path $path) {
            $oldFiles = Get-ChildItem -Path $path -Recurse -File | 
                Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-$DaysOld) }
            
            foreach ($file in $oldFiles) {
                try {
                    Remove-Item $file.FullName -Force
                    $totalCleaned += $file.Length
                }
                catch {
                    Write-Warning "Impossible de supprimer $($file.FullName): $($_.Exception.Message)"
                }
            }
        }
    }
    
    Write-Host "Nettoyage terminé. Espace libéré: $([math]::Round($totalCleaned/1MB, 2)) MB"
}

Clear-TempFiles -DaysOld 30
```

### 3.2 Gestion des permissions NTFS

```powershell
# Analyser les permissions
function Get-NTFSPermissions {
    param([string]$Path)
    
    Get-Acl -Path $Path | ForEach-Object {
        $_.Access | ForEach-Object {
            [PSCustomObject]@{
                Path = $Path
                IdentityReference = $_.IdentityReference
                FileSystemRights = $_.FileSystemRights
                AccessControlType = $_.AccessControlType
                IsInherited = $_.IsInherited
            }
        }
    }
}

Get-NTFSPermissions -Path "C:\Important" | Format-Table -AutoSize

# Modifier les permissions
$acl = Get-Acl -Path "C:\Shared"
$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "DOMAIN\Users", 
    "ReadAndExecute", 
    "ContainerInherit,ObjectInherit", 
    "None", 
    "Allow"
)
$acl.SetAccessRule($accessRule)
Set-Acl -Path "C:\Shared" -AclObject $acl

# Audit des permissions récursives
function Audit-Permissions {
    param([string]$RootPath, [string]$UserName)
    
    Get-ChildItem -Path $RootPath -Recurse | ForEach-Object {
        $acl = Get-Acl -Path $_.FullName
        $userAccess = $acl.Access | Where-Object IdentityReference -like "*$UserName*"
        
        if ($userAccess) {
            [PSCustomObject]@{
                Path = $_.FullName
                User = $UserName
                Rights = ($userAccess.FileSystemRights -join ", ")
                Type = $userAccess.AccessControlType
            }
        }
    }
}

Audit-Permissions -RootPath "C:\Data" -UserName "john.doe" | Export-Csv -Path "permissions_audit.csv" -NoTypeInformation
```

### 3.3 Gestion des disques et volumes

```powershell
# Informations sur les disques
Get-Disk | Select-Object Number, FriendlyName, Size, PartitionStyle, OperationalStatus

# Partitions
Get-Partition | Select-Object DiskNumber, PartitionNumber, DriveLetter, Size, Type

# Volumes
Get-Volume | Select-Object DriveLetter, FileSystem, Size, SizeRemaining, HealthStatus

# Créer une nouvelle partition
$disk = Get-Disk | Where-Object PartitionStyle -eq 'RAW' | Select-Object -First 1
if ($disk) {
    Initialize-Disk -Number $disk.Number -PartitionStyle GPT
    New-Partition -DiskNumber $disk.Number -UseMaximumSize -AssignDriveLetter | Format-Volume -FileSystem NTFS -NewFileSystemLabel "Data"
}

# Monter/démonter des volumes
Mount-DiskImage -ImagePath "C:\Images\backup.vhdx"
$Dismounted = Dismount-DiskImage -ImagePath "C:\Images\backup.vhdx"

# Vérification de l'intégrité
Repair-Volume -DriveLetter C -Scan

# Défragmentation
Optimize-Volume -DriveLetter C -Defrag
```

## Section 4 : Gestion des utilisateurs et groupes

### 4.1 Utilisateurs locaux

```powershell
# Lister les utilisateurs locaux
Get-LocalUser | Select-Object Name, Enabled, LastLogon, PasswordLastSet

# Créer un nouvel utilisateur
New-LocalUser -Name "john.doe" -Password (ConvertTo-SecureString "P@ssw0rd123!" -AsPlainText -Force) -FullName "John Doe" -Description "Utilisateur test"

# Modifier un utilisateur
Set-LocalUser -Name "john.doe" -PasswordNeverExpires $true -UserMayChangePassword $false

# Gérer les groupes locaux
Get-LocalGroup | Select-Object Name, Description

# Ajouter un utilisateur à un groupe
Add-LocalGroupMember -Group "Administrators" -Member "john.doe"

# Créer un groupe
New-LocalGroup -Name "Developers" -Description "Groupe des développeurs"

# Lister les membres d'un groupe
Get-LocalGroupMember -Group "Administrators"
```

### 4.2 Utilisateurs Active Directory (si disponible)

```powershell
# Importer le module AD
Import-Module ActiveDirectory

# Recherche d'utilisateurs
Get-ADUser -Filter * -Properties * | Select-Object Name, SamAccountName, EmailAddress, Department, Enabled

# Créer un utilisateur AD
New-ADUser -Name "Jane Smith" `
    -SamAccountName "jsmith" `
    -UserPrincipalName "jsmith@domain.com" `
    -GivenName "Jane" `
    -Surname "Smith" `
    -EmailAddress "jane.smith@domain.com" `
    -Department "IT" `
    -Title "System Administrator" `
    -AccountPassword (ConvertTo-SecureString "TempPass123!" -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $true

# Gérer les groupes AD
Get-ADGroup -Filter { Name -like "*admin*" } | Select-Object Name, GroupCategory, GroupScope

# Ajouter à un groupe
Add-ADGroupMember -Identity "Domain Admins" -Members "jsmith"

# Audit des comptes expirés
Search-ADAccount -AccountExpired | Select-Object Name, SamAccountName, AccountExpirationDate

# Réinitialisation de mots de passe en masse
$users = Get-ADUser -Filter { Department -eq "Sales" -and Enabled -eq $true }
foreach ($user in $users) {
    $newPassword = ConvertTo-SecureString "NewPass$(Get-Random -Maximum 9999)!" -AsPlainText -Force
    Set-ADAccountPassword -Identity $user -NewPassword $newPassword -Reset
    Set-ADUser -Identity $user -ChangePasswordAtLogon $true
}
```

## Section 5 : Configuration système

### 5.1 Registre Windows

```powershell
# Navigation dans le registre
Set-Location HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion

# Lire des valeurs
Get-ItemProperty -Path . -Name ProgramFilesDir
(Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion -Name ProgramFilesDir).ProgramFilesDir

# Modifier des valeurs
Set-ItemProperty -Path HKLM:\SOFTWARE\MyApp -Name Version -Value "2.0"
New-ItemProperty -Path HKLM:\SOFTWARE\MyApp -Name InstallDate -Value (Get-Date) -PropertyType String

# Recherche dans le registre
Get-ChildItem -Path HKLM:\SOFTWARE -Recurse | Where-Object { $_.Name -like "*microsoft*" }

# Sauvegarde du registre
reg export "HKLM\SOFTWARE\MyApp" "C:\Backup\myapp.reg"

# Import de paramètres registre
reg import "C:\Config\settings.reg"
```

### 5.2 Variables d'environnement

```powershell
# Lister toutes les variables d'environnement
Get-ChildItem Env:

# Variables spécifiques
$env:PATH
$env:COMPUTERNAME
$env:USERNAME

# Modifier des variables
$env:MyVar = "Nouvelle valeur"
[Environment]::SetEnvironmentVariable("MyVar", "Valeur persistante", "User")

# Ajouter au PATH
$currentPath = [Environment]::GetEnvironmentVariable("PATH", "Machine")
$newPath = "$currentPath;C:\MyTools"
[Environment]::SetEnvironmentVariable("PATH", $newPath, "Machine")
```

### 5.3 Planificateur de tâches

```powershell
# Lister les tâches planifiées
Get-ScheduledTask | Where-Object State -eq 'Ready' | Select-Object TaskName, TaskPath, State, LastRunTime

# Créer une tâche simple
$action = New-ScheduledTaskAction -Execute "PowerShell.exe" -Argument "-File C:\Scripts\MyScript.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "09:00"
Register-ScheduledTask -TaskName "DailyScript" -Action $action -Trigger $trigger -User "SYSTEM" -RunLevel Highest

# Tâche avec conditions
$trigger = New-ScheduledTaskTrigger -AtLogOn
$settings = New-ScheduledTaskSettingsSet -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
Register-ScheduledTask -TaskName "LogonScript" -Action $action -Trigger $trigger -Settings $settings

# Exécuter une tâche manuellement
Start-ScheduledTask -TaskName "DailyScript"

# Désactiver/activer des tâches
Disable-ScheduledTask -TaskName "OldTask"
Enable-ScheduledTask -TaskName "DailyScript"
```

## Section 6 : Monitoring et alertes

### 6.1 Collecte de métriques système

```powershell
# Fonction de monitoring système complet
function Get-SystemMetrics {
    $cpu = Get-WmiObject Win32_Processor | Measure-Object -Property LoadPercentage -Average
    $memory = Get-WmiObject Win32_OperatingSystem
    $disk = Get-WmiObject Win32_LogicalDisk | Where-Object { $_.DriveType -eq 3 }
    
    [PSCustomObject]@{
        Timestamp = Get-Date
        CPUUsage = [math]::Round($cpu.Average, 1)
        MemoryTotalGB = [math]::Round($memory.TotalVisibleMemorySize / 1MB, 2)
        MemoryUsedGB = [math]::Round(($memory.TotalVisibleMemorySize - $memory.FreePhysicalMemory) / 1MB, 2)
        MemoryUsagePercent = [math]::Round((($memory.TotalVisibleMemorySize - $memory.FreePhysicalMemory) / $memory.TotalVisibleMemorySize) * 100, 1)
        DiskTotalGB = [math]::Round(($disk | Measure-Object -Property Size -Sum).Sum / 1GB, 2)
        DiskFreeGB = [math]::Round(($disk | Measure-Object -Property FreeSpace -Sum).Sum / 1GB, 2)
        Uptime = (Get-Date) - $memory.LastBootUpTime
    }
}

# Collecte continue avec export
$metrics = @()
$endTime = (Get-Date).AddMinutes(60)

while ((Get-Date) -lt $endTime) {
    $metrics += Get-SystemMetrics
    Start-Sleep -Seconds 30
}

$metrics | Export-Csv -Path "system_metrics.csv" -NoTypeInformation
```

### 6.2 Alertes et notifications

```powershell
# Fonction d'alerte intelligente
function New-SystemAlert {
    param(
        [string]$AlertType,
        [string]$Message,
        [string]$Severity = "Warning",
        [string[]]$NotificationEmails = @()
    )
    
    $alert = [PSCustomObject]@{
        Timestamp = Get-Date
        Type = $AlertType
        Message = $Message
        Severity = $Severity
        ComputerName = $env:COMPUTERNAME
    }
    
    # Log dans l'Event Log
    $eventType = switch ($Severity) {
        "Critical" { "Error" }
        "Warning" { "Warning" }
        default { "Information" }
    }
    
    Write-EventLog -LogName Application -Source "SystemMonitor" -EventId 1000 -EntryType $eventType -Message $Message
    
    # Notification par email
    if ($NotificationEmails.Count -gt 0) {
        $body = @"
Alerte système détectée:

Type: $AlertType
Sévérité: $Severity
Message: $Message
Ordinateur: $env:COMPUTERNAME
Horodatage: $($alert.Timestamp)

-- Moniteur système automatique
"@
        
        Send-MailMessage -To $NotificationEmails -Subject "Alerte système: $AlertType" -Body $body -SmtpServer "smtp.company.com"
    }
    
    return $alert
}

# Monitoring avec alertes
function Start-SystemMonitor {
    param([int]$CheckIntervalSeconds = 300)
    
    Write-Host "Démarrage du monitoring système..."
    
    while ($true) {
        try {
            $metrics = Get-SystemMetrics
            
            # Alertes CPU
            if ($metrics.CPUUsage -gt 90) {
                New-SystemAlert -AlertType "HighCPU" -Message "Utilisation CPU: $($metrics.CPUUsage)%" -Severity "Critical" -NotificationEmails "admin@company.com"
            } elseif ($metrics.CPUUsage -gt 75) {
                New-SystemAlert -AlertType "HighCPU" -Message "Utilisation CPU: $($metrics.CPUUsage)%" -Severity "Warning"
            }
            
            # Alertes mémoire
            if ($metrics.MemoryUsagePercent -gt 90) {
                New-SystemAlert -AlertType "LowMemory" -Message "Mémoire utilisée: $($metrics.MemoryUsagePercent)%" -Severity "Critical"
            }
            
            # Alertes disque
            if ($metrics.DiskFreeGB -lt 10) {
                New-SystemAlert -AlertType "LowDiskSpace" -Message "Espace disque libre: $($metrics.DiskFreeGB) GB" -Severity "Warning"
            }
            
        } catch {
            New-SystemAlert -AlertType "MonitorError" -Message "Erreur de monitoring: $($_.Exception.Message)" -Severity "Critical"
        }
        
        Start-Sleep -Seconds $CheckIntervalSeconds
    }
}

# Démarrer le monitoring en arrière-plan
Start-Job -ScriptBlock ${function:Start-SystemMonitor} -Name "SystemMonitor"
```

## Conclusion : L'automatisation comme standard

PowerShell transforme l'administration système Windows d'une série de tâches manuelles répétitives en un ensemble de processus automatisés, fiables et maintenables. La combinaison du pipeline objet, des cmdlets spécialisés et de l'accès direct aux API système permet d'accomplir des tâches complexes avec élégance.

Dans le prochain chapitre, nous explorerons la gestion réseau avec PowerShell, découvrant comment administrer les configurations réseau, le firewall, et les connexions distantes.

---

**Exercice pratique :** Créez un script de maintenance système complet qui :
1. Nettoie les fichiers temporaires
2. Vérifie l'intégrité des disques
3. Met à jour les services critiques
4. Génère un rapport de santé système
5. Envoie des alertes en cas de problème

**Challenge :** Développez un module PowerShell personnalisé pour la gestion de votre environnement spécifique (serveurs web, bases de données, etc.).

**Réflexion :** Comment PowerShell change-t-il votre approche de l'administration système par rapport aux outils graphiques traditionnels ?

