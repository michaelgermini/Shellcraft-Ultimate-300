# Chapitre 140 - PowerShell pour DevOps et CI/CD

> "PowerShell n'est pas seulement un langage de script, c'est le liant invisible qui transforme les pratiques DevOps en une symphonie d'automatisation où chaque pipeline devient une œuvre d'art technique." - Citation inspirée des philosophies DevOps modernes

## Introduction : PowerShell comme catalyseur DevOps

PowerShell transcende son rôle d'outil d'administration pour devenir le catalyseur des pratiques DevOps modernes. En intégrant les pipelines CI/CD, l'infrastructure as code, les tests automatisés, et les déploiements continus, PowerShell crée des workflows d'une élégance et d'une fiabilité sans précédent.

Dans ce chapitre, nous explorerons l'intégration complète de PowerShell dans les écosystèmes DevOps, des pipelines aux déploiements en passant par les tests et le monitoring.

## Section 1 : Pipelines CI/CD avec PowerShell

### 1.1 Framework de pipeline PowerShell

**Système complet de pipeline CI/CD :**
```powershell
# Framework avancé de pipeline CI/CD PowerShell
class PowerShellPipelineEngine {
    [System.Collections.Generic.Dictionary[string, hashtable]]$Pipelines
    [System.Collections.Generic.Dictionary[string, PSObject]]$PipelineRuns
    [System.Collections.Generic.List[PSCustomObject]]$BuildQueue
    [hashtable]$PipelineTemplates
    [scriptblock]$NotificationCallback

    PowerShellPipelineEngine() {
        $this.Pipelines = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.PipelineRuns = [System.Collections.Generic.Dictionary[string, PSObject]]::new()
        $this.BuildQueue = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.InitializePipelineTemplates()
        $this.NotificationCallback = {
            param($notification)
            Write-Host "PIPELINE NOTIFICATION: $($notification.Message)" -ForegroundColor (switch ($notification.Level) {
                "SUCCESS" { "Green" }
                "WARNING" { "Yellow" }
                "ERROR" { "Red" }
                default { "White" }
            })
        }
    }

    hidden [void]InitializePipelineTemplates() {
        $this.PipelineTemplates = @{
            # Pipeline .NET
            "DotNetWebApp" = @{
                Stages = @(
                    @{
                        Name = "Checkout"
                        Steps = @(
                            @{ Name = "CloneRepository"; Type = "GitClone"; Parameters = @{ Repository = "{RepositoryUrl}"; Branch = "{Branch}" } }
                        )
                    },
                    @{
                        Name = "Build"
                        Steps = @(
                            @{ Name = "RestorePackages"; Type = "DotNetRestore"; Parameters = @{} }
                            @{ Name = "BuildSolution"; Type = "DotNetBuild"; Parameters = @{ Configuration = "Release" } }
                            @{ Name = "RunTests"; Type = "DotNetTest"; Parameters = @{ CollectCoverage = $true } }
                        )
                    },
                    @{
                        Name = "Package"
                        Steps = @(
                            @{ Name = "CreatePackage"; Type = "DotNetPublish"; Parameters = @{ OutputPath = ".\publish" } }
                            @{ Name = "CreateArtifact"; Type = "ArchiveFiles"; Parameters = @{ SourcePath = ".\publish"; ArchivePath = "artifacts\app.zip" } }
                        )
                    },
                    @{
                        Name = "Deploy"
                        Steps = @(
                            @{ Name = "DeployToStaging"; Type = "AzureWebAppDeploy"; Parameters = @{ Slot = "staging" } }
                            @{ Name = "RunIntegrationTests"; Type = "PSTest"; Parameters = @{ TestPath = "tests\integration" } }
                            @{ Name = "SwapSlots"; Type = "AzureSlotSwap"; Parameters = @{} }
                        )
                    }
                )
            }

            # Pipeline PowerShell Module
            "PowerShellModule" = @{
                Stages = @(
                    @{
                        Name = "Validate"
                        Steps = @(
                            @{ Name = "PSScriptAnalyzer"; Type = "PSScriptAnalyzer"; Parameters = @{ Rules = "PSAvoidUsingWriteHost,PSUseApprovedVerbs" } }
                            @{ Name = "ImportModule"; Type = "PSImportModule"; Parameters = @{} }
                        )
                    },
                    @{
                        Name = "Test"
                        Steps = @(
                            @{ Name = "RunPesterTests"; Type = "PesterTest"; Parameters = @{ OutputFormat = "NUnitXml"; OutputPath = "test-results.xml" } }
                            @{ Name = "PublishTestResults"; Type = "PublishTestResults"; Parameters = @{ TestResultsPath = "test-results.xml" } }
                        )
                    },
                    @{
                        Name = "Build"
                        Steps = @(
                            @{ Name = "BuildModule"; Type = "PSBuildModule"; Parameters = @{} }
                            @{ Name = "CreateNuGetPackage"; Type = "NuGetPack"; Parameters = @{ Version = "{BuildNumber}" } }
                        )
                    },
                    @{
                        Name = "Publish"
                        Steps = @(
                            @{ Name = "PublishToGallery"; Type = "PowerShellGalleryPublish"; Parameters = @{ ApiKey = "{GalleryApiKey}" } }
                        )
                    }
                )
            }

            # Pipeline Infrastructure as Code
            "InfrastructureAsCode" = @{
                Stages = @(
                    @{
                        Name = "Lint"
                        Steps = @(
                            @{ Name = "TerraformValidate"; Type = "TerraformValidate"; Parameters = @{} }
                            @{ Name = "TerraformFormat"; Type = "TerraformFormat"; Parameters = @{ Check = $true } }
                        )
                    },
                    @{
                        Name = "Plan"
                        Steps = @(
                            @{ Name = "TerraformInit"; Type = "TerraformInit"; Parameters = @{} }
                            @{ Name = "TerraformPlan"; Type = "TerraformPlan"; Parameters = @{ Out = "tfplan" } }
                        )
                    },
                    @{
                        Name = "Apply"
                        Steps = @(
                            @{ Name = "TerraformApply"; Type = "TerraformApply"; Parameters = @{ Plan = "tfplan"; AutoApprove = $false } }
                        )
                    }
                )
            }
        }
    }

    [void]CreatePipeline([string]$name, [hashtable]$pipelineConfig) {
        # Validation de la configuration
        $this.ValidatePipelineConfig($pipelineConfig)

        $pipeline = @{
            Name = $name
            Config = $pipelineConfig.Clone()
            Created = Get-Date
            LastRun = $null
            Status = "Created"
            Triggers = $pipelineConfig.Triggers ?? @()
            Variables = $pipelineConfig.Variables ?? @{}
            Artifacts = @()
        }

        $this.Pipelines[$name] = $pipeline

        Write-Host "Pipeline '$name' created successfully" -ForegroundColor Green
    }

    [void]CreatePipelineFromTemplate([string]$pipelineName, [string]$templateName, [hashtable]$customizations = @{}) {
        if (-not $this.PipelineTemplates.ContainsKey($templateName)) {
            throw "Pipeline template '$templateName' not found"
        }

        $template = $this.PipelineTemplates[$templateName]
        $pipelineConfig = @{
            Template = $templateName
            Stages = $this.CustomizeTemplate($template.Stages, $customizations)
            Triggers = $customizations.Triggers ?? @()
            Variables = $customizations.Variables ?? @{}
        }

        $this.CreatePipeline($pipelineName, $pipelineConfig)
    }

    hidden [array]CustomizeTemplate([array]$stages, [hashtable]$customizations) {
        $customizedStages = @()

        foreach ($stage in $stages) {
            $customizedStage = @{
                Name = $stage.Name
                Steps = @()
            }

            foreach ($step in $stage.Steps) {
                $customizedStep = $step.Clone()

                # Application des customizations
                if ($customizations.ContainsKey($step.Name)) {
                    $stepCustomizations = $customizations[$step.Name]
                    foreach ($key in $stepCustomizations.Keys) {
                        $customizedStep.Parameters[$key] = $stepCustomizations[$key]
                    }
                }

                $customizedStage.Steps += $customizedStep
            }

            $customizedStages += $customizedStage
        }

        return $customizedStages
    }

    hidden [void]ValidatePipelineConfig([hashtable]$config) {
        if (-not $config.ContainsKey('Stages')) {
            throw "Pipeline configuration must contain 'Stages'"
        }

        foreach ($stage in $config.Stages) {
            if (-not $stage.Name -or -not $stage.Steps) {
                throw "Each stage must have a Name and Steps"
            }

            foreach ($step in $stage.Steps) {
                if (-not $step.Name -or -not $step.Type) {
                    throw "Each step must have a Name and Type"
                }
            }
        }
    }

    [PSCustomObject]StartPipeline([string]$pipelineName, [hashtable]$parameters = @{}) {
        if (-not $this.Pipelines.ContainsKey($pipelineName)) {
            throw "Pipeline '$pipelineName' not found"
        }

        $pipeline = $this.Pipelines[$pipelineName]
        $runId = [guid]::NewGuid().ToString()

        $pipelineRun = [PSCustomObject]@{
            Id = $runId
            PipelineName = $pipelineName
            StartTime = Get-Date
            EndTime = $null
            Status = "Running"
            Parameters = $parameters
            Stages = @()
            CurrentStage = 0
            Artifacts = @()
            Variables = $pipeline.Variables.Clone()
            # Fusion des variables de paramètres
            $pipelineRun.Variables += $parameters
        }

        $this.PipelineRuns[$runId] = $pipelineRun

        # Ajout à la queue de build
        $this.BuildQueue.Add([PSCustomObject]@{
            RunId = $runId
            PipelineName = $pipelineName
            Priority = $parameters.Priority ?? 1
            QueuedAt = Get-Date
        })

        # Démarrage asynchrone du pipeline
        $job = Start-Job -ScriptBlock {
            param($engine, $runId)

            try {
                $engine.ExecutePipelineRun($runId)
            } catch {
                Write-Error "Pipeline execution failed: $_"
            }

        } -ArgumentList $this, $runId

        Write-Host "Pipeline '$pipelineName' started with run ID: $runId" -ForegroundColor Green

        return $pipelineRun
    }

    hidden [void]ExecutePipelineRun([string]$runId) {
        $pipelineRun = $this.PipelineRuns[$runId]
        $pipeline = $this.Pipelines[$pipelineRun.PipelineName]

        Write-Host "Executing pipeline run: $runId" -ForegroundColor Cyan

        try {
            foreach ($stageIndex in 0..($pipeline.Config.Stages.Count - 1)) {
                $stage = $pipeline.Config.Stages[$stageIndex]
                $pipelineRun.CurrentStage = $stageIndex

                Write-Host "Executing stage: $($stage.Name)" -ForegroundColor Yellow

                $stageResult = $this.ExecuteStage($stage, $pipelineRun)

                $pipelineRun.Stages += @{
                    Name = $stage.Name
                    Status = $stageResult.Status
                    Duration = $stageResult.Duration
                    Steps = $stageResult.Steps
                }

                if ($stageResult.Status -eq "Failed") {
                    $pipelineRun.Status = "Failed"
                    break
                }
            }

            if ($pipelineRun.Status -eq "Running") {
                $pipelineRun.Status = "Succeeded"
            }

        } catch {
            $pipelineRun.Status = "Failed"
            Write-Error "Pipeline execution error: $_"
        }

        $pipelineRun.EndTime = Get-Date
        $pipeline.LastRun = Get-Date
        $pipeline.Status = $pipelineRun.Status

        # Notification
        $this.SendNotification(@{
            Type = "PipelineCompleted"
            PipelineName = $pipelineRun.PipelineName
            RunId = $runId
            Status = $pipelineRun.Status
            Duration = $pipelineRun.EndTime - $pipelineRun.StartTime
            Message = "Pipeline $($pipelineRun.PipelineName) $($pipelineRun.Status.ToLower())"
            Level = if ($pipelineRun.Status -eq "Succeeded") { "SUCCESS" } else { "ERROR" }
        })

        # Nettoyage de la queue
        $queueItem = $this.BuildQueue | Where-Object { $_.RunId -eq $runId } | Select-Object -First 1
        if ($queueItem) {
            $this.BuildQueue.Remove($queueItem)
        }
    }

    hidden [PSCustomObject]ExecuteStage([hashtable]$stage, [PSCustomObject]$pipelineRun) {
        $stageStart = Get-Date
        $stepResults = @()

        foreach ($step in $stage.Steps) {
            Write-Host "  Executing step: $($step.Name)" -ForegroundColor Gray

            $stepStart = Get-Date
            $stepResult = $this.ExecuteStep($step, $pipelineRun)

            $stepResult.Duration = (Get-Date) - $stepStart
            $stepResults += $stepResult

            if (-not $stepResult.Success) {
                return @{
                    Status = "Failed"
                    Duration = (Get-Date) - $stageStart
                    Steps = $stepResults
                }
            }
        }

        return @{
            Status = "Succeeded"
            Duration = (Get-Date) - $stageStart
            Steps = $stepResults
        }
    }

    hidden [PSCustomObject]ExecuteStep([hashtable]$step, [PSCustomObject]$pipelineRun) {
        try {
            # Résolution des paramètres avec les variables
            $resolvedParameters = @{}
            foreach ($paramKey in $step.Parameters.Keys) {
                $paramValue = $step.Parameters[$paramKey]
                if ($paramValue -is [string] -and $paramValue -match '\{([^}]+)\}') {
                    # Remplacement des variables
                    $paramValue = $paramValue -replace '\{([^}]+)\}', {
                        $varName = $args[0].Groups[1].Value
                        if ($pipelineRun.Variables.ContainsKey($varName)) {
                            $pipelineRun.Variables[$varName]
                        } else {
                            "{$varName}"  # Garder intact si non trouvé
                        }
                    }
                }
                $resolvedParameters[$paramKey] = $paramValue
            }

            # Exécution selon le type de step
            $result = $this.ExecuteStepByType($step.Type, $resolvedParameters, $pipelineRun)

            return [PSCustomObject]@{
                Name = $step.Name
                Type = $step.Type
                Success = $result.Success
                Output = $result.Output
                Error = $result.Error
                Artifacts = $result.Artifacts ?? @()
            }

        } catch {
            return [PSCustomObject]@{
                Name = $step.Name
                Type = $step.Type
                Success = $false
                Output = $null
                Error = $_.Exception.Message
                Artifacts = @()
            }
        }
    }

    hidden [PSCustomObject]ExecuteStepByType([string]$stepType, [hashtable]$parameters, [PSCustomObject]$pipelineRun) {
        switch ($stepType) {
            "GitClone" {
                $repo = $parameters.Repository
                $branch = $parameters.Branch ?? "main"
                $output = git clone -b $branch $repo . 2>&1
                return @{ Success = $LASTEXITCODE -eq 0; Output = $output; Artifacts = @(".\.git") }
            }

            "DotNetRestore" {
                $output = dotnet restore 2>&1
                return @{ Success = $LASTEXITCODE -eq 0; Output = $output }
            }

            "DotNetBuild" {
                $config = $parameters.Configuration ?? "Release"
                $output = dotnet build --configuration $config 2>&1
                return @{ Success = $LASTEXITCODE -eq 0; Output = $output }
            }

            "DotNetTest" {
                $args = @("--logger", "trx")
                if ($parameters.CollectCoverage) {
                    $args += @("--collect", "XPlat Code Coverage")
                }
                $output = dotnet test @args 2>&1
                return @{ Success = $LASTEXITCODE -eq 0; Output = $output; Artifacts = Get-ChildItem "*.trx" -Recurse }
            }

            "PSScriptAnalyzer" {
                $rules = $parameters.Rules -split ','
                $results = Invoke-ScriptAnalyzer -Path "." -IncludeRule $rules
                $hasErrors = $results | Where-Object { $_.Severity -eq "Error" }
                return @{ Success = $hasErrors.Count -eq 0; Output = $results }
            }

            "PesterTest" {
                $outputPath = $parameters.OutputPath ?? "test-results.xml"
                $outputFormat = $parameters.OutputFormat ?? "NUnitXml"

                $pesterConfig = @{
                    Run = @{ Path = "."; PassThru = $true }
                    TestResult = @{ FilePath = $outputPath; OutputFormat = $outputFormat }
                }

                $result = Invoke-Pester -Configuration $pesterConfig
                return @{ Success = $result.FailedCount -eq 0; Output = $result; Artifacts = @($outputPath) }
            }

            "TerraformValidate" {
                $output = terraform validate 2>&1
                return @{ Success = $LASTEXITCODE -eq 0; Output = $output }
            }

            "AzureWebAppDeploy" {
                # Simulation de déploiement Azure
                Start-Sleep -Seconds 2
                return @{ Success = $true; Output = "Deployed to Azure Web App" }
            }

            default {
                throw "Unknown step type: $stepType"
            }
        }
    }

    [void]AddTrigger([string]$pipelineName, [hashtable]$trigger) {
        if (-not $this.Pipelines.ContainsKey($pipelineName)) {
            throw "Pipeline '$pipelineName' not found"
        }

        $pipeline = $this.Pipelines[$pipelineName]
        if (-not $pipeline.Triggers) {
            $pipeline.Triggers = @()
        }

        $pipeline.Triggers += $trigger

        Write-Host "Trigger added to pipeline '$pipelineName'" -ForegroundColor Green
    }

    [void]ProcessTriggers() {
        foreach ($pipelineName in $this.Pipelines.Keys) {
            $pipeline = $this.Pipelines[$pipelineName]

            foreach ($trigger in $pipeline.Triggers) {
                if ($this.EvaluateTrigger($trigger)) {
                    Write-Host "Trigger activated for pipeline '$pipelineName'" -ForegroundColor Yellow
                    $this.StartPipeline($pipelineName, @{ TriggeredBy = $trigger.Type })
                }
            }
        }
    }

    hidden [bool]EvaluateTrigger([hashtable]$trigger) {
        switch ($trigger.Type) {
            "GitPush" {
                # Vérification des changements Git
                $lastCommit = git log -1 --format="%H %ct" 2>$null
                if ($lastCommit) {
                    $commitHash, $timestamp = $lastCommit -split " "
                    $commitTime = [DateTime]::FromUnixTimeSeconds($timestamp)

                    if (-not $trigger.LastProcessed -or $commitTime -gt $trigger.LastProcessed) {
                        $trigger.LastProcessed = $commitTime
                        return $true
                    }
                }
                return $false
            }

            "Schedule" {
                # Vérification des horaires planifiés
                $now = Get-Date
                if ($trigger.CronExpression) {
                    # Simulation d'évaluation cron (nécessiterait une vraie lib cron)
                    return ($now.Minute % 15) -eq 0  # Toutes les 15 minutes pour la démo
                }
                return $false
            }

            "FileChange" {
                # Vérification des changements de fichiers
                $files = Get-ChildItem $trigger.Path -Recurse -Include $trigger.Filter
                $newestFile = $files | Sort-Object LastWriteTime -Descending | Select-Object -First 1

                if ($newestFile -and (-not $trigger.LastProcessed -or $newestFile.LastWriteTime -gt $trigger.LastProcessed)) {
                    $trigger.LastProcessed = $newestFile.LastWriteTime
                    return $true
                }
                return $false
            }

            default {
                return $false
            }
        }
    }

    [PSCustomObject[]]GetPipelineRuns([string]$pipelineName = $null, [int]$limit = 10) {
        $runs = if ($pipelineName) {
            $this.PipelineRuns.Values | Where-Object { $_.PipelineName -eq $pipelineName }
        } else {
            $this.PipelineRuns.Values
        }

        return $runs | Sort-Object StartTime -Descending | Select-Object -First $limit
    }

    [hashtable]GetPipelineMetrics() {
        $runs = $this.PipelineRuns.Values

        $metrics = @{
            TotalPipelines = $this.Pipelines.Count
            TotalRuns = $runs.Count
            SuccessfulRuns = ($runs | Where-Object { $_.Status -eq "Succeeded" }).Count
            FailedRuns = ($runs | Where-Object { $_.Status -eq "Failed" }).Count
            AverageRunTime = $null
            SuccessRate = 0
            QueuedBuilds = $this.BuildQueue.Count
        }

        if ($runs) {
            $completedRuns = $runs | Where-Object { $_.EndTime }
            if ($completedRuns) {
                $totalTime = ($completedRuns | ForEach-Object { $_.EndTime - $_.StartTime } | Measure-Object -Sum Ticks).Sum
                $metrics.AverageRunTime = [TimeSpan]::FromTicks($totalTime / $completedRuns.Count)
            }

            $metrics.SuccessRate = [math]::Round(($metrics.SuccessfulRuns / $metrics.TotalRuns) * 100, 1)
        }

        return $metrics
    }

    hidden [void]SendNotification([hashtable]$notification) {
        & $this.NotificationCallback $notification
    }

    [string]GenerateReport() {
        $metrics = $this.GetPipelineMetrics()
        $recentRuns = $this.GetPipelineRuns($null, 5)

        $report = @"
POWERSHELL CI/CD PIPELINE REPORT
================================
Generated: $(Get-Date)

PIPELINE METRICS
================
Total Pipelines: $($metrics.TotalPipelines)
Total Runs: $($metrics.TotalRuns)
Success Rate: $($metrics.SuccessRate)%
Queued Builds: $($metrics.QueuedBuilds)

$($metrics.AverageRunTime ? "Average Run Time: $([math]::Round($metrics.AverageRunTime.TotalSeconds, 1)) seconds" : "No completed runs yet")

RECENT PIPELINE RUNS
====================
$($recentRuns | ForEach-Object { "$($_.PipelineName) - $($_.Status) - $([math]::Round(($_.EndTime - $_.StartTime).TotalSeconds, 1))s" } | Out-String)
"@

        return $report
    }
}

# Créateur de pipeline DSL
class PipelineDSL {
    [PowerShellPipelineEngine]$Engine
    [hashtable]$CurrentPipeline
    [string]$CurrentStage

    PipelineDSL([PowerShellPipelineEngine]$engine) {
        $this.Engine = $engine
    }

    [PipelineDSL]Pipeline([string]$name, [scriptblock]$definition) {
        $this.CurrentPipeline = @{
            Name = $name
            Stages = @()
            Triggers = @()
            Variables = @{}
        }

        & $definition

        $this.Engine.CreatePipeline($name, $this.CurrentPipeline)
        return $this
    }

    [PipelineDSL]Stage([string]$name, [scriptblock]$definition) {
        $this.CurrentStage = $name
        $stage = @{
            Name = $name
            Steps = @()
        }

        & $definition

        $this.CurrentPipeline.Stages += $stage
        return $this
    }

    [PipelineDSL]Step([string]$name, [string]$type, [hashtable]$parameters = @{}) {
        $step = @{
            Name = $name
            Type = $type
            Parameters = $parameters
        }

        # Ajout du step au stage courant
        $currentStage = $this.CurrentPipeline.Stages | Where-Object { $_.Name -eq $this.CurrentStage } | Select-Object -First 1
        if ($currentStage) {
            $currentStage.Steps += $step
        }

        return $this
    }

    [PipelineDSL]Trigger([hashtable]$trigger) {
        $this.CurrentPipeline.Triggers += $trigger
        return $this
    }

    [PipelineDSL]Variable([string]$name, $value) {
        $this.CurrentPipeline.Variables[$name] = $value
        return $this
    }

    [PipelineDSL]UseTemplate([string]$templateName, [hashtable]$customizations = @{}) {
        $this.Engine.CreatePipelineFromTemplate($this.CurrentPipeline.Name, $templateName, $customizations)
        return $this
    }
}

# Fonctions utilitaires pour le DSL
function New-PipelineEngine {
    return [PowerShellPipelineEngine]::new()
}

function New-PipelineDSL {
    param([PowerShellPipelineEngine]$engine = $null)

    if (-not $engine) {
        $engine = New-PipelineEngine
    }

    return [PipelineDSL]::new($engine)
}

# Démonstration du framework CI/CD
Write-Host "=== POWERSHELL CI/CD PIPELINE DEMONSTRATION ===" -ForegroundColor Cyan

# Création du moteur de pipeline
$engine = New-PipelineEngine

# Configuration des notifications
$engine.NotificationCallback = {
    param($notification)
    $color = switch ($notification.Level) {
        "SUCCESS" { "Green" }
        "WARNING" { "Yellow" }
        "ERROR" { "Red" }
        default { "White" }
    }
    Write-Host "[$($notification.Type.ToUpper())] $($notification.Message)" -ForegroundColor $color
}

# Utilisation du DSL pour créer des pipelines
$dsl = New-PipelineDSL -Engine $engine

# Pipeline pour une application .NET
$dsl.Pipeline("MyDotNetApp", {
    Stage "Build", {
        Step "Checkout" "GitClone" @{ Repository = "https://github.com/myorg/myapp.git"; Branch = "main" }
        Step "Restore" "DotNetRestore" @{}
        Step "Build" "DotNetBuild" @{ Configuration = "Release" }
    }

    Stage "Test", {
        Step "UnitTests" "DotNetTest" @{ CollectCoverage = $true }
        Step "IntegrationTests" "PSTest" @{ TestPath = "tests\integration" }
    }

    Stage "Deploy", {
        Step "Package" "DotNetPublish" @{ OutputPath = ".\publish" }
        Step "DeployStaging" "AzureWebAppDeploy" @{ Slot = "staging" }
        Step "Swap" "AzureSlotSwap" @{}
    }

    Trigger @{ Type = "GitPush"; Branch = "main" }
    Variable "BuildNumber" "1.0.0"
    Variable "Environment" "Production"
}) | Out-Null

# Pipeline pour un module PowerShell
$dsl.Pipeline("PowerShellModule", {
    UseTemplate "PowerShellModule" @{
        Variables = @{
            BuildNumber = "2.1.0"
            GalleryApiKey = "your-api-key"
        }
    }
}) | Out-Null

# Pipeline Infrastructure as Code
$dsl.Pipeline("Infrastructure", {
    UseTemplate "InfrastructureAsCode" @{}
}) | Out-Null

# Démarrage des pipelines
$run1 = $engine.StartPipeline("MyDotNetApp", @{ BuildNumber = "1.2.3"; Environment = "Staging" })
$run2 = $engine.StartPipeline("PowerShellModule")

# Traitement des triggers
$engine.ProcessTriggers()

# Attente de la fin des pipelines
Start-Sleep -Seconds 5

# Métriques des pipelines
$metrics = $engine.GetPipelineMetrics()
Write-Host "`n=== PIPELINE METRICS ===" -ForegroundColor Cyan
Write-Host "Total pipelines: $($metrics.TotalPipelines)"
Write-Host "Total runs: $($metrics.TotalRuns)"
Write-Host "Success rate: $($metrics.SuccessRate)%"
Write-Host "Queued builds: $($metrics.QueuedBuilds)"

