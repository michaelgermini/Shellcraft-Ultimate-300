# Chapitre 138 - Intégrations cloud avancées avec PowerShell

> "Le cloud n'est pas une destination, c'est un écosystème vivant où PowerShell sert de pont entre les mondes locaux et distribués, orchestrant l'infini avec élégance et précision." - Citation inspirée des architectures cloud-native

## Introduction : PowerShell comme citoyen du cloud

PowerShell transcende les limites des environnements locaux pour devenir l'orchestrateur par excellence des infrastructures cloud. En maîtrisant les APIs d'Azure, AWS, et GCP à travers des modules spécialisés et des patterns d'intégration avancés, les développeurs PowerShell créent des architectures hybrides et multi-cloud d'une sophistication sans précédent.

Dans ce chapitre, nous explorerons les intégrations cloud avancées, l'infrastructure as code, et les orchestrations multi-cloud.

## Section 1 : Azure PowerShell avancé

### 1.1 Gestion avancée des ressources Azure

**Framework complet de gestion Azure :**
```powershell
# Framework avancé de gestion Azure avec PowerShell
class AzureResourceManager {
    [string]$SubscriptionId
    [string]$TenantId
    [PSCredential]$Credentials
    [System.Collections.Generic.Dictionary[string, PSObject]]$Resources
    [System.Collections.Generic.List[PSCustomObject]]$OperationsLog
    [hashtable]$AzureConfig

    AzureResourceManager([string]$subscriptionId, [string]$tenantId, [PSCredential]$credentials) {
        $this.SubscriptionId = $subscriptionId
        $this.TenantId = $tenantId
        $this.Credentials = $credentials
        $this.Resources = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.OperationsLog = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.AzureConfig = @{
            Location = "East US"
            ResourceGroupPrefix = "ps-managed"
            Tags = @{ Creator = "PowerShell"; Environment = "Development" }
            RetryCount = 3
            Timeout = 300
        }

        $this.ConnectToAzure()
        $this.RefreshResourceInventory()
    }

    hidden [void]ConnectToAzure() {
        Write-Host "Connecting to Azure..." -ForegroundColor Cyan

        try {
            $context = Connect-AzAccount -Credential $this.Credentials -TenantId $this.TenantId -SubscriptionId $this.SubscriptionId -ErrorAction Stop
            Write-Host "Connected to Azure as: $($context.Context.Account.Id)" -ForegroundColor Green

            $this.LogOperation("AzureConnected", "Successfully connected to Azure", @{
                Account = $context.Context.Account.Id
                Subscription = $context.Context.Subscription.Name
            })

        } catch {
            Write-Error "Failed to connect to Azure: $_"
            throw
        }
    }

    hidden [void]RefreshResourceInventory() {
        Write-Host "Refreshing Azure resource inventory..." -ForegroundColor Gray

        try {
            $resources = Get-AzResource | Select-Object -Property Name, ResourceType, ResourceGroupName, Location, Tags, CreatedTime

            $this.Resources.Clear()
            foreach ($resource in $resources) {
                $this.Resources[$resource.Name] = $resource
            }

            $this.LogOperation("InventoryRefreshed", "Azure resource inventory refreshed", @{
                ResourceCount = $this.Resources.Count
            })

        } catch {
            Write-Warning "Failed to refresh resource inventory: $_"
        }
    }

    [PSObject]CreateResourceGroup([string]$name, [string]$location = $null) {
        $location = $location ?? $this.AzureConfig.Location
        $rgName = "$($this.AzureConfig.ResourceGroupPrefix)-$name"

        Write-Host "Creating resource group: $rgName" -ForegroundColor Yellow

        try {
            $resourceGroup = New-AzResourceGroup -Name $rgName -Location $location -Tag $this.AzureConfig.Tags

            $this.LogOperation("ResourceGroupCreated", "Resource group created successfully", @{
                Name = $rgName
                Location = $location
            })

            $this.RefreshResourceInventory()
            return $resourceGroup

        } catch {
            $this.LogOperation("ResourceGroupCreateFailed", "Failed to create resource group", @{
                Name = $rgName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]CreateVirtualMachine([hashtable]$vmConfig) {
        Write-Host "Creating Azure VM: $($vmConfig.Name)" -ForegroundColor Yellow

        # Création du resource group si nécessaire
        $rgName = $vmConfig.ResourceGroup ?? "$($this.AzureConfig.ResourceGroupPrefix)-vms"
        if (-not (Get-AzResourceGroup -Name $rgName -ErrorAction SilentlyContinue)) {
            $this.CreateResourceGroup($rgName)
        }

        # Configuration de la VM
        $vmParams = @{
            ResourceGroupName = $rgName
            Name = $vmConfig.Name
            Location = $vmConfig.Location ?? $this.AzureConfig.Location
            VirtualNetworkName = $vmConfig.VNetName ?? "$($vmConfig.Name)-vnet"
            SubnetName = $vmConfig.SubnetName ?? "default"
            SecurityGroupName = $vmConfig.NsgName ?? "$($vmConfig.Name)-nsg"
            PublicIpAddressName = $vmConfig.PublicIpName ?? "$($vmConfig.Name)-pip"
            OpenPorts = $vmConfig.OpenPorts ?? 3389
        }

        # Ajout des paramètres optionnels
        if ($vmConfig.Image) { $vmParams.ImageName = $vmConfig.Image }
        if ($vmConfig.Size) { $vmParams.Size = $vmConfig.Size }
        if ($vmConfig.Credential) { $vmParams.Credential = $vmConfig.Credential }

        try {
            $vm = New-AzVM @vmParams

            # Configuration supplémentaire
            if ($vmConfig.DataDisks) {
                foreach ($disk in $vmConfig.DataDisks) {
                    Add-AzVMDataDisk -VM $vm -Name $disk.Name -DiskSizeInGB $disk.Size -CreateOption Empty -Lun $disk.Lun
                    Update-AzVM -ResourceGroupName $rgName -VM $vm
                }
            }

            if ($vmConfig.Extensions) {
                foreach ($extension in $vmConfig.Extensions) {
                    Set-AzVMExtension @extension -ResourceGroupName $rgName -VMName $vmConfig.Name
                }
            }

            $this.LogOperation("VMCreated", "Azure VM created successfully", @{
                VMName = $vmConfig.Name
                ResourceGroup = $rgName
                Location = $vmParams.Location
            })

            $this.RefreshResourceInventory()
            return $vm

        } catch {
            $this.LogOperation("VMCreateFailed", "Failed to create Azure VM", @{
                VMName = $vmConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]CreateStorageAccount([hashtable]$storageConfig) {
        Write-Host "Creating Azure Storage Account: $($storageConfig.Name)" -ForegroundColor Yellow

        $rgName = $storageConfig.ResourceGroup ?? "$($this.AzureConfig.ResourceGroupPrefix)-storage"

        if (-not (Get-AzResourceGroup -Name $rgName -ErrorAction SilentlyContinue)) {
            $this.CreateResourceGroup($rgName)
        }

        $storageParams = @{
            ResourceGroupName = $rgName
            Name = $storageConfig.Name
            Location = $storageConfig.Location ?? $this.AzureConfig.Location
            SkuName = $storageConfig.Sku ?? "Standard_LRS"
            Kind = $storageConfig.Kind ?? "StorageV2"
            AccessTier = $storageConfig.AccessTier ?? "Hot"
        }

        try {
            $storageAccount = New-AzStorageAccount @storageParams

            # Configuration des règles réseau si spécifiées
            if ($storageConfig.NetworkRules) {
                Update-AzStorageAccountNetworkRuleSet -ResourceGroupName $rgName -Name $storageConfig.Name @storageConfig.NetworkRules
            }

            # Création de conteneurs par défaut
            if ($storageConfig.Containers) {
                $ctx = $storageAccount.Context
                foreach ($container in $storageConfig.Containers) {
                    New-AzStorageContainer -Name $container -Context $ctx -Permission Off
                }
            }

            $this.LogOperation("StorageAccountCreated", "Storage account created successfully", @{
                AccountName = $storageConfig.Name
                ResourceGroup = $rgName
                Sku = $storageParams.SkuName
            })

            $this.RefreshResourceInventory()
            return $storageAccount

        } catch {
            $this.LogOperation("StorageAccountCreateFailed", "Failed to create storage account", @{
                AccountName = $storageConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]StartVM([string]$vmName, [string]$resourceGroup) {
        Write-Host "Starting Azure VM: $vmName" -ForegroundColor Yellow

        try {
            $result = Start-AzVM -ResourceGroupName $resourceGroup -Name $vmName

            $this.LogOperation("VMStarted", "VM started successfully", @{
                VMName = $vmName
                ResourceGroup = $resourceGroup
            })

        } catch {
            $this.LogOperation("VMStartFailed", "Failed to start VM", @{
                VMName = $vmName
                ResourceGroup = $resourceGroup
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]StopVM([string]$vmName, [string]$resourceGroup, [switch]$Force) {
        Write-Host "Stopping Azure VM: $vmName" -ForegroundColor Yellow

        try {
            if ($Force) {
                $result = Stop-AzVM -ResourceGroupName $resourceGroup -Name $vmName -Force
            } else {
                $result = Stop-AzVM -ResourceGroupName $resourceGroup -Name $vmName
            }

            $this.LogOperation("VMStopped", "VM stopped successfully", @{
                VMName = $vmName
                ResourceGroup = $resourceGroup
                Force = $Force
            })

        } catch {
            $this.LogOperation("VMStopFailed", "Failed to stop VM", @{
                VMName = $vmName
                ResourceGroup = $resourceGroup
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [hashtable]GetResourceCosts([DateTime]$startDate, [DateTime]$endDate) {
        Write-Host "Retrieving Azure resource costs..." -ForegroundColor Gray

        try {
            $costs = Get-AzConsumptionUsageDetail -StartDate $startDate -EndDate $endDate

            $costSummary = @{
                TotalCost = ($costs | Measure-Object -Property PretaxCost -Sum).Sum
                CostsByService = @{}
                CostsByResourceGroup = @{}
                TopCostResources = @()
            }

            # Coûts par service
            $costSummary.CostsByService = $costs | Group-Object ConsumedService | ForEach-Object {
                @{ $_.Name = ($_.Group | Measure-Object PretaxCost -Sum).Sum }
            }

            # Coûts par resource group
            $costSummary.CostsByResourceGroup = $costs | Group-Object ResourceGroup | ForEach-Object {
                @{ $_.Name = ($_.Group | Measure-Object PretaxCost -Sum).Sum }
            }

            # Ressources les plus coûteuses
            $costSummary.TopCostResources = $costs | Sort-Object PretaxCost -Descending | Select-Object -First 10 | ForEach-Object {
                @{
                    ResourceName = $_.InstanceName
                    Cost = $_.PretaxCost
                    Service = $_.ConsumedService
                    Date = $_.Date
                }
            }

            return $costSummary

        } catch {
            Write-Warning "Failed to retrieve cost data: $_"
            return @{ Error = $_.Exception.Message }
        }
    }

    [PSObject[]]GetResources([string]$filter = "*") {
        return $this.Resources.Values | Where-Object { $_.Name -like $filter }
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
        $costs = $this.GetResourceCosts((Get-Date).AddDays(-30), (Get-Date))
        $operations = $this.GetOperationsLog(24)

        $report = @"
AZURE RESOURCE MANAGEMENT REPORT
=================================
Subscription: $($this.SubscriptionId)
Generated: $(Get-Date)

RESOURCE INVENTORY
==================
Total Resources: $($this.Resources.Count)
Resource Groups: $((Get-AzResourceGroup).Count)
Virtual Machines: $((Get-AzVM).Count)
Storage Accounts: $((Get-AzStorageAccount).Count)

COST ANALYSIS (Last 30 days)
===========================
Total Cost: `$$([math]::Round($costs.TotalCost, 2))

