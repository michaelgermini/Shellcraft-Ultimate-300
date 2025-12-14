# Chapitre 137 - PowerShell pour la virtualisation et les conteneurs

> "La virtualisation et les conteneurs ne sont pas seulement des technologies d'infrastructure, ce sont les pinceaux avec lesquels PowerShell peint des paysages d'automatisation infiniment scalables et élastiques." - Citation inspirée des paradigmes de l'infrastructure as code

## Introduction : PowerShell comme orchestrateur d'infrastructures modernes

PowerShell transcende son rôle traditionnel d'interface système pour devenir l'orchestrateur par excellence des environnements virtualisés et conteneurisés. En maîtrisant les cmdlets Hyper-V, Docker, et Kubernetes, les administrateurs PowerShell créent des infrastructures dynamiques capables de s'adapter instantanément aux demandes changeantes des applications modernes.

Dans ce chapitre, nous explorerons la gestion complète des environnements virtualisés et conteneurisés avec PowerShell.

## Section 1 : Gestion avancée d'Hyper-V

### 1.1 Automatisation complète d'Hyper-V

**Framework de gestion Hyper-V avancé :**
```powershell
# Framework complet de gestion Hyper-V
class HyperVManager {
    [string]$HyperVHost
    [PSCredential]$Credential
    [System.Collections.Generic.Dictionary[string, PSObject]]$VMs
    [hashtable]$HostCapabilities
    [System.Collections.Generic.List[PSCustomObject]]$OperationsLog

    HyperVManager([string]$hostName = $env:COMPUTERNAME, [PSCredential]$credential = $null) {
        $this.HyperVHost = $hostName
        $this.Credential = $credential
        $this.VMs = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.OperationsLog = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.InitializeHostCapabilities()
        $this.RefreshVMInventory()
    }

    hidden [void]InitializeHostCapabilities() {
        Write-Host "Initializing Hyper-V host capabilities..." -ForegroundColor Cyan

        $scriptBlock = {
            $capabilities = @{
                HyperVEnabled = (Get-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V).State -eq 'Enabled'
                VirtualizationEnabled = (Get-CimInstance -ClassName Win32_ComputerSystem).HypervisorPresent
                VMCount = (Get-VM).Count
                TotalMemoryGB = [math]::Round((Get-CimInstance -ClassName Win32_ComputerSystem).TotalPhysicalMemory / 1GB, 2)
                AvailableMemoryGB = [math]::Round((Get-CimInstance -ClassName Win32_OperatingSystem).FreePhysicalMemory / 1MB, 2)
                ProcessorCount = (Get-CimInstance -ClassName Win32_Processor).Count
                ProcessorCores = (Get-CimInstance -ClassName Win32_Processor | Measure-Object -Property NumberOfCores -Sum).Sum
                StoragePools = @()
            }

            # Informations sur le stockage
            $capabilities.StoragePools = Get-VirtualDisk | Group-Object PSComputerName | ForEach-Object {
                @{
                    ComputerName = $_.Name
                    VirtualDisks = $_.Count
                    TotalSizeGB = [math]::Round(($_.Group | Measure-Object -Property Size -Sum).Sum / 1GB, 2)
                }
            }

            return $capabilities
        }

        if ($this.HyperVHost -eq $env:COMPUTERNAME) {
            $this.HostCapabilities = & $scriptBlock
        } else {
            $this.HostCapabilities = Invoke-Command -ComputerName $this.HyperVHost -Credential $this.Credential -ScriptBlock $scriptBlock
        }

        $this.LogOperation("HostCapabilitiesInitialized", "Host capabilities initialized", @{ Host = $this.HyperVHost })
    }

    hidden [void]RefreshVMInventory() {
        Write-Host "Refreshing VM inventory..." -ForegroundColor Gray

        $scriptBlock = {
            $vms = Get-VM | ForEach-Object {
                $vm = $_
                $vmInfo = @{
                    Name = $vm.Name
                    State = $vm.State.ToString()
                    Status = $vm.Status
                    CPUUsage = $vm.CPUUsage
                    MemoryAssignedMB = $vm.MemoryAssigned / 1MB
                    Uptime = $vm.Uptime
                    CreationTime = $vm.CreationTime
                    Generation = $vm.Generation
                    Version = $vm.Version
                    AutomaticStartAction = $vm.AutomaticStartAction.ToString()
                    AutomaticStopAction = $vm.AutomaticStopAction.ToString()
                }

                # Informations sur les disques durs virtuels
                $hardDrives = Get-VMHardDiskDrive -VMName $vm.Name
                $vmInfo.HardDrives = $hardDrives | ForEach-Object {
                    @{
                        Path = $_.Path
                        SizeGB = [math]::Round((Get-VHD -Path $_.Path).Size / 1GB, 2)
                        Type = (Get-VHD -Path $_.Path).VhdType
                        Attached = $true
                    }
                }

                # Informations sur les adaptateurs réseau
                $networkAdapters = Get-VMNetworkAdapter -VMName $vm.Name
                $vmInfo.NetworkAdapters = $networkAdapters | ForEach-Object {
                    @{
                        Name = $_.Name
                        SwitchName = $_.SwitchName
                        MacAddress = $_.MacAddress
                        IPAddresses = $_.IPAddresses
                    }
                }

                # Informations sur les snapshots
                $snapshots = Get-VMSnapshot -VMName $vm.Name
                $vmInfo.Snapshots = $snapshots | ForEach-Object {
                    @{
                        Name = $_.Name
                        CreationTime = $_.CreationTime
                        ParentSnapshotName = $_.ParentSnapshotName
                    }
                }

                $vmInfo
            }

            return $vms
        }

        $vmData = if ($this.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock
        } else {
            Invoke-Command -ComputerName $this.HyperVHost -Credential $this.Credential -ScriptBlock $scriptBlock
        }

        $this.VMs.Clear()
        foreach ($vm in $vmData) {
            $this.VMs[$vm.Name] = [PSCustomObject]$vm
        }

        $this.LogOperation("VMInventoryRefreshed", "VM inventory refreshed", @{ VMCount = $this.VMs.Count })
    }

    [PSCustomObject]CreateVM([hashtable]$vmConfig) {
        Write-Host "Creating VM: $($vmConfig.Name)" -ForegroundColor Yellow

        # Validation de la configuration
        $this.ValidateVMConfiguration($vmConfig)

        $scriptBlock = {
            param($config)

            # Création de la VM
            $vm = New-VM -Name $config.Name -MemoryStartupBytes $config.MemoryMB * 1MB -Generation $config.Generation

            # Configuration du processeur
            if ($config.ProcessorCount) {
                Set-VMProcessor -VMName $config.Name -Count $config.ProcessorCount
            }

            # Configuration du réseau
            if ($config.SwitchName) {
                $networkAdapter = Add-VMNetworkAdapter -VMName $config.Name -SwitchName $config.SwitchName
                if ($config.VLanId) {
                    Set-VMNetworkAdapterVlan -VMName $config.Name -VlanId $config.VLanId -Access
                }
            }

            # Création et attachement du disque dur virtuel
            if ($config.VHDPath -and $config.VHDSizeGB) {
                $vhdPath = Join-Path $config.VHDPath "$($config.Name).vhdx"
                New-VHD -Path $vhdPath -SizeBytes ($config.VHDSizeGB * 1GB) -Dynamic
                Add-VMHardDiskDrive -VMName $config.Name -Path $vhdPath
            }

            # Configuration du DVD si spécifié
            if ($config.ISOPath) {
                Add-VMDvdDrive -VMName $config.Name
                Set-VMDvdDrive -VMName $config.Name -Path $config.ISOPath
            }

            # Configuration du démarrage automatique
            if ($config.AutomaticStart) {
                Set-VM -Name $config.Name -AutomaticStartAction Start
            }

            return Get-VM -Name $config.Name
        }

        $result = if ($this.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock $vmConfig
        } else {
            Invoke-Command -ComputerName $this.HyperVHost -Credential $this.Credential -ScriptBlock $scriptBlock -ArgumentList $vmConfig
        }

        $this.RefreshVMInventory()
        $this.LogOperation("VMCreated", "VM created successfully", @{ VMName = $vmConfig.Name; Host = $this.HyperVHost })

        return $result
    }

    hidden [void]ValidateVMConfiguration([hashtable]$config) {
        $requiredFields = @('Name', 'MemoryMB')
        foreach ($field in $requiredFields) {
            if (-not $config.ContainsKey($field)) {
                throw "VM configuration missing required field: $field"
            }
        }

        if ($config.MemoryMB -gt ($this.HostCapabilities.AvailableMemoryGB * 1024)) {
            throw "Requested memory ($($config.MemoryMB)MB) exceeds available host memory"
        }

        if ($this.VMs.ContainsKey($config.Name)) {
            throw "VM with name '$($config.Name)' already exists"
        }
    }

    [void]StartVM([string]$vmName) {
        $this.ExecuteVMOperation($vmName, "Start", {
            param($name)
            Start-VM -Name $name
        })
    }

    [void]StopVM([string]$vmName, [switch]$Force) {
        $this.ExecuteVMOperation($vmName, "Stop", {
            param($name, $force)
            if ($force) {
                Stop-VM -Name $name -Force
            } else {
                Stop-VM -Name $name -TurnOff
            }
        }, $Force)
    }

    [void]RestartVM([string]$vmName) {
        $this.ExecuteVMOperation($vmName, "Restart", {
            param($name)
            Restart-VM -Name $name
        })
    }

    [void]RemoveVM([string]$vmName, [switch]$IncludeDisks) {
        if (-not $this.VMs.ContainsKey($vmName)) {
            throw "VM '$vmName' not found"
        }

        Write-Host "Removing VM: $vmName" -ForegroundColor Red

        $scriptBlock = {
            param($name, $includeDisks)

            $vm = Get-VM -Name $name

            # Arrêt de la VM si elle est en cours d'exécution
            if ($vm.State -eq 'Running') {
                Stop-VM -Name $name -Force
            }

            # Suppression des snapshots
            Get-VMSnapshot -VMName $name | ForEach-Object {
                Remove-VMSnapshot -VMSnapshot $_ -Confirm:$false
            }

            # Suppression des disques si demandé
            if ($includeDisks) {
                Get-VMHardDiskDrive -VMName $name | ForEach-Object {
                    Remove-Item -Path $_.Path -Force
                }
            }

            # Suppression de la VM
            Remove-VM -Name $name -Force
        }

        if ($this.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock $vmName $IncludeDisks
        } else {
            Invoke-Command -ComputerName $this.HyperVHost -Credential $this.Credential -ScriptBlock $scriptBlock -ArgumentList $vmName, $IncludeDisks
        }

        $this.VMs.Remove($vmName)
        $this.LogOperation("VMRemoved", "VM removed successfully", @{ VMName = $vmName; IncludeDisks = $IncludeDisks })
    }

    hidden [void]ExecuteVMOperation([string]$vmName, [string]$operation, [scriptblock]$action, $additionalArgs = $null) {
        if (-not $this.VMs.ContainsKey($vmName)) {
            throw "VM '$vmName' not found"
        }

        Write-Host "${operation}ing VM: $vmName" -ForegroundColor Yellow

        $scriptBlock = {
            param($name, $op, $args)
            & $action $name $args
        }

        if ($this.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock $vmName $operation $additionalArgs
        } else {
            Invoke-Command -ComputerName $this.HyperVHost -Credential $this.Credential -ScriptBlock $scriptBlock -ArgumentList $vmName, $operation, $additionalArgs
        }

        $this.RefreshVMInventory()
        $this.LogOperation("VMOperation", "VM operation completed", @{ VMName = $vmName; Operation = $operation })
    }

    [PSCustomObject[]]GetVMs([string]$filter = "*") {
        return $this.VMs.Values | Where-Object { $_.Name -like $filter }
    }

    [hashtable]GetHostMetrics() {
        return $this.HostCapabilities.Clone()
    }

    [PSCustomObject[]]GetOperationsLog([int]$lastHours = 24) {
        $cutoff = (Get-Date).AddHours(-$lastHours)
        return $this.OperationsLog | Where-Object { $_.Timestamp -ge $cutoff } | Sort-Object Timestamp -Descending
    }

    hidden [void]LogOperation([string]$operation, [string]$message, [hashtable]$details = @{}) {
        $logEntry = [PSCustomObject]@{
            Timestamp = Get-Date
            Operation = $operation
            Message = $message
            Details = $details
            Host = $this.HyperVHost
        }

        $this.OperationsLog.Add($logEntry)
    }

    [string]GenerateReport() {
        $vmCount = $this.VMs.Count
        $runningVMs = ($this.VMs.Values | Where-Object { $_.State -eq "Running" }).Count
        $stoppedVMs = ($this.VMs.Values | Where-Object { $_.State -eq "Off" }).Count

        $totalMemoryUsed = ($this.VMs.Values | Measure-Object -Property MemoryAssignedMB -Sum).Sum
        $totalCPUUsage = ($this.VMs.Values | Where-Object { $_.State -eq "Running" } | Measure-Object -Property CPUUsage -Sum).Sum

        $report = @"
HYPER-V HOST REPORT
===================
Host: $($this.HyperVHost)
Generated: $(Get-Date)

HOST CAPABILITIES
=================
Hyper-V Enabled: $($this.HostCapabilities.HyperVEnabled)
Virtualization: $($this.HostCapabilities.VirtualizationEnabled)
Total Memory: $($this.HostCapabilities.TotalMemoryGB) GB
Available Memory: $($this.HostCapabilities.AvailableMemoryGB) GB
Processors: $($this.HostCapabilities.ProcessorCount)
Cores: $($this.HostCapabilities.ProcessorCores)

VM STATISTICS
=============
Total VMs: $vmCount
Running VMs: $runningVMs
Stopped VMs: $stoppedVMs
Memory Used: $([math]::Round($totalMemoryUsed / 1024, 2)) GB
CPU Usage: $totalCPUUsage%

VM DETAILS
==========
"@

        foreach ($vm in $this.VMs.Values | Sort-Object Name) {
            $report += @"

$($vm.Name)
  State: $($vm.State)
  CPU Usage: $($vm.CPUUsage)%
  Memory: $([math]::Round($vm.MemoryAssignedMB / 1024, 2)) GB
  Uptime: $($vm.Uptime)
  Hard Drives: $($vm.HardDrives.Count)
  Network Adapters: $($vm.NetworkAdapters.Count)
  Snapshots: $($vm.Snapshots.Count)
"@
        }

        $report += @"

RECENT OPERATIONS (Last 24h)
===========================
$($this.GetOperationsLog(24) | Select-Object -First 10 | ForEach-Object { "$($_.Timestamp): $($_.Operation) - $($_.Message)" } | Out-String)
"@

        return $report
    }
}

# Utilisation du framework Hyper-V
$hvManager = [HyperVManager]::new()

# Affichage des capacités de l'hôte
$hostMetrics = $hvManager.GetHostMetrics()
Write-Host "=== HYPER-V HOST CAPABILITIES ===" -ForegroundColor Cyan
Write-Host "Host: $($hvManager.HyperVHost)"
Write-Host "Hyper-V Enabled: $($hostMetrics.HyperVEnabled)"
Write-Host "Available Memory: $($hostMetrics.AvailableMemoryGB) GB"
Write-Host "VM Count: $($hostMetrics.VMCount)"

# Création d'une nouvelle VM
$vmConfig = @{
    Name = "TestVM-$(Get-Date -Format 'yyyyMMdd_HHmmss')"
    MemoryMB = 2048
    Generation = 2
    ProcessorCount = 2
    SwitchName = "Default Switch"
    VHDPath = "C:\VMs"
    VHDSizeGB = 50
    ISOPath = "C:\ISO\WindowsServer.iso"
    AutomaticStart = $true
}

try {
    $newVM = $hvManager.CreateVM($vmConfig)
    Write-Host "VM created successfully: $($newVM.Name)" -ForegroundColor Green
} catch {
    Write-Warning "Failed to create VM: $_"
}

# Gestion des VMs existantes
$vms = $hvManager.GetVMs()
Write-Host "`n=== VIRTUAL MACHINES ===" -ForegroundColor Yellow
foreach ($vm in $vms) {
    Write-Host "$($vm.Name): $($vm.State) - CPU: $($vm.CPUUsage)%, Memory: $([math]::Round($vm.MemoryAssignedMB / 1024, 2)) GB"
}

