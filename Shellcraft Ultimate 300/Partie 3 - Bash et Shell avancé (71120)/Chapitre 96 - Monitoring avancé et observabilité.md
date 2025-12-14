# Chapitre 96 - Monitoring avancé et observabilité

> "L'observabilité n'est pas un ensemble d'outils : c'est l'art de comprendre son système comme un organisme vivant, où chaque métrique est un battement de cœur, chaque log un murmure du système, chaque alerte un signal d'alarme." - Observability Sage

## Introduction : L'organisme vivant observé

Imaginez-vous en tant que médecin d'un patient numérique complexe : chaque métrique est un signe vital, chaque log un symptôme, chaque alerte un signal d'urgence. Le monitoring avancé en Bash transforme l'observation système d'un ensemble de commandes éparses en une médecine prédictive où l'on anticipe les maladies avant qu'elles ne se déclarent.

Dans ce chapitre, nous construirons le système nerveux sensitif de nos architectures : métriques intelligentes, agrégation de logs, tracing distribué, alertes proactives, et tableaux de bord qui transforment les données brutes en intelligence actionable.

## Section 1 : Collecte et traitement des métriques

### 1.1 Framework de métriques temps réel

Système de collecte et analyse de métriques avec calculs en continu :

```bash
#!/bin/bash

# Framework de métriques temps réel
echo "=== Framework de métriques temps réel ==="

# Real-Time Metrics Framework
MetricsFramework() {
    local self="$1"
    
    declare -A $self._metrics_definitions
    declare -A $self._metrics_data
    declare -A $self._metrics_aggregations
    declare -A $self._alert_conditions
    
    # Définition d'une métrique
    $self.define_metric() {
        local metric_name="$1"
        local metric_type="$2"  # gauge, counter, histogram, summary
        local description="$3"
        local unit="${4:-}"
        local labels="${5:-}"
        
        $self._metrics_definitions["${metric_name}_type"]="$metric_type"
        $self._metrics_definitions["${metric_name}_description"]="$description"
        $self._metrics_definitions["${metric_name}_unit"]="$unit"
        $self._metrics_definitions["${metric_name}_labels"]="$labels"
        $self._metrics_definitions["${metric_name}_created"]="$(date +%s)"
        
        # Initialisation des données
        $self._metrics_data["${metric_name}_values"]=""
        $self._metrics_data["${metric_name}_timestamps"]=""
        $self._metrics_data["${metric_name}_count"]=0
        
        echo "✓ Métrique définie: $metric_name ($metric_type)"
    }
    
    # Enregistrement d'une valeur métrique
    $self.record_metric() {
        local metric_name="$1"
        local value="$2"
        local labels="${3:-}"
        
        if [[ -z "${$self._metrics_definitions[${metric_name}_type]}" ]]; then
            echo "❌ Métrique introuvable: $metric_name" >&2
            return 1
        fi
        
        local timestamp
        timestamp="$(date +%s.%N)"
        
        local metric_type="${$self._metrics_definitions[${metric_name}_type]}"
        
        # Validation selon le type
        case "$metric_type" in
            counter)
                # Les compteurs ne peuvent que croître
                local last_value
                last_value="$($self.get_latest_value "$metric_name")"
                if [[ -n "$last_value" ]] && (( $(echo "$value < $last_value" | bc -l) )); then
                    echo "❌ Valeur compteur invalide: $value < $last_value" >&2
                    return 1
                fi
                ;;
                
            gauge|histogram|summary)
                # Validation numérique
                if ! [[ "$value" =~ ^-?[0-9]*\.?[0-9]+$ ]]; then
                    echo "❌ Valeur numérique attendue: $value" >&2
                    return 1
                fi
                ;;
        esac
        
        # Stockage de la valeur
        local current_values="${$self._metrics_data[${metric_name}_values]}"
        local current_timestamps="${$self._metrics_data[${metric_name}_timestamps]}"
        
        $self._metrics_data["${metric_name}_values"]="${current_values:+$current_values;}$value"
        $self._metrics_data["${metric_name}_timestamps"]="${current_timestamps:+$current_timestamps;}$timestamp"
        $self._metrics_data["${metric_name}_count"]=$(( ${$self._metrics_data[${metric_name}_count]} + 1 ))
        
        # Calcul des agrégations en temps réel
        $self._update_aggregations "$metric_name"
        
        # Vérification des conditions d'alerte
        $self._check_alert_conditions "$metric_name" "$value" "$labels"
        
        echo "✓ Valeur enregistrée: $metric_name = $value"
    }
    
    # Récupération de la dernière valeur
    $self.get_latest_value() {
        local metric_name="$1"
        
        local values="${$self._metrics_data[${metric_name}_values]}"
        
        if [[ -z "$values" ]]; then
            return
        fi
        
        # Retourne la dernière valeur
        echo "$values" | awk -F';' '{print $NF}'
    }
    
    # Calcul des agrégations
    $self._update_aggregations() {
        local metric_name="$1"
        
        local values="${$self._metrics_data[${metric_name}_values]}"
        local count="${$self._metrics_data[${metric_name}_count]}"
        
        if (( count == 0 )); then
            return
        fi
        
        # Conversion en tableau
        IFS=';' read -ra value_array <<< "$values"
        
        # Calculs statistiques de base
        local sum=0 min=${value_array[0]} max=${value_array[0]}
        
        for value in "${value_array[@]}"; do
            sum=$(echo "$sum + $value" | bc -l)
            min=$(echo "if ($value < $min) $value else $min" | bc -l)
            max=$(echo "if ($value > $max) $value else $max" | bc -l)
        done
        
        local avg=$(echo "scale=4; $sum / $count" | bc -l)
        
        # Calcul de l'écart-type
        local variance=0
        for value in "${value_array[@]}"; do
            local diff=$(echo "$value - $avg" | bc -l)
            local squared_diff=$(echo "$diff * $diff" | bc -l)
            variance=$(echo "$variance + $squared_diff" | bc -l)
        done
        local stddev=$(echo "scale=4; sqrt($variance / $count)" | bc -l)
        
        # Stockage des agrégations
        $self._metrics_aggregations["${metric_name}_sum"]="$sum"
        $self._metrics_aggregations["${metric_name}_avg"]="$avg"
        $self._metrics_aggregations["${metric_name}_min"]="$min"
        $self._metrics_aggregations["${metric_name}_max"]="$max"
        $self._metrics_aggregations["${metric_name}_stddev"]="$stddev"
        $self._metrics_aggregations["${metric_name}_count"]="$count"
    }
    
    # Définition d'une condition d'alerte
    $self.define_alert_condition() {
        local alert_name="$1"
        local metric_name="$2"
        local condition="$3"  # gt, lt, eq, ne, rate_gt, rate_lt
        local threshold="$4"
        local duration="${5:-0}"  # durée en secondes pour que l'alerte se déclenche
        local severity="${6:-warning}"  # info, warning, error, critical
        local message="$7"
        
        $self._alert_conditions["${alert_name}_metric"]="$metric_name"
        $self._alert_conditions["${alert_name}_condition"]="$condition"
        $self._alert_conditions["${alert_name}_threshold"]="$threshold"
        $self._alert_conditions["${alert_name}_duration"]="$duration"
        $self._alert_conditions["${alert_name}_severity"]="$severity"
        $self._alert_conditions["${alert_name}_message"]="$message"
        $self._alert_conditions["${alert_name}_active_since"]=""
        $self._alert_conditions["${alert_name}_last_triggered"]=""
        
        echo "✓ Condition d'alerte définie: $alert_name ($severity)"
    }
    
    # Vérification des conditions d'alerte
    $self._check_alert_conditions() {
        local metric_name="$1"
        local current_value="$2"
        local labels="$3"
        
        for alert_key in "${!$self._alert_conditions[@]}"; do
            if [[ "$alert_key" =~ _metric$ && "${$self._alert_conditions[$alert_key]}" == "$metric_name" ]]; then
                local alert_name="${alert_key%_metric}"
                $self._evaluate_alert_condition "$alert_name" "$current_value"
            fi
        done
    }
    
    # Évaluation d'une condition d'alerte
    $self._evaluate_alert_condition() {
        local alert_name="$1"
        local current_value="$2"
        
        local condition="${$self._alert_conditions[${alert_name}_condition]}"
        local threshold="${$self._alert_conditions[${alert_name}_threshold]}"
        local duration="${$self._alert_conditions[${alert_name}_duration]}"
        local severity="${$self._alert_conditions[${alert_name}_severity]}"
        local message="${$self._alert_conditions[${alert_name}_message]}"
        
        local alert_triggered=false
        
        case "$condition" in
            gt)
                (( $(echo "$current_value > $threshold" | bc -l) )) && alert_triggered=true
                ;;
                
            lt)
                (( $(echo "$current_value < $threshold" | bc -l) )) && alert_triggered=true
                ;;
                
            eq)
                (( $(echo "$current_value == $threshold" | bc -l) )) && alert_triggered=true
                ;;
                
            ne)
                (( $(echo "$current_value != $threshold" | bc -l) )) && alert_triggered=true
                ;;
                
            rate_gt)
                # Calcul du taux de variation (simplifié)
                local metric_name="${$self._alert_conditions[${alert_name}_metric]}"
                local rate
                rate="$($self._calculate_rate "$metric_name")"
                (( $(echo "$rate > $threshold" | bc -l) )) && alert_triggered=true
                ;;
        esac
        
        if [[ "$alert_triggered" == "true" ]]; then
            local current_time
            current_time="$(date +%s)"
            local active_since="${$self._alert_conditions[${alert_name}_active_since]}"
            
            if [[ -z "$active_since" ]]; then
                # Première détection
                $self._alert_conditions["${alert_name}_active_since"]="$current_time"
                
                if (( duration == 0 )); then
                    # Alerte immédiate
                    $self._trigger_alert "$alert_name" "$severity" "$message"
                fi
            else
                # Vérification de la durée
                local active_duration=$(( current_time - active_since ))
                
                if (( active_duration >= duration )); then
                    $self._trigger_alert "$alert_name" "$severity" "$message"
                    $self._alert_conditions["${alert_name}_last_triggered"]="$current_time"
                fi
            fi
        else
            # Reset de l'état d'alerte
            $self._alert_conditions["${alert_name}_active_since"]=""
        fi
    }
    
    # Déclenchement d'une alerte
    $self._trigger_alert() {
        local alert_name="$1"
        local severity="$2"
        local message="$3"
        
        local timestamp
        timestamp="$(date '+%Y-%m-%d %H:%M:%S')"
        
        echo "🚨 [$timestamp] ALERTE $severity: $alert_name"
        echo "   $message"
        
        # Ici on pourrait envoyer des notifications (email, Slack, etc.)
        # $self._send_notification "$severity" "$alert_name" "$message"
    }
    
    # Calcul du taux de variation
    $self._calculate_rate() {
        local metric_name="$1"
        
        local values="${$self._metrics_data[${metric_name}_values]}"
        local timestamps="${$self._metrics_data[${metric_name}_timestamps]}"
        
        if [[ -z "$values" ]]; then
            echo "0"
            return
        fi
        
        # Prendre les 5 dernières valeurs pour calculer la tendance
        local value_array
        IFS=';' read -ra value_array <<< "$values"
        local timestamp_array
        IFS=';' read -ra timestamp_array <<< "$timestamps"
        
        local len="${#value_array[@]}"
        if (( len < 2 )); then
            echo "0"
            return
        fi
        
        # Calcul de la régression linéaire simple
        local n=5
        (( len < n )) && n=$len
        
        local sum_x=0 sum_y=0 sum_xy=0 sum_x2=0
        
        for ((i = len - n; i < len; i++)); do
            local x=$(( i - (len - n) ))
            local y="${value_array[$i]}"
            
            sum_x=$(( sum_x + x ))
            sum_y=$(echo "$sum_y + $y" | bc -l)
            sum_xy=$(echo "$sum_xy + ($x * $y)" | bc -l)
            sum_x2=$(( sum_x2 + (x * x) ))
        done
        
        local slope=$(echo "($n * $sum_xy - $sum_x * $sum_y) / ($n * $sum_x2 - $sum_x * $sum_x)" | bc -l)
        
        echo "$slope"
    }
    
    # Collecte automatique de métriques système
    $self.start_system_metrics_collection() {
        local interval="${1:-30}"
        
        echo "=== COLLECTE MÉTRIQUES SYSTÈME ==="
        echo "Intervalle: ${interval}s"
        echo "Arrêt: Ctrl+C"
        echo
        
        while true; do
            local timestamp
            timestamp="$(date '+%H:%M:%S')"
            
            # Métriques CPU
            local cpu_usage
            cpu_usage="$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')"
            $self.record_metric "system_cpu_usage" "$cpu_usage"
            
            # Métriques mémoire
            local mem_total mem_used
            mem_total="$(free | grep Mem | awk '{print $2}')"
            mem_used="$(free | grep Mem | awk '{print $3}')"
            local mem_usage_percent=$(echo "scale=2; $mem_used * 100 / $mem_total" | bc)
            $self.record_metric "system_memory_usage_percent" "$mem_usage_percent"
            
            # Métriques disque
            local disk_usage
            disk_usage="$(df / | tail -1 | awk '{print $5}' | sed 's/%//')"
            $self.record_metric "system_disk_usage_percent" "$disk_usage"
            
            # Métriques réseau
            local rx_bytes tx_bytes
            rx_bytes="$(cat /proc/net/dev | grep -E "^[[:space:]]*eth0|^[[:space:]]*enp" | awk '{print $2}')"
            tx_bytes="$(cat /proc/net/dev | grep -E "^[[:space:]]*eth0|^[[:space:]]*enp" | awk '{print $10}')"
            $self.record_metric "network_rx_bytes" "${rx_bytes:-0}"
            $self.record_metric "network_tx_bytes" "${tx_bytes:-0}"
            
            # Métriques processus
            local process_count
            process_count="$(ps aux | wc -l)"
            $self.record_metric "system_process_count" "$process_count"
            
            echo "[$timestamp] Métriques collectées"
            
            sleep "$interval"
        done
    }
    
    # Génération de rapport de métriques
    $self.generate_metrics_report() {
        local output_file="${1:-metrics_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DE MÉTRIQUES TEMPS RÉEL"
            echo "==============================="
            echo "Généré le: $(date)"
            echo
            
            echo "MÉTRIQUES DÉFINIES"
            echo "=================="
            
            for metric_key in "${!$self._metrics_definitions[@]}"; do
                if [[ "$metric_key" =~ _type$ ]]; then
                    local metric_name="${metric_key%_type}"
                    local metric_type="${$self._metrics_definitions[$metric_key]}"
                    local description="${$self._metrics_definitions[${metric_name}_description]}"
                    local unit="${$self._metrics_definitions[${metric_name}_unit]}"
                    local count="${$self._metrics_data[${metric_name}_count]}"
                    
                    echo "Métrique: $metric_name"
                    echo "  Type: $metric_type"
                    echo "  Description: $description"
                    echo "  Unité: ${unit:-N/A}"
                    echo "  Valeurs collectées: $count"
                    
                    if (( count > 0 )); then
                        local latest_value
                        latest_value="$($self.get_latest_value "$metric_name")"
                        echo "  Dernière valeur: $latest_value"
                        
                        # Statistiques
                        local avg="${$self._metrics_aggregations[${metric_name}_avg]}"
                        local min="${$self._metrics_aggregations[${metric_name}_min]}"
                        local max="${$self._metrics_aggregations[${metric_name}_max]}"
                        
                        if [[ -n "$avg" ]]; then
                            echo "  Moyenne: $avg"
                            echo "  Min/Max: $min / $max"
                        fi
                    fi
                    
                    echo
                fi
            done
            
            echo "CONDITIONS D'ALERTE"
            echo "==================="
            
            for alert_key in "${!$self._alert_conditions[@]}"; do
                if [[ "$alert_key" =~ _metric$ ]]; then
                    local alert_name="${alert_key%_metric}"
                    local metric="${$self._alert_conditions[$alert_key]}"
                    local condition="${$self._alert_conditions[${alert_name}_condition]}"
                    local threshold="${$self._alert_conditions[${alert_name}_threshold]}"
                    local severity="${$self._alert_conditions[${alert_name}_severity]}"
                    local last_triggered="${$self._alert_conditions[${alert_name}_last_triggered]}"
                    
                    echo "Alerte: $alert_name"
                    echo "  Métrique: $metric"
                    echo "  Condition: $condition $threshold"
                    echo "  Sévérité: $severity"
                    
                    if [[ -n "$last_triggered" ]]; then
                        echo "  Dernier déclenchement: $(date -d "@$last_triggered" '+%Y-%m-%d %H:%M:%S')"
                    else
                        echo "  Statut: Inactive"
                    fi
                    
                    echo
                fi
            done
            
            echo "APERÇU DES DONNÉES RÉCENTES"
            echo "==========================="
            
            for metric_key in "${!$self._metrics_definitions[@]}"; do
                if [[ "$metric_key" =~ _type$ ]]; then
                    local metric_name="${metric_key%_type}"
                    local values="${$self._metrics_data[${metric_name}_values]}"
                    local timestamps="${$self._metrics_data[${metric_name}_timestamps]}"
                    
                    if [[ -n "$values" ]]; then
                        echo "$metric_name (5 dernières valeurs):"
                        
                        # Afficher les 5 dernières valeurs
                        local value_array timestamp_array
                        IFS=';' read -ra value_array <<< "$values"
                        IFS=';' read -ra timestamp_array <<< "$timestamps"
                        
                        local len="${#value_array[@]}"
                        local start=$(( len > 5 ? len - 5 : 0 ))
                        
                        for ((i = start; i < len; i++)); do
                            local ts="${timestamp_array[$i]}"
                            local val="${value_array[$i]}"
                            printf "  %s: %s\n" "$(date -d "@$ts" '+%H:%M:%S')" "$val"
                        done
                        
                        echo
                    fi
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Export des métriques au format Prometheus
    $self.export_prometheus_metrics() {
        local output_file="${1:-metrics.prom}"
        
        {
            echo "# Métriques exportées depuis Bash Metrics Framework"
            echo "# Généré le $(date)"
            echo
            
            for metric_key in "${!$self._metrics_definitions[@]}"; do
                if [[ "$metric_key" =~ _type$ ]]; then
                    local metric_name="${metric_key%_type}"
                    local metric_type="${$self._metrics_definitions[$metric_key]}"
                    local description="${$self._metrics_definitions[${metric_name}_description]}"
                    local unit="${$self._metrics_definitions[${metric_name}_unit]}"
                    local latest_value
                    latest_value="$($self.get_latest_value "$metric_name")"
                    
                    if [[ -n "$latest_value" ]]; then
                        # En-tête HELP
                        echo "# HELP ${metric_name} ${description}"
                        
                        # En-tête TYPE
                        echo "# TYPE ${metric_name} ${metric_type}"
                        
                        # Valeur
                        if [[ -n "$unit" ]]; then
                            echo "${metric_name}{unit=\"${unit}\"} $latest_value"
                        else
                            echo "$metric_name $latest_value"
                        fi
                        
                        echo
                    fi
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Métriques exportées au format Prometheus: $output_file"
    }
}

# Définition des conditions d'alerte système
define_system_alerts() {
    local metrics="$1"
    
    # Alertes CPU
    $metrics.define_alert_condition "high_cpu_usage" "system_cpu_usage" "gt" "80" "300" "warning" "Utilisation CPU supérieure à 80% pendant 5 minutes"
    $metrics.define_alert_condition "critical_cpu_usage" "system_cpu_usage" "gt" "95" "60" "critical" "Utilisation CPU supérieure à 95% pendant 1 minute"
    
    # Alertes mémoire
    $metrics.define_alert_condition "high_memory_usage" "system_memory_usage_percent" "gt" "85" "300" "warning" "Utilisation mémoire supérieure à 85% pendant 5 minutes"
    $metrics.define_alert_condition "critical_memory_usage" "system_memory_usage_percent" "gt" "95" "60" "critical" "Utilisation mémoire supérieure à 95% pendant 1 minute"
    
    # Alertes disque
    $metrics.define_alert_condition "high_disk_usage" "system_disk_usage_percent" "gt" "90" "600" "warning" "Utilisation disque supérieure à 90% pendant 10 minutes"
    $metrics.define_alert_condition "critical_disk_usage" "system_disk_usage_percent" "gt" "98" "60" "critical" "Utilisation disque supérieure à 98% pendant 1 minute"
    
    # Alertes réseau (simulé)
    $metrics.define_alert_condition "network_anomaly" "network_rx_bytes" "rate_gt" "1000000" "60" "info" "Trafic réseau anormalement élevé"
    
    # Alertes processus
    $metrics.define_alert_condition "too_many_processes" "system_process_count" "gt" "1000" "300" "warning" "Nombre de processus anormalement élevé"
}

# Démonstration du framework de métriques
echo "--- Framework de métriques temps réel ---"

MetricsFramework "metrics_system"

# Définition des métriques
metrics_system.define_metric "system_cpu_usage" "gauge" "Pourcentage d'utilisation CPU" "percent"
metrics_system.define_metric "system_memory_usage_percent" "gauge" "Pourcentage d'utilisation mémoire" "percent"
metrics_system.define_metric "system_disk_usage_percent" "gauge" "Pourcentage d'utilisation disque" "percent"
metrics_system.define_metric "network_rx_bytes" "counter" "Octets reçus sur le réseau"
metrics_system.define_metric "network_tx_bytes" "counter" "Octets transmis sur le réseau"
metrics_system.define_metric "system_process_count" "gauge" "Nombre de processus système"
metrics_system.define_metric "request_count" "counter" "Nombre de requêtes traitées"
metrics_system.define_metric "response_time" "histogram" "Temps de réponse en millisecondes" "ms"

# Définition des alertes
define_system_alerts "metrics_system"

echo
echo "--- Enregistrement de valeurs de test ---"

# Simulation de métriques système
for i in {1..10}; do
    cpu_usage=$(( 10 + RANDOM % 80 ))
    mem_usage=$(( 20 + RANDOM % 70 ))
    disk_usage=$(( 15 + RANDOM % 80 ))
    process_count=$(( 100 + RANDOM % 400 ))
    
    metrics_system.record_metric "system_cpu_usage" "$cpu_usage"
    metrics_system.record_metric "system_memory_usage_percent" "$mem_usage"
    metrics_system.record_metric "system_disk_usage_percent" "$disk_usage"
    metrics_system.record_metric "system_process_count" "$process_count"
    
    # Simulation de requêtes
    request_count=$(( 1 + RANDOM % 5 ))
    for ((j=1; j<=request_count; j++)); do
        response_time=$(( 50 + RANDOM % 200 ))
        metrics_system.record_metric "request_count" "1"
        metrics_system.record_metric "response_time" "$response_time"
    done
    
    echo "Itération $i: CPU=${cpu_usage}%, MEM=${mem_usage}%, DISK=${disk_usage}%, PROC=$process_count"
    
    sleep 0.1
done

echo
echo "--- Génération de rapport ---"
metrics_system.generate_metrics_report

echo
echo "--- Export Prometheus ---"
metrics_system.export_prometheus_metrics

# Nettoyage
rm -f metrics_report_*.txt metrics.prom
```

