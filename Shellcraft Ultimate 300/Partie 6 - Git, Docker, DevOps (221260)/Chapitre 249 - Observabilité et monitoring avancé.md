# Chapitre 249 - Observabilité et monitoring avancé

## Table des matières
- [Introduction](#introduction)
- [Stack de monitoring](#stack-de-monitoring)
- [Logging centralisé](#logging-centralisé)
- [Tracing distribué](#tracing-distribué)
- [Conclusion](#conclusion)

## Introduction

L'observabilité combine métriques, logs, et traces pour donner une vue complète de votre système.

## Stack de monitoring

**Prometheus + Grafana** :
```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
  
  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

## Logging centralisé

**ELK Stack** :
```yaml
# docker-compose.logging.yml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.0.0
    environment:
      - discovery.type=single-node
  
  logstash:
    image: logstash:8.0.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
  
  kibana:
    image: kibana:8.0.0
    ports:
      - "5601:5601"
```

## Tracing distribué

**Jaeger** :
```yaml
# docker-compose.tracing.yml
version: '3.8'

services:
  jaeger:
    image: jaegertracing/all-in-one
    ports:
      - "16686:16686"
      - "14268:14268"
```

## Conclusion

L'observabilité complète nécessite métriques, logs, et traces pour comprendre complètement le comportement du système.

