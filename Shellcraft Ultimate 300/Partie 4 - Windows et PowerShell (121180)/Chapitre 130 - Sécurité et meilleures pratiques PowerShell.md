# Chapitre 130 - Sécurité et meilleures pratiques PowerShell

> "La sécurité n'est pas un produit, c'est un processus - et dans PowerShell, ce processus commence par chaque ligne de code que nous écrivons." - Expert en sécurité PowerShell

## Introduction : Sécurité first dans PowerShell

La sécurité est un aspect fondamental du développement PowerShell. Contrairement à d'autres langages où la sécurité peut être une réflexion après coup, PowerShell intègre la sécurité dans son ADN même - des politiques d'exécution aux meilleures pratiques de codage. Dans ce chapitre, nous explorerons les mécanismes de sécurité intégrés, les vulnérabilités communes, et les pratiques qui transforment les scripts PowerShell en solutions sécurisées et fiables.

De la validation des entrées à la gestion des secrets, nous couvrirons l'ensemble du spectre de la sécurité dans l'écosystème PowerShell.

## Section 1 : Politiques d'exécution et contrôle d'accès

### 1.1 Politiques d'exécution (Execution Policies)

```powershell
# Vérifier la politique d'exécution actuelle
Get-ExecutionPolicy -List

# Politiques disponibles :
# Restricted : Aucun script ne peut être exécuté (défaut)
# AllSigned : Seuls les scripts signés peuvent être exécutés
# RemoteSigned : Scripts locaux OK, scripts distants doivent être signés
# Unrestricted : Tous les scripts peuvent être exécutés
# Bypass : Rien n'est bloqué (dangereux)
# Undefined : Pas de politique définie (hérite du parent)

# Changer la politique pour l'utilisateur actuel
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Changer la politique pour tous les utilisateurs (nécessite admin)
Set-ExecutionPolicy -ExecutionPolicy AllSigned -Scope LocalMachine

# Forcer une politique pour une session
powershell.exe -ExecutionPolicy Bypass -File "script.ps1"

# Vérifier si un script peut être exécuté
function Test-ScriptExecution {
    param([string]$ScriptPath)
    
    try {
        $executionPolicy = Get-ExecutionPolicy
        $scriptSignature = Get-AuthenticodeSignature -FilePath $ScriptPath
        
        $canExecute = $true
        $reason = ""
        
        switch ($executionPolicy) {
            "Restricted" { 
                $canExecute = $false
                $reason = "Politique Restricted"
            }
            "AllSigned" {
                if ($scriptSignature.Status -ne "Valid") {
                    $canExecute = $false
                    $reason = "Script non signé ou signature invalide"
                }
            }
            "RemoteSigned" {
                $isRemote = $ScriptPath -like "\\*" -or $ScriptPath -like "http*"
                if ($isRemote -and $scriptSignature.Status -ne "Valid") {
                    $canExecute = $false
                    $reason = "Script distant non signé"
                }
            }
        }
        
        [PSCustomObject]@{
            ScriptPath = $ScriptPath
            CanExecute = $canExecute
            ExecutionPolicy = $executionPolicy
            SignatureStatus = $scriptSignature.Status
            Reason = $reason
        }
        
    } catch {
        Write-Error "Erreur lors de la vérification: $($_.Exception.Message)"
    }
}

Test-ScriptExecution -ScriptPath "C:\Scripts\MyScript.ps1"
```

### 1.2 Signature de code et authenticode