### 1.2 Système d'agrégation et corrélation de logs

Agrégateur intelligent de logs avec analyse de corrélation et recherche :

```bash
#!/bin/bash

# Système d'agrégation et corrélation de logs
echo "=== Système d'agrégation et corrélation de logs ==="

# Log Aggregation and Correlation System
LogAggregationSystem() {
    local self="$1"
    
    declare -A $self._log_sources
    declare -A $self._log_patterns
    declare -A $self._correlation_rules
    declare -A $self._log_storage
    declare -A $self._alert_rules
    
    # Enregistrement d'une source de logs
    $self.register_log_source() {
        local source_name="$1"
        local source_type="$2"  # file, journald, syslog, command
        local source_path="$3"
        local format="${4:-text}"  # text, json, csv, keyvalue
        local tags="${5:-}"
        
        $self._log_sources["${source_name}_type"]="$source_type"
        $self._log_sources["${source_name}_path"]="$source_path"
        $self._log_sources["${source_name}_format"]="$format"
        $self._log_sources["${source_name}_tags"]="$tags"
        $self._log_sources["${source_name}_last_read"]=""
        
        # Initialisation du stockage
        $self._log_storage["${source_name}_logs"]=""
        $self._log_storage["${source_name}_count"]=0
        
        echo "✓ Source de logs enregistrée: $source_name ($source_type)"
    }
    
    # Définition d'un pattern de log
    $self.define_log_pattern() {
        local pattern_name="$1"
        local regex="$2"
        local severity="${3:-info}"  # debug, info, warning, error, critical
        local category="${4:-general}"
        local extract_fields="${5:-}"  # liste des champs à extraire
        
        $self._log_patterns["${pattern_name}_regex"]="$regex"
        $self._log_patterns["${pattern_name}_severity"]="$severity"
        $self._log_patterns["${pattern_name}_category"]="$category"
        $self._log_patterns["${pattern_name}_fields"]="$extract_fields"
        
        echo "✓ Pattern de log défini: $pattern_name ($severity)"
    }
    
    # Définition d'une règle de corrélation
    $self.define_correlation_rule() {
        local rule_name="$1"
        local conditions="$2"  # pattern1 AND pattern2 WITHIN 5min
        local action="$3"  # alert, group, ignore
        local description="$4"
        
        $self._correlation_rules["${rule_name}_conditions"]="$conditions"
        $self._correlation_rules["${rule_name}_action"]="$action"
        $self._correlation_rules["${rule_name}_description"]="$description"
        
        echo "✓ Règle de corrélation définie: $rule_name"
    }
    
    # Collecte des logs depuis une source
    $self.collect_logs() {
        local source_name="$1"
        local max_lines="${2:-100}"
        
        local source_type="${$self._log_sources[${source_name}_type]}"
        local source_path="${$self._log_sources[${source_name}_path]}"
        
        if [[ -z "$source_type" ]]; then
            echo "❌ Source introuvable: $source_name" >&2
            return 1
        fi
        
        echo "Collecte depuis $source_name ($source_type)..."
        
        local logs=""
        
        case "$source_type" in
            file)
                if [[ -f "$source_path" ]]; then
                    logs="$(tail -n "$max_lines" "$source_path")"
                else
                    echo "Fichier introuvable: $source_path" >&2
                    return 1
                fi
                ;;
                
            journald)
                logs="$(journalctl -n "$max_lines" --no-pager 2>/dev/null)"
                ;;
                
            syslog)
                logs="$(tail -n "$max_lines" /var/log/syslog 2>/dev/null || tail -n "$max_lines" /var/log/messages 2>/dev/null)"
                ;;
                
            command)
                logs="$(eval "$source_path" 2>/dev/null | tail -n "$max_lines")"
                ;;
                
            *)
                echo "Type de source non supporté: $source_type" >&2
                return 1
                ;;
        esac
        
        if [[ -z "$logs" ]]; then
            echo "Aucun log collecté"
            return 0
        fi
        
        # Traitement des logs
        local processed=0
        while IFS= read -r log_line; do
            if [[ -n "$log_line" ]]; then
                $self._process_log_line "$source_name" "$log_line"
                ((processed++))
            fi
        done <<< "$logs"
        
        echo "✓ $processed logs traités depuis $source_name"
        
        # Mise à jour du timestamp de dernière lecture
        $self._log_sources["${source_name}_last_read"]="$(date +%s)"
    }
    
    # Traitement d'une ligne de log
    $self._process_log_line() {
        local source_name="$1"
        local log_line="$2"
        
        local timestamp
        timestamp="$(date +%s)"
        
        # Parsing selon le format
        local parsed_log
        parsed_log="$($self._parse_log_line "$source_name" "$log_line")"
        
        # Application des patterns
        local matched_patterns=""
        for pattern_key in "${!$self._log_patterns[@]}"; do
            if [[ "$pattern_key" =~ _regex$ ]]; then
                local pattern_name="${pattern_key%_regex}"
                local regex="${$self._log_patterns[$pattern_key]}"
                
                if echo "$log_line" | grep -qE "$regex"; then
                    matched_patterns="${matched_patterns:+$matched_patterns;}$pattern_name"
                    
                    # Extraction des champs si définis
                    local fields="${$self._log_patterns[${pattern_name}_fields]}"
                    if [[ -n "$fields" ]]; then
                        $self._extract_fields "$pattern_name" "$log_line" "$fields"
                    fi
                fi
            fi
        done
        
        # Stockage du log
        local log_entry="$timestamp|$source_name|$log_line|$matched_patterns|$parsed_log"
        local current_logs="${$self._log_storage[${source_name}_logs]}"
        
        $self._log_storage["${source_name}_logs"]="${current_logs:+$current_logs$'\n'}$log_entry"
        $self._log_storage["${source_name}_count"]=$(( ${$self._log_storage[${source_name}_count]} + 1 ))
        
        # Vérification des règles de corrélation
        $self._check_correlation_rules "$source_name" "$log_line" "$matched_patterns"
        
        # Vérification des règles d'alerte
        $self._check_alert_rules "$source_name" "$log_line" "$matched_patterns"
    }
    
    # Parsing d'une ligne de log
    $self._parse_log_line() {
        local source_name="$1"
        local log_line="$2"
        
        local format="${$self._log_sources[${source_name}_format]}"
        
        case "$format" in
            json)
                # Tentative d'extraction JSON basique
                echo "$log_line" | grep -o '"[^"]*":[^,}]*' | tr '\n' ' ' || echo "unparsed"
                ;;
                
            keyvalue)
                # Format clé=valeur
                echo "$log_line" | sed 's/ /;/g' || echo "unparsed"
                ;;
                
            text|*)
                # Texte brut
                echo "raw_text"
                ;;
        esac
    }
    
    # Extraction de champs
    $self._extract_fields() {
        local pattern_name="$1"
        local log_line="$2"
        local fields="$3"
        
        echo "Extraction champs pour $pattern_name: $fields"
        
        # Ici on pourrait implémenter une extraction plus sophistiquée
        # Pour l'exemple, on simule une extraction simple
        for field in $fields; do
            case "$field" in
                timestamp)
                    # Extraction simplifiée
                    echo "  $field: $(date +%s)"
                    ;;
                    
                level)
                    if echo "$log_line" | grep -qi "error"; then
                        echo "  $field: ERROR"
                    elif echo "$log_line" | grep -qi "warn"; then
                        echo "  $field: WARNING"
                    else
                        echo "  $field: INFO"
                    fi
                    ;;
                    
                message)
                    echo "  $field: $(echo "$log_line" | cut -d: -f2-)"
                    ;;
            esac
        done
    }
    
    # Vérification des règles de corrélation
    $self._check_correlation_rules() {
        local source_name="$1"
        local log_line="$2"
        local matched_patterns="$3"
        
        for rule_key in "${!$self._correlation_rules[@]}"; do
            if [[ "$rule_key" =~ _conditions$ ]]; then
                local rule_name="${rule_key%_conditions}"
                local conditions="${$self._correlation_rules[$rule_key]}"
                local action="${$self._correlation_rules[${rule_name}_action]}"
                local description="${$self._correlation_rules[${rule_name}_description]}"
                
                if $self._evaluate_correlation "$conditions" "$matched_patterns"; then
                    echo "🔗 CORRÉLATION: $rule_name - $description"
                    
                    case "$action" in
                        alert)
                            echo "🚨 Alerte de corrélation déclenchée"
                            ;;
                            
                        group)
                            echo "📊 Événements groupés"
                            ;;
                            
                        ignore)
                            echo "🔇 Événement ignoré selon la règle"
                            ;;
                    esac
                fi
            fi
        done
    }
    
    # Évaluation d'une condition de corrélation
    $self._evaluate_correlation() {
        local conditions="$1"
        local matched_patterns="$2"
        
        # Évaluation simplifiée - en vrai, nécessiterait un parser plus sophistiqué
        if echo "$conditions" | grep -q "error.*AND.*timeout"; then
            if echo "$matched_patterns" | grep -q "error_pattern" && echo "$matched_patterns" | grep -q "timeout_pattern"; then
                return 0
            fi
        fi
        
        return 1
    }
    
    # Définition d'une règle d'alerte
    $self.define_alert_rule() {
        local alert_name="$1"
        local pattern="$2"
        local threshold="${3:-1}"  # nombre d'occurrences
        local window="${4:-300}"  # fenêtre de temps en secondes
        local severity="${5:-warning}"
        local message="$6"
        
        $self._alert_rules["${alert_name}_pattern"]="$pattern"
        $self._alert_rules["${alert_name}_threshold"]="$threshold"
        $self._alert_rules["${alert_name}_window"]="$window"
        $self._alert_rules["${alert_name}_severity"]="$severity"
        $self._alert_rules["${alert_name}_message"]="$message"
        $self._alert_rules["${alert_name}_occurrences"]=""
        
        echo "✓ Règle d'alerte définie: $alert_name"
    }
    
    # Vérification des règles d'alerte
    $self._check_alert_rules() {
        local source_name="$1"
        local log_line="$2"
        local matched_patterns="$3"
        
        for alert_key in "${!$self._alert_rules[@]}"; do
            if [[ "$alert_key" =~ _pattern$ ]]; then
                local alert_name="${alert_key%_pattern}"
                local pattern="${$self._alert_rules[$alert_key]}"
                
                if echo "$log_line" | grep -q "$pattern"; then
                    local current_time
                    current_time="$(date +%s)"
                    
                    # Ajout de l'occurrence
                    local occurrences="${$self._alert_rules[${alert_name}_occurrences]}"
                    $self._alert_rules["${alert_name}_occurrences"]="${occurrences:+$occurrences;}$current_time"
                    
                    # Comptage dans la fenêtre
                    local window="${$self._alert_rules[${alert_name}_window]}"
                    local threshold="${$self._alert_rules[${alert_name}_threshold]}"
                    
                    local recent_occurrences=0
                    IFS=';' read -ra occurrence_array <<< "$occurrences"
                    
                    for occurrence in "${occurrence_array[@]}"; do
                        if (( current_time - occurrence <= window )); then
                            ((recent_occurrences++))
                        fi
                    done
                    
                    if (( recent_occurrences >= threshold )); then
                        local severity="${$self._alert_rules[${alert_name}_severity]}"
                        local message="${$self._alert_rules[${alert_name}_message]}"
                        
                        echo "🚨 ALERTE LOG: $alert_name ($severity)"
                        echo "   $message"
                        echo "   Occurrences récentes: $recent_occurrences (seuil: $threshold)"
                        
                        # Reset des occurrences après alerte
                        $self._alert_rules["${alert_name}_occurrences"]=""
                    fi
                fi
            fi
        done
    }
    
    # Recherche dans les logs
    $self.search_logs() {
        local query="$1"
        local source_filter="${2:-all}"
        local time_filter="${3:-all}"  # last_1h, last_24h, all
        local limit="${4:-50}"
        
        echo "=== RECHERCHE LOGS ==="
        echo "Requête: $query"
        echo "Sources: $source_filter"
        echo "Période: $time_filter"
        echo "Limite: $limit résultats"
        echo
        
        local current_time
        current_time="$(date +%s)"
        local time_threshold=0
        
        case "$time_filter" in
            last_1h) time_threshold=$(( current_time - 3600 )) ;;
            last_24h) time_threshold=$(( current_time - 86400 )) ;;
            all) time_threshold=0 ;;
        esac
        
        local results=()
        local total_matches=0
        
        # Recherche dans toutes les sources
        for storage_key in "${!$self._log_storage[@]}"; do
            if [[ "$storage_key" =~ _logs$ ]]; then
                local source_name="${storage_key%_logs}"
                
                if [[ "$source_filter" == "all" || "$source_filter" == "$source_name" ]]; then
                    local logs="${$self._log_storage[$storage_key]}"
                    
                    while IFS= read -r log_entry; do
                        if [[ -n "$log_entry" ]]; then
                            local timestamp source log_line patterns parsed
                            IFS='|' read timestamp source log_line patterns parsed <<< "$log_entry"
                            
                            # Filtre temporel
                            if (( timestamp >= time_threshold )); then
                                # Recherche dans le contenu
                                if echo "$log_line" | grep -qi "$query"; then
                                    results+=("$timestamp|$source|$log_line")
                                    ((total_matches++))
                                    
                                    if (( total_matches >= limit )); then
                                        break 2
                                    fi
                                fi
                            fi
                        fi
                    done <<< "$logs"
                fi
            fi
        done
        
        echo "Résultats trouvés: $total_matches"
        echo
        
        # Affichage des résultats
        for result in "${results[@]}"; do
            local timestamp source log_line
            IFS='|' read timestamp source log_line <<< "$result"
            
            printf "[%s] %s: %s\n" "$(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S')" "$source" "$log_line"
        done
        
        if (( total_matches >= limit )); then
            echo "... (limite atteinte)"
        fi
    }
    
    # Génération de rapport d'agrégation de logs
    $self.generate_aggregation_report() {
        local output_file="${1:-log_aggregation_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT D'AGRÉGATION DE LOGS"
            echo "============================"
            echo "Généré le: $(date)"
            echo
            
            echo "SOURCES DE LOGS"
            echo "==============="
            
            for source_key in "${!$self._log_sources[@]}"; do
                if [[ "$source_key" =~ _type$ ]]; then
                    local source_name="${source_key%_type}"
                    local source_type="${$self._log_sources[$source_key]}"
                    local source_path="${$self._log_sources[${source_name}_path]}"
                    local log_count="${$self._log_storage[${source_name}_count]}"
                    local last_read="${$self._log_sources[${source_name}_last_read]}"
                    
                    echo "Source: $source_name ($source_type)"
                    echo "  Chemin: $source_path"
                    echo "  Logs collectés: $log_count"
                    
                    if [[ -n "$last_read" ]]; then
                        echo "  Dernière collecte: $(date -d "@$last_read" '+%Y-%m-%d %H:%M:%S')"
                    fi
                    
                    echo
                fi
            done
            
            echo "PATTERNS DE LOG"
            echo "==============="
            
            for pattern_key in "${!$self._log_patterns[@]}"; do
                if [[ "$pattern_key" =~ _regex$ ]]; then
                    local pattern_name="${pattern_key%_regex}"
                    local severity="${$self._log_patterns[${pattern_name}_severity]}"
                    local category="${$self._log_patterns[${pattern_name}_category]}"
                    
                    echo "Pattern: $pattern_name"
                    echo "  Sévérité: $severity"
                    echo "  Catégorie: $category"
                    echo
                fi
            done
            
            echo "RÈGLES DE CORRÉLATION"
            echo "===================="
            
            for rule_key in "${!$self._correlation_rules[@]}"; do
                if [[ "$rule_key" =~ _conditions$ ]]; then
                    local rule_name="${rule_key%_conditions}"
                    local conditions="${$self._correlation_rules[$rule_key]}"
                    local action="${$self._correlation_rules[${rule_name}_action]}"
                    local description="${$self._correlation_rules[${rule_name}_description]}"
                    
                    echo "Règle: $rule_name"
                    echo "  Conditions: $conditions"
                    echo "  Action: $action"
                    echo "  Description: $description"
                    echo
                fi
            done
            
            echo "STATISTIQUES"
            echo "============"
            
            local total_logs=0 total_sources=0
            
            for storage_key in "${!$self._log_storage[@]}"; do
                if [[ "$storage_key" =~ _count$ ]]; then
                    total_logs=$(( total_logs + ${$self._log_storage[$storage_key]} ))
                    ((total_sources++))
                fi
            done
            
            echo "Sources actives: $total_sources"
            echo "Logs total collectés: $total_logs"
            
            if (( total_sources > 0 )); then
                local avg_logs_per_source=$(( total_logs / total_sources ))
                echo "Moyenne logs par source: $avg_logs_per_source"
            fi
            
            echo
            echo "DERNIERS LOGS (5 par source)"
            echo "============================"
            
            for storage_key in "${!$self._log_storage[@]}"; do
                if [[ "$storage_key" =~ _logs$ ]]; then
                    local source_name="${storage_key%_logs}"
                    local logs="${$self._log_storage[$storage_key]}"
                    
                    if [[ -n "$logs" ]]; then
                        echo "Source: $source_name"
                        
                        # Afficher les 5 derniers logs
                        echo "$logs" | tail -5 | while IFS= read -r log_entry; do
                            local timestamp source log_line
                            IFS='|' read timestamp source log_line <<< "$log_entry"
                            printf "  [%s] %s\n" "$(date -d "@$timestamp" '+%H:%M:%S')" "$log_line"
                        done
                        
                        echo
                    fi
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Export des logs au format Elasticsearch
    $self.export_elasticsearch_bulk() {
        local output_file="${1:-logs_elasticsearch_bulk.json}"
        local index_name="${2:-bash_logs}"
        
        {
            for storage_key in "${!$self._log_storage[@]}"; do
                if [[ "$storage_key" =~ _logs$ ]]; then
                    local source_name="${storage_key%_logs}"
                    local logs="${$self._log_storage[$storage_key]}"
                    
                    while IFS= read -r log_entry; do
                        if [[ -n "$log_entry" ]]; then
                            local timestamp source log_line patterns parsed
                            IFS='|' read timestamp source log_line patterns parsed <<< "$log_entry"
                            
                            # En-tête d'index Elasticsearch
                            cat << EOF
{"index":{"_index":"$index_name","_id":"${source_name}_${timestamp}"}}
{"@timestamp":"$(date -d "@$timestamp" '+%Y-%m-%dT%H:%M:%S.%3NZ')","source":"$source","message":"${log_line//\"/\\\"}","patterns":"$patterns","parsed":"$parsed"}
EOF
                        fi
                    done <<< "$logs"
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Export Elasticsearch généré: $output_file"
    }
}

# Définition des patterns de log courants
define_common_log_patterns() {
    local log_system="$1"
    
    # Patterns système
    $log_system.define_log_pattern "kernel_error" "kernel.*error\|kernel.*fail" "error" "system" "timestamp,level,message"
    $log_system.define_log_pattern "service_start" "Started\|Starting.*service" "info" "system" "timestamp,message"
    $log_system.define_log_pattern "service_stop" "Stopped\|Stopping.*service" "warning" "system" "timestamp,message"
    
    # Patterns sécurité
    $log_system.define_log_pattern "auth_failure" "authentication failure\|invalid user\|bad password" "error" "security" "timestamp,message"
    $log_system.define_log_pattern "sudo_usage" "sudo.*COMMAND" "info" "security" "timestamp,message"
    
    # Patterns réseau
    $log_system.define_log_pattern "network_error" "network.*error\|connection.*fail\|timeout" "warning" "network" "timestamp,message"
    $log_system.define_log_pattern "dns_query" "query.*\|reply.*" "debug" "network"
    
    # Patterns application
    $log_system.define_log_pattern "app_error" "Exception\|Error\|ERROR" "error" "application" "timestamp,level,message"
    $log_system.define_log_pattern "app_warning" "Warning\|WARN" "warning" "application" "timestamp,message"
    $log_system.define_log_pattern "http_request" "GET\|POST\|PUT\|DELETE.*HTTP" "info" "application"
}

# Définition des règles de corrélation
define_correlation_rules() {
    local log_system="$1"
    
    $log_system.define_correlation_rule "security_incident" "auth_failure AND sudo_usage WITHIN 5min" "alert" "Tentative d'accès non autorisé détectée"
    $log_system.define_correlation_rule "service_instability" "service_stop AND service_start WITHIN 10min" "group" "Redémarrage de service fréquent"
    $log_system.define_correlation_rule "network_issues" "network_error AND dns_query WITHIN 2min" "alert" "Problèmes réseau corrélés"
}

# Définition des règles d'alerte
define_alert_rules() {
    local log_system="$1"
    
    $log_system.define_alert_rule "frequent_auth_failures" "authentication failure" "5" "300" "warning" "Multiples échecs d'authentification détectés"
    $log_system.define_alert_rule "service_crashes" "service.*stop\|service.*crash" "3" "600" "error" "Arrêts de service répétés"
    $log_system.define_alert_rule "disk_space_warnings" "No space left\|disk.*full" "1" "60" "critical" "Espace disque critique"
}

# Démonstration du système d'agrégation de logs
echo "--- Système d'agrégation et corrélation de logs ---"

LogAggregationSystem "log_aggregator"

# Définition des patterns et règles
define_common_log_patterns "log_aggregator"
define_correlation_rules "log_aggregator"
define_alert_rules "log_aggregator"

# Enregistrement des sources de logs
log_aggregator.register_log_source "syslog" "file" "/var/log/syslog" "text" "system"
log_aggregator.register_log_source "auth" "file" "/var/log/auth.log" "text" "security"
log_aggregator.register_log_source "app_logs" "file" "/tmp/app.log" "text" "application"
log_aggregator.register_log_source "kernel" "journald" "" "text" "system"
log_aggregator.register_log_source "network_status" "command" "ip addr show && echo '---' && ip route show" "text" "network"

echo
echo "--- Création de logs de test ---"

# Création de logs de test
cat > /tmp/app.log << 'EOF'
2024-01-15 10:00:01 INFO Application started successfully
2024-01-15 10:05:23 ERROR Database connection failed
2024-01-15 10:05:45 WARN Retrying database connection (attempt 1)
2024-01-15 10:06:01 INFO Database connection restored
2024-01-15 10:15:30 ERROR Authentication failure for user admin
2024-01-15 10:15:35 ERROR Authentication failure for user admin
2024-01-15 10:15:40 ERROR Authentication failure for user admin
2024-01-15 10:20:15 INFO User admin logged in successfully
2024-01-15 10:25:01 WARN High memory usage detected: 85%
2024-01-15 10:30:22 ERROR Service httpd stopped unexpectedly
2024-01-15 10:30:25 INFO Service httpd starting
2024-01-15 10:35:01 INFO Backup completed successfully
EOF

echo "Logs de test créés"

echo
echo "--- Collecte des logs ---"

log_aggregator.collect_logs "app_logs"
log_aggregator.collect_logs "network_status"

echo
echo "--- Recherche dans les logs ---"

log_aggregator.search_logs "ERROR" "all" "all" "10"

echo
echo "--- Génération de rapport ---"
log_aggregator.generate_aggregation_report

echo
echo "--- Export Elasticsearch ---"
log_aggregator.export_elasticsearch_bulk

# Nettoyage
rm -f /tmp/app.log log_aggregation_report_*.txt logs_elasticsearch_bulk.json
```

