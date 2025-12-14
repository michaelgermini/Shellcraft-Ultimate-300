# Chapitre 90 - Automatisation avancée et orchestration

> "L'automatisation n'est pas qu'un outil : c'est une philosophie qui transforme l'impossible en routine." - DevOps Proverbe

## Introduction : L'orchestre des systèmes

Imaginez-vous chef d'orchestre d'un immense opéra numérique où chaque instrument (serveur, base de données, réseau, application) joue sa partition à la perfection, synchronisé avec tous les autres. L'automatisation avancée et l'orchestration en Bash transforment vos scripts en véritables chefs d'orchestre capables de déployer, configurer, et maintenir des infrastructures complexes à l'échelle industrielle.

Dans ce chapitre, nous allons construire des systèmes d'orchestration complets : pipelines de déploiement automatisés, gestionnaires de configuration sophistiqués, orchestrateurs d'infrastructure, et chaînes d'automatisation qui font de l'impossible une routine quotidienne.

## Section 1 : Pipelines de déploiement automatisé

### 1.1 Orchestrateur de pipelines

Système complet de pipelines CI/CD en Bash :

```bash
#!/bin/bash

# Orchestrateur de pipelines
echo "=== Orchestrateur de pipelines ==="

# Pipeline Orchestrator
PipelineOrchestrator() {
    local self="$1"
    
    declare -a $self._pipelines
    declare -A $self._pipeline_configs
    declare -A $self._pipeline_status
    
    # Définition d'un pipeline
    $self.define_pipeline() {
        local name="$1"
        local trigger="$2"  # manual, webhook, schedule, commit
        local stages="$3"   # Liste des stages séparés par ;
        
        $self._pipelines+=("$name")
        $self._pipeline_configs["${name}_trigger"]="$trigger"
        $self._pipeline_configs["${name}_stages"]="$stages"
        $self._pipeline_configs["${name}_status"]="idle"
        $self._pipeline_configs["${name}_last_run"]="never"
        $self._pipeline_configs["${name}_last_success"]="never"
        
        echo "✓ Pipeline défini: $name (trigger: $trigger)"
    }
    
    # Configuration d'un stage
    $self.configure_stage() {
        local pipeline_name="$1"
        local stage_name="$2"
        local commands="$3"
        local dependencies="${4:-}"
        local timeout="${5:-300}"
        
        $self._pipeline_configs["${pipeline_name}_${stage_name}_commands"]="$commands"
        $self._pipeline_configs["${pipeline_name}_${stage_name}_dependencies"]="$dependencies"
        $self._pipeline_configs["${pipeline_name}_${stage_name}_timeout"]="$timeout"
        
        echo "✓ Stage configuré: $pipeline_name/$stage_name"
    }
    
    # Exécution d'un pipeline
    $self.execute_pipeline() {
        local pipeline_name="$1"
        local trigger_source="${2:-manual}"
        
        echo "=== EXÉCUTION PIPELINE: $pipeline_name ==="
        echo "Déclencheur: $trigger_source"
        echo "Timestamp: $(date)"
        
        # Vérification de l'existence du pipeline
        if ! $self._pipeline_exists "$pipeline_name"; then
            echo "❌ Pipeline inexistant: $pipeline_name"
            return 1
        fi
        
        # Mise à jour du statut
        $self._pipeline_configs["${pipeline_name}_status"]="running"
        $self._pipeline_configs["${pipeline_name}_last_run"]="$(date +%s)"
        
        local stages="${$self._pipeline_configs[${pipeline_name}_stages]}"
        IFS=';' read -ra stage_list <<< "$stages"
        
        declare -A stage_status
        declare -A stage_start_time
        declare -A stage_end_time
        
        local failed_stages=()
        local pipeline_success=true
        
        for stage_name in "${stage_list[@]}"; do
            echo "--- Stage: $stage_name ---"
            
            # Vérification des dépendances
            local dependencies="${$self._pipeline_configs[${pipeline_name}_${stage_name}_dependencies]}"
            if [[ -n "$dependencies" ]]; then
                IFS=',' read -ra deps <<< "$dependencies"
                local deps_ok=true
                
                for dep in "${deps[@]}"; do
                    if [[ "${stage_status[$dep]}" != "success" ]]; then
                        echo "❌ Dépendance '$dep' non satisfaite"
                        deps_ok=false
                        break
                    fi
                done
                
                if [[ "$deps_ok" != "true" ]]; then
                    stage_status["$stage_name"]="skipped"
                    continue
                fi
            fi
            
            # Exécution du stage
            stage_start_time["$stage_name"]="$(date +%s)"
            
            if $self._execute_stage "$pipeline_name" "$stage_name"; then
                stage_status["$stage_name"]="success"
                echo "✓ Stage réussi"
            else
                stage_status["$stage_name"]="failed"
                failed_stages+=("$stage_name")
                pipeline_success=false
                echo "❌ Stage échoué"
                
                # Arrêt sur échec (peut être configuré)
                break
            fi
            
            stage_end_time["$stage_name"]="$(date +%s)"
        done
        
        # Finalisation du pipeline
        local end_status="success"
        if [[ "$pipeline_success" != "true" ]]; then
            end_status="failed"
        fi
        
        $self._pipeline_configs["${pipeline_name}_status"]="$end_status"
        
        if [[ "$end_status" == "success" ]]; then
            $self._pipeline_configs["${pipeline_name}_last_success"]="$(date +%s)"
        fi
        
        # Rapport d'exécution
        $self._generate_pipeline_report "$pipeline_name" stage_status stage_start_time stage_end_time failed_stages
        
        return $(( ! pipeline_success ))
    }
    
    # Exécution d'un stage
    $self._execute_stage() {
        local pipeline_name="$1"
        local stage_name="$2"
        
        local commands="${$self._pipeline_configs[${pipeline_name}_${stage_name}_commands]}"
        local timeout="${$self._pipeline_configs[${pipeline_name}_${stage_name}_timeout]}"
        
        # Création d'un environnement isolé pour le stage
        local stage_dir="/tmp/pipeline_${pipeline_name}_${stage_name}_$$"
        mkdir -p "$stage_dir"
        cd "$stage_dir"
        
        # Variables d'environnement du stage
        export PIPELINE_NAME="$pipeline_name"
        export STAGE_NAME="$stage_name"
        export STAGE_DIR="$stage_dir"
        
        # Parsing et exécution des commandes
        IFS=';' read -ra cmd_list <<< "$commands"
        local cmd_success=true
        
        for cmd in "${cmd_list[@]}"; do
            cmd=$(echo "$cmd" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
            
            if [[ -z "$cmd" ]]; then
                continue
            fi
            
            echo "Exécution: $cmd"
            
            # Exécution avec timeout
            if timeout "$timeout" bash -c "$cmd" 2>&1; then
                echo "  ✓ Commande réussie"
            else
                echo "  ❌ Commande échouée"
                cmd_success=false
                break
            fi
        done
        
        # Collecte des artifacts
        if [[ -n "$(ls -A "$stage_dir" 2>/dev/null)" ]]; then
            local artifact_file="/tmp/${pipeline_name}_${stage_name}_$(date +%Y%m%d_%H%M%S).tar.gz"
            tar -czf "$artifact_file" -C "$stage_dir" . 2>/dev/null
            echo "  📦 Artifact sauvegardé: $artifact_file"
        fi
        
        # Nettoyage
        cd - >/dev/null
        rm -rf "$stage_dir"
        
        return $(( ! cmd_success ))
    }
    
    # Génération de rapport
    $self._generate_pipeline_report() {
        local pipeline_name="$1"
        local -n status_ref="$2"
        local -n start_ref="$3"
        local -n end_ref="$4"
        local -n failed_ref="$5"
        
        local report_file="/tmp/pipeline_report_${pipeline_name}_$(date +%Y%m%d_%H%M%S).txt"
        
        {
            echo "=== RAPPORT D'EXÉCUTION PIPELINE ==="
            echo "Pipeline: $pipeline_name"
            echo "Date: $(date)"
            echo "Statut: ${$self._pipeline_configs[${pipeline_name}_status]}"
            echo
            
            echo "RÉSULTATS PAR STAGE:"
            
            local total_duration=0
            for stage in "${!status_ref[@]}"; do
                local status="${status_ref[$stage]}"
                local start="${start_ref[$stage]}"
                local end="${end_ref[$stage]}"
                local duration=$((end - start))
                total_duration=$((total_duration + duration))
                
                case "$status" in
                    success) echo "✓ $stage - SUCCÈS (${duration}s)" ;;
                    failed) echo "❌ $stage - ÉCHEC (${duration}s)" ;;
                    skipped) echo "⚠️  $stage - IGNORÉ" ;;
                esac
            done
            
            echo
            echo "STATISTIQUES:"
            echo "  Durée totale: ${total_duration}s"
            echo "  Stages exécutés: ${#status_ref[@]}"
            echo "  Stages échoués: ${#failed_ref[@]}"
            
            if (( ${#failed_ref[@]} > 0 )); then
                echo
                echo "STAGES EN ÉCHEC:"
                for failed in "${failed_ref[@]}"; do
                    echo "  - $failed"
                done
            fi
            
        } > "$report_file"
        
        echo "📋 Rapport généré: $report_file"
    }
    
    # Surveillance des déclencheurs
    $self.monitor_triggers() {
        local poll_interval="${1:-60}"
        
        echo "=== SURVEILLANCE DES DÉCLENCHEURS ==="
        echo "Intervalle: ${poll_interval}s"
        
        while true; do
            local current_time="$(date +%s)"
            
            for pipeline_name in "${$self._pipelines[@]}"; do
                local trigger="${$self._pipeline_configs[${pipeline_name}_trigger]}"
                local status="${$self._pipeline_configs[${pipeline_name}_status]}"
                
                if [[ "$status" == "running" ]]; then
                    continue
                fi
                
                case "$trigger" in
                    schedule:*)  # schedule:hourly, schedule:daily, etc.
                        local schedule_type="${trigger#schedule:}"
                        local should_trigger=false
                        
                        case "$schedule_type" in
                            hourly)
                                local last_run="${$self._pipeline_configs[${pipeline_name}_last_run]}"
                                if (( current_time - last_run > 3600 )); then
                                    should_trigger=true
                                fi
                                ;;
                            daily)
                                local last_run="${$self._pipeline_configs[${pipeline_name}_last_run]}"
                                if (( current_time - last_run > 86400 )); then
                                    should_trigger=true
                                fi
                                ;;
                        esac
                        
                        if [[ "$should_trigger" == "true" ]]; then
                            echo "[$(date)] Déclenchement automatique: $pipeline_name ($trigger)"
                            $self.execute_pipeline "$pipeline_name" "scheduled"
                        fi
                        ;;
                        
                    webhook)
                        # Simulation de surveillance webhook
                        if [[ -f "/tmp/webhook_trigger_${pipeline_name}" ]]; then
                            echo "[$(date)] Déclenchement webhook: $pipeline_name"
                            rm -f "/tmp/webhook_trigger_${pipeline_name}"
                            $self.execute_pipeline "$pipeline_name" "webhook"
                        fi
                        ;;
                        
                    commit)
                        # Simulation de surveillance commits
                        if [[ -f "/tmp/git_commit_trigger" ]]; then
                            local last_commit
                            last_commit=$(stat -c%Y "/tmp/git_commit_trigger" 2>/dev/null || echo "0")
                            local last_check="${$self._pipeline_configs[${pipeline_name}_last_run]}"
                            
                            if (( last_commit > last_check )); then
                                echo "[$(date)] Déclenchement commit: $pipeline_name"
                                $self.execute_pipeline "$pipeline_name" "commit"
                            fi
                        fi
                        ;;
                esac
            done
            
            sleep "$poll_interval"
        done
    }
    
    # Liste des pipelines
    $self.list_pipelines() {
        echo "=== PIPELINES CONFIGURÉS ==="
        
        for pipeline_name in "${$self._pipelines[@]}"; do
            local trigger="${$self._pipeline_configs[${pipeline_name}_trigger]}"
            local status="${$self._pipeline_configs[${pipeline_name}_status]}"
            local last_run="${$self._pipeline_configs[${pipeline_name}_last_run]}"
            local last_success="${$self._pipeline_configs[${pipeline_name}_last_success]}"
            
            echo "Pipeline: $pipeline_name"
            echo "  Trigger: $trigger"
            echo "  Status: $status"
            echo "  Last run: $(date -d "@$last_run" '+%Y-%m-%d %H:%M:%S' 2>/dev/null || echo 'never')"
            echo "  Last success: $(date -d "@$last_success" '+%Y-%m-%d %H:%M:%S' 2>/dev/null || echo 'never')"
            
            local stages="${$self._pipeline_configs[${pipeline_name}_stages]}"
            echo "  Stages: ${stages//;/, }"
            echo
        done
    }
    
    # Fonctions utilitaires
    $self._pipeline_exists() {
        local pipeline_name="$1"
        
        for existing in "${$self._pipelines[@]}"; do
            if [[ "$existing" == "$pipeline_name" ]]; then
                return 0
            fi
        done
        return 1
    }
}

# Démonstration de l'orchestrateur de pipelines
echo "--- Orchestrateur de pipelines ---"

PipelineOrchestrator "orchestrator"

# Définition des pipelines
orchestrator.define_pipeline "build_app" "commit" "lint;test;build;package"
orchestrator.define_pipeline "deploy_prod" "manual" "backup;deploy;smoke_test;cleanup"
orchestrator.define_pipeline "nightly_backup" "schedule:daily" "backup;verify;report"

# Configuration des stages pour build_app
orchestrator.configure_stage "build_app" "lint" "echo 'Vérification syntaxe'; sleep 1" "" 60
orchestrator.configure_stage "build_app" "test" "echo 'Exécution tests'; sleep 2" "lint" 120
orchestrator.configure_stage "build_app" "build" "echo 'Construction application'; sleep 3" "test" 300
orchestrator.configure_stage "build_app" "package" "echo 'Création package'; sleep 1" "build" 60

# Configuration des stages pour deploy_prod
orchestrator.configure_stage "deploy_prod" "backup" "echo 'Sauvegarde base'; sleep 2" "" 180
orchestrator.configure_stage "deploy_prod" "deploy" "echo 'Déploiement application'; sleep 5" "backup" 600
orchestrator.configure_stage "deploy_prod" "smoke_test" "echo 'Tests de fumée'; sleep 3" "deploy" 120
orchestrator.configure_stage "deploy_prod" "cleanup" "echo 'Nettoyage'; sleep 1" "smoke_test" 60

# Liste des pipelines
orchestrator.list_pipelines

echo
echo "--- Exécution de pipeline ---"
orchestrator.execute_pipeline "build_app" "manual"

echo
echo "--- Simulation de déclencheur webhook ---"
touch "/tmp/webhook_trigger_deploy_prod"

echo
echo "--- Exécution déclenchée ---"
orchestrator.execute_pipeline "deploy_prod" "webhook"

# Nettoyage
rm -f /tmp/pipeline_report_*.txt /tmp/webhook_trigger_deploy_prod
```

