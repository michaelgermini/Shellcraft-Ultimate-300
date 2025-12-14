# Chapitre 123 - Cmdlets et pipeline objet

> "Le pipeline objet est la feature qui rend PowerShell supérieur à tous les autres shells - c'est la différence entre manipuler du texte et manipuler des données structurées." - Maître PowerShell

## Introduction : Le cœur battant de PowerShell

Si la syntaxe forme le squelette de PowerShell, les cmdlets et le pipeline objet en sont le cœur et l'âme. Cette architecture unique permet de composer des opérations complexes avec une élégance et une puissance sans précédent dans le monde des shells.

Dans ce chapitre, nous explorerons en profondeur le système de cmdlets, la mécanique du pipeline objet, et comment ces éléments se combinent pour créer des workflows d'automatisation sophistiqués. Nous découvrirons également les providers qui étendent l'accès unifié aux données.

## Section 1 : Architecture des cmdlets

### 1.1 Qu'est-ce qu'un cmdlet ?

Un cmdlet (command-let) est une commande légère .NET qui suit des conventions strictes :

```powershell
# Structure d'un cmdlet
Verbe-Nom

# Exemples de cmdlets intégrés
Get-Process      # Récupère les processus
Set-Location     # Change le répertoire courant
New-Object       # Crée un nouvel objet
Remove-Item      # Supprime un élément
```

**Caractéristiques des cmdlets :**
- **Orientés objet** : Reçoivent et retournent des objets .NET
- **Convention de nommage** : Verbe-Nom systématique
- **Paramétrage riche** : Paramètres positionnels et nommés
- **Aide intégrée** : Documentation automatique
- **Gestion d'erreur** : Retour d'objets d'erreur structurés

### 1.2 Cycle de vie d'un cmdlet

```powershell
# 1. Découverte des cmdlets disponibles
Get-Command -CommandType Cmdlet

# 2. Obtention d'aide détaillée
Get-Help Get-Process -Detailed

# 3. Découverte des paramètres
Get-Command Get-Process -Syntax

# 4. Exécution avec différents paramètres
Get-Process -Name "powershell"
Get-Process -Id 1234
Get-Process | Where-Object CPU -gt 10
```

### 1.3 Création de cmdlets personnalisés

```powershell
# Fonction comme cmdlet simple
function Get-SystemInfo {
    [CmdletBinding()]
    param()
    
    $osInfo = Get-WmiObject Win32_OperatingSystem
    $computerInfo = Get-WmiObject Win32_ComputerSystem
    
    [PSCustomObject]@{
        ComputerName = $env:COMPUTERNAME
        OSVersion = $osInfo.Caption
        OSArchitecture = $osInfo.OSArchitecture
        TotalMemoryGB = [math]::Round($computerInfo.TotalPhysicalMemory / 1GB, 2)
        Manufacturer = $computerInfo.Manufacturer
        Model = $computerInfo.Model
    }
}

# Utilisation
Get-SystemInfo
```

## Section 2 : Le pipeline objet

### 2.1 Comprendre le pipeline traditionnel vs objet

**Pipeline traditionnel (Bash) :**
```bash
ps aux | grep nginx | awk '{print $2}' | xargs kill
# Manipule du texte -> parsing -> texte -> parsing...
```

**Pipeline objet (PowerShell) :**
```powershell
Get-Process nginx | Stop-Process
# Manipule des objets -> propriétés directes -> méthodes natives
```

### 2.2 Mécanisme du pipeline objet

```powershell
# Le pipeline passe des objets entiers
Get-Service | Where-Object { $_.Status -eq 'Running' } | Select-Object Name, DisplayName

# Chaque étape reçoit des objets et en retourne
$services = Get-Service                    # Retourne des objets ServiceController
$running = $services | Where-Object { $_.Status -eq 'Running' }  # Filtre les objets
$selected = $running | Select-Object Name, DisplayName          # Projette les propriétés
```

### 2.3 Cmdlets de manipulation d'objets

