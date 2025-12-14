# Chapitre 228 - Observabilité et monitoring

> "L'observabilité n'est pas seulement une question de métriques. C'est la capacité à comprendre pourquoi votre système se comporte d'une certaine façon, même face à des situations imprévues." - Charity Majors, co-fondatrice de Honeycomb

## Introduction : Au-delà du monitoring traditionnel

L'observabilité représente l'évolution du monitoring traditionnel vers une compréhension profonde du comportement des systèmes complexes. Alors que le monitoring se contente souvent de seuils et d'alertes, l'observabilité fournit les outils pour investiguer les problèmes complexes et comprendre les comportements émergents.

Dans ce chapitre, nous explorerons les trois piliers de l'observabilité (métriques, logs, traces), les outils modernes, et les patterns d'investigation.

## Section 1 : Les piliers de l'observabilité

### 1.1 Métriques (Metrics)

**Types de métriques :**
```bash
echo "=== TYPES DE MÉTRIQUES ==="
echo ""
echo "1. Métriques système:"
echo "   • CPU, mémoire, disque, réseau"
echo "   • Latence, throughput, erreurs"
echo "   • Utilisation des ressources"
echo ""
echo "2. Métriques applicatives:"
echo "   • Requêtes par seconde (RPS)"
echo "   • Temps de réponse (latency)"
echo "   • Taux d'erreur (error rate)"
echo "   • Utilisation du cache"
echo ""
echo "3. Métriques métier:"
echo "   • Conversions utilisateur"
echo "   • Revenus"
echo "   • Satisfaction client"
echo "   • Métriques SLA"
echo ""
echo "4. Types de collecte:"
echo "   • Counters: Valeurs cumulatives"
echo "   • Gauges: Valeurs instantanées"
echo "   • Histograms: Distributions"
echo "   • Summaries: Quantiles"
```

**Bonnes pratiques :**
- **RED Method** (Rate, Errors, Duration) pour les services
- **USE Method** (Utilization, Saturation, Errors) pour les ressources
- Métriques d'or : Latence, Trafic, Erreurs, Saturation
- Alertes basées sur des percentiles, pas des moyennes

### 1.2 Logs et traçabilité

**Structure des logs :**
```json
{
  "timestamp": "2023-11-10T14:30:45.123Z",
  "level": "INFO",
  "service": "web-api",
  "version": "1.2.3",
  "trace_id": "abc123def456",
  "span_id": "span789",
  "user_id": "user123",
  "request_id": "req456",
  "method": "GET",
  "path": "/api/users",
  "status_code": 200,
  "response_time_ms": 45,
  "user_agent": "Mozilla/5.0...",
  "ip_address": "192.168.1.100",
  "message": "User profile retrieved successfully",
  "metadata": {
    "db_query_time": 12,
    "cache_hit": true,
    "feature_flags": ["new_ui", "beta_feature"]
  }
}
```

**Patterns de logging :**
- **Structured Logging** : Format JSON avec champs cohérents
- **Correlation IDs** : Traçabilité des requêtes à travers les services
- **Log Levels** : DEBUG, INFO, WARN, ERROR, FATAL
- **Sampling** : Échantillonnage intelligent pour gérer le volume
- **Centralisation** : Agrégation dans Elasticsearch ou Loki

### 1.3 Traces distribuées

**Concepts de tracing :**
```bash
echo "=== TRACING DISTRIBUÉ ==="
echo ""
echo "1. Terminologie:"
echo "   • Trace: Parcours complet d'une requête"
echo "   • Span: Unité de travail dans une trace"
echo "   • Context: Métadonnées propagées"
echo ""
echo "2. Propagation:"
echo "   • Headers HTTP (X-Trace-Id, X-Span-Id)"
echo "   • Context propagation (OpenTelemetry)"
echo "   • Service mesh (Istio, Linkerd)"
echo ""
echo "3. Sampling:"
echo "   • Head-based: Décision en début de trace"
echo "   • Tail-based: Analyse post-mortem"
echo "   • Probabilistic: Échantillonnage aléatoire"
echo ""
echo "4. Intégration:"
echo "   • Instrumentation automatique"
echo "   • Instrumentation manuelle"
echo "   • Libraries (OpenTelemetry, Jaeger)"
```

## Section 2 : Stack d'observabilité moderne

### 2.1 Prometheus et Grafana

```yaml
# prometheus.yml - Configuration Prometheus
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
    
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
    
  - job_name: 'web-app'
    static_configs:
      - targets: ['web-app:8080']
    metrics_path: '/metrics'
    scrape_interval: 5s
    
  - job_name: 'database'
    static_configs:
      - targets: ['postgres-exporter:9187']
    
  - job_name: 'kubernetes'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - action: keep
        regex: __meta_kubernetes_pod_annotation_prometheus_io_scrape
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      - action: replace
        regex: (.+)
        source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        target_label: __metrics_path__
      - action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        target_label: __address__
```