```powershell
# Créer un certificat de signature (pour développement)
$cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject "CN=PowerShellDev" -CertStoreLocation Cert:\CurrentUser\My

# Signer un script
Set-AuthenticodeSignature -FilePath "C:\Scripts\MyScript.ps1" -Certificate $cert

# Vérifier la signature
Get-AuthenticodeSignature -FilePath "C:\Scripts\MyScript.ps1"

# Lister les certificats de signature
Get-ChildItem Cert:\CurrentUser\My -CodeSigningCert

# Signature avec un certificat de l'entreprise
$enterpriseCert = Get-ChildItem Cert:\CurrentUser\My -CodeSigningCert | Where-Object Subject -like "*Company*"
Set-AuthenticodeSignature -FilePath "C:\Scripts\CompanyScript.ps1" -Certificate $enterpriseCert

# Validation de signature dans un script
function Assert-ScriptSignature {
    param([string]$ScriptPath)
    
    $signature = Get-AuthenticodeSignature -FilePath $ScriptPath
    
    if ($signature.Status -ne "Valid") {
        throw "Script non signé ou signature invalide: $($signature.Status)"
    }
    
    # Vérifier l'émetteur du certificat
    $issuer = $signature.SignerCertificate.Issuer
    if ($issuer -notlike "*Trusted Authority*") {
        Write-Warning "Certificat émis par: $issuer"
    }
    
    Write-Host "✓ Signature validée pour $ScriptPath"
}

# Utilisation dans un script critique
Assert-ScriptSignature -ScriptPath $PSCommandPath
```

### 1.3 Constrained Language Mode et AppLocker

```powershell
# Vérifier le mode de langage
$ExecutionContext.SessionState.LanguageMode

# Modes disponibles :
# FullLanguage : Fonctionnalités complètes (défaut)
# RestrictedLanguage : Limité (utilisé dans certaines configurations)
# NoLanguage : Très limité
# ConstrainedLanguage : Fonctionnalités limitées pour la sécurité

# Forcer le mode Constrained Language
$ExecutionContext.SessionState.LanguageMode = "ConstrainedLanguage"

# Fonctions disponibles en mode Constrained
# En mode ConstrainedLanguage, seules certaines constructions sont autorisées :
# - Cmdlets approuvés
# - Fonctions de base
# - Pas d'accès aux types .NET arbitraires
# - Pas de Add-Type
# - Pas d'invocation de méthodes sur certains objets

# Configuration AppLocker pour PowerShell
# Via GPO ou directement dans le registre
$regPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System"
Set-ItemProperty -Path $regPath -Name "EnableLinkedConnections" -Value 1

# Règles AppLocker pour PowerShell
$appLockerRules = @"
<?xml version="1.0" encoding="utf-8"?>
<AppLockerPolicy Version="1">
  <RuleCollection Type="Exe" EnforcementMode="Enabled">
    <FilePathRule Id="921cc481-6e17-4653-8f75-050b80acca20" Name="PowerShell.exe" Description="Autoriser PowerShell.exe" UserOrGroupSid="S-1-1-0" Action="Allow">
      <Conditions>
        <FilePathCondition Path="%SYSTEM32%\WindowsPowerShell\v1.0\powershell.exe" />
      </Conditions>
    </FilePathRule>
  </RuleCollection>
</AppLockerPolicy>
"@

# Appliquer les règles AppLocker
Set-AppLockerPolicy -XmlPolicy $appLockerRules -Merge
```

## Section 2 : Validation et sécurisation des entrées

### 2.1 Validation des paramètres

```powershell
function Get-SecureUserData {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [ValidatePattern('^[a-zA-Z][a-zA-Z0-9_-]{2,15}$')]
        [string]$UserName,
        
        [Parameter(Mandatory = $true)]
        [ValidateLength(8, 128)]
        [string]$Password,
        
        [Parameter(Mandatory = $false)]
        [ValidateSet('Admin', 'User', 'Guest')]
        [string]$Role = 'User',
        
        [Parameter(Mandatory = $false)]
        [ValidateRange(18, 120)]
        [int]$Age,
        
        [Parameter(Mandatory = $false)]
        [ValidateScript({
            if (-not (Test-Path $_)) { throw "Le chemin n'existe pas" }
            if (-not (Test-Path $_ -PathType Leaf)) { throw "Le chemin doit être un fichier" }
            $true
        })]
        [string]$ConfigFile,
        
        [Parameter(Mandatory = $false)]
        [ValidateNotNullOrEmpty()]
        [string]$Email
    )
    
    # Validation supplémentaire pour l'email
    if ($Email -and $Email -notmatch '^[\w\.-]+@[\w\.-]+\.\w+$') {
        throw "Format d'email invalide"
    }
    
    # Validation de la complexité du mot de passe
    if ($Password -notmatch '(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^\da-zA-Z]).{8,}') {
        throw "Le mot de passe doit contenir au moins 8 caractères avec majuscules, minuscules, chiffres et caractères spéciaux"
    }
    
    Write-Host "✓ Toutes les validations passées"
    
    [PSCustomObject]@{
        UserName = $UserName
        Role = $Role
        Age = $Age
        ConfigFile = $ConfigFile
        Email = $Email
        SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force
    }
}

# Utilisation avec validation automatique
$userData = Get-SecureUserData -UserName "john_doe" -Password "MySecurePass123!" -Email "john.doe@company.com"
```

