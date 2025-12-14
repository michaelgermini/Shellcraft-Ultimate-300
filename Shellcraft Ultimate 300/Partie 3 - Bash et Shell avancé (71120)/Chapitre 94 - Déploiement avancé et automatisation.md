# Chapitre 94 - Déploiement avancé et automatisation

> "Le déploiement n'est pas un événement : c'est un état d'esprit où chaque changement est une opportunité d'amélioration, chaque rollback une leçon apprise, chaque automatisation une victoire sur la répétition." - DevOps Shell Master

## Introduction : L'art du déploiement intelligent

Imaginez-vous comme un grand maître des échecs DevOps : chaque déploiement est un coup stratégiquement calculé, chaque automatisation une pièce qui se positionne parfaitement sur l'échiquier de l'infrastructure. Le déploiement avancé en Bash transcende la simple copie de fichiers - il devient une symphonie d'automatisation où les pipelines CI/CD, la gestion de configuration, et l'orchestration se fondent en une danse élégante et infaillible.

Dans ce chapitre, nous construirons des systèmes de déploiement qui transforment l'artisanat du déploiement manuel en une science de l'automatisation prédictive, capable de déployer, surveiller, et rollback avec la précision d'un horloger suisse.

## Section 1 : Pipelines de déploiement avancés

### 1.1 Framework de pipeline CI/CD en Bash

Système complet de pipelines d'intégration et déploiement continus :