```yaml
# alert_rules.yml - Règles d'alerte
groups:
  - name: example
    rules:
      - alert: HighRequestLatency
        expr: http_request_duration_seconds{quantile="0.5"} > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High request latency on {{ $labels.instance }}"
          description: "{{ $labels.instance }} has a median request latency above 0.5s for 10 minutes."
      
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "{{ $labels.instance }} has an error rate above 5% for 5 minutes."
      
      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "{{ $labels.instance }} has less than 10% disk space available."
      
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.job }} on {{ $labels.instance }} has been down for more than 1 minute."
```

```bash
# Démarrage de la stack Prometheus/Grafana
#!/bin/bash
# start_monitoring.sh

set -euo pipefail

# Créer le réseau
docker network create monitoring || true

# Démarrer Prometheus
docker run -d \
  --name prometheus \
  --network monitoring \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  -v prometheus_data:/prometheus \
  prom/prometheus

# Démarrer Grafana
docker run -d \
  --name grafana \
  --network monitoring \
  -p 3000:3000 \
  -e GF_SECURITY_ADMIN_PASSWORD=admin \
  -v grafana_data:/var/lib/grafana \
  grafana/grafana

# Démarrer Node Exporter
docker run -d \
  --name node-exporter \
  --network monitoring \
  -p 9100:9100 \
  --volume="/proc:/host/proc:ro" \
  --volume="/sys:/host/sys:ro" \
  --volume="/:/rootfs:ro" \
  prom/node-exporter

# Démarrer Alertmanager
docker run -d \
  --name alertmanager \
  --network monitoring \
  -p 9093:9093 \
  -v $(pwd)/alertmanager.yml:/etc/alertmanager/alertmanager.yml \
  prom/alertmanager

echo "Stack de monitoring démarrée:"
echo "• Prometheus: http://localhost:9090"
echo "• Grafana: http://localhost:3000 (admin/admin)"
echo "• Alertmanager: http://localhost:9093"
```

### 2.2 ELK Stack pour les logs

```yaml
# docker-compose.elk.yml - Stack ELK complète
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.5.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./logstash.yml:/usr/share/logstash/config/logstash.yml
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "9600:9600"
    environment:
      LS_JAVA_OPTS: "-Xmx256m -Xms256m"
    networks:
      - elk
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.5.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - elk
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.5.0
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml
      - /var/log:/var/log:ro
    networks:
      - elk
    depends_on:
      - logstash

volumes:
  elasticsearch_data:

networks:
  elk:
    driver: bridge
```

```conf
# logstash.conf - Configuration Logstash
input {
  beats {
    port => 5044
  }
  
  tcp {
    port => 5000
    codec => json
  }
  
  udp {
    port => 5000
    codec => json
  }
}

filter {
  # Parser les timestamps
  date {
    match => ["timestamp", "ISO8601", "YYYY-MM-dd HH:mm:ss,SSS", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
  }
  
  # Extraire des champs structurés
  json {
    source => "message"
  }
  
  # Géolocalisation des IPs
  geoip {
    source => "client_ip"
    target => "geoip"
  }
  
  # Anonymisation des données sensibles
  mutate {
    remove_field => ["password", "token", "secret"]
  }
  
  # Ajouter des tags
  mutate {
    add_tag => ["processed"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
  
  stdout {
    codec => rubydebug
  }
}
```

```yaml
# filebeat.yml - Configuration Filebeat
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
    - /var/log/syslog
    - /var/log/auth.log
  fields:
    service: system
    environment: production

- type: container
  enabled: true
  paths:
    - /var/lib/docker/containers/*/*.log
  processors:
    - add_docker_metadata:
        host: "unix:///var/run/docker.sock"
    - decode_json_fields:
        fields: ["message"]
        target: ""

output.logstash:
  hosts: ["logstash:5044"]
```

### 2.3 Jaeger pour le tracing

```yaml
# docker-compose.jaeger.yml - Stack Jaeger complète
version: '3.8'

services:
  jaeger-collector:
    image: jaegertracing/jaeger-collector:latest
    ports:
      - "14268:14268"
      - "14250:14250"
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
    networks:
      - jaeger
    depends_on:
      - elasticsearch

  jaeger-query:
    image: jaegertracing/jaeger-query:latest
    ports:
      - "16686:16686"
      - "16687:16687"
    environment:
      - SPAN_STORAGE_TYPE=elasticsearch
    networks:
      - jaeger
    depends_on:
      - elasticsearch

  jaeger-agent:
    image: jaegertracing/jaeger-agent:latest
    ports:
      - "6831:6831/udp"
      - "6832:6832/udp"
      - "5778:5778"
    environment:
      - REPORTER_GRPC_HOST_PORT=jaeger-collector:14250
    networks:
      - jaeger
    depends_on:
      - jaeger-collector

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - jaeger

volumes:
  elasticsearch_data:

networks:
  jaeger:
    driver: bridge
```

