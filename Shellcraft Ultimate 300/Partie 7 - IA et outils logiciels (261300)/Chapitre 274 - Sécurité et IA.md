# Chapitre 274 - Sécurité et IA

## Table des matières
- [Introduction](#introduction)
- [Détection de menaces avec IA](#détection-de-menaces-avec-ia)
- [Analyse de vulnérabilités intelligente](#analyse-de-vulnérabilités-intelligente)
- [Audit de sécurité automatisé](#audit-de-sécurité-automatisé)
- [Réponse aux incidents assistée par IA](#réponse-aux-incidents-assistée-par-ia)
- [Conclusion](#conclusion)

## Introduction

La sécurité informatique moderne nécessite non seulement des outils de protection, mais aussi une capacité à comprendre les patterns, détecter les anomalies, et répondre rapidement aux menaces. L'intelligence artificielle apporte une dimension nouvelle à la sécurité : la capacité d'apprendre des attaques passées, de détecter des patterns subtils, et de s'adapter aux nouvelles menaces.

Imaginez la sécurité avec IA comme un garde du corps expérimenté : il ne se contente pas de surveiller, il comprend les comportements suspects, reconnaît les patterns d'attaque, et peut anticiper les menaces avant qu'elles ne se matérialisent.

## Détection de menaces avec IA

### Analyse de logs de sécurité

**Analyseur de logs avec IA** :
```bash
#!/bin/bash
# Analyse de logs de sécurité avec IA

SECURITY_LOGS="${SECURITY_LOGS:-/var/log/auth.log /var/log/secure}"
ANALYSIS_WINDOW="${ANALYSIS_WINDOW:-3600}"  # 1 heure

analyze_security_logs() {
    local log_files="$SECURITY_LOGS"
    local window_start=$(date -d "@$(( $(date +%s) - ANALYSIS_WINDOW ))" +%Y-%m-%d\ %H:%M:%S)
    
    # Collecter les événements récents
    local events=$(for log in $log_files; do
        if [ -f "$log" ]; then
            grep -E "(Failed|Invalid|Unauthorized|Attack|Breach)" "$log" | \
            awk -v start="$window_start" '$0 > start'
        fi
    done)
    
    # Analyser avec IA
    local analysis=$(ai_analyze_threats "$events")
    
    # Traiter les menaces détectées
    process_threats "$analysis"
}

ai_analyze_threats() {
    local events="$1"
    
    local prompt="Analyse ces logs de sécurité et détecte les menaces potentielles:
$events

Identifie:
- Tentatives d'intrusion
- Patterns d'attaque
- Comptes compromis
- Activités suspectes
- Niveau de risque (low, medium, high, critical)
- Actions recommandées

Format: JSON avec détails de chaque menace"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 2000
        }" | jq -r '.choices[0].message.content'
}

process_threats() {
    local analysis="$1"
    
    echo "$analysis" | jq -r '.threats[]' | \
    while IFS= read -r threat; do
        local risk=$(echo "$threat" | jq -r '.risk_level')
        local description=$(echo "$threat" | jq -r '.description')
        local actions=$(echo "$threat" | jq -r '.recommended_actions[]')
        
        echo "⚠ Menace détectée: $description (Risque: $risk)"
        
        case "$risk" in
            critical)
                # Actions immédiates
                execute_critical_response "$threat"
                ;;
            high)
                # Actions urgentes
                execute_high_priority_response "$threat"
                ;;
            *)
                # Logging et monitoring
                log_threat "$threat"
                ;;
        esac
    done
}
```

### Détection d'intrusion intelligente

**Système de détection d'intrusion** :
```bash
#!/bin/bash
# Détection d'intrusion avec IA

IDS_RULES="/etc/ids/rules.json"
IDS_LOG="/var/log/ids.log"

detect_intrusion() {
    local network_traffic="$1"
    local system_events="$2"
    
    # Analyser avec IA
    local detection=$(ai_detect_intrusion "$network_traffic" "$system_events")
    
    local threat_level=$(echo "$detection" | jq -r '.threat_level')
    
    if [[ "$threat_level" == "high" ]] || [[ "$threat_level" == "critical" ]]; then
        # Bloquer l'attaque
        block_attack "$detection"
        
        # Alerter
        send_security_alert "$detection"
        
        # Isoler si nécessaire
        if [[ "$threat_level" == "critical" ]]; then
            isolate_system "$detection"
        fi
    fi
}

ai_detect_intrusion() {
    local traffic="$1"
    local events="$2"
    
    local prompt="Analyse ce trafic réseau et ces événements système pour détecter des intrusions:
Trafic réseau:
$traffic

Événements système:
$events

Détecte:
- Tentatives d'intrusion
- Exploits
- Comportements anormaux
- Niveau de menace
- Source de l'attaque
- Actions de mitigation

Format: JSON"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 1500
        }" | jq -r '.choices[0].message.content'
}

block_attack() {
    local detection="$1"
    local source_ip=$(echo "$detection" | jq -r '.source_ip')
    local attack_type=$(echo "$detection" | jq -r '.attack_type')
    
    echo "Blocage de l'attaque depuis $source_ip (Type: $attack_type)"
    
    # Bloquer avec iptables/firewall
    if command -v iptables &> /dev/null; then
        iptables -A INPUT -s "$source_ip" -j DROP
        iptables -A OUTPUT -d "$source_ip" -j DROP
        echo "IP bloquée: $source_ip"
    fi
    
    # Logger
    echo "[$(date)] BLOCKED: $source_ip - $attack_type" >> "$IDS_LOG"
}
```

## Analyse de vulnérabilités intelligente

### Scanner de vulnérabilités

**Scanner avec IA** :
```bash
#!/bin/bash
# Scanner de vulnérabilités avec IA

scan_vulnerabilities() {
    local target="${1:-localhost}"
    local scan_type="${2:-comprehensive}"  # quick, standard, comprehensive
    
    echo "=== Scan de vulnérabilités ==="
    echo "Cible: $target"
    echo "Type: $scan_type"
    echo
    
    # Collecter les informations
    local system_info=$(collect_system_info "$target")
    local installed_packages=$(collect_packages "$target")
    local open_ports=$(scan_ports "$target")
    local services=$(scan_services "$target")
    
    # Analyser avec IA
    local vulnerabilities=$(ai_analyze_vulnerabilities \
        "$system_info" "$installed_packages" "$open_ports" "$services")
    
    # Générer le rapport
    generate_vulnerability_report "$vulnerabilities"
    
    # Recommandations de correction
    generate_fix_recommendations "$vulnerabilities"
}

ai_analyze_vulnerabilities() {
    local system_info="$1"
    local packages="$2"
    local ports="$3"
    local services="$4"
    
    local prompt="Analyse ces informations système pour identifier les vulnérabilités:
Système: $system_info
Packages installés: $packages
Ports ouverts: $ports
Services actifs: $services

Identifie:
- Vulnérabilités connues (CVE)
- Configurations dangereuses
- Services exposés inutilement
- Versions obsolètes
- Risques de sécurité
- Score de risque global

Format: JSON avec détails de chaque vulnérabilité"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 3000
        }" | jq -r '.choices[0].message.content'
}

generate_fix_recommendations() {
    local vulnerabilities="$1"
    
    local prompt="Génère des recommandations de correction pour ces vulnérabilités:
$vulnerabilities

Pour chaque vulnérabilité, fournis:
- Commande de correction
- Script de patch
- Vérification post-correction
- Impact de la correction

Format: JSON"

    local fixes=$(curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 2000
        }" | jq -r '.choices[0].message.content')
    
    echo "$fixes" > vulnerability_fixes.json
    
    # Générer les scripts de correction
    echo "$fixes" | jq -r '.fixes[] | .script' | \
    while IFS= read -r script; do
        local fix_num=$((++fix_counter))
        echo "$script" > "fix_${fix_num}.sh"
        chmod +x "fix_${fix_num}.sh"
    done
}
```

## Audit de sécurité automatisé

### Auditeur de sécurité

**Audit automatisé** :
```bash
#!/bin/bash
# Audit de sécurité automatisé avec IA

perform_security_audit() {
    local audit_scope="${1:-full}"  # full, network, system, application
    
    echo "=== Audit de sécurité ==="
    echo "Portée: $audit_scope"
    echo
    
    # Collecter les données selon la portée
    local audit_data=$(collect_audit_data "$audit_scope")
    
    # Analyser avec IA
    local findings=$(ai_security_audit "$audit_data" "$audit_scope")
    
    # Générer le rapport
    generate_audit_report "$findings"
    
    # Score de sécurité
    calculate_security_score "$findings"
}

collect_audit_data() {
    local scope="$1"
    local data="{}"
    
    case "$scope" in
        full|system)
            data=$(jq -n \
                --arg users "$(get_user_accounts)" \
                --arg permissions "$(get_permissions)" \
                --arg services "$(get_services)" \
                '{users: $users, permissions: $permissions, services: $services}')
            ;;
        network)
            data=$(jq -n \
                --arg firewall "$(get_firewall_rules)" \
                --arg ports "$(get_open_ports)" \
                --arg connections "$(get_network_connections)" \
                '{firewall: $firewall, ports: $ports, connections: $connections}')
            ;;
        application)
            data=$(jq -n \
                --arg configs "$(get_app_configs)" \
                --arg logs "$(get_app_logs)" \
                '{configs: $configs, logs: $logs}')
            ;;
    esac
    
    echo "$data"
}

ai_security_audit() {
    local data="$1"
    local scope="$2"
    
    local prompt="Effectue un audit de sécurité $scope sur ces données:
$data

Évalue:
- Conformité aux standards de sécurité
- Bonnes pratiques respectées
- Problèmes de sécurité identifiés
- Recommandations d'amélioration
- Score de sécurité (0-100)

Format: JSON structuré"

    curl -s https://api.openai.com/v1/chat/completions \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer ${OPENAI_API_KEY}" \
        -d "{
            \"model\": \"gpt-4\",
            \"messages\": [{\"role\": \"user\", \"content\": \"$prompt\"}],
            \"max_tokens\": 3000
        }" | jq -r '.choices[0].message.content'
}
```

## Réponse aux incidents assistée par IA

### Répondeur d'incidents

**Système de réponse aux incidents** :
```bash
#!/bin/bash
# Réponse aux incidents assistée par IA

respond_to_incident() {
    local incident_type="$1"
    local incident_data="$2"
    
    echo "=== Réponse à l'incident ==="
    echo "Type: $incident_type"
    echo
    
    # Analyser l'incident avec IA
    local analysis=$(ai_analyze_incident "$incident_type" "$incident_data")
    
    # Générer le plan de réponse
    local response_plan=$(generate_response_plan "$analysis")
    
    # Exécuter le plan
    execute_response_plan "$response_plan"
    
    # Documenter
    document_incident "$incident_type" "$incident_data" "$response_plan"
}

ai_analyze_incident() {
    local type="$1"
    local data="$2"
    
    local prompt="Analyse cet incident de sécurité et détermine:
- Gravité de l'incident
- Cause probable
- Étendue de l'impact
- Actions immédiates nécessaires
- Plan de containment
- Plan de récupération

Type d'incident: $type
Données: $data

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

generate_response_plan() {
    local analysis="$1"
    
    local severity=$(echo "$analysis" | jq -r '.severity')
    local containment=$(echo "$analysis" | jq -r '.containment_plan')
    local recovery=$(echo "$analysis" | jq -r '.recovery_plan')
    
    cat > response_plan.json << EOF
{
    "severity": "$severity",
    "containment": $containment,
    "recovery": $recovery,
    "generated_at": "$(date -Iseconds)"
}
EOF
    
    cat response_plan.json
}

execute_response_plan() {
    local plan="$1"
    
    echo "=== Exécution du plan de réponse ==="
    
    # Phase 1: Containment
    echo "Phase 1: Containment..."
    echo "$plan" | jq -r '.containment.steps[]' | \
    while IFS= read -r step; do
        echo "Exécution: $step"
        execute_step "$step"
    done
    
    # Phase 2: Investigation
    echo "Phase 2: Investigation..."
    investigate_incident "$plan"
    
    # Phase 3: Récupération
    echo "Phase 3: Récupération..."
    echo "$plan" | jq -r '.recovery.steps[]' | \
    while IFS= read -r step; do
        echo "Exécution: $step"
        execute_step "$step"
    done
}
```

## Conclusion

La sécurité avec IA transforme la défense informatique d'une activité réactive en une discipline proactive et intelligente. En utilisant l'IA pour analyser les patterns, détecter les anomalies, et générer des réponses adaptées, nous créons des systèmes de sécurité qui s'améliorent continuellement et s'adaptent aux nouvelles menaces.

Cette approche ne remplace pas les experts en sécurité, mais les amplifie, leur permettant de se concentrer sur les décisions stratégiques tandis que l'IA gère la détection et l'analyse des menaces.

Dans le chapitre suivant, nous explorerons l'optimisation de workflow, découvrant comment utiliser l'IA pour améliorer continuellement vos processus d'automatisation.