# Génération du rapport
$report = $hvManager.GenerateReport()
$report | Out-File -FilePath "HyperVReport_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nHyper-V management demonstration completed!" -ForegroundColor Green
```

### 1.2 Gestion avancée des snapshots et migrations

**Système de snapshots intelligents et migration automatisée :**
```powershell
# Système avancé de gestion des snapshots Hyper-V
class HyperVSnapshotManager {
    [HyperVManager]$HyperVManager
    [System.Collections.Generic.Dictionary[string, PSObject[]]]$Snapshots
    [hashtable]$SnapshotPolicies
    [System.Collections.Generic.List[PSCustomObject]]$SnapshotOperations

    HyperVSnapshotManager([HyperVManager]$hvManager) {
        $this.HyperVManager = $hvManager
        $this.Snapshots = [System.Collections.Generic.Dictionary[string, PSObject[]]]::new()
        $this.SnapshotOperations = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.InitializeSnapshotPolicies()
        $this.RefreshSnapshots()
    }

    hidden [void]InitializeSnapshotPolicies() {
        $this.SnapshotPolicies = @{
            # Politique pour les serveurs de développement
            Development = @{
                MaxSnapshots = 10
                RetentionDays = 7
                AutoCleanup = $true
                CompressionEnabled = $false
                NamingConvention = "Dev_{VM}_{Date}_{Type}"
            }
            # Politique pour les serveurs de production
            Production = @{
                MaxSnapshots = 5
                RetentionDays = 30
                AutoCleanup = $true
                CompressionEnabled = $true
                NamingConvention = "Prod_{VM}_{Date}_{Type}"
            }
            # Politique pour les serveurs de test
            Testing = @{
                MaxSnapshots = 20
                RetentionDays = 3
                AutoCleanup = $true
                CompressionEnabled = $false
                NamingConvention = "Test_{VM}_{Date}_{Type}"
            }
        }
    }

