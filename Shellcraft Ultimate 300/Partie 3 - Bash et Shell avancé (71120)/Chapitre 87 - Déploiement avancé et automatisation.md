# Chapitre 87 - Déploiement avancé et automatisation

> "L'automatisation n'est pas qu'un outil : c'est une philosophie qui transforme l'exceptionnel en routine." - Anonyme

## Introduction : De l'artisanat à l'usine logicielle

Imaginez-vous architecte d'une chaîne de production automobile moderne : chaque pièce arrive au bon moment, chaque robot effectue sa tâche avec précision, et l'ensemble produit des véhicules parfaits à un rythme industriel. Le déploiement avancé en Bash, c'est exactement cela : transformer vos scripts en véritables usines logicielles capables de déployer, configurer, et maintenir des infrastructures complexes.

Dans ce chapitre, nous allons explorer les stratégies de déploiement à grande échelle : gestion de configuration, orchestration, déploiement continu, et automatisation infrastructurelle avec Bash comme langage de base.

## Section 1 : Gestion de configuration avancée

### 1.1 Systèmes de templates et variables

Création de configurations dynamiques et maintenables :

```bash
#!/bin/bash

# Gestion de configuration avancée
echo "=== Gestion de configuration avancée ==="

# Moteur de templates avancé
TemplateEngine() {
    local self="$1"
    
    declare -A $self._templates
    declare -A $self._variables
    
    # Enregistrement d'un template
    $self.register_template() {
        local name="$1"
        local template="$2"
        
        $self._templates["$name"]="$template"
        echo "✓ Template '$name' enregistré"
    }
    
    # Définition de variables
    $self.set_variable() {
        local key="$1"
        local value="$2"
        
        $self._variables["$key"]="$value"
    }
    
    # Définition de variables par lot
    $self.set_variables() {
        local vars_string="$1"
        
        # Parsing du format "clé1=valeur1,clé2=valeur2"
        IFS=',' read -ra vars <<< "$vars_string"
        for var in "${vars[@]}"; do
            local key="${var%%=*}"
            local value="${var#*=}"
            $self.set_variable "$key" "$value"
        done
    }
    
    # Rendu d'un template
    $self.render() {
        local template_name="$1"
        local output_file="$2"
        
        local template="${$self._templates[$template_name]}"
        
        if [[ -z "$template" ]]; then
            echo "❌ Template '$template_name' inexistant" >&2
            return 1
        fi
        
        # Rendu des variables
        local rendered="$template"
        
        for key in "${!self._variables[@]}"; do
            local value="${self._variables[$key]}"
            # Échappement pour sed
            local escaped_key
            escaped_key=$(printf '%s\n' "$key" | sed 's/[[\.*^$()+?{|]/\\&/g')
            local escaped_value
            escaped_value=$(printf '%s\n' "$value" | sed 's/[[\.*^$()+?{|]/\\&/g')
            
            rendered=$(echo "$rendered" | sed "s/{{$escaped_key}}/$escaped_value/g")
        done
        
        # Écriture du résultat
        echo "$rendered" > "$output_file"
        echo "✓ Template rendu: $output_file"
    }
    
    # Rendu conditionnel
    $self.render_conditional() {
        local template_name="$1"
        local output_file="$2"
        local conditions="$3"  # Format "var1=value1,var2=value2"
        
        # Application des conditions
        if [[ -n "$conditions" ]]; then
            IFS=',' read -ra conds <<< "$conditions"
            for cond in "${conds[@]}"; do
                local var="${cond%%=*}"
                local val="${cond#*=}"
                $self.set_variable "$var" "$val"
            done
        fi
        
        $self.render "$template_name" "$output_file"
    }
    
    # Validation des variables requises
    $self.validate_template() {
        local template_name="$1"
        
        local template="${$self._templates[$template_name]}"
        local missing_vars=()
        
        # Recherche des variables {{var}} dans le template
        while IFS= read -r line; do
            local vars
            vars=$(echo "$line" | grep -o '{{[^}]*}}' | sed 's/[{}]//g')
            
            for var in $vars; do
                if [[ -z "${$self._variables[$var]}" ]]; then
                    missing_vars+=("$var")
                fi
            done
        done <<< "$template"
        
        if (( ${#missing_vars[@]} > 0 )); then
            echo "❌ Variables manquantes: ${missing_vars[*]}" >&2
            return 1
        fi
        
        echo "✓ Template valide"
        return 0
    }
}

# Configuration multi-environnements
EnvironmentManager() {
    local self="$1"
    
    declare -A $self._environments
    
    # Définition d'un environnement
    $self.define_environment() {
        local name="$1"
        local vars="$2"
        
        $self._environments["$name"]="$vars"
        echo "✓ Environnement '$name' défini"
    }
    
    # Application d'un environnement
    $self.apply_environment() {
        local name="$1"
        
        local vars="${$self._environments[$name]}"
        
        if [[ -z "$vars" ]]; then
            echo "❌ Environnement '$name' inexistant" >&2
            return 1
        fi
        
        echo "Application de l'environnement: $name"
        IFS=',' read -ra env_vars <<< "$vars"
        for var in "${env_vars[@]}"; do
            local key="${var%%=*}"
            local value="${var#*=}"
            export "$key=$value"
            echo "  $key=$value"
        done
    }
    
    # Liste des environnements
    $self.list_environments() {
        echo "Environnements disponibles:"
        for env in "${!$self._environments[@]}"; do
            echo "  $env"
        done
    }
}

# Démonstration du système de templates
echo "--- Système de templates ---"

TemplateEngine "templater"
EnvironmentManager "env_mgr"

# Définition d'environnements
env_mgr.define_environment "development" "APP_ENV=dev,DEBUG=true,DB_HOST=localhost,DB_NAME=dev_db"
env_mgr.define_environment "production" "APP_ENV=prod,DEBUG=false,DB_HOST=prod-db.example.com,DB_NAME=prod_db"
env_mgr.define_environment "staging" "APP_ENV=staging,DEBUG=false,DB_HOST=staging-db.example.com,DB_NAME=staging_db"

# Template d'application
templater.register_template "app_config" '{
  "application": {
    "name": "{{APP_NAME}}",
    "environment": "{{APP_ENV}}",
    "debug": {{DEBUG}},
    "version": "{{VERSION}}"
  },
  "database": {
    "host": "{{DB_HOST}}",
    "name": "{{DB_NAME}}",
    "port": {{DB_PORT}}
  },
  "cache": {
    "enabled": {{CACHE_ENABLED}},
    "ttl": {{CACHE_TTL}}
  }
}'

# Template de déploiement
templater.register_template "deploy_script" '#!/bin/bash
# Script de déploiement généré automatiquement
# Généré le: {{GENERATION_DATE}}
# Pour l environnement: {{APP_ENV}}

set -euo pipefail

echo "=== DÉPLOIEMENT {{APP_NAME}} v{{VERSION}} ==="
echo "Environnement: {{APP_ENV}}"
echo "Serveur: {{DEPLOY_HOST}}"

# Vérifications pré-déploiement
check_prerequisites() {
    echo "Vérification des prérequis..."
    
    # Vérifier l espace disque
    local available_space
    available_space=$(df {{DEPLOY_PATH}} | tail -1 | awk "{print \$4}")
    if (( available_space < 1000000 )); then
        echo "❌ Espace disque insuffisant"
        exit 1
    fi
    
    echo "✓ Prérequis OK"
}

# Sauvegarde
create_backup() {
    echo "Création de la sauvegarde..."
    local backup_name="{{APP_NAME}}_backup_$(date +%Y%m%d_%H%M%S).tar.gz"
    tar -czf "/tmp/$backup_name" -C "{{DEPLOY_PATH}}" .
    echo "✓ Sauvegarde créée: /tmp/$backup_name"
}

# Déploiement
deploy_application() {
    echo "Déploiement de l application..."
    
    # Arrêt du service
    systemctl stop {{SERVICE_NAME}} || true
    
    # Déploiement des fichiers
    rsync -av --delete "{{SOURCE_PATH}}/" "{{DEPLOY_PATH}}/"
    
    # Configuration des permissions
    chown -R {{APP_USER}}:{{APP_GROUP}} "{{DEPLOY_PATH}}"
    chmod -R 755 "{{DEPLOY_PATH}}"
    
    # Démarrage du service
    systemctl start {{SERVICE_NAME}}
    systemctl enable {{SERVICE_NAME}}
    
    echo "✓ Application déployée"
}

# Tests post-déploiement
run_tests() {
    echo "Exécution des tests..."
    
    # Test de santé
    if curl -f -s "{{HEALTH_CHECK_URL}}" > /dev/null; then
        echo "✓ Test de santé réussi"
    else
        echo "❌ Test de santé échoué"
        exit 1
    fi
}

main() {
    check_prerequisites
    create_backup
    deploy_application
    run_tests
    
    echo "🎉 Déploiement réussi!"
}

main "$@"
'

# Application des environnements et rendu
echo "--- Rendu avec environnement development ---"
env_mgr.apply_environment "development"
templater.set_variables "APP_NAME=MyApp,VERSION=1.2.3,DB_PORT=5432,CACHE_ENABLED=true,CACHE_TTL=3600"
templater.render "app_config" "/tmp/app_config_dev.json"

echo "--- Rendu avec environnement production ---"
env_mgr.apply_environment "production"
templater.set_variables "APP_NAME=MyApp,VERSION=1.2.3,DB_PORT=5432,CACHE_ENABLED=true,CACHE_TTL=7200,GENERATION_DATE=$(date),DEPLOY_HOST=prod-server.example.com,SOURCE_PATH=/tmp/source,DEPLOY_PATH=/opt/myapp,SERVICE_NAME=myapp,APP_USER=myapp,APP_GROUP=myapp,HEALTH_CHECK_URL=http://localhost:8080/health"
templater.render "deploy_script" "/tmp/deploy_prod.sh"

echo "Fichiers générés:"
ls -la /tmp/app_config_dev.json /tmp/deploy_prod.sh

# Nettoyage
rm -f /tmp/app_config_dev.json /tmp/deploy_prod.sh
```