```javascript
// instrumentation.js - Instrumentation OpenTelemetry pour Node.js
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { SimpleSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { registerInstrumentations } = require('@opentelemetry/instrumentation');
const { HttpInstrumentation } = require('@opentelemetry/instrumentation-http');
const { ExpressInstrumentation } = require('@opentelemetry/instrumentation-express');

// Configuration de l'exporter Jaeger
const exporter = new JaegerExporter({
  endpoint: 'http://jaeger-collector:14268/api/traces',
});

// Configuration du provider de trace
const provider = new NodeTracerProvider();

// Configuration du processeur de span
provider.addSpanProcessor(new SimpleSpanProcessor(exporter));

// Enregistrement du provider
provider.register();

// Instrumentation automatique
registerInstrumentations({
  instrumentations: [
    new HttpInstrumentation(),
    new ExpressInstrumentation(),
  ],
});

// Instrumentation manuelle
const { trace } = require('@opentelemetry/api');

const tracer = trace.getTracer('my-app', '1.0.0');

app.get('/api/users/:id', (req, res) => {
  const span = tracer.startSpan('getUser', {
    attributes: {
      'user.id': req.params.id,
      'http.method': req.method,
      'http.url': req.url,
    },
  });

  // Simulation d'une opération
  setTimeout(() => {
    span.setAttribute('db.query.duration', 150);
    span.setAttribute('cache.hit', false);
    
    span.end();
    res.json({ id: req.params.id, name: 'John Doe' });
  }, 100);
});
```

## Section 3 : Instrumentation des applications

### 3.1 Métriques applicatives

```python
# metrics.py - Instrumentation Python avec Prometheus
from prometheus_client import Counter, Histogram, Gauge, Summary
import time

# Compteurs
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total number of HTTP requests',
    ['method', 'endpoint', 'status_code']
)

# Histogrammes
REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency in seconds',
    ['method', 'endpoint']
)

# Gauges
ACTIVE_CONNECTIONS = Gauge(
    'active_connections',
    'Number of active connections'
)

# Summaries
DB_QUERY_DURATION = Summary(
    'db_query_duration_seconds',
    'Database query duration in seconds',
    ['query_type']
)

def track_request(method, endpoint):
    """Décorateur pour tracker les requêtes HTTP"""
    def decorator(func):
        def wrapper(*args, **kwargs):
            start_time = time.time()
            
            try:
                result = func(*args, **kwargs)
                status_code = 200
                return result
            except Exception as e:
                status_code = 500
                raise e
            finally:
                REQUEST_COUNT.labels(
                    method=method,
                    endpoint=endpoint,
                    status_code=status_code
                ).inc()
                
                REQUEST_LATENCY.labels(
                    method=method,
                    endpoint=endpoint
                ).observe(time.time() - start_time)
        
        return wrapper
    return decorator

# Utilisation dans Flask
from flask import Flask, request

app = Flask(__name__)

@app.route('/api/users')
@track_request('GET', '/api/users')
def get_users():
    with DB_QUERY_DURATION.labels(query_type='select').time():
        # Simulation d'une requête DB
        time.sleep(0.1)
        
    return {'users': [{'id': 1, 'name': 'John'}]}

@app.route('/api/users', methods=['POST'])
@track_request('POST', '/api/users')
def create_user():
    with DB_QUERY_DURATION.labels(query_type='insert').time():
        # Simulation d'une insertion
        time.sleep(0.05)
        
    return {'id': 2, 'name': 'Jane'}, 201

# Exposition des métriques
@app.route('/metrics')
def metrics():
    from prometheus_client import generate_latest
    return generate_latest()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

```go
// metrics.go - Instrumentation Go
package main

import (
    "net/http"
    "time"
    "log"
    
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// Compteurs
var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status_code"},
    )
    
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
    
    activeConnections = promauto.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
)

func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Wrapper pour capturer le status code
        rw := &responseWriter{ResponseWriter: w, statusCode: 200}
        
        activeConnections.Inc()
        defer activeConnections.Dec()
        
        next.ServeHTTP(rw, r)
        
        duration := time.Since(start).Seconds()
        
        requestsTotal.WithLabelValues(
            r.Method,
            r.URL.Path,
            string(rune(rw.statusCode)),
        ).Inc()
        
        requestDuration.WithLabelValues(
            r.Method,
            r.URL.Path,
        ).Observe(duration)
    })
}

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func main() {
    mux := http.NewServeMux()
    
    // Route API avec middleware
    apiHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        time.Sleep(100 * time.Millisecond) // Simulation
        w.WriteHeader(200)
        w.Write([]byte(`{"message": "Hello World"}`))
    })
    
    mux.Handle("/api/", metricsMiddleware(apiHandler))
    
    // Exposition des métriques
    mux.Handle("/metrics", promhttp.Handler())
    
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", mux))
}
```

### 3.2 Logging structuré

```python
# logging_config.py - Configuration de logging structuré
import logging
import json
import sys
from datetime import datetime
from pythonjsonlogger import jsonlogger

class StructuredFormatter(jsonlogger.JsonFormatter):
    """Formatter JSON personnalisé"""
    
    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)
        
        # Ajouter des champs personnalisés
        log_record['timestamp'] = datetime.utcnow().isoformat() + 'Z'
        log_record['level'] = record.levelname
        log_record['service'] = 'my-app'
        log_record['version'] = '1.0.0'
        
        # Correlation ID depuis le contexte
        import flask
        try:
            log_record['correlation_id'] = flask.g.correlation_id
        except:
            log_record['correlation_id'] = 'unknown'

