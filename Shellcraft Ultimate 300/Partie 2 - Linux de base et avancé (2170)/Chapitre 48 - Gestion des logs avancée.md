# Chapitre 48 - Gestion des logs avancée

## Table des matières
- [Introduction](#introduction)
- [Architecture de logging moderne](#architecture-de-logging-moderne)
- [rsyslog : configuration avancée](#rsyslog--configuration-avancée)
- [journald : système de logs systemd](#journald--système-de-logs-systemd)
- [Logging structuré](#logging-structuré)
- [Outils de gestion des logs](#outils-de-gestion-des-logs)
- [Centralisation des logs](#centralisation-des-logs)
- [ELK Stack : Elasticsearch, Logstash, Kibana](#elk-stack--elasticsearch-logstash-kibana)
- [Loki et Promtail](#loki-et-promtail)
- [Analyse et traitement](#analyse-et-traitement)
- [Parsing et extraction de données](#parsing-et-extraction-de-données)
- [Corrélation et agrégation](#corrélation-et-agrégation)
- [Monitoring et alertes basées sur les logs](#monitoring-et-alertes-basées-sur-les-logs)
- [Sécurité et conformité](#sécurité-et-conformité)
- [Chiffrement et intégrité](#chiffrement-et-intégrité)
- [Automation et intelligence](#automation-et-intelligence)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La gestion avancée des logs constitue l'œil et l'oreille du système d'information. Au-delà de la simple collecte, il s'agit d'analyser, corréler, et agir sur les informations contenues dans les logs pour maintenir la santé et la sécurité du système.

Imaginez les logs comme les archives d'un grand navire : ils enregistrent chaque événement, chaque décision, chaque anomalie. Une bonne gestion des logs permet de retracer les problèmes passés, d'anticiper les futurs, et de naviguer en sécurité même dans les tempêtes les plus violentes. Dans un environnement moderne, cette gestion s'étend de la centralisation multi-serveur à l'analyse en temps réel avec l'intelligence artificielle.

## Architecture de logging moderne

### Principes de design

**Architecture moderne** :
```bash
#!/bin/bash
# Architecture de logging moderne

# 1. Collecte distribuée
# - Agents sur chaque serveur
# - Collecte locale puis envoi centralisé

# 2. Transport sécurisé
# - Chiffrement TLS
# - Authentification

# 3. Stockage scalable
# - Indexation pour recherche rapide
# - Compression pour économie d'espace

# 4. Analyse en temps réel
# - Streaming des logs
# - Détection d'anomalies

# 5. Visualisation
# - Dashboards interactifs
# - Rapports automatisés
```

**Flux de données** :
```bash
#!/bin/bash
# Flux de données dans un système de logging moderne

# Application → Agent de collecte → Transport → Stockage → Analyse → Visualisation
#     ↓              ↓                ↓          ↓          ↓           ↓
#  Logs locaux   rsyslog/        TLS/SSH    Elasticsearch  Logstash   Kibana
#                journald                    /Loki          /Grafana
```

## rsyslog : configuration avancée

### Modules et filtres avancés

**Configuration rsyslog avancée** :
```bash
#!/bin/bash
# Configuration rsyslog avancée

# /etc/rsyslog.conf

# Charger les modules nécessaires
module(load="imuxsock")    # Socket Unix
module(load="imjournal")   # journald
module(load="imklog")      # Messages kernel
module(load="imudp")       # UDP
module(load="imtcp")       # TCP
module(load="ommysql")     # MySQL output
module(load="omelasticsearch")  # Elasticsearch output
module(load="omfile")      # File output
module(load="mmjsonparse") # JSON parsing
module(load="mmnormalize") # Normalisation

# Configuration UDP/TCP
input(type="imudp" port="514")
input(type="imtcp" port="514")

# Règles de filtrage avancées
# Router les logs selon des critères complexes
if $programname == 'sshd' and $msg contains 'Failed' then {
    action(type="omfile" file="/var/log/ssh-failures.log")
    stop
}

# Parsing JSON
if $msg contains '"level":"error"' then {
    action(type="omfile" file="/var/log/errors.log")
}

# Normalisation
template(name="normalized" type="string" string="%timestamp% %hostname% %programname%: %msg%\n")
```

**Templates personnalisés** :
```bash
#!/bin/bash
# Templates rsyslog personnalisés

# Template JSON
template(name="json-template" type="string" string='{"timestamp":"%timegenerated:::date-rfc3339%","host":"%hostname%","program":"%programname%","pid":"%procid%","message":"%msg:::json%"}')

# Template pour base de données
template(name="db-template" type="string" string="insert into logs (timestamp, host, program, message) values ('%timegenerated:::date-mysql%', '%hostname%', '%programname%', '%msg:::mysql-escape%')")

# Utilisation des templates
*.* action(type="omfile" file="/var/log/all.log" template="json-template")
```

## journald : système de logs systemd

### Configuration journald

**Configuration journald** :
```bash
#!/bin/bash
# Configuration journald

# Fichier : /etc/systemd/journald.conf

configure_journald() {
    sudo tee -a /etc/systemd/journald.conf << 'EOF'
[Journal]
# Stockage persistant
Storage=persistent

# Taille maximale des fichiers
SystemMaxUse=1G
SystemKeepFree=2G
SystemMaxFileSize=100M

# Rotation
MaxRetentionSec=1month
MaxFileSec=1day

# Compression
Compress=yes

# Sécurité
Seal=yes
SplitMode=uid
EOF
    
    sudo systemctl restart systemd-journald
}
```

**Commandes journald avancées** :
```bash
#!/bin/bash
# Commandes journald avancées

# Requêtes complexes
journalctl _SYSTEMD_UNIT=apache2.service _PID=1234
journalctl _UID=1000
journalctl _COMM=sshd

# Requêtes combinées
journalctl _SYSTEMD_UNIT=apache2.service + _PID=1234

# Requêtes par boot
journalctl -b                    # Boot actuel
journalctl -b -1                 # Boot précédent
journalctl --list-boots          # Liste des boots

# Export et import
journalctl --since "2024-01-01" --until "2024-01-31" > logs_january.txt
journalctl --since "2024-01-01" --until "2024-01-31" -o json > logs_january.json

# Rotation manuelle
sudo journalctl --vacuum-time=30d
sudo journalctl --vacuum-size=500M
```

## Logging structuré

### Format JSON

**Logging structuré JSON** :
```bash
#!/bin/bash
# Logging structuré JSON

log_json() {
    local level="$1"
    local message="$2"
    local extra_fields="${3:-{}}"
    
    cat << EOF | jq -c .
{
    "timestamp": "$(date -Iseconds)",
    "hostname": "$(hostname)",
    "level": "$level",
    "message": "$message",
    "pid": $$,
    "script": "${0##*/}",
    "extra": $extra_fields
}
EOF
}

# Utilisation
log_json "INFO" "Opération réussie" '{"user":"alice","action":"login"}'
log_json "ERROR" "Échec de connexion" '{"host":"db.example.com","port":3306}'
```

**Intégration avec rsyslog** :
```bash
#!/bin/bash
# Intégration logging JSON avec rsyslog

# Configuration rsyslog pour parser JSON
# /etc/rsyslog.d/01-json.conf
cat > /etc/rsyslog.d/01-json.conf << 'EOF'
module(load="mmjsonparse")

# Parser les messages JSON
action(type="mmjsonparse")

# Router selon le champ level
if $!level == "error" then {
    action(type="omfile" file="/var/log/errors.log")
}
EOF
```

## Outils de gestion des logs

### logrotate : rotation avancée

**Configuration logrotate avancée** :
```bash
#!/bin/bash
# Configuration logrotate avancée

# /etc/logrotate.d/myapp
configure_logrotate() {
    sudo tee /etc/logrotate.d/myapp << 'EOF'
/var/log/myapp/*.log {
    daily
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 appuser appgroup
    sharedscripts
    postrotate
        systemctl reload myapp
    endscript
    prerotate
        # Script avant rotation
        /usr/local/bin/pre-rotate.sh
    endscript
}
EOF
}
```

**Rotation avec compression** :
```bash
#!/bin/bash
# Rotation avec compression

# Configuration pour logs volumineux
cat > /etc/logrotate.d/large-logs << 'EOF'
/var/log/large/*.log {
    size 100M
    rotate 10
    compress
    compresscmd /usr/bin/pigz
    compressext .gz
    create 0640 user group
    dateext
    dateformat -%Y%m%d
}
EOF
```

### multitail : visualisation multiple

**Utilisation multitail** :
```bash
#!/bin/bash
# Utilisation multitail

# Installation
sudo apt install multitail

# Visualiser plusieurs fichiers
multitail /var/log/syslog /var/log/auth.log

# Avec coloration
multitail -c /var/log/apache2/access.log -c /var/log/apache2/error.log

# Configuration personnalisée
multitail -s 2 -cT ansi /var/log/syslog
```

## Centralisation des logs

### Configuration serveur rsyslog

**Serveur de centralisation** :
```bash
#!/bin/bash
# Configuration serveur rsyslog

configure_rsyslog_server() {
    sudo tee /etc/rsyslog.d/99-central-server.conf << 'EOF'
# Accepter les logs distants
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")

# Séparer les logs par hôte
template(name="RemoteHost" type="string" string="/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log")

# Appliquer le template aux logs distants
if $fromhost-ip != '127.0.0.1' then {
    action(type="omfile" dynaFile="RemoteHost")
    stop
}
EOF
    
    sudo systemctl restart rsyslog
}
```

**Configuration client** :
```bash
#!/bin/bash
# Configuration client rsyslog

configure_rsyslog_client() {
    sudo tee /etc/rsyslog.d/99-forward.conf << 'EOF'
# Envoyer tous les logs au serveur central
*.* @logserver.example.com:514

# Avec TCP pour fiabilité
*.* @@logserver.example.com:514

# Avec TLS (sécurisé)
*.* @@(o)logserver.example.com:6514
EOF
    
    sudo systemctl restart rsyslog
}
```

## ELK Stack : Elasticsearch, Logstash, Kibana

### Installation ELK

**Installation ELK Stack** :
```bash
#!/bin/bash
# Installation ELK Stack

# Elasticsearch
install_elasticsearch() {
    wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
    echo "deb https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
    sudo apt update
    sudo apt install elasticsearch
    sudo systemctl enable elasticsearch
    sudo systemctl start elasticsearch
}

# Logstash
install_logstash() {
    sudo apt install logstash
    sudo systemctl enable logstash
}

# Kibana
install_kibana() {
    sudo apt install kibana
    sudo systemctl enable kibana
    sudo systemctl start kibana
}
```

### Configuration Logstash

**Pipeline Logstash** :
```bash
#!/bin/bash
# Configuration Logstash

configure_logstash() {
    sudo tee /etc/logstash/conf.d/01-syslog.conf << 'EOF'
input {
  syslog {
    port => 514
    type => "syslog"
  }
  
  file {
    path => "/var/log/apache2/access.log"
    type => "apache_access"
    start_position => "beginning"
  }
  
  file {
    path => "/var/log/apache2/error.log"
    type => "apache_error"
    start_position => "beginning"
  }
}

filter {
  if [type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGLINE}" }
    }
  }
  
  if [type] == "apache_access" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
  }
  
  date {
    match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
  }
}
EOF
    
    sudo systemctl restart logstash
}
```

## Loki et Promtail

### Installation Loki

**Stack Loki** :
```bash
#!/bin/bash
# Installation Loki

# Loki (serveur)
install_loki() {
    wget https://github.com/grafana/loki/releases/download/v2.9.0/loki-linux-amd64.zip
    unzip loki-linux-amd64.zip
    sudo mv loki-linux-amd64 /usr/local/bin/loki
    sudo chmod +x /usr/local/bin/loki
}

# Promtail (agent)
install_promtail() {
    wget https://github.com/grafana/loki/releases/download/v2.9.0/promtail-linux-amd64.zip
    unzip promtail-linux-amd64.zip
    sudo mv promtail-linux-amd64 /usr/local/bin/promtail
    sudo chmod +x /usr/local/bin/promtail
}
```

**Configuration Promtail** :
```bash
#!/bin/bash
# Configuration Promtail

configure_promtail() {
    sudo tee /etc/promtail/config.yml << 'EOF'
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki-server:3100/loki/api/v1/push

scrape_configs:
  - job_name: syslog
    static_configs:
      - targets:
          - localhost
        labels:
          job: syslog
          __path__: /var/log/syslog

  - job_name: apache
    static_configs:
      - targets:
          - localhost
        labels:
          job: apache
          __path__: /var/log/apache2/*.log
EOF
    
    sudo systemctl enable promtail
    sudo systemctl start promtail
}
```

## Analyse et traitement

### Parsing avec awk et sed

**Analyse de logs avec awk** :
```bash
#!/bin/bash
# Analyse de logs avec awk

# Statistiques par IP
analyze_ip_stats() {
    awk '{print $1}' /var/log/apache2/access.log | \
    sort | uniq -c | sort -rn | head -10
}

# Requêtes par heure
analyze_hourly_requests() {
    awk '{print $4}' /var/log/apache2/access.log | \
    cut -d: -f2 | sort | uniq -c
}

# Codes de statut HTTP
analyze_status_codes() {
    awk '{print $9}' /var/log/apache2/access.log | \
    sort | uniq -c | sort -rn
}

# Top pages visitées
analyze_top_pages() {
    awk '{print $7}' /var/log/apache2/access.log | \
    sort | uniq -c | sort -rn | head -20
}
```

### Analyse avec jq (JSON)

**Analyse de logs JSON** :
```bash
#!/bin/bash
# Analyse de logs JSON

# Filtrer les erreurs
filter_errors() {
    jq 'select(.level == "error")' logs.json
}

# Grouper par niveau
group_by_level() {
    jq 'group_by(.level) | map({level: .[0].level, count: length})' logs.json
}

# Statistiques temporelles
time_stats() {
    jq '[.[] | .timestamp] | group_by(.[:13]) | map({time: .[0], count: length})' logs.json
}
```

## Parsing et extraction de données

### Grok patterns

**Patterns Grok** :
```bash
#!/bin/bash
# Patterns Grok pour Logstash

# Pattern syslog
%{SYSLOGTIMESTAMP:timestamp} %{IPORHOST:host} %{PROG:program}(?:\[%{POSINT:pid}\])?: %{GREEDYDATA:message}

# Pattern Apache
%{IPORHOST:clientip} %{USER:ident} %{USER:auth} \[%{HTTPDATE:timestamp}\] "(?:%{WORD:verb} %{NOTSPACE:request}(?: HTTP/%{NUMBER:httpversion})?|%{DATA:rawrequest})" %{NUMBER:response} (?:%{NUMBER:bytes}|-)

# Pattern personnalisé
%{TIMESTAMP_ISO8601:timestamp} \[%{LOGLEVEL:level}\] %{GREEDYDATA:message}
```

## Corrélation et agrégation

### Scripts de corrélation

**Corrélation d'événements** :
```bash
#!/bin/bash
# Corrélation d'événements

correlate_events() {
    local log_file="$1"
    local time_window="${2:-300}"  # 5 minutes
    
    # Grouper les événements par fenêtre temporelle
    awk -v window="$time_window" '
    {
        # Extraire timestamp et convertir en secondes
        ts = mktime($1 " " $2 " " $3 " " $4 " " $5 " " $6)
        window_start = int(ts / window) * window
        
        events[window_start]++
    }
    END {
        for (w in events) {
            print w, events[w]
        }
    }' "$log_file"
}
```

## Monitoring et alertes basées sur les logs

### Détection d'anomalies

**Détection automatique** :
```bash
#!/bin/bash
# Détection d'anomalies dans les logs

detect_anomalies() {
    local log_file="$1"
    local threshold="${2:-10}"
    
    # Détecter les pics d'erreurs
    local error_count=$(grep -c "ERROR\|CRITICAL\|FATAL" "$log_file" 2>/dev/null || echo 0)
    
    if [ "$error_count" -gt "$threshold" ]; then
        echo "ALERTE: $error_count erreurs détectées"
        return 1
    fi
    
    return 0
}

# Monitoring continu
monitor_logs_continuously() {
    local log_file="$1"
    
    tail -f "$log_file" | while read line; do
        if echo "$line" | grep -qE "ERROR|CRITICAL|FATAL"; then
            echo "ALERTE: $line"
            # Envoyer notification
        fi
    done
}
```

## Sécurité et conformité

### Chiffrement des logs

**Chiffrement des logs** :
```bash
#!/bin/bash
# Chiffrement des logs

encrypt_logs() {
    local log_file="$1"
    local encrypted_file="${log_file}.enc"
    
    # Chiffrer avec GPG
    gpg --symmetric --cipher-algo AES256 --output "$encrypted_file" "$log_file"
    
    # Supprimer le fichier original
    rm -f "$log_file"
    
    echo "Log chiffré : $encrypted_file"
}

# Déchiffrer
decrypt_logs() {
    local encrypted_file="$1"
    local output_file="${encrypted_file%.enc}"
    
    gpg --decrypt --output "$output_file" "$encrypted_file"
    
    echo "Log déchiffré : $output_file"
}
```

### Intégrité des logs

**Vérification d'intégrité** :
```bash
#!/bin/bash
# Vérification d'intégrité des logs

verify_log_integrity() {
    local log_file="$1"
    local checksum_file="${log_file}.sha256"
    
    # Vérifier le checksum
    if sha256sum -c "$checksum_file" 2>/dev/null; then
        echo "✓ Intégrité vérifiée"
        return 0
    else
        echo "✗ Intégrité compromise"
        return 1
    fi
}

# Créer le checksum
create_log_checksum() {
    local log_file="$1"
    sha256sum "$log_file" > "${log_file}.sha256"
}
```

## Automation et intelligence

### Scripts d'analyse automatique

**Analyseur automatique** :
```bash
#!/bin/bash
# Analyseur automatique de logs

auto_log_analyzer() {
    local log_dir="${1:-/var/log}"
    local report_file="${2:-/tmp/log_analysis_$(date +%Y%m%d).txt}"
    
    {
        echo "=== Analyse automatique des logs ==="
        echo "Date: $(date)"
        echo ""
        
        echo "=== Erreurs récentes ==="
        find "$log_dir" -name "*.log" -type f -exec grep -h "ERROR\|CRITICAL\|FATAL" {} \; | tail -20
        
        echo ""
        echo "=== Top IPs ==="
        find "$log_dir" -name "access.log" -type f -exec awk '{print $1}' {} \; | sort | uniq -c | sort -rn | head -10
        
        echo ""
        echo "=== Statistiques par service ==="
        journalctl --since "24 hours ago" --no-pager | \
        awk '{print $5}' | cut -d'[' -f1 | sort | uniq -c | sort -rn | head -10
        
    } > "$report_file"
    
    echo "Rapport généré : $report_file"
}
```

## Scripts d'automatisation

### Gestionnaire de logs complet

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de logs complet

set -euo pipefail

log_manager() {
    local action="$1"
    shift
    
    case "$action" in
        analyze)
            analyze_logs "$@"
            ;;
        rotate)
            rotate_logs "$@"
            ;;
        centralize)
            centralize_logs "$@"
            ;;
        monitor)
            monitor_logs "$@"
            ;;
        *)
            echo "Usage: log_manager {analyze|rotate|centralize|monitor}"
            exit 1
            ;;
    esac
}

analyze_logs() {
    local log_file="$1"
    
    echo "=== Analyse de $log_file ==="
    
    # Statistiques de base
    echo "Lignes totales: $(wc -l < "$log_file")"
    echo "Erreurs: $(grep -c "ERROR\|CRITICAL" "$log_file" || echo 0)"
    echo "Avertissements: $(grep -c "WARNING\|WARN" "$log_file" || echo 0)"
}

rotate_logs() {
    sudo logrotate -f /etc/logrotate.conf
    echo "Rotation des logs effectuée"
}

centralize_logs() {
    local servers=("$@")
    
    for server in "${servers[@]}"; do
        echo "Centralisation depuis $server..."
        ssh "$server" "sudo journalctl --since '1 hour ago' --no-pager" >> /var/log/central/all.log
    done
}

monitor_logs() {
    local log_file="$1"
    
    tail -f "$log_file" | while read line; do
        if echo "$line" | grep -qE "ERROR|CRITICAL"; then
            echo "[ALERTE] $line"
        fi
    done
}

# Utilisation
# log_manager analyze /var/log/syslog
# log_manager rotate
# log_manager centralize server1 server2 server3
# log_manager monitor /var/log/app.log
```

## Conclusion

La gestion avancée des logs est essentielle pour maintenir la visibilité et la sécurité des systèmes modernes. En utilisant des outils comme rsyslog, journald, ELK Stack, et Loki, vous pouvez créer des systèmes de logging robustes qui collectent, analysent, et alertent sur les événements critiques.

Un système bien configuré utilise plusieurs couches : collecte locale efficace, transport sécurisé vers un serveur central, stockage scalable avec indexation, et analyse en temps réel pour la détection proactive des problèmes. La compréhension approfondie de ces outils permet de créer des architectures de logging qui s'adaptent aux besoins spécifiques de chaque environnement.

Dans le chapitre suivant, nous explorerons les alertes et notifications automatisées, découvrant comment créer des systèmes qui alertent automatiquement les équipes lorsque des problèmes sont détectés.