### 2.2 Sanitisation des données d'entrée

```powershell
# Fonction de sanitisation générale
function Sanitize-Input {
    param(
        [Parameter(ValueFromPipeline = $true)]
        [string]$InputString,
        
        [switch]$AllowHtml,
        [switch]$AllowSql,
        [switch]$AllowScript
    )
    
    $sanitized = $InputString
    
    # Suppression des caractères de contrôle
    $sanitized = $sanitized -replace '[\x00-\x1F\x7F-\x9F]', ''
    
    # Sanitisation HTML si non autorisé
    if (-not $AllowHtml) {
        $sanitized = $sanitized -replace '<[^>]+>', ''
    }
    
    # Sanitisation SQL si non autorisé
    if (-not $AllowSql) {
        $sanitized = $sanitized -replace "([';]|\b(union|select|insert|delete|update|drop|create)\b)", ''
    }
    
    # Sanitisation des scripts si non autorisé
    if (-not $AllowScript) {
        # Supprimer les patterns dangereux
        $sanitized = $sanitized -replace '(\$\{.*?\})', ''
        $sanitized = $sanitized -replace '(Invoke-Expression|iex)', ''
        $sanitized = $sanitized -replace '(&\s*[\w]+)', ''
    }
    
    # Limitation de longueur
    if ($sanitized.Length -gt 1000) {
        $sanitized = $sanitized.Substring(0, 1000) + "..."
    }
    
    return $sanitized
}

# Tests de sanitisation
$suspiciousInput = "<script>alert('xss')</script> ; DROP TABLE users; `$(Invoke-Expression 'calc.exe')"
$sanitized = $suspiciousInput | Sanitize-Input
Write-Host "Original: $suspiciousInput"
Write-Host "Sanitized: $sanitized"
```

### 2.3 Gestion sécurisée des mots de passe

```powershell
# Classe pour la gestion sécurisée des credentials
class SecureCredentialManager {
    [string]$CredentialStorePath
    
    SecureCredentialManager([string]$storePath) {
        $this.CredentialStorePath = $storePath
        
        # Créer le répertoire de stockage si nécessaire
        $storeDir = Split-Path $storePath -Parent
        if (-not (Test-Path $storeDir)) {
            New-Item -ItemType Directory -Path $storeDir -Force | Out-Null
        }
    }
    
    [void] StoreCredential([string]$targetName, [PSCredential]$credential) {
        $credentialData = @{
            UserName = $credential.UserName
            Password = $credential.Password | ConvertFrom-SecureString
            TargetName = $targetName
            Timestamp = Get-Date
        }
        
        $jsonData = $credentialData | ConvertTo-Json
        $encryptedData = $this.EncryptData($jsonData)
        
        $encryptedData | Out-File -FilePath "$($this.CredentialStorePath)\$targetName.cred" -Encoding UTF8
        Write-Host "✓ Credential stocké pour $targetName"
    }
    
    [PSCredential] RetrieveCredential([string]$targetName) {
        $credFile = "$($this.CredentialStorePath)\$targetName.cred"
        
        if (-not (Test-Path $credFile)) {
            throw "Credential non trouvé: $targetName"
        }
        
        $encryptedData = Get-Content $credFile -Raw
        $jsonData = $this.DecryptData($encryptedData)
        $credentialData = $jsonData | ConvertFrom-Json
        
        $securePassword = $credentialData.Password | ConvertTo-SecureString
        return New-Object PSCredential($credentialData.UserName, $securePassword)
    }
    
    [string] EncryptData([string]$data) {
        # Chiffrement avec la clé de l'utilisateur actuel
        $secureString = ConvertTo-SecureString $data -AsPlainText -Force
        return $secureString | ConvertFrom-SecureString
    }
    
