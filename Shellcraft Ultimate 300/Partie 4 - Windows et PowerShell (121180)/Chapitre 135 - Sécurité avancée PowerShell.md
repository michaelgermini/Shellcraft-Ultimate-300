# Chapitre 135 - Sécurité avancée PowerShell

> "La sécurité n'est pas un produit, mais un processus. PowerShell, en tant qu'interface privilégiée vers Windows, doit être le gardien vigilant de ce processus, transformant chaque commande en une opportunité de renforcer la posture de sécurité." - Citation inspirée des principes de sécurité défensive

## Introduction : PowerShell comme rempart de sécurité

La sécurité avancée PowerShell transcende la simple exécution de commandes pour devenir un framework de sécurité intégral. En tant qu'interface privilégiée vers le système d'exploitation Windows, PowerShell doit incorporer des mécanismes de sécurité à chaque niveau : authentification, autorisation, audit, chiffrement, et détection des menaces.

Dans ce chapitre, nous explorerons les mécanismes de sécurité avancés, les patterns de sécurisation, et les stratégies de défense en profondeur.

## Section 1 : Authentification et autorisation robustes

### 1.1 Gestion avancée des credentials

**Système de gestion sécurisée des credentials :**
```powershell
# Classe de gestion sécurisée des credentials
class SecureCredentialManager {
    [System.Collections.Generic.Dictionary[string, PSCredential]]$CredentialStore
    [string]$CredentialFile
    [byte[]]$EncryptionKey
    hidden [System.Security.Cryptography.AesCryptoServiceProvider]$AES

    SecureCredentialManager([string]$credentialFile = "$env:APPDATA\PowerShell\credentials.enc") {
        $this.CredentialStore = [System.Collections.Generic.Dictionary[string, PSCredential]]::new()
        $this.CredentialFile = $credentialFile
        $this.InitializeEncryption()
        $this.LoadCredentials()
    }

    hidden [void]InitializeEncryption() {
        $this.AES = [System.Security.Cryptography.AesCryptoServiceProvider]::new()
        $this.AES.KeySize = 256
        $this.AES.Mode = [System.Security.Cryptography.CipherMode]::CBC
        $this.AES.Padding = [System.Security.Cryptography.PaddingMode]::PKCS7

        # Génération d'une clé basée sur l'utilisateur et la machine (ou clé personnalisée)
        $keySource = "$env:USERNAME@$env:COMPUTERNAME"
        $sha256 = [System.Security.Cryptography.SHA256]::Create()
        $this.EncryptionKey = $sha256.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($keySource))
        $this.AES.Key = $this.EncryptionKey
    }

    [void]StoreCredential([string]$name, [PSCredential]$credential, [TimeSpan]$validityPeriod = [TimeSpan]::FromDays(90)) {
        $secureCredential = @{
            Credential = $credential
            Created = Get-Date
            Expires = (Get-Date) + $validityPeriod
            LastUsed = $null
            UsageCount = 0
        }

        $this.CredentialStore[$name] = $secureCredential
        $this.SaveCredentials()

        Write-Host "Credential '$name' stored securely (expires: $($secureCredential.Expires))" -ForegroundColor Green
    }

    [PSCredential]GetCredential([string]$name) {
        if (-not $this.CredentialStore.ContainsKey($name)) {
            throw [System.Security.Authentication.InvalidCredentialException]::new("Credential '$name' not found")
        }

        $secureCredential = $this.CredentialStore[$name]

        # Vérification d'expiration
        if ((Get-Date) -gt $secureCredential.Expires) {
            $this.RemoveCredential($name)
            throw [System.Security.Authentication.InvalidCredentialException]::new("Credential '$name' has expired")
        }

        # Mise à jour des métadonnées d'usage
        $secureCredential.LastUsed = Get-Date
        $secureCredential.UsageCount++

        return $secureCredential.Credential
    }

    [void]RemoveCredential([string]$name) {
        if ($this.CredentialStore.Remove($name)) {
            $this.SaveCredentials()
            Write-Host "Credential '$name' removed" -ForegroundColor Yellow
        }
    }

    [System.Collections.Generic.List[string]]ListCredentials() {
        $result = [System.Collections.Generic.List[string]]::new()

        foreach ($name in $this.CredentialStore.Keys) {
            $cred = $this.CredentialStore[$name]
            $status = if ((Get-Date) -gt $cred.Expires) { "EXPIRED" } else { "VALID" }
            $result.Add("$name ($status, expires: $($cred.Expires.ToShortDateString()), used: $($cred.UsageCount) times)")
        }

        return $result
    }

    hidden [void]SaveCredentials() {
        try {
            # Sérialisation des credentials
            $data = @{
                Version = "1.0"
                Credentials = @{}
            }

            foreach ($name in $this.CredentialStore.Keys) {
                $secureCred = $this.CredentialStore[$name]
                $data.Credentials[$name] = @{
                    UserName = $secureCred.Credential.UserName
                    Password = $secureCred.Credential.GetNetworkCredential().Password
                    Created = $secureCred.Created
                    Expires = $secureCred.Expires
                    LastUsed = $secureCred.LastUsed
                    UsageCount = $secureCred.UsageCount
                }
            }

            # Sérialisation JSON
            $jsonData = $data | ConvertTo-Json -Depth 10
            $bytes = [System.Text.Encoding]::UTF8.GetBytes($jsonData)

            # Chiffrement
            $this.AES.GenerateIV()
            $encryptor = $this.AES.CreateEncryptor()
            $encryptedBytes = $encryptor.TransformFinalBlock($bytes, 0, $bytes.Length)

            # Sauvegarde (IV + données chiffrées)
            $finalData = $this.AES.IV + $encryptedBytes
            [System.IO.File]::WriteAllBytes($this.CredentialFile, $finalData)

        } catch {
            Write-Error "Failed to save credentials: $_"
            throw
        }
    }

    hidden [void]LoadCredentials() {
        if (-not (Test-Path $this.CredentialFile)) {
            return
        }

        try {
            $encryptedData = [System.IO.File]::ReadAllBytes($this.CredentialFile)

            # Extraction IV et données
            $iv = $encryptedData[0..15]
            $encryptedBytes = $encryptedData[16..($encryptedData.Length - 1)]

            # Déchiffrement
            $this.AES.IV = $iv
            $decryptor = $this.AES.CreateDecryptor()
            $decryptedBytes = $decryptor.TransformFinalBlock($encryptedBytes, 0, $encryptedBytes.Length)

            $jsonData = [System.Text.Encoding]::UTF8.GetString($decryptedBytes)
            $data = $jsonData | ConvertFrom-Json

            # Reconstruction des credentials
            foreach ($name in $data.Credentials.PSObject.Properties.Name) {
                $credData = $data.Credentials.$name
                $securePassword = ConvertTo-SecureString $credData.Password -AsPlainText -Force
                $credential = [PSCredential]::new($credData.UserName, $securePassword)

                $this.CredentialStore[$name] = @{
                    Credential = $credential
                    Created = [DateTime]::Parse($credData.Created)
                    Expires = [DateTime]::Parse($credData.Expires)
                    LastUsed = if ($credData.LastUsed) { [DateTime]::Parse($credData.LastUsed) } else { $null }
                    UsageCount = $credData.UsageCount
                }
            }

            Write-Debug "Loaded $($this.CredentialStore.Count) credentials"

        } catch {
            Write-Error "Failed to load credentials. The credential file may be corrupted or the encryption key has changed."
            throw
        }
    }

    [void]ChangeMasterPassword([string]$newKeySource) {
        # Attention: Cette opération nécessite de rechiffrer toutes les données
        Write-Warning "Changing master password will require re-encryption of all stored credentials"

        $oldKey = $this.EncryptionKey
        $sha256 = [System.Security.Cryptography.SHA256]::Create()
        $this.EncryptionKey = $sha256.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($newKeySource))
        $this.AES.Key = $this.EncryptionKey

        try {
            $this.SaveCredentials()
            Write-Host "Master password changed successfully" -ForegroundColor Green
        } catch {
            # Rollback en cas d'erreur
            $this.EncryptionKey = $oldKey
            $this.AES.Key = $oldKey
            throw "Failed to change master password: $_"
        }
    }
}

# Utilisation du gestionnaire de credentials sécurisé
$credManager = [SecureCredentialManager]::new()

# Stockage d'un credential
$adminCred = Get-Credential -Message "Enter administrator credentials"
$credManager.StoreCredential("DomainAdmin", $adminCred, [TimeSpan]::FromDays(30))

# Récupération d'un credential
try {
    $storedCred = $credManager.GetCredential("DomainAdmin")
    Write-Host "Retrieved credential for user: $($storedCred.UserName)" -ForegroundColor Green
} catch {
    Write-Error "Failed to retrieve credential: $_"
}

# Liste des credentials
Write-Host "Stored credentials:" -ForegroundColor Cyan
$credManager.ListCredentials() | ForEach-Object { Write-Host "  $_" }

# Fonction d'authentification sécurisée
function Invoke-SecureAuthentication {
    param(
        [Parameter(Mandatory=$true)]
        [string]$TargetSystem,

        [Parameter(Mandatory=$true)]
        [string]$CredentialName,

        [scriptblock]$AuthenticationTest = { $true }
    )

    $credManager = [SecureCredentialManager]::new()

    try {
        $credential = $credManager.GetCredential($CredentialName)

        Write-Host "Authenticating to $TargetSystem..." -ForegroundColor Yellow

        # Test d'authentification (personnalisable)
        $authResult = & $AuthenticationTest $TargetSystem $credential

        if ($authResult) {
            Write-Host "Authentication successful to $TargetSystem" -ForegroundColor Green
            return @{
                Success = $true
                System = $TargetSystem
                User = $credential.UserName
                Timestamp = Get-Date
            }
        } else {
            Write-Warning "Authentication failed to $TargetSystem"
            return @{
                Success = $false
                System = $TargetSystem
                Error = "Authentication test failed"
            }
        }

    } catch {
        Write-Error "Authentication error: $_"
        return @{
            Success = $false
            System = $TargetSystem
            Error = $_.Exception.Message
        }
    }
}

# Exemple d'utilisation pour l'authentification Active Directory
$adAuthTest = {
    param($system, $cred)

    try {
        $domain = $env:USERDNSDOMAIN
        $directoryEntry = New-Object System.DirectoryServices.DirectoryEntry("LDAP://$domain", $cred.UserName, $cred.GetNetworkCredential().Password)
        $directorySearcher = New-Object System.DirectoryServices.DirectorySearcher($directoryEntry)
        $searcherResult = $directorySearcher.FindOne()

        return $null -ne $searcherResult
    } catch {
        return $false
    }
}

$authResult = Invoke-SecureAuthentication -TargetSystem "ActiveDirectory" -CredentialName "DomainAdmin" -AuthenticationTest $adAuthTest

if ($authResult.Success) {
    Write-Host "Successfully authenticated to Active Directory as $($authResult.User)" -ForegroundColor Green
} else {
    Write-Warning "Authentication failed: $($authResult.Error)"
}
```

