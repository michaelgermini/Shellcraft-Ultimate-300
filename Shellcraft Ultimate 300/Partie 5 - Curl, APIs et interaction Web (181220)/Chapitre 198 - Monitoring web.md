# Chapitre 198 - Monitoring web

## Table des matières
- [Introduction](#introduction)
- [Monitoring de disponibilité](#monitoring-de-disponibilité)
- [Monitoring de performance](#monitoring-de-performance)
- [Alertes automatiques](#alertes-automatiques)
- [Conclusion](#conclusion)

## Introduction

Le monitoring web avec cURL permet de surveiller la disponibilité et les performances des services web de manière automatisée.

## Monitoring de disponibilité

**Vérification de disponibilité** :
```bash
#!/bin/bash
# Monitoring de disponibilité

check_availability() {
    local url="$1"
    local timeout="${2:-10}"
    
    local http_code=$(curl -s -o /dev/null -w "%{http_code}" \
        --max-time "$timeout" \
        "$url")
    
    if [ "$http_code" -ge 200 ] && [ "$http_code" -lt 400 ]; then
        echo "✓ Disponible (HTTP $http_code)"
        return 0
    else
        echo "✗ Indisponible (HTTP $http_code)"
        return 1
    fi
}

# Monitoring continu
monitor_continuous() {
    local url="$1"
    local interval="${2:-60}"
    
    while true; do
        if ! check_availability "$url"; then
            send_alert "Service indisponible: $url"
        fi
        sleep "$interval"
    done
}
```

## Monitoring de performance

**Mesure de performance** :
```bash
#!/bin/bash
# Monitoring de performance

measure_performance() {
    local url="$1"
    
    local metrics=$(curl -s -o /dev/null -w \
        "time_total:%{time_total}\ntime_connect:%{time_connect}\ntime_starttransfer:%{time_starttransfer}\nsize_download:%{size_download}\n" \
        "$url")
    
    echo "$metrics"
}

# Logging des métriques
log_metrics() {
    local url="$1"
    local log_file="${2:-metrics.log}"
    
    local timestamp=$(date -Iseconds)
    local metrics=$(measure_performance "$url")
    
    echo "[$timestamp] $url" >> "$log_file"
    echo "$metrics" | sed 's/^/  /' >> "$log_file"
}
```

## Alertes automatiques

**Système d'alertes** :
```bash
#!/bin/bash
# Alertes automatiques

send_alert() {
    local message="$1"
    local severity="${2:-warning}"
    
    # Email
    echo "$message" | mail -s "Alert: $severity" admin@example.com
    
    # Slack (si webhook configuré)
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -X POST "$SLACK_WEBHOOK" \
             -H "Content-Type: application/json" \
             -d "{\"text\":\"$message\"}"
    fi
}
```

## Conclusion

Le monitoring web avec cURL permet de surveiller automatiquement la santé des services web.