## Section 2 : Tableaux de bord et visualisation

### 2.1 Générateur de tableaux de bord texte

Création de dashboards en mode texte avec métriques en temps réel :

```bash
#!/bin/bash

# Générateur de tableaux de bord texte
echo "=== Générateur de tableaux de bord texte ==="

# Text Dashboard Generator
TextDashboard() {
    local self="$1"
    
    declare -A $self._dashboard_panels
    declare -A $self._dashboard_layout
    declare -A $self._dashboard_data
    
    # Définition d'un panneau de dashboard
    $self.define_panel() {
        local panel_name="$1"
        local panel_type="$2"  # gauge, chart, table, text, list
        local data_source="$3"
        local position="$4"  # row,col,width,height
        local title="$5"
        local refresh_interval="${6:-30}"
        
        $self._dashboard_panels["${panel_name}_type"]="$panel_type"
        $self._dashboard_panels["${panel_name}_source"]="$data_source"
        $self._dashboard_panels["${panel_name}_position"]="$position"
        $self._dashboard_panels["${panel_name}_title"]="$title"
        $self._dashboard_panels["${panel_name}_refresh"]="$refresh_interval"
        $self._dashboard_panels["${panel_name}_last_update"]=0
        
        echo "✓ Panneau défini: $panel_name ($panel_type)"
    }
    
    # Configuration de la disposition
    $self.configure_layout() {
        local layout_name="$1"
        local dimensions="$2"  # width,height
        local theme="${3:-default}"  # default, dark, light
        
        $self._dashboard_layout["name"]="$layout_name"
        $self._dashboard_layout["dimensions"]="$dimensions"
        $self._dashboard_layout["theme"]="$theme"
        
        echo "✓ Disposition configurée: $layout_name (${dimensions})"
    }
    
    # Mise à jour des données d'un panneau
    $self.update_panel_data() {
        local panel_name="$1"
        
        local data_source="${$self._dashboard_panels[${panel_name}_source]}"
        local panel_type="${$self._dashboard_panels[${panel_name}_type]}"
        
        if [[ -z "$panel_type" ]]; then
            echo "❌ Panneau introuvable: $panel_name" >&2
            return 1
        fi
        
        # Collecte des données selon la source
        local data=""
        
        case "$data_source" in
            system_cpu)
                data="$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')"
                ;;
                
            system_memory)
                data="$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100.0}')"
                ;;
                
            system_disk)
                data="$(df / | tail -1 | awk '{print $5}' | sed 's/%//')"
                ;;
                
            network_connections)
                data="$(netstat -tun | grep ESTABLISHED | wc -l)"
                ;;
                
            process_count)
                data="$(ps aux | wc -l)"
                ;;
                
            load_average)
                data="$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)"
                ;;
                
            custom:*)
                local custom_cmd="${data_source#custom:}"
                data="$(eval "$custom_cmd" 2>/dev/null)"
                ;;
                
            *)
                data="N/A"
                ;;
        esac
        
        # Stockage des données
        $self._dashboard_data["${panel_name}_current"]="$data"
        $self._dashboard_data["${panel_name}_history"]="${$self._dashboard_data[${panel_name}_history]:+$self._dashboard_data[${panel_name}_history];}$(date +%s):$data"
        $self._dashboard_panels["${panel_name}_last_update"]="$(date +%s)"
        
        echo "✓ Données mises à jour pour $panel_name: $data"
    }
    
    # Rendu d'un panneau
    $self.render_panel() {
        local panel_name="$1"
        local width="$2"
        local height="$3"
        
        local panel_type="${$self._dashboard_panels[${panel_name}_type]}"
        local title="${$self._dashboard_panels[${panel_name}_title]}"
        local current_data="${$self._dashboard_data[${panel_name}_current]}"
        
        if [[ -z "$panel_type" ]]; then
            echo "Panneau vide"
            return
        fi
        
        # Calcul des dimensions intérieures (sans bordures)
        local inner_width=$(( width - 4 ))
        local inner_height=$(( height - 3 ))
        
        # Fonction d'aide pour centrer le texte
        center_text() {
            local text="$1"
            local target_width="$2"
            local padding=$(( (target_width - ${#text}) / 2 ))
            printf "%*s%s" "$padding" "" "$text"
        }
        
        # Bords du panneau
        local top_border bottom_border
        printf -v top_border "┌%*s┐" "$inner_width" "" | tr ' ' '─'
        printf -v bottom_border "└%*s┘" "$inner_width" "" | tr ' ' '─'
        
        # Rendu selon le type
        case "$panel_type" in
            gauge)
                $self._render_gauge_panel "$title" "$current_data" "$inner_width" "$inner_height"
                ;;
                
            chart)
                $self._render_chart_panel "$title" "$panel_name" "$inner_width" "$inner_height"
                ;;
                
            table)
                $self._render_table_panel "$title" "$panel_name" "$inner_width" "$inner_height"
                ;;
                
            text)
                $self._render_text_panel "$title" "$current_data" "$inner_width" "$inner_height"
                ;;
                
            list)
                $self._render_list_panel "$title" "$panel_name" "$inner_width" "$inner_height"
                ;;
                
            *)
                echo "Type de panneau non supporté: $panel_type"
                ;;
        esac
    }
    
    # Rendu panneau jauge
    $self._render_gauge_panel() {
        local title="$1"
        local value="$2"
        local width="$3"
        local height="$4"
        
        # Conversion en nombre
        local num_value
        num_value=$(echo "$value" | sed 's/[^0-9.]//g')
        
        if [[ -z "$num_value" ]]; then
            num_value=0
        fi
        
        # Calcul du pourcentage de remplissage
        local percentage=$(( num_value > 100 ? 100 : num_value ))
        local filled=$(( percentage * (width - 2) / 100 ))
        local empty=$(( (width - 2) - filled ))
        
        echo "┌─ $title ─┐"
        echo "│$(printf '█%.0s' $(seq 1 "$filled"))$(printf '░%.0s' $(seq 1 "$empty"))│"
        echo "│$(printf '%*s' "$((width - 2))" "" | tr ' ' ' ')│"
        printf "└%*s┘" "$width" "" | tr ' ' '─'
        echo "  ${value}%"
    }
    
    # Rendu panneau graphique
    $self._render_chart_panel() {
        local title="$1"
        local panel_name="$2"
        local width="$3"
        local height="$4"
        
        local history="${$self._dashboard_data[${panel_name}_history]}"
        
        echo "┌─ $title ─┐"
        
        if [[ -n "$history" ]]; then
            # Prendre les 10 dernières valeurs
            local values
            values="$(echo "$history" | tr ';' '\n' | tail -10 | cut -d: -f2)"
            
            # Trouver min/max
            local min_val max_val
            min_val=$(echo "$values" | sort -n | head -1)
            max_val=$(echo "$values" | sort -n | tail -1)
            
            local range=$(( max_val - min_val ))
            (( range == 0 )) && range=1
            
            # Dessiner le graphique sparkline-style
            local chart_line=""
            echo "$values" | while read -r val; do
                if [[ -n "$val" ]]; then
                    local normalized=$(( (val - min_val) * 7 / range ))
                    local char
                    case "$normalized" in
                        0) char="▁" ;;
                        1) char="▂" ;;
                        2) char="▃" ;;
                        3) char="▄" ;;
                        4) char="▅" ;;
                        5) char="▆" ;;
                        6|7) char="▇" ;;
                        *) char="░" ;;
                    esac
                    chart_line="${chart_line}${char}"
                fi
            done
            
            echo "│$chart_line│"
        else
            echo "│$(printf '%*s' "$((width - 2))" "No data")│"
        fi
        
        # Lignes vides
        for ((i = 3; i < height; i++)); do
            echo "│$(printf '%*s' "$((width - 2))" "")│"
        done
        
        printf "└%*s┘" "$width" "" | tr ' ' '─'
    }
    
    # Rendu panneau texte
    $self._render_text_panel() {
        local title="$1"
        local content="$2"
        local width="$3"
        local height="$4"
        
        echo "┌─ $title ─┐"
        
        # Découper le contenu en lignes
        local line_num=1
        while IFS= read -r line && (( line_num < height - 1 )); do
            printf "│ %-*s │\n" "$((width - 4))" "${line:0:$((width - 4))}"
            ((line_num++))
        done <<< "$content"
        
        # Remplir les lignes restantes
        while (( line_num < height - 1 )); do
            echo "│$(printf '%*s' "$((width - 2))" "")│"
            ((line_num++))
        done
        
        printf "└%*s┘" "$width" "" | tr ' ' '─'
    }
    
    # Rendu panneau liste
    $self._render_list_panel() {
        local title="$1"
        local panel_name="$2"
        local width="$3"
        local height="$4"
        
        echo "┌─ $title ─┐"
        
        # Simulation d'une liste de processus
        if [[ "$panel_name" == "top_processes" ]]; then
            ps aux --sort=-%cpu | head -5 | tail -4 | while read -r user pid cpu mem command; do
                printf "│ %5s %5s %4s │\n" "$pid" "${cpu}%" "${command:0:10}"
            done
        else
            echo "│$(printf '%*s' "$((width - 2))" "Liste vide")│"
        fi
        
        # Remplir
        for ((i = 6; i < height; i++)); do
            echo "│$(printf '%*s' "$((width - 2))" "")│"
        done
        
        printf "└%*s┘" "$width" "" | tr ' ' '─'
    }
    
    # Rendu du dashboard complet
    $self.render_dashboard() {
        local clear_screen="${1:-true}"
        
        if [[ "$clear_screen" == "true" ]]; then
            clear
        fi
        
        local layout_name="${$self._dashboard_layout[name]}"
        local dimensions="${$self._dashboard_layout[dimensions]}"
        local theme="${$self._dashboard_layout[theme]}"
        
        echo "=== DASHBOARD: $layout_name ==="
        echo "Dimensions: $dimensions | Thème: $theme"
        echo "Dernière mise à jour: $(date)"
        echo
        
        # Mise à jour de tous les panneaux
        for panel_key in "${!$self._dashboard_panels[@]}"; do
            if [[ "$panel_key" =~ _type$ ]]; then
                local panel_name="${panel_key%_type}"
                $self.update_panel_data "$panel_name"
            fi
        done
        
        echo
        
        # Disposition simple: panneaux les uns sous les autres
        local panel_width=50
        local panel_height=8
        
        for panel_key in "${!$self._dashboard_panels[@]}"; do
            if [[ "$panel_key" =~ _type$ ]]; then
                local panel_name="${panel_key%_type}"
                $self.render_panel "$panel_name" "$panel_width" "$panel_height"
                echo
            fi
        done
        
        # Légende
        echo "Légende: █ = Actif | ░ = Inactif | ▁▂▃▄▅▆▇ = Niveaux (graphiques)"
    }
    
    # Mode surveillance continue
    $self.start_monitoring_mode() {
        local refresh_interval="${1:-5}"
        
        echo "=== MODE SURVEILLANCE ==="
        echo "Intervalle: ${refresh_interval}s | Ctrl+C pour quitter"
        echo
        
        while true; do
            $self.render_dashboard "true"
            sleep "$refresh_interval"
        done
    }
    
    # Export du dashboard en HTML
    $self.export_html_dashboard() {
        local output_file="${1:-dashboard.html}"
        
        {
            cat << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>System Dashboard</title>
    <style>
        body { font-family: monospace; background: #000; color: #0f0; margin: 20px; }
        .panel { border: 1px solid #0f0; margin: 10px; padding: 10px; float: left; }
        .gauge { position: relative; height: 20px; background: #333; }
        .gauge-fill { height: 100%; background: #0f0; }
        .chart { font-size: 20px; }
        .timestamp { color: #666; font-size: 12px; }
    </style>
</head>
<body>
    <h1>System Dashboard</h1>
    <div class="timestamp">Generated: 
EOF
            date
            cat << 'EOF'
    </div>
EOF
            
            # Génération des panneaux HTML
            for panel_key in "${!$self._dashboard_panels[@]}"; do
                if [[ "$panel_key" =~ _type$ ]]; then
                    local panel_name="${panel_key%_type}"
                    local panel_type="${$self._dashboard_panels[$panel_key]}"
                    local title="${$self._dashboard_panels[${panel_name}_title]}"
                    local current_data="${$self._dashboard_data[${panel_name}_current]}"
                    
                    echo "<div class=\"panel\">"
                    echo "<h3>$title</h3>"
                    
                    case "$panel_type" in
                        gauge)
                            local percentage=$(echo "$current_data" | sed 's/[^0-9.]//g')
                            echo "<div class=\"gauge\">"
                            echo "<div class=\"gauge-fill\" style=\"width: ${percentage}%\"></div>"
                            echo "</div>"
                            echo "<div>$current_data%</div>"
                            ;;
                            
                        chart)
                            local history="${$self._dashboard_data[${panel_name}_history]}"
                            echo "<div class=\"chart\">"
                            if [[ -n "$history" ]]; then
                                echo "$history" | tr ';' '\n' | tail -10 | cut -d: -f2 | tr '\n' ' ' | sed 's/ /▇/g'
                            fi
                            echo "</div>"
                            ;;
                            
                        text)
                            echo "<pre>$current_data</pre>"
                            ;;
                            
                        *)
                            echo "<div>$current_data</div>"
                            ;;
                    esac
                    
                    echo "</div>"
                fi
            done
            
            cat << 'EOF'
</body>
</html>
EOF
            
        } > "$output_file"
        
        echo "✓ Dashboard HTML exporté: $output_file"
    }
}

# Démonstration du générateur de dashboard texte
echo "--- Générateur de tableaux de bord texte ---"

TextDashboard "text_dashboard"

# Configuration du layout
text_dashboard.configure_layout "system_monitor" "120x40" "dark"

# Définition des panneaux
text_dashboard.define_panel "cpu_gauge" "gauge" "system_cpu" "0,0,50,8" "CPU Usage" "5"
text_dashboard.define_panel "memory_gauge" "gauge" "system_memory" "0,8,50,8" "Memory Usage" "5"
text_dashboard.define_panel "disk_gauge" "gauge" "system_disk" "0,16,50,8" "Disk Usage" "30"
text_dashboard.define_panel "cpu_chart" "chart" "system_cpu" "50,0,50,8" "CPU History" "5"
text_dashboard.define_panel "memory_chart" "chart" "system_memory" "50,8,50,8" "Memory History" "5"
text_dashboard.define_panel "system_info" "text" "custom:uname -a && echo '---' && uptime" "0,24,100,6" "System Info" "60"
text_dashboard.define_panel "top_processes" "list" "process_count" "0,30,100,8" "Top Processes" "10"

echo
echo "--- Rendu du dashboard ---"
text_dashboard.render_dashboard "false"

echo
echo "--- Export HTML ---"
text_dashboard.export_html_dashboard

# Nettoyage
rm -f dashboard.html
```