### 1.2 Gestion de configuration hiérarchique

Systèmes de configuration avec héritage et surcharge :

```bash
#!/bin/bash

# Gestion de configuration hiérarchique
echo "=== Gestion de configuration hiérarchique ==="

# Système de configuration hiérarchique
ConfigHierarchy() {
    local self="$1"
    
    declare -A $self._configs
    declare -a $self._hierarchy
    
    # Définition d'un niveau de configuration
    $self.define_level() {
        local level="$1"
        local config_data="$2"  # Format JSON-like simplifié
        
        $self._configs["$level"]="$config_data"
        $self._hierarchy+=("$level")
        
        echo "✓ Niveau '$level' défini"
    }
    
    # Construction de la hiérarchie
    $self.build_hierarchy() {
        local -A final_config
        
        # Application des niveaux dans l'ordre (du plus général au plus spécifique)
        for level in "${$self._hierarchy[@]}"; do
            local config_data="${$self._configs[$level]}"
            
            # Parsing simplifié des paires clé=valeur
            while IFS=';' read -ra pairs; do
                for pair in "${pairs[@]}"; do
                    if [[ "$pair" =~ ^[[:space:]]*([^=]+)[[:space:]]*=[[:space:]]*(.+)[[:space:]]*$ ]]; then
                        local key="${BASH_REMATCH[1]}"
                        local value="${BASH_REMATCH[2]}"
                        final_config["$key"]="$value"
                    fi
                done
            done <<< "$config_data"
        done
        
        # Sérialisation du résultat
        local result="{"
        local first=true
        for key in "${!final_config[@]}"; do
            if [[ "$first" == "true" ]]; then
                first=false
            else
                result+=","
            fi
            result+="\"$key\":\"${final_config[$key]}\""
        done
        result+="}"
        
        echo "$result"
    }
    
    # Récupération d'une valeur avec résolution hiérarchique
    $self.get_value() {
        local key="$1"
        local default_value="${2:-}"
        
        # Construction de la configuration finale
        local config_json
        config_json=$($self.build_hierarchy)
        
        # Extraction de la valeur (parsing JSON simplifié)
        local value
        value=$(echo "$config_json" | grep -o "\"$key\":\"[^\"]*\"" | cut -d'"' -f4)
        
        if [[ -n "$value" ]]; then
            echo "$value"
        else
            echo "$default_value"
        fi
    }
    
    # Validation de cohérence
    $self.validate_consistency() {
        echo "Validation de cohérence de la configuration..."
        
        local config_json
        config_json=$($self.build_hierarchy)
        
        local errors=0
        
        # Vérifications métier
        local app_env
        app_env=$($self.get_value "app.environment")
        local debug_enabled
        debug_enabled=$($self.get_value "debug.enabled")
        
        # En production, le debug doit être désactivé
        if [[ "$app_env" == "production" && "$debug_enabled" == "true" ]]; then
            echo "❌ Incohérence: debug activé en production"
            ((errors++))
        fi
        
        local db_host
        db_host=$($self.get_value "database.host")
        local cache_host
        cache_host=$($self.get_value "cache.host")
        
        # Les services ne doivent pas pointer sur localhost en production
        if [[ "$app_env" == "production" ]]; then
            if [[ "$db_host" == "localhost" || "$cache_host" == "localhost" ]]; then
                echo "❌ Incohérence: services pointant sur localhost en production"
                ((errors++))
            fi
        fi
        
        if (( errors == 0 )); then
            echo "✓ Configuration cohérente"
            return 0
        else
            echo "❌ $errors incohérences détectées"
            return 1
        fi
    }
    
    # Export vers différents formats
    $self.export_config() {
        local format="$1"
        local output_file="$2"
        
        local config_json
        config_json=$($self.build_hierarchy)
        
        case "$format" in
            json)
                echo "$config_json" > "$output_file"
                ;;
            env)
                # Conversion en variables d'environnement
                echo "$config_json" | grep -o '"[^"]*":"[^"]*"' | while IFS=':' read -r key value; do
                    key=$(echo "$key" | tr -d '"')
                    value=$(echo "$value" | tr -d '"')
                    echo "export ${key^^}=${value}" >> "$output_file"
                done
                ;;
            yaml)
                # Conversion simplifiée en YAML
                echo "$config_json" | sed 's/[{}]//g' | sed 's/","/"\n/g' | sed 's/":/": /g' | sed 's/"//g' > "$output_file"
                ;;
            *)
                echo "❌ Format non supporté: $format" >&2
                return 1
                ;;
        esac
        
        echo "✓ Configuration exportée au format $format: $output_file"
    }
}

# Démonstration de la configuration hiérarchique
echo "--- Configuration hiérarchique ---"

ConfigHierarchy "config_hierarchy"

# Définition des niveaux (du général au spécifique)
config_hierarchy.define_level "global" '
app.name=MyApp;
app.version=1.0.0;
debug.enabled=true;
database.host=localhost;
database.port=5432;
cache.host=localhost;
cache.port=6379;
cache.ttl=3600
'

config_hierarchy.define_level "environment" '
app.environment=development;
database.name=dev_db;
cache.enabled=true
'

config_hierarchy.define_level "server" '
app.environment=staging;
server.host=web01.example.com;
database.host=db01.example.com;
cache.host=redis01.example.com;
debug.enabled=false
'

# Construction et affichage
echo "--- Configuration finale ---"
final_config=$(config_hierarchy.build_hierarchy)
echo "$final_config" | python3 -m json.tool 2>/dev/null || echo "$final_config"

echo
echo "--- Récupération de valeurs ---"
echo "App environment: $(config_hierarchy.get_value "app.environment")"
echo "Database host: $(config_hierarchy.get_value "database.host")"
echo "Cache TTL: $(config_hierarchy.get_value "cache.ttl" "300")"

echo
echo "--- Validation de cohérence ---"
config_hierarchy.validate_consistency

echo
echo "--- Exports ---"
config_hierarchy.export_config "env" "/tmp/config.env"
config_hierarchy.export_config "yaml" "/tmp/config.yaml"

echo "Fichiers exportés:"
head -5 /tmp/config.env
echo "---"
head -5 /tmp/config.yaml

# Nettoyage
rm -f /tmp/config.env /tmp/config.yaml
```

