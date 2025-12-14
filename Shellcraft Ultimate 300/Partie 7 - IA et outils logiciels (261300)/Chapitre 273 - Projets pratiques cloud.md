# Chapitre 273 - Projets pratiques cloud

## Table des matières
- [Introduction](#introduction)
- [Projet 1 : Infrastructure as Code avec IA](#projet-1--infrastructure-as-code-avec-ia)
- [Projet 2 : Auto-scaling intelligent](#projet-2--auto-scaling-intelligent)
- [Projet 3 : Migration cloud automatisée](#projet-3--migration-cloud-automatisée)
- [Projet 4 : Optimisation de coûts cloud](#projet-4--optimisation-de-coûts-cloud)
- [Conclusion](#conclusion)

## Introduction

Les projets pratiques cloud combinent l'automatisation shell, l'infrastructure cloud, et l'intelligence artificielle pour créer des solutions complètes et professionnelles. Ces projets démontrent comment orchestrer des infrastructures complexes, optimiser les ressources, et gérer des environnements cloud à grande échelle.

Ces projets représentent l'application pratique de toutes les techniques apprises, créant des systèmes qui sont à la fois puissants, intelligents, et maintenables.

## Projet 1 : Infrastructure as Code avec IA

### Objectif

Créer un système qui génère automatiquement du code d'infrastructure (Terraform, CloudFormation) à partir de descriptions en langage naturel.

### Implémentation

**Générateur d'infrastructure** :
```bash
#!/bin/bash
# Générateur d'infrastructure avec IA

set -euo pipefail

INFRA_DESCRIPTION="${1:-}"
OUTPUT_FORMAT="${2:-terraform}"  # terraform, cloudformation, ansible
CLOUD_PROVIDER="${3:-aws}"  # aws, azure, gcp

if [ -z "$INFRA_DESCRIPTION" ]; then
    echo "Usage: $0 '<description infrastructure>' [format] [provider]"
    exit 1
fi

generate_infrastructure() {
    echo "=== Génération d'infrastructure ==="
    echo "Description: $INFRA_DESCRIPTION"
    echo "Format: $OUTPUT_FORMAT"
    echo "Provider: $CLOUD_PROVIDER"
    echo
    
    # Phase 1: Analyse avec IA
    echo "Phase 1: Analyse des besoins..."
    local analysis=$(analyze_infrastructure_needs "$INFRA_DESCRIPTION")
    
    # Phase 2: Génération du code
    echo "Phase 2: Génération du code..."
    local infrastructure_code=$(generate_infra_code "$analysis" "$OUTPUT_FORMAT" "$CLOUD_PROVIDER")
    
    # Phase 3: Validation
    echo "Phase 3: Validation..."
    validate_infrastructure_code "$infrastructure_code" "$OUTPUT_FORMAT"
    
    # Phase 4: Sauvegarde
    local output_file="infrastructure.${OUTPUT_FORMAT}"
    echo "$infrastructure_code" > "$output_file"
    echo "✓ Infrastructure générée: $output_file"
}

analyze_infrastructure_needs() {
    local description="$1"
    
    local prompt="Analyse cette description d'infrastructure et identifie:
- Composants nécessaires (VMs, load balancers, databases, etc.)
- Configuration réseau
- Sécurité requise
- Scalabilité nécessaire
- Coûts estimés

Description: $description

Format: JSON structuré"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 2000
        }" | jq -r '.choices[0].message.content'
}

generate_infra_code() {
    local analysis="$1"
    local format="$2"
    local provider="$3"
    
    local prompt="Génère du code $format pour $provider basé sur cette analyse:
$analysis

Le code doit:
- Être production-ready
- Suivre les meilleures pratiques
- Inclure la sécurité
- Être modulaire et maintenable
- Inclure des commentaires

Génère uniquement le code, sans explications."

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 3000
        }" | jq -r '.choices[0].message.content'
}

validate_infrastructure_code() {
    local code="$1"
    local format="$2"
    
    case "$format" in
        terraform)
            echo "$code" > /tmp/validate.tf
            terraform fmt -check /tmp/validate.tf && \
            terraform validate -no-color /tmp/validate.tf || {
                echo "⚠ Validation Terraform échouée"
                return 1
            }
            ;;
        cloudformation)
            # Validation CloudFormation
            echo "$code" > /tmp/validate.yaml
            aws cloudformation validate-template \
                --template-body file:///tmp/validate.yaml || {
                echo "⚠ Validation CloudFormation échouée"
                return 1
            }
            ;;
    esac
    
    echo "✓ Code validé"
}
```

## Projet 2 : Auto-scaling intelligent

### Objectif

Créer un système d'auto-scaling qui utilise l'IA pour prédire la charge et ajuster les ressources de manière optimale.

### Implémentation

**Auto-scaler intelligent** :
```bash
#!/bin/bash
# Auto-scaling intelligent avec IA

set -euo pipefail

SCALING_GROUP="${1:-}"
METRICS_WINDOW="${2:-3600}"  # 1 heure

if [ -z "$SCALING_GROUP" ]; then
    echo "Usage: $0 <scaling_group_name> [metrics_window]"
    exit 1
fi

intelligent_autoscaling() {
    while true; do
        echo "=== Cycle d'auto-scaling ==="
        
        # Collecter les métriques
        local current_metrics=$(collect_scaling_metrics "$SCALING_GROUP")
        local historical_metrics=$(get_historical_metrics "$METRICS_WINDOW")
        
        # Prédire la charge future avec IA
        local prediction=$(predict_load "$current_metrics" "$historical_metrics")
        
        # Déterminer l'action de scaling
        local scaling_action=$(determine_scaling_action "$prediction" "$current_metrics")
        
        # Exécuter l'action
        execute_scaling_action "$SCALING_GROUP" "$scaling_action"
        
        sleep 60  # Vérifier toutes les minutes
    done
}

predict_load() {
    local current="$1"
    local historical="$2"
    
    local prompt="Prédit la charge système pour les 30 prochaines minutes basé sur:
Métriques actuelles:
$current

Historique (1 heure):
$historical

Prédit:
- Charge CPU attendue
- Charge mémoire attendue
- Charge réseau attendue
- Nombre d'instances nécessaires

Format: JSON"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 500
        }" | jq -r '.choices[0].message.content'
}

determine_scaling_action() {
    local prediction="$1"
    local current="$2"
    
    local current_instances=$(get_current_instance_count)
    local predicted_instances=$(echo "$prediction" | jq -r '.instances_needed')
    
    local action="none"
    local delta=$((predicted_instances - current_instances))
    
    if [ "$delta" -gt 2 ]; then
        action="scale-up"
        local count=$delta
    elif [ "$delta" -lt -2 ]; then
        action="scale-down"
        local count=$((delta * -1))
    fi
    
    echo "{\"action\": \"$action\", \"count\": ${count:-0}}"
}

execute_scaling_action() {
    local group="$1"
    local action_data="$2"
    
    local action=$(echo "$action_data" | jq -r '.action')
    local count=$(echo "$action_data" | jq -r '.count')
    
    case "$action" in
        scale-up)
            echo "Scaling up: +$count instances"
            scale_up "$group" "$count"
            ;;
        scale-down)
            echo "Scaling down: -$count instances"
            scale_down "$group" "$count"
            ;;
        none)
            echo "Aucun scaling nécessaire"
            ;;
    esac
}
```

## Projet 3 : Migration cloud automatisée

### Objectif

Automatiser la migration d'applications entre différents fournisseurs cloud avec validation et rollback automatique.

### Implémentation

**Migrateur automatisé** :
```bash
#!/bin/bash
# Migration cloud automatisée avec IA

set -euo pipefail

SOURCE_PROVIDER="${1:-}"
TARGET_PROVIDER="${2:-}"
RESOURCE_LIST="${3:-}"

if [ -z "$SOURCE_PROVIDER" ] || [ -z "$TARGET_PROVIDER" ]; then
    echo "Usage: $0 <source_provider> <target_provider> [resource_list.json]"
    exit 1
fi

automated_migration() {
    echo "=== Migration automatisée ==="
    echo "Source: $SOURCE_PROVIDER"
    echo "Cible: $TARGET_PROVIDER"
    echo
    
    # Phase 1: Analyse et planification
    echo "Phase 1: Analyse et planification..."
    local migration_plan=$(generate_migration_plan)
    
    # Phase 2: Préparation
    echo "Phase 2: Préparation..."
    prepare_migration "$migration_plan"
    
    # Phase 3: Réplication
    echo "Phase 3: Réplication..."
    replicate_resources "$migration_plan"
    
    # Phase 4: Validation
    echo "Phase 4: Validation..."
    if validate_migration "$migration_plan"; then
        echo "✓ Migration validée"
    else
        echo "✗ Échec de validation, rollback"
        rollback_migration
        exit 1
    fi
    
    # Phase 5: Basculement
    echo "Phase 5: Basculement..."
    read -p "Basculer vers $TARGET_PROVIDER ? (o/N): " confirm
    if [[ "$confirm" =~ ^[Oo]$ ]]; then
        perform_cutover "$migration_plan"
        echo "✓ Migration terminée"
    else
        echo "Migration annulée"
    fi
}

generate_migration_plan() {
    local resources=$(cat "${RESOURCE_LIST:-/dev/stdin}")
    
    local prompt="Génère un plan de migration détaillé:
Source: $SOURCE_PROVIDER
Cible: $TARGET_PROVIDER
Ressources: $resources

Le plan doit inclure:
- Mapping des équivalents entre providers
- Ordre de migration
- Dépendances
- Tests de validation
- Plan de rollback
- Estimation de temps et coûts

Format: JSON structuré"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 3000
        }" | jq -r '.choices[0].message.content' > migration_plan.json
    
    cat migration_plan.json
}
```

## Projet 4 : Optimisation de coûts cloud

### Objectif

Créer un système qui analyse les coûts cloud et génère automatiquement des recommandations d'optimisation avec scripts d'implémentation.

### Implémentation

**Optimiseur de coûts** :
```bash
#!/bin/bash
# Optimisation de coûts cloud avec IA

set -euo pipefail

PROVIDER="${1:-all}"  # aws, azure, gcp, all
ANALYSIS_PERIOD="${2:-30}"  # jours

optimize_costs() {
    echo "=== Analyse des coûts ==="
    
    # Collecter les données de coûts
    local cost_data=$(collect_cost_data "$PROVIDER" "$ANALYSIS_PERIOD")
    
    # Analyser avec IA
    echo "Analyse avec IA..."
    local recommendations=$(ai_analyze_costs "$cost_data")
    
    # Générer les scripts d'optimisation
    echo "Génération des scripts d'optimisation..."
    generate_optimization_scripts "$recommendations"
    
    # Rapport
    echo "$recommendations" > cost_optimization_report.json
    generate_human_readable_report "$recommendations"
}

collect_cost_data() {
    local provider="$1"
    local period="$2"
    
    case "$provider" in
        aws|all)
            aws ce get-cost-and-usage \
                --time-period Start=$(date -d "$period days ago" +%Y-%m-01),End=$(date +%Y-%m-%d) \
                --granularity MONTHLY \
                --metrics BlendedCost > aws_costs.json
            ;;
        azure|all)
            az consumption usage list \
                --start-date $(date -d "$period days ago" +%Y-%m-%d) \
                --end-date $(date +%Y-%m-%d) > azure_costs.json
            ;;
        gcp|all)
            gcloud billing accounts list > gcp_billing.json
            ;;
    esac
    
    # Combiner les données
    jq -s '.' aws_costs.json azure_costs.json gcp_billing.json 2>/dev/null || echo '{}'
}

ai_analyze_costs() {
    local cost_data="$1"
    
    local prompt="Analyse ces données de coûts cloud et génère des recommandations d'optimisation:
$cost_data

Pour chaque recommandation, fournis:
- Description du problème
- Économies potentielles
- Impact sur les performances
- Script d'implémentation
- Risques et mitigations

Format: JSON avec array de recommandations"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 3000
        }" | jq -r '.choices[0].message.content'
}

generate_optimization_scripts() {
    local recommendations="$1"
    
    local count=1
    echo "$recommendations" | jq -r '.recommendations[]' | \
    while IFS= read -r rec; do
        local script=$(echo "$rec" | jq -r '.implementation_script')
        local title=$(echo "$rec" | jq -r '.title' | tr ' ' '_' | tr -d '[:punct:]')
        
        echo "$script" > "optimize_${count}_${title}.sh"
        chmod +x "optimize_${count}_${title}.sh"
        ((count++))
    done
    
    echo "Scripts d'optimisation générés: optimize_*.sh"
}
```

## Conclusion

Ces projets pratiques cloud démontrent comment combiner l'automatisation shell, l'infrastructure cloud, et l'intelligence artificielle pour créer des solutions complètes et professionnelles. Chaque projet résout un problème réel et fournit une valeur immédiate.

En appliquant ces projets, vous développez non seulement vos compétences techniques, mais aussi votre capacité à concevoir des systèmes cloud intelligents qui s'optimisent et s'adaptent automatiquement.

Dans le chapitre suivant, nous explorerons la sécurité et l'IA, découvrant comment utiliser l'intelligence artificielle pour renforcer la sécurité de vos systèmes et détecter les menaces.

