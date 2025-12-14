# Chapitre 125 - Administration réseau avec PowerShell

> "PowerShell a fait de l'administration réseau ce que les routeurs ont fait du réseau : quelque chose de programmable, automatisable et intelligent." - Architecte réseau

## Introduction : Le réseau programmable

L'administration réseau sous Windows a traditionnellement reposé sur des outils disparates : ipconfig, netstat, route, netsh, et diverses interfaces graphiques. PowerShell unifie tout cela dans un environnement cohérent, offrant des cmdlets puissants pour gérer les configurations réseau, diagnostiquer les problèmes, et automatiser les tâches complexes.

Dans ce chapitre, nous explorerons comment PowerShell révolutionne l'administration réseau, de la configuration basique aux scénarios avancés d'automatisation et de monitoring.

## Section 1 : Configuration réseau de base

### 1.1 Inspection de la configuration réseau

```powershell
# Informations sur les adaptateurs réseau
Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, MacAddress, LinkSpeed

# Configurations IP détaillées
Get-NetIPConfiguration | Format-Table -AutoSize

# Adresses IP
Get-NetIPAddress | Where-Object { $_.AddressFamily -eq 'IPv4' } | 
    Select-Object InterfaceAlias, IPAddress, PrefixLength, AddressState

# Passerelles par défaut
Get-NetRoute | Where-Object { $_.DestinationPrefix -eq '0.0.0.0/0' } | 
    Select-Object InterfaceAlias, NextHop, RouteMetric

# Serveurs DNS
Get-DnsClientServerAddress | Select-Object InterfaceAlias, ServerAddresses

# Informations complètes sur un adaptateur
Get-NetAdapter -Name "Ethernet" | Get-NetIPConfiguration | Format-List *
```

### 1.2 Configuration IP statique/dynamique

```powershell
# Configuration IP statique
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "192.168.1.100" -PrefixLength 24 -DefaultGateway "192.168.1.1"

# Changer une adresse IP existante
Set-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "192.168.1.101" -PrefixLength 24

# Configuration DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("8.8.8.8", "8.8.4.4")

# Activer DHCP
Set-NetIPInterface -InterfaceAlias "Ethernet" -Dhcp Enabled
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses

# Renommer un adaptateur
Rename-NetAdapter -Name "Ethernet" -NewName "LAN Connection"

# Désactiver/activer un adaptateur
Disable-NetAdapter -Name "Ethernet" -Confirm:$false
Enable-NetAdapter -Name "Ethernet"
```

### 1.3 Gestion des routes

```powershell
# Afficher la table de routage
Get-NetRoute | Where-Object { $_.AddressFamily -eq 'IPv4' } | 
    Select-Object InterfaceAlias, DestinationPrefix, NextHop, RouteMetric | Format-Table -AutoSize

# Ajouter une route statique
New-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "192.168.2.0/24" -NextHop "192.168.1.254"

# Route persistante
New-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "10.0.0.0/8" -NextHop "192.168.1.1" -PolicyStore "PersistentStore"

# Supprimer une route
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "192.168.2.0/24" -Confirm:$false

# Routes pour plusieurs interfaces
$routes = @(
    @{Prefix="10.10.0.0/16"; Gateway="192.168.1.254"; Metric=10},
    @{Prefix="10.20.0.0/16"; Gateway="192.168.1.253"; Metric=20}
)

foreach ($route in $routes) {
    New-NetRoute -InterfaceAlias "Ethernet" `
        -DestinationPrefix $route.Prefix `
        -NextHop $route.Gateway `
        -RouteMetric $route.Metric
}
```

## Section 2 : Diagnostic et dépannage réseau

### 2.1 Tests de connectivité

```powershell
# Test ping simple
Test-Connection -ComputerName "google.com"

# Test ping détaillé
Test-Connection -ComputerName "192.168.1.1" -Count 4 -Delay 1 -BufferSize 32

# Test de port spécifique
Test-NetConnection -ComputerName "example.com" -Port 80

# Diagnostic complet
Test-NetConnection -ComputerName "google.com" -InformationLevel Detailed

# Test de plusieurs hôtes
$hosts = @("google.com", "github.com", "stackoverflow.com")
foreach ($host in $hosts) {
    $result = Test-NetConnection -ComputerName $host -Port 443
    [PSCustomObject]@{
        Host = $host
        Pingable = $result.PingSucceeded
        PortOpen = $result.TcpTestSucceeded
        RemoteAddress = $result.RemoteAddress
    }
}
```

### 2.2 Analyse du trafic réseau

```powershell
# Statistiques des adaptateurs
Get-NetAdapterStatistics | Select-Object Name, ReceivedBytes, SentBytes, 
    @{Name='ReceivedMB'; Expression={[math]::Round($_.ReceivedBytes/1MB, 2)}},
    @{Name='SentMB'; Expression={[math]::Round($_.SentBytes/1MB, 2)}}