## Section 2 : Orchestration et déploiement automatisé

### 2.1 Framework de déploiement

Système complet de déploiement automatisé :

```bash
#!/bin/bash

# Framework de déploiement
echo "=== Framework de déploiement ==="

# Orchestrateur de déploiement
DeploymentOrchestrator() {
    local self="$1"
    
    declare -a $self._targets
    declare -A $self._deployments
    declare -A $self._rollback_steps
    
    # Définition d'une cible de déploiement
    $self.add_target() {
        local name="$1"
        local connection_string="$2"
        local properties="$3"
        
        $self._targets+=("$name")
        $self._deployments["${name}_connection"]="$connection_string"
        $self._deployments["${name}_properties"]="$properties"
        
        echo "✓ Cible '$name' ajoutée"
    }
    
    # Définition d'une étape de déploiement
    $self.add_deployment_step() {
        local target="$1"
        local step_name="$2"
        local command="$3"
        local rollback_command="${4:-}"
        
        local step_key="${target}_${step_name}"
        $self._deployments["${step_key}_command"]="$command"
        $self._deployments["${step_key}_rollback"]="$rollback_command"
        
        echo "✓ Étape '$step_name' ajoutée pour '$target'"
    }
    
    # Exécution du déploiement
    $self.deploy() {
        local target="$1"
        local dry_run="${2:-false}"
        
        echo "=== DÉPLOIEMENT: $target ==="
        
        if [[ "$dry_run" == "true" ]]; then
            echo "🔍 DRY RUN - Simulation uniquement"
        fi
        
        # Vérification de la cible
        if ! $self._target_exists "$target"; then
            echo "❌ Cible inexistante: $target"
            return 1
        fi
        
        local connection="${$self._deployments[${target}_connection]}"
        local deployed_steps=()
        local success=true
        
        # Récupération des étapes pour cette cible
        local step_keys
        step_keys=$(echo "${!$self._deployments[@]}" | tr ' ' '\n' | grep "^${target}_" | grep "_command$" | sed "s/_command$//")
        
        for step_key in $step_keys; do
            local step_name="${step_key#${target}_}"
            local command="${$self._deployments[${step_key}_command]}"
            local rollback="${$self._deployments[${step_key}_rollback]}"
            
            echo "--- Étape: $step_name ---"
            echo "Commande: $command"
            
            if [[ "$dry_run" == "false" ]]; then
                # Exécution via SSH ou local
                if $self._execute_command "$connection" "$command"; then
                    echo "✓ Étape réussie"
                    deployed_steps+=("$step_key")
                else
                    echo "❌ Étape échouée"
                    success=false
                    break
                fi
            else
                echo "✓ Étape simulée"
                deployed_steps+=("$step_key")
            fi
        done
        
        if [[ "$success" == "true" ]]; then
            echo "🎉 Déploiement réussi!"
            return 0
        else
            echo "💥 Déploiement échoué - Lancement du rollback..."
            $self.rollback "$target" deployed_steps
            return 1
        fi
    }
    
    # Rollback
    $self.rollback() {
        local target="$1"
        local -a steps_to_rollback=("${!2}")
        
        echo "=== ROLLBACK: $target ==="
        
        local connection="${$self._deployments[${target}_connection]}"
        
        # Rollback dans l'ordre inverse
        for ((idx=${#steps_to_rollback[@]}-1; idx>=0; idx--)); do
            local step_key="${steps_to_rollback[$idx]}"
            local step_name="${step_key#${target}_}"
            local rollback_cmd="${$self._deployments[${step_key}_rollback]}"
            
            if [[ -n "$rollback_cmd" ]]; then
                echo "--- Rollback: $step_name ---"
                echo "Commande: $rollback_cmd"
                
                if $self._execute_command "$connection" "$rollback_cmd"; then
                    echo "✓ Rollback réussi"
                else
                    echo "❌ Rollback échoué"
                fi
            else
                echo "⚠️  Pas de rollback défini pour: $step_name"
            fi
        done
        
        echo "Rollback terminé"
    }
    
    # Exécution de commande
    $self._execute_command() {
        local connection="$1"
        local command="$2"
        
        if [[ "$connection" == "local" ]]; then
            # Exécution locale
            eval "$command"
        else
            # Exécution via SSH
            ssh "$connection" "$command"
        fi
    }
    
    # Vérification d'existence de cible
    $self._target_exists() {
        local target="$1"
        
        for existing_target in "${$self._targets[@]}"; do
            if [[ "$existing_target" == "$target" ]]; then
                return 0
            fi
        done
        return 1
    }
    
    # Liste des cibles
    $self.list_targets() {
        echo "Cibles de déploiement disponibles:"
        for target in "${$self._targets[@]}"; do
            echo "  $target (${$self._deployments[${target}_connection]})"
        done
    }
    
    # Validation pré-déploiement
    $self.validate_deployment() {
        local target="$1"
        
        echo "Validation du déploiement pour: $target"
        
        if ! $self._target_exists "$target"; then
            echo "❌ Cible inexistante"
            return 1
        fi
        
        local connection="${$self._deployments[${target}_connection]}"
        
        # Test de connectivité
        if ! $self._execute_command "$connection" "echo 'Connection test'"; then
            echo "❌ Connexion impossible"
            return 1
        fi
        
        # Vérification des prérequis
        if ! $self._execute_command "$connection" "which rsync tar systemctl"; then
            echo "❌ Outils manquants sur la cible"
            return 1
        fi
        
        echo "✓ Validation réussie"
        return 0
    }
}

# Démonstration du framework de déploiement
echo "--- Framework de déploiement ---"

DeploymentOrchestrator "orchestrator"

# Définition des cibles
orchestrator.add_target "web_server" "user@web01.example.com" "type=web,env=prod"
orchestrator.add_target "db_server" "user@db01.example.com" "type=database,env=prod"
orchestrator.add_target "local_dev" "local" "type=development,env=dev"

# Définition des étapes de déploiement pour le serveur web
orchestrator.add_deployment_step "web_server" "backup" \
    "tar -czf /tmp/backup_$(date +%Y%m%d_%H%M%S).tar.gz -C /var/www/html ." \
    "rm -f /tmp/backup_*.tar.gz"

orchestrator.add_deployment_step "web_server" "stop_service" \
    "systemctl stop nginx" \
    "systemctl start nginx"

orchestrator.add_deployment_step "web_server" "deploy_files" \
    "rsync -av --delete /tmp/deploy_source/ /var/www/html/" \
    "rm -rf /var/www/html/* && tar -xzf /tmp/backup_*.tar.gz -C /var/www/html/"

orchestrator.add_deployment_step "web_server" "update_permissions" \
    "chown -R www-data:www-data /var/www/html && chmod -R 755 /var/www/html"

orchestrator.add_deployment_step "web_server" "start_service" \
    "systemctl start nginx && systemctl enable nginx"

orchestrator.add_deployment_step "web_server" "health_check" \
    "curl -f http://localhost/health || exit 1"

# Liste des cibles
orchestrator.list_targets

echo
echo "--- Validation ---"
orchestrator.validate_deployment "local_dev"

echo
echo "--- Déploiement simulé ---"
orchestrator.deploy "local_dev" "true"

echo
echo "--- Déploiement réel (commenté pour sécurité) ---"
# orchestrator.deploy "local_dev"
```

