# Chapitre 89 - Monitoring avancé et observabilité

> "Ce que vous ne pouvez pas mesurer, vous ne pouvez pas améliorer." - Peter Drucker

## Introduction : Voir l'invisible et prévoir l'imprévisible

Imaginez-vous capitaine de vaisseau spatial naviguant dans l'obscurité totale de l'espace profond. Vos instruments de bord ne se contentent pas d'afficher des voyants lumineux : ils vous donnent une vision complète de votre environnement, anticipent les dangers, et vous guident vers votre destination. Le monitoring avancé en Bash, c'est exactement cela : transformer vos systèmes en observatoires complets capables de voir l'invisible, prévoir l'imprévisible, et maintenir la stabilité dans la complexité.

Dans ce chapitre, nous allons construire des systèmes d'observabilité complets : collecteurs de métriques sophistiqués, systèmes d'alertes intelligents, agrégateurs de logs distribués, et tableaux de bord en temps réel.

## Section 1 : Collecte avancée de métriques

### 1.1 Framework de métriques multi-sources

Système unifié pour collecter des métriques depuis diverses sources :

```bash
#!/bin/bash

# Framework de métriques multi-sources
echo "=== Framework de métriques multi-sources ==="

# Collecteur de métriques unifié
MetricsCollector() {
    local self="$1"
    
    declare -A $self._metrics
    declare -A $self._collectors
    declare -a $self._collection_order
    
    # Enregistrement d'un collecteur
    $self.register_collector() {
        local name="$1"
        local collector_function="$2"
        local interval="${3:-60}"
        local priority="${4:-10}"
        
        $self._collectors["${name}_function"]="$collector_function"
        $self._collectors["${name}_interval"]="$interval"
        $self._collectors["${name}_priority"]="$priority"
        $self._collectors["${name}_last_run"]="0"
        $self._collectors["${name}_enabled"]="true"
        
        # Insertion dans l'ordre de priorité
        $self._insert_by_priority "$name" "$priority"
        
        echo "✓ Collecteur enregistré: $name (intervalle: ${interval}s, priorité: $priority)"
    }
    
    # Insertion par priorité
    $self._insert_by_priority() {
        local name="$1"
        local priority="$2"
        
        local -a new_order
        local inserted=false
        
        for existing in "${$self._collection_order[@]}"; do
            local existing_priority="${$self._collectors[${existing}_priority]}"
            
            if (( priority < existing_priority )) && [[ "$inserted" != "true" ]]; then
                new_order+=("$name")
                inserted=true
            fi
            
            new_order+=("$existing")
        done
        
        if [[ "$inserted" != "true" ]]; then
            new_order+=("$name")
        fi
        
        $self._collection_order=("${new_order[@]}")
    }
    
    # Collecte des métriques
    $self.collect_metrics() {
        local current_time
        current_time=$(date +%s)
        
        echo "=== COLLECTE DES MÉTRIQUES ==="
        echo "Timestamp: $current_time"
        
        local collected=0
        
        for collector_name in "${$self._collection_order[@]}"; do
            if [[ "${$self._collectors[${collector_name}_enabled]}" != "true" ]]; then
                continue
            fi
            
            local last_run="${$self._collectors[${collector_name}_last_run]}"
            local interval="${$self._collectors[${collector_name}_interval]}"
            
            # Vérification de l'intervalle
            if (( current_time - last_run >= interval )); then
                echo "--- Collecteur: $collector_name ---"
                
                local collector_function="${$self._collectors[${collector_name}_function]}"
                
                if $self._run_collector "$collector_function" "$collector_name"; then
                    $self._collectors["${collector_name}_last_run"]="$current_time"
                    ((collected++))
                    echo "✓ Collecté"
                else
                    echo "❌ Échec de collecte"
                fi
            fi
        done
        
        echo "Collecteurs exécutés: $collected"
        return $(( collected > 0 ))
    }
    
    # Exécution d'un collecteur
    $self._run_collector() {
        local collector_function="$1"
        local collector_name="$2"
        
        # Exécution dans un sous-shell pour l'isolation
        (
            # Fonction disponible dans le sous-shell
            $collector_function() {
                # Collecte des métriques système
                case "$collector_name" in
                    cpu)
                        local cpu_usage
                        cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
                        $self._store_metric "system.cpu.usage_percent" "$cpu_usage" "gauge" "Utilisation CPU"
                        ;;
                    memory)
                        local mem_total mem_used
                        mem_total=$(free | grep Mem | awk '{print $2}')
                        mem_used=$(free | grep Mem | awk '{print $3}')
                        local mem_usage=$((mem_used * 100 / mem_total))
                        $self._store_metric "system.memory.usage_percent" "$mem_usage" "gauge" "Utilisation mémoire"
                        $self._store_metric "system.memory.used_bytes" "$mem_used" "gauge" "Mémoire utilisée"
                        ;;
                    disk)
                        df -BG | grep '^/dev/' | while read fs size used avail use mount; do
                            local usage_percent="${use%%%}"
                            $self._store_metric "system.disk.usage_percent{mount=\"$mount\"}" "$usage_percent" "gauge" "Utilisation disque $mount"
                        done
                        ;;
                    network)
                        # Interfaces réseau actives
                        ip route show | grep -v "default" | while read network dev rest; do
                            local interface="$dev"
                            local rx_bytes tx_bytes
                            rx_bytes=$(cat "/sys/class/net/$interface/statistics/rx_bytes" 2>/dev/null || echo "0")
                            tx_bytes=$(cat "/sys/class/net/$interface/statistics/tx_bytes" 2>/dev/null || echo "0")
                            $self._store_metric "system.network.rx_bytes{interface=\"$interface\"}" "$rx_bytes" "counter" "Octets reçus $interface"
                            $self._store_metric "system.network.tx_bytes{interface=\"$interface\"}" "$tx_bytes" "counter" "Octets transmis $interface"
                        done
                        ;;
                    processes)
                        local proc_count
                        proc_count=$(ps aux | wc -l)
                        $self._store_metric "system.processes.count" "$proc_count" "gauge" "Nombre de processus"
                        
                        local zombie_count
                        zombie_count=$(ps aux | awk '{print $8}' | grep -c 'Z')
                        $self._store_metric "system.processes.zombie_count" "$zombie_count" "gauge" "Processus zombies"
                        ;;
                    load)
                        local load1 load5 load15
                        read load1 load5 load15 < /proc/loadavg
                        $self._store_metric "system.load.1m" "$load1" "gauge" "Charge système 1m"
                        $self._store_metric "system.load.5m" "$load5" "gauge" "Charge système 5m"
                        $self._store_metric "system.load.15m" "$load15" "gauge" "Charge système 15m"
                        ;;
                    custom)
                        # Collecteur personnalisé - exemple
                        local custom_value=$((RANDOM % 100))
                        $self._store_metric "custom.random_value" "$custom_value" "gauge" "Valeur aléatoire personnalisée"
                        ;;
                esac
            }
            
            # Exécution
            $collector_function
        )
    }
    
    # Stockage d'une métrique
    $self._store_metric() {
        local name="$1"
        local value="$2"
        local type="$3"
        local description="$4"
        
        local timestamp
        timestamp=$(date +%s)
        
        # Stockage avec timestamp
        $self._metrics["${name}_${timestamp}"]="$value"
        $self._metrics["${name}_type"]="$type"
        $self._metrics["${name}_description"]="$description"
        $self._metrics["${name}_last_update"]="$timestamp"
    }
    
    # Récupération des métriques
    $self.get_metrics() {
        local pattern="${1:-}"
        local format="${2:-text}"
        
        case "$format" in
            prometheus)
                $self._export_prometheus "$pattern"
                ;;
            json)
                $self._export_json "$pattern"
                ;;
            text|*)
                $self._export_text "$pattern"
                ;;
        esac
    }
    
    # Export Prometheus
    $self._export_prometheus() {
        local pattern="$1"
        
        echo "# Métriques système - Export Prometheus"
        echo "# Généré le $(date)"
        
        for metric_key in "${!$self._metrics[@]}"; do
            if [[ "$metric_key" =~ _last_update$ ]]; then
                local base_name="${metric_key%_last_update}"
                
                if [[ -z "$pattern" || "$base_name" =~ $pattern ]]; then
                    local value="${$self._metrics[${base_name}_last_update]}"
                    local type="${$self._metrics[${base_name}_type]}"
                    local description="${$self._metrics[${base_name}_description]}"
                    
                    echo "# HELP ${base_name} ${description}"
                    echo "# TYPE ${base_name} ${type}"
                    echo "${base_name} ${value}"
                fi
            fi
        done
    }
    
    # Export JSON
    $self._export_json() {
        local pattern="$1"
        
        echo "{"
        echo "  \"timestamp\": \"$(date -Iseconds)\","
        echo "  \"metrics\": {"
        
        local first=true
        for metric_key in "${!$self._metrics[@]}"; do
            if [[ "$metric_key" =~ _last_update$ ]]; then
                local base_name="${metric_key%_last_update}"
                
                if [[ -z "$pattern" || "$base_name" =~ $pattern ]]; then
                    if [[ "$first" == "true" ]]; then
                        first=false
                    else
                        echo ","
                    fi
                    
                    local value="${$self._metrics[${base_name}_last_update]}"
                    local type="${$self._metrics[${base_name}_type]}"
                    local description="${$self._metrics[${base_name}_description]}"
                    
                    echo -n "    \"$base_name\": {"
                    echo -n "\"value\": $value,"
                    echo -n "\"type\": \"$type\","
                    echo -n "\"description\": \"$description\""
                    echo -n "}"
                fi
            fi
        done
        echo
        echo "  }"
        echo "}"
    }
    
    # Export texte
    $self._export_text() {
        local pattern="$1"
        
        echo "=== MÉTRIQUES SYSTÈME ==="
        echo "Timestamp: $(date)"
        echo
        
        for metric_key in "${!$self._metrics[@]}"; do
            if [[ "$metric_key" =~ _last_update$ ]]; then
                local base_name="${metric_key%_last_update}"
                
                if [[ -z "$pattern" || "$base_name" =~ $pattern ]]; then
                    local value="${$self._metrics[${base_name}_last_update]}"
                    local type="${$self._metrics[${base_name}_type]}"
                    local description="${$self._metrics[${base_name}_description]}"
                    
                    printf "%-40s %-10s %s\n" "$base_name" "$value" "$description"
                fi
            fi
        done
    }
    
    # Statistiques des collecteurs
    $self.collector_stats() {
        echo "=== STATISTIQUES DES COLLECTEURS ==="
        
        for collector_name in "${$self._collection_order[@]}"; do
            local enabled="${$self._collectors[${collector_name}_enabled]}"
            local interval="${$self._collectors[${collector_name}_interval]}"
            local priority="${$self._collectors[${collector_name}_priority]}"
            local last_run="${$self._collectors[${collector_name}_last_run]}"
            
            local status="✓"
            if [[ "$enabled" != "true" ]]; then
                status="❌"
            fi
            
            printf "%-15s %s Priorité: %2d Intervalle: %3ds Dernière exéc: %s\n" \
                   "$collector_name" "$status" "$priority" "$interval" \
                   "$(date -d "@$last_run" '+%H:%M:%S' 2>/dev/null || echo 'jamais')"
        done
    }
}

# Démonstration du framework de métriques
echo "--- Framework de métriques multi-sources ---"

MetricsCollector "metrics"

# Enregistrement des collecteurs avec différentes priorités
metrics.register_collector "cpu" "cpu_collector" 5 1      # Haute priorité, fréquence élevée
metrics.register_collector "memory" "memory_collector" 10 2
metrics.register_collector "disk" "disk_collector" 30 3
metrics.register_collector "network" "network_collector" 15 2
metrics.register_collector "processes" "processes_collector" 20 4
metrics.register_collector "load" "load_collector" 10 2
metrics.register_collector "custom" "custom_collector" 60 5  # Basse priorité

# Collecte initiale
metrics.collect_metrics

echo
echo "--- Statistiques des collecteurs ---"
metrics.collector_stats

echo
echo "--- Export des métriques (format texte) ---"
metrics.get_metrics

echo
echo "--- Export des métriques (format Prometheus) ---"
metrics.get_metrics "" "prometheus"

echo
echo "--- Collecte après quelques secondes ---"
sleep 3
metrics.collect_metrics

echo
echo "--- Métriques CPU uniquement ---"
metrics.get_metrics "cpu"
```