# Connexions TCP actives
Get-NetTCPConnection | Where-Object { $_.State -eq 'Established' } | 
    Select-Object LocalAddress, LocalPort, RemoteAddress, RemotePort, State, OwningProcess |
    Format-Table -AutoSize

# Ports ouverts
Get-NetTCPConnection | Where-Object { $_.State -eq 'Listen' } | 
    Select-Object LocalAddress, LocalPort, OwningProcess |
    Sort-Object LocalPort

# Statistiques par processus
Get-NetTCPConnection | Group-Object OwningProcess | 
    Select-Object @{Name='ProcessId'; Expression={$_.Name}}, Count | 
    Sort-Object Count -Descending

# Résolution DNS
Resolve-DnsName -Name "google.com" -Type A
Resolve-DnsName -Name "google.com" -Type MX
Resolve-DnsName -Name "google.com" -Type TXT

# Cache DNS
Get-DnsClientCache | Select-Object Name, Type, Data, TimeToLive | Format-Table -AutoSize
Clear-DnsClientCache
```

### 2.3 Outils de diagnostic avancés

```powershell
# Traceroute avec Test-NetConnection
1..30 | ForEach-Object {
    $result = Test-NetConnection -ComputerName "google.com" -Traceroute
    if ($result) {
        Write-Host "Hop $_ : $($result.RemoteAddress)"
    }
}

# Test de débit (nécessite module NetworkTesting)
# Install-Module -Name NetworkTesting
# Test-NetworkSpeed -ComputerName "fileserver" -Count 10

# Analyse des paquets (nécessite Wireshark ou outils similaires)
# Simulation d'analyse de capture
function Analyze-NetworkCapture {
    param([string]$CaptureFile)
    
    # Simulation de parsing d'un fichier de capture
    $packets = Get-Content $CaptureFile | Where-Object { $_ -match '^\d+:' }
    
    $analysis = $packets | ForEach-Object {
        # Parser les paquets (simulation)
        if ($_ -match 'TCP') {
            [PSCustomObject]@{Type='TCP'; Source='192.168.1.100:1234'; Destination='10.0.0.1:80'}
        } elseif ($_ -match 'UDP') {
            [PSCustomObject]@{Type='UDP'; Source='192.168.1.100:53'; Destination='8.8.8.8:53'}
        } else {
            [PSCustomObject]@{Type='Other'; Source='Unknown'; Destination='Unknown'}
        }
    }
    
    $analysis | Group-Object Type | Select-Object Name, Count
}

# Test de latence réseau
function Test-NetworkLatency {
    param(
        [string]$TargetHost,
        [int]$SampleCount = 10,
        [int]$TimeoutMs = 1000
    )
    
    $results = 1..$SampleCount | ForEach-Object {
        $start = Get-Date
        $ping = Test-Connection -ComputerName $TargetHost -Count 1 -TimeoutSeconds ($TimeoutMs/1000)
        $end = Get-Date
        
        if ($ping.Status -eq 'Success') {
            [PSCustomObject]@{
                Sample = $_
                LatencyMs = [math]::Round(($end - $start).TotalMilliseconds, 2)
                Status = 'Success'
            }
        } else {
            [PSCustomObject]@{
                Sample = $_
                LatencyMs = $null
                Status = 'Failed'
            }
        }
    }
    
    # Statistiques
    $successful = $results | Where-Object Status -eq 'Success'
    if ($successful) {
        $stats = $successful | Measure-Object -Property LatencyMs -Average -Minimum -Maximum
        
        Write-Host "Test de latence vers $TargetHost :"
        Write-Host "  Échantillons réussis: $($successful.Count)/$SampleCount"
        Write-Host "  Latence moyenne: $([math]::Round($stats.Average, 2)) ms"
        Write-Host "  Latence minimale: $([math]::Round($stats.Minimum, 2)) ms"
        Write-Host "  Latence maximale: $([math]::Round($stats.Maximum, 2)) ms"
    }
    
    return $results
}