### 2.2 Déploiement continu et intégration

Système de CI/CD basique en Bash :

```bash
#!/bin/bash

# Déploiement continu et intégration
echo "=== Déploiement continu et intégration ==="

# Pipeline CI/CD
CIPipeline() {
    local self="$1"
    
    declare -a $self._stages
    declare -A $self._stage_configs
    declare -A $self._artifacts
    
    # Définition d'un stage
    $self.add_stage() {
        local stage_name="$1"
        local commands="$2"
        local dependencies="${3:-}"
        
        $self._stages+=("$stage_name")
        $self._stage_configs["${stage_name}_commands"]="$commands"
        $self._stage_configs["${stage_name}_dependencies"]="$dependencies"
        
        echo "✓ Stage '$stage_name' ajouté"
    }
    
    # Exécution du pipeline
    $self.execute_pipeline() {
        local trigger="${1:-manual}"
        
        echo "=== PIPELINE CI/CD ==="
        echo "Déclencheur: $trigger"
        echo "Date: $(date)"
        echo
        
        local -A stage_status
        local failed_stages=()
        
        for stage in "${$self._stages[@]}"; do
            echo "--- Stage: $stage ---"
            
            # Vérification des dépendances
            local dependencies="${$self._stage_configs[${stage}_dependencies]}"
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
                    stage_status["$stage"]="skipped"
                    continue
                fi
            fi
            
            # Exécution du stage
            local commands="${$self._stage_configs[${stage}_commands]}"
            
            if $self._execute_stage "$stage" "$commands"; then
                stage_status["$stage"]="success"
                echo "✓ Stage réussi"
            else
                stage_status["$stage"]="failed"
                failed_stages+=("$stage")
                echo "❌ Stage échoué"
                
                # Arrêt sur échec
                break
            fi
            
            echo
        done
        
        # Rapport final
        $self._generate_report stage_status failed_stages
        
        # Nettoyage des artifacts si échec
        if (( ${#failed_stages[@]} > 0 )); then
            $self._cleanup_on_failure
            return 1
        else
            return 0
        fi
    }
    
    # Exécution d'un stage
    $self._execute_stage() {
        local stage_name="$1"
        local commands="$2"
        
        local stage_dir="/tmp/pipeline_${stage_name}_$$"
        mkdir -p "$stage_dir"
        cd "$stage_dir"
        
        # Parsing et exécution des commandes
        IFS=';' read -ra cmds <<< "$commands"
        local cmd_num=1
        
        for cmd in "${cmds[@]}"; do
            cmd=$(echo "$cmd" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
            
            if [[ -z "$cmd" ]]; then
                continue
            fi
            
            echo "[$cmd_num] $cmd"
            
            if eval "$cmd" 2>&1; then
                echo "  ✓ Commande réussie"
            else
                echo "  ❌ Commande échouée"
                cd - >/dev/null
                rm -rf "$stage_dir"
                return 1
            fi
            
            ((cmd_num++))
        done
        
        # Sauvegarde des artifacts
        if [[ -n "$(ls -A "$stage_dir" 2>/dev/null)" ]]; then
            local artifact_name="${stage_name}_$(date +%Y%m%d_%H%M%S).tar.gz"
            tar -czf "/tmp/$artifact_name" -C "$stage_dir" .
            $self._artifacts["$stage_name"]="/tmp/$artifact_name"
            echo "  📦 Artifact sauvegardé: /tmp/$artifact_name"
        fi
        
        cd - >/dev/null
        rm -rf "$stage_dir"
        return 0
    }
    
    # Génération de rapport
    $self._generate_report() {
        local -n status_ref="$1"
        local -n failed_ref="$2"
        
        local report_file="/tmp/pipeline_report_$(date +%Y%m%d_%H%M%S).txt"
        
        {
            echo "=== RAPPORT D'EXÉCUTION PIPELINE ==="
            echo "Date: $(date)"
            echo "Durée totale: ${SECONDS}s"
            echo
            echo "RÉSULTATS PAR STAGE:"
            
            for stage in "${$self._stages[@]}"; do
                local status="${status_ref[$stage]:-not_run}"
                case "$status" in
                    success) echo "✓ $stage - SUCCÈS" ;;
                    failed) echo "❌ $stage - ÉCHEC" ;;
                    skipped) echo "⚠️  $stage - IGNORÉ" ;;
                    not_run) echo "- $stage - NON EXÉCUTÉ" ;;
                esac
            done
            
            echo
            echo "ARTIFACTS GÉNÉRÉS:"
            for stage in "${!$self._artifacts[@]}"; do
                echo "  $stage: ${$self._artifacts[$stage]}"
            done
            
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
    
    # Nettoyage en cas d'échec
    $self._cleanup_on_failure() {
        echo "🧹 Nettoyage après échec..."
        
        # Suppression des artifacts temporaires
        for artifact in "${$self._artifacts[@]}"; do
            rm -f "$artifact"
        done
        
        # Nettoyage des répertoires temporaires
        rm -rf /tmp/pipeline_*_$$
    }
    
    # Trigger automatique (surveillance de fichiers)
    $self.watch_and_trigger() {
        local watch_dir="${1:-.}"
        local trigger_pattern="${2:-*.sh}"
        
        echo "Surveillance de $watch_dir pour les fichiers $trigger_pattern..."
        
        local last_trigger=0
        
        while true; do
            # Recherche de fichiers modifiés récemment
            local modified_files
            modified_files=$(find "$watch_dir" -name "$trigger_pattern" -newer /tmp/pipeline_last_check 2>/dev/null || true)
            
            if [[ -n "$modified_files" ]]; then
                echo "Fichiers modifiés détectés - Déclenchement du pipeline..."
                touch /tmp/pipeline_last_check
                
                if $self.execute_pipeline "auto"; then
                    echo "✓ Pipeline automatique réussi"
                else
                    echo "❌ Pipeline automatique échoué"
                fi
                
                last_trigger=$SECONDS
            fi
            
            sleep 5
        done
    }
}

# Démonstration du pipeline CI/CD
echo "--- Pipeline CI/CD ---"

CIPipeline "pipeline"

# Définition des stages
pipeline.add_stage "lint" "shellcheck script.sh || echo 'Shellcheck non disponible'; echo 'Linting terminé'"
pipeline.add_stage "test" "echo 'Exécution des tests unitaires'; sleep 2; echo 'Tests passés'"
pipeline.add_stage "build" "echo 'Construction de l'\''application'; mkdir -p build; echo 'App built' > build/app; sleep 1"
pipeline.add_stage "package" "echo 'Création du package'; tar -czf package.tar.gz build/; sleep 1" "build"
pipeline.add_stage "deploy" "echo 'Déploiement de l'\''application'; mkdir -p /tmp/deploy; cp package.tar.gz /tmp/deploy/; sleep 1" "package"
pipeline.add_stage "smoke_test" "echo 'Tests de fumée'; ls /tmp/deploy/package.tar.gz && echo 'Tests réussis'" "deploy"

# Exécution du pipeline
pipeline.execute_pipeline "manual"

# Affichage du dernier rapport
echo
echo "Dernier rapport:"
ls -la /tmp/pipeline_report_*.txt | tail -1
cat "$(ls -t /tmp/pipeline_report_*.txt | head -1)"

# Nettoyage
rm -f /tmp/pipeline_report_*.txt /tmp/package.tar.gz
rm -rf /tmp/pipeline_* build /tmp/deploy
```