```powershell
# Where-Object : filtrage
Get-Process | Where-Object { $_.CPU -gt 5 }
Get-Service | Where-Object Status -eq 'Stopped'
Get-EventLog -LogName System | Where-Object { $_.EntryType -eq 'Error' -and $_.TimeGenerated -gt (Get-Date).AddHours(-24) }

# Select-Object : projection et transformation
Get-Process | Select-Object Name, CPU, @{Name='MemoryMB'; Expression={$_.WorkingSet / 1MB}}
Get-ADUser -Filter * | Select-Object Name, SamAccountName, @{Name='Department'; Expression={(Get-ADUser $_.SamAccountName -Properties Department).Department}}

# Sort-Object : tri
Get-Process | Sort-Object CPU -Descending
Get-Service | Sort-Object Status, Name

# Group-Object : groupage
Get-Process | Group-Object Company | Sort-Object Count -Descending
Get-EventLog -LogName System | Group-Object EntryType

# Measure-Object : agrégation
Get-Process | Measure-Object -Property CPU -Sum -Average -Maximum
Get-ChildItem | Measure-Object -Property Length -Sum

# Compare-Object : comparaison
Compare-Object (Get-Content fichier1.txt) (Get-Content fichier2.txt)
Compare-Object (Get-Process) (Get-Process) -Property Name
```

### 2.4 Pipeline avancé et optimisation

```powershell
# ForEach-Object : traitement individuel
1..10 | ForEach-Object { $_ * 2 }
Get-Process | ForEach-Object { 
    [PSCustomObject]@{
        Name = $_.Name
        CPUPercent = [math]::Round($_.CPU, 2)
        MemoryMB = [math]::Round($_.WorkingSet / 1MB, 2)
    }
}

# Tee-Object : bifurcation du pipeline
Get-Process | Tee-Object -Variable allProcesses | Where-Object CPU -gt 10 | Stop-Process
# $allProcesses contient tous les processus originaux

# Pipeline parallélisation (PowerShell 7+)
1..100 | ForEach-Object -Parallel { 
    Start-Sleep -Milliseconds (Get-Random -Maximum 1000)
    "Traitement de $_ terminé"
} -ThrottleLimit 10
```

## Section 3 : Les providers PowerShell

### 3.1 Concept des providers

Les providers permettent un accès unifié à différentes sources de données :

```powershell
# Liste des providers disponibles
Get-PSProvider

# Navigation unifiée
Set-Location C:\                # Système de fichiers
Set-Location HKLM:\             # Registre
Set-Location Cert:\             # Certificats
Set-Location WSMan:\            # WS-Management
Set-Location Function:\         # Fonctions PowerShell
Set-Location Alias:\            # Alias
```

### 3.2 Provider FileSystem

```powershell
# Navigation et manipulation de fichiers
Set-Location C:\Windows

# Lister avec métadonnées
Get-ChildItem | Select-Object Name, Length, LastWriteTime, Attributes

# Recherche récursive
Get-ChildItem -Recurse -Include *.log | Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) }

# Création et manipulation
New-Item -ItemType Directory -Path "C:\Temp\MonDossier"
New-Item -ItemType File -Path "C:\Temp\test.txt" -Value "Contenu du fichier"

# Copie et déplacement
Copy-Item "C:\source\*" "C:\destination\" -Recurse
Move-Item "C:\ancien\nom.txt" "C:\nouveau\nom.txt"

# Propriétés étendues
Get-Item "C:\Windows\notepad.exe" | Select-Object * | Format-List
```

### 3.3 Provider Registry

```powershell
# Navigation dans le registre
Set-Location HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion

# Lecture de valeurs
Get-ItemProperty . -Name ProgramFilesDir
(Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion -Name ProgramFilesDir).ProgramFilesDir

# Modification du registre
New-Item -Path HKCU:\Software -Name "MonApplication"
New-ItemProperty -Path HKCU:\Software\MonApplication -Name "Version" -Value "1.0" -PropertyType String
Set-ItemProperty -Path HKCU:\Software\MonApplication -Name "Version" -Value "1.1"

# Suppression
Remove-ItemProperty -Path HKCU:\Software\MonApplication -Name "Version"
Remove-Item -Path HKCU:\Software\MonApplication
```

### 3.4 Provider Certificate

```powershell
# Gestion des certificats
Set-Location Cert:\CurrentUser\My

# Lister les certificats personnels
Get-ChildItem | Select-Object Subject, Issuer, NotAfter, Thumbprint

# Certificats expirés
Get-ChildItem | Where-Object { $_.NotAfter -lt (Get-Date) }

# Import/Export
$cert = Get-PfxCertificate -FilePath "C:\certificates\moncert.pfx"
Export-Certificate -Cert $cert -FilePath "C:\certificates\moncert.cer"

# Recherche dans tous les stores
Get-ChildItem Cert:\ -Recurse | Where-Object { $_.Subject -like "*example*" }
```

### 3.5 Création de providers personnalisés