### 1.2 Collecte de métriques applicatives

Instrumentation d'applications pour l'observabilité :

```bash
#!/bin/bash

# Collecte de métriques applicatives
echo "=== Collecte de métriques applicatives ==="

# Framework d'instrumentation applicative
ApplicationMetrics() {
    local self="$1"
    
    declare -A $self._app_metrics
    declare -a $self._timers
    declare -A $self._counters
    declare -A $self._gauges
    declare -A $self._histograms
    
    # Timer pour mesurer les durées
    $self.start_timer() {
        local name="$1"
        
        $self._timers["${name}_start"]="$(date +%s.%N)"
        echo "✓ Timer démarré: $name"
    }
    
    $self.stop_timer() {
        local name="$1"
        local description="${2:-Durée d'exécution}"
        
        local start_time="${$self._timers[${name}_start]}"
        
        if [[ -z "$start_time" ]]; then
            echo "❌ Timer non démarré: $name"
            return 1
        fi
        
        local end_time
        end_time="$(date +%s.%N)"
        local duration
        duration="$(echo "$end_time - $start_time" | bc)"
        
        # Stockage de la métrique
        $self._store_metric "app.timer.${name}" "$duration" "histogram" "$description"
        
        # Statistiques
        local count="${$self._timers[${name}_count]:-0}"
        local total="${$self._timers[${name}_total]:-0}"
        
        count=$((count + 1))
        total="$(echo "$total + $duration" | bc)"
        local avg="$(echo "scale=4; $total / $count" | bc)"
        
        $self._timers["${name}_count"]="$count"
        $self._timers["${name}_total"]="$total"
        $self._timers["${name}_avg"]="$avg"
        
        echo "✓ Timer arrêté: $name (${duration}s, moyenne: ${avg}s)"
        
        return 0
    }
    
    # Compteur d'événements
    $self.increment_counter() {
        local name="$1"
        local value="${2:-1}"
        local description="${3:-Compteur d'événements}"
        
        local current="${$self._counters[$name]:-0}"
        current=$((current + value))
        $self._counters["$name"]="$current"
        
        $self._store_metric "app.counter.${name}" "$current" "counter" "$description"
        
        echo "✓ Compteur incrémenté: $name = $current"
    }
    
    # Jauge (valeur instantanée)
    $self.set_gauge() {
        local name="$1"
        local value="$2"
        local description="${3:-Valeur instantanée}"
        
        $self._gauges["$name"]="$value"
        $self._store_metric "app.gauge.${name}" "$value" "gauge" "$description"
        
        echo "✓ Jauge définie: $name = $value"
    }
    
    # Histogramme pour distributions
    $self.record_histogram() {
        local name="$1"
        local value="$2"
        local description="${3:-Distribution de valeurs}"
        
        # Stockage de la valeur dans l'histogramme
        local -a histogram_values
        local histogram_key="${name}_values"
        
        # Récupération des valeurs existantes (simulation simplifiée)
        local existing="${$self._histograms[$histogram_key]}"
        if [[ -n "$existing" ]]; then
            histogram_values=($existing)
        fi
        
        histogram_values+=("$value")
        $self._histograms["$histogram_key"]="${histogram_values[*]}"
        
        # Calcul des statistiques
        local count="${#histogram_values[@]}"
        local sum=0 min="${histogram_values[0]}" max="${histogram_values[0]}"
        
        for val in "${histogram_values[@]}"; do
            sum=$((sum + val))
            if (( val < min )); then min="$val"; fi
            if (( val > max )); then max="$val"; fi
        done
        
        local avg=$((sum / count))
        
        $self._store_metric "app.histogram.${name}_count" "$count" "counter" "$description - Nombre"
        $self._store_metric "app.histogram.${name}_sum" "$sum" "counter" "$description - Somme"
        $self._store_metric "app.histogram.${name}_avg" "$avg" "gauge" "$description - Moyenne"
        $self._store_metric "app.histogram.${name}_min" "$min" "gauge" "$description - Minimum"
        $self._store_metric "app.histogram.${name}_max" "$max" "gauge" "$description - Maximum"
        
        echo "✓ Histogramme enregistré: $name = $value (count: $count, avg: $avg)"
    }
    
    # Stockage générique de métriques
    $self._store_metric() {
        local name="$1"
        local value="$2"
        local type="$3"
        local description="$4"
        
        local timestamp
        timestamp="$(date +%s)"
        
        $self._app_metrics["${name}_value"]="$value"
        $self._app_metrics["${name}_type"]="$type"
        $self._app_metrics["${name}_description"]="$description"
        $self._app_metrics["${name}_timestamp"]="$timestamp"
    }
    
    # Instrumentation automatique de fonctions
    $self.instrument_function() {
        local func_name="$1"
        
        # Sauvegarde de la fonction originale
        local original_func
        original_func="$(declare -f "$func_name")"
        
        # Création de la version instrumentée
        eval "
$func_name() {
    $self.start_timer '${func_name}'
    $self.increment_counter '${func_name}_calls'
    
    # Exécution de la fonction originale
    ${original_func#*$func_name()}
    
    $self.stop_timer '${func_name}' 'Durée d'\''exécution de ${func_name}'
}
"
        
        echo "✓ Fonction instrumentée: $func_name"
    }
    
    # Export des métriques applicatives
    $self.export_metrics() {
        local format="${1:-json}"
        
        case "$format" in
            json)
                $self._export_json
                ;;
            prometheus)
                $self._export_prometheus
                ;;
            text|*)
                $self._export_text
                ;;
        esac
    }
    
    # Export JSON
    $self._export_json() {
        echo "{"
        echo "  \"timestamp\": \"$(date -Iseconds)\","
        echo "  \"application_metrics\": {"
        
        local first=true
        for key in "${!$self._app_metrics[@]}"; do
            if [[ "$key" =~ _value$ ]]; then
                local base_name="${key%_value}"
                
                if [[ "$first" == "true" ]]; then
                    first=false
                else
                    echo ","
                fi
                
                local value="${$self._app_metrics[${key}]}"
                local type="${$self._app_metrics[${base_name}_type]}"
                local description="${$self._app_metrics[${base_name}_description]}"
                local timestamp="${$self._app_metrics[${base_name}_timestamp]}"
                
                echo -n "    \"$base_name\": {"
                echo -n "\"value\": \"$value\","
                echo -n "\"type\": \"$type\","
                echo -n "\"description\": \"$description\","
                echo -n "\"timestamp\": \"$timestamp\""
                echo -n "}"
            fi
        done
        echo
        echo "  }"
        echo "}"
    }
    
    # Export Prometheus
    $self._export_prometheus() {
        echo "# Métriques applicatives - Export Prometheus"
        echo "# Généré le $(date)"
        
        for key in "${!$self._app_metrics[@]}"; do
            if [[ "$key" =~ _value$ ]]; then
                local base_name="${key%_value}"
                local value="${$self._app_metrics[${key}]}"
                local type="${$self._app_metrics[${base_name}_type]}"
                local description="${$self._app_metrics[${base_name}_description]}"
                
                echo "# HELP ${base_name} ${description}"
                echo "# TYPE ${base_name} ${type}"
                echo "${base_name} ${value}"
            fi
        done
    }
    
    # Export texte
    $self._export_text() {
        echo "=== MÉTRIQUES APPLICATIVES ==="
        echo "Timestamp: $(date)"
        echo
        
        for key in "${!$self._app_metrics[@]}"; do
            if [[ "$key" =~ _value$ ]]; then
                local base_name="${key%_value}"
                local value="${$self._app_metrics[${key}]}"
                local description="${$self._app_metrics[${base_name}_description]}"
                
                printf "%-40s %-15s %s\n" "$base_name" "$value" "$description"
            fi
        done
    }
}

# Fonctions de démonstration
slow_function() {
    echo "Exécution d'une fonction lente..."
    sleep 1
    echo "Calcul complexe..."
    local result=0
    for ((i=1; i<=10000; i++)); do
        result=$((result + i))
    done
    echo "Résultat: $result"
}

fast_function() {
    echo "Fonction rapide"
    echo "Traitement instantané"
}

error_prone_function() {
    local error_chance=$((RANDOM % 3))
    if (( error_chance == 0 )); then
        echo "Erreur simulée"
        return 1
    else
        echo "Exécution réussie"
        return 0
    fi
}

# Démonstration de l'instrumentation
echo "--- Collecte de métriques applicatives ---"

ApplicationMetrics "app_metrics"

# Instrumentation automatique
app_metrics.instrument_function "slow_function"
app_metrics.instrument_function "fast_function"
app_metrics.instrument_function "error_prone_function"

# Tests des fonctions instrumentées
echo "--- Test des fonctions instrumentées ---"

slow_function
echo
fast_function
echo
error_prone_function
echo
error_prone_function
echo
error_prone_function

echo
echo "--- Métriques manuelles ---"

# Métriques manuelles
app_metrics.set_gauge "active_users" "42" "Utilisateurs actifs"
app_metrics.increment_counter "requests_total" "1" "Requêtes totales"
app_metrics.increment_counter "requests_total" "1" "Requêtes totales"
app_metrics.record_histogram "response_time" "150" "Temps de réponse"
app_metrics.record_histogram "response_time" "200" "Temps de réponse"
app_metrics.record_histogram "response_time" "120" "Temps de réponse"

echo
echo "--- Export des métriques ---"
app_metrics.export_metrics "text"

echo
echo "--- Export Prometheus ---"
app_metrics.export_metrics "prometheus"
```