Test-NetworkLatency -TargetHost "google.com" -SampleCount 20
```

## Section 3 : Gestion du firewall Windows

### 3.1 Inspection des règles firewall

```powershell
# Règles firewall actives
Get-NetFirewallRule | Where-Object { $_.Enabled -eq $true } | 
    Select-Object Name, DisplayName, Direction, Action | Format-Table -AutoSize

# Règles par profil
Get-NetFirewallRule | Group-Object Profile | Select-Object Name, Count

# Règles bloquées/autorisées
Get-NetFirewallRule | Where-Object Action -eq 'Block' | Select-Object Name, DisplayName
Get-NetFirewallRule | Where-Object Action -eq 'Allow' | Select-Object Name, DisplayName

# Profil firewall actuel
Get-NetFirewallProfile | Select-Object Name, Enabled, DefaultInboundAction, DefaultOutboundAction
```

### 3.2 Création et gestion des règles

```powershell
# Autoriser un programme
New-NetFirewallRule -DisplayName "Allow MyApp" `
    -Direction Inbound `
    -Program "C:\MyApp\MyApp.exe" `
    -Action Allow

# Autoriser un port
New-NetFirewallRule -DisplayName "Allow HTTP" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 80 `
    -Action Allow

# Bloquer une adresse IP
New-NetFirewallRule -DisplayName "Block Bad IP" `
    -Direction Inbound `
    -RemoteAddress "192.168.1.100" `
    -Action Block

# Règle avancée pour RDP
New-NetFirewallRule -DisplayName "Allow RDP" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 3389 `
    -Action Allow `
    -Profile Domain,Private `
    -Enabled True

# Modifier une règle existante
Set-NetFirewallRule -DisplayName "Allow HTTP" -Action Block

# Supprimer une règle
Remove-NetFirewallRule -DisplayName "Block Bad IP"
```

### 3.3 Gestion des profils firewall

```powershell
# Configuration des profils
Set-NetFirewallProfile -Profile Domain -Enabled True -DefaultInboundAction Block -DefaultOutboundAction Allow
Set-NetFirewallProfile -Profile Private -Enabled True -DefaultInboundAction Block -DefaultOutboundAction Allow
Set-NetFirewallProfile -Profile Public -Enabled True -DefaultInboundAction Block -DefaultOutboundAction Block

# Journalisation du firewall
Set-NetFirewallProfile -Profile Domain -LogAllowed True -LogBlocked True -LogFileName "C:\Logs\firewall.log"

# Règles temporaires (durée limitée)
New-NetFirewallRule -DisplayName "Temp Rule" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 8080 `
    -Action Allow `
    -Enabled True

# La règle sera automatiquement supprimée après redémarrage
# Pour une durée personnalisée, utiliser des tâches planifiées

# Sauvegarde et restauration des règles
Export-NetFirewallRule -Path "C:\Backup\firewall_rules.wfw"
Import-NetFirewallRule -Path "C:\Backup\firewall_rules.wfw"
```

### 3.4 Monitoring et audit du firewall

```powershell
# Événements firewall récents
Get-WinEvent -LogName "Security" | Where-Object { $_.Id -eq 5152 } | 
    Select-Object TimeCreated, Message | Format-List

# Statistiques des règles firewall
Get-NetFirewallStatistics | Select-Object PolicyStore, TotalRules, ActiveRules

# Règles non utilisées (audit)
$allRules = Get-NetFirewallRule
$usedRules = Get-NetFirewallStatistics | Select-Object -ExpandProperty MatchedRules

$unusedRules = $allRules | Where-Object { $_.Name -notin $usedRules.RuleName }
$unusedRules | Select-Object Name, DisplayName, Direction, Action | Format-Table -AutoSize

# Rapport de sécurité firewall
function Get-FirewallSecurityReport {
    $report = [PSCustomObject]@{
        Timestamp = Get-Date
        Profiles = Get-NetFirewallProfile
        EnabledRules = (Get-NetFirewallRule | Where-Object Enabled -eq $true).Count
        BlockedRules = (Get-NetFirewallRule | Where-Object Action -eq 'Block').Count
        AllowedRules = (Get-NetFirewallRule | Where-Object Action -eq 'Allow').Count
        InboundDefaultAction = (Get-NetFirewallProfile | Select-Object -First 1).DefaultInboundAction
        OutboundDefaultAction = (Get-NetFirewallProfile | Select-Object -First 1).DefaultOutboundAction
    }
    
    # Vérifications de sécurité
    $issues = @()
    
    if ($report.InboundDefaultAction -ne 'Block') {
        $issues += "Action inbound par défaut non sécurisée"
    }
    
    if ((Get-NetFirewallRule | Where-Object { $_.DisplayName -like "*Remote Desktop*" -and $_.Enabled -eq $true }).Count -gt 0) {
        $issues += "RDP potentiellement exposé"
    }
    
    $report | Add-Member -MemberType NoteProperty -Name SecurityIssues -Value $issues
    
    return $report
}