    hidden [void]RefreshSnapshots() {
        Write-Host "Refreshing snapshots inventory..." -ForegroundColor Gray

        $scriptBlock = {
            $vms = Get-VM
            $snapshotData = @{}

            foreach ($vm in $vms) {
                $snapshots = Get-VMSnapshot -VMName $vm.Name | Sort-Object CreationTime -Descending
                $snapshotData[$vm.Name] = $snapshots
            }

            return $snapshotData
        }

        $snapshotData = if ($this.HyperVManager.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock
        } else {
            Invoke-Command -ComputerName $this.HyperVManager.HyperVHost -Credential $this.HyperVManager.Credential -ScriptBlock $scriptBlock
        }

        $this.Snapshots.Clear()
        foreach ($vmName in $snapshotData.Keys) {
            $this.Snapshots[$vmName] = $snapshotData[$vmName]
        }
    }

    [PSObject]CreateSnapshot([string]$vmName, [string]$snapshotType = "Manual", [hashtable]$metadata = @{}) {
        if (-not $this.HyperVManager.VMs.ContainsKey($vmName)) {
            throw "VM '$vmName' not found"
        }

        # Détermination de la politique à appliquer
        $policy = $this.DeterminePolicy($vmName, $metadata)

        # Génération du nom du snapshot
        $snapshotName = $this.GenerateSnapshotName($vmName, $snapshotType, $policy)

        Write-Host "Creating snapshot for VM: $vmName (Type: $snapshotType)" -ForegroundColor Yellow

        $scriptBlock = {
            param($vm, $name, $notes)

            $snapshot = Checkpoint-VM -Name $vm -SnapshotName $name

            # Ajout de métadonnées dans les notes
            if ($notes) {
                $snapshot.Notes = $notes | ConvertTo-Json
            }

            return $snapshot
        }

        $notes = @{
            Type = $snapshotType
            CreatedBy = $env:USERNAME
            Policy = $policy.Name ?? "Default"
            Timestamp = Get-Date
            Metadata = $metadata
        } | ConvertTo-Json

        $snapshot = if ($this.HyperVManager.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock $vmName $snapshotName $notes
        } else {
            Invoke-Command -ComputerName $this.HyperVManager.HyperVHost -Credential $this.HyperVManager.Credential -ScriptBlock $scriptBlock -ArgumentList $vmName, $snapshotName, $notes
        }

        $this.RefreshSnapshots()

        # Nettoyage automatique selon la politique
        $this.ApplyRetentionPolicy($vmName, $policy)

        $this.LogSnapshotOperation("Created", $vmName, $snapshotName, @{ Policy = $policy.Name ?? "Default"; Type = $snapshotType })

        return $snapshot
    }