## Section 2 : Systèmes d'alertes intelligents

### 2.1 Moteur d'alertes multi-canaux

Système d'alertes adaptatives avec escalade automatique :

```bash
#!/bin/bash

# Système d'alertes intelligents
echo "=== Système d'alertes intelligents ==="

# Moteur d'alertes multi-canaux
AlertEngine() {
    local self="$1"
    
    declare -a $self._alert_rules
    declare -A $self._active_alerts
    declare -A $self._alert_history
    declare -A $self._notification_channels
    
    # Définition d'une règle d'alerte
    $self.add_alert_rule() {
        local name="$1"
        local condition="$2"
        local severity="$3"
        local message="$4"
        local channels="$5"
        local cooldown="${6:-300}"  # 5 minutes par défaut
        
        $self._alert_rules+=("$name")
        $self._alert_rules["${name}_condition"]="$condition"
        $self._alert_rules["${name}_severity"]="$severity"
        $self._alert_rules["${name}_message"]="$message"
        $self._alert_rules["${name}_channels"]="$channels"
        $self._alert_rules["${name}_cooldown"]="$cooldown"
        $self._alert_rules["${name}_last_triggered"]="0"
        
        echo "✓ Règle d'alerte ajoutée: $name (sévérité: $severity)"
    }
    
    # Configuration d'un canal de notification
    $self.configure_channel() {
        local name="$1"
        local type="$2"
        local config="$3"
        
        $self._notification_channels["${name}_type"]="$type"
        $self._notification_channels["${name}_config"]="$config"
        
        echo "✓ Canal configuré: $name ($type)"
    }
    
    # Évaluation des règles d'alertes
    $self.evaluate_alerts() {
        local current_time
        current_time="$(date +%s)"
        
        echo "--- ÉVALUATION DES ALERTES ---"
        
        local triggered=0
        
        for rule_name in "${$self._alert_rules[@]}"; do
            local condition="${$self._alert_rules[${rule_name}_condition]}"
            local cooldown="${$self._alert_rules[${rule_name}_cooldown]}"
            local last_triggered="${$self._alert_rules[${rule_name}_last_triggered]}"
            
            # Vérification du cooldown
            if (( current_time - last_triggered < cooldown )); then
                continue
            fi
            
            # Évaluation de la condition
            if $self._evaluate_condition "$condition"; then
                $self._trigger_alert "$rule_name"
                $self._alert_rules["${rule_name}_last_triggered"]="$current_time"
                ((triggered++))
            fi
        done
        
        echo "Alertes déclenchées: $triggered"
    }
    
    # Évaluation d'une condition
    $self._evaluate_condition() {
        local condition="$1"
        
        # Conditions prédéfinies
        case "$condition" in
            cpu_high)
                local cpu_usage
                cpu_usage="$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')"
                (( $(echo "$cpu_usage > 80" | bc -l) ))
                ;;
            memory_high)
                local mem_usage
                mem_usage="$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')"
                (( mem_usage > 85 ))
                ;;
            disk_full)
                local disk_usage
                disk_usage="$(df / | tail -1 | awk '{print $5}' | sed 's/%//')"
                (( disk_usage > 90 ))
                ;;
            load_high)
                local load_avg
                load_avg="$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)"
                (( $(echo "$load_avg > $(nproc)" | bc -l) ))
                ;;
            service_down)
                ! systemctl is-active --quiet nginx 2>/dev/null
                ;;
            network_down)
                ! ping -c 1 -W 2 8.8.8.8 >/dev/null 2>&1
                ;;
            custom:*)
                # Condition personnalisée
                local custom_condition="${condition#custom:}"
                eval "$custom_condition"
                ;;
            *)
                echo "Condition inconnue: $condition" >&2
                false
                ;;
        esac
    }
    
    # Déclenchement d'une alerte
    $self._trigger_alert() {
        local rule_name="$1"
        
        local severity="${$self._alert_rules[${rule_name}_severity]}"
        local message="${$self._alert_rules[${rule_name}_message]}"
        local channels="${$self._alert_rules[${rule_name}_channels]}"
        
        local alert_id="${rule_name}_$(date +%s)"
        
        # Marquage comme alerte active
        $self._active_alerts["$alert_id"]="$rule_name"
        
        # Enregistrement dans l'historique
        $self._alert_history["${alert_id}_timestamp"]="$(date +%s)"
        $self._alert_history["${alert_id}_severity"]="$severity"
        $self._alert_history["${alert_id}_message"]="$message"
        
        echo "🚨 ALERTE [$severity]: $message"
        
        # Notification sur tous les canaux
        IFS=',' read -ra channel_list <<< "$channels"
        for channel in "${channel_list[@]}"; do
            $self._send_notification "$channel" "$severity" "$message" "$rule_name"
        done
    }
    
    # Envoi de notification
    $self._send_notification() {
        local channel="$1"
        local severity="$2"
        local message="$3"
        local rule_name="$4"
        
        local channel_type="${$self._notification_channels[${channel}_type]}"
        local channel_config="${$self._notification_channels[${channel}_config]}"
        
        echo "  Notification via $channel ($channel_type)"
        
        case "$channel_type" in
            email)
                # Simulation d'envoi d'email
                local recipient="$channel_config"
                echo "    To: $recipient"
                echo "    Subject: [$severity] Alerte système: $rule_name"
                echo "    Body: $message"
                echo "    Timestamp: $(date)"
                ;;
                
            webhook)
                # Simulation d'appel webhook
                local url="$channel_config"
                local payload="{\"severity\":\"$severity\",\"message\":\"$message\",\"rule\":\"$rule_name\",\"timestamp\":\"$(date -Iseconds)\"}"
                echo "    URL: $url"
                echo "    Payload: $payload"
                # curl -X POST "$url" -H "Content-Type: application/json" -d "$payload" 2>/dev/null || true
                ;;
                
            slack)
                # Simulation Slack
                local webhook_url="$channel_config"
                local slack_payload="{\"text\":\"[$severity] $message\",\"username\":\"AlertBot\",\"icon_emoji\":\":warning:\"}"
                echo "    Slack webhook: $webhook_url"
                echo "    Payload: $slack_payload"
                ;;
                
            log)
                # Écriture dans un fichier de log
                local log_file="$channel_config"
                echo "[$(date)] [$severity] $rule_name: $message" >> "$log_file"
                echo "    Loggé dans: $log_file"
                ;;
                
            console|*)
                # Affichage console
                echo "    [$severity] $message"
                ;;
        esac
    }
    
    # Résolution d'alerte
    $self.resolve_alert() {
        local alert_id="$1"
        
        if [[ -n "${$self._active_alerts[$alert_id]}" ]]; then
            unset $self._active_alerts["$alert_id"]
            echo "✓ Alerte résolue: $alert_id"
        else
            echo "⚠️  Alerte inconnue: $alert_id"
        fi
    }
    
    # État des alertes
    $self.alert_status() {
        echo "=== ÉTAT DES ALERTES ==="
        
        echo "Alertes actives:"
        if (( ${#$self._active_alerts[@]} > 0 )); then
            for alert_id in "${!$self._active_alerts[@]}"; do
                local rule_name="${$self._active_alerts[$alert_id]}"
                local severity="${$self._alert_rules[${rule_name}_severity]}"
                local timestamp="${$self._alert_history[${alert_id}_timestamp]}"
                
                printf "  %-20s %-8s %s\n" "$alert_id" "[$severity]" "$(date -d "@$timestamp" '+%H:%M:%S')"
            done
        else
            echo "  Aucune alerte active"
        fi
        
        echo
        echo "Statistiques:"
        local total_alerts="${#$self._alert_history[@]}"
        local active_alerts="${#$self._active_alerts[@]}"
        local resolved_alerts=$((total_alerts / 4 - active_alerts))  # Approximation
        
        echo "  Total: $total_alerts événements"
        echo "  Actives: $active_alerts"
        echo "  Résolues: $resolved_alerts"
    }
    
    # Nettoyage des anciennes alertes
    $self.cleanup_old_alerts() {
        local max_age="${1:-86400}"  # 24h par défaut
        
        local current_time
        current_time="$(date +%s)"
        local cleaned=0
        
        for alert_id in "${!$self._alert_history[@]}"; do
            if [[ "$alert_id" =~ _timestamp$ ]]; then
                local base_id="${alert_id%_timestamp}"
                local timestamp="${$self._alert_history[$alert_id]}"
                
                if (( current_time - timestamp > max_age )); then
                    unset $self._alert_history["${base_id}_timestamp"]
                    unset $self._alert_history["${base_id}_severity"]
                    unset $self._alert_history["${base_id}_message"]
                    ((cleaned++))
                fi
            fi
        done
        
        echo "✓ $cleaned anciennes alertes nettoyées"
    }
}

# Démonstration du système d'alertes
echo "--- Système d'alertes intelligents ---"

AlertEngine "alerts"

# Configuration des canaux de notification
alerts.configure_channel "console" "console" ""
alerts.configure_channel "email_admin" "email" "admin@example.com"
alerts.configure_channel "webhook" "webhook" "https://api.example.com/alerts"
alerts.configure_channel "system_log" "log" "/var/log/system_alerts.log"

# Définition des règles d'alertes
alerts.add_alert_rule "high_cpu" "cpu_high" "WARNING" "Utilisation CPU élevée détectée" "console,email_admin" 60
alerts.add_alert_rule "high_memory" "memory_high" "CRITICAL" "Mémoire système critique" "console,webhook,email_admin" 120
alerts.add_alert_rule "disk_full" "disk_full" "CRITICAL" "Disque presque plein" "console,webhook" 300
alerts.add_alert_rule "high_load" "load_high" "WARNING" "Charge système élevée" "console" 180
alerts.add_alert_rule "nginx_down" "service_down" "CRITICAL" "Service nginx arrêté" "console,email_admin,webhook" 30
alerts.add_alert_rule "network_issues" "network_down" "WARNING" "Problèmes de connectivité réseau" "console" 60

# Évaluation initiale
alerts.evaluate_alerts

echo
echo "--- Simulation de conditions d'alerte ---"

# Simulation de haute charge CPU (commenté pour éviter la surcharge)
# for i in {1..4}; do (while true; do true; done) & done
# sleep 2

# Simulation d'espace disque faible (commenté pour éviter les problèmes)
# dd if=/dev/zero of=/tmp/fill_disk bs=1M count=100 2>/dev/null || true

# Évaluation après simulation
alerts.evaluate_alerts

echo
echo "--- État des alertes ---"
alerts.alert_status

echo
echo "--- Nettoyage ---"
alerts.cleanup_old_alerts 1  # Nettoie tout pour la démo

# Nettoyage des fichiers de test
rm -f /tmp/fill_disk
```

