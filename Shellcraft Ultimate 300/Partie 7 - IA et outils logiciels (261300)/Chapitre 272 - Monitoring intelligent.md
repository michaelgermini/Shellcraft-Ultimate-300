# Chapitre 272 - Monitoring intelligent

## Table des matières
- [Introduction](#introduction)
- [Monitoring prédictif avec IA](#monitoring-prédictif-avec-ia)
- [Détection d'anomalies intelligente](#détection-danomalies-intelligente)
- [Alertes contextuelles](#alertes-contextuelles)
- [Dashboard intelligent](#dashboard-intelligent)
- [Conclusion](#conclusion)

## Introduction

Le monitoring intelligent va au-delà de la simple collecte de métriques : il comprend le contexte, détecte les patterns, prédit les problèmes, et fournit des insights actionnables. En combinant les outils de monitoring traditionnels avec l'intelligence artificielle, nous créons des systèmes qui non seulement surveillent, mais comprennent et anticipent.

Imaginez le monitoring intelligent comme un médecin expérimenté : il ne se contente pas de mesurer votre température, il analyse tous les symptômes, comprend votre historique, et peut prédire les problèmes avant qu'ils ne deviennent critiques.

## Monitoring prédictif avec IA

### Collecte de métriques intelligente

**Collecteur de métriques adaptatif** :
```bash
#!/bin/bash
# Collecteur de métriques intelligent

METRICS_STORAGE="/var/metrics"
COLLECTION_INTERVAL="${COLLECTION_INTERVAL:-60}"

collect_intelligent_metrics() {
    local timestamp=$(date +%s)
    local metrics_file="$METRICS_STORAGE/metrics_${timestamp}.json"
    
    # Collecter les métriques de base
    local cpu=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
    local memory=$(free | grep Mem | awk '{printf "%.2f", $3/$2 * 100}')
    local disk=$(df -h / | awk 'NR==2 {print $5}' | sed 's/%//')
    local load=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | tr -d ',')
    
    # Collecter les métriques réseau
    local network_in=$(cat /proc/net/dev | grep -E "eth0|ens" | awk '{print $2}')
    local network_out=$(cat /proc/net/dev | grep -E "eth0|ens" | awk '{print $10}')
    
    # Collecter les métriques de processus
    local process_count=$(ps aux | wc -l)
    local zombie_count=$(ps aux | grep -c '<defunct>' || echo 0)
    
    # Créer le JSON
    cat > "$metrics_file" << EOF
{
    "timestamp": $timestamp,
    "hostname": "$(hostname)",
    "metrics": {
        "cpu": {
            "usage": $cpu,
            "cores": $(nproc)
        },
        "memory": {
            "usage_percent": $memory,
            "total_mb": $(free -m | grep Mem | awk '{print $2}'),
            "used_mb": $(free -m | grep Mem | awk '{print $3}')
        },
        "disk": {
            "usage_percent": $disk,
            "mount": "/"
        },
        "load": {
            "1min": $load,
            "5min": $(uptime | awk -F'load average:' '{print $2}' | awk '{print $2}' | tr -d ','),
            "15min": $(uptime | awk -F'load average:' '{print $2}' | awk '{print $3}' | tr -d ',')
        },
        "network": {
            "bytes_in": $network_in,
            "bytes_out": $network_out
        },
        "processes": {
            "total": $process_count,
            "zombies": $zombie_count
        }
    }
}
EOF
    
    echo "$metrics_file"
}

# Boucle de collecte adaptative
adaptive_collection() {
    local base_interval="$COLLECTION_INTERVAL"
    local current_interval="$base_interval"
    
    while true; do
        local metrics_file=$(collect_intelligent_metrics)
        
        # Analyser la charge avec IA pour ajuster l'intervalle
        local load=$(jq -r '.metrics.load."1min"' "$metrics_file")
        
        if (( $(echo "$load > 2.0" | bc -l) )); then
            # Charge élevée, réduire la fréquence
            current_interval=$((base_interval * 2))
        elif (( $(echo "$load < 0.5" | bc -l) )); then
            # Charge faible, augmenter la fréquence
            current_interval=$((base_interval / 2))
        else
            current_interval="$base_interval"
        fi
        
        sleep "$current_interval"
    done
}
```

### Prédiction de problèmes

**Prédicteur IA** :
```bash
#!/bin/bash
# Prédiction de problèmes avec IA

PREDICTION_WINDOW="${PREDICTION_WINDOW:-3600}"  # 1 heure

predict_issues() {
    local historical_data=$(get_historical_metrics "$PREDICTION_WINDOW")
    
    local prompt="Analyse ces métriques système historiques et prédit les problèmes futurs:
$historical_data

Prédit pour les prochaines 2 heures:
- Probabilité de saturation CPU (>90%)
- Probabilité de saturation mémoire (>95%)
- Probabilité de saturation disque (>90%)
- Risque de crash système
- Actions préventives recommandées

Format: JSON avec probabilités et recommandations"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 1500
        }" | jq -r '.choices[0].message.content'
}

get_historical_metrics() {
    local window="$1"
    local cutoff=$(date -d "@$(( $(date +%s) - window ))" +%s)
    
    find "$METRICS_STORAGE" -name "metrics_*.json" -type f | \
    while read -r file; do
        local timestamp=$(jq -r '.timestamp' "$file" 2>/dev/null)
        if [ -n "$timestamp" ] && [ "$timestamp" -gt "$cutoff" ]; then
            cat "$file"
        fi
    done | jq -s '.'
}
```

## Détection d'anomalies intelligente

### Détecteur d'anomalies

**Système de détection** :
```bash
#!/bin/bash
# Détection d'anomalies avec IA

detect_anomalies() {
    local current_metrics="$1"
    local baseline_metrics="$2"
    
    local prompt="Compare ces métriques actuelles avec le baseline et détecte les anomalies:
Métriques actuelles:
$current_metrics

Baseline (moyenne sur 7 jours):
$baseline_metrics

Détecte:
- Anomalies significatives (écart > 2 écarts-types)
- Patterns inhabituels
- Causes probables
- Gravité (low, medium, high, critical)

Format: JSON"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 1000
        }" | jq -r '.choices[0].message.content'
}

# Calcul du baseline
calculate_baseline() {
    local days="${1:-7}"
    local cutoff=$(date -d "@$(( $(date +%s) - days * 86400 ))" +%s)
    
    # Collecter les métriques sur la période
    local metrics=$(find "$METRICS_STORAGE" -name "metrics_*.json" -type f | \
        while read -r file; do
            local timestamp=$(jq -r '.timestamp' "$file" 2>/dev/null)
            if [ -n "$timestamp" ] && [ "$timestamp" -gt "$cutoff" ]; then
                cat "$file"
            fi
        done | jq -s '.')
    
    # Calculer les moyennes et écarts-types
    jq '[.[] | .metrics] | {
        cpu_avg: [.[].cpu.usage] | add / length,
        memory_avg: [.[].memory.usage_percent] | add / length,
        disk_avg: [.[].disk.usage_percent] | add / length,
        load_avg: [.[].load."1min"] | add / length
    }' <<< "$metrics"
}
```

### Alertes intelligentes

**Système d'alerte contextuel** :
```bash
#!/bin/bash
# Alertes intelligentes avec contexte

send_intelligent_alert() {
    local anomaly_data="$1"
    local severity=$(echo "$anomaly_data" | jq -r '.severity')
    local description=$(echo "$anomaly_data" | jq -r '.description')
    local context=$(echo "$anomaly_data" | jq -r '.context')
    
    # Enrichir l'alerte avec IA
    local enriched_alert=$(ai_enrich_alert "$anomaly_data")
    
    # Déterminer les canaux selon la gravité
    case "$severity" in
        critical)
            send_alert_all_channels "$enriched_alert"
            ;;
        high)
            send_alert_email_slack "$enriched_alert"
            ;;
        medium)
            send_alert_slack "$enriched_alert"
            ;;
        low)
            log_alert "$enriched_alert"
            ;;
    esac
}

ai_enrich_alert() {
    local anomaly="$1"
    
    local prompt="Enrichis cette alerte système avec:
- Explication en langage clair
- Impact potentiel
- Actions recommandées immédiates
- Liens vers la documentation pertinente

Anomalie:
$anomaly

Format: JSON avec title, description, impact, actions, documentation"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 800
        }" | jq -r '.choices[0].message.content'
}
```

## Alertes contextuelles

### Groupement intelligent d'alertes

**Groupeur d'alertes** :
```bash
#!/bin/bash
# Groupement intelligent d'alertes

ALERT_BUFFER="/tmp/alert_buffer.json"
ALERT_WINDOW=300  # 5 minutes

group_alerts() {
    local new_alert="$1"
    
    # Charger les alertes récentes
    local recent_alerts=$(get_recent_alerts "$ALERT_WINDOW")
    
    # Analyser avec IA pour déterminer si c'est lié
    local grouping=$(ai_group_alerts "$new_alert" "$recent_alerts")
    
    local group_id=$(echo "$grouping" | jq -r '.group_id')
    
    if [ "$group_id" != "new" ]; then
        # Ajouter à un groupe existant
        add_to_group "$group_id" "$new_alert"
        echo "Alerte ajoutée au groupe: $group_id"
    else
        # Créer un nouveau groupe
        create_new_group "$new_alert"
        echo "Nouveau groupe créé"
    fi
}

ai_group_alerts() {
    local new_alert="$1"
    local recent_alerts="$2"
    
    local prompt="Détermine si cette nouvelle alerte est liée aux alertes récentes:
Nouvelle alerte:
$new_alert

Alertes récentes:
$recent_alerts

Si liée, retourne le group_id existant, sinon 'new'.
Considère:
- Cause racine commune
- Ressources affectées
- Temporalité (dans la même fenêtre)"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 200
        }" | jq -r '.choices[0].message.content'
}
```

## Dashboard intelligent

### Générateur de dashboard

**Dashboard adaptatif** :
```bash
#!/bin/bash
# Générateur de dashboard intelligent

generate_dashboard() {
    local metrics_period="${1:-3600}"  # 1 heure par défaut
    local dashboard_type="${2:-overview}"  # overview, detailed, executive
    
    # Collecter les données
    local metrics=$(get_metrics_for_period "$metrics_period")
    local anomalies=$(get_recent_anomalies)
    local predictions=$(get_predictions)
    
    # Générer le dashboard avec IA
    local dashboard=$(ai_generate_dashboard "$metrics" "$anomalies" "$predictions" "$dashboard_type")
    
    # Rendre le dashboard
    render_dashboard "$dashboard" "$dashboard_type"
}

ai_generate_dashboard() {
    local metrics="$1"
    local anomalies="$2"
    local predictions="$3"
    local type="$4"
    
    local prompt="Génère un dashboard $type avec:
- Métriques clés
- Graphiques suggérés
- Alertes actives
- Prédictions
- Recommandations

Données:
Métriques: $metrics
Anomalies: $anomalies
Prédictions: $predictions

Format: JSON avec sections et visualisations"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 2000
        }" | jq -r '.choices[0].message.content'
}

render_dashboard() {
    local dashboard_json="$1"
    local type="$2"
    
    case "$type" in
        overview)
            render_overview_dashboard "$dashboard_json"
            ;;
        detailed)
            render_detailed_dashboard "$dashboard_json"
            ;;
        executive)
            render_executive_dashboard "$dashboard_json"
            ;;
    esac
}

render_overview_dashboard() {
    local dashboard="$1"
    
    clear
    echo "╔════════════════════════════════════════╗"
    echo "║   DASHBOARD SYSTÈME - VUE D'ENSEMBLE   ║"
    echo "╚════════════════════════════════════════╝"
    echo
    
    # Métriques clés
    echo "=== MÉTRIQUES CLÉS ==="
    echo "$dashboard" | jq -r '.metrics[] | "\(.name): \(.value)\(.unit)"'
    echo
    
    # Alertes actives
    echo "=== ALERTES ACTIVES ==="
    echo "$dashboard" | jq -r '.alerts[] | "[\(.severity)] \(.message)"'
    echo
    
    # Recommandations
    echo "=== RECOMMANDATIONS ==="
    echo "$dashboard" | jq -r '.recommendations[] | "- \(.)"'
}
```

## Conclusion

Le monitoring intelligent transforme la surveillance système d'une activité réactive en une discipline proactive. En utilisant l'IA pour analyser les patterns, prédire les problèmes, et fournir des insights contextuels, nous créons des systèmes qui comprennent vraiment leur environnement et peuvent agir avant que les problèmes ne surviennent.

Cette approche réduit non seulement les temps d'arrêt, mais transforme également la façon dont nous gérons les infrastructures, passant d'une gestion par réaction à une gestion par anticipation.

Dans le chapitre suivant, nous explorerons les projets pratiques cloud, découvrant comment appliquer toutes ces techniques dans des environnements cloud réels.