    [string] DecryptData([string]$encryptedData) {
        # Déchiffrement avec la clé de l'utilisateur actuel
        $secureString = $encryptedData | ConvertTo-SecureString
        $ptr = [System.Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureString)
        try {
            return [System.Runtime.InteropServices.Marshal]::PtrToStringBSTR($ptr)
        } finally {
            [System.Runtime.InteropServices.Marshal]::ZeroFreeBSTR($ptr)
        }
    }
    
    [void] RemoveCredential([string]$targetName) {
        $credFile = "$($this.CredentialStorePath)\$targetName.cred"
        
        if (Test-Path $credFile) {
            Remove-Item $credFile -Force
            Write-Host "✓ Credential supprimé: $targetName"
        } else {
            Write-Warning "Credential non trouvé: $targetName"
        }
    }
    
    [string[]] ListStoredCredentials() {
        $credFiles = Get-ChildItem "$($this.CredentialStorePath)\*.cred"
        return $credFiles.BaseName
    }
}

# Utilisation du gestionnaire de credentials
$credManager = [SecureCredentialManager]::new("$env:USERPROFILE\.credentials")

# Stocker des credentials
$adminCred = Get-Credential -UserName "Administrator" -Message "Entrez les credentials admin"
$credManager.StoreCredential("DomainAdmin", $adminCred)

# Récupérer des credentials
$storedCred = $credManager.RetrieveCredential("DomainAdmin")
Write-Host "Utilisateur stocké: $($storedCred.UserName)"

# Lister les credentials stockés
$credManager.ListStoredCredentials()
```

## Section 3 : Gestion des erreurs et logging sécurisé

### 3.1 Gestion d'erreurs sécurisée

```powershell
# Fonction avec gestion d'erreurs complète
function Invoke-SecureOperation {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory = $true)]
        [scriptblock]$Operation,
        
        [Parameter(Mandatory = $false)]
        [int]$MaxRetries = 3,
        
        [Parameter(Mandatory = $false)]
        [int]$RetryDelaySeconds = 5,
        
        [Parameter(Mandatory = $false)]
        [scriptblock]$OnError = { param($error) Write-Error "Erreur: $($error.Exception.Message)" }
    )
    
    $attempt = 0
    $lastError = $null
    
    do {
        $attempt++
        
        try {
            # Journaliser le début de l'opération
            Write-Verbose "Tentative $attempt/$MaxRetries"
            
            # Exécuter l'opération
            $result = & $Operation
            
            # Journaliser le succès
            Write-Verbose "Opération réussie à la tentative $attempt"
            
            return $result
            
        } catch {
            $lastError = $_
            
            # Journaliser l'erreur
            Write-Warning "Tentative $attempt échouée: $($_.Exception.Message)"
            
            # Exécuter le handler d'erreur personnalisé
            & $OnError $_
            
            # Attendre avant retry (sauf dernière tentative)
            if ($attempt -lt $MaxRetries) {
                Write-Verbose "Attente de $RetryDelaySeconds secondes avant retry..."
                Start-Sleep -Seconds $RetryDelaySeconds
            }
        }
        
    } while ($attempt -lt $MaxRetries)
    
    # Toutes les tentatives ont échoué
    throw "Opération échouée après $MaxRetries tentatives. Dernière erreur: $($lastError.Exception.Message)"
}

# Utilisation avec gestion d'erreurs
$result = Invoke-SecureOperation -Operation {
    # Opération potentiellement dangereuse
    if ((Get-Random -Maximum 10) -lt 7) {
        throw "Erreur simulée"
    }
    return "Opération réussie"
} -MaxRetries 5 -RetryDelaySeconds 2 -OnError {
    param($error)
    Write-Host "Gestion personnalisée de l'erreur: $($error.Exception.Message)" -ForegroundColor Yellow
}
```

### 3.2 Logging sécurisé et audit

```powershell
# Classe de logging sécurisé
class SecureLogger {
    [string]$LogPath
    [string]$LogLevel
    hidden [System.IO.StreamWriter]$LogWriter
    