## Conclusion : L'observabilité comme conscience

L'observabilité avancée en Bash transforme l'administration système d'une suite de commandes réactives en une intelligence proactive capable de prédire les problèmes, corréler les événements, et présenter l'état du système avec une clarté cristalline. Cette approche transforme les métriques brutes en insights actionnables, les logs éparpillés en récits cohérents, et les tableaux de bord statiques en visualisations vivantes.

**Points clés à retenir :**

1. **Métriques temps réel** : Frameworks de collecte, agrégation et alerte avec calculs continus et seuils intelligents
2. **Agrégation de logs** : Systèmes de corrélation, recherche et analyse avec patterns et règles d'alerte
3. **Tableaux de bord** : Interfaces texte riches avec jauges, graphiques et mises à jour en temps réel

Dans le prochain chapitre, nous explorerons les techniques avancées de scripting réseau et d'opérations distribuées, pour que nos systèmes observables puissent communiquer et coordonner à travers les vastes réseaux distribués.

---

**Exercice pratique :** Créez un système d'observabilité complet incluant :
- Collecte de métriques système avec alertes configurables
- Agrégation de logs multi-sources avec corrélation d'événements
- Dashboard texte en temps réel avec visualisation de métriques

**Réflexion :** Comment concevriez-vous une architecture d'observabilité où les scripts Bash deviennent les capteurs nerveux d'un système distribué conscient de son propre état, capable d'auto-diagnostic et d'auto-guérison ?