```bash
#!/bin/bash

# Framework de pipeline CI/CD en Bash
echo "=== Framework de pipeline CI/CD ==="

# CI/CD Pipeline Framework
CIPipeline() {
    local self="$1"
    
    declare -A $self._pipeline_stages
    declare -A $self._pipeline_artifacts
    declare -A $self._pipeline_metrics
    declare -A $self._rollback_strategies
    
    # Définition d'un stage de pipeline
    $self.define_pipeline_stage() {
        local stage_name="$1"
        local stage_type="$2"
        local commands="$3"
        local dependencies="${4:-}"
        local timeout="${5:-300}"
        local retries="${6:-0}"
        
        $self._pipeline_stages["${stage_name}_type"]="$stage_type"
        $self._pipeline_stages["${stage_name}_commands"]="$commands"
        $self._pipeline_stages["${stage_name}_dependencies"]="$dependencies"
        $self._pipeline_stages["${stage_name}_timeout"]="$timeout"
        $self._pipeline_stages["${stage_name}_retries"]="$retries"
        $self._pipeline_stages["${stage_name}_status"]="defined"
        
        echo "✓ Stage défini: $stage_name ($stage_type)"
    }
    
    # Définition d'un artefact
    $self.define_artifact() {
        local artifact_name="$1"
        local source_path="$2"
        local destination_pattern="$3"
        local retention_policy="${4:-keep_last_5}"
        
        $self._pipeline_artifacts["${artifact_name}_source"]="$source_path"
        $self._pipeline_artifacts["${artifact_name}_destination"]="$destination_pattern"
        $self._pipeline_artifacts["${artifact_name}_retention"]="$retention_policy"
        
        echo "✓ Artefact défini: $artifact_name"
    }
    
    # Définition d'une stratégie de rollback
    $self.define_rollback_strategy() {
        local strategy_name="$1"
        local rollback_commands="$2"
        local trigger_conditions="$3"
        
        $self._rollback_strategies["${strategy_name}_commands"]="$rollback_commands"
        $self._rollback_strategies["${strategy_name}_conditions"]="$trigger_conditions"
        
        echo "✓ Stratégie rollback définie: $strategy_name"
    }
    
    # Exécution du pipeline
    $self.execute_pipeline() {
        local pipeline_name="${1:-default}"
        local dry_run="${2:-false}"
        
        echo "=== EXÉCUTION PIPELINE: $pipeline_name ==="
        echo "Mode: $([[ "$dry_run" == "true" ]] && echo "SIMULATION" || echo "RÉEL")"
        echo "Début: $(date)"
        echo
        
        local pipeline_start_time
        pipeline_start_time="$(date +%s)"
        
        # Initialisation des métriques
        $self._pipeline_metrics["${pipeline_name}_start_time"]="$pipeline_start_time"
        $self._pipeline_metrics["${pipeline_name}_status"]="running"
        
        # Résolution des dépendances et ordonnancement
        local execution_order
        execution_order="$($self._resolve_dependencies)"
        
        if [[ -z "$execution_order" ]]; then
            echo "❌ Erreur de résolution des dépendances"
            $self._pipeline_metrics["${pipeline_name}_status"]="failed"
            return 1
        fi
        
        echo "Ordre d'exécution déterminé: $execution_order"
        echo
        
        local overall_success=true
        local stage_results=()
        
        # Exécution des stages
        for stage_name in $execution_order; do
            local stage_start_time
            stage_start_time="$(date +%s)"
            
            echo "--- STAGE: $stage_name ---"
            
            if [[ "$dry_run" == "true" ]]; then
                echo "🔍 SIMULATION: $(echo "${$self._pipeline_stages[${stage_name}_commands]}" | head -1)"
                stage_results+=("$stage_name:SIMULATED:0")
                echo "✓ Simulé"
            else
                if $self._execute_stage "$stage_name"; then
                    local stage_duration=$(( $(date +%s) - stage_start_time ))
                    stage_results+=("$stage_name:SUCCESS:$stage_duration")
                    echo "✓ Réussi (${stage_duration}s)"
                else
                    local stage_duration=$(( $(date +%s) - stage_start_time ))
                    stage_results+=("$stage_name:FAILED:$stage_duration")
                    echo "❌ Échec (${stage_duration}s)"
                    overall_success=false
                    
                    # Tentative de rollback automatique
                    $self._attempt_automatic_rollback "$stage_name"
                    break
                fi
            fi
            
            echo
        done
        
        # Finalisation du pipeline
        local pipeline_end_time
        pipeline_end_time="$(date +%s)"
        local pipeline_duration=$(( pipeline_end_time - pipeline_start_time ))
        
        echo "=== RÉSULTATS PIPELINE ==="
        echo "Durée totale: ${pipeline_duration}s"
        echo "Statut: $([[ "$overall_success" == "true" ]] && echo "SUCCÈS" || echo "ÉCHEC")"
        echo
        
        # Résumé détaillé
        echo "Résumé par stage:"
        for result in "${stage_results[@]}"; do
            local stage_name status duration
            IFS=':' read stage_name status duration <<< "$result"
            printf "  %-20s %-10s %ss\n" "$stage_name" "$status" "$duration"
        done
        
        # Mise à jour des métriques finales
        $self._pipeline_metrics["${pipeline_name}_end_time"]="$pipeline_end_time"
        $self._pipeline_metrics["${pipeline_name}_duration"]="$pipeline_duration"
        $self._pipeline_metrics["${pipeline_name}_status"]=$([[ "$overall_success" == "true" ]] && echo "success" || echo "failed")
        
        # Archivage des artefacts si succès
        if [[ "$overall_success" == "true" && "$dry_run" == "false" ]]; then
            $self._archive_artifacts "$pipeline_name"
        fi
        
        return $(( overall_success == false ))
    }
    
    # Résolution des dépendances
    $self._resolve_dependencies() {
        local -A visited pending
        
        # Fonction récursive pour le tri topologique
        resolve_stage() {
            local stage="$1"
            
            if [[ "${visited[$stage]}" == "visiting" ]]; then
                echo "Erreur: Dépendance circulaire détectée" >&2
                return 1
            fi
            
            if [[ "${visited[$stage]}" == "visited" ]]; then
                return 0
            fi
            
            visited["$stage"]="visiting"
            
            local dependencies="${$self._pipeline_stages[${stage}_dependencies]}"
            if [[ -n "$dependencies" ]]; then
                for dep in $dependencies; do
                    if ! resolve_stage "$dep"; then
                        return 1
                    fi
                done
            fi
            
            visited["$stage"]="visited"
            pending["$stage"]="done"
        }
        
        # Résoudre tous les stages
        for stage_key in "${!$self._pipeline_stages[@]}"; do
            if [[ "$stage_key" =~ _type$ ]]; then
                local stage_name="${stage_key%_type}"
                
                if [[ "${visited[$stage_name]}" != "visited" ]]; then
                    if ! resolve_stage "$stage_name"; then
                        return 1
                    fi
                fi
            fi
        done
        
        # Retourner l'ordre d'exécution
        echo "${!pending[@]}"
    }
    
    # Exécution d'un stage
    $self._execute_stage() {
        local stage_name="$1"
        
        local stage_type="${$self._pipeline_stages[${stage_name}_type]}"
        local commands="${$self._pipeline_stages[${stage_name}_commands]}"
        local timeout="${$self._pipeline_stages[${stage_name}_timeout]}"
        local retries="${$self._pipeline_stages[${stage_name}_retries]}"
        
        echo "Type: $stage_type | Timeout: ${timeout}s | Retries: $retries"
        
        local attempt=1
        local max_attempts=$(( retries + 1 ))
        
        while (( attempt <= max_attempts )); do
            echo "Tentative $attempt/$max_attempts"
            
            # Exécution avec timeout
            if timeout "$timeout" bash -c "$commands" 2>&1; then
                $self._pipeline_stages["${stage_name}_status"]="success"
                return 0
            else
                local exit_code=$?
                echo "Échec tentative $attempt (code: $exit_code)"
                
                if (( attempt < max_attempts )); then
                    local backoff=$(( attempt * 5 ))
                    echo "Attente ${backoff}s avant retry..."
                    sleep "$backoff"
                fi
            fi
            
            ((attempt++))
        done
        
        $self._pipeline_stages["${stage_name}_status"]="failed"
        return 1
    }
    
    # Rollback automatique
    $self._attempt_automatic_rollback() {
        local failed_stage="$1"
        
        echo "🔄 TENTATIVE ROLLBACK AUTOMATIQUE"
        
        # Recherche d'une stratégie de rollback appropriée
        for strategy_key in "${!$self._rollback_strategies[@]}"; do
            if [[ "$strategy_key" =~ _conditions$ ]]; then
                local strategy_name="${strategy_key%_conditions}"
                local conditions="${$self._rollback_strategies[$strategy_key]}"
                
                # Évaluation simple des conditions
                if [[ "$conditions" == *"stage_failed"* ]]; then
                    echo "Application stratégie: $strategy_name"
                    
                    local rollback_commands="${$self._rollback_strategies[${strategy_name}_commands]}"
                    
                    if bash -c "$rollback_commands" 2>&1; then
                        echo "✓ Rollback réussi"
                        return 0
                    else
                        echo "❌ Rollback échoué"
                        return 1
                    fi
                fi
            fi
        done
        
        echo "⚠️ Aucune stratégie rollback applicable"
        return 1
    }
    
    # Archivage des artefacts
    $self._archive_artifacts() {
        local pipeline_name="$1"
        
        echo "📦 ARCHIVAGE ARTEFACTS"
        
        local archive_dir="/tmp/pipeline_artifacts_${pipeline_name}_$(date +%s)"
        mkdir -p "$archive_dir"
        
        for artifact_key in "${!$self._pipeline_artifacts[@]}"; do
            if [[ "$artifact_key" =~ _source$ ]]; then
                local artifact_name="${artifact_key%_source}"
                local source_path="${$self._pipeline_artifacts[$artifact_key]}"
                local destination="${$self._pipeline_artifacts[${artifact_name}_destination]}"
                
                if [[ -e "$source_path" ]]; then
                    cp -r "$source_path" "$archive_dir/"
                    echo "✓ $artifact_name archivé"
                else
                    echo "⚠️ Artefact manquant: $artifact_name"
                fi
            fi
        done
        
        # Nettoyage selon la politique de rétention
        $self._apply_retention_policy
        
        echo "Artefacts archivés dans: $archive_dir"
    }
    
    # Application de la politique de rétention
    $self._apply_retention_policy() {
        local artifact_dirs
        artifact_dirs="$(ls -dt /tmp/pipeline_artifacts_* 2>/dev/null | tail -n +6)"
        
        if [[ -n "$artifact_dirs" ]]; then
            echo "Nettoyage anciennes archives:"
            echo "$artifact_dirs" | while read -r old_dir; do
                echo "  Suppression: $(basename "$old_dir")"
                rm -rf "$old_dir"
            done
        fi
    }
    
    # Génération de rapport de pipeline
    $self.generate_pipeline_report() {
        local pipeline_name="$1"
        local output_file="${2:-pipeline_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT D'EXÉCUTION PIPELINE"
            echo "============================"
            echo "Pipeline: $pipeline_name"
            echo "Date: $(date)"
            echo
            
            echo "MÉTRIQUES GLOBALES"
            echo "=================="
            
            local start_time="${$self._pipeline_metrics[${pipeline_name}_start_time]}"
            local end_time="${$self._pipeline_metrics[${pipeline_name}_end_time]}"
            local duration="${$self._pipeline_metrics[${pipeline_name}_duration]}"
            local status="${$self._pipeline_metrics[${pipeline_name}_status]}"
            
            echo "Début: $(date -d "@$start_time" '+%Y-%m-%d %H:%M:%S')"
            echo "Fin: $(date -d "@$end_time" '+%Y-%m-%d %H:%M:%S')"
            echo "Durée: ${duration}s"
            echo "Statut: $status"
            echo
            
            echo "STATUTS DES STAGES"
            echo "=================="
            
            for stage_key in "${!$self._pipeline_stages[@]}"; do
                if [[ "$stage_key" =~ _status$ && "$stage_key" != "${pipeline_name}_status" ]]; then
                    local stage_name="${stage_key%_status}"
                    local stage_status="${$self._pipeline_stages[$stage_key]}"
                    local stage_type="${$self._pipeline_stages[${stage_name}_type]}"
                    
                    printf "%-20s %-10s %s\n" "$stage_name" "$stage_type" "$stage_status"
                fi
            done
            
            echo
            echo "ARTEFACTS"
            echo "========="
            
            for artifact_key in "${!$self._pipeline_artifacts[@]}"; do
                if [[ "$artifact_key" =~ _source$ ]]; then
                    local artifact_name="${artifact_key%_source}"
                    local source="${$self._pipeline_artifacts[$artifact_key]}"
                    
                    echo "$artifact_name: $source"
                fi
            done
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            if [[ "$status" == "failed" ]]; then
                echo "• Analyser les logs des stages échoués"
                echo "• Vérifier les dépendances et prérequis"
                echo "• Considérer l'ajout de tests supplémentaires"
            fi
            
            if (( duration > 600 )); then
                echo "• Pipeline lent - envisager parallélisation ou optimisation"
            fi
            
            local failed_stages=0
            for stage_key in "${!$self._pipeline_stages[@]}"; do
                if [[ "$stage_key" =~ _status$ && "${$self._pipeline_stages[$stage_key]}" == "failed" ]]; then
                    ((failed_stages++))
                fi
            done
            
            if (( failed_stages > 0 )); then
                echo "• $failed_stages stage(s) en échec - rollback automatique appliqué"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Surveillance continue du pipeline
    $self.monitor_pipeline() {
        local pipeline_name="$1"
        local interval="${2:-30}"
        
        echo "=== SURVEILLANCE PIPELINE: $pipeline_name ==="
        echo "Intervalle: ${interval}s"
        echo
        
        local last_status=""
        
        while true; do
            local current_status="${$self._pipeline_metrics[${pipeline_name}_status]}"
            
            if [[ "$current_status" != "$last_status" ]]; then
                echo "[$(date)] Statut: $current_status"
                last_status="$current_status"
                
                if [[ "$current_status" == "success" || "$current_status" == "failed" ]]; then
                    echo "Pipeline terminé - arrêt surveillance"
                    break
                fi
            fi
            
            sleep "$interval"
        done
    }
    
    # Export de configuration pipeline
    $self.export_pipeline_config() {
        local output_file="$1"
        
        {
            echo "# Configuration Pipeline - Exporté le $(date)"
            echo
            echo "# Stages"
            for stage_key in "${!$self._pipeline_stages[@]}"; do
                if [[ "$stage_key" =~ _type$ ]]; then
                    local stage_name="${stage_key%_type}"
                    local stage_type="${$self._pipeline_stages[$stage_key]}"
                    local commands="${$self._pipeline_stages[${stage_name}_commands]}"
                    local dependencies="${$self._pipeline_stages[${stage_name}_dependencies]}"
                    
                    echo "STAGE:$stage_name:$stage_type:$(echo "$commands" | tr '\n' ';'):$dependencies"
                fi
            done
            
            echo
            echo "# Artefacts"
            for artifact_key in "${!$self._pipeline_artifacts[@]}"; do
                if [[ "$artifact_key" =~ _source$ ]]; then
                    local artifact_name="${artifact_key%_source}"
                    local source="${$self._pipeline_artifacts[$artifact_key]}"
                    local destination="${$self._pipeline_artifacts[${artifact_name}_destination]}"
                    
                    echo "ARTIFACT:$artifact_name:$source:$destination"
                fi
            done
            
            echo
            echo "# Stratégies Rollback"
            for strategy_key in "${!$self._rollback_strategies[@]}"; do
                if [[ "$strategy_key" =~ _commands$ ]]; then
                    local strategy_name="${strategy_key%_commands}"
                    local commands="${$self._rollback_strategies[$strategy_key]}"
                    local conditions="${$self._rollback_strategies[${strategy_name}_conditions]}"
                    
                    echo "ROLLBACK:$strategy_name:$(echo "$commands" | tr '\n' ';'):$conditions"
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Configuration exportée: $output_file"
    }
}

# Définition d'un pipeline CI/CD complet
define_complete_pipeline() {
    local pipeline="$1"
    
    # Stages de construction
    $pipeline.define_pipeline_stage "checkout" "source" \
        "echo 'Récupération du code source...'; git clone https://github.com/example/app.git /tmp/app_source 2>/dev/null || mkdir -p /tmp/app_source" \
        "" "60" "1"
    
    $pipeline.define_pipeline_stage "dependencies" "build" \
        "echo 'Installation des dépendances...'; cd /tmp/app_source; echo 'npm install simulé'" \
        "checkout" "120" "2"
    
    $pipeline.define_pipeline_stage "build" "build" \
        "echo 'Construction de l'\''application...'; cd /tmp/app_source; echo 'npm run build simulé'" \
        "dependencies" "180" "1"
    
    $pipeline.define_pipeline_stage "test_unit" "test" \
        "echo 'Tests unitaires...'; cd /tmp/app_source; echo 'Tests réussis (simulation)'" \
        "build" "90" "0"
    
    $pipeline.define_pipeline_stage "test_integration" "test" \
        "echo 'Tests d'\''intégration...'; cd /tmp/app_source; echo 'Tests d'\''intégration réussis'" \
        "test_unit" "120" "1"
    
    $pipeline.define_pipeline_stage "security_scan" "security" \
        "echo 'Scan de sécurité...'; cd /tmp/app_source; echo 'Aucune vulnérabilité détectée'" \
        "build" "60" "0"
    
    $pipeline.define_pipeline_stage "package" "package" \
        "echo 'Empaquetage...'; cd /tmp/app_source; mkdir -p dist; echo 'Application empaquetée' > dist/app.tar.gz" \
        "test_integration security_scan" "60" "1"
    
    $pipeline.define_pipeline_stage "deploy_staging" "deploy" \
        "echo 'Déploiement en staging...'; mkdir -p /tmp/staging; cp /tmp/app_source/dist/app.tar.gz /tmp/staging/" \
        "package" "30" "2"
    
    $pipeline.define_pipeline_stage "smoke_test" "test" \
        "echo 'Tests de fumée...'; ls -la /tmp/staging/app.tar.gz >/dev/null && echo 'Application déployée correctement'" \
        "deploy_staging" "30" "1"
    
    $pipeline.define_pipeline_stage "deploy_production" "deploy" \
        "echo 'Déploiement en production...'; mkdir -p /tmp/production; cp /tmp/staging/app.tar.gz /tmp/production/" \
        "smoke_test" "45" "3"
    
    # Artefacts
    $pipeline.define_artifact "source_code" "/tmp/app_source" "/tmp/artifacts/source"
    $pipeline.define_artifact "built_app" "/tmp/app_source/dist" "/tmp/artifacts/build"
    $pipeline.define_artifact "test_reports" "/tmp/app_source/reports" "/tmp/artifacts/tests"
    
    # Stratégies de rollback
    $pipeline.define_rollback_strategy "staging_rollback" \
        "echo 'Rollback staging...'; rm -rf /tmp/staging/*; echo 'Précédente version restorée'" \
        "stage_failed:deploy_staging"
    
    $pipeline.define_rollback_strategy "production_rollback" \
        "echo 'Rollback production...'; rm -rf /tmp/production/*; cp /tmp/backup_production/* /tmp/production/ 2>/dev/null || echo 'Aucune sauvegarde disponible'" \
        "stage_failed:deploy_production"
}

# Démonstration du framework pipeline CI/CD
echo "--- Framework Pipeline CI/CD ---"

CIPipeline "ci_pipeline"

# Définition du pipeline complet
define_complete_pipeline "ci_pipeline"

echo
echo "--- Exécution du pipeline (simulation) ---"
ci_pipeline.execute_pipeline "demo_pipeline" "true"

echo
echo "--- Exécution réelle du pipeline ---"
ci_pipeline.execute_pipeline "demo_pipeline" "false"

echo
echo "--- Génération de rapport ---"
ci_pipeline.generate_rapport_pipeline "demo_pipeline"

echo
echo "--- Export de configuration ---"
ci_pipeline.export_pipeline_config "/tmp/pipeline_config.txt"

echo "Configuration exportée:"
head -10 /tmp/pipeline_config.txt

# Nettoyage
rm -rf /tmp/app_source /tmp/staging /tmp/production /tmp/artifacts pipeline_report_*.txt /tmp/pipeline_config.txt
```