# Rapport des pipelines
$report = $engine.GenerateReport()
$report | Out-File -FilePath "PipelineReport_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nCI/CD pipeline demonstration completed!" -ForegroundColor Green
```

### 1.2 Intégration avec les outils DevOps

**Framework d'intégration avec les outils DevOps populaires :**
```powershell
# Framework d'intégration DevOps avancé
class DevOpsIntegrationHub {
    [System.Collections.Generic.Dictionary[string, hashtable]]$Integrations
    [System.Collections.Generic.List[PSCustomObject]]$SyncOperations
    [hashtable]$ToolConfigurations

    DevOpsIntegrationHub() {
        $this.Integrations = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.SyncOperations = [System.Collections.Generic.List[PSCustomObject]]::new()

        $this.InitializeToolConfigurations()
    }

    hidden [void]InitializeToolConfigurations() {
        $this.ToolConfigurations = @{
            AzureDevOps = @{
                ApiVersion = "7.1"
                BaseUrl = "https://dev.azure.com"
                Authentication = "PAT"  # Personal Access Token
            }
            GitHub = @{
                ApiVersion = "2022-11-28"
                BaseUrl = "https://api.github.com"
                Authentication = "Token"
            }
            Jenkins = @{
                ApiVersion = "2.0"
                Authentication = "Basic"
            }
            GitLab = @{
                ApiVersion = "v4"
                BaseUrl = "https://gitlab.com/api"
                Authentication = "Token"
            }
            Jira = @{
                ApiVersion = "3"
                BaseUrl = "https://your-domain.atlassian.net/rest/api"
                Authentication = "Basic"
            }
            ServiceNow = @{
                ApiVersion = "v2"
                BaseUrl = "https://your-instance.service-now.com/api"
                Authentication = "Basic"
            }
        }
    }

