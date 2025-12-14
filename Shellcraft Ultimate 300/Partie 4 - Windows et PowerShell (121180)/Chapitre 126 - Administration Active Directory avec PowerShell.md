# Chapitre 126 - Administration Active Directory avec PowerShell

> "Active Directory n'était qu'une base de données hiérarchique jusqu'à ce que PowerShell la transforme en un système programmable où chaque objet devient une opportunité d'automatisation." - Administrateur Active Directory

## Introduction : AD comme API programmable

Active Directory (AD) est le cœur de la plupart des environnements Windows d'entreprise. Traditionnellement administré via des outils graphiques lourds ou des scripts VBScript limités, AD devient avec PowerShell un système hautement programmable où chaque aspect de l'infrastructure d'identité peut être automatisé.

Dans ce chapitre, nous explorerons comment PowerShell révolutionne l'administration AD, de la gestion des utilisateurs aux politiques de groupe, en passant par l'audit et la sécurité.

## Section 1 : Premiers pas avec le module Active Directory

### 1.1 Installation et configuration

```powershell
# Vérifier si le module AD est disponible
Get-Module -Name ActiveDirectory -ListAvailable

# Installer le module RSAT (si nécessaire)
# Sur Windows 10/11 : Activer via Paramètres > Applications > Fonctionnalités facultatives
# Ou via PowerShell :
Get-WindowsCapability -Name "Rsat.ActiveDirectory.DS-LDS.Tools*" -Online | Add-WindowsCapability -Online

# Importer le module
Import-Module ActiveDirectory

# Vérifier la connectivité au domaine
Get-ADDomain

# Informations sur le domaine
Get-ADDomain | Select-Object Name, DomainMode, ForestMode, DNSRoot

# Contrôleur de domaine actuel
Get-ADDomainController
```

### 1.2 Connexion à des domaines distants

```powershell
# Se connecter à un domaine spécifique
$domainCredential = Get-Credential -UserName "DOMAIN\Administrator" -Message "Credentials for domain access"
$domainController = "DC01.domain.com"

# Utiliser un contrôleur de domaine spécifique
Get-ADUser -Filter * -Server $domainController -Credential $domainCredential | Select-Object -First 5

# Configuration d'une session AD persistante
$adSession = New-PSSession -ComputerName $domainController -Credential $domainCredential
Invoke-Command -Session $adSession -ScriptBlock { Import-Module ActiveDirectory }

# Utilisation de la session pour les commandes AD
Invoke-Command -Session $adSession -ScriptBlock { 
    Get-ADUser -Filter {Enabled -eq $true} | Measure-Object 
}
```

### 1.3 Exploration de la structure AD

```powershell
# Structure du domaine
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName, Description

# Comptage des objets
Write-Host "=== STATISTIQUES DU DOMAINE ==="
Write-Host "Utilisateurs: $((Get-ADUser -Filter *).Count)"
Write-Host "Groupes: $((Get-ADGroup -Filter *).Count)"
Write-Host "Ordinateurs: $((Get-ADComputer -Filter *).Count)"
Write-Host "Unités d'organisation: $((Get-ADOrganizationalUnit -Filter *).Count)"

# Hiérarchie des OU
function Get-OUHierarchy {
    param([string]$BaseOU = (Get-ADDomain).DistinguishedName)
    
    Get-ADOrganizationalUnit -SearchBase $BaseOU -Filter * | ForEach-Object {
        $level = ($_.DistinguishedName -split ',OU=').Count - 1
        $indent = "  " * $level
        
        [PSCustomObject]@{
            Level = $level
            Name = $_.Name
            DN = $_.DistinguishedName
            Description = $_.Description
        }
    } | Sort-Object Level, Name
}

Get-OUHierarchy | Format-Table -Property @{Name='OU'; Expression={"  " * $_.Level + $_.Name}}, Description -AutoSize
```

## Section 2 : Gestion des utilisateurs

### 2.1 Recherche et consultation d'utilisateurs