    [void]RestoreSnapshot([string]$vmName, [string]$snapshotName) {
        if (-not $this.Snapshots.ContainsKey($vmName)) {
            throw "No snapshots found for VM '$vmName'"
        }

        $snapshot = $this.Snapshots[$vmName] | Where-Object { $_.Name -eq $snapshotName } | Select-Object -First 1
        if (-not $snapshot) {
            throw "Snapshot '$snapshotName' not found for VM '$vmName'"
        }

        Write-Host "Restoring snapshot for VM: $vmName (Snapshot: $snapshotName)" -ForegroundColor Red

        # Arrêt de la VM si elle est en cours d'exécution
        $this.HyperVManager.StopVM($vmName, $true)

        $scriptBlock = {
            param($vm, $snap)
            Restore-VMSnapshot -VMName $vm -VMSnapshotName $snap -Confirm:$false
        }

        if ($this.HyperVManager.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock $vmName $snapshotName
        } else {
            Invoke-Command -ComputerName $this.HyperVManager.HyperVHost -Credential $this.HyperVManager.Credential -ScriptBlock $scriptBlock -ArgumentList $vmName, $snapshotName
        }

        $this.LogSnapshotOperation("Restored", $vmName, $snapshotName, @{ Action = "Restore" })

        Write-Host "Snapshot restored. VM is now stopped. Start it manually if needed." -ForegroundColor Yellow
    }

    [void]RemoveSnapshot([string]$vmName, [string]$snapshotName) {
        if (-not $this.Snapshots.ContainsKey($vmName)) {
            throw "No snapshots found for VM '$vmName'"
        }

        $snapshot = $this.Snapshots[$vmName] | Where-Object { $_.Name -eq $snapshotName } | Select-Object -First 1
        if (-not $snapshot) {
            throw "Snapshot '$snapshotName' not found for VM '$vmName'"
        }

        Write-Host "Removing snapshot: $snapshotName (VM: $vmName)" -ForegroundColor Yellow

        $scriptBlock = {
            param($vm, $snap)
            Remove-VMSnapshot -VMName $vm -VMSnapshotName $snap -Confirm:$false
        }

        if ($this.HyperVManager.HyperVHost -eq $env:COMPUTERNAME) {
            & $scriptBlock $vmName $snapshotName
        } else {
            Invoke-Command -ComputerName $this.HyperVManager.HyperVHost -Credential $this.HyperVManager.Credential -ScriptBlock $scriptBlock -ArgumentList $vmName, $snapshotName
        }

        $this.RefreshSnapshots()
        $this.LogSnapshotOperation("Removed", $vmName, $snapshotName, @{ Action = "Remove" })
    }

    [void]ApplyRetentionPolicy([string]$vmName, [hashtable]$policy) {
        if (-not $policy.AutoCleanup) { return }

        $vmSnapshots = $this.Snapshots[$vmName]
        if (-not $vmSnapshots -or $vmSnapshots.Count -le $policy.MaxSnapshots) { return }

        Write-Host "Applying retention policy for VM: $vmName" -ForegroundColor Gray

        # Tri par date de création (plus ancien en premier)
        $snapshotsToRemove = $vmSnapshots | Sort-Object CreationTime | Select-Object -First ($vmSnapshots.Count - $policy.MaxSnapshots)

        foreach ($snapshot in $snapshotsToRemove) {
            # Vérification de l'âge
            $age = (Get-Date) - $snapshot.CreationTime
            if ($age.TotalDays -gt $policy.RetentionDays) {
                $this.RemoveSnapshot($vmName, $snapshot.Name)
            }
        }
    }

    [PSObject[]]GetSnapshots([string]$vmName) {
        if (-not $this.Snapshots.ContainsKey($vmName)) {
            return @()
        }

        return $this.Snapshots[$vmName]
    }

    [hashtable]GetSnapshotStatistics() {
        $stats = @{
            TotalSnapshots = 0
            VMsWithSnapshots = 0
            OldestSnapshot = $null
            NewestSnapshot = $null
            AverageSnapshotsPerVM = 0
        }

        $totalSnapshots = 0
        $vmsWithSnapshots = 0
        $allSnapshots = @()

        foreach ($vmName in $this.Snapshots.Keys) {
            $vmSnapshots = $this.Snapshots[$vmName]
            if ($vmSnapshots.Count -gt 0) {
                $vmsWithSnapshots++
                $totalSnapshots += $vmSnapshots.Count
                $allSnapshots += $vmSnapshots
            }
        }

        $stats.TotalSnapshots = $totalSnapshots
        $stats.VMsWithSnapshots = $vmsWithSnapshots
        $stats.AverageSnapshotsPerVM = if ($vmsWithSnapshots -gt 0) { [math]::Round($totalSnapshots / $vmsWithSnapshots, 2) } else { 0 }

        if ($allSnapshots) {
            $stats.OldestSnapshot = $allSnapshots | Sort-Object CreationTime | Select-Object -First 1
            $stats.NewestSnapshot = $allSnapshots | Sort-Object CreationTime -Descending | Select-Object -First 1
        }

        return $stats
    }

    hidden [hashtable]DeterminePolicy([string]$vmName, [hashtable]$metadata) {
        # Logique de détermination de la politique basée sur les métadonnées
        if ($metadata.ContainsKey('Environment')) {
            $env = $metadata.Environment
            if ($this.SnapshotPolicies.ContainsKey($env)) {
                return $this.SnapshotPolicies[$env] + @{ Name = $env }
            }
        }

        # Recherche dans le nom de la VM
        if ($vmName -match 'dev|development') {
            return $this.SnapshotPolicies.Development + @{ Name = 'Development' }
        } elseif ($vmName -match 'prod|production') {
            return $this.SnapshotPolicies.Production + @{ Name = 'Production' }
        } elseif ($vmName -match 'test|testing') {
            return $this.SnapshotPolicies.Testing + @{ Name = 'Testing' }
        }

        # Politique par défaut
        return @{ Name = 'Default'; MaxSnapshots = 5; RetentionDays = 14; AutoCleanup = $true; CompressionEnabled = $false }
    }

    hidden [string]GenerateSnapshotName([string]$vmName, [string]$snapshotType, [hashtable]$policy) {
        $date = Get-Date -Format 'yyyyMMdd_HHmmss'
        $namingConvention = $policy.NamingConvention ?? "{VM}_{Date}_{Type}"

        $name = $namingConvention -replace '{VM}', $vmName -replace '{Date}', $date -replace '{Type}', $snapshotType

        return $name
    }

    hidden [void]LogSnapshotOperation([string]$operation, [string]$vmName, [string]$snapshotName, [hashtable]$details) {
        $logEntry = [PSCustomObject]@{
            Timestamp = Get-Date
            Operation = $operation
            VMName = $vmName
            SnapshotName = $snapshotName
            Details = $details
        }

        $this.SnapshotOperations.Add($logEntry)
    }

    [PSCustomObject[]]GetOperationsLog([int]$lastHours = 24) {
        $cutoff = (Get-Date).AddHours(-$lastHours)
        return $this.SnapshotOperations | Where-Object { $_.Timestamp -ge $cutoff } | Sort-Object Timestamp -Descending
    }
}

# Système de migration de VMs Hyper-V
class HyperVVMMigrationManager {
    [HyperVManager]$SourceManager
    [HyperVManager]$DestinationManager
    [System.Collections.Generic.List[PSCustomObject]]$MigrationJobs
    [hashtable]$MigrationSettings

    HyperVVMMigrationManager([HyperVManager]$source, [HyperVManager]$destination) {
        $this.SourceManager = $source
        $this.DestinationManager = $destination
        $this.MigrationJobs = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.MigrationSettings = @{
            MaxConcurrentMigrations = 2
            BandwidthLimitMBps = 100
            CompressionEnabled = $true
            VerifyAfterMigration = $true
            KeepSourceVM = $false
        }
    }