    SecureLogger([string]$path, [string]$level = "Information") {
        $this.LogPath = $path
        $this.LogLevel = $level
        
        # Créer le répertoire si nécessaire
        $logDir = Split-Path $path -Parent
        if (-not (Test-Path $logDir)) {
            New-Item -ItemType Directory -Path $logDir -Force | Out-Null
        }
        
        # Ouvrir le fichier en mode append
        $this.LogWriter = New-Object System.IO.StreamWriter($path, $true)
        $this.LogWriter.AutoFlush = $true
        
        # Log de démarrage
        $this.Log("Logger initialized", "Information")
    }
    
    [void] Log([string]$message, [string]$level = "Information") {
        $levels = @("Debug", "Information", "Warning", "Error", "Critical")
        $currentLevelIndex = $levels.IndexOf($this.LogLevel)
        $messageLevelIndex = $levels.IndexOf($level)
        
        # Ne logger que si le niveau est suffisant
        if ($messageLevelIndex -lt $currentLevelIndex) {
            return
        }
        
        $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss.fff"
        $userName = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
        $computerName = $env:COMPUTERNAME
        
        $logEntry = "[$timestamp] [$level] [$computerName\$userName] $message"
        
        # Écrire dans le fichier
        $this.LogWriter.WriteLine($logEntry)
        
        # Écrire dans l'Event Log si critique
        if ($level -eq "Critical" -or $level -eq "Error") {
            Write-EventLog -LogName Application -Source "PowerShellScript" -EventId 1000 -EntryType Error -Message $logEntry
        }
    }
    
    [void] LogError([System.Management.Automation.ErrorRecord]$error) {
        $message = "Exception: $($error.Exception.Message)`nScript: $($error.InvocationInfo.ScriptName)`nLine: $($error.InvocationInfo.ScriptLineNumber)`nCommand: $($error.InvocationInfo.Line)"
        $this.Log($message, "Error")
    }
    
    [void] LogSecurityEvent([string]$action, [string]$target, [string]$result) {
        $message = "SECURITY: Action='$action', Target='$target', Result='$result'"
        $this.Log($message, "Information")
        
        # Log dans le Security Event Log
        Write-EventLog -LogName Security -Source "PowerShellScript" -EventId 1001 -EntryType Information -Message $message
    }
    
    [void] Dispose() {
        if ($this.LogWriter) {
            $this.Log("Logger disposed", "Information")
            $this.LogWriter.Dispose()
        }
    }
}

# Utilisation du logger sécurisé
$logger = [SecureLogger]::new("$env:TEMP\secure_script.log", "Debug")