### 2.2 Agrégation et corrélation d'alertes

Système intelligent de groupement et d'analyse d'alertes :

```bash
#!/bin/bash

# Agrégation et corrélation d'alertes
echo "=== Agrégation et corrélation d'alertes ==="

# Système d'agrégation d'alertes
AlertAggregator() {
    local self="$1"
    
    declare -A $self._alert_groups
    declare -A $self._correlation_rules
    declare -a $self._alert_buffer
    
    # Définition d'un groupe d'alertes
    $self.define_alert_group() {
        local group_name="$1"
        local pattern="$2"
        local aggregation_window="${3:-300}"
        local min_alerts="${4:-2}"
        local description="$5"
        
        $self._alert_groups["${group_name}_pattern"]="$pattern"
        $self._alert_groups["${group_name}_window"]="$aggregation_window"
        $self._alert_groups["${group_name}_min_alerts"]="$min_alerts"
        $self._alert_groups["${group_name}_description"]="$description"
        $self._alert_groups["${group_name}_alerts"]=""
        
        echo "✓ Groupe d'alertes défini: $group_name"
    }
    
    # Ajout d'une alerte brute
    $self.add_alert() {
        local alert_id="$1"
        local severity="$2"
        local message="$3"
        local source="$4"
        local timestamp="${5:-$(date +%s)}"
        
        local alert_data="$timestamp|$severity|$message|$source|$alert_id"
        $self._alert_buffer+=("$alert_data")
        
        echo "✓ Alerte ajoutée: $alert_id ($severity) - $message"
        
        # Traitement immédiat des corrélations
        $self._process_correlations "$alert_data"
    }
    
    # Traitement des corrélations
    $self._process_correlations() {
        local alert_data="$1"
        
        IFS='|' read -r timestamp severity message source alert_id <<< "$alert_data"
        
        # Recherche de groupes correspondants
        for group_key in "${!$self._alert_groups[@]}"; do
            if [[ "$group_key" =~ _pattern$ ]]; then
                local group_name="${group_key%_pattern}"
                local pattern="${$self._alert_groups[$group_key]}"
                
                if [[ "$message" =~ $pattern || "$source" =~ $pattern ]]; then
                    $self._add_to_group "$group_name" "$alert_data"
                fi
            fi
        done
    }
    
    # Ajout à un groupe
    $self._add_to_group() {
        local group_name="$1"
        local alert_data="$2"
        
        local group_alerts="${$self._alert_groups[${group_name}_alerts]}"
        
        if [[ -z "$group_alerts" ]]; then
            $self._alert_groups["${group_name}_alerts"]="$alert_data"
            $self._alert_groups["${group_name}_first_alert"]="$(echo "$alert_data" | cut -d'|' -f1)"
        else
            $self._alert_groups["${group_name}_alerts"]="${group_alerts};${alert_data}"
        fi
        
        $self._check_group_threshold "$group_name"
    }
    
    # Vérification du seuil de déclenchement du groupe
    $self._check_group_threshold() {
        local group_name="$1"
        
        local group_alerts="${$self._alert_groups[${group_name}_alerts]}"
        local min_alerts="${$self._alert_groups[${group_name}_min_alerts]}"
        local window="${$self._alert_groups[${group_name}_window]}"
        local first_alert="${$self._alert_groups[${group_name}_first_alert]}"
        local description="${$self._alert_groups[${group_name}_description]}"
        
        # Comptage des alertes dans la fenêtre
        local alert_count=0
        local current_time="$(date +%s)"
        local window_start=$((current_time - window))
        
        IFS=';' read -ra alerts <<< "$group_alerts"
        for alert in "${alerts[@]}"; do
            local alert_time="$(echo "$alert" | cut -d'|' -f1)"
            if (( alert_time >= window_start )); then
                ((alert_count++))
            fi
        done
        
        # Déclenchement si seuil atteint
        if (( alert_count >= min_alerts )); then
            local severity="WARNING"
            if (( alert_count >= min_alerts * 2 )); then
                severity="CRITICAL"
            fi
            
            echo "🚨 ALERTE AGRÉGÉE [$severity]: $description"
            echo "  Groupe: $group_name"
            echo "  Alertes dans la fenêtre: $alert_count"
            echo "  Fenêtre: ${window}s"
            echo "  Description: $description"
            
            # Réinitialisation du groupe après déclenchement
            $self._alert_groups["${group_name}_alerts"]=""
            $self._alert_groups["${group_name}_first_alert"]=""
        fi
    }
    
    # Règles de corrélation
    $self.add_correlation_rule() {
        local rule_name="$1"
        local condition="$2"
        local action="$3"
        local description="$4"
        
        $self._correlation_rules["${rule_name}_condition"]="$condition"
        $self._correlation_rules["${rule_name}_action"]="$action"
        $self._correlation_rules["${rule_name}_description"]="$description"
        
        echo "✓ Règle de corrélation ajoutée: $rule_name"
    }
    
    # Analyse des corrélations complexes
    $self.analyze_correlations() {
        echo "=== ANALYSE DES CORRÉLATIONS ==="
        
        local current_time="$(date +%s)"
        local analysis_window=600  # 10 minutes
        
        # Collecte des alertes récentes
        local recent_alerts=()
        for alert_data in "${$self._alert_buffer[@]}"; do
            local alert_time="$(echo "$alert_data" | cut -d'|' -f1)"
            if (( current_time - alert_time <= analysis_window )); then
                recent_alerts+=("$alert_data")
            fi
        done
        
        echo "Alertes analysées: ${#recent_alerts[@]} dans les ${analysis_window}s"
        
        # Application des règles de corrélation
        for rule_key in "${!$self._correlation_rules[@]}"; do
            if [[ "$rule_key" =~ _condition$ ]]; then
                local rule_name="${rule_key%_condition}"
                local condition="${$self._correlation_rules[$rule_key]}"
                local action="${$self._correlation_rules[${rule_name}_action]}"
                local description="${$self._correlation_rules[${rule_name}_description]}"
                
                if $self._evaluate_correlation "$condition" recent_alerts; then
                    echo "🔗 CORRÉLATION DÉTECTÉE: $description"
                    echo "  Règle: $rule_name"
                    echo "  Action recommandée: $action"
                fi
            fi
        done
    }
    
    # Évaluation d'une règle de corrélation
    $self._evaluate_correlation() {
        local condition="$1"
        local -a alerts=("${!2}")
        
        case "$condition" in
            multiple_cpu_alerts)
                # Plusieurs alertes CPU dans un court intervalle
                local cpu_alerts=0
                for alert in "${alerts[@]}"; do
                    if [[ "$alert" =~ cpu ]]; then
                        ((cpu_alerts++))
                    fi
                done
                (( cpu_alerts >= 3 ))
                ;;
                
            disk_and_memory_pressure)
                # Pression disque ET mémoire
                local disk_alerts=0 memory_alerts=0
                for alert in "${alerts[@]}"; do
                    if [[ "$alert" =~ disk ]]; then ((disk_alerts++)); fi
                    if [[ "$alert" =~ memory ]]; then ((memory_alerts++)); fi
                done
                (( disk_alerts >= 1 && memory_alerts >= 1 ))
                ;;
                
            network_service_correlation)
                # Problèmes réseau ET services
                local network_alerts=0 service_alerts=0
                for alert in "${alerts[@]}"; do
                    if [[ "$alert" =~ network ]]; then ((network_alerts++)); fi
                    if [[ "$alert" =~ service ]]; then ((service_alerts++)); fi
                done
                (( network_alerts >= 1 && service_alerts >= 2 ))
                ;;
                
            custom:*)
                # Condition personnalisée
                local custom_condition="${condition#custom:}"
                # Évaluation simplifiée
                [[ "$custom_condition" == "true" ]]
                ;;
                
            *)
                false
                ;;
        esac
    }
    
    # Rapport d'état
    $self.status_report() {
        echo "=== RAPPORT D'AGRÉGATION D'ALERTES ==="
        
        echo "Groupes d'alertes actifs:"
        for group_key in "${!$self._alert_groups[@]}"; do
            if [[ "$group_key" =~ _alerts$ && -n "${$self._alert_groups[$group_key]}" ]]; then
                local group_name="${group_key%_alerts}"
                local alert_count="$(echo "${$self._alert_groups[$group_key]}" | tr -cd ';' | wc -c)"
                ((alert_count++))
                
                echo "  $group_name: $alert_count alertes"
            fi
        done
        
        echo
        echo "Alertes dans le buffer: ${#$self._alert_buffer[@]}"
        
        echo
        echo "Règles de corrélation: $((${#$self._correlation_rules[@]} / 3))"
    }
    
    # Nettoyage
    $self.cleanup() {
        $self._alert_buffer=()
        
        for group_key in "${!$self._alert_groups[@]}"; do
            if [[ "$group_key" =~ _alerts$ ]]; then
                $self._alert_groups["$group_key"]=""
            fi
        done
        
        echo "✓ Buffer d'alertes nettoyé"
    }
}

# Démonstration de l'agrégation d'alertes
echo "--- Agrégation et corrélation d'alertes ---"

AlertAggregator "aggregator"

# Définition des groupes d'alertes
aggregator.define_alert_group "cpu_storm" "cpu" 60 3 "Tempête d'alertes CPU détectée"
aggregator.define_alert_group "memory_pressure" "memory" 120 2 "Pression mémoire continue"
aggregator.define_alert_group "disk_issues" "disk" 300 2 "Problèmes de stockage persistants"

# Définition des règles de corrélation
aggregator.add_correlation_rule "cpu_memory_crisis" "disk_and_memory_pressure" "Augmenter la surveillance, préparer plan de contingence" "Crise simultanée disque/mémoire"
aggregator.add_correlation_rule "infrastructure_failure" "network_service_correlation" "Activer procédure de failover" "Échec d'infrastructure réseau"
aggregator.add_correlation_rule "system_overload" "multiple_cpu_alerts" "Réduire la charge, optimiser les processus" "Surcharge système généralisée"

# Simulation d'alertes
echo
echo "--- Simulation d'alertes ---"

# Alertes isolées (pas de déclenchement de groupe)
aggregator.add_alert "cpu_001" "WARNING" "CPU usage 85%" "system_monitor" "$(date +%s)"

# Alertes groupées CPU
aggregator.add_alert "cpu_002" "WARNING" "CPU usage 87%" "system_monitor" "$(date +%s)"
aggregator.add_alert "cpu_003" "WARNING" "CPU usage 89%" "system_monitor" "$(( $(date +%s) + 10 ))"
aggregator.add_alert "cpu_004" "WARNING" "CPU usage 91%" "system_monitor" "$(( $(date +%s) + 20 ))"

# Alertes mémoire et disque (pour corrélation)
aggregator.add_alert "mem_001" "CRITICAL" "Memory usage 92%" "system_monitor" "$(date +%s)"
aggregator.add_alert "disk_001" "WARNING" "Disk usage 88%" "system_monitor" "$(date +%s)"
aggregator.add_alert "disk_002" "CRITICAL" "Disk usage 95%" "system_monitor" "$(( $(date +%s) + 30 ))"

# Analyse des corrélations
echo
aggregator.analyze_correlations

echo
echo "--- Rapport d'état ---"
aggregator.status_report

echo
echo "--- Nettoyage ---"
aggregator.cleanup
```