    [void]ConfigureIntegration([string]$toolName, [hashtable]$config) {
        if (-not $this.ToolConfigurations.ContainsKey($toolName)) {
            throw "Tool '$toolName' is not supported"
        }

        $integration = @{
            ToolName = $toolName
            Config = $config
            Connected = $false
            LastSync = $null
            Status = "Configured"
        }

        $this.Integrations[$toolName] = $integration

        Write-Host "Integration configured for: $toolName" -ForegroundColor Green
    }

    [void]ConnectIntegration([string]$toolName) {
        if (-not $this.Integrations.ContainsKey($toolName)) {
            throw "Integration '$toolName' not configured"
        }

        $integration = $this.Integrations[$toolName]
        $toolConfig = $this.ToolConfigurations[$toolName]

        try {
            # Test de connexion selon l'outil
            $connectionTest = $this.TestToolConnection($toolName, $integration.Config, $toolConfig)

            if ($connectionTest.Success) {
                $integration.Connected = $true
                $integration.Status = "Connected"
                $integration.LastSync = Get-Date

                Write-Host "Successfully connected to: $toolName" -ForegroundColor Green
            } else {
                throw $connectionTest.Error
            }

        } catch {
            $integration.Status = "ConnectionFailed"
            Write-Error "Failed to connect to $toolName`: $_"
        }
    }