try {
    $logger.Log("Script démarré", "Information")
    $logger.LogSecurityEvent("ScriptExecution", $MyInvocation.MyCommand.Name, "Started")
    
    # Opération potentiellement dangereuse
    $logger.Log("Tentative d'opération dangereuse", "Warning")
    
    # Simuler une erreur
    throw "Erreur de test"
    
} catch {
    $logger.LogError($_)
    $logger.LogSecurityEvent("ScriptExecution", $MyInvocation.MyCommand.Name, "Failed")
    
} finally {
    $logger.LogSecurityEvent("ScriptExecution", $MyInvocation.MyCommand.Name, "Completed")
    $logger.Dispose()
}
```

## Section 4 : Bonnes pratiques de sécurité

### 4.1 Principe du moindre privilège

```powershell
# Fonction pour exécuter du code avec des privilèges réduits
function Invoke-WithReducedPrivileges {
    param([scriptblock]$ScriptBlock)
    
    # Créer un nouveau processus PowerShell avec un utilisateur limité
    $startInfo = New-Object System.Diagnostics.ProcessStartInfo
    $startInfo.FileName = "powershell.exe"
    $startInfo.Arguments = "-NoProfile -Command `"& { $ScriptBlock }`""
    $startInfo.UseShellExecute = $false
    $startInfo.RedirectStandardOutput = $true
    $startInfo.RedirectStandardError = $true
    
    # Configurer l'utilisateur limité (nécessite configuration préalable)
    $startInfo.UserName = "LimitedUser"
    $securePassword = ConvertTo-SecureString "LimitedPass" -AsPlainText -Force
    $startInfo.Password = $securePassword
    
    $process = New-Object System.Diagnostics.Process
    $process.StartInfo = $startInfo
    
    $process.Start() | Out-Null
    $output = $process.StandardOutput.ReadToEnd()
    $errorOutput = $process.StandardError.ReadToEnd()
    $process.WaitForExit()
    
    if ($process.ExitCode -ne 0) {
        Write-Error "Erreur dans l'exécution avec privilèges réduits: $errorOutput"
    }
    
    return $output
}

# Vérification des privilèges avant exécution
function Assert-RequiredPrivileges {
    param([string[]]$RequiredRoles)
    
    $currentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent()
    $principal = New-Object System.Security.Principal.WindowsPrincipal($currentUser)
    
    foreach ($role in $RequiredRoles) {
        if (-not $principal.IsInRole([System.Security.Principal.WindowsBuiltInRole]::$role)) {
            throw "Privilèges insuffisants. Rôle requis: $role"
        }
    }
    
    Write-Host "✓ Privilèges vérifiés pour les rôles: $($RequiredRoles -join ', ')"
}

# Utilisation
Assert-RequiredPrivileges -RequiredRoles "Administrator"
```

### 4.2 Audit et conformité

```powershell
# Fonction d'audit de sécurité PowerShell
function Invoke-PowerShellSecurityAudit {
    param([string]$OutputPath = ".\SecurityAudit_$(Get-Date -Format 'yyyyMMdd_HHmmss').json")
    
    $audit = [PSCustomObject]@{
        Timestamp = Get-Date
        ComputerName = $env:COMPUTERNAME
        UserName = $env:USERNAME
        
        ExecutionPolicy = @{
            CurrentUser = Get-ExecutionPolicy -Scope CurrentUser
            LocalMachine = Get-ExecutionPolicy -Scope LocalMachine
        }
        
        PowerShellVersion = $PSVersionTable.PSVersion
        CLRVersion = $PSVersionTable.CLRVersion
        
        Modules = Get-InstalledModule | Select-Object Name, Version, Repository
        Scripts = Get-ChildItem -Path $env:USERPROFILE -Filter "*.ps1" -Recurse -ErrorAction SilentlyContinue | 
            ForEach-Object {
                [PSCustomObject]@{
                    Path = $_.FullName
                    Signature = (Get-AuthenticodeSignature $_.FullName).Status
                    LastModified = $_.LastWriteTime
                    Size = $_.Length
                }
            }
        
        SecuritySettings = @{
            UACEnabled = (Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System).EnableLUA
            RemoteDesktopEnabled = (Get-ItemProperty HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server).fDenyTSConnections -eq 0
            FirewallEnabled = (Get-NetFirewallProfile | Where-Object Name -eq "Domain").Enabled
        }
        
        RecentSecurityEvents = Get-WinEvent -LogName Security -MaxEvents 10 | 
            Select-Object TimeCreated, Id, Message | 
            Where-Object { $_.Id -in @(4625, 4634, 4648, 4672) } # Échecs/connexions privilégiées
    }
    
    # Tests de sécurité supplémentaires
    $audit | Add-Member -MemberType NoteProperty -Name "SecurityTests" -Value @{
        CanExecuteUnsignedScripts = $false
        HasWeakPasswords = $false
        HasExpiredCertificates = $false
        RemoteManagementEnabled = $false
    }
    
    # Test d'exécution de scripts non signés
    try {
        Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope Process -ErrorAction Stop
        $audit.SecurityTests.CanExecuteUnsignedScripts = $true
    } catch {
        $audit.SecurityTests.CanExecuteUnsignedScripts = $false
    } finally {
        Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope Process -ErrorAction SilentlyContinue
    }
    
    # Vérifier WinRM
    try {
        $winrmConfig = Get-WSManInstance -ResourceURI winrm/config/service
        $audit.SecurityTests.RemoteManagementEnabled = $winrmConfig.AllowRemoteAccess
    } catch {
        $audit.SecurityTests.RemoteManagementEnabled = $false
    }
    
    # Sauvegarder l'audit
    $audit | ConvertTo-Json -Depth 5 | Out-File -FilePath $OutputPath -Encoding UTF8
    
    Write-Host "✓ Audit de sécurité terminé: $OutputPath"
    
    return $audit
}

# Exécuter l'audit
$auditResult = Invoke-PowerShellSecurityAudit
```

### 4.3 Recommandations de sécurité

```powershell
# Fonction de recommandations de sécurité
function Get-PowerShellSecurityRecommendations {
    $recommendations = @()
    
    # Vérifier la politique d'exécution
    $execPolicy = Get-ExecutionPolicy
    if ($execPolicy -eq "Unrestricted" -or $execPolicy -eq "Bypass") {
        $recommendations += [PSCustomObject]@{
            Category = "ExecutionPolicy"
            Severity = "High"
            Recommendation = "Changez la politique d'exécution vers RemoteSigned ou AllSigned"
            CurrentValue = $execPolicy
            Remediation = "Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope LocalMachine"
        }
    }
    
    # Vérifier la version PowerShell
    $psVersion = $PSVersionTable.PSVersion
    if ($psVersion.Major -lt 5) {
        $recommendations += [PSCustomObject]@{
            Category = "Version"
            Severity = "Medium"
            Recommendation = "Mettez à jour vers PowerShell 5.1 ou PowerShell Core 7+"
            CurrentValue = $psVersion.ToString()
            Remediation = "Téléchargez depuis https://github.com/PowerShell/PowerShell"
        }
    }
    
    # Vérifier l'UAC
    $uacEnabled = (Get-ItemProperty HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System).EnableLUA
    if (-not $uacEnabled) {
        $recommendations += [PSCustomObject]@{
            Category = "UAC"
            Severity = "High"
            Recommendation = "Activez le contrôle de compte d'utilisateur (UAC)"
            CurrentValue = $uacEnabled
            Remediation = "Activez l'UAC dans les paramètres de sécurité"
        }
    }
    
    # Vérifier le firewall
    $firewallEnabled = (Get-NetFirewallProfile | Where-Object Name -eq "Domain").Enabled
    if (-not $firewallEnabled) {
        $recommendations += [PSCustomObject]@{
            Category = "Firewall"
            Severity = "Critical"
            Recommendation = "Activez le firewall Windows"
            CurrentValue = $firewallEnabled
            Remediation = "Set-NetFirewallProfile -Profile Domain -Enabled True"
        }
    }
    
    # Vérifier les mises à jour
    $lastUpdate = (Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 1).InstalledOn
    $daysSinceUpdate = (Get-Date) - $lastUpdate
    if ($daysSinceUpdate.Days -gt 30) {
        $recommendations += [PSCustomObject]@{
            Category = "Updates"
            Severity = "High"
            Recommendation = "Installez les mises à jour de sécurité récentes"
            CurrentValue = "$($daysSinceUpdate.Days) jours"
            Remediation = "Exécutez Windows Update"
        }
    }
    
    # Retourner les recommandations triées par sévérité
    $severityOrder = @{"Critical" = 1; "High" = 2; "Medium" = 3; "Low" = 4}
    $recommendations | Sort-Object { $severityOrder[$_.Severity] }
}

# Afficher les recommandations
$securityRecs = Get-PowerShellSecurityRecommendations
$securityRecs | Format-Table -AutoSize -Wrap
```

## Conclusion : La sécurité comme fondement

La sécurité dans PowerShell n'est pas une fonctionnalité optionnelle - c'est une philosophie qui imprègne chaque aspect du développement. Des politiques d'exécution à la validation des entrées, en passant par l'audit et les meilleures pratiques, PowerShell offre les outils nécessaires pour créer des solutions non seulement puissantes, mais aussi intrinsèquement sécurisées.

Dans le prochain chapitre, nous explorerons les tendances futures et l'évolution de PowerShell, en nous projetant dans l'avenir du shell scripting.

---

**Exercice pratique :** Créez un framework de sécurité PowerShell complet qui inclut :
1. Validation automatique des scripts et signatures
2. Gestion sécurisée des credentials
3. Logging d'audit complet
4. Tests de sécurité automatisés
5. Recommandations de hardening

**Challenge avancé :** Développez un module de conformité qui :
- Audit automatiquement les configurations PowerShell
- Applique des politiques de sécurité
- Génère des rapports de conformité
- S'intègre avec des outils SIEM
- Implémente des contrôles de sécurité en temps réel

**Réflexion :** Comment la sécurité intégrée de PowerShell change-t-elle notre approche du développement de scripts par rapport à d'autres langages de scripting ?

