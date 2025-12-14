# Chapitre 200 - Alertes et reporting

## Table des matières
- [Introduction](#introduction)
- [Système d'alertes](#système-dalertes)
- [Génération de rapports](#génération-de-rapports)
- [Notification multi-canaux](#notification-multi-canaux)
- [Conclusion](#conclusion)

## Introduction

Les alertes et le reporting automatisent la notification des problèmes et la génération de rapports sur l'état des APIs et services web.

## Système d'alertes

**Alertes configurables** :
```bash
#!/bin/bash
# Système d'alertes

ALERT_THRESHOLD="${ALERT_THRESHOLD:-5000}"  # ms

check_and_alert() {
    local url="$1"
    local response_time=$(curl -s -o /dev/null -w "%{time_total}" "$url" | awk '{print int($1*1000)}')
    
    if [ "$response_time" -gt "$ALERT_THRESHOLD" ]; then
        send_alert "Performance dégradée: $url (${response_time}ms)"
    fi
}
```

## Génération de rapports

**Génération de rapports** :
```bash
#!/bin/bash
# Génération de rapports

generate_report() {
    local period="${1:-daily}"
    local report_file="report_$(date +%Y%m%d).txt"
    
    {
        echo "=== Rapport $period ==="
        echo "Date: $(date)"
        echo
        echo "=== Disponibilité ==="
        check_availability_stats
        echo
        echo "=== Performance ==="
        check_performance_stats
    } > "$report_file"
    
    echo "Rapport généré: $report_file"
}
```

## Notification multi-canaux

**Notifications multiples** :
```bash
#!/bin/bash
# Notification multi-canaux

notify() {
    local message="$1"
    
    # Email
    echo "$message" | mail -s "Alert" admin@example.com
    
    # Slack
    curl -X POST "$SLACK_WEBHOOK" -d "{\"text\":\"$message\"}"
    
    # SMS (via API)
    curl -X POST "$SMS_API" -d "message=$message"
}
```

## Conclusion

Les alertes et le reporting automatisent la surveillance et la communication sur l'état des services.