def setup_logging(level=logging.INFO):
    """Configuration du logging"""
    
    # Handler console
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(StructuredFormatter())
    
    # Handler fichier
    file_handler = logging.FileHandler('app.log')
    file_handler.setFormatter(StructuredFormatter())
    
    # Configuration du logger
    logging.basicConfig(
        level=level,
        handlers=[console_handler, file_handler]
    )
    
    # Logger spécifique pour les requêtes
    request_logger = logging.getLogger('request')
    request_logger.setLevel(logging.INFO)
    request_logger.addHandler(console_handler)

def get_logger(name):
    """Récupération d'un logger configuré"""
    return logging.getLogger(name)

# Middleware Flask pour logging des requêtes
def request_logging_middleware(app):
    @app.before_request
    def before_request():
        import flask
        import uuid
        
        # Générer correlation ID
        correlation_id = flask.request.headers.get('X-Correlation-ID', str(uuid.uuid4()))
        flask.g.correlation_id = correlation_id
        
        # Logger de requête
        logger = get_logger('request')
        logger.info('Request started', extra={
            'method': flask.request.method,
            'url': flask.request.url,
            'headers': dict(flask.request.headers),
            'correlation_id': correlation_id
        })
    
    @app.after_request
    def after_request(response):
        import flask
        
        logger = get_logger('request')
        logger.info('Request completed', extra={
            'status_code': response.status_code,
            'response_time': getattr(flask.g, 'response_time', 0),
            'correlation_id': flask.g.correlation_id
        })
        
        return response

# Utilisation
setup_logging()

logger = get_logger(__name__)

# Logging avec contexte
logger.info('User login successful', extra={
    'user_id': 123,
    'ip_address': '192.168.1.100',
    'user_agent': 'Mozilla/5.0...'
})

# Logging d'erreur avec traceback
try:
    1 / 0
except Exception as e:
    logger.error('Division by zero', exc_info=True, extra={
        'operation': 'calculate_average',
        'input_value': 0
    })
```

### 3.3 Tracing distribué

```python
# tracing.py - Configuration du tracing OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
import flask

def setup_tracing(service_name, jaeger_host='localhost', jaeger_port=14268):
    """Configuration du tracing"""
    
    # Configuration de l'exporter Jaeger
    jaeger_exporter = JaegerExporter(
        agent_host_name=jaeger_host,
        agent_port=jaeger_port,
    )
    
    # Configuration du provider de trace
    trace.set_tracer_provider(TracerProvider())
    tracer_provider = trace.get_tracer_provider()
    
    # Ajout du processeur de span
    tracer_provider.add_span_processor(
        BatchSpanProcessor(jaeger_exporter)
    )
    
    # Instrumentation automatique
    FlaskInstrumentor().instrument()
    RequestsInstrumentor().instrument()
    
    return trace.get_tracer(service_name)

# Utilisation dans une app Flask
app = flask.Flask(__name__)
tracer = setup_tracing('web-api')

@app.route('/api/users/<int:user_id>')
def get_user(user_id):
    with tracer.start_as_span('get_user') as span:
        span.set_attribute('user.id', user_id)
        span.set_attribute('operation', 'fetch_user_profile')
        
        # Simulation d'appel DB
        with tracer.start_as_span('db_query') as db_span:
            db_span.set_attribute('db.statement', 'SELECT * FROM users WHERE id = ?')
            db_span.set_attribute('db.table', 'users')
            
            import time
            time.sleep(0.05)  # Simulation
            
            db_span.set_attribute('db.rows_affected', 1)
        
        # Simulation d'appel API externe
        with tracer.start_as_span('external_api_call') as api_span:
            api_span.set_attribute('http.method', 'GET')
            api_span.set_attribute('http.url', 'https://api.example.com/profile')
            
            import requests
            response = requests.get('https://httpbin.org/delay/0.1')
            
            api_span.set_attribute('http.status_code', response.status_code)
        
        span.set_attribute('result.success', True)
        
        return {
            'id': user_id,
            'name': 'John Doe',
            'email': 'john@example.com'
        }

@app.route('/api/users', methods=['POST'])
def create_user():
    with tracer.start_as_span('create_user') as span:
        data = flask.request.json
        
        span.set_attribute('user.name', data.get('name'))
        span.set_attribute('validation.required', True)
        
        # Validation
        with tracer.start_as_span('validate_input') as validation_span:
            if not data.get('name'):
                validation_span.set_attribute('validation.error', 'name_required')
                span.set_attribute('result.success', False)
                flask.abort(400)
            
            validation_span.set_attribute('validation.passed', True)
        
        # Insertion DB
        with tracer.start_as_span('db_insert') as db_span:
            db_span.set_attribute('db.statement', 'INSERT INTO users (...)')
            db_span.set_attribute('db.table', 'users')
            
            import time
            time.sleep(0.03)
            
            user_id = 123  # Simulation
            db_span.set_attribute('db.inserted_id', user_id)
        
        span.set_attribute('result.success', True)
        span.set_attribute('user.created_id', user_id)
        
        return {'id': user_id}, 201

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