    hidden [PSCustomObject]TestToolConnection([string]$toolName, [hashtable]$config, [hashtable]$toolConfig) {
        switch ($toolName) {
            "AzureDevOps" {
                # Test de connexion Azure DevOps
                $headers = @{
                    "Authorization" = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$($config.PAT)"))
                    "Content-Type" = "application/json"
                }

                $url = "$($toolConfig.BaseUrl)/$($config.Organization)/$($config.Project)/_apis/build/builds?api-version=$($toolConfig.ApiVersion)"
                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Get

                return @{ Success = $true; Data = $response }
            }

            "GitHub" {
                # Test de connexion GitHub
                $headers = @{
                    "Authorization" = "token $($config.Token)"
                    "Accept" = "application/vnd.github.v3+json"
                }

                $url = "$($toolConfig.BaseUrl)/user"
                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Get

                return @{ Success = $true; Data = $response }
            }

            "Jenkins" {
                # Test de connexion Jenkins
                $pair = "$($config.UserName):$($config.ApiToken)"
                $bytes = [System.Text.Encoding]::ASCII.GetBytes($pair)
                $credentials = [System.Convert]::ToBase64String($bytes)

                $headers = @{
                    "Authorization" = "Basic $credentials"
                }

                $url = "$($config.Url)/api/json"
                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Get

                return @{ Success = $true; Data = $response }
            }

            "GitLab" {
                # Test de connexion GitLab
                $headers = @{
                    "PRIVATE-TOKEN" = $config.Token
                }

                $url = "$($toolConfig.BaseUrl)/$($toolConfig.ApiVersion)/user"
                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Get

                return @{ Success = $true; Data = $response }
            }

            default {
                return @{ Success = $false; Error = "Tool '$toolName' connection test not implemented" }
            }
        }
    }