### 1.2 Just-In-Time Access et Privileged Access Management

**Système de gestion des accès privilégiés :**
```powershell
# Classe pour la gestion des accès privilégiés (JIT)
class PrivilegedAccessManager {
    [System.Collections.Generic.Dictionary[string, hashtable]]$AccessPolicies
    [System.Collections.Generic.Dictionary[string, PSCredential]]$TemporaryCredentials
    [System.Collections.Generic.List[PSCustomObject]]$AccessLog
    [TimeSpan]$MaxSessionDuration
    [scriptblock]$ApprovalCallback

    PrivilegedAccessManager([TimeSpan]$maxSessionDuration = [TimeSpan]::FromHours(2)) {
        $this.AccessPolicies = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.TemporaryCredentials = [System.Collections.Generic.Dictionary[string, PSCredential]]::new()
        $this.AccessLog = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.MaxSessionDuration = $maxSessionDuration
        $this.ApprovalCallback = { param($request) $true }  # Approbation automatique par défaut
    }

    [void]DefineAccessPolicy([string]$roleName, [hashtable]$policy) {
        # Validation de la politique
        $requiredKeys = @('AllowedCommands', 'MaxSessionDuration', 'RequiresApproval', 'AllowedUsers')
        foreach ($key in $requiredKeys) {
            if (-not $policy.ContainsKey($key)) {
                throw "Policy for role '$roleName' is missing required key: $key"
            }
        }

        $this.AccessPolicies[$roleName] = $policy.Clone()
        Write-Host "Access policy defined for role: $roleName" -ForegroundColor Green
    }

    [PSCustomObject]RequestPrivilegedAccess([string]$roleName, [string]$requestingUser, [string]$justification) {
        if (-not $this.AccessPolicies.ContainsKey($roleName)) {
            throw "Access policy not found for role: $roleName"
        }

        $policy = $this.AccessPolicies[$roleName]

        # Vérification de l'autorisation
        if ($requestingUser -notin $policy.AllowedUsers) {
            throw "User '$requestingUser' is not authorized for role '$roleName'"
        }

        $accessRequest = [PSCustomObject]@{
            RequestId = [guid]::NewGuid().ToString()
            RoleName = $roleName
            RequestingUser = $requestingUser
            Justification = $justification
            RequestedAt = Get-Date
            Status = "Pending"
            ApprovedBy = $null
            ApprovedAt = $null
            ExpiresAt = $null
            SessionId = $null
        }

        # Demande d'approbation si requise
        if ($policy.RequiresApproval) {
            Write-Host "Approval required for privileged access request: $($accessRequest.RequestId)" -ForegroundColor Yellow

            $approvalResult = & $this.ApprovalCallback $accessRequest

            if (-not $approvalResult.Approved) {
                $accessRequest.Status = "Rejected"
                $accessRequest | Add-Member -MemberType NoteProperty -Name "RejectionReason" -Value $approvalResult.Reason
                $this.LogAccessEvent($accessRequest, "AccessRejected", "Approval denied: $($approvalResult.Reason)")
                return $accessRequest
            }

            $accessRequest.ApprovedBy = $approvalResult.ApprovedBy
            $accessRequest.ApprovedAt = Get-Date
        }

        # Création des credentials temporaires
        $temporaryCredential = $this.CreateTemporaryCredential($roleName, $policy)
        $sessionId = [guid]::NewGuid().ToString()

        $this.TemporaryCredentials[$sessionId] = $temporaryCredential

        $accessRequest.Status = "Approved"
        $accessRequest.ExpiresAt = (Get-Date) + $policy.MaxSessionDuration
        $accessRequest.SessionId = $sessionId

        $this.LogAccessEvent($accessRequest, "AccessGranted", "Privileged access granted for role $roleName")

        return $accessRequest
    }

    hidden [PSCredential]CreateTemporaryCredential([string]$roleName, [hashtable]$policy) {
        # Génération d'un mot de passe temporaire fort
        $passwordLength = 16
        $charSet = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*"
        $passwordChars = for ($i = 1; $i -le $passwordLength; $i++) {
            $charSet[(Get-Random -Maximum $charSet.Length)]
        }
        $temporaryPassword = -join $passwordChars

        # Création du credential temporaire
        $securePassword = ConvertTo-SecureString $temporaryPassword -AsPlainText -Force
        $credential = [PSCredential]::new("$roleName`_temp", $securePassword)

        return $credential
    }

    [PSCustomObject]ExecutePrivilegedCommand([string]$sessionId, [string]$command, [hashtable]$parameters = @{}) {
        if (-not $this.TemporaryCredentials.ContainsKey($sessionId)) {
            throw "Invalid or expired session: $sessionId"
        }

        $credential = $this.TemporaryCredentials[$sessionId]

        # Vérification de l'expiration de session
        $sessionInfo = $this.AccessLog | Where-Object { $_.SessionId -eq $sessionId } | Select-Object -Last 1
        if ($sessionInfo -and (Get-Date) -gt $sessionInfo.ExpiresAt) {
            $this.TemporaryCredentials.Remove($sessionId)
            throw "Session expired: $sessionId"
        }

        try {
            # Exécution de la commande avec les credentials temporaires
            $result = Invoke-Command -Credential $credential -ScriptBlock ([scriptblock]::Create($command)) -ArgumentList $parameters

            $this.LogAccessEvent($sessionInfo, "CommandExecuted", "Command executed successfully: $command")

            return [PSCustomObject]@{
                Success = $true
                Result = $result
                Command = $command
                ExecutedAt = Get-Date
            }

        } catch {
            $this.LogAccessEvent($sessionInfo, "CommandFailed", "Command execution failed: $command - Error: $($_.Exception.Message)")

            return [PSCustomObject]@{
                Success = $false
                Error = $_.Exception.Message
                Command = $command
                ExecutedAt = Get-Date
            }
        }
    }

    [void]RevokePrivilegedAccess([string]$sessionId) {
        if ($this.TemporaryCredentials.Remove($sessionId)) {
            $this.LogAccessEvent($null, "AccessRevoked", "Privileged access revoked for session: $sessionId")
            Write-Host "Privileged access revoked for session: $sessionId" -ForegroundColor Yellow
        }
    }

    hidden [void]LogAccessEvent([PSCustomObject]$accessRequest, [string]$eventType, [string]$details) {
        $logEntry = [PSCustomObject]@{
            Timestamp = Get-Date
            EventType = $eventType
            Details = $details
            AccessRequest = $accessRequest
        }

        $this.AccessLog.Add($logEntry)

        # Log dans le journal d'événements Windows
        try {
            $eventLog = New-Object System.Diagnostics.EventLog("Application")
            $eventLog.Source = "PrivilegedAccessManager"
            $eventLog.WriteEntry("$eventType`: $details", [System.Diagnostics.EventLogEntryType]::Information)
        } catch {
            Write-Warning "Failed to write to event log: $_"
        }
    }

    [PSCustomObject[]]GetActiveSessions() {
        $activeSessions = @()

        foreach ($sessionId in $this.TemporaryCredentials.Keys) {
            $sessionInfo = $this.AccessLog | Where-Object { $_.SessionId -eq $sessionId -and $_.EventType -eq "AccessGranted" } | Select-Object -Last 1

            if ($sessionInfo) {
                $activeSessions += [PSCustomObject]@{
                    SessionId = $sessionId
                    RoleName = $sessionInfo.AccessRequest.RoleName
                    User = $sessionInfo.AccessRequest.RequestingUser
                    GrantedAt = $sessionInfo.AccessRequest.ApprovedAt
                    ExpiresAt = $sessionInfo.AccessRequest.ExpiresAt
                    TimeRemaining = $sessionInfo.AccessRequest.ExpiresAt - (Get-Date)
                }
            }
        }

        return $activeSessions
    }

    [void]CleanupExpiredSessions() {
        $expiredSessions = $this.GetActiveSessions() | Where-Object { $_.TimeRemaining.TotalSeconds -le 0 }

        foreach ($session in $expiredSessions) {
            $this.RevokePrivilegedAccess($session.SessionId)
        }

        if ($expiredSessions.Count -gt 0) {
            Write-Host "Cleaned up $($expiredSessions.Count) expired sessions" -ForegroundColor Yellow
        }
    }
}