```powershell
# Exemple de provider simple pour JSON
function New-JsonProvider {
    param([string]$JsonFile)
    
    $json = Get-Content $JsonFile | ConvertFrom-Json
    
    # Créer une structure navigable
    $provider = [PSCustomObject]@{
        PSTypeName = 'JsonProvider'
        Path = $JsonFile
        Data = $json
    }
    
    # Méthodes de navigation
    Add-Member -InputObject $provider -MemberType ScriptMethod -Name GetChildItem -Value {
        $this.Data.PSObject.Properties | ForEach-Object {
            [PSCustomObject]@{
                Name = $_.Name
                Value = $_.Value
                Type = $_.Value.GetType().Name
            }
        }
    }
    
    return $provider
}

# Utilisation
$jsonProvider = New-JsonProvider "config.json"
$jsonProvider.GetChildItem()
```

## Section 4 : Cmdlets avancés et composition

### 4.1 Cmdlets de formatage

```powershell
# Format-Table : affichage tabulaire
Get-Process | Format-Table -Property Name, CPU, @{Name='Memory(MB)'; Expression={[math]::Round($_.WorkingSet/1MB,1)}}

# Format-List : affichage détaillé
Get-Service | Format-List Name, Status, DisplayName, StartType

# Format-Wide : affichage compact
Get-Process | Format-Wide -Column 4

# Format-Custom : format personnalisé
Get-Process | Format-Custom -Property Name, Id, CPU

# Out-GridView : interface graphique
Get-Process | Out-GridView -Title "Sélectionnez des processus"

# Export vers différents formats
Get-Service | Export-Csv -Path services.csv -NoTypeInformation
Get-Process | Export-Clixml -Path processus.xml
```

### 4.2 Cmdlets de conversion et parsing

```powershell
# ConvertTo/ConvertFrom
$xml = Get-Process | ConvertTo-Xml
$processes = $xml | ConvertFrom-Xml

# JSON
$json = Get-Service | ConvertTo-Json
$services = $json | ConvertFrom-Json

# CSV
$csv = Import-Csv -Path "utilisateurs.csv"
$csv | ForEach-Object { 
    New-ADUser -Name $_.Name -SamAccountName $_.SamAccountName 
}

# Encodage/Base64
$text = "Hello World"
$encoded = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($text))
$decoded = [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($encoded))
```

### 4.3 Cmdlets asynchrones et parallèles

```powershell
# Start-Job : exécution en arrière-plan
$job = Start-Job -ScriptBlock { 
    Get-Process | Where-Object CPU -gt 5 
}

# Gestion des jobs
Get-Job
Receive-Job -Job $job -Wait
Remove-Job -Job $job

# Invoke-Command : exécution distante
$session = New-PSSession -ComputerName "server1", "server2"
Invoke-Command -Session $session -ScriptBlock { 
    Get-Service | Where-Object Status -eq 'Stopped' 
}

# Workflows (PowerShell 3.0+)
workflow Parallel-Processing {
    parallel {
        InlineScript { Get-Process -Name "chrome" }
        InlineScript { Get-Service -Name "*sql*" }
        InlineScript { Get-EventLog -LogName System -Newest 10 }
    }
}
Parallel-Processing
```

## Section 5 : Performance et optimisation

### 5.1 Mesure et profiling

```powershell
# Measure-Command : mesure du temps d'exécution
$time = Measure-Command { 
    1..10000 | ForEach-Object { $_ * 2 } 
}
Write-Host "Temps d'exécution: $($time.TotalMilliseconds) ms"

# Comparaison de performances
# Pipeline vs foreach
Measure-Command { 1..1000 | ForEach-Object { $_ } } | Select-Object TotalMilliseconds
Measure-Command { foreach ($i in 1..1000) { $i } } | Select-Object TotalMilliseconds

# Optimisation avec -Filter quand disponible
Measure-Command { Get-Process | Where-Object Name -like "*power*" }
Measure-Command { Get-Process -Name "*power*" }
```

### 5.2 Patterns d'optimisation

```powershell
# 1. Utiliser les paramètres natifs plutôt que Where-Object
# Lent
Get-ChildItem C:\Windows -Recurse | Where-Object { $_.Extension -eq '.exe' }

# Plus rapide
Get-ChildItem C:\Windows -Recurse -Filter *.exe

# 2. Éviter les pipelines inutiles
# Au lieu de :
Get-Process | Select-Object Name | Sort-Object Name

# Mieux :
Get-Process | Sort-Object Name | Select-Object Name

# 3. Utiliser les cmdlets spécialisés
# Au lieu de parser du texte :
netstat -ano | Select-String "LISTENING"

# Mieux :
Get-NetTCPConnection | Where-Object State -eq 'Listen'

# 4. Préférer les tableaux aux pipelines pour les données réutilisées
$processes = Get-Process  # Un seul appel
$cpuIntensive = $processes | Where-Object CPU -gt 10
$memoryIntensive = $processes | Where-Object WorkingSet -gt 100MB
```