    [PSCustomObject]SyncWorkItems([string]$sourceTool, [string]$targetTool, [hashtable]$syncConfig) {
        $syncId = [guid]::NewGuid().ToString()

        $syncOperation = [PSCustomObject]@{
            Id = $syncId
            SourceTool = $sourceTool
            TargetTool = $targetTool
            Status = "Running"
            StartTime = Get-Date
            EndTime = $null
            ItemsSynced = 0
            Errors = 0
            Config = $syncConfig
        }

        $this.SyncOperations.Add($syncOperation)

        try {
            # Récupération des éléments depuis la source
            $sourceItems = $this.GetWorkItems($sourceTool, $syncConfig.SourceQuery)

            # Transformation et synchronisation
            foreach ($sourceItem in $sourceItems) {
                try {
                    $transformedItem = $this.TransformWorkItem($sourceItem, $syncConfig.Mapping)
                    $this.CreateWorkItem($targetTool, $transformedItem, $syncConfig.TargetConfig)

                    $syncOperation.ItemsSynced++
                } catch {
                    $syncOperation.Errors++
                    Write-Warning "Failed to sync item $($sourceItem.Id): $_"
                }
            }

            $syncOperation.Status = "Completed"

        } catch {
            $syncOperation.Status = "Failed"
            $syncOperation.Error = $_.Exception.Message
            Write-Error "Sync operation failed: $_"
        }

        $syncOperation.EndTime = Get-Date

        return $syncOperation
    }

    hidden [array]GetWorkItems([string]$toolName, [hashtable]$query) {
        $integration = $this.Integrations[$toolName]

        switch ($toolName) {
            "AzureDevOps" {
                $headers = @{
                    "Authorization" = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$($integration.Config.PAT)"))
                    "Content-Type" = "application/json"
                }

                $url = "$($this.ToolConfigurations[$toolName].BaseUrl)/$($integration.Config.Organization)/$($integration.Config.Project)/_apis/wit/wiql?api-version=$($this.ToolConfigurations[$toolName].ApiVersion)"

                $queryBody = @{
                    query = $query.WIQL
                }

                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Post -Body ($queryBody | ConvertTo-Json)
                return $response.workItems
            }

            "Jira" {
                $pair = "$($integration.Config.UserName):$($integration.Config.ApiToken)"
                $bytes = [System.Text.Encoding]::ASCII.GetBytes($pair)
                $credentials = [System.Convert]::ToBase64String($bytes)

                $headers = @{
                    "Authorization" = "Basic $credentials"
                    "Content-Type" = "application/json"
                }

                $url = "$($this.ToolConfigurations[$toolName].BaseUrl)/$($this.ToolConfigurations[$toolName].ApiVersion)/search"
                $jql = $query.JQL ?? "project = $($query.Project)"

                $body = @{
                    jql = $jql
                    startAt = 0
                    maxResults = 100
                    fields = @("summary", "description", "status", "assignee")
                }

                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Post -Body ($body | ConvertTo-Json)
                return $response.issues
            }

            default {
                throw "Work item retrieval not implemented for tool: $toolName"
            }
        }
    }

    hidden [hashtable]TransformWorkItem([PSCustomObject]$sourceItem, [hashtable]$mapping) {
        $transformed = @{}

        foreach ($targetField in $mapping.Keys) {
            $sourceField = $mapping[$targetField]

            if ($sourceField -is [scriptblock]) {
                # Transformation avec scriptblock
                $transformed[$targetField] = & $sourceField $sourceItem
            } else {
                # Mapping direct
                $value = $sourceItem
                foreach ($field in $sourceField -split '\.') {
                    $value = $value.$field
                }
                $transformed[$targetField] = $value
            }
        }

        return $transformed
    }

    hidden [void]CreateWorkItem([string]$toolName, [hashtable]$item, [hashtable]$config) {
        $integration = $this.Integrations[$toolName]

        switch ($toolName) {
            "AzureDevOps" {
                $headers = @{
                    "Authorization" = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$($integration.Config.PAT)"))
                    "Content-Type" = "application/json"
                }

                $url = "$($this.ToolConfigurations[$toolName].BaseUrl)/$($integration.Config.Organization)/$($integration.Config.Project)/_apis/wit/workitems/`$Task?api-version=$($this.ToolConfigurations[$toolName].ApiVersion)"

                $patches = @()
                foreach ($field in $item.Keys) {
                    $patches += @{
                        op = "add"
                        path = "/fields/$field"
                        value = $item[$field]
                    }
                }

                Invoke-RestMethod -Uri $url -Headers $headers -Method Patch -Body ($patches | ConvertTo-Json) | Out-Null
            }

            "Jira" {
                $pair = "$($integration.Config.UserName):$($integration.Config.ApiToken)"
                $bytes = [System.Text.Encoding]::ASCII.GetBytes($pair)
                $credentials = [System.Convert]::ToBase64String($bytes)

                $headers = @{
                    "Authorization" = "Basic $credentials"
                    "Content-Type" = "application/json"
                }

                $url = "$($this.ToolConfigurations[$toolName].BaseUrl)/$($this.ToolConfigurations[$toolName].ApiVersion)/issue"

                $issueData = @{
                    fields = @{
                        project = @{ key = $config.ProjectKey }
                        summary = $item.Summary
                        description = $item.Description
                        issuetype = @{ name = "Task" }
                    }
                }

                if ($item.Assignee) {
                    $issueData.fields.assignee = @{ name = $item.Assignee }
                }

                Invoke-RestMethod -Uri $url -Headers $headers -Method Post -Body ($issueData | ConvertTo-Json) | Out-Null
            }
        }
    }

    [PSCustomObject]SyncBuildStatus([string]$sourceTool, [string]$targetTool, [string]$buildId) {
        $syncId = [guid]::NewGuid().ToString()

        $syncOperation = [PSCustomObject]@{
            Id = $syncId
            Operation = "BuildStatusSync"
            SourceTool = $sourceTool
            TargetTool = $targetTool
            BuildId = $buildId
            Status = "Running"
            StartTime = Get-Date
            EndTime = $null
        }

        try {
            # Récupération du statut du build depuis la source
            $buildStatus = $this.GetBuildStatus($sourceTool, $buildId)

            # Synchronisation vers la cible
            $this.UpdateBuildStatus($targetTool, $buildStatus)

            $syncOperation.Status = "Completed"

        } catch {
            $syncOperation.Status = "Failed"
            $syncOperation.Error = $_.Exception.Message
        }

        $syncOperation.EndTime = Get-Date
        return $syncOperation
    }