## Section 4 : Alerting et réponse aux incidents

### 4.1 Stratégies d'alerting

```yaml
# alertmanager.yml - Configuration Alertmanager
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'password'

route:
  group_by: ['alertname', 'service', 'environment']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'team-email'
  
  routes:
    - match:
        severity: critical
      receiver: 'critical-pager'
      continue: true
      
    - match:
        service: database
      receiver: 'db-team'
      
    - match:
        environment: production
      receiver: 'prod-team'

receivers:
  - name: 'team-email'
    email_configs:
      - to: 'team@example.com'
        subject: '{{ template "email.subject" . }}'
        body: '{{ template "email.body" . }}'
        
  - name: 'critical-pager'
    pagerduty_configs:
      - service_key: 'your-pagerduty-key'
      
  - name: 'db-team'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/.../.../...'
        channel: '#db-alerts'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        
  - name: 'prod-team'
    webhook_configs:
      - url: 'http://incident-management.example.com/webhook'
        http_config:
          basic_auth:
            username: 'alertmanager'
            password: 'secret'

templates:
  - 'templates/*.tmpl'
```

```html
<!-- templates/email.tmpl -->
{{ define "email.subject" }}
[{{ .GroupLabels.alertname }}] {{ .GroupLabels.service }} - {{ .Status | title }}
{{ end }}

{{ define "email.body" }}
{{ range .Alerts }}
Alert: {{ .Annotations.summary }}
Description: {{ .Annotations.description }}
Labels:
{{ range $key, $value := .Labels }}
  {{ $key }}: {{ $value }}
{{ end }}
{{ end }}
{{ end }}
```

```json
<!-- templates/slack.tmpl -->
{{ define "slack.title" }}
{{ .GroupLabels.alertname }} - {{ .Status | title }}
{{ end }}

{{ define "slack.text" }}
{{ range .Alerts }}
*{{ .Annotations.summary }}*
{{ .Annotations.description }}

*Labels:*
{{ range $key, $value := .Labels }}
• {{ $key }}: {{ $value }}
{{ end }}
{{ end }}
{{ end }}
```

### 4.2 Runbooks et automatisation

```yaml
# runbook.yml - Runbook automatisé
runbook:
  name: "Database Connection Pool Exhausted"
  description: "Le pool de connexions de la base de données est épuisé"
  
  symptoms:
    - "Erreurs de connexion DB"
    - "Timeouts des requêtes"
    - "Dégradation des performances"
  
  investigation:
    - name: "Vérifier l'état du pool"
      command: "kubectl exec -it postgres-0 -- psql -c 'SELECT * FROM pg_stat_activity;'"
      
    - name: "Vérifier les métriques"
      query: "rate(postgres_connections_active[5m])"
      
    - name: "Vérifier les logs d'application"
      query: "SELECT * FROM logs WHERE service='web-api' AND message LIKE '%connection pool%' ORDER BY timestamp DESC LIMIT 10"
  
  remediation:
    - name: "Redémarrer les pods applicatifs"
      action: "kubectl rollout restart deployment/web-api"
      condition: "postgres_connections_active > 80% of max_connections"
      
    - name: "Augmenter le pool de connexions"
      action: "kubectl patch configmap app-config -p '{\"data\":{\"db.max_connections\":\"100\"}}'"
      condition: "manual_approval_required"
      
    - name: "Scale horizontal"
      action: "kubectl scale deployment web-api --replicas=10"
      condition: "cpu_usage > 70% AND memory_usage > 70%"
  
  escalation:
    - time: "5m"
      action: "notify_oncall_engineer"
      
    - time: "15m" 
      action: "notify_team_lead"
      
    - time: "30m"
      action: "notify_management"
```