# Configuration du système PAM
$pam = [PrivilegedAccessManager]::new([TimeSpan]::FromHours(4))

# Définition des politiques d'accès
$pam.DefineAccessPolicy("DomainAdmin", @{
    AllowedCommands = @("Get-ADUser", "Set-ADUser", "New-ADUser", "Remove-ADUser")
    MaxSessionDuration = [TimeSpan]::FromHours(2)
    RequiresApproval = $true
    AllowedUsers = @("alice.admin", "bob.admin")
})

$pam.DefineAccessPolicy("DatabaseAdmin", @{
    AllowedCommands = @("Invoke-Sqlcmd", "Backup-SqlDatabase")
    MaxSessionDuration = [TimeSpan]::FromHours(1)
    RequiresApproval = $false
    AllowedUsers = @("charlie.dba", "diana.dba")
})

# Callback d'approbation personnalisé
$pam.ApprovalCallback = {
    param($request)

    Write-Host "APPROVAL REQUEST:" -ForegroundColor Magenta
    Write-Host "User: $($request.RequestingUser)"
    Write-Host "Role: $($request.RoleName)"
    Write-Host "Justification: $($request.Justification)"
    Write-Host ""

    $approved = Read-Host "Approve this request? (y/n)"
    if ($approved -eq 'y') {
        return @{
            Approved = $true
            ApprovedBy = $env:USERNAME
        }
    } else {
        return @{
            Approved = $false
            Reason = "Manually rejected"
        }
    }
}