    hidden [PSCustomObject]GetBuildStatus([string]$toolName, [string]$buildId) {
        $integration = $this.Integrations[$toolName]

        switch ($toolName) {
            "AzureDevOps" {
                $headers = @{
                    "Authorization" = "Basic " + [Convert]::ToBase64String([Text.Encoding]::ASCII.GetBytes(":$($integration.Config.PAT)"))
                    "Content-Type" = "application/json"
                }

                $url = "$($this.ToolConfigurations[$toolName].BaseUrl)/$($integration.Config.Organization)/$($integration.Config.Project)/_apis/build/builds/$buildId/?api-version=$($this.ToolConfigurations[$toolName].ApiVersion)"

                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Get

                return @{
                    BuildId = $response.id
                    Status = $response.status
                    Result = $response.result
                    StartTime = $response.startTime
                    FinishTime = $response.finishTime
                    SourceBranch = $response.sourceBranch
                    RequestedBy = $response.requestedBy.displayName
                }
            }

            "Jenkins" {
                $pair = "$($integration.Config.UserName):$($integration.Config.ApiToken)"
                $bytes = [System.Text.Encoding]::ASCII.GetBytes($pair)
                $credentials = [System.Convert]::ToBase64String($bytes)

                $headers = @{
                    "Authorization" = "Basic $credentials"
                }

                $url = "$($integration.Config.Url)/job/$($integration.Config.JobName)/$buildId/api/json"
                $response = Invoke-RestMethod -Uri $url -Headers $headers -Method Get

                return @{
                    BuildId = $response.number
                    Status = if ($response.building) { "inProgress" } elseif ($response.result -eq "SUCCESS") { "succeeded" } else { "failed" }
                    Result = $response.result
                    StartTime = [DateTime]::FromUnixTimeMilliseconds($response.timestamp)
                    FinishTime = if (-not $response.building) { [DateTime]::FromUnixTimeMilliseconds($response.timestamp + $response.duration) } else { $null }
                    TriggeredBy = $response.actions | Where-Object { $_.causes } | ForEach-Object { $_.causes[0].userName } | Select-Object -First 1
                }
            }

            default {
                throw "Build status retrieval not implemented for tool: $toolName"
            }
        }
    }

    hidden [void]UpdateBuildStatus([string]$toolName, [PSCustomObject]$buildStatus) {
        # Cette méthode serait implémentée pour mettre à jour le statut dans l'outil cible
        Write-Host "Updating build status in $toolName..." -ForegroundColor Gray
        # Implémentation spécifique à l'outil cible
    }

    [hashtable]GetIntegrationMetrics() {
        $metrics = @{
            TotalIntegrations = $this.Integrations.Count
            ConnectedIntegrations = ($this.Integrations.Values | Where-Object { $_.Connected }).Count
            SyncOperations = $this.SyncOperations.Count
            SuccessfulSyncs = ($this.SyncOperations | Where-Object { $_.Status -eq "Completed" }).Count
            FailedSyncs = ($this.SyncOperations | Where-Object { $_.Status -eq "Failed" }).Count
        }

        $metrics.SyncSuccessRate = if ($metrics.SyncOperations -gt 0) {
            [math]::Round(($metrics.SuccessfulSyncs / $metrics.SyncOperations) * 100, 1)
        } else { 0 }

        return $metrics
    }

    [PSCustomObject[]]GetSyncOperations([int]$lastHours = 24) {
        $cutoff = (Get-Date).AddHours(-$lastHours)
        return $this.SyncOperations | Where-Object { $_.StartTime -ge $cutoff } | Sort-Object StartTime -Descending
    }

    [string]GenerateReport() {
        $metrics = $this.GetIntegrationMetrics()
        $syncOps = $this.GetSyncOperations(24)

        $report = @"
DEVOPS INTEGRATION HUB REPORT
=============================
Generated: $(Get-Date)

INTEGRATION METRICS
==================
Total Integrations: $($metrics.TotalIntegrations)
Connected Integrations: $($metrics.ConnectedIntegrations)
Sync Operations: $($metrics.SyncOperations)
Sync Success Rate: $($metrics.SyncSuccessRate)%

CONNECTED TOOLS
===============
$($this.Integrations.Values | Where-Object { $_.Connected } | ForEach-Object { "• $($_.ToolName) - Last sync: $($_.LastSync)" } | Out-String)

RECENT SYNC OPERATIONS
======================
$($syncOps | Select-Object -First 10 | ForEach-Object { "$($_.StartTime): $($_.SourceTool) -> $($_.TargetTool) - $($_.Status) ($($_.ItemsSynced) items)" } | Out-String)
"@

        return $report
    }
}

# Démonstration du hub d'intégration DevOps
Write-Host "=== DEVOPS INTEGRATION HUB DEMONSTRATION ===" -ForegroundColor Cyan

$hub = [DevOpsIntegrationHub]::new()

# Configuration des intégrations
$hub.ConfigureIntegration("AzureDevOps", @{
    Organization = "myorg"
    Project = "myproject"
    PAT = "your-personal-access-token"
})

$hub.ConfigureIntegration("GitHub", @{
    Token = "your-github-token"
    Repository = "myorg/myrepo"
})

$hub.ConfigureIntegration("Jira", @{
    Url = "https://mycompany.atlassian.net"
    UserName = "powershell@company.com"
    ApiToken = "your-jira-api-token"
})

# Connexion aux outils
foreach ($toolName in $hub.Integrations.Keys) {
    try {
        $hub.ConnectIntegration($toolName)
    } catch {
        Write-Warning "Failed to connect to $toolName`: $_"
    }
}

# Synchronisation des work items (exemple)
$syncConfig = @{
    SourceQuery = @{ WIQL = "SELECT [System.Id] FROM WorkItems WHERE [System.State] = 'Active'" }
    TargetConfig = @{ ProjectKey = "PROJ" }
    Mapping = @{
        Summary = "fields.'System.Title'"
        Description = "fields.'System.Description'"
        Assignee = { param($item) $item.fields.'System.AssignedTo' }
    }
}

try {
    $syncResult = $hub.SyncWorkItems("AzureDevOps", "Jira", $syncConfig)
    Write-Host "Work item sync completed: $($syncResult.ItemsSynced) items synced" -ForegroundColor Green
} catch {
    Write-Warning "Work item sync failed: $_"
}

# Rapport d'intégration
$report = $hub.GenerateReport()
$report | Out-File -FilePath "DevOpsIntegrationReport_$(Get-Date -Format 'yyyy-MM-dd_HH-mm-ss').txt" -Encoding UTF8

Write-Host "`nDevOps integration hub demonstration completed!" -ForegroundColor Green
```

## Section 2 : Tests automatisés et qualité de code

### 2.1 Framework de tests PowerShell avancés

**Suite de tests complète avec mocking et coverage :**
```powershell
# Framework de tests avancés PowerShell avec mocking et métriques de couverture
class PowerShellTestSuite {
    [System.Collections.Generic.List[PSCustomObject]]$Tests
    [System.Collections.Generic.Dictionary[string, scriptblock]]$Mocks
    [System.Collections.Generic.Dictionary[string, hashtable]]$TestData
    [hashtable]$TestMetrics
    [System.Collections.Generic.List[string]]$CoverageData
    [bool]$EnableCoverage

    PowerShellTestSuite([bool]$enableCoverage = $true) {
        $this.Tests = [System.Collections.Generic.List[PSCustomObject]]::new()
        $this.Mocks = [System.Collections.Generic.Dictionary[string, scriptblock]]::new()
        $this.TestData = [System.Collections.Generic.Dictionary[string, hashtable]]::new()
        $this.CoverageData = [System.Collections.Generic.List[string]]::new()
        $this.EnableCoverage = $enableCoverage

        $this.TestMetrics = @{
            TotalTests = 0
            PassedTests = 0
            FailedTests = 0
            SkippedTests = 0
            CoveragePercentage = 0
            AverageTestTime = 0
            TotalTestTime = 0
        }
    }

    [void]AddTest([string]$name, [string]$category, [scriptblock]$testBlock, [string[]]$tags = @()) {
        $test = [PSCustomObject]@{
            Name = $name
            Category = $category
            TestBlock = $testBlock
            Tags = $tags
            Status = "Pending"
            ExecutionTime = 0
            Error = $null
            Assertions = 0
            Failures = 0
        }

        $this.Tests.Add($test)
    }

    [void]AddMock([string]$commandName, [scriptblock]$mockImplementation) {
        $this.Mocks[$commandName] = $mockImplementation
    }

    [void]AddTestData([string]$dataSetName, [hashtable]$data) {
        $this.TestData[$dataSetName] = $data
    }