Get-FirewallSecurityReport | Format-List
```

## Section 4 : Administration réseau distante

### 4.1 PowerShell Remoting (WinRM)

```powershell
# Activer WinRM
Enable-PSRemoting -Force

# Configuration WinRM
Set-Item WSMan:\localhost\Client\TrustedHosts -Value "*" -Force
Restart-Service WinRM

# Connexion distante
Enter-PSSession -ComputerName "remote-server"

# Exécution de commandes distantes
Invoke-Command -ComputerName "server1", "server2" -ScriptBlock {
    Get-NetAdapter | Select-Object Name, Status
}

# Sessions persistantes
$session = New-PSSession -ComputerName "remote-server"
Invoke-Command -Session $session -ScriptBlock { Get-Process }
Remove-PSSession -Session $session

# Copie de fichiers distante
Copy-Item -Path "C:\local\file.txt" -Destination "C:\remote\" -ToSession $session
```

### 4.2 Gestion de serveurs distants

```powershell
# Inventaire réseau
$servers = @("web01", "db01", "app01")

$inventory = Invoke-Command -ComputerName $servers -ScriptBlock {
    [PSCustomObject]@{
        ComputerName = $env:COMPUTERNAME
        OSVersion = (Get-WmiObject Win32_OperatingSystem).Caption
        TotalMemoryGB = [math]::Round((Get-WmiObject Win32_ComputerSystem).TotalPhysicalMemory / 1GB, 2)
        CPUCores = (Get-WmiObject Win32_Processor | Measure-Object -Property NumberOfCores -Sum).Sum
        ServicesCount = (Get-Service).Count
        ProcessesCount = (Get-Process).Count
    }
}

$inventory | Format-Table -AutoSize

# Déploiement de configuration
$configuration = @{
    ServicesToStop = @("unwanted-service")
    ServicesToStart = @("required-service")
    RegistrySettings = @{
        "HKLM:\SOFTWARE\MyApp" = @{
            "Version" = "1.0"
            "InstallPath" = "C:\Program Files\MyApp"
        }
    }
}

Invoke-Command -ComputerName $servers -ScriptBlock {
    param($config)
    
    # Arrêter les services indésirables
    foreach ($service in $config.ServicesToStop) {
        Stop-Service -Name $service -ErrorAction SilentlyContinue
        Set-Service -Name $service -StartupType Disabled
    }
    
    # Démarrer les services requis
    foreach ($service in $config.ServicesToStart) {
        Set-Service -Name $service -StartupType Automatic
        Start-Service -Name $service -ErrorAction SilentlyContinue
    }
    
    # Appliquer les paramètres registre
    foreach ($key in $config.RegistrySettings.Keys) {
        if (-not (Test-Path $key)) {
            New-Item -Path $key -Force
        }
        
        foreach ($value in $config.RegistrySettings[$key].Keys) {
            Set-ItemProperty -Path $key -Name $value -Value $config.RegistrySettings[$key][$value]
        }
    }
    
} -ArgumentList $configuration
```

### 4.3 Monitoring réseau distribué

```powershell
# Collecte de métriques réseau sur plusieurs serveurs
function Get-DistributedNetworkMetrics {
    param([string[]]$ComputerNames)
    
    $metrics = Invoke-Command -ComputerName $ComputerNames -ScriptBlock {
        $adapters = Get-NetAdapter | Where-Object Status -eq 'Up'
        $connections = Get-NetTCPConnection | Where-Object State -eq 'Established'
        
        [PSCustomObject]@{
            ComputerName = $env:COMPUTERNAME
            NetworkAdapters = $adapters.Count
            ActiveConnections = $connections.Count
            TotalBytesReceived = ($adapters | Measure-Object -Property ReceivedBytes -Sum).Sum
            TotalBytesSent = ($adapters | Measure-Object -Property SentBytes -Sum).Sum
            DNSServers = (Get-DnsClientServerAddress | Select-Object -First 1).ServerAddresses -join ', '
        }
    }
    
    return $metrics
}

$networkHealth = Get-DistributedNetworkMetrics -ComputerNames "server1", "server2", "server3"
$networkHealth | Format-Table -AutoSize