## Section 3 : Infrastructure as Code avec Bash

### 3.1 Gestion d'infrastructure déclarative

Création et gestion d'infrastructure via des scripts déclaratifs :

```bash
#!/bin/bash

# Infrastructure as Code avec Bash
echo "=== Infrastructure as Code avec Bash ==="

# Gestionnaire d'infrastructure
InfrastructureManager() {
    local self="$1"
    
    declare -A $self._resources
    declare -A $self._states
    
    # Définition d'une ressource
    $self.define_resource() {
        local type="$1"
        local name="$2"
        local properties="$3"
        
        local resource_key="${type}_${name}"
        $self._resources["$resource_key"]="$properties"
        $self._states["$resource_key"]="undefined"
        
        echo "✓ Ressource définie: $type/$name"
    }
    
    # Application de l'infrastructure
    $self.apply() {
        echo "=== APPLICATION DE L'INFRASTRUCTURE ==="
        
        local applied=0
        local failed=0
        
        for resource_key in "${!$self._resources[@]}"; do
            local type="${resource_key%%_*}"
            local name="${resource_key#*_}"
            local properties="${$self._resources[$resource_key]}"
            
            echo "--- Application: $type/$name ---"
            
            if $self._apply_resource "$type" "$name" "$properties"; then
                $self._states["$resource_key"]="applied"
                ((applied++))
                echo "✓ Appliqué"
            else
                $self._states["$resource_key"]="failed"
                ((failed++))
                echo "❌ Échec"
            fi
        done
        
        echo
        echo "Résumé: $applied appliqués, $failed échoués"
        
        return $(( failed > 0 ))
    }
    
    # Application d'une ressource spécifique
    $self._apply_resource() {
        local type="$1"
        local name="$2"
        local properties="$3"
        
        case "$type" in
            file)
                $self._apply_file "$name" "$properties"
                ;;
            directory)
                $self._apply_directory "$name" "$properties"
                ;;
            user)
                $self._apply_user "$name" "$properties"
                ;;
            service)
                $self._apply_service "$name" "$properties"
                ;;
            package)
                $self._apply_package "$name" "$properties"
                ;;
            *)
                echo "Type de ressource inconnu: $type"
                return 1
                ;;
        esac
    }
    
    # Application d'un fichier
    $self._apply_file() {
        local path="$1"
        local properties="$2"
        
        # Parsing des propriétés
        local content owner mode
        
        content=$(echo "$properties" | grep -o 'content=[^,]*' | cut -d= -f2-)
        owner=$(echo "$properties" | grep -o 'owner=[^,]*' | cut -d= -f2)
        mode=$(echo "$properties" | grep -o 'mode=[^,]*' | cut -d= -f2)
        
        # Création du répertoire parent
        mkdir -p "$(dirname "$path")"
        
        # Écriture du contenu
        echo -e "$content" > "$path"
        
        # Application des permissions
        if [[ -n "$owner" ]]; then
            chown "$owner" "$path"
        fi
        
        if [[ -n "$mode" ]]; then
            chmod "$mode" "$path"
        fi
        
        echo "Fichier créé: $path"
    }
    
    # Application d'un répertoire
    $self._apply_directory() {
        local path="$1"
        local properties="$2"
        
        local owner mode
        
        owner=$(echo "$properties" | grep -o 'owner=[^,]*' | cut -d= -f2)
        mode=$(echo "$properties" | grep -o 'mode=[^,]*' | cut -d= -f2)
        
        mkdir -p "$path"
        
        if [[ -n "$owner" ]]; then
            chown "$owner" "$path"
        fi
        
        if [[ -n "$mode" ]]; then
            chmod "$mode" "$path"
        fi
        
        echo "Répertoire créé: $path"
    }
    
    # Application d'un utilisateur
    $self._apply_user() {
        local username="$1"
        local properties="$2"
        
        local uid gid home shell
        
        uid=$(echo "$properties" | grep -o 'uid=[^,]*' | cut -d= -f2)
        gid=$(echo "$properties" | grep -o 'gid=[^,]*' | cut -d= -f2)
        home=$(echo "$properties" | grep -o 'home=[^,]*' | cut -d= -f2)
        shell=$(echo "$properties" | grep -o 'shell=[^,]*' | cut -d= -f2)
        
        # Simulation de création d'utilisateur (nécessite sudo en vrai)
        echo "Utilisateur à créer: $username"
        echo "  UID: ${uid:-auto}"
        echo "  GID: ${gid:-auto}"
        echo "  Home: ${home:-/home/$username}"
        echo "  Shell: ${shell:-/bin/bash}"
        
        # En simulation seulement
        echo "useradd -u $uid -g $gid -d $home -s $shell $username"
    }
    
    # Application d'un service
    $self._apply_service() {
        local service_name="$1"
        local properties="$2"
        
        local state command
        
        state=$(echo "$properties" | grep -o 'state=[^,]*' | cut -d= -f2)
        command=$(echo "$properties" | grep -o 'command=[^,]*' | cut -d= -f2-)
        
        case "$state" in
            started|enabled)
                echo "Service à démarrer: $service_name"
                echo "systemctl enable $service_name"
                echo "systemctl start $service_name"
                ;;
            stopped)
                echo "Service à arrêter: $service_name"
                echo "systemctl stop $service_name"
                ;;
            disabled)
                echo "Service à désactiver: $service_name"
                echo "systemctl disable $service_name"
                ;;
        esac
        
        if [[ -n "$command" ]]; then
            echo "Commande personnalisée: $command"
        fi
    }
    
    # Application d'un package
    $self._apply_package() {
        local package_name="$1"
        local properties="$2"
        
        local state version
        
        state=$(echo "$properties" | grep -o 'state=[^,]*' | cut -d= -f2)
        version=$(echo "$properties" | grep -o 'version=[^,]*' | cut -d= -f2)
        
        case "$state" in
            present|installed)
                echo "Package à installer: $package_name${version:+=$version}"
                echo "apt install $package_name${version:+=$version}  # ou équivalent"
                ;;
            absent|removed)
                echo "Package à supprimer: $package_name"
                echo "apt remove $package_name"
                ;;
        esac
    }
    
    # Destruction de l'infrastructure
    $self.destroy() {
        echo "=== DESTRUCTION DE L'INFRASTRUCTURE ==="
        
        local destroyed=0
        
        for resource_key in "${!$self._resources[@]}"; do
            local type="${resource_key%%_*}"
            local name="${resource_key#*_}"
            
            echo "--- Destruction: $type/$name ---"
            
            if $self._destroy_resource "$type" "$name"; then
                $self._states["$resource_key"]="destroyed"
                ((destroyed++))
                echo "✓ Détruit"
            else
                echo "❌ Échec destruction"
            fi
        done
        
        echo "Résumé: $destroyed ressources détruites"
    }
    
    $self._destroy_resource() {
        local type="$1"
        local name="$2"
        
        case "$type" in
            file)
                rm -f "$name"
                echo "Fichier supprimé: $name"
                ;;
            directory)
                rmdir "$name" 2>/dev/null || echo "Répertoire non vide ou inexistant: $name"
                ;;
            user)
                echo "userdel $name  # Simulation"
                ;;
            service)
                echo "systemctl stop $name && systemctl disable $name  # Simulation"
                ;;
            package)
                echo "apt remove $name  # Simulation"
                ;;
        esac
    }
    
    # État de l'infrastructure
    $self.status() {
        echo "=== ÉTAT DE L'INFRASTRUCTURE ==="
        
        for resource_key in "${!$self._resources[@]}"; do
            local type="${resource_key%%_*}"
            local name="${resource_key#*_}"
            local state="${$self._states[$resource_key]}"
            
            case "$state" in
                applied) echo "✓ $type/$name - APPLIQUÉ" ;;
                failed) echo "❌ $type/$name - ÉCHEC" ;;
                destroyed) echo "🗑️  $type/$name - DÉTRUIT" ;;
                undefined) echo "❓ $type/$name - NON DÉFINI" ;;
            esac
        done
    }
}

# Démonstration d'Infrastructure as Code
echo "--- Infrastructure as Code ---"

InfrastructureManager "infra"

# Définition de l'infrastructure
infra.define_resource "directory" "/tmp/myapp" "owner=root,mode=755"
infra.define_resource "file" "/tmp/myapp/config.txt" "content=port=8080\ndb_host=localhost,owner=root,mode=644"
infra.define_resource "user" "myapp" "uid=1001,gid=1001,home=/home/myapp,shell=/bin/bash"
infra.define_resource "package" "nginx" "state=present,version=latest"
infra.define_resource "service" "nginx" "state=started,command=nginx -g 'daemon off;'"

# État initial
infra.status

echo
echo "--- Application ---"
infra.apply

echo
echo "--- État après application ---"
infra.status

echo
echo "--- Destruction ---"
infra.destroy

echo
echo "--- État final ---"
infra.status

# Nettoyage
rm -rf /tmp/myapp
```

