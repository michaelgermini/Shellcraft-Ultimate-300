# Chapitre 234 - Monitoring de conteneurs

## Table des matières
- [Introduction](#introduction)
- [Monitoring de base](#monitoring-de-base)
- [Outils de monitoring](#outils-de-monitoring)
- [Alertes et notifications](#alertes-et-notifications)
- [Conclusion](#conclusion)

## Introduction

Le monitoring de conteneurs est essentiel pour maintenir la santé et les performances de vos applications conteneurisées. Il permet de détecter les problèmes avant qu'ils n'affectent les utilisateurs.

## Monitoring de base

**Scripts de monitoring** :
```bash
#!/bin/bash
# Monitoring de conteneurs Docker

# État des conteneurs
check_containers() {
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
}

# Utilisation des ressources
monitor_resources() {
    docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
}

# Logs récents
recent_logs() {
    local container="${1:-}"
    local lines="${2:-50}"
    
    if [ -z "$container" ]; then
        docker ps --format "{{.Names}}" | while read -r name; do
            echo "=== Logs de $name ==="
            docker logs --tail "$lines" "$name"
            echo
        done
    else
        docker logs --tail "$lines" "$container"
    fi
}
```

## Outils de monitoring

**Prometheus et Grafana** :
```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana

volumes:
  prometheus_data:
  grafana_data:
```

## Conclusion

Le monitoring de conteneurs garantit la santé et les performances de vos applications.