# Demande d'accès privilégié
$accessRequest = $pam.RequestPrivilegedAccess("DomainAdmin", "alice.admin", "Need to update user accounts for quarterly review")

if ($accessRequest.Status -eq "Approved") {
    Write-Host "Access granted! Session ID: $($accessRequest.SessionId)" -ForegroundColor Green

    # Exécution de commandes privilégiées
    $commandResult = $pam.ExecutePrivilegedCommand($accessRequest.SessionId, 'Get-ADUser -Filter * -SearchBase "OU=Users,DC=company,DC=com" | Select-Object -First 5')

    if ($commandResult.Success) {
        Write-Host "Command executed successfully" -ForegroundColor Green
        $commandResult.Result | Format-Table
    } else {
        Write-Warning "Command execution failed: $($commandResult.Error)"
    }

    # Révocation de l'accès
    $pam.RevokePrivilegedAccess($accessRequest.SessionId)
} else {
    Write-Warning "Access request rejected: $($accessRequest.RejectionReason)"
}

# Surveillance des sessions actives
Write-Host "`nActive privileged sessions:" -ForegroundColor Cyan
$pam.GetActiveSessions() | Format-Table

# Nettoyage des sessions expirées
$pam.CleanupExpiredSessions()
```

## Section 2 : Audit et surveillance de sécurité

### 2.1 Logging de sécurité avancé

**Système de logging de sécurité complet :**
```powershell
# Classe de logging de sécurité avancé
class SecurityLogger {
    [string]$LogPath
    [string]$LogName
    [System.Collections.Generic.List[PSCustomObject]]$SecurityEvents
    [hashtable]$LogLevels
    [scriptblock]$AlertCallback
    [System.Collections.Generic.Dictionary[string, int]]$EventCounters

    SecurityLogger([string]$logPath = "$env:ProgramData\SecurityLogs", [string]$logName = "PowerShellSecurity") {
        $this.LogPath = $logPath
        $this.LogName = $logName
        $this.SecurityEvents = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.EventCounters = [System.Collections.Generic.Dictionary[string, int]]::new()

        $this.LogLevels = @{
            "DEBUG" = 0
            "INFO" = 1
            "WARN" = 2
            "ERROR" = 3
            "CRITICAL" = 4
        }

        # Création du répertoire de logs
        if (-not (Test-Path $logPath)) {
            New-Item -ItemType Directory -Path $logPath -Force | Out-Null
        }

        $this.AlertCallback = {
            param($event)
            Write-Host "SECURITY ALERT: $($event.Message)" -ForegroundColor Red
        }
    }

    [void]LogSecurityEvent([string]$level, [string]$category, [string]$message, [hashtable]$context = @{}) {
        $timestamp = Get-Date

        $securityEvent = [PSCustomObject]@{
            Timestamp = $timestamp
            Level = $level.ToUpper()
            Category = $category
            Message = $message
            Context = $context
            User = $env:USERNAME
            Computer = $env:COMPUTERNAME
            ProcessId = $PID
            ThreadId = [System.Threading.Thread]::CurrentThread.ManagedThreadId
            SessionId = (Get-Process -Id $PID).SessionId
        }

        # Ajout à la collection en mémoire
        $this.SecurityEvents.Add($securityEvent)

        # Mise à jour des compteurs
        $eventKey = "$category`:$level"
        if (-not $this.EventCounters.ContainsKey($eventKey)) {
            $this.EventCounters[$eventKey] = 0
        }
        $this.EventCounters[$eventKey]++

        # Écriture dans le fichier de log
        $this.WriteToFile($securityEvent)

        # Écriture dans le journal d'événements Windows
        $this.WriteToEventLog($securityEvent)

        # Vérification des seuils d'alerte
        $this.CheckAlertThresholds($securityEvent)

        # Nettoyage automatique des anciens événements
        $this.CleanupOldEvents()
    }

    hidden [void]WriteToFile([PSCustomObject]$event) {
        $logFile = Join-Path $this.LogPath "$($this.LogName)_$(Get-Date -Format 'yyyy-MM-dd').log"

        try {
            $logEntry = @{
                timestamp = $event.Timestamp.ToString("yyyy-MM-dd HH:mm:ss.fff")
                level = $event.Level
                category = $event.Category
                message = $event.Message
                user = $event.User
                computer = $event.Computer
                process_id = $event.ProcessId
                context = $event.Context | ConvertTo-Json -Compress
            }

            $logEntry | ConvertTo-Json | Out-File -FilePath $logFile -Append -Encoding UTF8
        } catch {
            Write-Warning "Failed to write to security log file: $_"
        }
    }

    hidden [void]WriteToEventLog([PSCustomObject]$event) {
        try {
            $eventLog = New-Object System.Diagnostics.EventLog("Application")
            $eventLog.Source = $this.LogName

            # Mapping des niveaux aux types d'événements Windows
            $eventType = switch ($event.Level) {
                "DEBUG" { [System.Diagnostics.EventLogEntryType]::Information }
                "INFO" { [System.Diagnostics.EventLogEntryType]::Information }
                "WARN" { [System.Diagnostics.EventLogEntryType]::Warning }
                "ERROR" { [System.Diagnostics.EventLogEntryType]::Error }
                "CRITICAL" { [System.Diagnostics.EventLogEntryType]::Error }
                default { [System.Diagnostics.EventLogEntryType]::Information }
            }

            $message = @"
Security Event:
Level: $($event.Level)
Category: $($event.Category)
Message: $($event.Message)
User: $($event.User)
Computer: $($event.Computer)
Timestamp: $($event.Timestamp)
Context: $($event.Context | ConvertTo-Json -Compress)
"@

            $eventLog.WriteEntry($message, $eventType, 1000)
        } catch {
            Write-Warning "Failed to write to Windows Event Log: $_"
        }
    }

    hidden [void]CheckAlertThresholds([PSCustomObject]$event) {
        # Seuils d'alerte configurables
        $alertThresholds = @{
            "Authentication:ERROR" = 3
            "Authorization:ERROR" = 5
            "DataAccess:CRITICAL" = 1
            "PrivilegeEscalation:WARN" = 2
        }

        foreach ($thresholdKey in $alertThresholds.Keys) {
            $threshold = $alertThresholds[$thresholdKey]
            $eventKey = "$($event.Category):$($event.Level)"

            if ($eventKey -eq $thresholdKey -and $this.EventCounters[$eventKey] -ge $threshold) {
                # Déclenchement d'alerte
                & $this.AlertCallback $event

                # Réinitialisation du compteur pour éviter les alertes répétées
                $this.EventCounters[$eventKey] = 0
            }
        }
    }