```python
# incident_response.py - Automatisation de la réponse aux incidents
import requests
import time
import logging
from datetime import datetime

class IncidentResponder:
    def __init__(self, prometheus_url, alertmanager_url):
        self.prometheus_url = prometheus_url
        self.alertmanager_url = alertmanager_url
        self.logger = logging.getLogger(__name__)
    
    def get_active_alerts(self):
        """Récupère les alertes actives"""
        response = requests.get(f"{self.alertmanager_url}/api/v1/alerts")
        return response.json().get('data', [])
    
    def query_prometheus(self, query):
        """Exécute une requête Prometheus"""
        response = requests.get(
            f"{self.prometheus_url}/api/v1/query",
            params={'query': query}
        )
        return response.json()
    
    def execute_runbook(self, alert_name):
        """Exécute le runbook pour une alerte donnée"""
        runbooks = {
            'HighErrorRate': self.handle_high_error_rate,
            'DatabaseDown': self.handle_database_down,
            'ServiceUnhealthy': self.handle_service_unhealthy,
        }
        
        if alert_name in runbooks:
            self.logger.info(f"Exécution du runbook pour {alert_name}")
            return runbooks[alert_name]()
        else:
            self.logger.warning(f"Pas de runbook pour {alert_name}")
            return False
    
    def handle_high_error_rate(self):
        """Gestion des taux d'erreur élevés"""
        # Vérifier les métriques actuelles
        query = "rate(http_requests_total{status=~'5..'}[5m]) / rate(http_requests_total[5m])"
        result = self.query_prometheus(query)
        
        if result['data']['result']:
            error_rate = float(result['data']['result'][0]['value'][1])
            
            if error_rate > 0.1:  # 10%
                self.logger.info("Taux d'erreur critique détecté, redémarrage du service")
                self.restart_service('web-api')
                return True
        
        return False
    
    def handle_database_down(self):
        """Gestion des pannes de base de données"""
        # Vérifier la connectivité
        if not self.check_database_connectivity():
            self.logger.info("Base de données indisponible, basculement")
            self.failover_database()
            return True
        
        return False
    
    def handle_service_unhealthy(self):
        """Gestion des services défaillants"""
        # Scale automatique
        cpu_usage = self.get_service_cpu_usage('web-api')
        
        if cpu_usage > 80:
            self.logger.info("CPU élevé détecté, scaling horizontal")
            self.scale_service('web-api', 5)
            return True
        
        return False
    
    def restart_service(self, service_name):
        """Redémarre un service"""
        import subprocess
        subprocess.run([
            'kubectl', 'rollout', 'restart', 
            f'deployment/{service_name}'
        ], check=True)
    
    def scale_service(self, service_name, replicas):
        """Scale un service"""
        import subprocess
        subprocess.run([
            'kubectl', 'scale', 'deployment', service_name,
            f'--replicas={replicas}'
        ], check=True)
    
    def check_database_connectivity(self):
        """Vérifie la connectivité DB"""
        # Simulation
        return True
    
    def failover_database(self):
        """Bascule vers une base secondaire"""
        # Simulation
        pass
    
    def get_service_cpu_usage(self, service_name):
        """Récupère l'utilisation CPU d'un service"""
        query = f"rate(container_cpu_usage_seconds_total{{pod=~'{service_name}-.*'}}[5m])"
        result = self.query_prometheus(query)
        
        if result['data']['result']:
            return float(result['data']['result'][0]['value'][1]) * 100
        
        return 0
    
    def run_incident_response(self):
        """Boucle principale de réponse aux incidents"""
        self.logger.info("Démarrage de la réponse automatique aux incidents")
        
        while True:
            try:
                alerts = self.get_active_alerts()
                
                for alert in alerts:
                    alert_name = alert['labels'].get('alertname')
                    
                    if alert_name and self.execute_runbook(alert_name):
                        self.logger.info(f"Runbook exécuté pour {alert_name}")
                
                time.sleep(30)  # Vérification toutes les 30 secondes
                
            except Exception as e:
                self.logger.error(f"Erreur dans la boucle de réponse: {e}")
                time.sleep(60)

if __name__ == '__main__':
    responder = IncidentResponder(
        prometheus_url='http://prometheus:9090',
        alertmanager_url='http://alertmanager:9093'
    )
    
    responder.run_incident_response()
```

## Section 5 : Patterns d'investigation et débogage

### 5.1 Analyse des métriques