    [PSCustomObject]MigrateVM([string]$vmName, [hashtable]$migrationOptions = @{}) {
        if (-not $this.SourceManager.VMs.ContainsKey($vmName)) {
            throw "VM '$vmName' not found on source host"
        }

        $migrationJob = [PSCustomObject]@{
            Id = [guid]::NewGuid().ToString()
            VMName = $vmName
            Status = "Initializing"
            StartTime = Get-Date
            EndTime = $null
            Progress = 0
            SourceHost = $this.SourceManager.HyperVHost
            DestinationHost = $this.DestinationManager.HyperVHost
            Options = $migrationOptions
            Error = $null
        }

        $this.MigrationJobs.Add($migrationJob)

        Write-Host "Starting migration of VM: $vmName" -ForegroundColor Cyan
        Write-Host "Source: $($migrationJob.SourceHost) -> Destination: $($migrationJob.DestinationHost)"

        # Exécution de la migration en arrière-plan
        $job = Start-Job -ScriptBlock {
            param($jobId, $vm, $sourceHost, $destHost, $options)

            try {
                # Étape 1: Préparation
                $migration = Get-Variable -Name MigrationJobs -ValueOnly | Where-Object { $_.Id -eq $jobId }
                $migration.Status = "Preparing"

                # Vérifications pré-migration
                $vmInfo = Get-VM -ComputerName $sourceHost -Name $vm

                # Étape 2: Migration
                $migration.Status = "Migrating"

                if ($sourceHost -eq $destHost) {
                    # Migration de stockage uniquement
                    Move-VMStorage -ComputerName $sourceHost -Name $vm -DestinationStoragePath $options.DestinationPath
                } else {
                    # Migration complète
                    Move-VM -ComputerName $sourceHost -Name $vm -DestinationHost $destHost -IncludeStorage:$true -DestinationStoragePath $options.DestinationPath
                }

                # Étape 3: Vérification
                $migration.Status = "Verifying"
                $destVM = Get-VM -ComputerName $destHost -Name $vm

                if ($destVM.State -eq 'Off') {
                    Start-VM -ComputerName $destHost -Name $vm
                }

                # Étape 4: Finalisation
                $migration.Status = "Completed"
                $migration.EndTime = Get-Date
                $migration.Progress = 100

                return @{ Success = $true; VM = $destVM }

            } catch {
                $migration.Status = "Failed"
                $migration.EndTime = Get-Date
                $migration.Error = $_.Exception.Message

                return @{ Success = $false; Error = $_.Exception.Message }
            }

        } -ArgumentList $migrationJob.Id, $vmName, $this.SourceManager.HyperVHost, $this.DestinationManager.HyperVHost, $migrationOptions

        # Attente de la fin de la migration
        $result = Wait-Job $job | Receive-Job
        Remove-Job $job

        if ($result.Success) {
            Write-Host "Migration completed successfully for VM: $vmName" -ForegroundColor Green

            # Mise à jour des inventaires
            $this.SourceManager.RefreshVMInventory()
            $this.DestinationManager.RefreshVMInventory()
        } else {
            Write-Error "Migration failed for VM '$vmName': $($result.Error)"
        }

        return $migrationJob
    }

    [PSCustomObject[]]GetMigrationJobs([string]$status = "*") {
        return $this.MigrationJobs | Where-Object { $_.Status -like $status }
    }

    [hashtable]GetMigrationStatistics() {
        $stats = @{
            TotalMigrations = $this.MigrationJobs.Count
            SuccessfulMigrations = ($this.MigrationJobs | Where-Object { $_.Status -eq "Completed" }).Count
            FailedMigrations = ($this.MigrationJobs | Where-Object { $_.Status -eq "Failed" }).Count
            InProgressMigrations = ($this.MigrationJobs | Where-Object { $_.Status -in @("Initializing", "Preparing", "Migrating", "Verifying") }).Count
        }

        $completedMigrations = $this.MigrationJobs | Where-Object { $_.Status -eq "Completed" -and $_.EndTime }
        if ($completedMigrations) {
            $stats.AverageMigrationTime = ($completedMigrations | ForEach-Object { ($_.EndTime - $_.StartTime).TotalMinutes } | Measure-Object -Average).Average
        }

        return $stats
    }
}

# Démonstration du système de snapshots et migration
$hvManager = [HyperVManager]::new()
$snapshotManager = [HyperVSnapshotManager]::new($hvManager)

# Création de snapshots selon les politiques
foreach ($vm in $hvManager.GetVMs()) {
    if ($vm.State -eq "Running") {
        try {
            $snapshot = $snapshotManager.CreateSnapshot($vm.Name, "Automated", @{ Environment = "Development" })
            Write-Host "Snapshot created for VM: $($vm.Name)" -ForegroundColor Green
        } catch {
            Write-Warning "Failed to create snapshot for VM $($vm.Name): $_"
        }
    }
}

# Statistiques des snapshots
$snapshotStats = $snapshotManager.GetSnapshotStatistics()
Write-Host "`n=== SNAPSHOT STATISTICS ===" -ForegroundColor Cyan
Write-Host "Total snapshots: $($snapshotStats.TotalSnapshots)"
Write-Host "VMs with snapshots: $($snapshotStats.VMsWithSnapshots)"
Write-Host "Average snapshots per VM: $($snapshotStats.AverageSnapshotsPerVM)"

# Configuration de la migration
$migrationManager = [HyperVVMMigrationManager]::new($hvManager, $hvManager)  # Même hôte pour la démo

# Simulation de migration (nécessiterait deux hôtes différents pour être fonctionnel)
$migrationOptions = @{
    DestinationPath = "D:\VMStorage"
    BandwidthLimit = 50MB
    CompressionEnabled = $true
}

# Rapport final
Write-Host "`n=== HYPER-V ADVANCED MANAGEMENT REPORT ===" -ForegroundColor Cyan

$hvReport = $hvManager.GenerateReport()
$snapshotOps = $snapshotManager.GetOperationsLog(1)
$migrationStats = $migrationManager.GetMigrationStatistics()

$finalReport = @"
$hvReport

SNAPSHOT OPERATIONS (Last Hour)
================================
$($snapshotOps | Select-Object -First 5 | ForEach-Object { "$($_.Timestamp): $($_.Operation) - $($_.VMName) - $($_.SnapshotName)" } | Out-String)

MIGRATION STATISTICS
===================
Total Migrations: $($migrationStats.TotalMigrations)
Successful: $($migrationStats.SuccessfulMigrations)
Failed: $($migrationStats.FailedMigrations)
In Progress: $($migrationStats.InProgressMigrations)
"@

$finalReport | Out-File -FilePath "HyperV_Advanced_Report_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nAdvanced Hyper-V management demonstration completed!" -ForegroundColor Green
```

## Section 2 : Gestion des conteneurs Docker

### 2.1 Orchestration Docker avec PowerShell

**Framework complet de gestion Docker :**
```powershell
# Framework avancé de gestion Docker avec PowerShell
class DockerManager {
    [string]$DockerHost
    [System.Collections.Generic.Dictionary[string, PSObject]]$Containers
    [System.Collections.Generic.Dictionary[string, PSObject]]$Images
    [System.Collections.Generic.List[PSCustomObject]]$OperationsLog
    [hashtable]$DockerConfig