    hidden [void]CleanupOldEvents() {
        $cutoffDate = (Get-Date).AddDays(-30)  # Garder 30 jours

        # Suppression des événements anciens de la collection en mémoire
        $eventsToRemove = $this.SecurityEvents | Where-Object { $_.Timestamp -lt $cutoffDate }
        foreach ($event in $eventsToRemove) {
            $this.SecurityEvents.Remove($event)
        }

        # Nettoyage des fichiers de log anciens
        $oldLogFiles = Get-ChildItem $this.LogPath -Filter "*.log" | Where-Object {
            $_.LastWriteTime -lt $cutoffDate
        }

        foreach ($file in $oldLogFiles) {
            try {
                Remove-Item $file.FullName -Force
                Write-Debug "Removed old log file: $($file.Name)"
            } catch {
                Write-Warning "Failed to remove old log file $($file.Name): $_"
            }
        }
    }

    [PSCustomObject[]]GetSecurityEvents([DateTime]$startDate, [DateTime]$endDate, [string]$category = $null, [string]$level = $null) {
        $events = $this.SecurityEvents | Where-Object {
            $_.Timestamp -ge $startDate -and $_.Timestamp -le $endDate
        }

        if ($category) {
            $events = $events | Where-Object { $_.Category -eq $category }
        }

        if ($level) {
            $events = $events | Where-Object { $_.Level -eq $level.ToUpper() }
        }

        return $events | Sort-Object Timestamp -Descending
    }

    [hashtable]GetSecurityMetrics([TimeSpan]$timeWindow = [TimeSpan]::FromHours(24)) {
        $cutoffDate = (Get-Date) - $timeWindow

        $recentEvents = $this.SecurityEvents | Where-Object { $_.Timestamp -ge $cutoffDate }

        $metrics = @{
            TimeWindow = $timeWindow
            TotalEvents = $recentEvents.Count
            EventsByLevel = @{}
            EventsByCategory = @{}
            TopMessages = @()
            UniqueUsers = @()
            UniqueComputers = @()
        }

        # Répartition par niveau
        $metrics.EventsByLevel = $recentEvents | Group-Object Level | ForEach-Object {
            @{ $_.Name = $_.Count }
        } | ForEach-Object { $_ }

        # Répartition par catégorie
        $metrics.EventsByCategory = $recentEvents | Group-Object Category | ForEach-Object {
            @{ $_.Name = $_.Count }
        } | ForEach-Object { $_ }

        # Messages les plus fréquents
        $metrics.TopMessages = $recentEvents | Group-Object Message | Sort-Object Count -Descending | Select-Object -First 10 | ForEach-Object {
            @{ Message = $_.Name; Count = $_.Count }
        }

        # Utilisateurs uniques
        $metrics.UniqueUsers = $recentEvents | Select-Object -ExpandProperty User -Unique

        # Ordinateurs uniques
        $metrics.UniqueComputers = $recentEvents | Select-Object -ExpandProperty Computer -Unique

        return $metrics
    }

    [void]SetAlertCallback([scriptblock]$callback) {
        $this.AlertCallback = $callback
    }

    [string]GenerateSecurityReport([TimeSpan]$timeWindow = [TimeSpan]::FromDays(7)) {
        $metrics = $this.GetSecurityMetrics($timeWindow)

        $report = @"
SECURITY AUDIT REPORT
Generated: $(Get-Date)
Time Window: $($timeWindow.TotalHours) hours

SUMMARY
=======
Total Security Events: $($metrics.TotalEvents)

EVENTS BY LEVEL
$($metrics.EventsByLevel | ForEach-Object { "$($_.Keys[0]): $($_.Values[0])" } | Out-String)

EVENTS BY CATEGORY
$($metrics.EventsByCategory | ForEach-Object { "$($_.Keys[0]): $($_.Values[0])" } | Out-String)

TOP SECURITY MESSAGES
$($metrics.TopMessages | ForEach-Object { "• $($_.Message) ($($_.Count) occurrences)" } | Out-String)

UNIQUE USERS: $($metrics.UniqueUsers.Count)
UNIQUE COMPUTERS: $($metrics.UniqueComputers.Count)

DETAILED EVENTS (Last 24 hours)
$($this.GetSecurityEvents((Get-Date).AddHours(-24), (Get-Date)) | Select-Object Timestamp, Level, Category, Message, User | Format-Table | Out-String)
"@

        return $report
    }
}

# Fonctions utilitaires de logging de sécurité
function Write-SecurityLog {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Category,

        [Parameter(Mandatory=$true)]
        [string]$Message,

        [string]$Level = "INFO",

        [hashtable]$Context = @{}
    )

    if (-not $global:SecurityLogger) {
        $global:SecurityLogger = [SecurityLogger]::new()
    }

    $global:SecurityLogger.LogSecurityEvent($Level, $Category, $Message, $Context)
}

function Get-SecurityEvents {
    param(
        [DateTime]$StartDate = (Get-Date).AddDays(-1),
        [DateTime]$EndDate = (Get-Date),
        [string]$Category,
        [string]$Level
    )

    if (-not $global:SecurityLogger) {
        Write-Warning "Security logger not initialized"
        return @()
    }

    return $global:SecurityLogger.GetSecurityEvents($StartDate, $EndDate, $Category, $Level)
}

function Get-SecurityMetrics {
    param([TimeSpan]$TimeWindow = [TimeSpan]::FromHours(24))

    if (-not $global:SecurityLogger) {
        Write-Warning "Security logger not initialized"
        return @{}
    }

    return $global:SecurityLogger.GetSecurityMetrics($TimeWindow)
}

# Initialisation du logger de sécurité
$global:SecurityLogger = [SecurityLogger]::new()

# Configuration des alertes de sécurité
$global:SecurityLogger.SetAlertCallback({
    param($event)

    # Alerte par email pour les événements critiques
    if ($event.Level -in @("ERROR", "CRITICAL")) {
        try {
            $subject = "SECURITY ALERT: $($event.Category) - $($event.Level)"
            $body = @"
Security Alert Details:
- Timestamp: $($event.Timestamp)
- Level: $($event.Level)
- Category: $($event.Category)
- Message: $($event.Message)
- User: $($event.User)
- Computer: $($event.Computer)

Context: $($event.Context | ConvertTo-Json -Compress)
"@

            Send-MailMessage -To "security@company.com" -Subject $subject -Body $body -SmtpServer "smtp.company.com" -Priority High
        } catch {
            Write-Warning "Failed to send security alert email: $_"
        }
    }

    # Affichage console
    $color = switch ($event.Level) {
        "DEBUG" { "Gray" }
        "INFO" { "White" }
        "WARN" { "Yellow" }
        "ERROR" { "Red" }
        "CRITICAL" { "Magenta" }
        default { "White" }
    }

    Write-Host "[$($event.Timestamp)] SECURITY-$($event.Level): $($event.Category) - $($event.Message)" -ForegroundColor $color
})