## Section 3 : Agrégation et analyse de logs distribués

### 3.1 Système d'agrégation de logs

Collecte et analyse centralisée des logs :

```bash
#!/bin/bash

# Système d'agrégation de logs distribués
echo "=== Système d'agrégation de logs distribués ==="

# Agrégateur de logs
LogAggregator() {
    local self="$1"
    
    declare -A $self._log_sources
    declare -a $self._log_entries
    declare -A $self._log_filters
    declare -A $self._log_stats
    
    # Ajout d'une source de logs
    $self.add_log_source() {
        local name="$1"
        local path="$2"
        local format="$3"
        local tags="$4"
        
        $self._log_sources["${name}_path"]="$path"
        $self._log_sources["${name}_format"]="$format"
        $self._log_sources["${name}_tags"]="$tags"
        $self._log_sources["${name}_last_position"]="0"
        
        echo "✓ Source de logs ajoutée: $name ($path)"
    }
    
    # Collecte des logs
    $self.collect_logs() {
        local max_entries="${1:-100}"
        
        echo "--- COLLECTE DES LOGS ---"
        
        local collected=0
        
        for source_key in "${!$self._log_sources[@]}"; do
            if [[ "$source_key" =~ _path$ ]]; then
                local source_name="${source_key%_path}"
                local source_path="${$self._log_sources[$source_key]}"
                
                if [[ -f "$source_path" ]]; then
                    local entries_read
                    entries_read="$($self._read_log_entries "$source_path" "$source_name" "$max_entries")"
                    collected=$((collected + entries_read))
                fi
            fi
        done
        
        echo "Entries collectées: $collected"
        
        # Application des filtres
        $self._apply_filters
    }
    
    # Lecture des entrées de log
    $self._read_log_entries() {
        local file_path="$1"
        local source_name="$2"
        local max_entries="$3"
        
        local last_position="${$self._log_sources[${source_name}_last_position]}"
        local current_size
        current_size="$(stat -f%z "$file_path" 2>/dev/null || stat -c%s "$file_path")"
        
        # Si le fichier a été tronqué ou réinitialisé
        if (( current_size < last_position )); then
            last_position=0
        fi
        
        local new_entries=0
        
        # Lecture des nouvelles lignes
        while IFS= read -r line && (( new_entries < max_entries )); do
            if [[ -n "$line" ]]; then
                local timestamp
                timestamp="$(date +%s)"
                
                local log_entry="$timestamp|$source_name|$line"
                $self._log_entries+=("$log_entry")
                
                ((new_entries++))
            fi
        done < <(tail -n +$((last_position + 1)) "$file_path" 2>/dev/null | head -n "$max_entries")
        
        # Mise à jour de la position
        $self._log_sources["${source_name}_last_position"]="$((last_position + new_entries))"
        
        echo "$new_entries"
    }
    
    # Définition d'un filtre
    $self.add_filter() {
        local name="$1"
        local pattern="$2"
        local action="$3"  # include, exclude, highlight, alert
        local priority="${4:-10}"
        
        $self._log_filters["${name}_pattern"]="$pattern"
        $self._log_filters["${name}_action"]="$action"
        $self._log_filters["${name}_priority"]="$priority"
        
        echo "✓ Filtre ajouté: $name ($action)"
    }
    
    # Application des filtres
    $self._apply_filters() {
        local -a filtered_entries
        
        for entry in "${$self._log_entries[@]}"; do
            local filtered_entry
            filtered_entry="$($self._apply_entry_filters "$entry")"
            
            if [[ -n "$filtered_entry" ]]; then
                filtered_entries+=("$filtered_entry")
            fi
        done
        
        $self._log_entries=("${filtered_entries[@]}")
    }
    
    # Application des filtres à une entrée
    $self._apply_entry_filters() {
        local entry="$1"
        
        IFS='|' read -r timestamp source line <<< "$entry"
        
        local include=true
        local highlight=false
        local alert=false
        
        # Application de tous les filtres par priorité
        for filter_key in "${!$self._log_filters[@]}"; do
            if [[ "$filter_key" =~ _pattern$ ]]; then
                local filter_name="${filter_key%_pattern}"
                local pattern="${$self._log_filters[$filter_key]}"
                local action="${$self._log_filters[${filter_name}_action]}"
                
                if [[ "$line" =~ $pattern ]]; then
                    case "$action" in
                        exclude)
                            include=false
                            ;;
                        highlight)
                            highlight=true
                            ;;
                        alert)
                            alert=true
                            ;;
                        include)
                            include=true
                            ;;
                    esac
                fi
            fi
        done
        
        # Formatage de l'entrée
        if [[ "$include" == "true" ]]; then
            local formatted_entry="$entry"
            
            if [[ "$highlight" == "true" ]]; then
                formatted_entry="$formatted_entry|HIGHLIGHT"
            fi
            
            if [[ "$alert" == "true" ]]; then
                formatted_entry="$formatted_entry|ALERT"
            fi
            
            echo "$formatted_entry"
        fi
    }
    
    # Analyse des logs
    $self.analyze_logs() {
        echo "=== ANALYSE DES LOGS ==="
        
        # Statistiques par source
        declare -A source_counts
        declare -A severity_counts
        declare -A error_patterns
        
        for entry in "${$self._log_entries[@]}"; do
            IFS='|' read -r timestamp source line flags <<< "$entry"
            
            # Comptage par source
            ((source_counts["$source"]++))
            
            # Détection de sévérité
            if [[ "$line" =~ (ERROR|CRITICAL|FATAL) ]]; then
                ((severity_counts["ERROR"]++))
            elif [[ "$line" =~ (WARNING|WARN) ]]; then
                ((severity_counts["WARNING"]++))
            elif [[ "$line" =~ (INFO|NOTICE) ]]; then
                ((severity_counts["INFO"]++))
            else
                ((severity_counts["UNKNOWN"]++))
            fi
            
            # Patterns d'erreur courants
            if [[ "$line" =~ "connection refused" ]]; then
                ((error_patterns["connection_refused"]++))
            elif [[ "$line" =~ "permission denied" ]]; then
                ((error_patterns["permission_denied"]++))
            elif [[ "$line" =~ "timeout" ]]; then
                ((error_patterns["timeout"]++))
            fi
        done
        
        echo "Statistiques par source:"
        for source in "${!source_counts[@]}"; do
            echo "  $source: ${source_counts[$source]} entrées"
        done
        
        echo
        echo "Répartition par sévérité:"
        for severity in "${!severity_counts[@]}"; do
            echo "  $severity: ${severity_counts[$severity]}"
        done
        
        echo
        echo "Patterns d'erreur détectés:"
        for pattern in "${!error_patterns[@]}"; do
            echo "  $pattern: ${error_patterns[$pattern]} occurrences"
        done
    }
    
    # Recherche dans les logs
    $self.search_logs() {
        local query="$1"
        local source_filter="${2:-}"
        
        echo "=== RECHERCHE DANS LES LOGS ==="
        echo "Requête: '$query'"
        if [[ -n "$source_filter" ]]; then
            echo "Filtre source: $source_filter"
        fi
        
        local matches=0
        
        for entry in "${$self._log_entries[@]}"; do
            IFS='|' read -r timestamp source line flags <<< "$entry"
            
            # Application du filtre source
            if [[ -n "$source_filter" && "$source" != "$source_filter" ]]; then
                continue
            fi
            
            # Recherche
            if [[ "$line" =~ $query ]]; then
                local time_str
                time_str="$(date -d "@$timestamp" '+%H:%M:%S')"
                
                echo "[$time_str] $source: $line"
                ((matches++))
            fi
        done
        
        echo "Résultats trouvés: $matches"
    }
    
    # Export des logs
    $self.export_logs() {
        local format="${1:-text}"
        local file_path="$2"
        
        case "$format" in
            json)
                $self._export_json "$file_path"
                ;;
            csv)
                $self._export_csv "$file_path"
                ;;
            text|*)
                $self._export_text "$file_path"
                ;;
        esac
        
        echo "✓ Logs exportés: $file_path (format: $format)"
    }
    
    # Export JSON
    $self._export_json() {
        local file_path="$1"
        
        echo "[" > "$file_path"
        
        local first=true
        for entry in "${$self._log_entries[@]}"; do
            IFS='|' read -r timestamp source line flags <<< "$entry"
            
            if [[ "$first" == "true" ]]; then
                first=false
            else
                echo "," >> "$file_path"
            fi
            
            cat >> "$file_path" << EOF
  {
    "timestamp": $timestamp,
    "source": "$source",
    "message": "$line",
    "flags": "$flags"
  }
EOF
        done
        
        echo "]" >> "$file_path"
    }
    
    # Export CSV
    $self._export_csv() {
        local file_path="$1"
        
        echo "timestamp,source,message,flags" > "$file_path"
        
        for entry in "${$self._log_entries[@]}"; do
            IFS='|' read -r timestamp source line flags <<< "$entry"
            echo "$timestamp,$source,\"$line\",$flags" >> "$file_path"
        done
    }
    
    # Export texte
    $self._export_text() {
        local file_path="$1"
        
        {
            echo "=== LOGS AGRÉGÉS ==="
            echo "Généré le: $(date)"
            echo "Total entries: ${#$self._log_entries[@]}"
            echo
            
            for entry in "${$self._log_entries[@]}"; do
                IFS='|' read -r timestamp source line flags <<< "$entry"
                local time_str
                time_str="$(date -d "@$timestamp" '+%H:%M:%S')"
                
                if [[ "$flags" =~ HIGHLIGHT ]]; then
                    echo "🔍 [$time_str] $source: $line"
                elif [[ "$flags" =~ ALERT ]]; then
                    echo "🚨 [$time_str] $source: $line"
                else
                    echo "[$time_str] $source: $line"
                fi
            done
        } > "$file_path"
    }
    
    # État du système
    $self.status() {
        echo "=== ÉTAT DE L'AGRÉGATEUR DE LOGS ==="
        
        echo "Sources configurées: ${#$self._log_sources[@]} / 3"  # path, format, tags par source
        
        echo "Entrées en mémoire: ${#$self._log_entries[@]}"
        
        echo "Filtres actifs: $((${#$self._log_filters[@]} / 3))"  # pattern, action, priority par filtre
        
        echo
        echo "Sources détaillées:"
        for source_key in "${!$self._log_sources[@]}"; do
            if [[ "$source_key" =~ _path$ ]]; then
                local source_name="${source_key%_path}"
                local path="${$self._log_sources[$source_key]}"
                local last_pos="${$self._log_sources[${source_name}_last_position]}"
                
                echo "  $source_name: $path (position: $last_pos)"
            fi
        done
    }
}

# Démonstration de l'agrégateur de logs
echo "--- Agrégateur de logs distribués ---"

LogAggregator "log_agg"

# Configuration des sources de logs
log_agg.add_log_source "system" "/var/log/syslog" "syslog" "system,kernel"
log_agg.add_log_source "auth" "/var/log/auth.log" "auth" "security,authentication"
log_agg.add_log_source "app" "/tmp/app.log" "custom" "application,user"

# Création de logs de test
echo "[$(date)] Application démarrée" > /tmp/app.log
echo "[$(date)] Connexion utilisateur alice" >> /tmp/app.log
echo "[$(date)] Erreur: connection refused" >> /tmp/app.log
echo "[$(date)] Avertissement: espace disque faible" >> /tmp/app.log
echo "[$(date)] Erreur: permission denied" >> /tmp/app.log

# Définition des filtres
log_agg.add_filter "errors" "(ERROR|CRITICAL|FATAL)" "highlight" 10
log_agg.add_filter "warnings" "(WARNING|WARN)" "highlight" 5
log_agg.add_filter "security" "(authentication|permission)" "alert" 20
log_agg.add_filter "spam" "debug" "exclude" 1

# Collecte des logs
log_agg.collect_logs 50

echo
echo "--- Analyse des logs ---"
log_agg.analyze_logs

echo
echo "--- Recherche dans les logs ---"
log_agg.search_logs "ERROR"
log_agg.search_logs "connexion" "app"

echo
echo "--- État du système ---"
log_agg.status

echo
echo "--- Exports ---"
log_agg.export_logs "text" "/tmp/logs_export.txt"
log_agg.export_logs "json" "/tmp/logs_export.json"

echo "Fichiers exportés:"
head -10 /tmp/logs_export.txt

# Nettoyage
rm -f /tmp/app.log /tmp/logs_export.txt /tmp/logs_export.json
```

