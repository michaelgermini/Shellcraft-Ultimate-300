# Chapitre 139 - PowerShell et Kubernetes

> "Kubernetes est l'orchestrateur des conteneurs, et PowerShell en est le chef d'orchestre humain. Ensemble, ils transforment les infrastructures en symphonies d'automatisation parfaitement synchronisées." - Citation inspirée des architectures de microservices

## Introduction : PowerShell comme interface Kubernetes

PowerShell transcende son rôle traditionnel pour devenir l'interface privilégiée de gestion des clusters Kubernetes. En maîtrisant kubectl, Helm, et les APIs Kubernetes à travers des modules spécialisés, les administrateurs PowerShell orchestrent des déploiements complexes, gèrent des microservices évolutifs, et maintiennent des infrastructures cloud-native avec une précision chirurgicale.

Dans ce chapitre, nous explorerons la gestion complète des clusters Kubernetes, les déploiements avancés, et les patterns d'orchestration.

## Section 1 : Gestion des clusters Kubernetes

### 1.1 Connexion et gestion de clusters

**Framework de gestion Kubernetes avancé :**
```powershell
# Framework avancé de gestion Kubernetes avec PowerShell
class KubernetesManager {
    [string]$ClusterName
    [string]$KubeConfigPath
    [System.Collections.Generic.Dictionary[string, PSObject]]$Namespaces
    [System.Collections.Generic.Dictionary[string, PSObject]]$Pods
    [System.Collections.Generic.Dictionary[string, PSObject]]$Services
    [System.Collections.Generic.List[PSCustomObject]]$OperationsLog
    [hashtable]$K8sConfig

    KubernetesManager([string]$clusterName, [string]$kubeConfigPath = "$env:USERPROFILE\.kube\config") {
        $this.ClusterName = $clusterName
        $this.KubeConfigPath = $kubeConfigPath
        $this.Namespaces = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.Pods = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.Services = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.OperationsLog = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.K8sConfig = @{
            DefaultNamespace = "default"
            RequestTimeout = 30
            RetryCount = 3
            Labels = @{ "managed-by" = "powershell"; "environment" = "development" }
        }

        $this.ConnectToCluster()
        $this.RefreshClusterInventory()
    }

    hidden [void]ConnectToCluster() {
        Write-Host "Connecting to Kubernetes cluster: $($this.ClusterName)" -ForegroundColor Cyan

        try {
            # Configuration de kubectl
            $env:KUBECONFIG = $this.KubeConfigPath

            # Test de connexion
            $version = kubectl version --client --short
            Write-Host "Connected to Kubernetes cluster" -ForegroundColor Green
            Write-Host "Client version: $($version -replace 'Client Version: ', '')" -ForegroundColor Gray

            $this.LogOperation("ClusterConnected", "Successfully connected to Kubernetes cluster", @{
                ClusterName = $this.ClusterName
                KubeConfig = $this.KubeConfigPath
            })

        } catch {
            Write-Error "Failed to connect to Kubernetes cluster: $_"
            throw
        }
    }

    hidden [void]RefreshClusterInventory() {
        Write-Host "Refreshing Kubernetes cluster inventory..." -ForegroundColor Gray

        try {
            # Récupération des namespaces
            $namespaces = kubectl get namespaces -o json | ConvertFrom-Json
            $this.Namespaces.Clear()
            foreach ($ns in $namespaces.items) {
                $this.Namespaces[$ns.metadata.name] = $ns
            }

            # Récupération des pods
            $pods = kubectl get pods --all-namespaces -o json | ConvertFrom-Json
            $this.Pods.Clear()
            foreach ($pod in $pods.items) {
                $key = "$($pod.metadata.namespace)/$($pod.metadata.name)"
                $this.Pods[$key] = $pod
            }

            # Récupération des services
            $services = kubectl get services --all-namespaces -o json | ConvertFrom-Json
            $this.Services.Clear()
            foreach ($svc in $services.items) {
                $key = "$($svc.metadata.namespace)/$($svc.metadata.name)"
                $this.Services[$key] = $svc
            }

            $this.LogOperation("InventoryRefreshed", "Kubernetes cluster inventory refreshed", @{
                Namespaces = $this.Namespaces.Count
                Pods = $this.Pods.Count
                Services = $this.Services.Count
            })

        } catch {
            Write-Warning "Failed to refresh cluster inventory: $_"
        }
    }

    [PSObject]CreateNamespace([string]$name, [hashtable]$labels = @{}) {
        Write-Host "Creating namespace: $name" -ForegroundColor Yellow

        $allLabels = $this.K8sConfig.Labels.Clone()
        $allLabels += $labels

        $yaml = @"
apiVersion: v1
kind: Namespace
metadata:
  name: $name
  labels:
$($allLabels.GetEnumerator() | ForEach-Object { "    $($_.Key): $($_.Value)" } | Out-String)
"@

        try {
            $yaml | kubectl apply -f -
            $this.RefreshClusterInventory()

            $this.LogOperation("NamespaceCreated", "Namespace created successfully", @{
                NamespaceName = $name
                Labels = $allLabels
            })

            return $this.Namespaces[$name]

        } catch {
            $this.LogOperation("NamespaceCreateFailed", "Failed to create namespace", @{
                NamespaceName = $name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject]DeployApplication([hashtable]$appConfig) {
        Write-Host "Deploying application: $($appConfig.Name)" -ForegroundColor Yellow

        $namespace = $appConfig.Namespace ?? $this.K8sConfig.DefaultNamespace

        # Création du namespace si nécessaire
        if (-not $this.Namespaces.ContainsKey($namespace)) {
            $this.CreateNamespace($namespace)
        }

        # Génération du déploiement YAML
        $deploymentYaml = $this.GenerateDeploymentYaml($appConfig)

        try {
            # Application du déploiement
            $deploymentYaml | kubectl apply -n $namespace -f -

            # Attente que le déploiement soit prêt
            Write-Host "Waiting for deployment to be ready..." -ForegroundColor Gray
            $timeout = $this.K8sConfig.RequestTimeout
            $ready = $false
            $elapsed = 0

            while (-not $ready -and $elapsed -lt $timeout) {
                $status = kubectl rollout status deployment/$($appConfig.Name) -n $namespace --timeout=10s
                if ($LASTEXITCODE -eq 0) {
                    $ready = $true
                } else {
                    Start-Sleep -Seconds 5
                    $elapsed += 5
                }
            }

            if (-not $ready) {
                throw "Deployment rollout timed out"
            }

            $this.RefreshClusterInventory()

            $this.LogOperation("ApplicationDeployed", "Application deployed successfully", @{
                ApplicationName = $appConfig.Name
                Namespace = $namespace
                Replicas = $appConfig.Replicas ?? 1
            })

            # Récupération du déploiement créé
            $deployment = kubectl get deployment $($appConfig.Name) -n $namespace -o json | ConvertFrom-Json
            return $deployment

        } catch {
            $this.LogOperation("ApplicationDeployFailed", "Failed to deploy application", @{
                ApplicationName = $appConfig.Name
                Namespace = $namespace
                Error = $_.Exception.Message
            })
            throw
        }
    }

    hidden [string]GenerateDeploymentYaml([hashtable]$appConfig) {
        $replicas = $appConfig.Replicas ?? 1
        $image = $appConfig.Image ?? "nginx:latest"
        $ports = $appConfig.Ports ?? @(@{ containerPort = 80 })
        $envVars = $appConfig.Environment ?? @()

        $portsYaml = $ports | ForEach-Object {
            @"

          - containerPort: $($_.containerPort)
            protocol: $($_.protocol ?? "TCP")
"@
        } | Out-String

        $envYaml = ""
        if ($envVars) {
            $envYaml = @"

          env:
$($envVars | ForEach-Object { "          - name: $($_.name)`n            value: $($_.value)" } | Out-String)
"@
        }

        $labels = $this.K8sConfig.Labels.Clone()
        if ($appConfig.Labels) {
            $labels += $appConfig.Labels
        }

        $yaml = @"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: $($appConfig.Name)
  labels:
$($labels.GetEnumerator() | ForEach-Object { "    $($_.Key): $($_.Value)" } | Out-String)
spec:
  replicas: $replicas
  selector:
    matchLabels:
      app: $($appConfig.Name)
  template:
    metadata:
      labels:
        app: $($appConfig.Name)
$($labels.GetEnumerator() | ForEach-Object { "        $($_.Key): $($_.Value)" } | Out-String)
    spec:
      containers:
      - name: $($appConfig.Name)
        image: $image
        ports:
$portsYaml$envYaml
        resources:
          requests:
            memory: "$($appConfig.MemoryRequest ?? "128Mi")"
            cpu: "$($appConfig.CPURequest ?? "100m")"
          limits:
            memory: "$($appConfig.MemoryLimit ?? "256Mi")"
            cpu: "$($appConfig.CPULimit ?? "200m")"
"@

        return $yaml
    }

    [PSObject]CreateService([hashtable]$serviceConfig) {
        Write-Host "Creating service: $($serviceConfig.Name)" -ForegroundColor Yellow

        $namespace = $serviceConfig.Namespace ?? $this.K8sConfig.DefaultNamespace
        $serviceType = $serviceConfig.Type ?? "ClusterIP"
        $ports = $serviceConfig.Ports ?? @(@{ port = 80; targetPort = 80 })

        $portsYaml = $ports | ForEach-Object {
            @"

  - port: $($_.port)
    targetPort: $($_.targetPort)
    protocol: $($_.protocol ?? "TCP")
"@
        } | Out-String

        $yaml = @"
apiVersion: v1
kind: Service
metadata:
  name: $($serviceConfig.Name)
  labels:
    app: $($serviceConfig.Name)
spec:
  type: $serviceType
  selector:
    app: $($serviceConfig.Name)
  ports:
$portsYaml
"@

        try {
            $yaml | kubectl apply -n $namespace -f -
            $this.RefreshClusterInventory()

            $this.LogOperation("ServiceCreated", "Service created successfully", @{
                ServiceName = $serviceConfig.Name
                Namespace = $namespace
                Type = $serviceType
            })

            $service = kubectl get service $($serviceConfig.Name) -n $namespace -o json | ConvertFrom-Json
            return $service

        } catch {
            $this.LogOperation("ServiceCreateFailed", "Failed to create service", @{
                ServiceName = $serviceConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]ScaleDeployment([string]$deploymentName, [string]$namespace, [int]$replicas) {
        Write-Host "Scaling deployment $deploymentName to $replicas replicas" -ForegroundColor Yellow

        try {
            kubectl scale deployment $deploymentName --replicas=$replicas -n $namespace

            $this.LogOperation("DeploymentScaled", "Deployment scaled successfully", @{
                DeploymentName = $deploymentName
                Namespace = $namespace
                NewReplicas = $replicas
            })

        } catch {
            $this.LogOperation("DeploymentScaleFailed", "Failed to scale deployment", @{
                DeploymentName = $deploymentName
                Namespace = $namespace
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]UpdateDeployment([string]$deploymentName, [string]$namespace, [string]$newImage) {
        Write-Host "Updating deployment $deploymentName with new image: $newImage" -ForegroundColor Yellow

        try {
            kubectl set image deployment/$deploymentName *=$newImage -n $namespace

            # Attente du rollout
            kubectl rollout status deployment/$deploymentName -n $namespace

            $this.LogOperation("DeploymentUpdated", "Deployment updated successfully", @{
                DeploymentName = $deploymentName
                Namespace = $namespace
                NewImage = $newImage
            })

        } catch {
            $this.LogOperation("DeploymentUpdateFailed", "Failed to update deployment", @{
                DeploymentName = $deploymentName
                Namespace = $namespace
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [hashtable]GetClusterMetrics() {
        try {
            $metrics = @{
                Nodes = @()
                TotalPods = 0
                RunningPods = 0
                TotalCPU = 0
                TotalMemory = 0
                Namespaces = $this.Namespaces.Count
            }

            # Métriques des nœuds
            $nodes = kubectl get nodes -o json | ConvertFrom-Json
            foreach ($node in $nodes.items) {
                $nodeMetrics = @{
                    Name = $node.metadata.name
                    Status = ($node.status.conditions | Where-Object { $_.type -eq "Ready" }).status
                    CPUCapacity = $node.status.capacity.cpu
                    MemoryCapacity = $node.status.capacity.memory
                }
                $metrics.Nodes += $nodeMetrics
            }

            # Métriques des pods
            $pods = kubectl get pods --all-namespaces -o json | ConvertFrom-Json
            $metrics.TotalPods = $pods.items.Count
            $metrics.RunningPods = ($pods.items | Where-Object { $_.status.phase -eq "Running" }).Count

            return $metrics

        } catch {
            return @{ Error = $_.Exception.Message }
        }
    }

    [PSObject[]]GetPods([string]$namespace = "*", [string]$filter = "*") {
        return $this.Pods.GetEnumerator() | Where-Object {
            ($namespace -eq "*" -or $_.Key.StartsWith("$namespace/")) -and
            $_.Value.metadata.name -like $filter
        } | ForEach-Object { $_.Value }
    }

    [PSObject[]]GetServices([string]$namespace = "*", [string]$filter = "*") {
        return $this.Services.GetEnumerator() | Where-Object {
            ($namespace -eq "*" -or $_.Key.StartsWith("$namespace/")) -and
            $_.Value.metadata.name -like $filter
        } | ForEach-Object { $_.Value }
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
        $metrics = $this.GetClusterMetrics()

        $report = @"
KUBERNETES CLUSTER REPORT
=========================
Cluster: $($this.ClusterName)
Generated: $(Get-Date)

CLUSTER METRICS
===============
Namespaces: $($metrics.Namespaces)
Total Pods: $($metrics.TotalPods)
Running Pods: $($metrics.RunningPods)
Pod Health: $([math]::Round(($metrics.RunningPods / $metrics.TotalPods) * 100, 1))%

NODE STATUS
===========
$($metrics.Nodes | ForEach-Object { "$($_.Name): $($_.Status) (CPU: $($_.CPUCapacity), Memory: $($_.MemoryCapacity))" } | Out-String)

TOP NAMESPACES BY POD COUNT
===========================
$($this.Pods.Values | Group-Object { $_.metadata.namespace } | Sort-Object Count -Descending | Select-Object -First 5 | ForEach-Object { "$($_.Name): $($_.Count) pods" } | Out-String)

RECENT OPERATIONS (Last 24h)
===========================
$($this.GetOperationsLog(24) | Select-Object -First 10 | ForEach-Object { "$($_.Timestamp): $($_.Operation) - $($_.Message)" } | Out-String)
"@

        return $report
    }
}

# Gestionnaire Helm pour Kubernetes
class HelmManager {
    [string]$HelmBinary
    [System.Collections.Generic.Dictionary[string, PSObject]]$Releases
    [System.Collections.Generic.List[PSCustomObject]]$HelmOperations
    [KubernetesManager]$K8sManager

    HelmManager([KubernetesManager]$k8sManager, [string]$helmBinary = "helm") {
        $this.K8sManager = $k8sManager
        $this.HelmBinary = $helmBinary
        $this.Releases = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.HelmOperations = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.RefreshReleases()
    }

    hidden [void]RefreshReleases() {
        Write-Host "Refreshing Helm releases..." -ForegroundColor Gray

        try {
            $releases = & $this.HelmBinary list --all-namespaces -o json | ConvertFrom-Json
            $this.Releases.Clear()
            foreach ($release in $releases) {
                $this.Releases[$release.name] = $release
            }
        } catch {
            Write-Warning "Failed to refresh Helm releases: $_"
        }
    }

    [PSObject]InstallChart([hashtable]$chartConfig) {
        Write-Host "Installing Helm chart: $($chartConfig.Name)" -ForegroundColor Yellow

        $installCmd = "$($this.HelmBinary) install $($chartConfig.Name)"

        if ($chartConfig.Chart) {
            $installCmd += " $($chartConfig.Chart)"
        } elseif ($chartConfig.Repo -and $chartConfig.ChartName) {
            $installCmd += " $($chartConfig.Repo)/$($chartConfig.ChartName)"
        }

        if ($chartConfig.Namespace) {
            $installCmd += " --namespace $($chartConfig.Namespace)"
            # Création du namespace si nécessaire
            if (-not $this.K8sManager.Namespaces.ContainsKey($chartConfig.Namespace)) {
                $this.K8sManager.CreateNamespace($chartConfig.Namespace)
            }
        }

        if ($chartConfig.Values) {
            $valuesFile = [System.IO.Path]::GetTempFileName() + ".yaml"
            $chartConfig.Values | ConvertTo-Yaml | Out-File -FilePath $valuesFile -Encoding UTF8
            $installCmd += " --values $valuesFile"
        }

        if ($chartConfig.Version) {
            $installCmd += " --version $($chartConfig.Version)"
        }

        try {
            $result = Invoke-Expression $installCmd

            if ($chartConfig.Values) {
                Remove-Item $valuesFile -ErrorAction SilentlyContinue
            }

            $this.RefreshReleases()

            $this.LogHelmOperation("ChartInstalled", "Helm chart installed successfully", @{
                ChartName = $chartConfig.Name
                Chart = $chartConfig.Chart ?? "$($chartConfig.Repo)/$($chartConfig.ChartName)"
                Namespace = $chartConfig.Namespace
                Version = $chartConfig.Version
            })

            return $this.Releases[$chartConfig.Name]

        } catch {
            if ($chartConfig.Values) {
                Remove-Item $valuesFile -ErrorAction SilentlyContinue
            }

            $this.LogHelmOperation("ChartInstallFailed", "Failed to install Helm chart", @{
                ChartName = $chartConfig.Name
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]UpgradeChart([string]$releaseName, [hashtable]$upgradeConfig) {
        Write-Host "Upgrading Helm release: $releaseName" -ForegroundColor Yellow

        $upgradeCmd = "$($this.HelmBinary) upgrade $releaseName"

        if ($upgradeConfig.Chart) {
            $upgradeCmd += " $($upgradeConfig.Chart)"
        }

        if ($upgradeConfig.Values) {
            $valuesFile = [System.IO.Path]::GetTempFileName() + ".yaml"
            $upgradeConfig.Values | ConvertTo-Yaml | Out-File -FilePath $valuesFile -Encoding UTF8
            $upgradeCmd += " --values $valuesFile"
        }

        if ($upgradeConfig.Version) {
            $upgradeCmd += " --version $($upgradeConfig.Version)"
        }

        try {
            $result = Invoke-Expression $upgradeCmd

            if ($upgradeConfig.Values) {
                Remove-Item $valuesFile -ErrorAction SilentlyContinue
            }

            $this.RefreshReleases()

            $this.LogHelmOperation("ChartUpgraded", "Helm chart upgraded successfully", @{
                ReleaseName = $releaseName
                Version = $upgradeConfig.Version
            })

        } catch {
            if ($upgradeConfig.Values) {
                Remove-Item $valuesFile -ErrorAction SilentlyContinue
            }

            $this.LogHelmOperation("ChartUpgradeFailed", "Failed to upgrade Helm chart", @{
                ReleaseName = $releaseName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [void]UninstallChart([string]$releaseName) {
        Write-Host "Uninstalling Helm release: $releaseName" -ForegroundColor Red

        try {
            $result = & $this.HelmBinary uninstall $releaseName

            $this.RefreshReleases()

            $this.LogHelmOperation("ChartUninstalled", "Helm chart uninstalled successfully", @{
                ReleaseName = $releaseName
            })

        } catch {
            $this.LogHelmOperation("ChartUninstallFailed", "Failed to uninstall Helm chart", @{
                ReleaseName = $releaseName
                Error = $_.Exception.Message
            })
            throw
        }
    }

    [PSObject[]]GetReleases([string]$filter = "*") {
        return $this.Releases.Values | Where-Object { $_.name -like $filter }
    }

    hidden [void]LogHelmOperation([string]$operation, [string]$message, [hashtable]$details = @{}) {
        $logEntry = [PSCustomObject]@{
            Timestamp = Get-Date
            Operation = $operation
            Message = $message
            Details = $details
        }

        $this.HelmOperations.Add($logEntry)
    }

    [PSCustomObject[]]GetOperationsLog([int]$lastHours = 24) {
        $cutoff = (Get-Date).AddHours(-$lastHours)
        return $this.HelmOperations | Where-Object { $_.Timestamp -ge $cutoff } | Sort-Object Timestamp -Descending
    }
}

# Fonction utilitaire pour convertir hashtable en YAML
function ConvertTo-Yaml {
    param([Parameter(ValueFromPipeline=$true)]$InputObject)

    $yaml = ""
    foreach ($key in $InputObject.Keys) {
        $value = $InputObject[$key]
        if ($value -is [hashtable]) {
            $yaml += "$key`:`n"
            foreach ($subKey in $value.Keys) {
                $yaml += "  $($subKey): $($value[$subKey])`n"
            }
        } else {
            $yaml += "$key`: $value`n"
        }
    }

    return $yaml
}

# Démonstration du framework Kubernetes
Write-Host "=== KUBERNETES MANAGEMENT DEMONSTRATION ===" -ForegroundColor Cyan

# Initialisation du gestionnaire Kubernetes
# $k8sManager = [KubernetesManager]::new("my-cluster")

# Déploiement d'une application simple
$appConfig = @{
    Name = "nginx-app"
    Image = "nginx:1.21"
    Replicas = 3
    Namespace = "web-apps"
    Ports = @(@{ containerPort = 80 })
    Environment = @(
        @{ name = "ENV"; value = "production" }
    )
    Labels = @{
        "tier" = "frontend"
        "version" = "1.0"
    }
    MemoryRequest = "64Mi"
    MemoryLimit = "128Mi"
    CPURequest = "50m"
    CPULimit = "100m"
}

# $deployment = $k8sManager.DeployApplication($appConfig)

# Création d'un service
$serviceConfig = @{
    Name = "nginx-service"
    Namespace = "web-apps"
    Type = "LoadBalancer"
    Ports = @(@{ port = 80; targetPort = 80 })
}

# $service = $k8sManager.CreateService($serviceConfig)

# Mise à l'échelle
# $k8sManager.ScaleDeployment("nginx-app", "web-apps", 5)

# Mise à jour
# $k8sManager.UpdateDeployment("nginx-app", "web-apps", "nginx:1.22")

# Initialisation du gestionnaire Helm
# $helmManager = [HelmManager]::new($k8sManager)

# Installation d'un chart Helm
$chartConfig = @{
    Name = "nginx-ingress"
    Repo = "ingress-nginx"
    ChartName = "ingress-nginx"
    Namespace = "ingress-nginx"
    Version = "4.0.0"
    Values = @{
        controller = @{
            replicaCount = 2
            service = @{
                type = "LoadBalancer"
            }
        }
    }
}

# $helmRelease = $helmManager.InstallChart($chartConfig)

# Rapport du cluster
# $report = $k8sManager.GenerateReport()
# $report | Out-File -FilePath "KubernetesReport_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nKubernetes management framework prepared!" -ForegroundColor Green
Write-Host "Uncomment the code above and configure kubectl/helm to run the full demonstration." -ForegroundColor Yellow
```

### 1.2 Gestion avancée des déploiements et rollbacks

**Système de déploiement avancé avec rollback automatique :**
```powershell
# Système de déploiement avancé avec stratégies de rollback
class KubernetesDeploymentManager : KubernetesManager {
    [System.Collections.Generic.Dictionary[string, System.Collections.Generic.List[PSObject]]]$DeploymentHistory
    [hashtable]$DeploymentStrategies
    [System.Collections.Generic.List[PSCustomObject]]$ActiveDeployments

    KubernetesDeploymentManager([string]$clusterName, [string]$kubeConfigPath = "$env:USERPROFILE\.kube\config") : base($clusterName, $kubeConfigPath) {
        $this.DeploymentHistory = [System.Collections.Generic.Dictionary[string, System.Collections.Generic.List[PSObject]]]::new()
        $this.ActiveDeployments = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.InitializeDeploymentStrategies()
    }

    hidden [void]InitializeDeploymentStrategies() {
        $this.DeploymentStrategies = @{
            # Déploiement bleu-vert
            BlueGreen = @{
                Description = "Blue-Green deployment with traffic switching"
                Steps = @("DeployGreen", "TestGreen", "SwitchTraffic", "TerminateBlue")
                RollbackSteps = @("SwitchTrafficBack", "TerminateGreen")
            }
            # Déploiement canary
            Canary = @{
                Description = "Canary deployment with gradual traffic increase"
                Steps = @("DeployCanary", "RouteTraffic", "MonitorMetrics", "FullRollout")
                RollbackSteps = @("TerminateCanary", "RestoreTraffic")
            }
            # Déploiement rolling update
            RollingUpdate = @{
                Description = "Rolling update with zero downtime"
                Steps = @("UpdatePods", "HealthCheck", "ContinueUpdate")
                RollbackSteps = @("RollbackToPrevious")
            }
        }
    }

    [PSCustomObject]DeployWithStrategy([string]$strategyName, [hashtable]$deploymentConfig) {
        if (-not $this.DeploymentStrategies.ContainsKey($strategyName)) {
            throw "Deployment strategy '$strategyName' not found"
        }

        $strategy = $this.DeploymentStrategies[$strategyName]
        $deploymentId = [guid]::NewGuid().ToString()

        $deployment = [PSCustomObject]@{
            Id = $deploymentId
            Strategy = $strategyName
            Config = $deploymentConfig
            Status = "Initializing"
            StartTime = Get-Date
            EndTime = $null
            CurrentStep = 0
            Steps = $strategy.Steps
            RollbackSteps = $strategy.RollbackSteps
            RollbackAvailable = $true
            Success = $false
        }

        $this.ActiveDeployments.Add($deployment)

        Write-Host "Starting $($strategy.Description) for $($deploymentConfig.Name)" -ForegroundColor Cyan

        try {
            $deployment.Status = "Running"

            foreach ($step in $strategy.Steps) {
                $deployment.CurrentStep++
                Write-Host "Executing step $($deployment.CurrentStep)/$($strategy.Steps.Count): $step" -ForegroundColor Yellow

                $result = $this.ExecuteDeploymentStep($step, $deploymentConfig)

                if (-not $result.Success) {
                    Write-Warning "Step $step failed: $($result.Message)"
                    $deployment.Status = "Failed"
                    $deployment.Success = $false

                    # Tentative de rollback automatique
                    if ($deployment.RollbackAvailable) {
                        Write-Host "Attempting automatic rollback..." -ForegroundColor Red
                        $rollbackResult = $this.ExecuteRollback($deployment)
                        $deployment.RollbackExecuted = $rollbackResult.Success
                    }

                    break
                }

                # Sauvegarde de l'état pour rollback
                $this.SaveDeploymentState($deploymentId, $step, $deploymentConfig)
            }

            if ($deployment.Status -eq "Running") {
                $deployment.Status = "Completed"
                $deployment.Success = $true
                $deployment.EndTime = Get-Date

                # Archivage de l'historique
                $this.ArchiveDeployment($deployment)
            }

        } catch {
            $deployment.Status = "Error"
            $deployment.Success = $false
            $deployment.EndTime = Get-Date
            $deployment.Error = $_.Exception.Message

            Write-Error "Deployment failed: $_"
        }

        $deployment.Duration = if ($deployment.EndTime) { $deployment.EndTime - $deployment.StartTime } else { (Get-Date) - $deployment.StartTime }

        $this.LogOperation("StrategicDeployment", "Strategic deployment $($deployment.Status.ToLower())", @{
            DeploymentId = $deploymentId
            Strategy = $strategyName
            Application = $deploymentConfig.Name
            Duration = $deployment.Duration.TotalSeconds
            Success = $deployment.Success
        })

        return $deployment
    }

    hidden [PSCustomObject]ExecuteDeploymentStep([string]$step, [hashtable]$config) {
        switch ($step) {
            "DeployGreen" {
                # Déploiement de la version verte
                $greenConfig = $config.Clone()
                $greenConfig.Name = "$($config.Name)-green"
                $greenConfig.Labels = @{ "version" = "green" }

                $this.DeployApplication($greenConfig)
                return @{ Success = $true; Message = "Green version deployed" }
            }

            "TestGreen" {
                # Tests de santé de la version verte
                Start-Sleep -Seconds 10  # Simulation de tests

                $greenPods = $this.GetPods($config.Namespace, "$($config.Name)-green-*")
                $healthyPods = $greenPods | Where-Object { $_.status.phase -eq "Running" }

                if ($healthyPods.Count -ge ($config.Replicas ?? 1)) {
                    return @{ Success = $true; Message = "Green version health checks passed" }
                } else {
                    return @{ Success = $false; Message = "Green version health checks failed" }
                }
            }

            "SwitchTraffic" {
                # Basculement du traffic vers la version verte
                $service = $this.GetServices($config.Namespace, $config.Name) | Select-Object -First 1
                if ($service) {
                    # Mise à jour des sélecteurs du service
                    kubectl patch service $service.metadata.name -n $config.Namespace -p "{""spec"":{""selector"":{""version"":""green""}}}"
                }

                return @{ Success = $true; Message = "Traffic switched to green version" }
            }

            "TerminateBlue" {
                # Terminaison de la version bleue
                $blueDeployment = "$($config.Name)-blue"
                kubectl delete deployment $blueDeployment -n $config.Namespace --ignore-not-found=true

                return @{ Success = $true; Message = "Blue version terminated" }
            }

            "DeployCanary" {
                # Déploiement d'une petite instance canary
                $canaryConfig = $config.Clone()
                $canaryConfig.Name = "$($config.Name)-canary"
                $canaryConfig.Replicas = 1
                $canaryConfig.Labels = @{ "version" = "canary" }

                $this.DeployApplication($canaryConfig)
                return @{ Success = $true; Message = "Canary version deployed" }
            }

            "RouteTraffic" {
                # Routage progressif du traffic vers le canary
                # Simulation avec un service mesh ou ingress
                Write-Host "Routing 10% of traffic to canary version..." -ForegroundColor Gray
                Start-Sleep -Seconds 5

                return @{ Success = $true; Message = "Traffic routing configured" }
            }

            "MonitorMetrics" {
                # Monitoring des métriques du canary
                Write-Host "Monitoring canary metrics..." -ForegroundColor Gray
                Start-Sleep -Seconds 10

                # Simulation d'analyse de métriques
                $errorRate = Get-Random -Minimum 0 -Maximum 5
                if ($errorRate -lt 2) {
                    return @{ Success = $true; Message = "Canary metrics acceptable" }
                } else {
                    return @{ Success = $false; Message = "Canary metrics unacceptable (error rate: $errorRate%)" }
                }
            }

            "FullRollout" {
                # Rollout complet vers la nouvelle version
                $this.ScaleDeployment($config.Name, $config.Namespace, 0)  # Scale down old version
                $canaryDeployment = "$($config.Name)-canary"
                $this.ScaleDeployment($canaryDeployment, $config.Namespace, $config.Replicas ?? 1)

                # Renommage du déploiement canary
                kubectl get deployment $canaryDeployment -n $config.Namespace -o yaml | 
                    kubectl apply -f - --dry-run=client -o yaml |
                    kubectl patch deployment $canaryDeployment -n $config.Namespace --type strategic -p "{""metadata"":{""name"":""$($config.Name)""}}"

                return @{ Success = $true; Message = "Full rollout completed" }
            }

            "UpdatePods" {
                # Mise à jour rolling des pods
                $this.UpdateDeployment($config.Name, $config.Namespace, $config.NewImage)
                return @{ Success = $true; Message = "Pods updated with rolling strategy" }
            }

            "HealthCheck" {
                # Vérification de santé post-update
                $deployment = kubectl get deployment $($config.Name) -n $config.Namespace -o json | ConvertFrom-Json
                $readyReplicas = $deployment.status.readyReplicas ?? 0
                $desiredReplicas = $deployment.spec.replicas ?? 1

                if ($readyReplicas -ge $desiredReplicas) {
                    return @{ Success = $true; Message = "Health checks passed" }
                } else {
                    return @{ Success = $false; Message = "Health checks failed: $readyReplicas/$desiredReplicas ready" }
                }
            }

            default {
                return @{ Success = $false; Message = "Unknown deployment step: $step" }
            }
        }
    }

    hidden [PSCustomObject]ExecuteRollback([PSCustomObject]$deployment) {
        Write-Host "Executing rollback for deployment $($deployment.Id)" -ForegroundColor Red

        $rollbackSuccess = $true
        $rollbackSteps = $deployment.RollbackSteps

        for ($i = 0; $i -lt $rollbackSteps.Count; $i++) {
            $step = $rollbackSteps[$i]
            Write-Host "Rollback step $($i + 1): $step" -ForegroundColor Yellow

            try {
                $result = $this.ExecuteRollbackStep($step, $deployment.Config)
                if (-not $result.Success) {
                    Write-Warning "Rollback step $step failed: $($result.Message)"
                    $rollbackSuccess = $false
                }
            } catch {
                Write-Error "Rollback step $step error: $_"
                $rollbackSuccess = $false
            }
        }

        return @{
            Success = $rollbackSuccess
            StepsExecuted = $rollbackSteps.Count
        }
    }

    hidden [PSCustomObject]ExecuteRollbackStep([string]$step, [hashtable]$config) {
        switch ($step) {
            "SwitchTrafficBack" {
                # Basculement du traffic vers la version bleue
                $service = $this.GetServices($config.Namespace, $config.Name) | Select-Object -First 1
                if ($service) {
                    kubectl patch service $service.metadata.name -n $config.Namespace -p "{""spec"":{""selector"":{""version"":""blue""}}}"
                }
                return @{ Success = $true; Message = "Traffic switched back to blue version" }
            }

            "TerminateGreen" {
                # Terminaison de la version verte
                $greenDeployment = "$($config.Name)-green"
                kubectl delete deployment $greenDeployment -n $config.Namespace --ignore-not-found=true
                return @{ Success = $true; Message = "Green version terminated" }
            }

            "TerminateCanary" {
                # Terminaison de la version canary
                $canaryDeployment = "$($config.Name)-canary"
                kubectl delete deployment $canaryDeployment -n $config.Namespace --ignore-not-found=true
                return @{ Success = $true; Message = "Canary version terminated" }
            }

            "RestoreTraffic" {
                # Restauration du traffic vers la version stable
                Write-Host "Restoring traffic to stable version..." -ForegroundColor Gray
                return @{ Success = $true; Message = "Traffic restored to stable version" }
            }

            "RollbackToPrevious" {
                # Rollback vers la version précédente
                kubectl rollout undo deployment/$($config.Name) -n $config.Namespace
                return @{ Success = $true; Message = "Rolled back to previous version" }
            }

            default {
                return @{ Success = $false; Message = "Unknown rollback step: $step" }
            }
        }
    }

    hidden [void]SaveDeploymentState([string]$deploymentId, [string]$step, [hashtable]$config) {
        if (-not $this.DeploymentHistory.ContainsKey($deploymentId)) {
            $this.DeploymentHistory[$deploymentId] = [System.Collections.Generic.List[PSObject]]::new()
        }

        $state = @{
            Step = $step
            Timestamp = Get-Date
            Config = $config.Clone()
            ClusterState = @{
                Pods = $this.GetPods($config.Namespace).Count
                Services = $this.GetServices($config.Namespace).Count
            }
        }

        $this.DeploymentHistory[$deploymentId].Add([PSCustomObject]$state)
    }

    hidden [void]ArchiveDeployment([PSCustomObject]$deployment) {
        # Archivage de l'historique de déploiement
        $archivePath = "$PSScriptRoot\DeploymentHistory\$($deployment.Id).json"
        $deployment | ConvertTo-Json -Depth 10 | Out-File -FilePath $archivePath -Encoding UTF8

        Write-Host "Deployment archived: $archivePath" -ForegroundColor Gray
    }

    [PSCustomObject]RollbackDeployment([string]$deploymentId) {
        $deployment = $this.ActiveDeployments | Where-Object { $_.Id -eq $deploymentId } | Select-Object -First 1

        if (-not $deployment) {
            throw "Deployment '$deploymentId' not found"
        }

        if (-not $deployment.RollbackAvailable) {
            throw "Rollback not available for deployment '$deploymentId'"
        }

        return $this.ExecuteRollback($deployment)
    }

    [hashtable]GetDeploymentMetrics() {
        $completedDeployments = $this.ActiveDeployments | Where-Object { $_.Status -in @("Completed", "Failed") }

        $metrics = @{
            TotalDeployments = $this.ActiveDeployments.Count
            SuccessfulDeployments = ($this.ActiveDeployments | Where-Object { $_.Success }).Count
            FailedDeployments = ($this.ActiveDeployments | Where-Object { -not $_.Success -and $_.Status -eq "Failed" }).Count
            ActiveDeployments = ($this.ActiveDeployments | Where-Object { $_.Status -eq "Running" }).Count
            AverageDeploymentTime = $null
            DeploymentSuccessRate = 0
            MostUsedStrategy = $null
        }

        if ($completedDeployments) {
            $totalTime = ($completedDeployments | Measure-Object -Property Duration.TotalSeconds -Sum).Sum
            $metrics.AverageDeploymentTime = [TimeSpan]::FromSeconds($totalTime / $completedDeployments.Count)

            $successfulCount = ($completedDeployments | Where-Object { $_.Success }).Count
            $metrics.DeploymentSuccessRate = [math]::Round(($successfulCount / $completedDeployments.Count) * 100, 1)

            $metrics.MostUsedStrategy = $this.ActiveDeployments | Group-Object Strategy | Sort-Object Count -Descending | Select-Object -First 1 | ForEach-Object { $_.Name }
        }

        return $metrics
    }

    [PSCustomObject[]]GetActiveDeployments() {
        return $this.ActiveDeployments.ToArray()
    }

    [PSCustomObject[]]GetDeploymentHistory([string]$deploymentId = $null) {
        if ($deploymentId) {
            return $this.DeploymentHistory[$deploymentId] ?? @()
        }

        # Retourne un résumé de tous les déploiements
        return $this.ActiveDeployments | ForEach-Object {
            [PSCustomObject]@{
                Id = $_.Id
                Strategy = $_.Strategy
                Application = $_.Config.Name
                Status = $_.Status
                Success = $_.Success
                Duration = $_.Duration
                StepsCompleted = $_.CurrentStep
                TotalSteps = $_.Steps.Count
            }
        }
    }
}

# Orchestrateur de microservices Kubernetes
class KubernetesMicroserviceOrchestrator : KubernetesDeploymentManager {
    [System.Collections.Generic.Dictionary[string, hashtable]]$Microservices
    [hashtable]$ServiceMeshConfig
    [System.Collections.Generic.List[PSCustomObject]]$ServiceDependencies

    KubernetesMicroserviceOrchestrator([string]$clusterName) : base($clusterName) {
        $this.Microservices = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.ServiceDependencies = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.InitializeServiceMesh()
    }

    hidden [void]InitializeServiceMesh() {
        $this.ServiceMeshConfig = @{
            Enabled = $true
            MeshType = "Istio"  # ou "Linkerd", "Consul", etc.
            AutoInjection = $true
            MutualTLS = $true
            TrafficPolicies = @{
                RetryPolicy = @{
                    Attempts = 3
                    PerTryTimeout = "2s"
                    RetryOn = "gateway-error,connect-failure,refused-stream"
                }
                CircuitBreaker = @{
                    MaxConnections = 100
                    MaxPendingRequests = 10
                    MaxRequests = 100
                    MaxRetries = 3
                }
                LoadBalancer = @{
                    Simple = "ROUND_ROBIN"
                }
            }
        }
    }

    [void]RegisterMicroservice([hashtable]$serviceConfig) {
        $this.Microservices[$serviceConfig.Name] = $serviceConfig.Clone()

        # Enregistrement des dépendances
        if ($serviceConfig.Dependencies) {
            foreach ($dependency in $serviceConfig.Dependencies) {
                $this.ServiceDependencies.Add([PSCustomObject]@{
                    Service = $serviceConfig.Name
                    DependsOn = $dependency.Service
                    Type = $dependency.Type ?? "HTTP"
                    Required = $dependency.Required ?? $true
                })
            }
        }

        Write-Host "Microservice '$($serviceConfig.Name)' registered" -ForegroundColor Green
    }

    [PSCustomObject]DeployMicroserviceArchitecture([string[]]$serviceNames) {
        $deploymentPlan = [PSCustomObject]@{
            Id = [guid]::NewGuid().ToString()
            Services = $serviceNames
            DeploymentOrder = $this.CalculateDeploymentOrder($serviceNames)
            Status = "Planning"
            StartTime = Get-Date
            ServiceDeployments = @()
        }

        Write-Host "Planning microservice architecture deployment..." -ForegroundColor Cyan

        $deploymentPlan.Status = "Deploying"

        foreach ($serviceName in $deploymentPlan.DeploymentOrder) {
            Write-Host "Deploying microservice: $serviceName" -ForegroundColor Yellow

            $serviceConfig = $this.Microservices[$serviceName]
            $deployment = $this.DeployApplication($serviceConfig)

            $deploymentPlan.ServiceDeployments += @{
                Service = $serviceName
                Deployment = $deployment
                Status = "Deployed"
                Timestamp = Get-Date
            }

            # Configuration du service mesh pour ce service
            if ($this.ServiceMeshConfig.Enabled) {
                $this.ConfigureServiceMesh($serviceName, $serviceConfig)
            }

            # Attente de stabilité
            Start-Sleep -Seconds 5
        }

        # Création des politiques de traffic
        $this.CreateTrafficPolicies($serviceNames)

        $deploymentPlan.Status = "Completed"
        $deploymentPlan.EndTime = Get-Date
        $deploymentPlan.Duration = $deploymentPlan.EndTime - $deploymentPlan.StartTime

        Write-Host "Microservice architecture deployment completed in $([math]::Round($deploymentPlan.Duration.TotalSeconds, 1)) seconds" -ForegroundColor Green

        return $deploymentPlan
    }

    hidden [string[]]CalculateDeploymentOrder([string[]]$serviceNames) {
        # Algorithme de tri topologique pour l'ordre de déploiement
        $visited = @{}
        $order = [System.Collections.Generic.List[string]]::new()

        function Visit-Service {
            param([string]$serviceName)

            if ($visited.ContainsKey($serviceName)) {
                return
            }

            $visited[$serviceName] = $true

            # Visiter les dépendances d'abord
            $dependencies = $this.ServiceDependencies | Where-Object { $_.Service -eq $serviceName -and $_.Required }
            foreach ($dep in $dependencies) {
                Visit-Service $dep.DependsOn
            }

            $order.Add($serviceName)
        }

        foreach ($serviceName in $serviceNames) {
            Visit-Service $serviceName
        }

        return $order.ToArray()
    }

    hidden [void]ConfigureServiceMesh([string]$serviceName, [hashtable]$serviceConfig) {
        Write-Host "Configuring service mesh for: $serviceName" -ForegroundColor Gray

        # Configuration Istio (exemple)
        $destinationRule = @"
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: $serviceName-dr
spec:
  host: $serviceName
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    loadBalancer:
      simple: ROUND_ROBIN
    connectionPool:
      http:
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutiveErrors: 3
      interval: 10s
      baseEjectionTime: 30s
"@

        # Application de la configuration
        $destinationRule | kubectl apply -n $($serviceConfig.Namespace ?? "default") -f -

        # Configuration des règles de retry
        $retryRule = @"
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: $serviceName-retry
spec:
  hosts:
  - $serviceName
  http:
  - route:
    - destination:
        host: $serviceName
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: gateway-error,connect-failure,refused-stream
"@

        $retryRule | kubectl apply -n $($serviceConfig.Namespace ?? "default") -f -
    }

    hidden [void]CreateTrafficPolicies([string[]]$serviceNames) {
        Write-Host "Creating traffic policies..." -ForegroundColor Gray

        foreach ($serviceName in $serviceNames) {
            $serviceConfig = $this.Microservices[$serviceName]

            # Création d'un VirtualService pour le routage avancé
            $virtualService = @"
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: $serviceName-routing
spec:
  hosts:
  - $serviceName
  http:
  - match:
    - headers:
        x-version:
          exact: v1
    route:
    - destination:
        host: $serviceName
        subset: v1
  - route:
    - destination:
        host: $serviceName
        subset: v2
"@

            $virtualService | kubectl apply -n $($serviceConfig.Namespace ?? "default") -f -
        }
    }

    [void]ScaleMicroservice([string]$serviceName, [int]$replicas) {
        if (-not $this.Microservices.ContainsKey($serviceName)) {
            throw "Microservice '$serviceName' not registered"
        }

        $serviceConfig = $this.Microservices[$serviceName]

        # Vérification des contraintes de scaling
        $maxReplicas = $serviceConfig.MaxReplicas ?? 10
        $minReplicas = $serviceConfig.MinReplicas ?? 1

        if ($replicas -lt $minReplicas -or $replicas -gt $maxReplicas) {
            throw "Replica count must be between $minReplicas and $maxReplicas"
        }

        # Scaling avec HPA (Horizontal Pod Autoscaler) si configuré
        if ($serviceConfig.AutoScaling) {
            $this.UpdateHPAScale($serviceName, $replicas)
        } else {
            $this.ScaleDeployment($serviceName, $serviceConfig.Namespace ?? "default", $replicas)
        }

        Write-Host "Microservice '$serviceName' scaled to $replicas replicas" -ForegroundColor Green
    }

    hidden [void]UpdateHPAScale([string]$serviceName, [int]$replicas) {
        # Mise à jour de l'HPA pour définir le nombre de replicas minimum
        kubectl patch hpa $serviceName -n default --type='json' -p="[{'op': 'replace', 'path': '/spec/minReplicas', 'value': $replicas}]"
    }

    [hashtable]GetMicroserviceHealth() {
        $health = @{
            Services = @{}
            Dependencies = @{}
            OverallHealth = "Healthy"
        }

        foreach ($serviceName in $this.Microservices.Keys) {
            $serviceHealth = $this.GetServiceHealth($serviceName)

            $health.Services[$serviceName] = $serviceHealth

            if ($serviceHealth.HealthPercentage -lt 80) {
                $health.OverallHealth = "Degraded"
            }
        }

        # Vérification des dépendances
        foreach ($dependency in $this.ServiceDependencies) {
            $serviceHealth = $this.GetServiceHealth($dependency.Service)
            $dependencyHealth = @{
                Service = $dependency.Service
                DependsOn = $dependency.DependsOn
                Status = if ($serviceHealth.HealthPercentage -gt 0) { "Available" } else { "Unavailable" }
                Type = $dependency.Type
            }

            $health.Dependencies["$($dependency.Service)->$($dependency.DependsOn)"] = $dependencyHealth
        }

        return $health
    }

    [PSCustomObject]UpdateMicroservice([string]$serviceName, [string]$newImage, [string]$strategy = "RollingUpdate") {
        if (-not $this.Microservices.ContainsKey($serviceName)) {
            throw "Microservice '$serviceName' not registered"
        }

        Write-Host "Updating microservice '$serviceName' to image '$newImage' using $strategy strategy" -ForegroundColor Yellow

        $updateConfig = @{
            Name = $serviceName
            NewImage = $newImage
            Namespace = $this.Microservices[$serviceName].Namespace ?? "default"
            Strategy = $strategy
        }

        return $this.DeployWithStrategy($strategy, $updateConfig)
    }
}

# Démonstration des fonctionnalités avancées
Write-Host "=== ADVANCED KUBERNETES MANAGEMENT DEMONSTRATION ===" -ForegroundColor Cyan

# Enregistrement de microservices
$orchestrator = [KubernetesMicroserviceOrchestrator]::new("advanced-cluster")

$orchestrator.RegisterMicroservice(@{
    Name = "api-gateway"
    Image = "nginx:1.21"
    Replicas = 2
    Namespace = "microservices"
    Ports = @(@{ containerPort = 80 })
    Dependencies = @(
        @{ Service = "auth-service"; Type = "HTTP"; Required = $true }
    )
    MinReplicas = 1
    MaxReplicas = 5
    AutoScaling = $true
})

$orchestrator.RegisterMicroservice(@{
    Name = "auth-service"
    Image = "keycloak/keycloak:19.0"
    Replicas = 1
    Namespace = "microservices"
    Ports = @(@{ containerPort = 8080 })
    Dependencies = @(
        @{ Service = "database"; Type = "JDBC"; Required = $true }
    )
})

$orchestrator.RegisterMicroservice(@{
    Name = "user-service"
    Image = "openjdk:17-jdk-alpine"
    Replicas = 3
    Namespace = "microservices"
    Ports = @(@{ containerPort = 8080 })
    Dependencies = @(
        @{ Service = "database"; Type = "JDBC"; Required = $true }
        @{ Service = "auth-service"; Type = "HTTP"; Required = $false }
    )
})

$orchestrator.RegisterMicroservice(@{
    Name = "database"
    Image = "postgres:14"
    Replicas = 1
    Namespace = "microservices"
    Ports = @(@{ containerPort = 5432 })
})

# Déploiement de l'architecture microservices
$deploymentPlan = $orchestrator.DeployMicroserviceArchitecture(@("database", "auth-service", "user-service", "api-gateway"))

Write-Host "`nDeployment completed:" -ForegroundColor Green
Write-Host "Services deployed: $($deploymentPlan.ServiceDeployments.Count)"
Write-Host "Deployment order: $($deploymentPlan.DeploymentOrder -join ' -> ')"
Write-Host "Total duration: $([math]::Round($deploymentPlan.Duration.TotalSeconds, 1)) seconds"

# Mise à l'échelle
$orchestrator.ScaleMicroservice("user-service", 5)

# Mise à jour avec stratégie avancée
$updateResult = $orchestrator.UpdateMicroservice("api-gateway", "nginx:1.22", "BlueGreen")

# Monitoring de santé
$health = $orchestrator.GetMicroserviceHealth()
Write-Host "`n=== MICROSERVICE HEALTH ===" -ForegroundColor Cyan
Write-Host "Overall health: $($health.OverallHealth)"
foreach ($service in $health.Services.Keys) {
    $serviceHealth = $health.Services[$service]
    Write-Host "$service`: $($serviceHealth.HealthPercentage)% healthy ($($serviceHealth.RunningContainers)/$($serviceHealth.TotalContainers))"
}

# Métriques de déploiement
$deploymentMetrics = $orchestrator.GetDeploymentMetrics()
Write-Host "`n=== DEPLOYMENT METRICS ===" -ForegroundColor Cyan
Write-Host "Total deployments: $($deploymentMetrics.TotalDeployments)"
Write-Host "Success rate: $($deploymentMetrics.DeploymentSuccessRate)%"
if ($deploymentMetrics.AverageDeploymentTime) {
    Write-Host "Average deployment time: $([math]::Round($deploymentMetrics.AverageDeploymentTime.TotalSeconds, 1)) seconds"
}

Write-Host "`nAdvanced Kubernetes management demonstration completed!" -ForegroundColor Green
```

## Conclusion : PowerShell comme orchestrateur Kubernetes

PowerShell transcende son rôle traditionnel pour devenir l'orchestrateur ultime des clusters Kubernetes. En maîtrisant kubectl, Helm, et les stratégies de déploiement avancées, les administrateurs PowerShell orchestrent des architectures microservices complexes avec une précision et une fiabilité inégalées.

Dans le prochain chapitre, nous explorerons les intégrations DevOps avancées avec PowerShell, complétant notre maîtrise de l'écosystème moderne du développement logiciel.

---

**Exercice pratique :** Créez une plateforme complète de gestion Kubernetes avec :
1. Un orchestrateur de microservices avec service mesh intégré
2. Des stratégies de déploiement avancées (blue-green, canary)
3. Un système de rollback automatique et intelligent
4. Des politiques de scaling et d'auto-healing
5. Un tableau de bord de monitoring et d'observabilité

**Challenge avancé :** Développez une plateforme d'orchestration Kubernetes qui :
- Implémente GitOps avec flux automatique de déploiement
- Fournit des capacités d'auto-scaling prédictif basé sur l'IA
- Intègre des tests de chaos engineering automatisés
- Supporte le multi-cluster et le failover automatique
- Implémente des politiques de sécurité zero-trust

**Réflexion :** Comment PowerShell transforme-t-il l'administration Kubernetes ? Les abstractions de haut niveau qu'il fournit simplifient-elles trop la complexité inhérente à Kubernetes, ou créent-elles les bons niveaux d'abstraction ? Quelle est la place de PowerShell dans l'écosystème des outils Kubernetes natifs ?