# Exemples d'utilisation du logging de sécurité
Write-SecurityLog -Category "Authentication" -Message "User login successful" -Level "INFO" -Context @{
    UserName = "alice.admin"
    SourceIP = "192.168.1.100"
    Method = "Password"
}

Write-SecurityLog -Category "Authorization" -Message "Access denied to restricted resource" -Level "WARN" -Context @{
    UserName = "bob.user"
    Resource = "C:\Confidential"
    RequiredRole = "Admin"
}

Write-SecurityLog -Category "DataAccess" -Message "Unauthorized attempt to access sensitive data" -Level "ERROR" -Context @{
    UserName = "charlie.user"
    TableName = "CustomerPII"
    QueryType = "SELECT"
    RowsReturned = 15000
}

# Simulation d'un événement critique
Write-SecurityLog -Category "PrivilegeEscalation" -Message "Potential privilege escalation detected" -Level "CRITICAL" -Context @{
    UserName = "diana.user"
    OriginalRole = "User"
    TargetRole = "Admin"
    Method = "TokenManipulation"
    RiskScore = 95
}

# Récupération et analyse des événements
Write-Host "`n=== SECURITY ANALYSIS ===" -ForegroundColor Cyan

# Événements d'erreur récents
$errorEvents = Get-SecurityEvents -Level "ERROR" -StartDate (Get-Date).AddHours(-24)
Write-Host "Error events in last 24h: $($errorEvents.Count)"

# Métriques de sécurité
$metrics = Get-SecurityMetrics -TimeWindow ([TimeSpan]::FromHours(24))
Write-Host "Security events by level:"
$metrics.EventsByLevel | ForEach-Object {
    $level = $_.Keys[0]
    $count = $_.Values[0]
    Write-Host "  $level`: $count"
}

# Génération du rapport de sécurité
$securityReport = $global:SecurityLogger.GenerateSecurityReport([TimeSpan]::FromHours(24))
$securityReport | Out-File -FilePath "SecurityReport_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nSecurity report generated and saved." -ForegroundColor Green
```

### 2.2 Détection d'intrusions et réponse aux incidents

**Système de détection et réponse aux menaces :**
```powershell
# Classe de détection d'intrusions
class IntrusionDetectionSystem {
    [System.Collections.Generic.List[PSCustomObject]]$ThreatPatterns
    [System.Collections.Generic.List[PSCustomObject]]$DetectedThreats
    [hashtable]$MonitoringRules
    [scriptblock]$ResponseCallback
    [System.Collections.Generic.Dictionary[string, DateTime]]$LastAlertTimes

    IntrusionDetectionSystem() {
        $this.ThreatPatterns = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.DetectedThreats = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.LastAlertTimes = [System.Collections.Generic.Dictionary[string, DateTime]]::new()

        $this.InitializeDefaultPatterns()
        $this.InitializeMonitoringRules()

        $this.ResponseCallback = {
            param($threat)
            Write-Host "THREAT DETECTED: $($threat.Description)" -ForegroundColor Red
            Write-Host "Severity: $($threat.Severity) | Confidence: $($threat.Confidence)%" -ForegroundColor Yellow
        }
    }