### 3.2 Tableaux de bord et visualisation

Création d'interfaces de monitoring en texte :

```bash
#!/bin/bash

# Tableaux de bord et visualisation
echo "=== Tableaux de bord et visualisation ==="

# Générateur de tableaux de bord
DashboardGenerator() {
    local self="$1"
    
    declare -a $self._widgets
    declare -A $self._widget_configs
    
    # Ajout d'un widget
    $self.add_widget() {
        local name="$1"
        local type="$2"
        local position="$3"  # row,col,width,height
        local config="$4"
        
        $self._widgets+=("$name")
        $self._widget_configs["${name}_type"]="$type"
        $self._widget_configs["${name}_position"]="$position"
        $self._widget_configs["${name}_config"]="$config"
        
        echo "✓ Widget ajouté: $name ($type)"
    }
    
    # Rendu du tableau de bord
    $self.render_dashboard() {
        local width="${1:-80}"
        local height="${2:-24}"
        
        echo "=== TABLEAU DE BORD SYSTÈME ==="
        echo "Taille: ${width}x${height} | Mise à jour: $(date)"
        echo
        
        # Collecte des données pour tous les widgets
        $self._collect_widget_data
        
        # Rendu des widgets
        for widget in "${$self._widgets[@]}"; do
            $self._render_widget "$widget" "$width"
            echo
        done
    }
    
    # Collecte des données des widgets
    $self._collect_widget_data() {
        # Collecte des métriques système
        $self._system_cpu="$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')"
        $self._system_memory="$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')"
        $self._system_disk="$(df / | tail -1 | awk '{print $5}' | sed 's/%//')"
        $self._system_load="$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)"
        
        # Collecte des informations réseau
        $self._network_interfaces="$(ip route show | grep -c "dev lo")"
        $self._network_connections="$(ss -tun | wc -l)"
        
        # Collecte des processus
        $self._process_total="$(ps aux | wc -l)"
        $self._process_zombie="$(ps aux | awk '{print $8}' | grep -c 'Z')"
        
        # Historique (simulation)
        local history_size=10
        for ((i=1; i<=history_size; i++)); do
            $self._cpu_history[$i]="$((RANDOM % 100))"
            $self._mem_history[$i]="$((RANDOM % 100))"
        done
    }
    
    # Rendu d'un widget
    $self._render_widget() {
        local widget_name="$1"
        local max_width="$2"
        
        local widget_type="${$self._widget_configs[${widget_name}_type]}"
        local widget_config="${$self._widget_configs[${widget_name}_config]}"
        
        echo "┌─ $widget_name ──────────────────────────────────────┐"
        
        case "$widget_type" in
            gauge)
                $self._render_gauge_widget "$widget_name" "$widget_config"
                ;;
            chart)
                $self._render_chart_widget "$widget_name" "$widget_config"
                ;;
            list)
                $self._render_list_widget "$widget_name" "$widget_config"
                ;;
            text)
                $self._render_text_widget "$widget_name" "$widget_config"
                ;;
            *)
                echo "│ Type de widget inconnu: $widget_type"
                ;;
        esac
        
        echo "└─────────────────────────────────────────────────────┘"
    }
    
    # Widget jauge (pour pourcentages)
    $self._render_gauge_widget() {
        local widget_name="$1"
        local config="$2"
        
        # Parsing de la config (metric:label:unit)
        local metric label unit
        IFS=':' read -r metric label unit <<< "$config"
        
        local value_var="_${metric}"
        local value="${!value_var}"
        
        if [[ -z "$value" ]]; then
            value="N/A"
        fi
        
        # Barre de progression
        local bar_width=40
        local filled=0
        
        if [[ "$value" =~ ^[0-9]+(\.[0-9]+)?$ ]]; then
            filled=$(( value * bar_width / 100 ))
        fi
        
        local bar=""
        for ((i=1; i<=bar_width; i++)); do
            if (( i <= filled )); then
                bar+="█"
            else
                bar+="░"
            fi
        done
        
        printf "│ %-15s │ %s │ %5s %s │\n" "$label" "$bar" "$value" "$unit"
    }
    
    # Widget graphique
    $self._render_chart_widget() {
        local widget_name="$1"
        local config="$2"
        
        local chart_width=50
        local chart_height=5
        
        case "$config" in
            cpu_history)
                echo "│ Historique CPU (10 dernières mesures):"
                $self._render_sparkline "cpu_history" "$chart_width"
                ;;
            memory_history)
                echo "│ Historique Mémoire:"
                $self._render_sparkline "memory_history" "$chart_width"
                ;;
            *)
                echo "│ Configuration graphique inconnue: $config"
                ;;
        esac
    }
    
    # Rendu d'un sparkline
    $self._render_sparkline() {
        local history_var="$1"
        local width="$2"
        
        local -a values=("${!history_var}")
        local max_value=0
        
        # Recherche du maximum
        for value in "${values[@]}"; do
            if (( value > max_value )); then
                max_value="$value"
            fi
        done
        
        # Génération du sparkline
        local sparkline=""
        for value in "${values[@]}"; do
            if (( max_value == 0 )); then
                sparkline+="▁"
            else
                local level=$(( value * 7 / max_value ))
                case "$level" in
                    0) sparkline+="▁" ;;
                    1) sparkline+="▂" ;;
                    2) sparkline+="▃" ;;
                    3) sparkline+="▄" ;;
                    4) sparkline+="▅" ;;
                    5) sparkline+="▆" ;;
                    6|7) sparkline+="▇" ;;
                esac
            fi
        done
        
        printf "│ %s │\n" "$sparkline"
    }
    
    # Widget liste
    $self._render_list_widget() {
        local widget_name="$1"
        local config="$2"
        
        case "$config" in
            top_processes)
                echo "│ Top 5 processus par CPU:"
                ps aux --sort=-%cpu | head -6 | tail -5 | while read -r line; do
                    local pid user cpu mem command
                    read -r user pid cpu mem command <<< "$line"
                    printf "│   %-8s %-5s %4s%% %s │\n" "$pid" "$user" "$cpu" "${command:0:20}"
                done
                ;;
            disk_usage)
                echo "│ Utilisation disque:"
                df -h | grep '^/dev/' | head -3 | while read -r fs size used avail use mount; do
                    printf "│   %-15s %5s/%-5s (%s) │\n" "$mount" "$used" "$size" "$use"
                done
                ;;
            network_status)
                echo "│ Statut réseau:"
                echo "│   Interfaces actives: $($self._network_interfaces)"
                echo "│   Connexions: $($self._network_connections)"
                ;;
            *)
                echo "│ Configuration liste inconnue: $config"
                ;;
        esac
    }
    
    # Widget texte
    $self._render_text_widget() {
        local widget_name="$1"
        local config="$2"
        
        case "$config" in
            system_info)
                echo "│ Informations système:"
                echo "│   OS: $(uname -s) $(uname -r)"
                echo "│   Uptime: $(uptime -p)"
                echo "│   Utilisateur: $(whoami)"
                ;;
            alerts_summary)
                echo "│ Résumé des alertes:"
                echo "│   Actives: 2 (CPU élevé, Disque plein)"
                echo "│   Résolues aujourd'hui: 5"
                echo "│   Critiques: 0"
                ;;
            *)
                echo "│ $config"
                ;;
        esac
    }
    
    # Mise à jour en temps réel
    $self.live_dashboard() {
        local interval="${1:-5}"
        local duration="${2:-60}"
        
        echo "Tableau de bord en temps réel (Ctrl+C pour arrêter)"
        echo "Intervalle: ${interval}s | Durée max: ${duration}s"
        echo
        
        local start_time=$(date +%s)
        
        while (( $(date +%s) - start_time < duration )); do
            clear
            $self.render_dashboard 80 24
            sleep "$interval"
        done
    }
}

# Démonstration du tableau de bord
echo "--- Tableaux de bord et visualisation ---"

DashboardGenerator "dashboard"

# Configuration des widgets
dashboard.add_widget "cpu_gauge" "gauge" "1,1,1,1" "system_cpu:CPU:%%"
dashboard.add_widget "memory_gauge" "gauge" "2,1,1,1" "system_memory:RAM:%%"
dashboard.add_widget "disk_gauge" "gauge" "3,1,1,1" "system_disk:Disque:%%"

dashboard.add_widget "cpu_chart" "chart" "1,2,1,1" "cpu_history"
dashboard.add_widget "memory_chart" "chart" "2,2,1,1" "memory_history"

dashboard.add_widget "top_processes" "list" "3,2,2,1" "top_processes"

dashboard.add_widget "system_info" "text" "1,3,1,1" "system_info"
dashboard.add_widget "network_status" "text" "2,3,1,1" "network_status"
dashboard.add_widget "alerts" "text" "3,3,1,1" "alerts_summary"

# Rendu statique
dashboard.render_dashboard 80 24

echo
echo "--- Version live (simulation courte) ---"
echo "Démarrage du dashboard live pendant 10 secondes..."
dashboard.live_dashboard 2 10

echo "Tableau de bord terminé."
```