### 3.2 Gestion d'état et idempotence

Systèmes garantissant l'idempotence des opérations :

```bash
#!/bin/bash

# Gestion d'état et idempotence
echo "=== Gestion d'état et idempotence ==="

# Gestionnaire d'état idempotent
IdempotentManager() {
    local self="$1"
    local state_file="${2:-/tmp/idempotent_state.json}"
    
    declare -A $self._desired_state
    declare -A $self._current_state
    
    # Initialisation
    $self._init() {
        # Chargement de l'état actuel depuis le fichier
        if [[ -f "$state_file" ]]; then
            # Parsing JSON simplifié
            while IFS=':' read -r key value; do
                key=$(echo "$key" | tr -d '"{}, ')
                value=$(echo "$value" | tr -d '"{}, ')
                if [[ -n "$key" && -n "$value" ]]; then
                    $self._current_state["$key"]="$value"
                fi
            done < <(grep -o '"[^"]*":"[^"]*"' "$state_file")
        fi
    }
    
    # Définition de l'état désiré
    $self.define_state() {
        local resource="$1"
        local properties="$2"
        
        $self._desired_state["$resource"]="$properties"
        echo "✓ État défini pour: $resource"
    }
    
    # Application idempotente
    $self.apply_idempotent() {
        echo "=== APPLICATION IDÉMPOTENTE ==="
        
        $self._init
        
        local changes=0
        
        for resource in "${!$self._desired_state[@]}"; do
            local desired="${$self._desired_state[$resource]}"
            local current="${$self._current_state[$resource]}"
            
            if [[ "$desired" != "$current" ]]; then
                echo "--- Changement: $resource ---"
                echo "  Actuel: ${current:-<non défini>}"
                echo "  Désiré: $desired"
                
                if $self._apply_change "$resource" "$desired"; then
                    $self._current_state["$resource"]="$desired"
                    $self._save_state
                    ((changes++))
                    echo "✓ Appliqué"
                else
                    echo "❌ Échec"
                fi
            else
                echo "✓ $resource déjà à jour"
            fi
        done
        
        echo "Résumé: $changes changements appliqués"
        return $(( changes > 0 ))
    }
    
    # Application d'un changement
    $self._apply_change() {
        local resource="$1"
        local properties="$2"
        
        case "$resource" in
            package:*)
                local package="${resource#package:}"
                local action=$(echo "$properties" | grep -o 'action=[^,]*' | cut -d= -f2)
                
                case "$action" in
                    install)
                        echo "Installation du package: $package"
                        # Simulation
                        sleep 0.1
                        ;;
                    remove)
                        echo "Suppression du package: $package"
                        sleep 0.1
                        ;;
                esac
                ;;
                
            file:*)
                local file_path="${resource#file:}"
                local content=$(echo "$properties" | grep -o 'content=[^,]*' | cut -d= -f2-)
                
                echo "Création/modification du fichier: $file_path"
                mkdir -p "$(dirname "$file_path")"
                echo "$content" > "$file_path"
                ;;
                
            service:*)
                local service="${resource#service:}"
                local state=$(echo "$properties" | grep -o 'state=[^,]*' | cut -d= -f2)
                
                echo "Configuration du service $service à l'état: $state"
                # Simulation
                sleep 0.1
                ;;
                
            user:*)
                local username="${resource#user:}"
                local action=$(echo "$properties" | grep -o 'action=[^,]*' | cut -d= -f2)
                
                echo "Gestion de l'utilisateur $username: $action"
                sleep 0.1
                ;;
                
            *)
                echo "Type de ressource inconnu: $resource"
                return 1
                ;;
        esac
        
        return 0
    }
    
    # Sauvegarde de l'état
    $self._save_state() {
        local json="{"
        local first=true
        
        for key in "${!$self._current_state[@]}"; do
            if [[ "$first" == "true" ]]; then
                first=false
            else
                json+=","
            fi
            json+="\"$key\":\"${$self._current_state[$key]}\""
        done
        
        json+="}"
        
        echo "$json" > "$state_file"
    }
    
    # Vérification de l'état
    $self.verify_state() {
        echo "=== VÉRIFICATION DE L'ÉTAT ==="
        
        $self._init
        
        local compliant=true
        
        for resource in "${!$self._desired_state[@]}"; do
            local desired="${$self._desired_state[$resource]}"
            local current="${$self._current_state[$resource]}"
            
            if [[ "$desired" == "$current" ]]; then
                echo "✓ $resource conforme"
            else
                echo "❌ $resource non conforme"
                echo "  Désiré: $desired"
                echo "  Actuel: ${current:-<non défini>}"
                compliant=false
            fi
        done
        
        if [[ "$compliant" == "true" ]]; then
            echo "🎉 Infrastructure conforme"
            return 0
        else
            echo "⚠️  Écarts détectés"
            return 1
        fi
    }
    
    # Rollback
    $self.rollback() {
        local backup_state="$1"
        
        if [[ ! -f "$backup_state" ]]; then
            echo "❌ Fichier de sauvegarde inexistant: $backup_state"
            return 1
        fi
        
        echo "=== ROLLBACK ==="
        
        # Chargement de l'état de sauvegarde
        declare -A backup_state_data
        while IFS=':' read -r key value; do
            key=$(echo "$key" | tr -d '"{}, ')
            value=$(echo "$value" | tr -d '"{}, ')
            if [[ -n "$key" && -n "$value" ]]; then
                backup_state_data["$key"]="$value"
            fi
        done < <(grep -o '"[^"]*":"[^"]*"' "$backup_state")
        
        # Application du rollback
        local rollbacks=0
        for resource in "${!backup_state_data[@]}"; do
            local backup_value="${backup_state_data[$resource]}"
            local current_value="${$self._current_state[$resource]}"
            
            if [[ "$backup_value" != "$current_value" ]]; then
                echo "Rollback de $resource: $current_value -> $backup_value"
                $self._apply_change "$resource" "$backup_value"
                $self._current_state["$resource"]="$backup_value"
                ((rollbacks++))
            fi
        done
        
        $self._save_state
        echo "Rollback terminé: $rollbacks changements"
    }
}

# Démonstration de l'idempotence
echo "--- Gestion d'état idempotent ---"

IdempotentManager "idempotent_mgr" "/tmp/idempotent_state.json"

# Définition de l'état désiré
idempotent_mgr.define_state "package:nginx" "action=install"
idempotent_mgr.define_state "file:/tmp/app.conf" "content=port=8080\nhost=localhost"
idempotent_mgr.define_state "service:nginx" "state=started"

# Première application
echo "--- Première application ---"
idempotent_mgr.apply_idempotent

# Vérification
idempotent_mgr.verify_state

# Deuxième application (devrait être idempotente)
echo "--- Deuxième application (idempotente) ---"
idempotent_mgr.apply_idempotent

# Modification manuelle pour tester
echo "Simulation de modification externe..."
echo '{"package:nginx":"action=install","file:/tmp/app.conf":"content=port=9090","service:nginx":"state=started"}' > /tmp/idempotent_state.json

# Vérification après modification
echo "--- Vérification après modification ---"
idempotent_mgr.verify_state

# Rollback
echo "--- Rollback ---"
cp /tmp/idempotent_state.json /tmp/backup_state.json
idempotent_mgr.rollback "/tmp/backup_state.json"

# Vérification finale
echo "--- Vérification finale ---"
idempotent_mgr.verify_state

# Nettoyage
rm -f /tmp/idempotent_state.json /tmp/backup_state.json /tmp/app.conf
```