```bash
#!/bin/bash
# analyze_metrics.sh - Analyse automatique des métriques

set -euo pipefail

PROMETHEUS_URL="${PROMETHEUS_URL:-http://prometheus:9090}"
SERVICE="${1:-web-api}"
TIME_RANGE="${2:-1h}"

# Fonction de requête Prometheus
query_prometheus() {
    local query="$1"
    curl -s -G "$PROMETHEUS_URL/api/v1/query" --data-urlencode "query=$query"
}

# Fonction d'analyse des latences
analyze_latency() {
    echo "=== ANALYSE DES LATENCES ==="
    
    # Latence moyenne
    local avg_latency
    avg_latency=$(query_prometheus "rate(http_request_duration_seconds_sum{service=\"$SERVICE\"}[$TIME_RANGE]) / rate(http_request_duration_seconds_count{service=\"$SERVICE\"}[$TIME_RANGE])" | jq -r '.data.result[0].value[1]')
    
    echo "Latence moyenne: ${avg_latency}s"
    
    # Percentiles
    for p in 50 90 95 99; do
        local p_value
        p_value=$(query_prometheus "histogram_quantile(0.$p, rate(http_request_duration_seconds_bucket{service=\"$SERVICE\"}[$TIME_RANGE]))" | jq -r '.data.result[0].value[1]')
        echo "P$p: ${p_value}s"
    done
    
    # Détection d'anomalies
    if (( $(echo "$avg_latency > 1.0" | bc -l) )); then
        echo "⚠️  Latence élevée détectée"
    fi
}

# Fonction d'analyse du taux d'erreur
analyze_error_rate() {
    echo "=== ANALYSE DU TAUX D'ERREUR ==="
    
    local error_rate
    error_rate=$(query_prometheus "rate(http_requests_total{service=\"$SERVICE\",status=~\"5..\"}[$TIME_RANGE]) / rate(http_requests_total{service=\"$SERVICE\"}[$TIME_RANGE])" | jq -r '.data.result[0].value[1]')
    
    echo "Taux d'erreur: $(printf "%.2f%%" $(echo "$error_rate * 100" | bc -l))"
    
    if (( $(echo "$error_rate > 0.05" | bc -l) )); then
        echo "⚠️  Taux d'erreur élevé détecté"
    fi
}

# Fonction d'analyse du trafic
analyze_traffic() {
    echo "=== ANALYSE DU TRAFIC ==="
    
    local rps
    rps=$(query_prometheus "rate(http_requests_total{service=\"$SERVICE\"}[$TIME_RANGE])" | jq -r '.data.result[0].value[1]')
    
    echo "Requêtes/seconde: ${rps}"
    
    # Analyse par endpoint
    echo "Top endpoints:"
    query_prometheus "topk(5, rate(http_requests_total{service=\"$SERVICE\"}[$TIME_RANGE]))" | jq -r '.data.result[] | "\(.metric.endpoint): \(.value[1]) RPS"'
}

# Fonction d'analyse des ressources
analyze_resources() {
    echo "=== ANALYSE DES RESSOURCES ==="
    
    # CPU
    local cpu_usage
    cpu_usage=$(query_prometheus "rate(container_cpu_usage_seconds_total{pod=~\"$SERVICE-.*\"}[$TIME_RANGE])" | jq -r '.data.result[0].value[1]')
    
    echo "Utilisation CPU: $(printf "%.1f%%" $(echo "$cpu_usage * 100" | bc -l))"
    
    # Mémoire
    local mem_usage
    mem_usage=$(query_prometheus "container_memory_usage_bytes{pod=~\"$SERVICE-.*\"} / container_spec_memory_limit_bytes" | jq -r '.data.result[0].value[1]')
    
    echo "Utilisation mémoire: $(printf "%.1f%%" $(echo "$mem_usage * 100" | bc -l))"
    
    # Alerte si ressources élevées
    if (( $(echo "$cpu_usage > 0.8" | bc -l) )) || (( $(echo "$mem_usage > 0.8" | bc -l) )); then
        echo "⚠️  Ressources élevées détectées"
    fi
}

# Fonction de corrélation d'événements
analyze_correlation() {
    echo "=== CORRÉLATION D'ÉVÉNEMENTS ==="
    
    # Chercher des patterns dans les métriques
    local high_latency
    high_latency=$(query_prometheus "http_request_duration_seconds{quantile=\"0.95\",service=\"$SERVICE\"} > 2" | jq -r '.data.result | length')
    
    local high_errors
    high_errors=$(query_prometheus "rate(http_requests_total{service=\"$SERVICE\",status=~\"5..\"}[5m]) > 10" | jq -r '.data.result | length')
    
    local high_cpu
    high_cpu=$(query_prometheus "rate(container_cpu_usage_seconds_total{pod=~\"$SERVICE-.*\"}[5m]) > 0.8" | jq -r '.data.result | length')
    
    if [[ "$high_latency" -gt 0 ]] && [[ "$high_errors" -gt 0 ]]; then
        echo "🔍 Pattern détecté: Latence élevée + erreurs élevées"
        echo "   → Possible surcharge du service"
    fi
    
    if [[ "$high_cpu" -gt 0 ]] && [[ "$high_latency" -gt 0 ]]; then
        echo "🔍 Pattern détecté: CPU élevé + latence élevée"
        echo "   → Possible bottleneck CPU"
    fi
}

# Fonction de génération de rapport
generate_report() {
    echo "=== RAPPORT D'ANALYSE ==="
    echo "Service: $SERVICE"
    echo "Période: $TIME_RANGE"
    echo "Généré le: $(date)"
    echo ""
    
    analyze_latency
    echo ""
    
    analyze_error_rate
    echo ""
    
    analyze_traffic
    echo ""
    
    analyze_resources
    echo ""
    
    analyze_correlation
}

# Exécution
generate_report
```

### 5.2 Investigation avec les logs