    DockerManager([string]$dockerHost = "localhost") {
        $this.DockerHost = $dockerHost
        $this.Containers = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.Images = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.OperationsLog = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.DockerConfig = @{
            Registry = "docker.io"
            Timeout = 300
            RetryCount = 3
            PullTimeout = 600
        }

        $this.RefreshInventory()
    }

    hidden [void]RefreshInventory() {
        Write-Host "Refreshing Docker inventory..." -ForegroundColor Gray

        try {
            # Récupération des conteneurs
            $containers = docker ps -a --format "json" | ConvertFrom-Json
            $this.Containers.Clear()
            foreach ($container in $containers) {
                $this.Containers[$container.Names] = $container
            }

            # Récupération des images
            $images = docker images --format "json" | ConvertFrom-Json
            $this.Images.Clear()
            foreach ($image in $images) {
                $key = "$($image.Repository):$($image.Tag)"
                $this.Images[$key] = $image
            }

            $this.LogOperation("InventoryRefreshed", "Docker inventory refreshed", @{
                Containers = $this.Containers.Count
                Images = $this.Images.Count
            })

        } catch {
            Write-Warning "Failed to refresh Docker inventory: $_"
        }
    }

    [PSObject]RunContainer([hashtable]$containerConfig) {
        Write-Host "Running container: $($containerConfig.Name)" -ForegroundColor Yellow

        # Validation de la configuration
        $this.ValidateContainerConfig($containerConfig)

        # Construction de la commande Docker
        $dockerCommand = "docker run -d --name $($containerConfig.Name)"

        # Ajout des ports
        if ($containerConfig.Ports) {
            foreach ($port in $containerConfig.Ports) {
                $dockerCommand += " -p $($port.Host):$($port.Container)"
            }
        }

        # Ajout des volumes
        if ($containerConfig.Volumes) {
            foreach ($volume in $containerConfig.Volumes) {
                $dockerCommand += " -v $($volume.Host):$($volume.Container)"
            }
        }

        # Ajout des variables d'environnement
        if ($containerConfig.Environment) {
            foreach ($env in $containerConfig.Environment.GetEnumerator()) {
                $dockerCommand += " -e $($env.Key)='$($env.Value)'"
            }
        }

        # Ajout des options réseau
        if ($containerConfig.Network) {
            $dockerCommand += " --network $($containerConfig.Network)"
        }

        # Ajout de l'image
        $dockerCommand += " $($containerConfig.Image)"

        # Ajout de la commande personnalisée
        if ($containerConfig.Command) {
            $dockerCommand += " $($containerConfig.Command)"
        }

        try {
            $result = Invoke-Expression $dockerCommand
            $this.RefreshInventory()

            $this.LogOperation("ContainerStarted", "Container started successfully", @{
                ContainerName = $containerConfig.Name
                Image = $containerConfig.Image
                Id = $result
            })

            return $this.Containers[$containerConfig.Name]

        } catch {
            $this.LogOperation("ContainerStartFailed", "Failed to start container", @{
                ContainerName = $containerConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    hidden [void]ValidateContainerConfig([hashtable]$config) {
        $requiredFields = @('Name', 'Image')
        foreach ($field in $requiredFields) {
            if (-not $config.ContainsKey($field)) {
                throw "Container configuration missing required field: $field"
            }
        }

        if ($this.Containers.ContainsKey($config.Name)) {
            throw "Container with name '$($config.Name)' already exists"
        }

        # Validation des ports
        if ($config.Ports) {
            foreach ($port in $config.Ports) {
                if (-not ($port.Host -and $port.Container)) {
                    throw "Invalid port configuration: $($port | ConvertTo-Json)"
                }
            }
        }
    }

    [void]StopContainer([string]$containerName) {
        $this.ExecuteContainerCommand($containerName, "stop", {
            param($name)
            docker stop $name
        })
    }

    [void]StartContainer([string]$containerName) {
        $this.ExecuteContainerCommand($containerName, "start", {
            param($name)
            docker start $name
        })
    }

    [void]RemoveContainer([string]$containerName, [switch]$Force) {
        $this.ExecuteContainerCommand($containerName, "remove", {
            param($name, $force)
            if ($force) {
                docker rm -f $name
            } else {
                docker rm $name
            }
        }, $Force)
    }

    hidden [void]ExecuteContainerCommand([string]$containerName, [string]$operation, [scriptblock]$command, $additionalArgs = $null) {
        if (-not $this.Containers.ContainsKey($containerName)) {
            throw "Container '$containerName' not found"
        }

        Write-Host "${operation}ing container: $containerName" -ForegroundColor Yellow

        try {
            & $command $containerName $additionalArgs
            $this.RefreshInventory()

            $this.LogOperation("ContainerOperation", "Container operation completed", @{
                ContainerName = $containerName
                Operation = $operation
            })

        } catch {
            $this.LogOperation("ContainerOperationFailed", "Container operation failed", @{
                ContainerName = $containerName
                Operation = $operation
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]BuildImage([string]$dockerfilePath, [string]$imageName, [string]$tag = "latest") {
        Write-Host "Building Docker image: $imageName:$tag" -ForegroundColor Yellow

        if (-not (Test-Path $dockerfilePath)) {
            throw "Dockerfile not found: $dockerfilePath"
        }

        $buildContext = Split-Path $dockerfilePath -Parent
        $fullImageName = "$imageName`:$tag"

        try {
            $result = docker build -t $fullImageName $buildContext

            $this.RefreshInventory()

            $this.LogOperation("ImageBuilt", "Docker image built successfully", @{
                ImageName = $fullImageName
                Dockerfile = $dockerfilePath
            })

            return $this.Images[$fullImageName]

        } catch {
            $this.LogOperation("ImageBuildFailed", "Failed to build Docker image", @{
                ImageName = $fullImageName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]PushImage([string]$imageName, [string]$tag = "latest") {
        $fullImageName = "$imageName`:$tag"

        if (-not $this.Images.ContainsKey($fullImageName)) {
            throw "Image '$fullImageName' not found locally"
        }

        Write-Host "Pushing image: $fullImageName" -ForegroundColor Yellow

        try {
            docker push $fullImageName

            $this.LogOperation("ImagePushed", "Image pushed to registry", @{
                ImageName = $fullImageName
            })

        } catch {
            $this.LogOperation("ImagePushFailed", "Failed to push image", @{
                ImageName = $fullImageName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject[]]GetContainers([string]$filter = "*") {
        return $this.Containers.Values | Where-Object { $_.Names -like $filter }
    }

    [PSObject[]]GetImages([string]$filter = "*") {
        return $this.Images.Values | Where-Object { "$($_.Repository):$($_.Tag)" -like $filter }
    }

    [hashtable]GetDockerStats() {
        try {
            $stats = docker stats --no-stream --format "json" | ConvertFrom-Json

            return @{
                ContainerCount = $stats.Count
                TotalCPUUsage = ($stats | Measure-Object -Property CPUPercentage -Sum).Sum
                TotalMemoryUsage = ($stats | Measure-Object -Property MemoryUsage -Sum).Sum
                RunningContainers = ($stats | Where-Object { $_.Container -and -not $_.Container.Contains("Exited") }).Count
            }
        } catch {
            Write-Warning "Failed to get Docker stats: $_"
            return @{ Error = $_.Exception.Message }
        }
    }

    hidden [void]LogOperation([string]$operation, [string]$message, [hashtable]$details = @{}) {
        $logEntry = [PSCustomObject]@{
            Timestamp = Get-Date
            Operation = $operation
            Message = $message
            Details = $details
        }

        $this.OperationsLog.Add($logEntry)
    }

    [PSCustomObject[]]GetOperationsLog([int]$lastHours = 24) {
        $cutoff = (Get-Date).AddHours(-$lastHours)
        return $this.OperationsLog | Where-Object { $_.Timestamp -ge $cutoff } | Sort-Object Timestamp -Descending
    }

    [string]GenerateReport() {
        $stats = $this.GetDockerStats()
        $operations = $this.GetOperationsLog(24)

        $report = @"
DOCKER ENVIRONMENT REPORT
=========================
Host: $($this.DockerHost)
Generated: $(Get-Date)

CONTAINER STATISTICS
===================
Total Containers: $($this.Containers.Count)
Running Containers: $(($this.Containers.Values | Where-Object { $_.State -eq "running" }).Count)
Stopped Containers: $(($this.Containers.Values | Where-Object { $_.State -eq "exited" }).Count)

IMAGE STATISTICS
================
Total Images: $($this.Images.Count)
Total Size: $([math]::Round(($this.Images.Values | Measure-Object -Property Size -Sum).Sum / 1MB, 2)) MB

PERFORMANCE METRICS
===================
CPU Usage: $($stats.TotalCPUUsage)%
Memory Usage: $([math]::Round($stats.TotalMemoryUsage / 1MB, 2)) MB
Active Containers: $($stats.RunningContainers)

RECENT OPERATIONS (Last 24h)
===========================
$($operations | Select-Object -First 10 | ForEach-Object { "$($_.Timestamp): $($_.Operation) - $($_.Message)" } | Out-String)
"@

        return $report
    }
}

# Orchestrateur de conteneurs Docker
class DockerOrchestrator : DockerManager {
    [System.Collections.Generic.Dictionary[string, hashtable]]$Services
    [hashtable]$OrchestrationConfig

    DockerOrchestrator([string]$dockerHost = "localhost") : base($dockerHost) {
        $this.Services = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.OrchestrationConfig = @{
            HealthCheckInterval = 30
            AutoRestart = $true
            LoadBalancing = $true
            ScalingEnabled = $true
        }
    }

    [void]DefineService([string]$serviceName, [hashtable]$serviceConfig) {
        $this.Services[$serviceName] = $serviceConfig.Clone()
        Write-Host "Service defined: $serviceName" -ForegroundColor Green
    }

    [void]DeployService([string]$serviceName) {
        if (-not $this.Services.ContainsKey($serviceName)) {
            throw "Service '$serviceName' not defined"
        }

        $service = $this.Services[$serviceName]
        Write-Host "Deploying service: $serviceName" -ForegroundColor Cyan

        # Déploiement selon la stratégie
        switch ($service.Strategy) {
            "Single" {
                $this.DeploySingleContainer($serviceName, $service)
            }
            "Replicated" {
                $this.DeployReplicatedService($serviceName, $service)
            }
            "LoadBalanced" {
                $this.DeployLoadBalancedService($serviceName, $service)
            }
            default {
                throw "Unknown deployment strategy: $($service.Strategy)"
            }
        }

        $this.LogOperation("ServiceDeployed", "Service deployed successfully", @{
            ServiceName = $serviceName
            Strategy = $service.Strategy
        })
    }

    hidden [void]DeploySingleContainer([string]$serviceName, [hashtable]$service) {
        $containerConfig = @{
            Name = $serviceName
            Image = $service.Image
            Ports = $service.Ports
            Environment = $service.Environment
            Volumes = $service.Volumes
        }

        $this.RunContainer($containerConfig)
    }

    hidden [void]DeployReplicatedService([string]$serviceName, [hashtable]$service) {
        $replicas = $service.Replicas ?? 1

        for ($i = 1; $i -le $replicas; $i++) {
            $containerName = "$serviceName-$i"
            $containerConfig = @{
                Name = $containerName
                Image = $service.Image
                Ports = $service.Ports
                Environment = $service.Environment + @{ REPLICA_ID = $i }
                Volumes = $service.Volumes
            }

            $this.RunContainer($containerConfig)
        }
    }

    hidden [void]DeployLoadBalancedService([string]$serviceName, [hashtable]$service) {
        # Déploiement du load balancer (nginx)
        $lbConfig = @{
            Name = "$serviceName-lb"
            Image = "nginx:alpine"
            Ports = @(@{ Host = $service.LoadBalancerPort; Container = 80 })
            Volumes = @(
                @{ Host = "$PSScriptRoot\$serviceName-nginx.conf"; Container = "/etc/nginx/nginx.conf" }
            )
        }

        $this.RunContainer($lbConfig)

        # Déploiement des instances d'application
        $this.DeployReplicatedService($serviceName, $service)
    }

    [void]ScaleService([string]$serviceName, [int]$newReplicaCount) {
        if (-not $this.Services.ContainsKey($serviceName)) {
            throw "Service '$serviceName' not defined"
        }

        $service = $this.Services[$serviceName]
        $currentContainers = $this.GetContainers("$serviceName-*")

        Write-Host "Scaling service $serviceName from $($currentContainers.Count) to $newReplicaCount replicas" -ForegroundColor Yellow

        if ($currentContainers.Count -lt $newReplicaCount) {
            # Scale up
            $containersToAdd = $newReplicaCount - $currentContainers.Count
            for ($i = 1; $i -le $containersToAdd; $i++) {
                $containerName = "$serviceName-$($currentContainers.Count + $i)"
                $containerConfig = @{
                    Name = $containerName
                    Image = $service.Image
                    Environment = $service.Environment + @{ REPLICA_ID = ($currentContainers.Count + $i) }
                    Volumes = $service.Volumes
                }

                $this.RunContainer($containerConfig)
            }
        } elseif ($currentContainers.Count -gt $newReplicaCount) {
            # Scale down
            $containersToRemove = $currentContainers.Count - $newReplicaCount
            $containersToRemove = $currentContainers | Sort-Object Names -Descending | Select-Object -First $containersToRemove

            foreach ($container in $containersToRemove) {
                $this.StopContainer($container.Names)
                $this.RemoveContainer($container.Names)
            }
        }

        $service.Replicas = $newReplicaCount
        $this.LogOperation("ServiceScaled", "Service scaled successfully", @{
            ServiceName = $serviceName
            NewReplicaCount = $newReplicaCount
        })
    }

    [void]UpdateService([string]$serviceName, [string]$newImage) {
        Write-Host "Updating service: $serviceName to image: $newImage" -ForegroundColor Yellow

        # Stratégie de déploiement rolling update
        $serviceContainers = $this.GetContainers("$serviceName-*")
        $totalContainers = $serviceContainers.Count

        for ($i = 0; $i -lt $totalContainers; $i++) {
            $oldContainer = $serviceContainers[$i]
            $newContainerName = "$serviceName-temp-$i"

            # Création du nouveau conteneur
            $service = $this.Services[$serviceName]
            $containerConfig = @{
                Name = $newContainerName
                Image = $newImage
                Ports = $service.Ports
                Environment = $service.Environment
                Volumes = $service.Volumes
            }

            $newContainer = $this.RunContainer($containerConfig)

            # Attente que le nouveau conteneur soit prêt
            Start-Sleep -Seconds 10

            # Arrêt de l'ancien conteneur
            $this.StopContainer($oldContainer.Names)
            $this.RemoveContainer($oldContainer.Names)

            # Renommage du nouveau conteneur
            docker rename $newContainerName $oldContainer.Names

            Write-Host "Updated container: $($oldContainer.Names)" -ForegroundColor Green
        }

        # Mise à jour de l'image dans la définition du service
        $this.Services[$serviceName].Image = $newImage

        $this.LogOperation("ServiceUpdated", "Service updated successfully", @{
            ServiceName = $serviceName
            NewImage = $newImage
        })
    }

    [hashtable]GetServiceHealth([string]$serviceName) {
        $serviceContainers = $this.GetContainers("$serviceName-*")

        $health = @{
            ServiceName = $serviceName
            TotalContainers = $serviceContainers.Count
            RunningContainers = 0
            UnhealthyContainers = 0
            HealthPercentage = 0
        }

        foreach ($container in $serviceContainers) {
            if ($container.State -eq "running") {
                $health.RunningContainers++

                # Vérification de santé basique (peut être étendue)
                try {
                    $stats = docker stats $container.Names --no-stream --format "json" | ConvertFrom-Json
                    if ($stats.CPUPercentage -gt 95 -or $stats.MemoryPercentage -gt 95) {
                        $health.UnhealthyContainers++
                    }
                } catch {
                    $health.UnhealthyContainers++
                }
            }
        }

        $healthyContainers = $health.RunningContainers - $health.UnhealthyContainers
        $health.HealthPercentage = if ($health.TotalContainers -gt 0) {
            [math]::Round(($healthyContainers / $health.TotalContainers) * 100, 2)
        } else { 0 }

        return $health
    }

    [void]StartHealthMonitoring() {
        Write-Host "Starting health monitoring..." -ForegroundColor Green

        $job = Start-Job -ScriptBlock {
            param($orchestrator)

            while ($true) {
                foreach ($serviceName in $orchestrator.Services.Keys) {
                    $health = $orchestrator.GetServiceHealth($serviceName)

                    if ($health.HealthPercentage -lt 80) {
                        Write-Host "HEALTH ALERT: Service $serviceName health at $($health.HealthPercentage)%" -ForegroundColor Red

                        # Auto-remédiation si configurée
                        if ($orchestrator.OrchestrationConfig.AutoRestart) {
                            $unhealthyContainers = $orchestrator.GetContainers("$serviceName-*") | Where-Object { $_.State -ne "running" }

                            foreach ($container in $unhealthyContainers) {
                                try {
                                    $orchestrator.StartContainer($container.Names)
                                    Write-Host "Auto-restarted container: $($container.Names)" -ForegroundColor Yellow
                                } catch {
                                    Write-Warning "Failed to restart container $($container.Names): $_"
                                }
                            }
                        }
                    }
                }

                Start-Sleep -Seconds $orchestrator.OrchestrationConfig.HealthCheckInterval
            }
        } -ArgumentList $this

        Write-Host "Health monitoring started. Job ID: $($job.Id)" -ForegroundColor Green
    }
}

# Démonstration du framework Docker
$docker = [DockerOrchestrator]::new()

# Définition de services
$docker.DefineService("web-app", @{
    Image = "nginx:alpine"
    Strategy = "LoadBalanced"
    Ports = @(@{ Host = 8080; Container = 80 })
    LoadBalancerPort = 8080
    Replicas = 2
    Environment = @{ APP_ENV = "production" }
})

$docker.DefineService("api", @{
    Image = "myapi:latest"
    Strategy = "Replicated"
    Ports = @(@{ Host = 3000; Container = 3000 })
    Replicas = 3
    Environment = @{ NODE_ENV = "production"; DB_HOST = "db" }
})

$docker.DefineService("database", @{
    Image = "postgres:13"
    Strategy = "Single"
    Ports = @(@{ Host = 5432; Container = 5432 })
    Environment = @{ POSTGRES_DB = "myapp"; POSTGRES_USER = "app"; POSTGRES_PASSWORD = "secret" }
    Volumes = @(@{ Host = "C:\DockerVolumes\postgres"; Container = "/var/lib/postgresql/data" })
})

# Déploiement des services
foreach ($serviceName in $docker.Services.Keys) {
    try {
        $docker.DeployService($serviceName)
        Write-Host "Service deployed: $serviceName" -ForegroundColor Green
    } catch {
        Write-Warning "Failed to deploy service $serviceName`: $_"
    }
}

# Mise à l'échelle d'un service
$docker.ScaleService("api", 5)

# Mise à jour d'un service
$docker.UpdateService("web-app", "nginx:latest")

# Surveillance de santé
$docker.StartHealthMonitoring()

# Statistiques finales
Write-Host "`n=== DOCKER ORCHESTRATION REPORT ===" -ForegroundColor Cyan

$dockerReport = $docker.GenerateReport()
$serviceHealth = @{}
foreach ($serviceName in $docker.Services.Keys) {
    $serviceHealth[$serviceName] = $docker.GetServiceHealth($serviceName)
}

$finalReport = @"
$dockerReport

SERVICE HEALTH STATUS
====================
$($serviceHealth.GetEnumerator() | ForEach-Object { "$($_.Key): $($_.Value.HealthPercentage)% healthy ($($_.Value.RunningContainers)/$($_.Value.TotalContainers) running)" } | Out-String)
"@

$finalReport | Out-File -FilePath "Docker_Orchestration_Report_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nDocker orchestration demonstration completed!" -ForegroundColor Green
```

## Conclusion : PowerShell comme maître des infrastructures virtualisées

PowerShell transcende son rôle traditionnel pour devenir l'orchestrateur ultime des environnements virtualisés et conteneurisés. En maîtrisant Hyper-V, Docker, et les patterns d'orchestration avancée, les administrateurs PowerShell créent des infrastructures dynamiques capables de s'adapter instantanément aux demandes des applications modernes.

Dans le prochain chapitre, nous explorerons les intégrations cloud avancées avec PowerShell, ouvrant les portes de l'infrastructure as code à l'échelle du cloud.

---

**Exercice pratique :** Créez une infrastructure complète virtualisée et conteneurisée avec :
1. Un cluster Hyper-V avec migration automatique
2. Un orchestreur Docker avec services répliqués
3. Des politiques de snapshots intelligents
4. Un système de monitoring et auto-remédiation
5. Des rapports d'état et de performance automatiques

**Challenge avancé :** Développez une plateforme d'orchestration hybride qui :
- Gère simultanément des VMs Hyper-V et des conteneurs Docker
- Implémente des stratégies de déploiement blue-green
- Fournit des capacités d'auto-scaling intelligentes
- Intègre des métriques de performance temps réel
- Supporte la migration transparente entre environnements

**Réflexion :** Comment PowerShell transforme-t-il l'administration des infrastructures modernes ? La virtualisation et les conteneurs rendent-ils l'administration système plus simple ou plus complexe ? Quels sont les défis de l'orchestration à grande échelle, et comment les surmonter ?