    [PSCustomObject[]]RunTests([string[]]$categories = @(), [string[]]$tags = @()) {
        Write-Host "=== RUNNING POWERSHELL TEST SUITE ===" -ForegroundColor Cyan
        Write-Host "Started at: $(Get-Date)" -ForegroundColor Gray

        # Filtrage des tests
        $testsToRun = $this.Tests

        if ($categories) {
            $testsToRun = $testsToRun | Where-Object { $_.Category -in $categories }
        }

        if ($tags) {
            $testsToRun = $testsToRun | Where-Object {
                $testTags = $_.Tags
                $tagsMatch = $true
                foreach ($tag in $tags) {
                    if ($tag -notin $testTags) {
                        $tagsMatch = $false
                        break
                    }
                }
                $tagsMatch
            }
        }

        Write-Host "Running $($testsToRun.Count) tests..." -ForegroundColor Yellow

        # Configuration du coverage si activé
        if ($this.EnableCoverage) {
            $this.StartCoverageTracking()
        }

        # Configuration des mocks
        $this.SetupMocks()

        $results = @()
        $totalStartTime = Get-Date

        foreach ($test in $testsToRun) {
            $result = $this.RunTest($test)
            $results += $result
        }

        $totalTime = (Get-Date) - $totalStartTime

        # Finalisation du coverage
        if ($this.EnableCoverage) {
            $this.StopCoverageTracking()
            $this.TestMetrics.CoveragePercentage = $this.CalculateCoverage()
        }

        # Mise à jour des métriques
        $this.UpdateTestMetrics($results, $totalTime)

        # Rapport des résultats
        $this.GenerateTestReport($results)

        return $results
    }

    hidden [PSCustomObject]RunTest([PSCustomObject]$test) {
        Write-Host "  Running test: $($test.Name)" -ForegroundColor Gray

        $test.Status = "Running"
        $testStartTime = Get-Date

        # Contexte d'assertion
        $assertContext = @{
            Assertions = 0
            Failures = 0
            Errors = [System.Collections.Generic.List[string]]::new()
        }

        # Fonctions d'assertion dans le scope du test
        $assertEqual = {
            param($actual, $expected, $message = "")
            $assertContext.Assertions++
            if ($actual -ne $expected) {
                $assertContext.Failures++
                $assertContext.Errors.Add("Expected '$expected', got '$actual'. $message")
            }
        }

        $assertTrue = {
            param($condition, $message = "")
            $assertContext.Assertions++
            if (-not $condition) {
                $assertContext.Failures++
                $assertContext.Errors.Add("Condition is false. $message")
            }
        }

        $assertNotNull = {
            param($value, $message = "")
            $assertContext.Assertions++
            if ($null -eq $value) {
                $assertContext.Failures++
                $assertContext.Errors.Add("Value is null. $message")
            }
        }

        $assertThrows = {
            param($scriptBlock, $expectedException = $null, $message = "")
            $assertContext.Assertions++
            try {
                & $scriptBlock
                $assertContext.Failures++
                $assertContext.Errors.Add("Expected exception was not thrown. $message")
            } catch {
                if ($expectedException -and $_.Exception.GetType().Name -ne $expectedException) {
                    $assertContext.Failures++
                    $assertContext.Errors.Add("Expected exception '$expectedException', got '$($_.Exception.GetType().Name)'. $message")
                }
            }
        }

        $assertGreaterThan = {
            param($actual, $expected, $message = "")
            $assertContext.Assertions++
            if ($actual -le $expected) {
                $assertContext.Failures++
                $assertContext.Errors.Add("Expected '$actual' to be greater than '$expected'. $message")
            }
        }

        try {
            # Exécution du test
            & $test.TestBlock

            $test.Status = if ($assertContext.Failures -eq 0) { "Passed" } else { "Failed" }
            $test.Assertions = $assertContext.Assertions
            $test.Failures = $assertContext.Failures

            if ($test.Status -eq "Passed") {
                Write-Host "    ✓ PASSED ($($test.Assertions) assertions)" -ForegroundColor Green
            } else {
                Write-Host "    ✗ FAILED ($($test.Failures) failures)" -ForegroundColor Red
                foreach ($error in $assertContext.Errors) {
                    Write-Host "      - $error" -ForegroundColor Red
                }
            }

        } catch {
            $test.Status = "Error"
            $test.Error = $_.Exception.Message
            Write-Host "    ✗ ERROR: $($_.Exception.Message)" -ForegroundColor Red
        }

        $test.ExecutionTime = ((Get-Date) - $testStartTime).TotalMilliseconds

        return $test
    }