```bash
#!/bin/bash
# log_investigation.sh - Investigation avec les logs

set -euo pipefail

ELASTICSEARCH_URL="${ELASTICSEARCH_URL:-http://elasticsearch:9200}"
SERVICE="${1:-web-api}"
TIME_RANGE="${2:-1h}"

# Fonction de recherche dans Elasticsearch
search_logs() {
    local query="$1"
    local size="${2:-100}"
    
    curl -s -X GET "$ELASTICSEARCH_URL/logs-*/_search" \
         -H 'Content-Type: application/json' \
         -d "{
           \"size\": $size,
           \"query\": {
             \"bool\": {
               \"must\": [
                 { \"match\": { \"service\": \"$SERVICE\" } },
                 { \"range\": { \"@timestamp\": { \"gte\": \"now-${TIME_RANGE}\" } } }
               ],
               \"filter\": $query
             }
           },
           \"sort\": [ { \"@timestamp\": { \"order\": \"desc\" } } ]
         }"
}

# Fonction d'analyse des erreurs
analyze_errors() {
    echo "=== ANALYSE DES ERREURS ==="
    
    # Erreurs récentes
    local error_count
    error_count=$(search_logs "{ \"match\": { \"level\": \"ERROR\" } }" 1000 | jq '.hits.total.value')
    
    echo "Nombre d'erreurs récentes: $error_count"
    
    # Top erreurs
    echo "Top types d'erreurs:"
    search_logs "{ \"match\": { \"level\": \"ERROR\" } }" 1000 | \
    jq -r '.hits.hits[]._source | "\(.error_type // "unknown"): \(.message | .[0:100])"' | \
    sort | uniq -c | sort -nr | head -5
}

# Fonction d'analyse des performances
analyze_performance() {
    echo "=== ANALYSE DES PERFORMANCES ==="
    
    # Requêtes lentes
    echo "Requêtes lentes (>1s):"
    search_logs "{ \"range\": { \"response_time\": { \"gte\": 1000 } } }" 50 | \
    jq -r '.hits.hits[]._source | "\(.method) \(.path): \(.response_time)ms"'
}

# Fonction de traçage d'utilisateur
trace_user() {
    local user_id="$1"
    
    echo "=== TRACE UTILISATEUR: $user_id ==="
    
    # Toutes les actions d'un utilisateur
    search_logs "{ \"match\": { \"user_id\": \"$user_id\" } }" 100 | \
    jq -r '.hits.hits[]._source | "\(.@timestamp): \(.method) \(.path) - \(.status_code)"'
}

# Fonction de corrélation avec les métriques
correlate_logs_metrics() {
    echo "=== CORRÉLATION LOGS/MÉTRIQUES ==="
    
    # Pics d'erreur
    local error_spikes
    error_spikes=$(search_logs "{ \"bool\": { \"must\": [ { \"match\": { \"level\": \"ERROR\" } }, { \"range\": { \"@timestamp\": { \"gte\": \"now-10m\" } } } ] } }" 1000 | jq '.hits.hits | length')
    
    if [[ "$error_spikes" -gt 50 ]]; then
        echo "⚠️  Pic d'erreurs détecté (dernières 10min: $error_spikes erreurs)"
        
        # Analyser les erreurs récurrentes
        search_logs "{ \"match\": { \"level\": \"ERROR\" } }" 100 | \
        jq -r '.hits.hits[]._source.message' | \
        sort | uniq -c | sort -nr | head -3
    fi
}

# Fonction de recherche par correlation ID
search_correlation_id() {
    local correlation_id="$1"
    
    echo "=== RECHERCHE CORRELATION ID: $correlation_id ==="
    
    search_logs "{ \"match\": { \"correlation_id\": \"$correlation_id\" } }" 200 | \
    jq -r '.hits.hits[]._source | "\(.@timestamp) [\(.service)] \(.level): \(.message)"'
}

# Fonction principale
main() {
    echo "=== INVESTIGATION LOGS ==="
    echo "Service: $SERVICE"
    echo "Période: $TIME_RANGE"
    echo ""
    
    analyze_errors
    echo ""
    
    analyze_performance
    echo ""
    
    correlate_logs_metrics
    echo ""
    
    # Recherche interactive si correlation ID fourni
    if [[ $# -ge 3 ]]; then
        search_correlation_id "$3"
    fi
}

# Interface utilisateur
case "${1:-analyze}" in
    "analyze")
        main "$@"
        ;;
    "trace")
        if [[ $# -lt 3 ]]; then
            echo "Usage: $0 trace <service> <user_id>"
            exit 1
        fi
        trace_user "$3"
        ;;
    "correlation")
        if [[ $# -lt 3 ]]; then
            echo "Usage: $0 correlation <service> <correlation_id>"
            exit 1
        fi
        search_correlation_id "$3"
        ;;
    *)
        echo "Usage: $0 [analyze|trace|correlation] <service> [time_range|user_id|correlation_id]"
        exit 1
        ;;
esac
```

## Conclusion : L'observabilité comme culture

L'observabilité transforme le débogage réactif en compréhension proactive des systèmes. En combinant métriques, logs et traces, les équipes peuvent identifier les problèmes avant qu'ils n'impactent les utilisateurs, optimiser les performances, et améliorer continuellement la fiabilité.

Dans le prochain chapitre, nous explorerons les architectures de microservices et les patterns de conception distribuée.

---

**Exercice pratique :** Créez une stack d'observabilité complète qui inclut :
1. Collecte de métriques avec Prometheus et instrumentation applicative
2. Centralisation des logs avec ELK stack
3. Tracing distribué avec Jaeger
4. Alerting intelligent avec Alertmanager
5. Dashboards en temps réel avec Grafana
6. Automatisation de la réponse aux incidents

**Challenge avancé :** Développez une plateforme d'observabilité multi-cloud qui :
- Agrège des métriques depuis AWS, Azure et GCP
- Corréle automatiquement les logs, métriques et traces
- Détecte les anomalies en temps réel avec du machine learning
- Fournit des insights prédictifs sur les pannes
- S'intègre avec les outils de gestion d'incidents (PagerDuty, OpsGenie)

**Réflexion :** Comment l'observabilité change-t-elle fondamentalement l'approche des opérations et du développement logiciel ? Quels sont les impacts sur la culture d'équipe et la qualité des produits ?