```powershell
# Tous les utilisateurs
Get-ADUser -Filter * | Select-Object Name, SamAccountName, UserPrincipalName, Enabled

# Utilisateurs actifs/inactifs
Get-ADUser -Filter {Enabled -eq $true} | Measure-Object
Get-ADUser -Filter {Enabled -eq $false} | Select-Object Name, SamAccountName, LastLogonDate

# Recherche par attributs
Get-ADUser -Filter {Department -eq "IT"} | Select-Object Name, Title, EmailAddress
Get-ADUser -Filter {City -eq "Paris"} | Select-Object Name, Office, TelephoneNumber

# Utilisateurs avec mot de passe expiré
Get-ADUser -Filter {PasswordExpired -eq $true} | Select-Object Name, SamAccountName, PasswordLastSet

# Dernière connexion
$lastLogonThreshold = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogon -lt $lastLogonThreshold -and Enabled -eq $true} | 
    Select-Object Name, SamAccountName, LastLogonDate

# Utilisateurs privilégiés
$adminGroups = @("Domain Admins", "Enterprise Admins", "Administrators")
foreach ($group in $adminGroups) {
    Get-ADGroupMember -Identity $group -Recursive | 
        Where-Object {$_.objectClass -eq 'user'} | 
        Select-Object @{Name='Group'; Expression={$group}}, Name, SamAccountName
}
```

### 2.2 Création et modification d'utilisateurs

```powershell
# Création d'utilisateur simple
New-ADUser -Name "Jean Dupont" `
    -SamAccountName "jdupont" `
    -UserPrincipalName "jdupont@domain.com" `
    -GivenName "Jean" `
    -Surname "Dupont" `
    -DisplayName "Jean Dupont" `
    -EmailAddress "jean.dupont@domain.com" `
    -Department "IT" `
    -Title "Technicien Système" `
    -Office "Bureau 101" `
    -AccountPassword (ConvertTo-SecureString "TempPass123!" -AsPlainText -Force) `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -ChangePasswordAtLogon $true

# Création en masse depuis CSV
$users = Import-Csv -Path "C:\Users\new_users.csv"