### 1.2 Gestionnaire de configuration avancé

Système de gestion de configuration avec versioning et rollback :

```bash
#!/bin/bash

# Gestionnaire de configuration avancé
echo "=== Gestionnaire de configuration avancé ==="

# Configuration Manager
ConfigurationManager() {
    local self="$1"
    
    declare -A $self._configurations
    declare -a $self._config_history
    declare -A $self._config_versions
    
    # Définition d'une configuration
    $self.define_configuration() {
        local name="$1"
        local type="$2"  # file, env, service, network
        local target="$3"
        local content="$4"
        
        $self._configurations["${name}_type"]="$type"
        $self._configurations["${name}_target"]="$target"
        $self._configurations["${name}_content"]="$content"
        $self._configurations["${name}_version"]="1"
        $self._configurations["${name}_last_modified"]="$(date +%s)"
        
        echo "✓ Configuration définie: $name ($type)"
    }
    
    # Application d'une configuration
    $self.apply_configuration() {
        local name="$1"
        local backup="${2:-true}"
        
        if ! $self._config_exists "$name"; then
            echo "❌ Configuration inexistante: $name"
            return 1
        fi
        
        local type="${$self._configurations[${name}_type]}"
        local target="${$self._configurations[${name}_target]}"
        local content="${$self._configurations[${name}_content]}"
        
        echo "--- Application configuration: $name ---"
        echo "Type: $type"
        echo "Cible: $target"
        
        # Sauvegarde si demandée
        if [[ "$backup" == "true" ]]; then
            $self._backup_configuration "$name"
        fi
        
        # Application selon le type
        case "$type" in
            file)
                $self._apply_file_config "$target" "$content"
                ;;
            env)
                $self._apply_env_config "$target" "$content"
                ;;
            service)
                $self._apply_service_config "$target" "$content"
                ;;
            network)
                $self._apply_network_config "$target" "$content"
                ;;
            *)
                echo "❌ Type de configuration inconnu: $type"
                return 1
                ;;
        esac
        
        # Mise à jour du versioning
        local current_version="${$self._configurations[${name}_version]}"
        local new_version=$((current_version + 1))
        $self._configurations["${name}_version"]="$new_version"
        $self._configurations["${name}_last_modified"]="$(date +%s)"
        
        # Historique
        $self._add_to_history "$name" "$new_version" "applied"
        
        echo "✓ Configuration appliquée (version $new_version)"
    }
    
    # Application configuration fichier
    $self._apply_file_config() {
        local target="$1"
        local content="$2"
        
        # Création du répertoire parent
        mkdir -p "$(dirname "$target")"
        
        # Écriture du contenu
        echo "$content" > "$target"
        echo "Fichier créé/modifié: $target"
    }
    
    # Application configuration environnement
    $self._apply_env_config() {
        local target="$1"
        local content="$2"
        
        # Parsing des variables (format VAR=value)
        while IFS='=' read -r var value; do
            if [[ -n "$var" ]]; then
                export "$var=$value"
                echo "Variable exportée: $var=$value"
            fi
        done <<< "$content"
    }
    
    # Application configuration service
    $self._apply_service_config() {
        local target="$1"
        local content="$2"
        
        local service_file="/etc/systemd/system/${target}.service"
        
        echo "$content" | sudo tee "$service_file" >/dev/null
        sudo systemctl daemon-reload
        echo "Service configuré: $target"
    }
    
    # Application configuration réseau
    $self._apply_network_config() {
        local target="$1"
        local content="$2"
        
        local net_file="/etc/network/interfaces.d/${target}"
        
        echo "$content" | sudo tee "$net_file" >/dev/null
        echo "Configuration réseau appliquée: $target"
    }
    
    # Sauvegarde d'une configuration
    $self._backup_configuration() {
        local name="$1"
        
        local type="${$self._configurations[${name}_type]}"
        local target="${$self._configurations[${name}_target]}"
        local version="${$self._configurations[${name}_version]}"
        
        local backup_name="${name}_v${version}_$(date +%Y%m%d_%H%M%S)"
        
        case "$type" in
            file)
                if [[ -f "$target" ]]; then
                    cp "$target" "/tmp/${backup_name}.backup"
                    echo "Sauvegarde créée: /tmp/${backup_name}.backup"
                fi
                ;;
            env)
                # Sauvegarde des variables d'environnement actuelles
                env | grep -E "^(${target//,/|})=" > "/tmp/${backup_name}.env"
                echo "Sauvegarde env créée: /tmp/${backup_name}.env"
                ;;
        esac
    }
    
    # Rollback d'une configuration
    $self.rollback_configuration() {
        local name="$1"
        local target_version="${2:-}"
        
        echo "--- Rollback configuration: $name ---"
        
        if [[ -z "$target_version" ]]; then
            # Rollback à la version précédente
            local current_version="${$self._configurations[${name}_version]}"
            target_version=$((current_version - 1))
        fi
        
        echo "Version cible: $target_version"
        
        # Recherche de la sauvegarde
        local backup_file=""
        for file in /tmp/${name}_v${target_version}_*.backup; do
            if [[ -f "$file" ]]; then
                backup_file="$file"
                break
            fi
        done
        
        if [[ -z "$backup_file" ]]; then
            echo "❌ Sauvegarde introuvable pour la version $target_version"
            return 1
        fi
        
        # Application du rollback
        local type="${$self._configurations[${name}_type]}"
        local target="${$self._configurations[${name}_target]}"
        
        case "$type" in
            file)
                cp "$backup_file" "$target"
                echo "✓ Rollback fichier effectué: $target"
                ;;
            env)
                # Rechargement des variables depuis la sauvegarde
                while IFS='=' read -r var value; do
                    export "$var=$value"
                done < "$backup_file"
                echo "✓ Rollback environnement effectué"
                ;;
        esac
        
        # Mise à jour du versioning
        $self._configurations["${name}_version"]="$target_version"
        $self._add_to_history "$name" "$target_version" "rolled_back"
        
        echo "✓ Rollback terminé (version $target_version)"
    }
    
    # Validation d'une configuration
    $self.validate_configuration() {
        local name="$1"
        
        echo "--- Validation configuration: $name ---"
        
        if ! $self._config_exists "$name"; then
            echo "❌ Configuration inexistante"
            return 1
        fi
        
        local type="${$self._configurations[${name}_type]}"
        local target="${$self._configurations[${name}_target]}"
        local content="${$self._configurations[${name}_content]}"
        
        local errors=()
        
        case "$type" in
            file)
                # Validation syntaxique selon l'extension
                case "$target" in
                    *.json)
                        if ! echo "$content" | python3 -m json.tool >/dev/null 2>&1; then
                            errors+=("JSON invalide")
                        fi
                        ;;
                    *.yaml|*.yml)
                        if command -v python3 >/dev/null && python3 -c "import yaml; yaml.safe_load('''$content''')" >/dev/null 2>&1; then
                            : # OK
                        else
                            errors+=("YAML invalide")
                        fi
                        ;;
                    *.sh)
                        if ! bash -n <(echo "$content") 2>/dev/null; then
                            errors+=("Script Bash invalide")
                        fi
                        ;;
                esac
                ;;
                
            env)
                # Validation des variables d'environnement
                while IFS='=' read -r var value; do
                    if [[ -n "$var" ]]; then
                        if ! [[ "$var" =~ ^[A-Z_][A-Z0-9_]*$ ]]; then
                            errors+=("Nom de variable invalide: $var")
                        fi
                    fi
                done <<< "$content"
                ;;
        esac
        
        if (( ${#errors[@]} > 0 )); then
            echo "❌ Erreurs de validation:"
            for error in "${errors[@]}"; do
                echo "  - $error"
            done
            return 1
        else
            echo "✓ Configuration valide"
            return 0
        fi
    }
    
    # Historique des configurations
    $self.show_history() {
        echo "=== HISTORIQUE DES CONFIGURATIONS ==="
        
        for entry in "${$self._config_history[@]}"; do
            echo "$entry"
        done
    }
    
    # Fonctions utilitaires
    $self._config_exists() {
        local name="$1"
        
        if [[ -n "${$self._configurations[${name}_type]}" ]]; then
            return 0
        fi
        return 1
    }
    
    $self._add_to_history() {
        local name="$1"
        local version="$2"
        local action="$3"
        
        local timestamp="$(date +%s)"
        local entry="[$(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S')] $name v$version - $action"
        
        $self._config_history+=("$entry")
    }
    
    # Liste des configurations
    $self.list_configurations() {
        echo "=== CONFIGURATIONS DÉFINIES ==="
        
        for name in $(echo "${!$self._configurations[@]}" | tr ' ' '\n' | grep '_type$' | sed 's/_type$//'); do
            local type="${$self._configurations[${name}_type]}"
            local target="${$self._configurations[${name}_target]}"
            local version="${$self._configurations[${name}_version]}"
            
            echo "Configuration: $name"
            echo "  Type: $type"
            echo "  Cible: $target"
            echo "  Version: $version"
            echo
        done
    }
}

# Démonstration du gestionnaire de configuration
echo "--- Gestionnaire de configuration avancé ---"

ConfigurationManager "config_mgr"

# Définition des configurations
config_mgr.define_configuration "app_config" "file" "/tmp/app.conf" '{
  "app_name": "MyApp",
  "version": "1.2.3",
  "port": 8080,
  "debug": false
}'

config_mgr.define_configuration "env_vars" "env" "APP_NAME,DEBUG,PORT" 'APP_NAME=MyApp
DEBUG=false
PORT=8080'

config_mgr.define_configuration "nginx_service" "service" "myapp" '[Unit]
Description=MyApp Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/myapp/app.py
Restart=always

[Install]
WantedBy=multi-user.target'

# Liste des configurations
config_mgr.list_configurations

echo
echo "--- Validation des configurations ---"
config_mgr.validate_configuration "app_config"
config_mgr.validate_configuration "env_vars"

echo
echo "--- Application des configurations ---"
config_mgr.apply_configuration "app_config"
config_mgr.apply_configuration "env_vars"

echo
echo "--- Contenu du fichier configuré ---"
cat /tmp/app.conf

echo
echo "--- Variables d'environnement ---"
echo "APP_NAME=$APP_NAME"
echo "DEBUG=$DEBUG"
echo "PORT=$PORT"

echo
echo "--- Rollback ---"
config_mgr.rollback_configuration "app_config"

echo
echo "--- Historique ---"
config_mgr.show_history

# Nettoyage
rm -f /tmp/app.conf /tmp/*backup /tmp/*env
unset APP_NAME DEBUG PORT
```