# Alertes réseau distribuées
function Monitor-NetworkHealth {
    param([string[]]$ComputerNames, [int]$CheckIntervalSeconds = 300)
    
    while ($true) {
        foreach ($computer in $ComputerNames) {
            try {
                $metrics = Invoke-Command -ComputerName $computer -ScriptBlock {
                    # Métriques critiques
                    $ping = Test-Connection -ComputerName "8.8.8.8" -Count 1 -Quiet
                    $adapters = Get-NetAdapter | Where-Object { $_.Status -eq 'Up' }
                    
                    [PSCustomObject]@{
                        Connectivity = $ping
                        ActiveAdapters = $adapters.Count
                        Timestamp = Get-Date
                    }
                }
                
                # Vérifications d'alerte
                if (-not $metrics.Connectivity) {
                    Write-Warning "[$computer] Perte de connectivité réseau détectée"
                }
                
                if ($metrics.ActiveAdapters -eq 0) {
                    Write-Warning "[$computer] Aucun adaptateur réseau actif"
                }
                
            } catch {
                Write-Error "Impossible de contacter $computer : $($_.Exception.Message)"
            }
        }
        
        Start-Sleep -Seconds $CheckIntervalSeconds
    }
}

# Démarrer le monitoring en arrière-plan
Start-Job -ScriptBlock ${function:Monitor-NetworkHealth} -ArgumentList @("server1", "server2") -Name "NetworkMonitor"
```

## Section 5 : Automatisation réseau avancée

### 5.1 Configuration réseau automatique

```powershell
# Fonction de configuration réseau complète
function Set-NetworkConfiguration {
    param(
        [string]$AdapterName,
        [string]$IPAddress,
        [int]$PrefixLength = 24,
        [string]$Gateway,
        [string[]]$DNSServers,
        [string]$NewName = $null
    )
    
    Write-Host "Configuration de l'adaptateur réseau: $AdapterName"
    
    # Désactiver DHCP
    Set-NetIPInterface -InterfaceAlias $AdapterName -Dhcp Disabled
    
    # Supprimer l'adresse IP existante
    Get-NetIPAddress -InterfaceAlias $AdapterName | Remove-NetIPAddress -Confirm:$false
    
    # Configurer la nouvelle adresse IP
    New-NetIPAddress -InterfaceAlias $AdapterName -IPAddress $IPAddress -PrefixLength $PrefixLength -DefaultGateway $Gateway
    
    # Configurer les serveurs DNS
    Set-DnsClientServerAddress -InterfaceAlias $AdapterName -ServerAddresses $DNSServers
    
    # Renommer l'adaptateur si demandé
    if ($NewName) {
        Rename-NetAdapter -Name $AdapterName -NewName $NewName
        $AdapterName = $NewName
    }
    
    # Tester la configuration
    $config = Get-NetIPConfiguration -InterfaceAlias $AdapterName
    Write-Host "Configuration appliquée:"
    Write-Host "  Adresse IP: $($config.IPv4Address.IPAddress)"
    Write-Host "  Passerelle: $($config.IPv4DefaultGateway.NextHop)"
    Write-Host "  DNS: $($config.DNSServer.ServerAddresses -join ', ')"
    
    # Test de connectivité
    $test = Test-NetConnection -ComputerName "google.com"
    if ($test.PingSucceeded) {
        Write-Host "✓ Connectivité Internet vérifiée"
    } else {
        Write-Warning "✗ Problème de connectivité détecté"
    }
}

# Exemple d'utilisation
Set-NetworkConfiguration -AdapterName "Ethernet" `
    -IPAddress "192.168.1.100" `
    -PrefixLength 24 `
    -Gateway "192.168.1.1" `
    -DNSServers @("8.8.8.8", "8.8.4.4") `
    -NewName "LAN Connection"
```

### 5.2 Gestion de VLAN et réseaux virtuels