### 1.2 Système de déploiement blue/green

Déploiement sans interruption avec basculement instantané :

```bash
#!/bin/bash

# Système de déploiement Blue/Green
echo "=== Système de déploiement Blue/Green ==="

# Blue-Green Deployment System
BlueGreenDeployment() {
    local self="$1"
    
    declare -A $self._environments
    declare -A $self._traffic_distribution
    declare -A $self._health_checks
    
    # Configuration d'un environnement
    $self.configure_environment() {
        local env_name="$1"
        local blue_path="$2"
        local green_path="$3"
        local load_balancer="${4:-nginx}"
        
        $self._environments["${env_name}_blue"]="$blue_path"
        $self._environments["${env_name}_green"]="$green_path"
        $self._environments["${env_name}_load_balancer"]="$load_balancer"
        $self._environments["${env_name}_active"]=""
        $self._environments["${env_name}_standby"]=""
        
        # Initialisation avec blue actif
        $self._switch_to_blue "$env_name"
        
        echo "✓ Environnement configuré: $env_name"
    }
    
    # Basculement vers blue
    $self._switch_to_blue() {
        local env_name="$1"
        
        $self._environments["${env_name}_active"]="${$self._environments[${env_name}_blue]}"
        $self._environments["${env_name}_standby"]="${$self._environments[${env_name}_green]}"
        $self._traffic_distribution["${env_name}_blue"]="100"
        $self._traffic_distribution["${env_name}_green"]="0"
    }
    
    # Basculement vers green
    $self._switch_to_green() {
        local env_name="$1"
        
        $self._environments["${env_name}_active"]="${$self._environments[${env_name}_green]}"
        $self._environments["${env_name}_standby"]="${$self._environments[${env_name}_blue]}"
        $self._traffic_distribution["${env_name}_blue"]="0"
        $self._traffic_distribution["${env_name}_green"]="100"
    }
    
    # Déploiement blue/green
    $self.deploy_blue_green() {
        local env_name="$1"
        local new_version_path="$2"
        local health_check_url="${3:-}"
        
        echo "=== DÉPLOIEMENT BLUE/GREEN: $env_name ==="
        echo "Nouvelle version: $new_version_path"
        
        # Détermination de l'environnement standby
        local standby_env="${$self._environments[${env_name}_standby]}"
        
        echo "Déploiement vers l'environnement standby: $standby_env"
        
        # Sauvegarde de l'ancienne version
        local backup_path="${standby_env}.backup.$(date +%s)"
        if [[ -d "$standby_env" ]]; then
            cp -r "$standby_env" "$backup_path"
            echo "✓ Sauvegarde créée: $backup_path"
        fi
        
        # Déploiement de la nouvelle version
        if $self._deploy_to_environment "$standby_env" "$new_version_path"; then
            echo "✓ Déploiement réussi dans $standby_env"
            
            # Tests de santé
            if [[ -n "$health_check_url" ]]; then
                if $self._run_health_checks "$standby_env" "$health_check_url"; then
                    echo "✓ Tests de santé réussis"
                    
                    # Basculement du trafic
                    $self._switch_traffic "$env_name" "$standby_env"
                    
                    # Nettoyage de l'ancienne version
                    $self._cleanup_old_version "$env_name"
                    
                    echo "✅ Déploiement blue/green réussi"
                    return 0
                else
                    echo "❌ Tests de santé échoués - Rollback"
                    
                    # Rollback immédiat
                    $self._rollback_deployment "$env_name" "$backup_path"
                    return 1
                fi
            else
                # Basculement sans tests de santé
                $self._switch_traffic "$env_name" "$standby_env"
                echo "✅ Déploiement réussi (sans tests de santé)"
                return 0
            fi
        else
            echo "❌ Échec du déploiement"
            return 1
        fi
    }
    
    # Déploiement vers un environnement spécifique
    $self._deploy_to_environment() {
        local target_env="$1"
        local source_path="$2"
        
        echo "Déploiement de $source_path vers $target_env"
        
        # Création du répertoire cible
        mkdir -p "$target_env"
        
        # Copie des fichiers (simulation d'un déploiement réel)
        if [[ -d "$source_path" ]]; then
            cp -r "$source_path"/* "$target_env/" 2>/dev/null || true
            echo "✓ Fichiers copiés"
        elif [[ -f "$source_path" ]]; then
            cp "$source_path" "$target_env/"
            echo "✓ Fichier copié"
        else
            echo "❌ Source introuvable: $source_path"
            return 1
        fi
        
        # Simulation de démarrage de services
        echo "Démarrage des services dans $target_env..."
        sleep 1
        
        return 0
    }
    
    # Exécution des tests de santé
    $self._run_health_checks() {
        local env_path="$1"
        local health_url="$2"
        
        echo "Exécution des tests de santé..."
        
        # Test 1: Existence des fichiers critiques
        local critical_files=("index.html" "app.js" "config.json")
        for file in "${critical_files[@]}"; do
            if [[ ! -f "$env_path/$file" ]]; then
                echo "❌ Fichier critique manquant: $file"
                return 1
            fi
        done
        
        # Test 2: Connectivité réseau (simulation)
        if [[ -n "$health_url" ]]; then
            # Simulation d'appel HTTP
            echo "Test de connectivité: $health_url"
            sleep 0.5
            echo "✓ Connectivité OK"
        fi
        
        # Test 3: Performance (simulation)
        echo "Test de performance..."
        sleep 0.5
        echo "✓ Performance OK"
        
        return 0
    }
    
    # Basculement du trafic
    $self._switch_traffic() {
        local env_name="$1"
        local new_active_env="$2"
        
        echo "🔄 Basculement du trafic vers: $new_active_env"
        
        local load_balancer="${$self._environments[${env_name}_load_balancer]}"
        
        case "$load_balancer" in
            nginx)
                $self._update_nginx_config "$env_name" "$new_active_env"
                ;;
                
            haproxy)
                $self._update_haproxy_config "$env_name" "$new_active_env"
                ;;
                
            aws_elb)
                $self._update_aws_elb "$env_name" "$new_active_env"
                ;;
                
            *)
                echo "⚠️ Load balancer non supporté: $load_balancer - basculement simulé"
                ;;
        esac
        
        # Mise à jour de l'état interne
        if [[ "$new_active_env" == "${$self._environments[${env_name}_blue]}" ]]; then
            $self._switch_to_blue "$env_name"
        else
            $self._switch_to_green "$env_name"
        fi
        
        echo "✓ Trafic basculé"
    }
    
    # Mise à jour de la configuration Nginx
    $self._update_nginx_config() {
        local env_name="$1"
        local active_env="$2"
        
        local nginx_config="/tmp/nginx_${env_name}.conf"
        
        cat > "$nginx_config" << EOF
upstream ${env_name}_backend {
    server unix:${active_env}/app.sock;
}

server {
    listen 80;
    server_name ${env_name}.example.com;
    
    location / {
        proxy_pass http://${env_name}_backend;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
        
        echo "✓ Configuration Nginx mise à jour: $nginx_config"
        
        # Rechargement simulé
        echo "Rechargement Nginx..."
        sleep 0.5
    }
    
    # Mise à jour de la configuration HAProxy
    $self._update_haproxy_config() {
        local env_name="$1"
        local active_env="$2"
        
        local haproxy_config="/tmp/haproxy_${env_name}.cfg"
        
        cat > "$haproxy_config" << EOF
frontend ${env_name}_frontend
    bind *:80
    default_backend ${env_name}_backend

backend ${env_name}_backend
    server blue ${$self._environments[${env_name}_blue]}:8080 check
    server green ${$self._environments[${env_name}_green]}:8080 check
    
    # Activer seulement le serveur actif
EOF
        
        echo "✓ Configuration HAProxy mise à jour: $haproxy_config"
    }
    
    # Mise à jour AWS ELB (simulation)
    $self._update_aws_elb() {
        local env_name="$1"
        local active_env="$2"
        
        echo "Mise à jour AWS ELB pour $env_name..."
        echo "Instances actives: $active_env"
        sleep 1
        echo "✓ ELB mis à jour"
    }
    
    # Rollback du déploiement
    $self._rollback_deployment() {
        local env_name="$1"
        local backup_path="$2"
        
        echo "🔄 ROLLBACK: Restauration depuis $backup_path"
        
        local standby_env="${$self._environments[${env_name}_standby]}"
        
        if [[ -d "$backup_path" ]]; then
            rm -rf "$standby_env"/*
            cp -r "$backup_path"/* "$standby_env/" 2>/dev/null || true
            echo "✓ Rollback effectué"
        else
            echo "❌ Aucune sauvegarde disponible pour rollback"
        fi
    }
    
    # Nettoyage des anciennes versions
    $self._cleanup_old_version() {
        local env_name="$1"
        
        local standby_env="${$self._environments[${env_name}_standby]}"
        
        echo "🧹 Nettoyage de l'ancienne version: $standby_env"
        
        # Suppression des anciens backups (garder les 3 derniers)
        local old_backups
        old_backups="$(ls -dt "${standby_env}.backup."* 2>/dev/null | tail -n +4)"
        
        if [[ -n "$old_backups" ]]; then
            echo "$old_backups" | while read -r old_backup; do
                echo "  Suppression: $(basename "$old_backup")"
                rm -rf "$old_backup"
            done
        fi
    }
    
    # Déploiement canary (progressif)
    $self.deploy_canary() {
        local env_name="$1"
        local new_version_path="$2"
        local canary_percentage="${3:-10}"
        local health_check_url="${4:-}"
        
        echo "=== DÉPLOIEMENT CANARY: $env_name ==="
        echo "Pourcentage initial: ${canary_percentage}%"
        
        # Déploiement initial en canary
        $self._deploy_canary_percentage "$env_name" "$new_version_path" "$canary_percentage"
        
        # Surveillance et augmentation progressive
        local current_percentage="$canary_percentage"
        
        while (( current_percentage < 100 )); do
            echo "Trafic canary: ${current_percentage}%"
            
            # Attente et surveillance
            sleep 30
            
            if [[ -n "$health_check_url" ]]; then
                if $self._run_health_checks "${$self._environments[${env_name}_green]}" "$health_check_url"; then
                    echo "✓ Santé canary OK"
                    
                    # Augmentation du trafic
                    local increment=20
                    current_percentage=$(( current_percentage + increment ))
                    (( current_percentage > 100 )) && current_percentage=100
                    
                    $self._adjust_canary_traffic "$env_name" "$current_percentage"
                else
                    echo "❌ Problème détecté - Rollback canary"
                    $self._rollback_canary "$env_name"
                    return 1
                fi
            else
                # Augmentation automatique sans surveillance
                current_percentage=100
                $self._adjust_canary_traffic "$env_name" "$current_percentage"
            fi
        done
        
        # Finalisation du déploiement canary
        $self._finalize_canary_deployment "$env_name"
        
        echo "✅ Déploiement canary réussi"
        return 0
    }
    
    # Déploiement canary avec pourcentage spécifique
    $self._deploy_canary_percentage() {
        local env_name="$1"
        local new_version_path="$2"
        local percentage="$3"
        
        local green_env="${$self._environments[${env_name}_green]}"
        
        $self._deploy_to_environment "$green_env" "$new_version_path"
        $self._adjust_canary_traffic "$env_name" "$percentage"
    }
    
    # Ajustement du trafic canary
    $self._adjust_canary_traffic() {
        local env_name="$1"
        local percentage="$2"
        
        local blue_percentage=$(( 100 - percentage ))
        
        $self._traffic_distribution["${env_name}_blue"]="$blue_percentage"
        $self._traffic_distribution["${env_name}_green"]="$percentage"
        
        echo "Ajustement trafic - Blue: ${blue_percentage}%, Green: ${percentage}%"
        
        # Mise à jour du load balancer
        $self._switch_traffic "$env_name" "canary_mode"
    }
    
    # Rollback canary
    $self._rollback_canary() {
        local env_name="$1"
        
        echo "Rollback canary - Retour à 100% blue"
        $self._adjust_canary_traffic "$env_name" "0"
    }
    
    # Finalisation du déploiement canary
    $self._finalize_canary_deployment() {
        local env_name="$1"
        
        echo "Finalisation déploiement canary - Basculement complet vers green"
        $self._switch_to_green "$env_name"
    }
    
    # Statut des environnements
    $self.show_environment_status() {
        local env_name="$1"
        
        echo "=== STATUT ENVIRONNEMENT: $env_name ==="
        
        local blue_env="${$self._environments[${env_name}_blue]}"
        local green_env="${$self._environments[${env_name}_green]}"
        local active_env="${$self._environments[${env_name}_active]}"
        local load_balancer="${$self._environments[${env_name}_load_balancer]}"
        
        echo "Environnement Blue: $blue_env"
        echo "Environnement Green: $green_env"
        echo "Environnement Actif: $active_env"
        echo "Load Balancer: $load_balancer"
        echo
        
        echo "Distribution du trafic:"
        echo "  Blue: ${$self._traffic_distribution[${env_name}_blue]:-0}%"
        echo "  Green: ${$self._traffic_distribution[${env_name}_green]:-0}%"
        echo
        
        echo "État des environnements:"
        
        if [[ -d "$blue_env" ]]; then
            local blue_files
            blue_files="$(find "$blue_env" -type f | wc -l)"
            echo "  Blue: ✅ Actif ($blue_files fichiers)"
        else
            echo "  Blue: ❌ Vide"
        fi
        
        if [[ -d "$green_env" ]]; then
            local green_files
            green_files="$(find "$green_env" -type f | wc -l)"
            echo "  Green: ✅ Prêt ($green_files fichiers)"
        else
            echo "  Green: ❌ Vide"
        fi
    }
    
    # Génération de rapport de déploiement
    $self.generate_deployment_report() {
        local env_name="$1"
        local output_file="${2:-deployment_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DE DÉPLOIEMENT BLUE/GREEN"
            echo "=================================="
            echo "Environnement: $env_name"
            echo "Date: $(date)"
            echo
            
            echo "CONFIGURATION"
            echo "============="
            
            echo "Blue: ${$self._environments[${env_name}_blue]}"
            echo "Green: ${$self._environments[${env_name}_green]}"
            echo "Load Balancer: ${$self._environments[${env_name}_load_balancer]}"
            echo "Actif: ${$self._environments[${env_name}_active]}"
            echo
            
            echo "TRAFFIC"
            echo "======="
            
            echo "Blue: ${$self._traffic_distribution[${env_name}_blue]:-0}%"
            echo "Green: ${$self._traffic_distribution[${env_name}_green]:-0}%"
            echo
            
            echo "ÉTAT DES ENVIRONNEMENTS"
            echo "======================="
            
            for env_type in "blue" "green"; do
                local env_path="${$self._environments[${env_name}_${env_type}]}"
                
                echo "$env_type:"
                if [[ -d "$env_path" ]]; then
                    echo "  Statut: ✅ Présent"
                    echo "  Fichiers: $(find "$env_path" -type f | wc -l)"
                    echo "  Taille: $(du -sh "$env_path" 2>/dev/null | cut -f1)"
                    
                    if [[ -f "$env_path/VERSION" ]]; then
                        echo "  Version: $(cat "$env_path/VERSION")"
                    fi
                else
                    echo "  Statut: ❌ Absent"
                fi
                echo
            done
            
            echo "RECOMMANDATIONS"
            echo "==============="
            
            local blue_pct="${$self._traffic_distribution[${env_name}_blue]:-0}"
            local green_pct="${$self._traffic_distribution[${env_name}_green]:-0}"
            
            if (( blue_pct == 100 && green_pct == 0 )); then
                echo "• Système en mode blue uniquement - prêt pour déploiement"
            elif (( blue_pct == 0 && green_pct == 100 )); then
                echo "• Système basculé vers green - considérer nettoyage blue"
            elif (( blue_pct > 0 && green_pct > 0 )); then
                echo "• Mode canary actif - surveiller les métriques"
            fi
            
            if [[ ! -d "${$self._environments[${env_name}_blue]}" ]]; then
                echo "• Environnement blue non initialisé"
            fi
            
            if [[ ! -d "${$self._environments[${env_name}_green]}" ]]; then
                echo "• Environnement green non initialisé"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
}

# Démonstration du système blue/green
echo "--- Système Blue/Green ---"

BlueGreenDeployment "bg_deployment"

# Configuration des environnements
bg_deployment.configure_environment "web_app" "/tmp/blue_env" "/tmp/green_env" "nginx"

# Affichage du statut initial
bg_deployment.show_environment_status "web_app"

echo
echo "--- Préparation des versions ---"

# Création des versions d'exemple
mkdir -p /tmp/blue_env /tmp/green_env

# Version blue (actuelle)
cat > /tmp/blue_env/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>App v1.0 (Blue)</title></head>
<body>
    <h1>Application v1.0</h1>
    <p>Environnement: Blue</p>
</body>
</html>
EOF

cat > /tmp/blue_env/app.js << 'EOF'
console.log("Application v1.0 - Blue Environment");
EOF

echo "v1.0" > /tmp/blue_env/VERSION

# Version green (nouvelle)
cat > /tmp/green_env/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>App v2.0 (Green)</title></head>
<body>
    <h1>Application v2.0</h1>
    <p>Environnement: Green</p>
</body>
</html>
EOF

cat > /tmp/green_env/app.js << 'EOF'
console.log("Application v2.0 - Green Environment");
EOF

echo "v2.0" > /tmp/green_env/VERSION

echo "Versions préparées"

echo
echo "--- Déploiement Blue/Green ---"
bg_deployment.deploy_blue_green "web_app" "/tmp/green_env"

echo
echo "--- Statut après déploiement ---"
bg_deployment.show_environment_status "web_app"

echo
echo "--- Déploiement Canary ---"
# Reset pour démonstration
bg_deployment.configure_environment "web_app" "/tmp/blue_env" "/tmp/green_env" "nginx"
bg_deployment.deploy_canary "web_app" "/tmp/green_env" "25"

echo
echo "--- Statut final ---"
bg_deployment.show_environment_status "web_app"

echo
echo "--- Génération de rapport ---"
bg_deployment.generate_deployment_report "web_app"

# Nettoyage
rm -rf /tmp/blue_env /tmp/green_env /tmp/nginx_*.conf deployment_report_*.txt
```