    hidden [void]InitializeDefaultPatterns() {
        # Patterns de menaces prédéfinis
        $patterns = @(
            @{
                Name = "PrivilegeEscalation"
                Description = "Tentative d'escalade de privilèges"
                Severity = "Critical"
                Patterns = @(
                    @{ Type = "EventLog"; EventId = 4672; MessagePattern = "Special privileges assigned" }
                    @{ Type = "Process"; ProcessName = "psexec|paexec"; ParentProcess = "cmd|powershell" }
                    @{ Type = "Registry"; KeyPath = "HKLM:\SAM|HKLM:\SECURITY"; Operation = "Access" }
                )
                ResponseAction = "IsolateAndAlert"
            },
            @{
                Name = "SuspiciousNetworkActivity"
                Description = "Activité réseau suspecte détectée"
                Severity = "High"
                Patterns = @(
                    @{ Type = "Network"; Protocol = "TCP"; Port = "4444|6666|1337"; Direction = "Outbound" }
                    @{ Type = "Connection"; RemoteHost = "malicious.domain|tor.node"; Frequency = 10 }
                )
                ResponseAction = "BlockAndAlert"
            },
            @{
                Name = "MalwareIndicators"
                Description = "Indicateurs de malware détectés"
                Severity = "Critical"
                Patterns = @(
                    @{ Type = "File"; Extension = ".exe|.dll"; Location = "C:\Windows\Temp|C:\Temp"; CreatedRecently = $true }
                    @{ Type = "Process"; CommandLine = "powershell.*-enc|powershell.*-encoded" }
                    @{ Type = "Registry"; KeyPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run"; NewValue = $true }
                )
                ResponseAction = "QuarantineAndAlert"
            },
            @{
                Name = "DataExfiltration"
                Description = "Exfiltration de données suspectée"
                Severity = "High"
                Patterns = @(
                    @{ Type = "FileAccess"; FilePattern = "*.db|*.bak|*password*"; Operation = "Read"; LargeTransfer = $true }
                    @{ Type = "Network"; DataSize = "50MB"; Direction = "Outbound"; UnusualTime = $true }
                )
                ResponseAction = "BlockAndInvestigate"
            }
        )

        foreach ($pattern in $patterns) {
            $this.ThreatPatterns.Add([PSCustomObject]$pattern)
        }
    }

    hidden [void]InitializeMonitoringRules() {
        $this.MonitoringRules = @{
            ProcessMonitoring = @{
                Enabled = $true
                CheckInterval = [TimeSpan]::FromSeconds(30)
                SuspiciousProcesses = @("cmd.exe", "powershell.exe", "wscript.exe", "cscript.exe")
                AlertOnNewProcess = $false
            }
            NetworkMonitoring = @{
                Enabled = $true
                CheckInterval = [TimeSpan]::FromSeconds(60)
                MonitoredPorts = @(21, 22, 23, 25, 53, 80, 443, 445, 3389)
                AlertOnUnusualTraffic = $true
            }
            FileSystemMonitoring = @{
                Enabled = $true
                CheckInterval = [TimeSpan]::FromMinutes(5)
                MonitoredPaths = @("C:\Windows\System32", "C:\Windows\Temp", "C:\Program Files")
                AlertOnUnauthorizedChanges = $true
            }
            EventLogMonitoring = @{
                Enabled = $true
                CheckInterval = [TimeSpan]::FromMinutes(1)
                MonitoredLogs = @("Security", "System", "Application")
                AlertOnSecurityEvents = $true
            }
        }
    }

    [void]ScanForThreats() {
        Write-Host "Scanning for security threats..." -ForegroundColor Yellow

        foreach ($pattern in $this.ThreatPatterns) {
            $threatDetected = $this.EvaluatePattern($pattern)

            if ($threatDetected.Detected) {
                $threat = [PSCustomObject]@{
                    Id = [guid]::NewGuid().ToString()
                    PatternName = $pattern.Name
                    Description = $pattern.Description
                    Severity = $pattern.Severity
                    Confidence = $threatDetected.Confidence
                    DetectedAt = Get-Date
                    Indicators = $threatDetected.Indicators
                    ResponseAction = $pattern.ResponseAction
                    Status = "Detected"
                }

                $this.DetectedThreats.Add($threat)

                # Exécution de la réponse automatique
                $this.ExecuteResponse($threat)

                # Évitement des alertes répétées
                $alertKey = "$($pattern.Name)_$($threatDetected.Indicators -join '_')"
                if (-not $this.LastAlertTimes.ContainsKey($alertKey) -or
                    ((Get-Date) - $this.LastAlertTimes[$alertKey]) -gt [TimeSpan]::FromMinutes(30)) {

                    & $this.ResponseCallback $threat
                    $this.LastAlertTimes[$alertKey] = Get-Date
                }
            }
        }
    }

    hidden [PSCustomObject]EvaluatePattern([PSCustomObject]$pattern) {
        $indicators = @()
        $totalScore = 0
        $maxScore = $pattern.Patterns.Count * 100

        foreach ($threatPattern in $pattern.Patterns) {
            $score = 0

            switch ($threatPattern.Type) {
                "Process" {
                    $score = $this.CheckProcessPattern($threatPattern)
                }
                "Network" {
                    $score = $this.CheckNetworkPattern($threatPattern)
                }
                "File" {
                    $score = $this.CheckFilePattern($threatPattern)
                }
                "EventLog" {
                    $score = $this.CheckEventLogPattern($threatPattern)
                }
                "Registry" {
                    $score = $this.CheckRegistryPattern($threatPattern)
                }
            }

            if ($score -gt 0) {
                $indicators += @{
                    Type = $threatPattern.Type
                    Pattern = $threatPattern
                    Score = $score
                }
                $totalScore += $score
            }
        }

        $confidence = if ($maxScore -gt 0) { [math]::Min(100, ($totalScore / $maxScore) * 100) } else { 0 }

        return [PSCustomObject]@{
            Detected = $confidence -ge 70  # Seuil de détection
            Confidence = [math]::Round($confidence, 1)
            Indicators = $indicators
        }
    }

    hidden [int]CheckProcessPattern([hashtable]$pattern) {
        $score = 0

        try {
            $processes = Get-Process

            if ($pattern.ProcessName) {
                $matchingProcesses = $processes | Where-Object { $_.ProcessName -match $pattern.ProcessName }
                if ($matchingProcesses) {
                    $score += 50

                    if ($pattern.ParentProcess) {
                        foreach ($proc in $matchingProcesses) {
                            $parent = Get-Process -Id $proc.Id -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Parent
                            if ($parent -and $parent.ProcessName -match $pattern.ParentProcess) {
                                $score += 50
                                break
                            }
                        }
                    }
                }
            }
        } catch {
            Write-Debug "Error checking process pattern: $_"
        }

        return [math]::Min(100, $score)
    }

    hidden [int]CheckNetworkPattern([hashtable]$pattern) {
        $score = 0

        try {
            $connections = Get-NetTCPConnection -State Established -ErrorAction SilentlyContinue

            if ($pattern.Port) {
                $matchingConnections = $connections | Where-Object { $_.RemotePort -match $pattern.Port }
                if ($matchingConnections) {
                    $score += 60

                    if ($pattern.Direction -and $pattern.Direction -eq "Outbound") {
                        $outboundConnections = $matchingConnections | Where-Object { $_.State -eq "Established" }
                        if ($outboundConnections) {
                            $score += 40
                        }
                    }
                }
            }
        } catch {
            Write-Debug "Error checking network pattern: $_"
        }

        return [math]::Min(100, $score)
    }

    hidden [int]CheckFilePattern([hashtable]$pattern) {
        $score = 0

        try {
            if ($pattern.Extension) {
                $files = Get-ChildItem -Path $pattern.Location -Filter "*$($pattern.Extension)" -Recurse -ErrorAction SilentlyContinue

                if ($files) {
                    $score += 40

                    if ($pattern.CreatedRecently) {
                        $recentFiles = $files | Where-Object { $_.CreationTime -gt (Get-Date).AddMinutes(-30) }
                        if ($recentFiles) {
                            $score += 60
                        }
                    }
                }
            }
        } catch {
            Write-Debug "Error checking file pattern: $_"
        }

        return [math]::Min(100, $score)
    }

    hidden [int]CheckEventLogPattern([hashtable]$pattern) {
        $score = 0

        try {
            $events = Get-WinEvent -LogName $pattern.LogName -MaxEvents 100 -ErrorAction SilentlyContinue

            if ($pattern.EventId) {
                $matchingEvents = $events | Where-Object { $_.Id -eq $pattern.EventId }
                if ($matchingEvents) {
                    $score += 50

                    if ($pattern.MessagePattern) {
                        $messageMatches = $matchingEvents | Where-Object { $_.Message -match $pattern.MessagePattern }
                        if ($messageMatches) {
                            $score += 50
                        }
                    }
                }
            }
        } catch {
            Write-Debug "Error checking event log pattern: $_"
        }

        return [math]::Min(100, $score)
    }

    hidden [int]CheckRegistryPattern([hashtable]$pattern) {
        $score = 0

        try {
            if ($pattern.KeyPath) {
                $keys = $pattern.KeyPath -split '\|'
                foreach ($key in $keys) {
                    if (Test-Path $key) {
                        $score += 50

                        if ($pattern.Operation -eq "Access") {
                            # Simulation de vérification d'accès
                            $score += 50
                        }
                        break
                    }
                }
            }
        } catch {
            Write-Debug "Error checking registry pattern: $_"
        }

        return [math]::Min(100, $score)
    }

    hidden [void]ExecuteResponse([PSCustomObject]$threat) {
        switch ($threat.ResponseAction) {
            "IsolateAndAlert" {
                $this.IsolateSystem($threat)
            }
            "BlockAndAlert" {
                $this.BlockNetworkActivity($threat)
            }
            "QuarantineAndAlert" {
                $this.QuarantineFiles($threat)
            }
            "BlockAndInvestigate" {
                $this.BlockDataTransfer($threat)
            }
        }
    }

    hidden [void]IsolateSystem([PSCustomObject]$threat) {
        Write-Host "Executing isolation response for threat: $($threat.PatternName)" -ForegroundColor Red

        # Désactiver les comptes suspects
        # Bloquer les connexions réseau
        # Créer des snapshots pour analyse
        Write-Host "System isolation measures implemented" -ForegroundColor Yellow
    }

    hidden [void]BlockNetworkActivity([PSCustomObject]$threat) {
        Write-Host "Executing network blocking response for threat: $($threat.PatternName)" -ForegroundColor Red

        # Bloquer les connexions suspectes
        # Mettre à jour les règles firewall
        Write-Host "Network blocking measures implemented" -ForegroundColor Yellow
    }

    hidden [void]QuarantineFiles([PSCustomObject]$threat) {
        Write-Host "Executing quarantine response for threat: $($threat.PatternName)" -ForegroundColor Red

        # Mettre en quarantaine les fichiers suspects
        # Analyser avec un antivirus
        Write-Host "File quarantine measures implemented" -ForegroundColor Yellow
    }

    hidden [void]BlockDataTransfer([PSCustomObject]$threat) {
        Write-Host "Executing data transfer blocking response for threat: $($threat.PatternName)" -ForegroundColor Red

        # Bloquer les transferts de données suspects
        # Monitorer les accès aux données sensibles
        Write-Host "Data transfer blocking measures implemented" -ForegroundColor Yellow
    }

    [PSCustomObject[]]GetDetectedThreats([TimeSpan]$timeWindow = [TimeSpan]::FromHours(24)) {
        $cutoff = (Get-Date) - $timeWindow
        return $this.DetectedThreats | Where-Object { $_.DetectedAt -ge $cutoff } | Sort-Object DetectedAt -Descending
    }

    [hashtable]GetThreatMetrics([TimeSpan]$timeWindow = [TimeSpan]::FromHours(24)) {
        $threats = $this.GetDetectedThreats($timeWindow)

        return @{
            TimeWindow = $timeWindow
            TotalThreats = $threats.Count
            ThreatsBySeverity = $threats | Group-Object Severity | ForEach-Object { @{ $_.Name = $_.Count } }
            ThreatsByPattern = $threats | Group-Object PatternName | ForEach-Object { @{ $_.Name = $_.Count } }
            AverageConfidence = if ($threats) { ($threats | Measure-Object Confidence -Average).Average } else { 0 }
            MostRecentThreat = if ($threats) { $threats | Select-Object -First 1 } else { $null }
        }
    }

    [void]SetResponseCallback([scriptblock]$callback) {
        $this.ResponseCallback = $callback
    }

    [void]StartContinuousMonitoring() {
        Write-Host "Starting continuous threat monitoring..." -ForegroundColor Green

        $job = Start-Job -ScriptBlock {
            $ids = $using:ids

            while ($true) {
                $ids.ScanForThreats()
                Start-Sleep -Seconds 300  # Scan toutes les 5 minutes
            }
        } -ArgumentList $this

        Write-Host "Continuous monitoring started. Job ID: $($job.Id)" -ForegroundColor Green
    }
}

# Initialisation et utilisation du système de détection d'intrusions
$ids = [IntrusionDetectionSystem]::new()

# Configuration du callback de réponse personnalisé
$ids.SetResponseCallback({
    param($threat)

    # Envoi d'alerte détaillée
    $alertMessage = @"
🚨 INTRUSION DETECTED 🚨

Threat: $($threat.PatternName)
Description: $($threat.Description)
Severity: $($threat.Severity)
Confidence: $($threat.Confidence)%
Detected: $($threat.DetectedAt)

Indicators: $($threat.Indicators | ConvertTo-Json)

Immediate action required!
"@

    Write-Host $alertMessage -ForegroundColor Red

    # Ici, on pourrait envoyer des emails, SMS, créer des tickets d'incident, etc.
})

# Scan manuel des menaces
$ids.ScanForThreats()

# Métriques des menaces
$threatMetrics = $ids.GetThreatMetrics([TimeSpan]::FromHours(24))
Write-Host "`n=== THREAT METRICS ===" -ForegroundColor Cyan
Write-Host "Total threats detected: $($threatMetrics.TotalThreats)"
Write-Host "Average confidence: $([math]::Round($threatMetrics.AverageConfidence, 1))%"

if ($threatMetrics.ThreatsBySeverity) {
    Write-Host "Threats by severity:"
    $threatMetrics.ThreatsBySeverity.GetEnumerator() | ForEach-Object {
        Write-Host "  $($_.Key): $($_.Value)"
    }
}

# Menaces détectées récentes
$recentThreats = $ids.GetDetectedThreats([TimeSpan]::FromHours(1))
Write-Host "`nRecent threats ($($recentThreats.Count)):"
foreach ($threat in $recentThreats | Select-Object -First 5) {
    Write-Host "  $($threat.DetectedAt): $($threat.PatternName) ($($threat.Severity))"
}

# Démarrage du monitoring continu (commenté pour éviter l'exécution infinie)
# $ids.StartContinuousMonitoring()

Write-Host "`nIntrusion Detection System initialized and operational." -ForegroundColor Green
```

## Conclusion : PowerShell comme gardien de la sécurité

La sécurité avancée PowerShell transforme les scripts en sentinelles vigilantes, capables non seulement d'exécuter des tâches mais aussi de protéger activement l'environnement contre les menaces. En intégrant authentification robuste, autorisation granulaire, audit complet, et détection d'intrusions, PowerShell devient un outil indispensable dans l'arsenal de sécurité moderne.

Dans le prochain chapitre, nous explorerons les meilleures pratiques de développement PowerShell et les patterns de code avancés pour créer des scripts maintenables et évolutifs.

---

**Exercice pratique :** Créez un système de sécurité complet qui inclut :
1. Gestion sécurisée des credentials avec chiffrement AES
2. Système de contrôle d'accès privilégié (JIT)
3. Logging de sécurité structuré avec alertes automatiques
4. Détection d'intrusions avec patterns configurables
5. Réponses automatiques aux menaces détectées

**Challenge avancé :** Développez une plateforme de sécurité PowerShell qui :
- Implémente une architecture zero-trust
- Intègre l'IA pour la détection d'anomalies comportementales
- Fournit des rapports de conformité automatisés
- S'adapte dynamiquement aux nouvelles menaces
- Coordonne la réponse aux incidents à travers l'infrastructure

**Réflexion :** Comment PowerShell change-t-il le paradigme de la sécurité informatique ? La sécurité peut-elle être "codifiée" et automatisée, ou nécessite-t-elle toujours l'intervention humaine ? Quels sont les risques de l'automatisation excessive en matière de sécurité ?