```powershell
# Configuration VLAN
function New-VirtualNetwork {
    param(
        [string]$AdapterName,
        [int]$VLANId,
        [string]$IPAddress,
        [int]$PrefixLength = 24
    )
    
    # Créer l'adaptateur VLAN
    $vlanAdapter = New-NetLbfoTeam -Name "VLAN${VLANId}Team" -TeamMembers $AdapterName -TeamingMode SwitchIndependent -LoadBalancingAlgorithm Dynamic
    
    # Configurer la VLAN
    Set-NetLbfoTeam -Name $vlanAdapter.Name -VlanID $VLANId
    
    # Configurer l'adresse IP
    New-NetIPAddress -InterfaceAlias $vlanAdapter.Name -IPAddress $IPAddress -PrefixLength $PrefixLength
    
    Write-Host "Réseau virtuel VLAN $VLANId créé sur $AdapterName"
}

# Gestion des commutateurs virtuels Hyper-V
if (Get-WindowsFeature -Name Hyper-V | Where-Object Installed) {
    # Créer un commutateur virtuel
    New-VMSwitch -Name "ExternalSwitch" -NetAdapterName "Ethernet" -AllowManagementOS $true
    
    # Lister les commutateurs
    Get-VMSwitch | Select-Object Name, SwitchType, NetAdapterInterfaceDescription
    
    # Configuration avancée
    Set-VMSwitch -Name "ExternalSwitch" -AllowManagementOS $false
}
```

### 5.3 Intégration avec des outils tiers

```powershell
# Intégration avec Nmap (nécessite nmap installé)
function Invoke-NmapScan {
    param(
        [string]$Target,
        [string]$ScanType = "TCP",
        [string]$Ports = "1-1024"
    )
    
    $nmapPath = "C:\Program Files\Nmap\nmap.exe"
    
    if (-not (Test-Path $nmapPath)) {
        Write-Error "Nmap n'est pas installé ou introuvable"
        return
    }
    
    $arguments = @("-s$ScanType", "-p", $Ports, $Target, "-oX", "-")
    
    $result = & $nmapPath $arguments
    
    # Parser le résultat XML
    $xml = [xml]$result
    $hosts = $xml.nmaprun.host
    
    foreach ($host in $hosts) {
        $address = $host.address.addr
        $ports = $host.ports.port | Where-Object { $_.state.state -eq 'open' }
        
        [PSCustomObject]@{
            IPAddress = $address
            OpenPorts = ($ports | ForEach-Object { "$($_.portid)/$($_.protocol)" }) -join ', '
            PortCount = $ports.Count
        }
    }
}

# Exemple d'utilisation
Invoke-NmapScan -Target "192.168.1.0/24" -ScanType "S" -Ports "22,80,443" | Format-Table -AutoSize

# Intégration avec Wireshark (simulation)
function Start-NetworkCapture {
    param(
        [string]$Interface = "Ethernet",
        [int]$DurationSeconds = 30,
        [string]$Filter = ""
    )
    
    $dumpcap = "C:\Program Files\Wireshark\dumpcap.exe"
    $outputFile = "$env:TEMP\capture_$(Get-Date -Format 'yyyyMMdd_HHmmss').pcapng"
    
    if (-not (Test-Path $dumpcap)) {
        Write-Error "Wireshark n'est pas installé"
        return
    }
    
    $arguments = @("-i", $Interface, "-a", "duration:$DurationSeconds", "-w", $outputFile)
    if ($Filter) {
        $arguments += @("-f", $Filter)
    }
    
    Write-Host "Capture réseau démarrée sur $Interface pour $DurationSeconds secondes..."
    & $dumpcap $arguments
    
    if (Test-Path $outputFile) {
        Write-Host "Capture sauvegardée: $outputFile"
        return $outputFile
    }
}

# Capture avec filtre HTTP
Start-NetworkCapture -Interface "Ethernet" -DurationSeconds 60 -Filter "tcp port 80 or tcp port 443"
```

## Conclusion : Le réseau comme code

PowerShell transforme l'administration réseau d'une discipline manuelle et complexe en un processus programmable, automatisable et maintenable. Des configurations de base aux architectures réseau distribuées, PowerShell offre les outils nécessaires pour traiter le réseau comme du code.

Dans le prochain chapitre, nous explorerons PowerShell pour l'administration Active Directory, découvrant comment gérer les utilisateurs, groupes, politiques, et l'infrastructure d'identité.

---

**Exercice pratique :** Créez un script d'audit réseau complet qui :
1. Inventorie tous les adaptateurs réseau
2. Teste la connectivité vers des hôtes critiques
3. Vérifie les règles firewall importantes
4. Analyse les connexions TCP actives
5. Génère un rapport de sécurité réseau

**Challenge avancé :** Développez un module PowerShell pour la gestion centralisée de configurations réseau sur plusieurs serveurs, incluant rollback automatique en cas d'erreur.

**Réflexion :** Comment PowerShell change-t-il votre approche de l'administration réseau par rapport aux outils traditionnels comme netsh ou les interfaces graphiques ?