## Section 2 : Automatisation de l'orchestration et de la gestion de configuration

### 2.1 Framework d'orchestration multi-serveurs

Gestion automatisée de déploiements distribués :

```bash
#!/bin/bash

# Framework d'orchestration multi-serveurs
echo "=== Framework d'orchestration multi-serveurs ==="

# Multi-Server Orchestration Framework
MultiServerOrchestration() {
    local self="$1"
    
    declare -A $self._servers
    declare -A $self._server_groups
    declare -A $self._orchestration_tasks
    declare -A $self._execution_results
    
    # Enregistrement d'un serveur
    $self.register_server() {
        local server_id="$1"
        local hostname="$2"
        local ssh_user="${3:-$USER}"
        local ssh_key="${4:-}"
        local tags="${5:-}"
        
        $self._servers["${server_id}_hostname"]="$hostname"
        $self._servers["${server_id}_user"]="$ssh_user"
        $self._servers["${server_id}_key"]="$ssh_key"
        $self._servers["${server_id}_tags"]="$tags"
        $self._servers["${server_id}_status"]="registered"
        
        echo "✓ Serveur enregistré: $server_id ($hostname)"
    }
    
    # Création d'un groupe de serveurs
    $self.create_server_group() {
        local group_name="$1"
        shift
        local -a servers=("$@")
        
        $self._server_groups["${group_name}_servers"]="${servers[*]}"
        echo "✓ Groupe créé: $group_name (${#servers[@]} serveurs)"
    }
    
    # Définition d'une tâche d'orchestration
    $self.define_orchestration_task() {
        local task_name="$1"
        local target="$2"  # server_id ou group_name
        local commands="$3"
        local strategy="${4:-parallel}"  # parallel, sequential, rolling
        local failure_strategy="${5:-stop}"  # stop, continue, ignore
        
        $self._orchestration_tasks["${task_name}_target"]="$target"
        $self._orchestration_tasks["${task_name}_commands"]="$commands"
        $self._orchestration_tasks["${task_name}_strategy"]="$strategy"
        $self._orchestration_tasks["${task_name}_failure_strategy"]="$failure_strategy"
        
        echo "✓ Tâche définie: $task_name (stratégie: $strategy)"
    }
    
    # Exécution d'une tâche d'orchestration
    $self.execute_orchestration_task() {
        local task_name="$1"
        
        echo "=== EXÉCUTION ORCHESTRATION: $task_name ==="
        
        local target="${$self._orchestration_tasks[${task_name}_target]}"
        local commands="${$self._orchestration_tasks[${task_name}_commands]}"
        local strategy="${$self._orchestration_tasks[${task_name}_strategy]}"
        local failure_strategy="${$self._orchestration_tasks[${task_name}_failure_strategy]}"
        
        # Résolution des cibles
        local -a target_servers
        if [[ -n "${$self._server_groups[${target}_servers]}" ]]; then
            # C'est un groupe
            local group_servers="${$self._server_groups[${target}_servers]}"
            target_servers=($group_servers)
        else
            # C'est un serveur individuel
            target_servers=("$target")
        fi
        
        echo "Cibles: ${target_servers[*]}"
        echo "Stratégie: $strategy"
        echo "Gestion d'échec: $failure_strategy"
        echo
        
        local task_start_time
        task_start_time="$(date +%s)"
        
        case "$strategy" in
            parallel)
                $self._execute_parallel "$task_name" "${target_servers[@]}" "$commands" "$failure_strategy"
                ;;
                
            sequential)
                $self._execute_sequential "$task_name" "${target_servers[@]}" "$commands" "$failure_strategy"
                ;;
                
            rolling)
                $self._execute_rolling "$task_name" "${target_servers[@]}" "$commands" "$failure_strategy"
                ;;
                
            *)
                echo "❌ Stratégie inconnue: $strategy"
                return 1
                ;;
        esac
        
        local task_end_time
        task_end_time="$(date +%s)"
        local task_duration=$(( task_end_time - task_start_time ))
        
        $self._orchestration_tasks["${task_name}_duration"]="$task_duration"
        
        echo
        echo "✓ Orchestration terminée (durée: ${task_duration}s)"
    }
    
    # Exécution parallèle
    $self._execute_parallel() {
        local task_name="$1"
        shift
        local -a servers=("$@")
        local commands="$1"
        local failure_strategy="$2"
        
        echo "Exécution parallèle sur ${#servers[@]} serveurs..."
        
        local -a pids=()
        local -A server_results=()
        
        # Lancement des exécutions en parallèle
        for server in "${servers[@]}"; do
            (
                local server_hostname="${$self._servers[${server}_hostname]}"
                local server_user="${$self._servers[${server}_user]}"
                
                echo "[$server] Début d'exécution"
                
                if $self._execute_on_server "$server" "$commands"; then
                    echo "[$server] ✅ Succès"
                    server_results["$server"]="success"
                else
                    echo "[$server] ❌ Échec"
                    server_results["$server"]="failed"
                fi
            ) &
            
            pids+=($!)
        done
        
        # Attente de tous les processus
        local failed_servers=0
        for pid in "${pids[@]}"; do
            wait "$pid" || ((failed_servers++))
        done
        
        # Gestion des échecs selon la stratégie
        if (( failed_servers > 0 )); then
            echo "Échecs détectés: $failed_servers"
            
            case "$failure_strategy" in
                stop)
                    echo "Stratégie: arrêt sur échec"
                    return 1
                    ;;
                    
                continue)
                    echo "Stratégie: continuation malgré les échecs"
                    return 0
                    ;;
                    
                ignore)
                    echo "Stratégie: ignorée des échecs"
                    return 0
                    ;;
            esac
        fi
        
        return 0
    }
    
    # Exécution séquentielle
    $self._execute_sequential() {
        local task_name="$1"
        shift
        local -a servers=("$@")
        local commands="$1"
        local failure_strategy="$2"
        
        echo "Exécution séquentielle sur ${#servers[@]} serveurs..."
        
        for server in "${servers[@]}"; do
            echo "[$server] Exécution..."
            
            if ! $self._execute_on_server "$server" "$commands"; then
                echo "[$server] ❌ Échec"
                
                case "$failure_strategy" in
                    stop)
                        echo "Arrêt sur échec"
                        return 1
                        ;;
                        
                    continue|ignore)
                        echo "Continuation malgré l'échec"
                        ;;
                esac
            else
                echo "[$server] ✅ Succès"
            fi
            
            # Petit délai entre les exécutions
            sleep 0.5
        done
        
        return 0
    }
    
    # Exécution rolling (par vagues)
    $self._execute_rolling() {
        local task_name="$1"
        shift
        local -a servers=("$@")
        local commands="$1"
        local failure_strategy="$2"
        
        local batch_size=2  # Nombre de serveurs par vague
        
        echo "Exécution rolling (lots de $batch_size) sur ${#servers[@]} serveurs..."
        
        for ((i = 0; i < ${#servers[@]}; i += batch_size)); do
            local batch_end=$((i + batch_size - 1))
            (( batch_end >= ${#servers[@]} )) && batch_end=$(( ${#servers[@]} - 1 ))
            
            echo "Vague $((i / batch_size + 1)): serveurs $((i + 1))-$((batch_end + 1))"
            
            # Exécution de la vague
            local -a batch_servers=("${servers[@]:i:batch_size}")
            
            for server in "${batch_servers[@]}"; do
                echo "[$server] Exécution..."
                
                if $self._execute_on_server "$server" "$commands"; then
                    echo "[$server] ✅ Succès"
                else
                    echo "[$server] ❌ Échec"
                    
                    case "$failure_strategy" in
                        stop)
                            echo "Arrêt sur échec"
                            return 1
                            ;;
                    esac
                fi
            done
            
            # Attente avant la prochaine vague (sauf dernière)
            if (( batch_end < ${#servers[@]} - 1 )); then
                echo "Attente avant prochaine vague..."
                sleep 5
            fi
        done
        
        return 0
    }
    
    # Exécution sur un serveur spécifique
    $self._execute_on_server() {
        local server_id="$1"
        local commands="$2"
        
        local hostname="${$self._servers[${server_id}_hostname]}"
        local user="${$self._servers[${server_id}_user]}"
        local key="${$self._servers[${server_id}_key]}"
        
        # Simulation d'exécution SSH (en vrai, utiliser ssh)
        echo "Simulation SSH vers $hostname (user: $user)"
        
        # Simulation de latence réseau
        sleep 0.1
        
        # Exécution des commandes (dans un sous-shell pour l'isolation)
        if bash -c "$commands" 2>&1; then
            $self._servers["${server_id}_status"]="success"
            $self._execution_results["${server_id}_last_execution"]="$(date +%s)"
            return 0
        else
            $self._servers["${server_id}_status"]="failed"
            return 1
        fi
    }
    
    # Test de connectivité des serveurs
    $self.test_server_connectivity() {
        local target="${1:-all}"  # all, group_name, ou server_id
        
        echo "=== TEST CONNECTIVITÉ ==="
        
        local -a servers_to_test
        
        if [[ "$target" == "all" ]]; then
            # Tous les serveurs
            for server_key in "${!$self._servers[@]}"; do
                if [[ "$server_key" =~ _hostname$ ]]; then
                    local server_id="${server_key%_hostname}"
                    servers_to_test+=("$server_id")
                fi
            done
        elif [[ -n "${$self._server_groups[${target}_servers]}" ]]; then
            # Groupe spécifique
            local group_servers="${$self._server_groups[${target}_servers]}"
            servers_to_test=($group_servers)
        else
            # Serveur spécifique
            servers_to_test=("$target")
        fi
        
        local reachable=0
        local unreachable=0
        
        for server_id in "${servers_to_test[@]}"; do
            local hostname="${$self._servers[${server_id}_hostname]}"
            
            echo -n "Test $server_id ($hostname)... "
            
            # Test de ping simulé
            if ping -c1 -W1 "$hostname" >/dev/null 2>&1; then
                echo "✅ Accessible"
                ((reachable++))
                $self._servers["${server_id}_reachable"]="true"
            else
                echo "❌ Inaccessible"
                ((unreachable++))
                $self._servers["${server_id}_reachable"]="false"
            fi
        done
        
        echo
        echo "Résumé connectivité:"
        echo "  Accessibles: $reachable"
        echo "  Inaccessibles: $unreachable"
        
        return $(( unreachable > 0 ))
    }
    
    # Collecte de métriques des serveurs
    $self.collect_server_metrics() {
        local target="${1:-all}"
        
        echo "=== COLLECTE MÉTRIQUES ==="
        
        local -a target_servers
        # Résolution similaire à test_connectivity
        
        for server_key in "${!$self._servers[@]}"; do
            if [[ "$server_key" =~ _hostname$ ]]; then
                local server_id="${server_key%_hostname}"
                
                if [[ "$target" == "all" || "$target" == "$server_id" ]]; then
                    echo "Collecte métriques pour $server_id..."
                    
                    # Métriques simulées
                    $self._servers["${server_id}_cpu_usage"]="$((RANDOM % 100))"
                    $self._servers["${server_id}_memory_usage"]="$((RANDOM % 100))"
                    $self._servers["${server_id}_disk_usage"]="$((RANDOM % 100))"
                    $self._servers["${server_id}_last_metric_collection"]="$(date +%s)"
                    
                    echo "  CPU: ${$self._servers[${server_id}_cpu_usage]}%"
                    echo "  RAM: ${$self._servers[${server_id}_memory_usage]}%"
                    echo "  Disk: ${$self._servers[${server_id}_disk_usage]}%"
                fi
            fi
        done
    }
    
    # Génération de rapport d'orchestration
    $self.generate_orchestration_report() {
        local output_file="${1:-orchestration_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT D'ORCHESTRATION MULTI-SERVEURS"
            echo "======================================"
            echo "Date: $(date)"
            echo
            
            echo "SERVEURS ENREGISTRÉS"
            echo "===================="
            
            for server_key in "${!$self._servers[@]}"; do
                if [[ "$server_key" =~ _hostname$ ]]; then
                    local server_id="${server_key%_hostname}"
                    local hostname="${$self._servers[$server_key]}"
                    local user="${$self._servers[${server_id}_user]}"
                    local status="${$self._servers[${server_id}_status]}"
                    local reachable="${$self._servers[${server_id}_reachable]:-unknown}"
                    
                    echo "Serveur: $server_id"
                    echo "  Hostname: $hostname"
                    echo "  User: $user"
                    echo "  Statut: $status"
                    echo "  Accessible: $reachable"
                    
                    # Métriques si disponibles
                    local cpu="${$self._servers[${server_id}_cpu_usage]:-N/A}"
                    local mem="${$self._servers[${server_id}_memory_usage]:-N/A}"
                    local disk="${$self._servers[${server_id}_disk_usage]:-N/A}"
                    
                    if [[ "$cpu" != "N/A" ]]; then
                        echo "  Métriques:"
                        echo "    CPU: $cpu%"
                        echo "    RAM: $mem%"
                        echo "    Disk: $disk%"
                    fi
                    
                    echo
                fi
            done
            
            echo "GROUPES DE SERVEURS"
            echo "==================="
            
            for group_key in "${!$self._server_groups[@]}"; do
                if [[ "$group_key" =~ _servers$ ]]; then
                    local group_name="${group_key%_servers}"
                    local servers="${$self._server_groups[$group_key]}"
                    
                    echo "Groupe: $group_name"
                    echo "  Serveurs: $servers"
                    echo
                fi
            done
            
            echo "TÂCHES D'ORCHESTRATION"
            echo "======================"
            
            for task_key in "${!$self._orchestration_tasks[@]}"; do
                if [[ "$task_key" =~ _target$ ]]; then
                    local task_name="${task_key%_target}"
                    local target="${$self._orchestration_tasks[$task_key]}"
                    local strategy="${$self._orchestration_tasks[${task_name}_strategy]}"
                    local duration="${$self._orchestration_tasks[${task_name}_duration]:-N/A}"
                    
                    echo "Tâche: $task_name"
                    echo "  Cible: $target"
                    echo "  Stratégie: $strategy"
                    echo "  Durée: ${duration}s"
                    echo
                fi
            done
            
            echo "RECOMMANDATIONS"
            echo "==============="
            
            # Analyse des serveurs inaccessibles
            local unreachable_count=0
            for server_key in "${!$self._servers[@]}"; do
                if [[ "$server_key" =~ _reachable$ && "${$self._servers[$server_key]}" == "false" ]]; then
                    ((unreachable_count++))
                fi
            done
            
            if (( unreachable_count > 0 )); then
                echo "• $unreachable_count serveur(s) inaccessible(s) - vérifier la connectivité"
            fi
            
            # Analyse des métriques élevées
            local high_cpu_count=0
            local high_mem_count=0
            
            for server_key in "${!$self._servers[@]}"; do
                if [[ "$server_key" =~ _cpu_usage$ ]]; then
                    local server_id="${server_key%_cpu_usage}"
                    local cpu="${$self._servers[$server_key]}"
                    local mem="${$self._servers[${server_id}_memory_usage]}"
                    
                    if (( cpu > 80 )); then
                        ((high_cpu_count++))
                    fi
                    
                    if (( mem > 80 )); then
                        ((high_mem_count++))
                    fi
                fi
            done
            
            if (( high_cpu_count > 0 )); then
                echo "• $high_cpu_count serveur(s) avec utilisation CPU élevée (>80%)"
            fi
            
            if (( high_mem_count > 0 )); then
                echo "• $high_mem_count serveur(s) avec utilisation mémoire élevée (>80%)"
            fi
            
            if (( unreachable_count == 0 && high_cpu_count == 0 && high_mem_count == 0 )); then
                echo "• État général satisfaisant - tous les serveurs opérationnels"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Export de configuration d'orchestration
    $self.export_orchestration_config() {
        local output_file="$1"
        
        {
            echo "# Configuration Orchestration - Exporté le $(date)"
            echo
            echo "# Serveurs"
            for server_key in "${!$self._servers[@]}"; do
                if [[ "$server_key" =~ _hostname$ ]]; then
                    local server_id="${server_key%_hostname}"
                    local hostname="${$self._servers[$server_key]}"
                    local user="${$self._servers[${server_id}_user]}"
                    local key="${$self._servers[${server_id}_key]}"
                    local tags="${$self._servers[${server_id}_tags]}"
                    
                    echo "SERVER:$server_id:$hostname:$user:$key:$tags"
                fi
            done
            
            echo
            echo "# Groupes"
            for group_key in "${!$self._server_groups[@]}"; do
                if [[ "$group_key" =~ _servers$ ]]; then
                    local group_name="${group_key%_servers}"
                    local servers="${$self._server_groups[$group_key]}"
                    
                    echo "GROUP:$group_name:$servers"
                fi
            done
            
            echo
            echo "# Tâches"
            for task_key in "${!$self._orchestration_tasks[@]}"; do
                if [[ "$task_key" =~ _target$ ]]; then
                    local task_name="${task_key%_target}"
                    local target="${$self._orchestration_tasks[$task_key]}"
                    local commands="${$self._orchestration_tasks[${task_name}_commands]}"
                    local strategy="${$self._orchestration_tasks[${task_name}_strategy]}"
                    local failure_strategy="${$self._orchestration_tasks[${task_name}_failure_strategy]}"
                    
                    echo "TASK:$task_name:$target:$(echo "$commands" | tr '\n' ';'):$strategy:$failure_strategy"
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Configuration exportée: $output_file"
    }
}

# Démonstration de l'orchestration multi-serveurs
echo "--- Orchestration Multi-Serveurs ---"

MultiServerOrchestration "multi_orchestration"

# Enregistrement des serveurs
multi_orchestration.register_server "web01" "web01.example.com" "deploy" "" "web,frontend"
multi_orchestration.register_server "web02" "web02.example.com" "deploy" "" "web,frontend"
multi_orchestration.register_server "db01" "db01.example.com" "deploy" "" "database,backend"
multi_orchestration.register_server "cache01" "cache01.example.com" "deploy" "" "cache,backend"

# Création des groupes
multi_orchestration.create_server_group "webservers" "web01" "web02"
multi_orchestration.create_server_group "backends" "db01" "cache01"
multi_orchestration.create_server_group "all_servers" "web01" "web02" "db01" "cache01"

# Définition des tâches d'orchestration
multi_orchestration.define_orchestration_task "update_packages" "all_servers" \
    "echo 'Mise à jour des paquets système...'; sleep 0.1" \
    "parallel" "continue"

multi_orchestration.define_orchestration_task "deploy_web_app" "webservers" \
    "echo 'Déploiement application web...'; sleep 0.2" \
    "rolling" "stop"

multi_orchestration.define_orchestration_task "update_database" "db01" \
    "echo 'Migration base de données...'; sleep 0.3" \
    "sequential" "stop"

multi_orchestration.define_orchestration_task "clear_cache" "cache01" \
    "echo 'Purge du cache...'; sleep 0.1" \
    "parallel" "ignore"

echo
echo "--- Test de connectivité ---"
multi_orchestration.test_server_connectivity "all"

echo
echo "--- Collecte de métriques ---"
multi_orchestration.collect_server_metrics "all"

echo
echo "--- Exécution des tâches d'orchestration ---"

echo "1. Mise à jour des paquets:"
multi_orchestration.execute_orchestration_task "update_packages"

echo
echo "2. Déploiement application web:"
multi_orchestration.execute_orchestration_task "deploy_web_app"

echo
echo "3. Migration base de données:"
multi_orchestration.execute_orchestration_task "update_database"

echo
echo "4. Purge du cache:"
multi_orchestration.execute_orchestration_task "clear_cache"

echo
echo "--- Génération de rapport ---"
multi_orchestration.generate_orchestration_report

echo
echo "--- Export de configuration ---"
multi_orchestration.export_orchestration_config "/tmp/orchestration_config.txt"

echo "Configuration exportée:"
head -10 /tmp/orchestration_config.txt

# Nettoyage
rm -f orchestration_report_*.txt /tmp/orchestration_config.txt
```