## Conclusion : L'automatisation à l'échelle

Le déploiement avancé et l'automatisation en Bash transforment les scripts individuels en véritables systèmes d'orchestration capables de gérer des infrastructures complexes. Comme une chaîne de production moderne, vos scripts peuvent désormais déployer, configurer, surveiller et maintenir automatiquement vos applications à grande échelle.

**Points clés à retenir :**

1. **Templates et configuration** : Créez des systèmes de templates dynamiques pour gérer la complexité des configurations multi-environnements
2. **Orchestration de déploiement** : Bâtissez des frameworks complets avec rollback automatique et gestion d'erreurs
3. **CI/CD en Bash** : Implémentez des pipelines complets de déploiement continu directement en Bash
4. **Infrastructure as Code** : Déclarez et gérez votre infrastructure comme du code, avec idempotence et gestion d'état
5. **Idempotence** : Assurez-vous que vos opérations peuvent être répétées sans effets secondaires

Dans le chapitre suivant, nous explorerons les techniques avancées de scripting réseau, pour que vos scripts Bash puissent communiquer, synchroniser, et orchestrer des opérations distribuées.

---

**Exercice pratique :** Créez un système complet de déploiement d'application web incluant :
- Gestion de configuration multi-environnements avec templates
- Pipeline CI/CD avec tests automatiques
- Orchestration de déploiement avec rollback
- Infrastructure as Code pour les serveurs web et base de données
- Monitoring post-déploiement et alertes

**Réflexion :** Comment adapteriez-vous ces techniques d'automatisation pour gérer un cluster Kubernetes ou un environnement cloud multi-régions ?