## Section 2 : Orchestration d'infrastructure

### 2.1 Gestionnaire d'infrastructure as Code

Système complet d'IaC en Bash :

```bash
#!/bin/bash

# Gestionnaire d'infrastructure as Code
echo "=== Gestionnaire d'infrastructure as Code ==="

# Infrastructure as Code Manager
InfrastructureManager() {
    local self="$1"
    
    declare -a $self._resources
    declare -A $self._resource_definitions
    declare -A $self._resource_states
    declare -A $self._providers
    
    # Enregistrement d'un provider
    $self.register_provider() {
        local provider_name="$1"
        local provider_function="$2"
        
        $self._providers["$provider_name"]="$provider_function"
        echo "✓ Provider enregistré: $provider_name"
    }
    
    # Définition d'une ressource
    $self.define_resource() {
        local resource_id="$1"
        local provider="$2"
        local type="$3"
        local properties="$4"
        
        $self._resources+=("$resource_id")
        $self._resource_definitions["${resource_id}_provider"]="$provider"
        $self._resource_definitions["${resource_id}_type"]="$type"
        $self._resource_definitions["${resource_id}_properties"]="$properties"
        $self._resource_states["${resource_id}_state"]="defined"
        
        echo "✓ Ressource définie: $resource_id ($provider:$type)"
    }
    
    # Planification des changements
    $self.plan() {
        echo "=== PLANIFICATION DES CHANGEMENTS ==="
        
        local changes=()
        
        for resource_id in "${$self._resources[@]}"; do
            local current_state="${$self._resource_states[${resource_id}_state]}"
            local desired_state="present"  # Par défaut, toutes les ressources définies doivent être présentes
            
            if [[ "$current_state" != "$desired_state" ]]; then
                changes+=("$resource_id: $current_state -> $desired_state")
            fi
        done
        
        if (( ${#changes[@]} == 0 )); then
            echo "✓ Infrastructure à jour - aucun changement nécessaire"
            return 0
        else
            echo "Changements planifiés:"
            for change in "${changes[@]}"; do
                echo "  - $change"
            done
            echo
            echo "Total: ${#changes[@]} changement(s)"
            return 1
        fi
    }
    
    # Application des changements
    $self.apply() {
        echo "=== APPLICATION DES CHANGEMENTS ==="
        
        local applied=0
        local failed=0
        
        for resource_id in "${$self._resources[@]}"; do
            local current_state="${$self._resource_states[${resource_id}_state]}"
            
            if [[ "$current_state" != "present" ]]; then
                echo "--- Application: $resource_id ---"
                
                if $self._apply_resource "$resource_id"; then
                    $self._resource_states["${resource_id}_state"]="present"
                    ((applied++))
                    echo "✓ Appliqué"
                else
                    ((failed++))
                    echo "❌ Échec"
                fi
            fi
        done
        
        echo
        echo "Résumé: $applied appliqués, $failed échoués"
        
        return $(( failed > 0 ))
    }
    
    # Application d'une ressource
    $self._apply_resource() {
        local resource_id="$1"
        
        local provider="${$self._resource_definitions[${resource_id}_provider]}"
        local type="${$self._resource_definitions[${resource_id}_type]}"
        local properties="${$self._resource_definitions[${resource_id}_properties]}"
        
        local provider_function="${$self._providers[$provider]}"
        
        if [[ -z "$provider_function" ]]; then
            echo "Provider inconnu: $provider"
            return 1
        fi
        
        # Appel du provider
        $provider_function "apply" "$resource_id" "$type" "$properties"
    }
    
    # Destruction d'une ressource
    $self.destroy_resource() {
        local resource_id="$1"
        
        echo "--- Destruction: $resource_id ---"
        
        if $self._apply_resource "$resource_id" "destroy"; then
            $self._resource_states["${resource_id}_state"]="destroyed"
            echo "✓ Détruit"
            return 0
        else
            echo "❌ Échec destruction"
            return 1
        fi
    }
    
    # État de l'infrastructure
    $self.show_state() {
        echo "=== ÉTAT DE L'INFRASTRUCTURE ==="
        
        for resource_id in "${$self._resources[@]}"; do
            local provider="${$self._resource_definitions[${resource_id}_provider]}"
            local type="${$self._resource_definitions[${resource_id}_type]}"
            local state="${$self._resource_states[${resource_id}_state]}"
            
            echo "$resource_id ($provider:$type): $state"
        done
    }
    
    # Validation de l'infrastructure
    $self.validate() {
        echo "=== VALIDATION DE L'INFRASTRUCTURE ==="
        
        local errors=0
        
        for resource_id in "${$self._resources[@]}"; do
            local state="${$self._resource_states[${resource_id}_state]}"
            
            if [[ "$state" != "present" ]]; then
                echo "❌ Ressource non présente: $resource_id"
                ((errors++))
            fi
        done
        
        if (( errors == 0 )); then
            echo "✓ Infrastructure valide"
            return 0
        else
            echo "❌ $errors problème(s) détecté(s)"
            return 1
        fi
    }
}

# Provider pour les ressources locales (fichiers, services, etc.)
local_provider() {
    local action="$1"
    local resource_id="$2"
    local type="$3"
    local properties="$4"
    
    case "$action" in
        apply)
            case "$type" in
                file)
                    local path content mode owner
                    path=$(echo "$properties" | grep -o 'path=[^,]*' | cut -d= -f2)
                    content=$(echo "$properties" | grep -o 'content=[^,]*' | cut -d= -f2-)
                    mode=$(echo "$properties" | grep -o 'mode=[^,]*' | cut -d= -f2)
                    owner=$(echo "$properties" | grep -o 'owner=[^,]*' | cut -d= -f2)
                    
                    mkdir -p "$(dirname "$path")"
                    echo "$content" > "$path"
                    
                    if [[ -n "$mode" ]]; then
                        chmod "$mode" "$path"
                    fi
                    
                    if [[ -n "$owner" ]]; then
                        chown "$owner" "$path"
                    fi
                    
                    echo "Fichier créé: $path"
                    ;;
                    
                directory)
                    local path mode owner
                    path=$(echo "$properties" | grep -o 'path=[^,]*' | cut -d= -f2)
                    mode=$(echo "$properties" | grep -o 'mode=[^,]*' | cut -d= -f2)
                    owner=$(echo "$properties" | grep -o 'owner=[^,]*' | cut -d= -f2)
                    
                    mkdir -p "$path"
                    
                    if [[ -n "$mode" ]]; then
                        chmod "$mode" "$path"
                    fi
                    
                    if [[ -n "$owner" ]]; then
                        chown "$owner" "$path"
                    fi
                    
                    echo "Répertoire créé: $path"
                    ;;
                    
                service)
                    local name state
                    name=$(echo "$properties" | grep -o 'name=[^,]*' | cut -d= -f2)
                    state=$(echo "$properties" | grep -o 'state=[^,]*' | cut -d= -f2)
                    
                    case "$state" in
                        started)
                            echo "Service à démarrer: $name"
                            # systemctl start "$name"  # Simulation
                            ;;
                        stopped)
                            echo "Service à arrêter: $name"
                            # systemctl stop "$name"   # Simulation
                            ;;
                        enabled)
                            echo "Service à activer: $name"
                            # systemctl enable "$name" # Simulation
                            ;;
                    esac
                    ;;
                    
                user)
                    local name uid gid home shell
                    name=$(echo "$properties" | grep -o 'name=[^,]*' | cut -d= -f2)
                    uid=$(echo "$properties" | grep -o 'uid=[^,]*' | cut -d= -f2)
                    gid=$(echo "$properties" | grep -o 'gid=[^,]*' | cut -d= -f2)
                    home=$(echo "$properties" | grep -o 'home=[^,]*' | cut -d= -f2)
                    shell=$(echo "$properties" | grep -o 'shell=[^,]*' | cut -d= -f2)
                    
                    echo "Utilisateur à créer: $name"
                    echo "  UID: ${uid:-auto}"
                    echo "  GID: ${gid:-auto}"
                    echo "  Home: ${home:-/home/$name}"
                    echo "  Shell: ${shell:-/bin/bash}"
                    
                    # useradd -u "$uid" -g "$gid" -d "$home" -s "$shell" "$name"  # Simulation
                    ;;
                    
                *)
                    echo "Type de ressource inconnu: $type"
                    return 1
                    ;;
            esac
            ;;
            
        destroy)
            case "$type" in
                file)
                    local path
                    path=$(echo "$properties" | grep -o 'path=[^,]*' | cut -d= -f2)
                    rm -f "$path"
                    echo "Fichier supprimé: $path"
                    ;;
                    
                directory)
                    local path
                    path=$(echo "$properties" | grep -o 'path=[^,]*' | cut -d= -f2)
                    rmdir "$path" 2>/dev/null || echo "Répertoire non vide: $path"
                    ;;
                    
                service)
                    local name
                    name=$(echo "$properties" | grep -o 'name=[^,]*' | cut -d= -f2)
                    echo "Service à supprimer: $name"
                    # systemctl stop "$name" && systemctl disable "$name"  # Simulation
                    ;;
                    
                user)
                    local name
                    name=$(echo "$properties" | grep -o 'name=[^,]*' | cut -d= -f2)
                    echo "Utilisateur à supprimer: $name"
                    # userdel "$name"  # Simulation
                    ;;
            esac
            ;;
    esac
}

# Provider pour les ressources cloud simulées
cloud_provider() {
    local action="$1"
    local resource_id="$2"
    local type="$3"
    local properties="$4"
    
    case "$action" in
        apply)
            case "$type" in
                vm)
                    local name image size region
                    name=$(echo "$properties" | grep -o 'name=[^,]*' | cut -d= -f2)
                    image=$(echo "$properties" | grep -o 'image=[^,]*' | cut -d= -f2)
                    size=$(echo "$properties" | grep -o 'size=[^,]*' | cut -d= -f2)
                    region=$(echo "$properties" | grep -o 'region=[^,]*' | cut -d= -f2)
                    
                    echo "VM à créer: $name"
                    echo "  Image: $image"
                    echo "  Taille: $size"
                    echo "  Région: $region"
                    
                    # Simulation API cloud
                    echo "API call: POST /v1/vms {name: $name, image: $image, size: $size, region: $region}"
                    ;;
                    
                database)
                    local name engine version size
                    name=$(echo "$properties" | grep -o 'name=[^,]*' | cut -d= -f2)
                    engine=$(echo "$properties" | grep -o 'engine=[^,]*' | cut -d= -f2)
                    version=$(echo "$properties" | grep -o 'version=[^,]*' | cut -d= -f2)
                    size=$(echo "$properties" | grep -o 'size=[^,]*' | cut -d= -f2)
                    
                    echo "Base de données à créer: $name"
                    echo "  Moteur: $engine"
                    echo "  Version: $version"
                    echo "  Taille: $size"
                    
                    # Simulation API cloud
                    echo "API call: POST /v1/databases {name: $name, engine: $engine, version: $version, size: $size}"
                    ;;
                    
                load_balancer)
                    local name ports targets
                    name=$(echo "$properties" | grep -o 'name=[^,]*' | cut -d= -f2)
                    ports=$(echo "$properties" | grep -o 'ports=[^,]*' | cut -d= -f2)
                    targets=$(echo "$properties" | grep -o 'targets=[^,]*' | cut -d= -f2-)
                    
                    echo "Load balancer à créer: $name"
                    echo "  Ports: $ports"
                    echo "  Cibles: $targets"
                    
                    # Simulation API cloud
                    echo "API call: POST /v1/loadbalancers {name: $name, ports: $ports, targets: $targets}"
                    ;;
                    
                *)
                    echo "Type de ressource cloud inconnu: $type"
                    return 1
                    ;;
            esac
            ;;
            
        destroy)
            echo "Destruction cloud simulée: $resource_id ($type)"
            ;;
    esac
}

# Démonstration de l'IaC
echo "--- Infrastructure as Code ---"

InfrastructureManager "iac"

# Enregistrement des providers
iac.register_provider "local" "local_provider"
iac.register_provider "aws" "cloud_provider"

# Définition de l'infrastructure
iac.define_resource "app_dir" "local" "directory" "path=/tmp/myapp,mode=755,owner=root"
iac.define_resource "config_file" "local" "file" "path=/tmp/myapp/config.json,content={\"app\":\"myapp\",\"version\":\"1.0\"},mode=644,owner=root"
iac.define_resource "app_user" "local" "user" "name=myapp,uid=1001,gid=1001,home=/home/myapp,shell=/bin/bash"
iac.define_resource "nginx_service" "local" "service" "name=nginx,state=started"

iac.define_resource "web_vm" "aws" "vm" "name=web01,image=ubuntu-20.04,size=t2.micro,region=us-east-1"
iac.define_resource "app_db" "aws" "database" "name=myapp,engine=mysql,version=8.0,size=db.t2.micro"
iac.define_resource "load_balancer" "aws" "load_balancer" "name=myapp-lb,ports=80,443,targets=web01"

# Planification
iac.plan

echo
echo "--- Application ---"
iac.apply

echo
echo "--- État ---"
iac.show_state

echo
echo "--- Validation ---"
iac.validate

# Nettoyage
rm -rf /tmp/myapp
```