## Conclusion : L'orchestration comme symphonie

Le déploiement avancé et l'automatisation en Bash transcendent la simple exécution de commandes : ils deviennent une symphonie orchestrale où pipelines CI/CD, déploiement blue/green, et orchestration multi-serveurs se fondent en une danse élégante et infaillible. Cette approche transforme le DevOps d'un ensemble de pratiques en un état d'esprit où chaque automatisation est une victoire sur la répétition, chaque pipeline une œuvre d'art opérationnelle.

**Points clés à retenir :**

1. **Pipelines CI/CD** : Frameworks complets avec stages, dépendances, rollback et surveillance continue
2. **Déploiement Blue/Green** : Basculement sans interruption avec tests automatisés et rollback instantané
3. **Orchestration multi-serveurs** : Gestion automatisée de déploiements distribués avec stratégies parallèles, séquentielles et rolling

Dans le prochain chapitre, nous explorerons les techniques de scripting réseau et d'opérations distribuées, pour que nos systèmes automatisés puissent communiquer et coordonner à travers les réseaux.

---

**Exercice pratique :** Créez un système complet de déploiement incluant :
- Pipeline CI/CD avec tests automatisés et déploiement conditionnel
- Stratégie blue/green avec health checks et rollback automatique
- Orchestration multi-serveurs avec gestion d'échecs et métriques de performance

**Réflexion :** Comment intégreriez-vous ces mécanismes avancés de déploiement avec des outils comme Kubernetes ou Docker Swarm pour créer une plateforme d'orchestration complète en pur Bash ?