Top Services by Cost:
$($costs.CostsByService.GetEnumerator() | Sort-Object Value -Descending | Select-Object -First 5 | ForEach-Object { "  $($_.Key): `$$([math]::Round($_.Value, 2))" } | Out-String)

Top Resource Groups by Cost:
$($costs.CostsByResourceGroup.GetEnumerator() | Sort-Object Value -Descending | Select-Object -First 5 | ForEach-Object { "  $($_.Key): `$$([math]::Round($_.Value, 2))" } | Out-String)

RECENT OPERATIONS (Last 24h)
===========================
$($operations | Select-Object -First 10 | ForEach-Object { "$($_.Timestamp): $($_.Operation) - $($_.Message)" } | Out-String)
"@

        return $report
    }
}

# Automatisation avancée des déploiements Azure
class AzureDeploymentOrchestrator : AzureResourceManager {
    [System.Collections.Generic.Dictionary[string, hashtable]]$Deployments
    [hashtable]$DeploymentTemplates

    AzureDeploymentOrchestrator([string]$subscriptionId, [string]$tenantId, [PSCredential]$credentials) : base($subscriptionId, $tenantId, $credentials) {
        $this.Deployments = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.DeploymentTemplates = $this.LoadDeploymentTemplates()
    }

    hidden [hashtable]LoadDeploymentTemplates() {
        # Templates prédéfinis pour des déploiements courants
        return @{
            "WebApp" = @{
                TemplatePath = "$PSScriptRoot\Templates\webapp.json"
                Parameters = @{
                    appName = @{ Type = "string"; DefaultValue = "mywebapp" }
                    location = @{ Type = "string"; DefaultValue = "East US" }
                    sku = @{ Type = "string"; DefaultValue = "F1" }
                }
            }
            "VMScaleSet" = @{
                TemplatePath = "$PSScriptRoot\Templates\vmscaleset.json"
                Parameters = @{
                    vmssName = @{ Type = "string"; DefaultValue = "myvmss" }
                    instanceCount = @{ Type = "int"; DefaultValue = 2 }
                    vmSize = @{ Type = "string"; DefaultValue = "Standard_DS1_v2" }
                }
            }
            "AKSCluster" = @{
                TemplatePath = "$PSScriptRoot\Templates\aks.json"
                Parameters = @{
                    clusterName = @{ Type = "string"; DefaultValue = "myaks" }
                    nodeCount = @{ Type = "int"; DefaultValue = 3 }
                    nodeSize = @{ Type = "string"; DefaultValue = "Standard_DS2_v2" }
                }
            }
        }
    }