    hidden [void]SetupMocks() {
        foreach ($commandName in $this.Mocks.Keys) {
            $mockImplementation = $this.Mocks[$commandName]

            # Création d'une fonction mock qui remplace la commande originale
            $mockFunction = [scriptblock]::Create(@"
                param(`$args__)
                & `$mockImplementation @args__
"@)

            Set-Item -Path "function:global:$commandName" -Value $mockFunction
        }
    }

    hidden [void]StartCoverageTracking() {
        # Hook pour tracer l'exécution des lignes
        $originalExecutionContext = $ExecutionContext

        $ExecutionContext = @{
            InvokeCommand = @{
                GetCommand = {
                    param($name, $type, $args)
                    # Tracer l'appel de commande
                    if ($name) {
                        $global:CoverageTracker.Add($name)
                    }
                    # Appel de la fonction originale
                    & $originalExecutionContext.InvokeCommand.GetCommand $name $type $args
                }
            }
        }

        $global:CoverageTracker = [System.Collections.Generic.HashSet[string]]::new()
    }

    hidden [void]StopCoverageTracking() {
        # Restauration du contexte d'exécution
        $ExecutionContext = $originalExecutionContext

        # Collecte des données de couverture
        $this.CoverageData.AddRange($global:CoverageTracker)
        Remove-Variable -Name "CoverageTracker" -Scope Global -ErrorAction SilentlyContinue
    }

    hidden [double]CalculateCoverage() {
        # Calcul simplifié de la couverture (à améliorer avec analyse AST réelle)
        $totalLines = Get-ChildItem "*.ps1" -Recurse | Get-Content | Measure-Object -Line | Select-Object -ExpandProperty Lines
        $coveredLines = $this.CoverageData.Count

        return if ($totalLines -gt 0) { [math]::Min(100, ($coveredLines / $totalLines) * 100) } else { 0 }
    }

    hidden [void]UpdateTestMetrics([PSCustomObject[]]$results, [TimeSpan]$totalTime) {
        $this.TestMetrics.TotalTests = $results.Count
        $this.TestMetrics.PassedTests = ($results | Where-Object { $_.Status -eq "Passed" }).Count
        $this.TestMetrics.FailedTests = ($results | Where-Object { $_.Status -eq "Failed" -or $_.Status -eq "Error" }).Count
        $this.TestMetrics.SkippedTests = ($results | Where-Object { $_.Status -eq "Skipped" }).Count
        $this.TestMetrics.TotalTestTime = $totalTime.TotalMilliseconds

        if ($this.TestMetrics.TotalTests -gt 0) {
            $this.TestMetrics.AverageTestTime = $this.TestMetrics.TotalTestTime / $this.TestMetrics.TotalTests
        }
    }

    hidden [void]GenerateTestReport([PSCustomObject[]]$results) {
        Write-Host "`n=== TEST SUITE RESULTS ===" -ForegroundColor Cyan
        Write-Host "Total tests: $($this.TestMetrics.TotalTests)"
        Write-Host "Passed: $($this.TestMetrics.PassedTests)" -ForegroundColor Green
        Write-Host "Failed: $($this.TestMetrics.FailedTests)" -ForegroundColor Red
        Write-Host "Skipped: $($this.TestMetrics.SkippedTests)" -ForegroundColor Yellow

        $successRate = [math]::Round(($this.TestMetrics.PassedTests / $this.TestMetrics.TotalTests) * 100, 1)
        Write-Host "Success rate: $successRate%"

        if ($this.EnableCoverage) {
            Write-Host "Code coverage: $([math]::Round($this.TestMetrics.CoveragePercentage, 1))%"
        }

        Write-Host "Total execution time: $([math]::Round($this.TestMetrics.TotalTestTime / 1000, 2)) seconds"
        Write-Host "Average test time: $([math]::Round($this.TestMetrics.AverageTestTime, 2)) ms"

        # Rapport détaillé des échecs
        $failedTests = $results | Where-Object { $_.Status -in @("Failed", "Error") }
        if ($failedTests) {
            Write-Host "`n=== FAILED TESTS ===" -ForegroundColor Red
            foreach ($test in $failedTests) {
                Write-Host "• $($test.Category).$($test.Name): $($test.Error)" -ForegroundColor Red
            }
        }

        # Rapport par catégorie
        Write-Host "`n=== RESULTS BY CATEGORY ===" -ForegroundColor Yellow
        $resultsByCategory = $results | Group-Object Category
        foreach ($category in $resultsByCategory) {
            $catPassed = ($category.Group | Where-Object { $_.Status -eq "Passed" }).Count
            $catTotal = $category.Count
            $catRate = [math]::Round(($catPassed / $catTotal) * 100, 1)
            Write-Host "$($category.Name): $catPassed/$catTotal tests passed ($catRate%)"
        }
    }

    [hashtable]GetTestData([string]$dataSetName) {
        return $this.TestData[$dataSetName]
    }

    [hashtable]GetMetrics() {
        return $this.TestMetrics.Clone()
    }

    [void]ExportResults([string]$outputPath) {
        $results = @{
            Metrics = $this.GetMetrics()
            Tests = $this.Tests
            CoverageData = $this.CoverageData
            GeneratedAt = Get-Date
        }

        $results | ConvertTo-Json -Depth 10 | Out-File -FilePath $outputPath -Encoding UTF8
        Write-Host "Test results exported to: $outputPath" -ForegroundColor Green
    }
}

# Utilitaires pour les tests
function New-TestSuite {
    param([bool]$EnableCoverage = $true)
    return [PowerShellTestSuite]::new($EnableCoverage)
}

function Mock-Command {
    param([string]$CommandName, [scriptblock]$Implementation)
    # Cette fonction serait utilisée dans le contexte d'une suite de tests
}

function Get-TestData {
    param([string]$DataSet)
    # Cette fonction serait utilisée dans le contexte d'une suite de tests
}

# Exemple d'utilisation du framework de tests
Write-Host "=== POWERSHELL TEST SUITE DEMONSTRATION ===" -ForegroundColor Cyan

$suite = New-TestSuite -EnableCoverage $true

# Ajout de mocks
$suite.AddMock("Get-ExternalData", {
    # Mock qui retourne des données de test
    return @{
        Id = 1
        Name = "Mocked Data"
        Status = "Active"
    }
})

$suite.AddMock("Send-Notification", {
    param($message)
    # Mock qui simule l'envoi de notification
    Write-Host "MOCK: Notification sent - $message" -ForegroundColor Gray
})

# Ajout de données de test
$suite.AddTestData("ValidUsers", @(
    @{ Id = 1; Name = "Alice"; Role = "Admin" }
    @{ Id = 2; Name = "Bob"; Role = "User" }
    @{ Id = 3; Name = "Charlie"; Role = "User" }
))

$suite.AddTestData("InvalidInputs", @(
    @{ Input = $null; ExpectedError = "Input cannot be null" }
    @{ Input = ""; ExpectedError = "Input cannot be empty" }
    @{ Input = "   "; ExpectedError = "Input cannot be whitespace" }
))

# Tests unitaires
$suite.AddTest("ValidateUserInput", "InputValidation", {
    $testData = Get-TestData "InvalidInputs"

    foreach ($data in $testData) {
        Assert-Throws { Validate-UserInput $data.Input } "ArgumentException" $data.ExpectedError
    }

    # Test des entrées valides
    Assert-Equal (Validate-UserInput "valid input") "valid input"
}, @("Unit", "Validation"))

$suite.AddTest("ProcessUserData", "DataProcessing", {
    $userData = Get-ExternalData  # Utilise le mock

    Assert-NotNull $userData
    Assert-Equal $userData.Name "Mocked Data"
    Assert-Equal $userData.Status "Active"
}, @("Unit", "Integration"))

$suite.AddTest("NotificationSystem", "Notification", {
    # Test du système de notification avec mock
    $result = Send-Notification "Test message"
    Assert-True $true  # Le mock ne lance pas d'exception
}, @("Unit", "Notification"))

# Tests d'intégration
$suite.AddTest("UserWorkflow", "Integration", {
    # Test d'un workflow complet
    $users = Get-TestData "ValidUsers"

    foreach ($user in $users) {
        $processedUser = Process-User $user
        Assert-NotNull $processedUser
        Assert-Equal $processedUser.Id $user.Id
        Assert-Equal $processedUser.Name $user.Name
    }

    # Vérification que les notifications ont été envoyées
    Assert-True $true  # Simulation
}, @("Integration", "Workflow"))

# Tests de performance
$suite.AddTest("PerformanceTest", "Performance", {
    $startTime = Get-Date

    # Simulation d'une opération qui devrait être rapide
    for ($i = 1; $i -le 1000; $i++) {
        $result = [math]::Sqrt($i)
    }

    $duration = ((Get-Date) - $startTime).TotalMilliseconds
    Assert-GreaterThan 500 $duration "Operation should complete in less than 500ms"
}, @("Performance", "Benchmark"))

# Exécution des tests
$results = $suite.RunTests()

# Exécution avec filtrage
Write-Host "`n=== RUNNING UNIT TESTS ONLY ===" -ForegroundColor Yellow
$unitResults = $suite.RunTests(@(), @("Unit"))

# Métriques finales
$metrics = $suite.GetMetrics()
Write-Host "`n=== FINAL TEST METRICS ===" -ForegroundColor Cyan
Write-Host "Coverage: $([math]::Round($metrics.CoveragePercentage, 1))%"
Write-Host "Average test time: $([math]::Round($metrics.AverageTestTime, 2))ms"

# Export des résultats
$suite.ExportResults("TestSuiteResults.json")

Write-Host "`nAdvanced test suite demonstration completed!" -ForegroundColor Green
```

## Conclusion : PowerShell comme moteur DevOps

PowerShell transcende son rôle d'outil de scripting pour devenir le moteur central des pratiques DevOps modernes. En intégrant les pipelines CI/CD, les tests automatisés, l'infrastructure as code, et les intégrations multi-outils, PowerShell crée des workflows d'automatisation d'une sophistication et d'une fiabilité sans précédent.

Dans le prochain chapitre, nous explorerons les techniques avancées de monitoring et d'observabilité avec PowerShell, complétant notre maîtrise de l'écosystème DevOps moderne.

---

**Exercice pratique :** Créez une plateforme DevOps complète avec PowerShell qui inclut :
1. Un pipeline CI/CD multi-étapes avec qualité de code intégrée
2. Des intégrations avec Azure DevOps, GitHub Actions, et Jenkins
3. Une suite de tests automatisés avec mocking et couverture de code
4. Des déploiements blue-green avec rollback automatique
5. Un tableau de bord de métriques et d'observabilité

**Challenge avancé :** Développez une plateforme DevOps qui :
- Implémente GitOps avec synchronisation bidirectionnelle
- Fournit des déploiements canary avec monitoring temps réel
- Intègre l'IA pour la prédiction des échecs de déploiement
- Supporte le multi-cloud avec failover automatique
- Implémente des politiques de sécurité zero-trust dans les pipelines

**Réflexion :** Comment PowerShell transforme-t-il les pratiques DevOps ? Les pipelines déclaratifs sont-ils plus maintenables que les scripts impératifs ? Quelle est la place de PowerShell dans l'écosystème des outils DevOps natifs cloud ? L'automatisation peut-elle devenir trop complexe, et si oui, comment maintenir la simplicité opérationnelle ?