### 2.2 Orchestrateur multi-nœuds

Coordination d'opérations sur plusieurs nœuds :

```bash
#!/bin/bash

# Orchestrateur multi-nœuds
echo "=== Orchestrateur multi-nœuds ==="

# Multi-node Orchestrator
MultiNodeOrchestrator() {
    local self="$1"
    
    declare -a $self._nodes
    declare -A $self._node_configs
    declare -A $self._orchestrations
    
    # Ajout d'un nœud
    $self.add_node() {
        local node_id="$1"
        local connection="$2"
        local role="$3"
        local capabilities="$4"
        
        $self._nodes+=("$node_id")
        $self._node_configs["${node_id}_connection"]="$connection"
        $self._node_configs["${node_id}_role"]="$role"
        $self._node_configs["${node_id}_capabilities"]="$capabilities"
        $self._node_configs["${node_id}_status"]="ready"
        
        echo "✓ Nœud ajouté: $node_id (rôle: $role)"
    }
    
    # Définition d'une orchestration
    $self.define_orchestration() {
        local orch_id="$1"
        local description="$2"
        local steps="$3"  # Format JSON simplifié
        
        $self._orchestrations["${orch_id}_description"]="$description"
        $self._orchestrations["${orch_id}_steps"]="$steps"
        $self._orchestrations["${orch_id}_status"]="defined"
        
        echo "✓ Orchestration définie: $orch_id"
    }
    
    # Exécution d'une orchestration
    $self.execute_orchestration() {
        local orch_id="$1"
        
        echo "=== EXÉCUTION ORCHESTRATION: $orch_id ==="
        
        local steps="${$self._orchestrations[${orch_id}_steps]}"
        
        # Parsing simplifié des étapes (format: étape1:{nœuds,commandes};étape2:{...})
        local step_num=1
        local success=true
        
        while IFS=';' read -ra step_defs; do
            for step_def in "${step_defs[@]}"; do
                if [[ "$step_def" =~ ^([^:]+):(.+)$ ]]; then
                    local step_name="${BASH_REMATCH[1]}"
                    local step_config="${BASH_REMATCH[2]}"
                    
                    echo "--- Étape $step_num: $step_name ---"
                    
                    if ! $self._execute_step "$step_name" "$step_config"; then
                        success=false
                        break 2
                    fi
                    
                    ((step_num++))
                fi
            done
        done <<< "$steps"
        
        $self._orchestrations["${orch_id}_status"]=$([[ "$success" == "true" ]] && echo "completed" || echo "failed")
        
        if [[ "$success" == "true" ]]; then
            echo "✓ Orchestration réussie"
            return 0
        else
            echo "❌ Orchestration échouée"
            return 1
        fi
    }
    
    # Exécution d'une étape
    $self._execute_step() {
        local step_name="$1"
        local step_config="$2"
        
        # Parsing de la configuration (format: {nœuds:"node1,node2",commandes:"cmd1;cmd2"})
        local nodes_part commands_part
        
        if [[ "$step_config" =~ nodes:\"([^\"]+)\" ]]; then
            nodes_part="${BASH_REMATCH[1]}"
        fi
        
        if [[ "$step_config" =~ commandes:\"([^\"]+)\" ]]; then
            commands_part="${BASH_REMATCH[1]}"
        fi
        
        # Sélection des nœuds
        local target_nodes=()
        if [[ "$nodes_part" == "all" ]]; then
            target_nodes=("${$self._nodes[@]}")
        elif [[ "$nodes_part" =~ ^role:(.+) ]]; then
            local required_role="${BASH_REMATCH[1]}"
            for node in "${$self._nodes[@]}"; do
                local node_role="${$self._node_configs[${node}_role]}"
                if [[ "$node_role" == "$required_role" ]]; then
                    target_nodes+=("$node")
                fi
            done
        else
            IFS=',' read -ra target_nodes <<< "$nodes_part"
        fi
        
        if (( ${#target_nodes[@]} == 0 )); then
            echo "Aucun nœud cible trouvé"
            return 1
        fi
        
        echo "Nœuds cibles: ${target_nodes[*]}"
        
        # Exécution des commandes sur chaque nœud
        IFS=';' read -ra commands <<< "$commands_part"
        local step_success=true
        
        for node in "${target_nodes[@]}"; do
            echo "Exécution sur $node:"
            
            for cmd in "${commands[@]}"; do
                cmd=$(echo "$cmd" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
                
                if [[ -z "$cmd" ]]; then
                    continue
                fi
                
                echo "  $cmd"
                
                # Simulation d'exécution distante
                local connection="${$self._node_configs[${node}_connection]}"
                
                if [[ "$connection" == "local" ]]; then
                    if eval "$cmd" 2>&1; then
                        echo "    ✓ Succès"
                    else
                        echo "    ❌ Échec"
                        step_success=false
                    fi
                else
                    # Simulation SSH
                    echo "    [SSH simulé vers $connection]"
                    sleep 0.5  # Simulation du délai réseau
                    
                    if (( RANDOM % 10 < 8 )); then  # 80% de succès
                        echo "    ✓ Succès"
                    else
                        echo "    ❌ Échec (simulé)"
                        step_success=false
                    fi
                fi
            done
        done
        
        return $(( ! step_success ))
    }
    
    # Surveillance de l'orchestration
    $self.monitor_orchestration() {
        local orch_id="$1"
        local duration="${2:-60}"
        
        echo "=== SURVEILLANCE ORCHESTRATION: $orch_id ($duration secondes) ==="
        
        local start_time=$(date +%s)
        
        while (( $(date +%s) - start_time < duration )); do
            echo "[$(date +%H:%M:%S)] === État des nœuds ==="
            
            for node in "${$self._nodes[@]}"; do
                local status="${$self._node_configs[${node}_status]}"
                local role="${$self._node_configs[${node}_role]}"
                
                # Simulation de métriques
                local cpu=$((RANDOM % 100))
                local mem=$((RANDOM % 100))
                local load=$((RANDOM % 4))
                
                printf "  %-10s %-8s CPU:%3d%% MEM:%3d%% LOAD:%d\n" "$node" "$status" "$cpu" "$mem" "$load"
            done
            
            local orch_status="${$self._orchestrations[${orch_id}_status]:-unknown}"
            echo "  Orchestration $orch_id: $orch_status"
            
            sleep 5
        done
    }
    
    # Rollback d'orchestration
    $self.rollback_orchestration() {
        local orch_id="$1"
        
        echo "=== ROLLBACK ORCHESTRATION: $orch_id ==="
        
        # Définition des étapes de rollback (inverse de l'orchestration normale)
        local steps="${$self._orchestrations[${orch_id}_steps]}"
        
        # Simulation de rollback
        echo "Étapes de rollback à exécuter:"
        echo "  - Arrêt des services déployés"
        echo "  - Suppression des fichiers créés"
        echo "  - Restauration des configurations"
        echo "  - Redémarrage des services originaux"
        
        # Ici on pourrait implémenter la logique de rollback réelle
        sleep 2
        
        $self._orchestrations["${orch_id}_status"]="rolled_back"
        echo "✓ Rollback terminé"
    }
    
    # État du cluster
    $self.cluster_status() {
        echo "=== ÉTAT DU CLUSTER ==="
        
        echo "Nœuds (${#$self._nodes[@]}):"
        for node in "${$self._nodes[@]}"; do
            local connection="${$self._node_configs[${node}_connection]}"
            local role="${$self._node_configs[${node}_role]}"
            local status="${$self._node_configs[${node}_status]}"
            local capabilities="${$self._node_configs[${node}_capabilities]}"
            
            echo "  $node ($role): $status"
            echo "    Connexion: $connection"
            echo "    Capacités: $capabilities"
        done
        
        echo
        echo "Orchestrations:"
        for orch_key in "${!$self._orchestrations[@]}"; do
            if [[ "$orch_key" =~ _description$ ]]; then
                local orch_id="${orch_key%_description}"
                local description="${$self._orchestrations[$orch_key]}"
                local status="${$self._orchestrations[${orch_id}_status]}"
                
                echo "  $orch_id: $description ($status)"
            fi
        done
    }
}

# Démonstration de l'orchestrateur multi-nœuds
echo "--- Orchestrateur multi-nœuds ---"

MultiNodeOrchestrator "orchestrator"

# Ajout de nœuds
orchestrator.add_node "control01" "local" "control" "deploy,monitor,backup"
orchestrator.add_node "web01" "user@web01.example.com" "web" "serve,proxy"
orchestrator.add_node "web02" "user@web02.example.com" "web" "serve,proxy"
orchestrator.add_node "db01" "user@db01.example.com" "database" "store,replicate"
orchestrator.add_node "cache01" "user@cache01.example.com" "cache" "store,retrieve"

# Définition d'orchestrations
orchestrator.define_orchestration "deploy_app" "Déploiement de l'application web" 'prep:{nœuds:"role:control",commandes:"mkdir -p /tmp/deploy; echo Préparation déploiement"};deploy_web:{nœuds:"role:web",commandes:"echo Déploiement application web; touch /tmp/app_deployed"};deploy_db:{nœuds:"role:database",commandes:"echo Configuration base de données; touch /tmp/db_configured"};deploy_cache:{nœuds:"role:cache",commandes:"echo Configuration cache; touch /tmp/cache_configured"};verify:{nœuds:"all",commandes:"echo Vérification déploiement; ls /tmp/*deployed /tmp/*configured"}'

orchestrator.define_orchestration "maintenance" "Maintenance système" 'backup:{nœuds:"all",commandes:"echo Création sauvegarde; touch /tmp/backup_$(date +%s)"};update:{nœuds:"all",commandes:"echo Mise à jour système; touch /tmp/system_updated"};restart:{nœuds:"role:web,role:cache",commandes:"echo Redémarrage services; touch /tmp/services_restarted"}'

# État du cluster
orchestrator.cluster_status

echo
echo "--- Exécution d'orchestration ---"
orchestrator.execute_orchestration "deploy_app"

echo
echo "--- Surveillance (courte) ---"
orchestrator.monitor_orchestration "deploy_app" 10 &
monitor_pid=$!

# Attente
sleep 5

echo
echo "--- Rollback simulé ---"
orchestrator.rollback_orchestration "deploy_app"

# Arrêt de la surveillance
kill "$monitor_pid" 2>/dev/null

# Nettoyage
rm -f /tmp/*deployed /tmp/*configured /tmp/backup_* /tmp/system_updated /tmp/services_restarted
```