## Conclusion : L'observabilité comme fondation

Le monitoring avancé et l'observabilité en Bash transforment les systèmes en entités auto-diagnostiques capables de s'observer, s'analyser, et s'alerter elles-mêmes. Comme un organisme vivant doté de sens aiguisés, vos applications deviennent conscientes de leur état et de leur environnement.

**Points clés à retenir :**

1. **Métriques multi-sources** : Collecteurs spécialisés pour CPU, mémoire, disque, réseau, et applications avec export dans divers formats
2. **Alertes intelligentes** : Systèmes d'alertes adaptatives avec escalade automatique et canaux de notification multiples
3. **Agrégation et corrélation** : Groupement intelligent d'alertes similaires et détection de patterns complexes
4. **Logs distribués** : Collecte, agrégation, et analyse centralisée des logs avec filtrage et recherche avancés
5. **Visualisation temps réel** : Tableaux de bord textuels avec jauges, graphiques, et mises à jour en direct

Dans le chapitre suivant, nous explorerons les techniques de déploiement automatisé et de gestion d'infrastructure, pour que vos scripts d'observabilité puissent être déployés et maintenus à grande échelle.

---

**Exercice pratique :** Créez un système de monitoring complet incluant :
- Collecteurs de métriques système et applicatives
- Moteur d'alertes avec corrélation et escalade
- Agrégateur de logs avec recherche et analyse
- Tableau de bord temps réel avec visualisations
- Export vers Prometheus et intégration webhook

**Réflexion :** Comment adapteriez-vous ces techniques d'observabilité pour surveiller des applications conteneurisées dans Kubernetes ou des fonctions serverless dans le cloud ?