    [PSObject]DeployTemplate([string]$templateName, [hashtable]$parameters, [string]$resourceGroup) {
        if (-not $this.DeploymentTemplates.ContainsKey($templateName)) {
            throw "Deployment template '$templateName' not found"
        }

        $template = $this.DeploymentTemplates[$templateName]

        Write-Host "Deploying template: $templateName to resource group: $resourceGroup" -ForegroundColor Cyan

        # Fusion des paramètres
        $deploymentParams = @{}
        foreach ($paramName in $template.Parameters.Keys) {
            $deploymentParams[$paramName] = $parameters.ContainsKey($paramName) ? $parameters[$paramName] : $template.Parameters[$paramName].DefaultValue
        }

        try {
            $deployment = New-AzResourceGroupDeployment -ResourceGroupName $resourceGroup -TemplateFile $template.TemplatePath -TemplateParameterObject $deploymentParams

            $this.Deployments[$deployment.DeploymentName] = @{
                Template = $templateName
                Parameters = $deploymentParams
                Result = $deployment
                Timestamp = Get-Date
            }

            $this.LogOperation("TemplateDeployed", "Template deployed successfully", @{
                TemplateName = $templateName
                ResourceGroup = $resourceGroup
                DeploymentName = $deployment.DeploymentName
            })

            return $deployment

        } catch {
            $this.LogOperation("TemplateDeploymentFailed", "Template deployment failed", @{
                TemplateName = $templateName
                ResourceGroup = $resourceGroup
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]CreateWebApplication([hashtable]$webAppConfig) {
        Write-Host "Creating web application: $($webAppConfig.Name)" -ForegroundColor Yellow

        $rgName = $webAppConfig.ResourceGroup ?? "$($this.AzureConfig.ResourceGroupPrefix)-webapps"

        if (-not (Get-AzResourceGroup -Name $rgName -ErrorAction SilentlyContinue)) {
            $this.CreateResourceGroup($rgName)
        }

        # Création du plan App Service
        $planName = $webAppConfig.PlanName ?? "$($webAppConfig.Name)-plan"
        $plan = New-AzAppServicePlan -ResourceGroupName $rgName -Name $planName -Location ($webAppConfig.Location ?? $this.AzureConfig.Location) -Tier ($webAppConfig.Tier ?? "Free")

        # Création de l'application web
        $webAppParams = @{
            ResourceGroupName = $rgName
            Name = $webAppConfig.Name
            Location = $webAppConfig.Location ?? $this.AzureConfig.Location
            AppServicePlan = $planName
        }

        if ($webAppConfig.Runtime) { $webAppParams.Runtime = $webAppConfig.Runtime }
        if ($webAppConfig.DeploymentSource) { $webAppParams.DeploymentSourceUrl = $webAppConfig.DeploymentSource }

        $webApp = New-AzWebApp @webAppParams

        # Configuration supplémentaire
        if ($webAppConfig.AppSettings) {
            Set-AzWebApp -ResourceGroupName $rgName -Name $webAppConfig.Name -AppSettings $webAppConfig.AppSettings
        }

        if ($webAppConfig.ConnectionStrings) {
            foreach ($conn in $webAppConfig.ConnectionStrings) {
                $webApp | New-AzWebAppDatabaseConnectionString @conn
            }
        }

        $this.LogOperation("WebAppCreated", "Web application created successfully", @{
            WebAppName = $webAppConfig.Name
            ResourceGroup = $rgName
            Plan = $planName
        })

        $this.RefreshResourceInventory()
        return $webApp
    }

    [PSObject]CreateAKSCluster([hashtable]$aksConfig) {
        Write-Host "Creating AKS cluster: $($aksConfig.Name)" -ForegroundColor Yellow

        $rgName = $aksConfig.ResourceGroup ?? "$($this.AzureConfig.ResourceGroupPrefix)-aks"

        if (-not (Get-AzResourceGroup -Name $rgName -ErrorAction SilentlyContinue)) {
            $this.CreateResourceGroup($rgName)
        }

        $aksParams = @{
            ResourceGroupName = $rgName
            Name = $aksConfig.Name
            NodeCount = $aksConfig.NodeCount ?? 3
            NodeVmSize = $aksConfig.NodeVmSize ?? "Standard_DS2_v2"
            Location = $aksConfig.Location ?? $this.AzureConfig.Location
        }

        if ($aksConfig.KubernetesVersion) { $aksParams.KubernetesVersion = $aksConfig.KubernetesVersion }
        if ($aksConfig.NetworkPlugin) { $aksParams.NetworkPlugin = $aksConfig.NetworkPlugin }
        if ($aksConfig.EnableRBAC) { $aksParams.EnableRBAC = $aksConfig.EnableRBAC }

        $aksCluster = New-AzAksCluster @aksParams

        # Configuration post-déploiement
        if ($aksConfig.Addons) {
            foreach ($addon in $aksConfig.Addons) {
                switch ($addon) {
                    "Monitoring" { 
                        # Activer Azure Monitor pour containers
                        # (Code d'activation du monitoring)
                    }
                    "VirtualNode" {
                        # Activer Virtual Node (ACI)
                        # (Code d'activation Virtual Node)
                    }
                }
            }
        }

        $this.LogOperation("AKSCreated", "AKS cluster created successfully", @{
            ClusterName = $aksConfig.Name
            ResourceGroup = $rgName
            NodeCount = $aksParams.NodeCount
        })

        $this.RefreshResourceInventory()
        return $aksCluster
    }

    [hashtable]GetDeploymentStatus([string]$deploymentName) {
        try {
            $deployment = Get-AzResourceGroupDeployment -DeploymentName $deploymentName

            return @{
                Name = $deployment.DeploymentName
                Status = $deployment.ProvisioningState
                Timestamp = $deployment.Timestamp
                Duration = (Get-Date) - $deployment.Timestamp
                Outputs = $deployment.Outputs
                Parameters = $deployment.Parameters
                TemplateLink = $deployment.TemplateLink
            }
        } catch {
            return @{ Error = $_.Exception.Message }
        }
    }

    [void]ScaleAKSCluster([string]$clusterName, [string]$resourceGroup, [int]$newNodeCount) {
        Write-Host "Scaling AKS cluster $clusterName to $newNodeCount nodes" -ForegroundColor Yellow

        try {
            $aksCluster = Get-AzAksCluster -ResourceGroupName $resourceGroup -Name $clusterName
            $aksCluster | Set-AzAksCluster -NodeCount $newNodeCount

            $this.LogOperation("AKSScaled", "AKS cluster scaled successfully", @{
                ClusterName = $clusterName
                ResourceGroup = $resourceGroup
                NewNodeCount = $newNodeCount
            })

        } catch {
            $this.LogOperation("AKSScaleFailed", "Failed to scale AKS cluster", @{
                ClusterName = $clusterName
                ResourceGroup = $resourceGroup
                Error = $_.Exception.Message
            })
            throw
        }
    }
}

# Démonstration du framework Azure
$azureCred = Get-Credential -Message "Enter Azure credentials"
$azureManager = [AzureDeploymentOrchestrator]::new("your-subscription-id", "your-tenant-id", $azureCred)

# Création d'une application web complète
$webAppConfig = @{
    Name = "myPowerShellWebApp-$(Get-Date -Format 'yyyyMMddHHmmss')"
    Location = "East US"
    Tier = "B1"
    Runtime = "dotnetcore:3.1"
    AppSettings = @{
        "APP_ENV" = "Production"
        "LOG_LEVEL" = "Information"
    }
}

try {
    $webApp = $azureManager.CreateWebApplication($webAppConfig)
    Write-Host "Web application created: $($webApp.DefaultHostName)" -ForegroundColor Green
} catch {
    Write-Warning "Failed to create web application: $_"
}

# Déploiement d'un cluster AKS
$aksConfig = @{
    Name = "myPowerShellAKS-$(Get-Date -Format 'yyyyMMddHHmmss')"
    NodeCount = 2
    NodeVmSize = "Standard_DS2_v2"
    EnableRBAC = $true
    NetworkPlugin = "azure"
    Addons = @("Monitoring")
}

try {
    $aksCluster = $azureManager.CreateAKSCluster($aksConfig)
    Write-Host "AKS cluster created: $($aksCluster.Fqdn)" -ForegroundColor Green
} catch {
    Write-Warning "Failed to create AKS cluster: $_"
}

# Génération du rapport Azure
$azureReport = $azureManager.GenerateReport()
$azureReport | Out-File -FilePath "Azure_Management_Report_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nAzure management demonstration completed!" -ForegroundColor Green
```

### 1.2 Azure Automation et Runbooks

**Système avancé d'automatisation Azure :**
```powershell
# Framework d'automatisation Azure avancé
class AzureAutomationManager : AzureResourceManager {
    [string]$AutomationAccountName
    [string]$ResourceGroup
    [System.Collections.Generic.Dictionary[string, PSObject]]$Runbooks
    [System.Collections.Generic.Dictionary[string, PSObject]]$Schedules
    [hashtable]$AutomationConfig

    AzureAutomationManager([string]$subscriptionId, [string]$tenantId, [PSCredential]$credentials, [string]$automationAccount, [string]$resourceGroup) 
        : base($subscriptionId, $tenantId, $credentials) {
        
        $this.AutomationAccountName = $automationAccount
        $this.ResourceGroup = $resourceGroup
        $this.Runbooks = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.Schedules = [System.Collections.Generic.Dictionary[string, PSObject]]::new()

        $this.AutomationConfig = @{
            Location = $this.AzureConfig.Location
            Tags = $this.AzureConfig.Tags
            LogRetentionDays = 30
            MaxConcurrency = 10
        }

        $this.EnsureAutomationAccount()
        $this.RefreshAutomationInventory()
    }

    hidden [void]EnsureAutomationAccount() {
        Write-Host "Ensuring Azure Automation account exists..." -ForegroundColor Gray

        try {
            $aa = Get-AzAutomationAccount -Name $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -ErrorAction SilentlyContinue

            if (-not $aa) {
                Write-Host "Creating Azure Automation account: $($this.AutomationAccountName)" -ForegroundColor Yellow

                $aa = New-AzAutomationAccount -Name $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -Location $this.AutomationConfig.Location -Plan Basic -Tags $this.AutomationConfig.Tags

                $this.LogOperation("AutomationAccountCreated", "Azure Automation account created", @{
                    AccountName = $this.AutomationAccountName
                    ResourceGroup = $this.ResourceGroup
                })
            }
        } catch {
            Write-Error "Failed to create/verify Automation account: $_"
            throw
        }
    }

    hidden [void]RefreshAutomationInventory() {
        Write-Host "Refreshing Automation inventory..." -ForegroundColor Gray

        try {
            $runbooks = Get-AzAutomationRunbook -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup
            $this.Runbooks.Clear()
            foreach ($rb in $runbooks) {
                $this.Runbooks[$rb.Name] = $rb
            }

            $schedules = Get-AzAutomationSchedule -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup
            $this.Schedules.Clear()
            foreach ($sched in $schedules) {
                $this.Schedules[$sched.Name] = $sched
            }

        } catch {
            Write-Warning "Failed to refresh Automation inventory: $_"
        }
    }

    [PSObject]CreateRunbook([hashtable]$runbookConfig) {
        Write-Host "Creating runbook: $($runbookConfig.Name)" -ForegroundColor Yellow

        $runbookParams = @{
            AutomationAccountName = $this.AutomationAccountName
            ResourceGroupName = $this.ResourceGroup
            Name = $runbookConfig.Name
            Type = $runbookConfig.Type ?? "PowerShell"
            Description = $runbookConfig.Description ?? ""
            LogProgress = $runbookConfig.LogProgress ?? $true
            LogVerbose = $runbookConfig.LogVerbose ?? $true
        }

        try {
            # Création du runbook
            $runbook = New-AzAutomationRunbook @runbookParams

            # Import du contenu si fourni
            if ($runbookConfig.Content) {
                $tempFile = [System.IO.Path]::GetTempFileName() + ".ps1"
                $runbookConfig.Content | Out-File -FilePath $tempFile -Encoding UTF8

                Import-AzAutomationRunbook -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -Name $runbookConfig.Name -Path $tempFile -Type $runbookParams.Type -Published

                Remove-Item $tempFile -Force
            }

            $this.RefreshAutomationInventory()

            $this.LogOperation("RunbookCreated", "Runbook created successfully", @{
                RunbookName = $runbookConfig.Name
                Type = $runbookParams.Type
                Published = $runbookConfig.Content ? $true : $false
            })

            return $runbook

        } catch {
            $this.LogOperation("RunbookCreateFailed", "Failed to create runbook", @{
                RunbookName = $runbookConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]StartRunbook([string]$runbookName, [hashtable]$parameters = @{}) {
        Write-Host "Starting runbook: $runbookName" -ForegroundColor Yellow

        try {
            $job = Start-AzAutomationRunbook -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -Name $runbookName -Parameters $parameters

            $this.LogOperation("RunbookStarted", "Runbook started successfully", @{
                RunbookName = $runbookName
                JobId = $job.JobId
                Parameters = $parameters
            })

            return $job

        } catch {
            $this.LogOperation("RunbookStartFailed", "Failed to start runbook", @{
                RunbookName = $runbookName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]GetRunbookJobStatus([string]$jobId) {
        try {
            $job = Get-AzAutomationJob -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -Id $jobId

            return @{
                JobId = $job.JobId
                RunbookName = $job.RunbookName
                Status = $job.Status
                StartTime = $job.StartTime
                EndTime = $job.EndTime
                Duration = if ($job.EndTime) { $job.EndTime - $job.StartTime } else { (Get-Date) - $job.StartTime }
                Output = Get-AzAutomationJobOutput -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -Id $jobId
                Exceptions = Get-AzAutomationJobOutput -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -Id $jobId -Stream Exception
            }
        } catch {
            return @{ Error = $_.Exception.Message }
        }
    }

    [PSObject]CreateSchedule([hashtable]$scheduleConfig) {
        Write-Host "Creating schedule: $($scheduleConfig.Name)" -ForegroundColor Yellow

        $scheduleParams = @{
            AutomationAccountName = $this.AutomationAccountName
            ResourceGroupName = $this.ResourceGroup
            Name = $scheduleConfig.Name
            StartTime = $scheduleConfig.StartTime ?? (Get-Date).AddMinutes(5)
            TimeZone = $scheduleConfig.TimeZone ?? (Get-TimeZone).Id
        }

        # Configuration du type de planification
        if ($scheduleConfig.OneTime) {
            # Planification ponctuelle
        } elseif ($scheduleConfig.Daily) {
            $scheduleParams.DaysOfWeek = $scheduleConfig.DaysOfWeek ?? @("Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday")
            $scheduleParams.DayInterval = $scheduleConfig.DayInterval ?? 1
        } elseif ($scheduleConfig.Weekly) {
            $scheduleParams.WeekInterval = $scheduleConfig.WeekInterval ?? 1
        } elseif ($scheduleConfig.Monthly) {
            $scheduleParams.MonthInterval = $scheduleConfig.MonthInterval ?? 1
        }

        try {
            $schedule = New-AzAutomationSchedule @scheduleParams

            $this.Schedules[$scheduleConfig.Name] = $schedule
            $this.LogOperation("ScheduleCreated", "Schedule created successfully", @{
                ScheduleName = $scheduleConfig.Name
                StartTime = $scheduleParams.StartTime
            })

            return $schedule

        } catch {
            $this.LogOperation("ScheduleCreateFailed", "Failed to create schedule", @{
                ScheduleName = $scheduleConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]LinkRunbookToSchedule([string]$runbookName, [string]$scheduleName, [hashtable]$parameters = @{}) {
        Write-Host "Linking runbook '$runbookName' to schedule '$scheduleName'" -ForegroundColor Yellow

        try {
            Register-AzAutomationScheduledRunbook -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup -RunbookName $runbookName -ScheduleName $scheduleName -Parameters $parameters

            $this.LogOperation("RunbookScheduled", "Runbook linked to schedule successfully", @{
                RunbookName = $runbookName
                ScheduleName = $scheduleName
            })

        } catch {
            $this.LogOperation("RunbookScheduleFailed", "Failed to link runbook to schedule", @{
                RunbookName = $runbookName
                ScheduleName = $scheduleName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]CreateBackupRunbook() {
        $backupRunbookContent = @'
param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName,

    [Parameter(Mandatory=$true)]
    [string]$StorageAccountName,

    [Parameter(Mandatory=$true)]
    [string]$ContainerName,

    [string]$BackupPrefix = "backup"
)

Write-Output "Starting Azure resource backup process..."

# Authentification automatique (dans le contexte Azure Automation)
$connection = Get-AutomationConnection -Name "AzureRunAsConnection"
Connect-AzAccount -ServicePrincipal -Tenant $connection.TenantID -ApplicationId $connection.ApplicationID -CertificateThumbprint $connection.CertificateThumbprint

# Création d'un snapshot de timestamp
$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupName = "$BackupPrefix`_$timestamp"

Write-Output "Creating backup: $backupName"

# Sauvegarde des VMs
$vms = Get-AzVM -ResourceGroupName $ResourceGroupName
foreach ($vm in $vms) {
    Write-Output "Creating snapshot for VM: $($vm.Name)"
    
    $snapshotConfig = @{
        Name = "$($vm.Name)_$backupName"
        ResourceGroupName = $ResourceGroupName
        Location = $vm.Location
        SourceUri = $vm.StorageProfile.OsDisk.Vhd.Uri
        CreateOption = "Copy"
    }
    
    New-AzSnapshot @snapshotConfig
}

# Sauvegarde des bases de données SQL
$sqlServers = Get-AzSqlServer -ResourceGroupName $ResourceGroupName
foreach ($server in $sqlServers) {
    $databases = Get-AzSqlDatabase -ServerName $server.ServerName -ResourceGroupName $ResourceGroupName | Where-Object { $_.DatabaseName -ne "master" }
    
    foreach ($db in $databases) {
        Write-Output "Creating backup for database: $($db.DatabaseName)"
        
        $backupFileName = "$($db.DatabaseName)_$backupName.bacpac"
        $bacpacUri = "https://$StorageAccountName.blob.core.windows.net/$ContainerName/$backupFileName"
        
        New-AzSqlDatabaseExport -DatabaseName $db.DatabaseName -ServerName $server.ServerName -ResourceGroupName $ResourceGroupName -StorageKeyType "StorageAccessKey" -StorageKey (Get-AzStorageAccountKey -ResourceGroupName $ResourceGroupName -Name $StorageAccountName).Value[0] -StorageUri $bacpacUri -AdministratorLogin $server.SqlAdministratorLogin -AdministratorLoginPassword (ConvertTo-SecureString "TempPassword123!" -AsPlainText -Force)
    }
}

Write-Output "Backup process completed successfully"
'@

        $backupRunbook = @{
            Name = "Azure-Backup-Automation"
            Description = "Automated backup of Azure resources"
            Type = "PowerShell"
            Content = $backupRunbookContent
        }

        $this.CreateRunbook($backupRunbook)
    }

    [void]CreateMonitoringRunbook() {
        $monitoringRunbookContent = @'
param(
    [Parameter(Mandatory=$true)]
    [string]$ResourceGroupName,

    [string]$AlertEmail = $null,

    [int]$CpuThreshold = 80,

    [int]$MemoryThreshold = 85
)

Write-Output "Starting Azure resource monitoring..."

# Authentification automatique
$connection = Get-AutomationConnection -Name "AzureRunAsConnection"
Connect-AzAccount -ServicePrincipal -Tenant $connection.TenantID -ApplicationId $connection.ApplicationID -CertificateThumbprint $connection.CertificateThumbprint

$alerts = @()

# Monitoring des VMs
$vms = Get-AzVM -ResourceGroupName $ResourceGroupName -Status
foreach ($vm in $vms) {
    $metrics = Get-AzMetric -ResourceId $vm.Id -MetricName "Percentage CPU" -TimeGrain 00:05:00 -StartTime (Get-Date).AddHours(-1)
    
    if ($metrics.Data) {
        $avgCpu = ($metrics.Data | Measure-Object -Property Average -Average).Average
        
        if ($avgCpu -gt $CpuThreshold) {
            $alerts += @{
                Type = "VM_CPU_High"
                Resource = $vm.Name
                Metric = "CPU"
                Value = $avgCpu
                Threshold = $CpuThreshold
                Severity = "Warning"
            }
        }
    }
}

# Monitoring des bases de données SQL
$sqlServers = Get-AzSqlServer -ResourceGroupName $ResourceGroupName
foreach ($server in $sqlServers) {
    $databases = Get-AzSqlDatabase -ServerName $server.ServerName -ResourceGroupName $ResourceGroupName
    
    foreach ($db in $databases) {
        $metrics = Get-AzMetric -ResourceId $db.Id -MetricName "dtu_consumption_percent" -TimeGrain 00:05:00 -StartTime (Get-Date).AddHours(-1)
        
        if ($metrics.Data) {
            $avgDtu = ($metrics.Data | Measure-Object -Property Average -Average).Average
            
            if ($avgDtu -gt 80) {  # DTU threshold
                $alerts += @{
                    Type = "SQL_DTU_High"
                    Resource = "$($server.ServerName)/$($db.DatabaseName)"
                    Metric = "DTU"
                    Value = $avgDtu
                    Threshold = 80
                    Severity = "Critical"
                }
            }
        }
    }
}

# Envoi d'alertes par email si configuré
if ($AlertEmail -and $alerts.Count -gt 0) {
    $subject = "Azure Resource Alerts - $($alerts.Count) issue(s) detected"
    $body = "The following alerts were detected:`n`n"
    
    foreach ($alert in $alerts) {
        $body += "• $($alert.Type): $($alert.Resource) - $($alert.Metric) = $([math]::Round($alert.Value, 2))% (Threshold: $($alert.Threshold)%))`n"
    }
    
    Send-MailMessage -To $AlertEmail -Subject $subject -Body $body -SmtpServer "smtp.office365.com" -UseSsl -Credential (Get-AutomationPSCredential -Name "AlertEmailCredential")
}

Write-Output "Monitoring completed. $($alerts.Count) alerts generated."
'@

        $monitoringRunbook = @{
            Name = "Azure-Monitoring-Automation"
            Description = "Automated monitoring of Azure resources"
            Type = "PowerShell"
            Content = $monitoringRunbookContent
        }

        $this.CreateRunbook($monitoringRunbook)
    }

    [hashtable]GetAutomationMetrics() {
        try {
            $jobs = Get-AzAutomationJob -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup

            $metrics = @{
                TotalJobs = $jobs.Count
                SuccessfulJobs = ($jobs | Where-Object { $_.Status -eq "Completed" }).Count
                FailedJobs = ($jobs | Where-Object { $_.Status -eq "Failed" }).Count
                RunningJobs = ($jobs | Where-Object { $_.Status -eq "Running" }).Count
                AverageJobDuration = $null
                RunbooksByType = @{}
                JobsByStatus = @{}
            }

            # Durée moyenne des jobs
            $completedJobs = $jobs | Where-Object { $_.Status -eq "Completed" -and $_.EndTime }
            if ($completedJobs) {
                $totalDuration = ($completedJobs | ForEach-Object { $_.EndTime - $_.StartTime } | Measure-Object -Sum Ticks).Sum
                $metrics.AverageJobDuration = [TimeSpan]::FromTicks($totalDuration / $completedJobs.Count)
            }

            # Répartition par statut
            $metrics.JobsByStatus = $jobs | Group-Object Status | ForEach-Object {
                @{ $_.Name = $_.Count }
            }

            # Runbooks par type
            $runbooks = Get-AzAutomationRunbook -AutomationAccountName $this.AutomationAccountName -ResourceGroupName $this.ResourceGroup
            $metrics.RunbooksByType = $runbooks | Group-Object RunbookType | ForEach-Object {
                @{ $_.Name = $_.Count }
            }

            return $metrics

        } catch {
            return @{ Error = $_.Exception.Message }
        }
    }
}

# Démonstration du système d'automatisation Azure
$automationCred = Get-Credential -Message "Enter Azure Automation credentials"
$automationManager = [AzureAutomationManager]::new("your-subscription-id", "your-tenant-id", $automationCred, "myAutomationAccount", "automation-rg")

# Création des runbooks automatisés
$automationManager.CreateBackupRunbook()
$automationManager.CreateMonitoringRunbook()

# Création d'un horaire de sauvegarde
$backupSchedule = @{
    Name = "Daily-Backup-Schedule"
    Daily = $true
    StartTime = (Get-Date).AddDays(1).Date.AddHours(2)  # 2h du matin
}

$schedule = $automationManager.CreateSchedule($backupSchedule)

# Liaison du runbook de sauvegarde à l'horaire
$automationManager.LinkRunbookToSchedule("Azure-Backup-Automation", $schedule.Name, @{
    ResourceGroupName = "production-rg"
    StorageAccountName = "backupstorageacct"
    ContainerName = "backups"
})

# Exécution manuelle du runbook de monitoring
$job = $automationManager.StartRunbook("Azure-Monitoring-Automation", @{
    ResourceGroupName = "production-rg"
    AlertEmail = "admin@company.com"
    CpuThreshold = 75
    MemoryThreshold = 80
})

Write-Host "Monitoring job started with ID: $($job.JobId)" -ForegroundColor Green

# Attente et vérification du statut
Start-Sleep -Seconds 30
$jobStatus = $automationManager.GetRunbookJobStatus($job.JobId)
Write-Host "Job status: $($jobStatus.Status)" -ForegroundColor Yellow

# Métriques d'automatisation
$automationMetrics = $automationManager.GetAutomationMetrics()
Write-Host "`n=== AUTOMATION METRICS ===" -ForegroundColor Cyan
Write-Host "Total jobs: $($automationMetrics.TotalJobs)"
Write-Host "Success rate: $([math]::Round(($automationMetrics.SuccessfulJobs / $automationMetrics.TotalJobs) * 100, 1))%"
if ($automationMetrics.AverageJobDuration) {
    Write-Host "Average job duration: $([math]::Round($automationMetrics.AverageJobDuration.TotalSeconds, 1)) seconds"
}

Write-Host "`nAzure Automation demonstration completed!" -ForegroundColor Green
```

## Section 2 : AWS PowerShell Tools

### 2.1 Gestion avancée des ressources AWS

**Framework complet de gestion AWS :**
```powershell
# Framework avancé de gestion AWS avec PowerShell
class AWSResourceManager {
    [string]$ProfileName
    [string]$Region
    [System.Collections.Generic.Dictionary[string, PSObject]]$EC2Instances
    [System.Collections.Generic.Dictionary[string, PSObject]]$S3Buckets
    [System.Collections.Generic.List[PSCustomObject]]$OperationsLog
    [hashtable]$AWSConfig

    AWSResourceManager([string]$profileName = "default", [string]$region = "us-east-1") {
        $this.ProfileName = $profileName
        $this.Region = $region
        $this.EC2Instances = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.S3Buckets = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.OperationsLog = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.AWSConfig = @{
            KeyName = "default-keypair"
            SecurityGroup = "default-sg"
            InstanceProfile = "EC2InstanceProfile"
            Tags = @(@{ Key = "ManagedBy"; Value = "PowerShell" }, @{ Key = "Environment"; Value = "Development" })
        }

        $this.InitializeAWSCredentials()
        $this.RefreshResourceInventory()
    }

    hidden [void]InitializeAWSCredentials() {
        Write-Host "Initializing AWS credentials..." -ForegroundColor Cyan

        try {
            # Vérification des credentials AWS
            $credentials = Get-AWSCredentials -ProfileName $this.ProfileName
            Set-AWSCredentials -ProfileName $this.ProfileName
            Set-DefaultAWSRegion -Region $this.Region

            Write-Host "AWS credentials initialized for profile: $($this.ProfileName)" -ForegroundColor Green

        } catch {
            Write-Error "Failed to initialize AWS credentials: $_"
            throw
        }
    }

    hidden [void]RefreshResourceInventory() {
        Write-Host "Refreshing AWS resource inventory..." -ForegroundColor Gray

        try {
            # Récupération des instances EC2
            $instances = Get-EC2Instance
            $this.EC2Instances.Clear()
            foreach ($reservation in $instances) {
                foreach ($instance in $reservation.Instances) {
                    $this.EC2Instances[$instance.InstanceId] = $instance
                }
            }

            # Récupération des buckets S3
            $buckets = Get-S3Bucket
            $this.S3Buckets.Clear()
            foreach ($bucket in $buckets) {
                $this.S3Buckets[$bucket.BucketName] = $bucket
            }

            $this.LogOperation("InventoryRefreshed", "AWS resource inventory refreshed", @{
                EC2Instances = $this.EC2Instances.Count
                S3Buckets = $this.S3Buckets.Count
            })

        } catch {
            Write-Warning "Failed to refresh AWS inventory: $_"
        }
    }

    [PSObject]CreateEC2Instance([hashtable]$instanceConfig) {
        Write-Host "Creating EC2 instance: $($instanceConfig.Name)" -ForegroundColor Yellow

        # Configuration par défaut
        $amiId = $instanceConfig.AMI ?? "ami-0abcdef1234567890"  # Remplacer par AMI appropriée
        $instanceType = $instanceConfig.InstanceType ?? "t3.micro"
        $keyName = $instanceConfig.KeyName ?? $this.AWSConfig.KeyName
        $securityGroupIds = $instanceConfig.SecurityGroupIds ?? @()
        $subnetId = $instanceConfig.SubnetId
        $userData = $instanceConfig.UserData

        $params = @{
            ImageId = $amiId
            InstanceType = $instanceType
            KeyName = $keyName
            MinCount = 1
            MaxCount = 1
        }

        if ($securityGroupIds) { $params.SecurityGroupId = $securityGroupIds }
        if ($subnetId) { $params.SubnetId = $subnetId }
        if ($userData) { $params.UserData = $userData }

        # Tags
        $tags = $this.AWSConfig.Tags + @(@{ Key = "Name"; Value = $instanceConfig.Name })
        if ($instanceConfig.Tags) {
            $tags += $instanceConfig.Tags
        }
        $params.TagSpecification = @(@{
            ResourceType = "instance"
            Tag = $tags
        })

        try {
            $result = New-EC2Instance @params
            $instance = $result.Instances[0]

            # Attente que l'instance soit en cours d'exécution
            Write-Host "Waiting for instance to be running..." -ForegroundColor Gray
            $instance = (Get-EC2Instance -InstanceId $instance.InstanceId).Instances[0]
            while ($instance.State.Name -ne "running") {
                Start-Sleep -Seconds 10
                $instance = (Get-EC2Instance -InstanceId $instance.InstanceId).Instances[0]
            }

            $this.RefreshResourceInventory()

            $this.LogOperation("EC2Created", "EC2 instance created successfully", @{
                InstanceId = $instance.InstanceId
                InstanceName = $instanceConfig.Name
                InstanceType = $instanceType
                State = $instance.State.Name
            })

            return $instance

        } catch {
            $this.LogOperation("EC2CreateFailed", "Failed to create EC2 instance", @{
                InstanceName = $instanceConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]CreateS3Bucket([hashtable]$bucketConfig) {
        Write-Host "Creating S3 bucket: $($bucketConfig.Name)" -ForegroundColor Yellow

        $bucketName = $bucketConfig.Name
        $region = $bucketConfig.Region ?? $this.Region

        try {
            # Création du bucket
            $bucket = New-S3Bucket -BucketName $bucketName -Region $region

            # Configuration du versioning si demandé
            if ($bucketConfig.Versioning) {
                Write-S3BucketVersioning -BucketName $bucketName -VersioningConfiguration_Status Enabled
            }

            # Configuration du chiffrement par défaut
            if ($bucketConfig.Encryption) {
                $encryptionConfig = @{
                    ServerSideEncryptionConfiguration = @{
                        ServerSideEncryptionRule = @(
                            @{
                                ServerSideEncryptionByDefault = @{
                                    SSEAlgorithm = "AES256"
                                }
                            }
                        )
                    }
                }
                Write-S3BucketEncryption -BucketName $bucketName -ServerSideEncryptionConfiguration $encryptionConfig
            }

            # Configuration des politiques d'accès
            if ($bucketConfig.Policy) {
                Write-S3BucketPolicy -BucketName $bucketName -Policy $bucketConfig.Policy
            }

            $this.RefreshResourceInventory()

            $this.LogOperation("S3BucketCreated", "S3 bucket created successfully", @{
                BucketName = $bucketName
                Region = $region
                Versioning = $bucketConfig.Versioning
                Encryption = $bucketConfig.Encryption
            })

            return $bucket

        } catch {
            $this.LogOperation("S3BucketCreateFailed", "Failed to create S3 bucket", @{
                BucketName = $bucketName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]StartEC2Instance([string]$instanceId) {
        $this.ExecuteEC2Operation($instanceId, "Start", {
            param($id)
            Start-EC2Instance -InstanceId $id
        })
    }

    [void]StopEC2Instance([string]$instanceId) {
        $this.ExecuteEC2Operation($instanceId, "Stop", {
            param($id)
            Stop-EC2Instance -InstanceId $id
        })
    }

    [void]TerminateEC2Instance([string]$instanceId) {
        Write-Host "Terminating EC2 instance: $instanceId" -ForegroundColor Red

        try {
            Remove-EC2Instance -InstanceId $instanceId -Force
            $this.RefreshResourceInventory()

            $this.LogOperation("EC2Terminated", "EC2 instance terminated", @{
                InstanceId = $instanceId
            })

        } catch {
            $this.LogOperation("EC2TerminateFailed", "Failed to terminate EC2 instance", @{
                InstanceId = $instanceId
                Error = $_.Exception.Message
            })
            throw
        }
    }

    hidden [void]ExecuteEC2Operation([string]$instanceId, [string]$operation, [scriptblock]$action) {
        if (-not $this.EC2Instances.ContainsKey($instanceId)) {
            throw "EC2 instance '$instanceId' not found"
        }

        Write-Host "${operation}ing EC2 instance: $instanceId" -ForegroundColor Yellow

        try {
            & $action $instanceId

            # Attente de la fin de l'opération
            Start-Sleep -Seconds 5

            $this.RefreshResourceInventory()

            $this.LogOperation("EC2Operation", "EC2 operation completed", @{
                InstanceId = $instanceId
                Operation = $operation
            })

        } catch {
            $this.LogOperation("EC2OperationFailed", "EC2 operation failed", @{
                InstanceId = $instanceId
                Operation = $operation
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]CreateRDSInstance([hashtable]$rdsConfig) {
        Write-Host "Creating RDS instance: $($rdsConfig.Name)" -ForegroundColor Yellow

        $params = @{
            DBInstanceIdentifier = $rdsConfig.Name
            DBInstanceClass = $rdsConfig.InstanceClass ?? "db.t3.micro"
            Engine = $rdsConfig.Engine ?? "mysql"
            MasterUsername = $rdsConfig.MasterUsername ?? "admin"
            MasterUserPassword = $rdsConfig.MasterUserPassword
            AllocatedStorage = $rdsConfig.AllocatedStorage ?? 20
            DBName = $rdsConfig.DBName
            VpcSecurityGroupId = $rdsConfig.SecurityGroupIds
            DBSubnetGroupName = $rdsConfig.DBSubnetGroupName
        }

        try {
            $rdsInstance = New-RDSDBInstance @params

            # Attente que l'instance soit disponible
            Write-Host "Waiting for RDS instance to be available..." -ForegroundColor Gray
            $instance = Get-RDSDBInstance -DBInstanceIdentifier $rdsConfig.Name
            while ($instance.DBInstanceStatus -ne "available") {
                Start-Sleep -Seconds 30
                $instance = Get-RDSDBInstance -DBInstanceIdentifier $rdsConfig.Name
            }

            $this.LogOperation("RDSCreated", "RDS instance created successfully", @{
                DBInstanceIdentifier = $rdsConfig.Name
                Engine = $params.Engine
                InstanceClass = $params.DBInstanceClass
            })

            return $instance

        } catch {
            $this.LogOperation("RDSCreateFailed", "Failed to create RDS instance", @{
                DBInstanceIdentifier = $rdsConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [hashtable]GetAWSMetrics() {
        try {
            $metrics = @{
                EC2Instances = @{
                    Total = $this.EC2Instances.Count
                    Running = ($this.EC2Instances.Values | Where-Object { $_.State.Name -eq "running" }).Count
                    Stopped = ($this.EC2Instances.Values | Where-Object { $_.State.Name -eq "stopped" }).Count
                }
                S3Buckets = @{
                    Total = $this.S3Buckets.Count
                    TotalSizeBytes = 0
                }
                Costs = $this.GetAWSCosts()
            }

            # Calcul de la taille totale S3 (estimation)
            foreach ($bucketName in $this.S3Buckets.Keys) {
                try {
                    $objects = Get-S3Object -BucketName $bucketName
                    $totalSize = ($objects | Measure-Object -Property Size -Sum).Sum
                    $metrics.S3Buckets.TotalSizeBytes += $totalSize
                } catch {
                    Write-Debug "Failed to get size for bucket $bucketName`: $_"
                }
            }

            return $metrics

        } catch {
            return @{ Error = $_.Exception.Message }
        }
    }

    hidden [hashtable]GetAWSCosts() {
        try {
            # Simulation de récupération des coûts (nécessiterait AWS Cost Explorer API)
            return @{
                TotalCost = 0
                ByService = @{}
                ByRegion = @{}
            }
        } catch {
            return @{ Error = $_.Exception.Message }
        }
    }

    [PSObject[]]GetEC2Instances([string]$filter = "*") {
        return $this.EC2Instances.Values | Where-Object { 
            $_.Tags | Where-Object { $_.Key -eq "Name" -and $_.Value -like $filter }
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
        $metrics = $this.GetAWSMetrics()

        $report = @"
AWS RESOURCE MANAGEMENT REPORT
==============================
Profile: $($this.ProfileName)
Region: $($this.Region)
Generated: $(Get-Date)

EC2 INSTANCES
=============
Total Instances: $($metrics.EC2Instances.Total)
Running: $($metrics.EC2Instances.Running)
Stopped: $($metrics.EC2Instances.Stopped)

S3 BUCKETS
==========
Total Buckets: $($metrics.S3Buckets.Total)
Total Size: $([math]::Round($metrics.S3Buckets.TotalSizeBytes / 1GB, 2)) GB

COST ANALYSIS
=============
Total Cost: `$$([math]::Round($metrics.Costs.TotalCost, 2))

INSTANCE DETAILS
================
"@

        foreach ($instance in $this.EC2Instances.Values | Sort-Object LaunchTime -Descending | Select-Object -First 10) {
            $name = ($instance.Tags | Where-Object { $_.Key -eq "Name" }).Value
            $report += @"

$name ($($instance.InstanceId))
  State: $($instance.State.Name)
  Type: $($instance.InstanceType)
  Launch Time: $($instance.LaunchTime)
  Private IP: $($instance.PrivateIpAddress)
  Public IP: $($instance.PublicIpAddress)
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

# Orchestrateur multi-cloud (Azure + AWS)
class MultiCloudOrchestrator {
    [AzureResourceManager]$AzureManager
    [AWSResourceManager]$AWSManager
    [System.Collections.Generic.List[PSCustomObject]]$CloudOperations
    [hashtable]$CloudPolicies

    MultiCloudOrchestrator([AzureResourceManager]$azure, [AWSResourceManager]$aws) {
        $this.AzureManager = $azure
        $this.AWSManager = $aws
        $this.CloudOperations = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.CloudPolicies = @{
            FailoverPolicy = @{
                PrimaryCloud = "Azure"
                SecondaryCloud = "AWS"
                AutoFailover = $true
                HealthCheckInterval = [TimeSpan]::FromMinutes(5)
            }
            CostOptimization = @{
                PreferredCloud = "AWS"  # Moins cher
                MaxAzureInstances = 5
                MaxAWSInstances = 10
            }
            Compliance = @{
                DataResidency = "EU"  # Contrainte de résidence des données
                EncryptionRequired = $true
                AuditLogging = $true
            }
        }
    }

    [PSCustomObject]DeployApplication([hashtable]$appConfig) {
        Write-Host "Deploying application across clouds: $($appConfig.Name)" -ForegroundColor Cyan

        $deployment = [PSCustomObject]@{
            ApplicationName = $appConfig.Name
            DeploymentId = [guid]::NewGuid().ToString()
            StartTime = Get-Date
            EndTime = $null
            Status = "Initializing"
            AzureResources = @()
            AWSResources = @()
            TotalCost = 0
        }

        $this.CloudOperations.Add($deployment)

        try {
            # Détermination de la stratégie de déploiement
            $strategy = $this.DetermineDeploymentStrategy($appConfig)

            # Déploiement selon la stratégie
            switch ($strategy.Cloud) {
                "Azure" {
                    $deployment.AzureResources = $this.DeployToAzure($appConfig, $strategy)
                }
                "AWS" {
                    $deployment.AWSResources = $this.DeployToAWS($appConfig, $strategy)
                }
                "Hybrid" {
                    $deployment.AzureResources = $this.DeployToAzure($appConfig, $strategy)
                    $deployment.AWSResources = $this.DeployToAWS($appConfig, $strategy)
                }
            }

            $deployment.Status = "Completed"
            $deployment.EndTime = Get-Date
            $deployment.TotalCost = $this.CalculateDeploymentCost($deployment)

            Write-Host "Application deployment completed successfully" -ForegroundColor Green
            return $deployment

        } catch {
            $deployment.Status = "Failed"
            $deployment.EndTime = Get-Date
            Write-Error "Application deployment failed: $_"
            throw
        }
    }

    hidden [PSCustomObject]DetermineDeploymentStrategy([hashtable]$appConfig) {
        $strategy = @{
            Cloud = "Azure"  # Défaut
            InstanceCount = $appConfig.InstanceCount ?? 1
            InstanceType = $appConfig.InstanceType ?? "Standard_DS1_v2"
            Region = $appConfig.Region ?? "East US"
        }

        # Application des politiques
        if ($this.CloudPolicies.CostOptimization.PreferredCloud -eq "AWS" -and $appConfig.AllowAWS) {
            $strategy.Cloud = "AWS"
            $strategy.InstanceType = "t3.medium"
            $strategy.Region = "us-east-1"
        }

        # Contraintes de conformité
        if ($this.CloudPolicies.Compliance.DataResidency -eq "EU") {
            if ($strategy.Cloud -eq "Azure") {
                $strategy.Region = "West Europe"
            } else {
                $strategy.Region = "eu-west-1"
            }
        }

        return [PSCustomObject]$strategy
    }

    hidden [array]DeployToAzure([hashtable]$appConfig, [PSCustomObject]$strategy) {
        Write-Host "Deploying to Azure..." -ForegroundColor Yellow

        $resources = @()

        # Création du resource group
        $rgName = "$($appConfig.Name)-rg"
        $rg = $this.AzureManager.CreateResourceGroup($rgName)
        $resources += @{ Type = "ResourceGroup"; Name = $rgName; Resource = $rg }

        # Déploiement des VMs
        for ($i = 1; $i -le $strategy.InstanceCount; $i++) {
            $vmName = "$($appConfig.Name)-vm$i"
            $vmConfig = @{
                Name = $vmName
                InstanceType = $strategy.InstanceType
                Location = $strategy.Region
                Tags = @(@{ Key = "Application"; Value = $appConfig.Name })
            }

            $vm = $this.AzureManager.CreateVirtualMachine($vmConfig)
            $resources += @{ Type = "VirtualMachine"; Name = $vmName; Resource = $vm }
        }

        return $resources
    }

    hidden [array]DeployToAWS([hashtable]$appConfig, [PSCustomObject]$strategy) {
        Write-Host "Deploying to AWS..." -ForegroundColor Yellow

        $resources = @()

        # Déploiement des instances EC2
        for ($i = 1; $i -le $strategy.InstanceCount; $i++) {
            $instanceName = "$($appConfig.Name)-ec2$i"
            $instanceConfig = @{
                Name = $instanceName
                InstanceType = $strategy.InstanceType
                Tags = @(@{ Key = "Application"; Value = $appConfig.Name })
            }

            $instance = $this.AWSManager.CreateEC2Instance($instanceConfig)
            $resources += @{ Type = "EC2Instance"; Name = $instanceName; Resource = $instance }
        }

        # Création d'un bucket S3 pour l'application
        $bucketName = "$($appConfig.Name.ToLower())-storage-$(Get-Date -Format 'yyyyMMdd')"
        $bucketConfig = @{
            Name = $bucketName
            Region = $strategy.Region
            Versioning = $true
            Encryption = $true
        }

        $bucket = $this.AWSManager.CreateS3Bucket($bucketConfig)
        $resources += @{ Type = "S3Bucket"; Name = $bucketName; Resource = $bucket }

        return $resources
    }

    hidden [double]CalculateDeploymentCost([PSCustomObject]$deployment) {
        # Calcul simplifié des coûts (à améliorer avec les vraies APIs de pricing)
        $cost = 0

        # Coûts Azure
        foreach ($resource in $deployment.AzureResources) {
            switch ($resource.Type) {
                "VirtualMachine" { $cost += 0.1 } # Coût horaire estimé
                "ResourceGroup" { $cost += 0 }
            }
        }

        # Coûts AWS
        foreach ($resource in $deployment.AWSResources) {
            switch ($resource.Type) {
                "EC2Instance" { $cost += 0.08 } # Coût horaire estimé
                "S3Bucket" { $cost += 0.02 } # Coût mensuel estimé
            }
        }

        return [math]::Round($cost, 2)
    }

    [void]MonitorHealth() {
        Write-Host "Monitoring multi-cloud health..." -ForegroundColor Cyan

        # Monitoring Azure
        $azureHealth = @{
            VMsRunning = ($this.AzureManager.GetResources() | Where-Object { $_.ResourceType -eq "Microsoft.Compute/virtualMachines" }).Count
            WebAppsRunning = ($this.AzureManager.GetResources() | Where-Object { $_.ResourceType -eq "Microsoft.Web/sites" }).Count
        }

        # Monitoring AWS
        $awsMetrics = $this.AWSManager.GetAWSMetrics()
        $awsHealth = @{
            EC2Running = $awsMetrics.EC2Instances.Running
            S3Buckets = $awsMetrics.S3Buckets.Total
        }

        Write-Host "Azure Health - VMs: $($azureHealth.VMsRunning), WebApps: $($azureHealth.WebAppsRunning)" -ForegroundColor Green
        Write-Host "AWS Health - EC2: $($awsHealth.EC2Running), S3: $($awsHealth.S3Buckets)" -ForegroundColor Green

        # Vérification des seuils d'alerte
        if ($azureHealth.VMsRunning -eq 0 -and $awsHealth.EC2Running -eq 0) {
            Write-Warning "CRITICAL: No compute resources running across both clouds!"
        }
    }

    [void]OptimizeCosts() {
        Write-Host "Optimizing multi-cloud costs..." -ForegroundColor Cyan

        # Optimisation Azure
        $azureVMs = $this.AzureManager.GetResources() | Where-Object { $_.ResourceType -eq "Microsoft.Compute/virtualMachines" }
        foreach ($vm in $azureVMs) {
            # Logique d'optimisation (redimensionnement, arrêt, etc.)
            Write-Host "Analyzing Azure VM: $($vm.Name)"
        }

        # Optimisation AWS
        $awsInstances = $this.AWSManager.GetEC2Instances()
        foreach ($instance in $awsInstances) {
            # Logique d'optimisation similaire
            $name = ($instance.Tags | Where-Object { $_.Key -eq "Name" }).Value
            Write-Host "Analyzing AWS EC2: $name"
        }

        Write-Host "Cost optimization analysis completed" -ForegroundColor Green
    }

    [hashtable]GetMultiCloudMetrics() {
        $azureMetrics = $this.AzureManager.GetHostMetrics()
        $awsMetrics = $this.AWSManager.GetAWSMetrics()

        return @{
            Azure = $azureMetrics
            AWS = $awsMetrics
            TotalResources = ($azureMetrics.TotalResources ?? 0) + ($awsMetrics.EC2Instances.Total ?? 0) + ($awsMetrics.S3Buckets.Total ?? 0)
            TotalCost = ($azureMetrics.TotalCost ?? 0) + ($awsMetrics.Costs.TotalCost ?? 0)
            OperationsCount = $this.CloudOperations.Count
        }
    }
}

# Démonstration des intégrations cloud
# Note: Les exemples nécessitent des credentials et configurations appropriées

Write-Host "=== CLOUD INTEGRATIONS DEMONSTRATION ===" -ForegroundColor Cyan
Write-Host "Note: This demonstration requires proper cloud credentials and configurations" -ForegroundColor Yellow

# Simulation des frameworks (commentés pour éviter les erreurs sans credentials)
<#
$azureCred = Get-Credential -Message "Enter Azure credentials"
$azureManager = [AzureResourceManager]::new("your-subscription-id", "your-tenant-id", $azureCred)

$awsManager = [AWSResourceManager]::new("your-profile", "us-east-1")

$multiCloud = [MultiCloudOrchestrator]::new($azureManager, $awsManager)

# Déploiement d'une application multi-cloud
$appConfig = @{
    Name = "MultiCloudApp"
    InstanceCount = 2
    AllowAWS = $true
    Region = "East US"
}

$deployment = $multiCloud.DeployApplication($appConfig)
Write-Host "Application deployed with ID: $($deployment.DeploymentId)" -ForegroundColor Green

# Monitoring et optimisation
$multiCloud.MonitorHealth()
$multiCloud.OptimizeCosts()

# Métriques multi-cloud
$metrics = $multiCloud.GetMultiCloudMetrics()
Write-Host "Total resources across clouds: $($metrics.TotalResources)" -ForegroundColor Green
Write-Host "Total estimated cost: `$$($metrics.TotalCost)" -ForegroundColor Green
#>

Write-Host "`nCloud integrations demonstration framework prepared!" -ForegroundColor Green
Write-Host "Uncomment the code above and configure proper credentials to run the full demonstration." -ForegroundColor Yellow
```

## Conclusion : PowerShell comme orchestrateur cloud

Les intégrations cloud avancées transforment PowerShell en un orchestrateur puissant capable de gérer des infrastructures multi-cloud complexes. En maîtrisant Azure Resource Manager, AWS Tools, et les patterns d'orchestration multi-cloud, les développeurs PowerShell créent des architectures résilientes et évolutives.

Dans le prochain chapitre, nous explorerons Kubernetes et l'orchestration de conteneurs avec PowerShell, complétant notre maîtrise des technologies d'infrastructure moderne.

---

**Exercice pratique :** Créez une infrastructure multi-cloud complète avec :
1. Un cluster Azure AKS avec monitoring intégré
2. Des instances EC2 AWS avec auto-scaling
3. Un système de failover automatique entre clouds
4. Des politiques de sécurité unifiées
5. Un tableau de bord de monitoring multi-cloud

**Challenge avancé :** Développez une plateforme d'orchestration cloud qui :
- Implémente l'infrastructure as code avec Terraform/Pulumi
- Fournit des capacités de déploiement blue-green
- Intègre des métriques et du monitoring temps réel
- Supporte la migration automatique de workloads
- Implémente des politiques de gouvernance et conformité

**Réflexion :** Comment PowerShell change-t-il les paradigmes de l'administration cloud ? L'approche "cloud-first" rend-elle les compétences traditionnelles obsolètes, ou crée-t-elle de nouvelles opportunités ? Quelle est la place de PowerShell dans l'écosystème des outils DevOps modernes ?