foreach ($user in $users) {
    $password = ConvertTo-SecureString $user.Password -AsPlainText -Force
    
    New-ADUser -Name "$($user.FirstName) $($user.LastName)" `
        -SamAccountName $user.SamAccountName `
        -UserPrincipalName "$($user.SamAccountName)@domain.com" `
        -GivenName $user.FirstName `
        -Surname $user.LastName `
        -Department $user.Department `
        -Title $user.Title `
        -AccountPassword $password `
        -Enabled $true `
        -Path $user.OU
}

# Modification d'utilisateurs existants
Set-ADUser -Identity "jdupont" `
    -EmailAddress "jean.dupont@newdomain.com" `
    -OfficePhone "01-23-45-67-89" `
    -MobilePhone "06-12-34-56-78" `
    -StreetAddress "123 Rue de la Paix" `
    -City "Paris" `
    -PostalCode "75001" `
    -Country "FR"

# Mise à jour en masse
Get-ADUser -Filter {Department -eq "OldDepartment"} | 
    Set-ADUser -Department "NewDepartment" -Title "Updated Title"

# Gestion des mots de passe
Set-ADAccountPassword -Identity "jdupont" -NewPassword (ConvertTo-SecureString "NewPass123!" -AsPlainText -Force)

# Forcer le changement de mot de passe
Set-ADUser -Identity "jdupont" -ChangePasswordAtLogon $true

# Débloquer un compte
Unlock-ADAccount -Identity "jdupont"

# Activer/désactiver des comptes
Disable-ADAccount -Identity "olduser"
Enable-ADAccount -Identity "jdupont"
```

### 2.3 Gestion avancée des comptes utilisateurs

```powershell
# Audit des comptes expirés
$expiredAccounts = Search-ADAccount -AccountExpired | Select-Object Name, SamAccountName, AccountExpirationDate
$expiredAccounts | Export-Csv -Path "expired_accounts.csv" -NoTypeInformation

# Comptes proches de l'expiration
$threshold = (Get-Date).AddDays(7)
$expiringSoon = Search-ADAccount -AccountExpiring -DateTime $threshold | 
    Select-Object Name, SamAccountName, AccountExpirationDate
$expiringSoon | Format-Table -AutoSize

# Comptes verrouillés
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LastLogonDate, BadLogonCount

# Réinitialisation des mots de passe en masse
function Reset-PasswordsBulk {
    param(
        [string]$OU,
        [string]$NewPassword = "TempPass$(Get-Random -Maximum 9999)!"
    )
    
    $users = Get-ADUser -Filter * -SearchBase $OU -Properties EmailAddress
    
    foreach ($user in $users) {
        $securePassword = ConvertTo-SecureString $NewPassword -AsPlainText -Force
        Set-ADAccountPassword -Identity $user -NewPassword $securePassword
        Set-ADUser -Identity $user -ChangePasswordAtLogon $true
        
        Write-Host "Mot de passe réinitialisé pour $($user.Name)"
        
        # Notification par email (si serveur SMTP configuré)
        if ($user.EmailAddress) {
            Send-MailMessage -To $user.EmailAddress `
                -Subject "Réinitialisation de votre mot de passe" `
                -Body "Votre nouveau mot de passe temporaire est: $NewPassword`nVeuillez le changer lors de votre prochaine connexion." `
                -SmtpServer "smtp.domain.com"
        }
    }
}

# Utilisation
Reset-PasswordsBulk -OU "OU=Users,DC=domain,DC=com"
```

## Section 3 : Gestion des groupes

### 3.1 Types de groupes et gestion de base

```powershell
# Lister les groupes
Get-ADGroup -Filter * | Select-Object Name, GroupCategory, GroupScope, Description

# Groupes de sécurité vs distribution
Get-ADGroup -Filter {GroupCategory -eq "Security"}
Get-ADGroup -Filter {GroupCategory -eq "Distribution"}

# Portée des groupes
Get-ADGroup -Filter {GroupScope -eq "Global"}
Get-ADGroup -Filter {GroupScope -eq "DomainLocal"}
Get-ADGroup -Filter {GroupScope -eq "Universal"}

# Création de groupes
New-ADGroup -Name "IT Support Staff" `
    -SamAccountName "ITSupport" `
    -GroupCategory Security `
    -GroupScope Global `
    -DisplayName "IT Support Staff" `
    -Description "Groupe des techniciens support IT" `
    -Path "OU=Groups,DC=domain,DC=com"

# Modification de groupes
Set-ADGroup -Identity "ITSupport" -Description "Équipe support informatique - Mise à jour"
```

### 3.2 Gestion des membres de groupes

```powershell
# Ajouter des membres
Add-ADGroupMember -Identity "ITSupport" -Members "jdupont", "asmith"

# Ajouter des utilisateurs depuis une OU
$users = Get-ADUser -Filter {Department -eq "IT"} -SearchBase "OU=IT,DC=domain,DC=com"
Add-ADGroupMember -Identity "ITSupport" -Members $users

# Lister les membres
Get-ADGroupMember -Identity "ITSupport" | Select-Object Name, SamAccountName, objectClass

# Membres récursifs (incluant les groupes imbriqués)
Get-ADGroupMember -Identity "Domain Admins" -Recursive | 
    Where-Object {$_.objectClass -eq 'user'} | 
    Select-Object Name, SamAccountName

# Supprimer des membres
Remove-ADGroupMember -Identity "ITSupport" -Members "olduser" -Confirm:$false

# Vider un groupe
Get-ADGroupMember -Identity "TempGroup" | ForEach-Object {
    Remove-ADGroupMember -Identity "TempGroup" -Members $_ -Confirm:$false
}
```

### 3.3 Gestion dynamique des groupes

```powershell
# Création de groupes dynamiques basés sur des règles
function Update-DynamicGroups {
    param([hashtable]$GroupRules)
    
    foreach ($groupName in $GroupRules.Keys) {
        $rule = $GroupRules[$groupName]
        
        # Trouver les utilisateurs correspondants à la règle
        $matchingUsers = Get-ADUser -Filter $rule.Filter -SearchBase $rule.SearchBase
        
        # Obtenir les membres actuels
        $currentMembers = Get-ADGroupMember -Identity $groupName | 
            Where-Object {$_.objectClass -eq 'user'}
        
        # Calculer les différences
        $toAdd = $matchingUsers | Where-Object { $_.DistinguishedName -notin $currentMembers.DistinguishedName }
        $toRemove = $currentMembers | Where-Object { $_.DistinguishedName -notin $matchingUsers.DistinguishedName }
        
        # Appliquer les changements
        if ($toAdd) {
            Add-ADGroupMember -Identity $groupName -Members $toAdd
            Write-Host "Ajouté $($toAdd.Count) membres au groupe $groupName"
        }
        
        if ($toRemove) {
            Remove-ADGroupMember -Identity $groupName -Members $toRemove -Confirm:$false
            Write-Host "Retiré $($toRemove.Count) membres du groupe $groupName"
        }
    }
}

# Définition des règles pour les groupes dynamiques
$dynamicGroupRules = @{
    "All_IT_Users" = @{
        Filter = "Department -eq 'IT'"
        SearchBase = "OU=Users,DC=domain,DC=com"
    }
    "Paris_Employees" = @{
        Filter = "City -eq 'Paris'"
        SearchBase = "OU=Users,DC=domain,DC=com"
    }
    "Managers" = @{
        Filter = "Title -like '*Manager*'"
        SearchBase = "OU=Users,DC=domain,DC=com"
    }
}

# Mise à jour des groupes dynamiques
Update-DynamicGroups -GroupRules $dynamicGroupRules
```

## Section 4 : Politiques de groupe (GPO)

### 4.1 Gestion des GPO

```powershell
# Lister les GPO
Get-GPO -All | Select-Object DisplayName, Id, GpoStatus, CreationTime, ModificationTime

# Informations détaillées sur une GPO
Get-GPO -Name "Default Domain Policy" | Format-List *

# Création de GPO
New-GPO -Name "IT Security Policy" -Comment "Politique de sécurité pour le département IT"

# Copie de GPO
Copy-GPO -SourceName "IT Security Policy" -TargetName "IT Security Policy - Backup"

# Sauvegarde de GPO
Backup-GPO -Name "IT Security Policy" -Path "C:\GPO_Backups"

# Restauration de GPO
Restore-GPO -Name "IT Security Policy" -Path "C:\GPO_Backups\{GUID}"

# Suppression de GPO
Remove-GPO -Name "Old Policy" -Confirm:$false
```

### 4.2 Liaison des GPO

```powershell
# Lister les liaisons GPO pour une OU
Get-GPInheritance -Target "OU=IT,DC=domain,DC=com"

# Créer une liaison GPO
New-GPLink -Name "IT Security Policy" -Target "OU=IT,DC=domain,DC=com" -LinkEnabled Yes -Order 1

# Modifier l'ordre des liaisons
Set-GPLink -Name "IT Security Policy" -Target "OU=IT,DC=domain,DC=com" -Order 2

# Désactiver une liaison
Set-GPLink -Name "IT Security Policy" -Target "OU=IT,DC=domain,DC=com" -LinkEnabled No

# Supprimer une liaison
Remove-GPLink -Name "IT Security Policy" -Target "OU=IT,DC=domain,DC=com"
```

### 4.3 Simulation et dépannage GPO

```powershell
# Résultante des stratégies (RSoP)
Get-GPResultantSetOfPolicy -Computer "WORKSTATION01" -User "domain\jdupont" -ReportType HTML -Path "C:\Reports\RSoP.html"

# Simulation d'application GPO
Invoke-GPUpdate -Computer "WORKSTATION01" -Force

# Audit des GPO
Get-GPO -All | ForEach-Object {
    $gpo = $_
    $reports = Get-GPOReport -Guid $gpo.Id -ReportType XML
    
    [PSCustomObject]@{
        Name = $gpo.DisplayName
        Status = $gpo.GpoStatus
        Links = (Get-GPInheritance -Target "OU=IT,DC=domain,DC=com" | Where-Object {$_.GpoLinks.GpoId -eq $gpo.Id}).Count
        LastModified = $gpo.ModificationTime
    }
} | Sort-Object LastModified -Descending
```

## Section 5 : Audit et sécurité AD

### 5.1 Audit des événements AD

```powershell
# Événements de sécurité récents
Get-WinEvent -LogName Security -MaxEvents 100 | Where-Object {
    $_.Id -eq 4720 -or # Création utilisateur
    $_.Id -eq 4722 -or # Activation compte
    $_.Id -eq 4725     # Désactivation compte
} | Select-Object TimeCreated, Id, Message | Format-Table -AutoSize

# Audit des changements de groupe
Get-WinEvent -LogName Security | Where-Object {
    $_.Id -eq 4728 -or # Ajout à groupe
    $_.Id -eq 4732     # Suppression de groupe
} | Select-Object TimeCreated, Id, Message

# Échecs d'authentification
Get-WinEvent -LogName Security | Where-Object {
    $_.Id -eq 4625 # Échec de connexion
} | Select-Object TimeCreated, @{Name='User'; Expression={$_.Properties[5].Value}}, @{Name='IP'; Expression={$_.Properties[18].Value}}
```

### 5.2 Rapports de conformité

```powershell
# Rapport d'inventaire utilisateurs
function Get-ADInventoryReport {
    $report = [PSCustomObject]@{
        TotalUsers = (Get-ADUser -Filter *).Count
        EnabledUsers = (Get-ADUser -Filter {Enabled -eq $true}).Count
        DisabledUsers = (Get-ADUser -Filter {Enabled -eq $false}).Count
        UsersWithoutPasswordExpiration = (Get-ADUser -Filter {PasswordNeverExpires -eq $true -and Enabled -eq $true}).Count
        ExpiredPasswords = (Get-ADUser -Filter {PasswordExpired -eq $true}).Count
        LockedAccounts = (Search-ADAccount -LockedOut).Count
        StaleAccounts = (Search-ADAccount -AccountInactive -TimeSpan 90).Count
    }
    
    return $report
}

Get-ADInventoryReport | Format-List

# Audit des privilèges élevés
function Get-PrivilegedAccountsReport {
    $privilegedGroups = @(
        "Domain Admins",
        "Enterprise Admins", 
        "Administrators",
        "Schema Admins",
        "Account Operators"
    )
    
    $report = @()
    
    foreach ($group in $privilegedGroups) {
        try {
            $members = Get-ADGroupMember -Identity $group -Recursive | 
                Where-Object {$_.objectClass -eq 'user'}
            
            foreach ($member in $members) {
                $user = Get-ADUser -Identity $member -Properties LastLogonDate, PasswordLastSet, AccountExpirationDate
                
                $report += [PSCustomObject]@{
                    Group = $group
                    User = $member.Name
                    SamAccountName = $member.SamAccountName
                    LastLogon = $user.LastLogonDate
                    PasswordLastSet = $user.PasswordLastSet
                    AccountExpires = $user.AccountExpirationDate
                }
            }
        } catch {
            Write-Warning "Impossible d'accéder au groupe $group : $($_.Exception.Message)"
        }
    }
    
    return $report
}

Get-PrivilegedAccountsReport | Export-Csv -Path "privileged_accounts_audit.csv" -NoTypeInformation
```

### 5.3 Automatisation de la conformité

```powershell
# Vérifications automatiques de conformité
function Test-ADCompliance {
    $issues = @()
    
    # 1. Comptes avec mots de passe expirés
    $expiredPasswords = Get-ADUser -Filter {PasswordExpired -eq $true -and Enabled -eq $true}
    if ($expiredPasswords.Count -gt 0) {
        $issues += "Comptes avec mots de passe expirés: $($expiredPasswords.Count)"
    }
    
    # 2. Comptes désactivés récemment
    $recentlyDisabled = Get-ADUser -Filter {Enabled -eq $false} -Properties WhenChanged | 
        Where-Object { $_.WhenChanged -gt (Get-Date).AddDays(-7) }
    if ($recentlyDisabled.Count -gt 0) {
        $issues += "Comptes désactivés récemment: $($recentlyDisabled.Count)"
    }
    
    # 3. Utilisateurs sans adresse email
    $noEmail = Get-ADUser -Filter {EmailAddress -notlike "*" -and Enabled -eq $true}
    if ($noEmail.Count -gt 0) {
        $issues += "Utilisateurs sans adresse email: $($noEmail.Count)"
    }
    
    # 4. Groupes vides
    $emptyGroups = Get-ADGroup -Filter * | Where-Object {
        (Get-ADGroupMember -Identity $_).Count -eq 0
    }
    if ($emptyGroups.Count -gt 0) {
        $issues += "Groupes vides: $($emptyGroups.Count)"
    }
    
    # Rapport final
    if ($issues.Count -eq 0) {
        Write-Host "✓ Toutes les vérifications de conformité passées"
        return $true
    } else {
        Write-Warning "Problèmes de conformité détectés:"
        $issues | ForEach-Object { Write-Warning "  - $_" }
        return $false
    }
}

# Exécution des vérifications
$compliant = Test-ADCompliance

# Actions correctives automatiques (avec confirmation)
if (-not $compliant) {
    $correct = Read-Host "Voulez-vous appliquer les corrections automatiques ? (o/N)"
    if ($correct -eq 'o') {
        # Désactiver les comptes expirés depuis plus de 30 jours
        $oldExpired = Get-ADUser -Filter {PasswordExpired -eq $true -and Enabled -eq $true} -Properties PasswordLastSet | 
            Where-Object { $_.PasswordLastSet -lt (Get-Date).AddDays(-30) }
        
        foreach ($user in $oldExpired) {
            Disable-ADAccount -Identity $user
            Write-Host "Compte désactivé: $($user.Name)"
        }
    }
}
```

## Section 6 : Automatisation avancée AD

### 6.1 Pipelines AD complexes

```powershell
# Pipeline complexe pour la gestion des comptes
Get-ADUser -Filter {Enabled -eq $true} -Properties LastLogonDate, Department |
    Where-Object { $_.LastLogonDate -lt (Get-Date).AddDays(-90) } |
    Group-Object Department |
    ForEach-Object {
        $dept = $_.Name
        $staleUsers = $_.Group
        
        Write-Host "Département $dept : $($staleUsers.Count) comptes inactifs"
        
        # Créer un rapport par département
        $staleUsers | Select-Object Name, SamAccountName, LastLogonDate, Department |
            Export-Csv -Path "C:\Reports\StaleAccounts_$dept.csv" -NoTypeInformation
    }

# Gestion des licences Office 365 via AD
$licensedUsers = Get-ADUser -Filter {Enabled -eq $true} -Properties extensionAttribute1 |
    Where-Object { $_.extensionAttribute1 -eq "O365_Licensed" }

# Synchronisation AD vers applications tierces
function Sync-ADToExternalSystem {
    param([string]$ExternalAPIUrl, [string]$APIToken)
    
    $headers = @{
        'Authorization' = "Bearer $APIToken"
        'Content-Type' = 'application/json'
    }
    
    $users = Get-ADUser -Filter {Enabled -eq $true} -Properties GivenName, Surname, EmailAddress, Department, Title
    
    foreach ($user in $users) {
        $userData = @{
            firstName = $user.GivenName
            lastName = $user.Surname
            email = $user.EmailAddress
            department = $user.Department
            title = $user.Title
            externalId = $user.SamAccountName
        }
        
        try {
            Invoke-RestMethod -Uri "$ExternalAPIUrl/users" -Method Post -Headers $headers -Body ($userData | ConvertTo-Json)
            Write-Host "Utilisateur synchronisé: $($user.Name)"
        } catch {
            Write-Warning "Erreur de synchronisation pour $($user.Name): $($_.Exception.Message)"
        }
    }
}
```

### 6.2 Gestion du cycle de vie des comptes

```powershell
# Automatisation du départ d'employés
function Invoke-EmployeeOffboarding {
    param(
        [string]$UserName,
        [bool]$RemoveFromGroups = $true,
        [bool]$DisableAccount = $true,
        [bool]$ResetPassword = $true,
        [string]$OffboardingOU = "OU=Disabled Users,DC=domain,DC=com"
    )
    
    try {
        # Désactiver le compte
        if ($DisableAccount) {
            Disable-ADAccount -Identity $UserName
            Write-Host "✓ Compte désactivé: $UserName"
        }
        
        # Réinitialiser le mot de passe
        if ($ResetPassword) {
            $randomPassword = "Offboarded$(Get-Random -Maximum 9999)!"
            Set-ADAccountPassword -Identity $UserName -NewPassword (ConvertTo-SecureString $randomPassword -AsPlainText -Force)
            Write-Host "✓ Mot de passe réinitialisé"
        }
        
        # Retirer des groupes
        if ($RemoveFromGroups) {
            $userGroups = Get-ADPrincipalGroupMembership -Identity $UserName | 
                Where-Object { $_.Name -ne "Domain Users" }
            
            foreach ($group in $userGroups) {
                Remove-ADGroupMember -Identity $group -Members $UserName -Confirm:$false
                Write-Host "✓ Retiré du groupe: $($group.Name)"
            }
        }
        
        # Déplacer vers OU désactivés
        if ($OffboardingOU) {
            Move-ADObject -Identity (Get-ADUser -Identity $UserName).DistinguishedName -TargetPath $OffboardingOU
            Write-Host "✓ Déplacé vers OU désactivés"
        }
        
        # Journaliser l'action
        Write-EventLog -LogName Application -Source "ADAutomation" -EventId 1001 -EntryType Information -Message "Offboarding terminé pour $UserName"
        
        Write-Host "✓ Offboarding terminé avec succès pour $UserName"
        
    } catch {
        Write-Error "Erreur lors de l'offboarding de $UserName : $($_.Exception.Message)"
        throw
    }
}

# Utilisation
Invoke-EmployeeOffboarding -UserName "leaving.employee"
```

## Conclusion : AD comme infrastructure programmable

PowerShell transforme Active Directory d'un système d'administration traditionnel en une infrastructure hautement programmable où l'automatisation devient la norme. Des tâches répétitives de gestion des utilisateurs aux politiques de sécurité complexes, tout devient scriptable et évolutif.

Dans le prochain chapitre, nous explorerons les fonctionnalités avancées de PowerShell, incluant les jobs, les workflows, et l'intégration avec des technologies cloud comme Azure AD.

---

**Exercice pratique :** Créez un système complet de gestion du cycle de vie des employés qui :
1. Automatise l'onboarding des nouveaux employés
2. Gère les mises à jour de poste/département
3. Implémente l'offboarding sécurisé
4. Génère des rapports d'audit
5. S'intègre avec des systèmes RH externes

**Challenge avancé :** Développez un module PowerShell pour la gestion des autorisations basées sur les rôles (RBAC) dans Active Directory, incluant l'analyse d'impact et les contrôles de séparation des pouvoirs.

**Réflexion :** Comment PowerShell change-t-il l'approche traditionnelle de l'administration Active Directory, et quels sont les impacts sur la sécurité et l'efficacité opérationnelle ?