## Conclusion : L'automatisation comme fondation

L'automatisation avancée et l'orchestration en Bash transforment les tâches complexes en routines fiables et reproductibles. Comme une partition musicale parfaitement orchestrée, vos systèmes deviennent des symphonies de coordination où chaque composant joue son rôle au bon moment.

**Points clés à retenir :**

1. **Pipelines CI/CD** : Chaînes d'automatisation complètes avec déclencheurs, étapes, et rapports détaillés
2. **Gestion de configuration** : Systèmes de configuration hiérarchiques avec versioning et rollback
3. **Infrastructure as Code** : Définition déclarative de l'infrastructure avec providers modulaires
4. **Orchestration multi-nœuds** : Coordination d'opérations complexes sur des clusters distribués

Dans le chapitre suivant, nous explorerons les techniques de scripting réseau avancées, pour que vos scripts Bash puissent communiquer, synchroniser, et orchestrer des opérations à travers des réseaux complexes.

---

**Exercice pratique :** Créez un système d'orchestration complet incluant :
- Pipeline CI/CD avec tests automatiques et déploiements
- Gestion de configuration multi-environnements avec rollback
- Infrastructure as Code pour déployer une application web complète
- Orchestration multi-nœuds pour la coordination de services distribués
- Interface de monitoring et tableaux de bord temps réel

**Réflexion :** Comment adapteriez-vous ces techniques d'orchestration pour gérer un environnement hybride mêlant serveurs physiques, cloud public, et conteneurs orchestrés par Kubernetes ?