### 5.3 Gestion de la mémoire

```powershell
# Forcer le garbage collection
[GC]::Collect()

# Mesure de l'utilisation mémoire
$before = [GC]::GetTotalMemory($false)
# Code à mesurer
$after = [GC]::GetTotalMemory($false)
$used = $after - $before
Write-Host "Mémoire utilisée: $([math]::Round($used/1MB, 2)) MB"

# Nettoyage des variables volumineuses
$bigData = $null
Remove-Variable bigData
[GC]::Collect()
```

## Section 6 : Création de cmdlets avancés

### 6.1 Classes et DSC Resources

```powershell
# Classes PowerShell (PowerShell 5+)
class MonCmdlet {
    [string]$Nom
    [int]$Valeur
    
    MonCmdlet([string]$nom, [int]$valeur) {
        $this.Nom = $nom
        $this.Valeur = $valeur
    }
    
    [string] ToString() {
        return "$($this.Nom): $($this.Valeur)"
    }
    
    [int] Double() {
        return $this.Valeur * 2
    }
}

# Utilisation
$objet = [MonCmdlet]::new("Test", 42)
$objet.ToString()
$objet.Double()
```

### 6.2 Modules binaires et compilation

```powershell
# Création d'un module binaire C#
Add-Type -TypeDefinition @"
using System;
using System.Management.Automation;

[Cmdlet(VerbsCommon.Get, "CustomInfo")]
public class GetCustomInfoCommand : PSCmdlet
{
    [Parameter(Mandatory = true)]
    public string Name { get; set; }
    
    protected override void ProcessRecord()
    {
        WriteObject($"Informations pour {Name}: {DateTime.Now}");
    }
}
"@ -Language CSharp

# Utilisation du cmdlet compilé
Get-CustomInfo -Name "Test"
```

### 6.3 Fonctions avancées avec validation

```powershell
function New-ValidatedUser {
    [CmdletBinding(SupportsShouldProcess = $true, ConfirmImpact = 'Medium')]
    param(
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [ValidatePattern('^[a-zA-Z][a-zA-Z0-9_-]{2,15}$')]
        [string]$UserName,
        
        [Parameter(Mandatory = $true)]
        [ValidateRange(18, 120)]
        [int]$Age,
        
        [Parameter(Mandatory = $false)]
        [ValidateSet('Admin', 'User', 'Guest')]
        [string]$Role = 'User',
        
        [Parameter(Mandatory = $false)]
        [ValidateScript({ Test-Path $_ })]
        [string]$HomeDirectory
    )
    
    begin {
        $usersCreated = 0
    }
    
    process {
        if ($PSCmdlet.ShouldProcess($UserName, "Créer utilisateur")) {
            # Simulation de création
            Write-Verbose "Création de l'utilisateur $UserName"
            
            [PSCustomObject]@{
                UserName = $UserName
                Age = $Age
                Role = $Role
                HomeDirectory = $HomeDirectory
                Created = Get-Date
            }
            
            $usersCreated++
        }
    }
    
    end {
        Write-Verbose "$usersCreated utilisateurs créés"
    }
}

# Utilisation avec validation
New-ValidatedUser -UserName "john_doe123" -Age 30 -Role "Admin" -Verbose
```

## Conclusion : La puissance de la composition

Les cmdlets et le pipeline objet forment le fondement de la puissance de PowerShell. En permettant la composition fluide d'opérations sur des objets structurés, PowerShell transcende les limitations des shells textuels traditionnels.

Dans le prochain chapitre, nous explorerons l'administration système avec PowerShell, découvrant comment ces concepts s'appliquent aux tâches d'administration Windows réelles.

---

**Exercice pratique :** Créez un script PowerShell qui :
1. Récupère tous les services arrêtés
2. Les groupe par type de démarrage
3. Affiche un rapport formaté avec compteurs
4. Permet de redémarrer interactivement certains services

**Challenge avancé :** Implémentez un cmdlet personnalisé qui étend Get-ChildItem pour afficher des informations supplémentaires sur les fichiers (hash, encodage détecté, etc.).

**Réflexion :** Comment le pipeline objet change-t-il votre approche de la résolution de problèmes par rapport aux shells traditionnels ?

