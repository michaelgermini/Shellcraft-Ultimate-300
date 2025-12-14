# Chapitre 49 - Alertes et notifications automatisées

## Table des matières
- [Introduction](#introduction)
- [Architecture des systèmes d'alertes](#architecture-des-systèmes-dalertes)
- [Systèmes de notification](#systèmes-de-notification)
- [Notifications par email](#notifications-par-email)
- [Notifications Slack et Discord](#notifications-slack-et-discord)
- [Notifications Telegram](#notifications-telegram)
- [Notifications SMS et téléphone](#notifications-sms-et-téléphone)
- [Webhooks et API](#webhooks-et-api)
- [Configuration des alertes](#configuration-des-alertes)
- [Seuils et conditions](#seuils-et-conditions)
- [Logique d'escalade](#logique-descalade)
- [Déduplication et regroupement](#déduplication-et-regroupement)
- [Intégration avec les outils de monitoring](#intégration-avec-les-outils-de-monitoring)
- [Nagios et Zabbix](#nagios-et-zabbix)
- [Prometheus Alertmanager](#prometheus-alertmanager)
- [Sécurité et fiabilité](#sécurité-et-fiabilité)
- [Bonnes pratiques](#bonnes-pratiques)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Les alertes et notifications automatisées constituent le système nerveux de l'administration proactive. Au lieu d'attendre que les utilisateurs signalent les problèmes, le système détecte les anomalies et alerte automatiquement les équipes compétentes.

Imaginez les alertes automatisées comme un système d'alarme sophistiqué dans une maison intelligente : elles détectent les intrusions (erreurs système), les fuites (problèmes de performance), les pannes (défaillances matérielles), et alertent les propriétaires ou les services d'urgence selon la gravité de la situation. Un système bien conçu évite la fatigue d'alerte en ne notifiant que lorsque c'est vraiment nécessaire, avec des informations contextuelles utiles pour une résolution rapide.

## Architecture des systèmes d'alertes

### Composants essentiels

**Architecture** :
```bash
#!/bin/bash
# Architecture d'un système d'alertes

# 1. Collecte de métriques
#    - Monitoring continu
#    - Collecte de données système

# 2. Évaluation des conditions
#    - Comparaison avec seuils
#    - Détection d'anomalies

# 3. Génération d'alertes
#    - Création d'événements d'alerte
#    - Enrichissement avec contexte

# 4. Routage et escalade
#    - Sélection du canal approprié
#    - Escalade selon gravité

# 5. Notification
#    - Envoi via canaux multiples
#    - Confirmation de réception

# 6. Résolution et suivi
#    - Tracking de l'alerte
#    - Historique et rapports
```

## Systèmes de notification

### Types de notifications

**Canaux disponibles** :
```bash
#!/bin/bash
# Types de notifications

# 1. Email
#    - Universel et fiable
#    - Support HTML et pièces jointes
#    - Peut être filtré/spam

# 2. Slack/Discord
#    - Temps réel
#    - Intégration avec workflows
#    - Historique consultable

# 3. SMS/Téléphone
#    - Urgence critique
#    - Coût associé
#    - Fiabilité élevée

# 4. Webhooks
#    - Intégration personnalisée
#    - Flexible
#    - Nécessite développement

# 5. Push notifications
#    - Applications mobiles
#    - Temps réel
#    - Nécessite infrastructure
```

## Notifications par email

### Configuration de base

**Envoi d'email simple** :
```bash
#!/bin/bash
# Envoi d'email simple

send_email() {
    local to="$1"
    local subject="$2"
    local body="$3"
    
    echo "$body" | mail -s "$subject" "$to"
}

# Utilisation
send_email "admin@example.com" "Alerte système" "Le serveur nécessite une attention"
```

**Configuration mail** :
```bash
#!/bin/bash
# Configuration du système de mail

# Installation
sudo apt install mailutils postfix

# Configuration Postfix
# /etc/postfix/main.cf
configure_postfix() {
    sudo tee -a /etc/postfix/main.cf << 'EOF'
# Configuration de base
myhostname = server.example.com
mydomain = example.com
myorigin = $mydomain

# Réseaux autorisés
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128

# Relais (si nécessaire)
relayhost = [smtp.example.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = encrypt
EOF
    
    sudo systemctl restart postfix
}
```

### Email formaté

**Email HTML** :
```bash
#!/bin/bash
# Email formaté HTML

send_formatted_email() {
    local to="$1"
    local subject="$2"
    local title="$3"
    local content="$4"
    local priority="${5:-normal}"  # low, normal, high, urgent
    
    local color
    case "$priority" in
        urgent) color="#dc3545" ;;
        high) color="#fd7e14" ;;
        normal) color="#0d6efd" ;;
        low) color="#6c757d" ;;
    esac
    
    local html_body=$(cat <<EOF
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { font-family: Arial, sans-serif; line-height: 1.6; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; }
        .header { background-color: $color; color: white; padding: 20px; border-radius: 5px 5px 0 0; }
        .content { background-color: #f8f9fa; padding: 20px; border-radius: 0 0 5px 5px; }
        .footer { text-align: center; margin-top: 20px; color: #6c757d; font-size: 12px; }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>$title</h2>
        </div>
        <div class="content">
            <p>$content</p>
            <p><strong>Serveur:</strong> $(hostname)</p>
            <p><strong>Date:</strong> $(date '+%Y-%m-%d %H:%M:%S')</p>
        </div>
        <div class="footer">
            <p>Notification automatique - Ne pas répondre</p>
        </div>
    </div>
</body>
</html>
EOF
)
    
    echo "$html_body" | mail -a "Content-Type: text/html" -s "$subject" "$to"
}

# Utilisation
send_formatted_email "admin@example.com" \
    "[CRITICAL] Disque plein" \
    "Alerte Critique" \
    "Le disque /var est plein à 95%" \
    "urgent"
```

### Email avec pièces jointes

**Email avec attachements** :
```bash
#!/bin/bash
# Email avec pièces jointes

send_email_with_attachment() {
    local to="$1"
    local subject="$2"
    local body="$3"
    local attachment="$4"
    
    (
        echo "To: $to"
        echo "Subject: $subject"
        echo "MIME-Version: 1.0"
        echo "Content-Type: multipart/mixed; boundary=boundary123"
        echo ""
        echo "--boundary123"
        echo "Content-Type: text/plain"
        echo ""
        echo "$body"
        echo ""
        echo "--boundary123"
        echo "Content-Type: application/octet-stream"
        echo "Content-Disposition: attachment; filename=$(basename "$attachment")"
        echo ""
        cat "$attachment"
        echo ""
        echo "--boundary123--"
    ) | sendmail "$to"
}
```

## Notifications Slack et Discord

### Intégration Slack

**Webhook Slack** :
```bash
#!/bin/bash
# Notifications Slack

SLACK_WEBHOOK_URL="${SLACK_WEBHOOK_URL:-}"

send_slack_message() {
    local channel="${1:-#alerts}"
    local message="$2"
    local color="${3:-good}"  # good, warning, danger
    local title="${4:-Notification}"
    
    if [ -z "$SLACK_WEBHOOK_URL" ]; then
        echo "Erreur: SLACK_WEBHOOK_URL non définie" >&2
        return 1
    fi
    
    local payload=$(cat <<EOF
{
    "channel": "$channel",
    "username": "System Monitor",
    "icon_emoji": ":computer:",
    "attachments": [{
        "color": "$color",
        "title": "$title",
        "text": "$message",
        "fields": [
            {
                "title": "Serveur",
                "value": "$(hostname)",
                "short": true
            },
            {
                "title": "Date",
                "value": "$(date '+%Y-%m-%d %H:%M:%S')",
                "short": true
            }
        ],
        "footer": "Shellcraft Ultimate 300",
        "ts": $(date +%s)
    }]
}
EOF
)
    
    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" \
        "$SLACK_WEBHOOK_URL" \
        --silent --show-error --fail
    
    return $?
}

# Utilisation
send_slack_message "#alerts" "Le serveur nécessite une attention immédiate" "danger" "Alerte Critique"
```

**Slack avec boutons d'action** :
```bash
#!/bin/bash
# Slack avec actions

send_slack_with_actions() {
    local message="$1"
    local alert_id="$2"
    
    local payload=$(cat <<EOF
{
    "text": "$message",
    "attachments": [{
        "color": "danger",
        "actions": [
            {
                "name": "acknowledge",
                "text": "Acquitter",
                "type": "button",
                "value": "$alert_id",
                "style": "primary"
            },
            {
                "name": "resolve",
                "text": "Résolu",
                "type": "button",
                "value": "$alert_id",
                "style": "danger"
            }
        ]
    }]
}
EOF
)
    
    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" \
        "$SLACK_WEBHOOK_URL"
}
```

### Intégration Discord

**Webhook Discord** :
```bash
#!/bin/bash
# Notifications Discord

DISCORD_WEBHOOK_URL="${DISCORD_WEBHOOK_URL:-}"

send_discord_message() {
    local message="$1"
    local color="${2:-16711680}"  # Rouge par défaut
    local title="${3:-Alerte système}"
    
    if [ -z "$DISCORD_WEBHOOK_URL" ]; then
        echo "Erreur: DISCORD_WEBHOOK_URL non définie" >&2
        return 1
    fi
    
    local payload=$(cat <<EOF
{
    "embeds": [{
        "title": "$title",
        "description": "$message",
        "color": $color,
        "fields": [
            {
                "name": "Serveur",
                "value": "$(hostname)",
                "inline": true
            },
            {
                "name": "Date",
                "value": "$(date '+%Y-%m-%d %H:%M:%S')",
                "inline": true
            }
        ],
        "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
    }]
}
EOF
)
    
    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" \
        "$DISCORD_WEBHOOK_URL" \
        --silent --show-error --fail
    
    return $?
}
```

## Notifications Telegram

### Bot Telegram

**Configuration bot Telegram** :
```bash
#!/bin/bash
# Notifications Telegram

TELEGRAM_BOT_TOKEN="${TELEGRAM_BOT_TOKEN:-}"
TELEGRAM_CHAT_ID="${TELEGRAM_CHAT_ID:-}"

send_telegram_message() {
    local message="$1"
    local parse_mode="${2:-HTML}"  # HTML ou Markdown
    
    if [ -z "$TELEGRAM_BOT_TOKEN" ] || [ -z "$TELEGRAM_CHAT_ID" ]; then
        echo "Erreur: Variables Telegram non configurées" >&2
        return 1
    fi
    
    local url="https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage"
    
    local payload=$(cat <<EOF
{
    "chat_id": "$TELEGRAM_CHAT_ID",
    "text": "$message",
    "parse_mode": "$parse_mode"
}
EOF
)
    
    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" \
        "$url" \
        --silent --show-error --fail
    
    return $?
}

# Message formaté
send_formatted_telegram() {
    local title="$1"
    local message="$2"
    local level="${3:-INFO}"  # INFO, WARNING, ERROR, CRITICAL
    
    local emoji
    case "$level" in
        CRITICAL) emoji="🚨" ;;
        ERROR) emoji="❌" ;;
        WARNING) emoji="⚠️" ;;
        *) emoji="ℹ️" ;;
    esac
    
    local formatted_message=$(cat <<EOF
<b>$emoji $title</b>

$message

<code>Serveur:</code> $(hostname)
<code>Date:</code> $(date '+%Y-%m-%d %H:%M:%S')
EOF
)
    
    send_telegram_message "$formatted_message" "HTML"
}
```

## Notifications SMS et téléphone

### SMS via API

**SMS avec Twilio** :
```bash
#!/bin/bash
# SMS avec Twilio

TWILIO_ACCOUNT_SID="${TWILIO_ACCOUNT_SID:-}"
TWILIO_AUTH_TOKEN="${TWILIO_AUTH_TOKEN:-}"
TWILIO_FROM="${TWILIO_FROM:-}"

send_sms_twilio() {
    local to="$1"
    local message="$2"
    
    if [ -z "$TWILIO_ACCOUNT_SID" ] || [ -z "$TWILIO_AUTH_TOKEN" ]; then
        echo "Erreur: Configuration Twilio manquante" >&2
        return 1
    fi
    
    local url="https://api.twilio.com/2010-04-01/Accounts/${TWILIO_ACCOUNT_SID}/Messages.json"
    
    curl -X POST "$url" \
        -u "${TWILIO_ACCOUNT_SID}:${TWILIO_AUTH_TOKEN}" \
        --data-urlencode "From=${TWILIO_FROM}" \
        --data-urlencode "To=${to}" \
        --data-urlencode "Body=${message}" \
        --silent --show-error --fail
    
    return $?
}
```

**Appel téléphonique** :
```bash
#!/bin/bash
# Appel téléphonique avec Twilio

make_phone_call() {
    local to="$1"
    local message="$2"
    
    local url="https://api.twilio.com/2010-04-01/Accounts/${TWILIO_ACCOUNT_SID}/Calls.json"
    
    curl -X POST "$url" \
        -u "${TWILIO_ACCOUNT_SID}:${TWILIO_AUTH_TOKEN}" \
        --data-urlencode "From=${TWILIO_FROM}" \
        --data-urlencode "To=${to}" \
        --data-urlencode "Url=http://example.com/twiml/alert.xml" \
        --silent --show-error --fail
}
```

## Webhooks et API

### Webhooks génériques

**Webhook personnalisé** :
```bash
#!/bin/bash
# Webhook générique

send_webhook() {
    local url="$1"
    local payload="$2"
    local method="${3:-POST}"
    
    curl -X "$method" \
        -H 'Content-Type: application/json' \
        --data "$payload" \
        "$url" \
        --silent --show-error --fail
    
    return $?
}

# Webhook avec authentification
send_authenticated_webhook() {
    local url="$1"
    local payload="$2"
    local auth_token="$3"
    
    curl -X POST \
        -H 'Content-Type: application/json' \
        -H "Authorization: Bearer $auth_token" \
        --data "$payload" \
        "$url" \
        --silent --show-error --fail
}
```

## Configuration des alertes

### Système d'alertes modulaire

**Framework d'alertes** :
```bash
#!/bin/bash
# Framework d'alertes modulaire

ALERT_LOG="/var/log/alerts.log"
ALERT_CONFIG="/etc/alerting.conf"

# Charger la configuration
load_alert_config() {
    if [ -f "$ALERT_CONFIG" ]; then
        source "$ALERT_CONFIG"
    else
        # Configuration par défaut
        ALERT_EMAIL="${ALERT_EMAIL:-admin@example.com}"
        ALERT_SLACK_ENABLED="${ALERT_SLACK_ENABLED:-false}"
        ALERT_TELEGRAM_ENABLED="${ALERT_TELEGRAM_ENABLED:-false}"
    fi
}

# Envoyer une alerte
send_alert() {
    local level="$1"  # INFO, WARNING, ERROR, CRITICAL
    local message="$2"
    local details="${3:-}"
    
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    local hostname=$(hostname)
    
    # Logger l'alerte
    echo "[$timestamp] [$level] [$hostname] $message" >> "$ALERT_LOG"
    
    # Envoyer selon le niveau
    case "$level" in
        CRITICAL)
            send_critical_alert "$message" "$details"
            ;;
        ERROR)
            send_error_alert "$message" "$details"
            ;;
        WARNING)
            send_warning_alert "$message" "$details"
            ;;
        INFO)
            send_info_alert "$message" "$details"
            ;;
    esac
}

send_critical_alert() {
    local message="$1"
    local details="$2"
    
    # Email
    send_formatted_email "$ALERT_EMAIL" \
        "[CRITICAL] $(hostname) - $(date '+%Y-%m-%d %H:%M:%S')" \
        "Alerte Critique" \
        "$message\n\n$details" \
        "urgent"
    
    # Slack
    if [ "$ALERT_SLACK_ENABLED" = "true" ]; then
        send_slack_message "#critical-alerts" "$message\n\n$details" "danger" "Alerte Critique"
    fi
    
    # Telegram
    if [ "$ALERT_TELEGRAM_ENABLED" = "true" ]; then
        send_formatted_telegram "Alerte Critique" "$message\n\n$details" "CRITICAL"
    fi
    
    # SMS pour les alertes critiques
    if [ -n "$ALERT_SMS_NUMBER" ]; then
        send_sms_twilio "$ALERT_SMS_NUMBER" "CRITICAL: $message"
    fi
}
```

## Seuils et conditions

### Détection de seuils

**Système de seuils** :
```bash
#!/bin/bash
# Système de seuils

check_threshold() {
    local metric="$1"
    local value="$2"
    local warning_threshold="$3"
    local critical_threshold="$4"
    local unit="${5:-%}"
    
    if (( $(echo "$value >= $critical_threshold" | bc -l) )); then
        send_alert "CRITICAL" \
            "Seuil critique dépassé: $metric = ${value}${unit}" \
            "Seuil critique: ${critical_threshold}${unit}"
        return 2
    elif (( $(echo "$value >= $warning_threshold" | bc -l) )); then
        send_alert "WARNING" \
            "Seuil d'avertissement dépassé: $metric = ${value}${unit}" \
            "Seuil d'avertissement: ${warning_threshold}${unit}"
        return 1
    fi
    
    return 0
}

# Utilisation
check_threshold "Utilisation disque /var" 95 80 90 "%"
check_threshold "Utilisation CPU" 85 70 80 "%"
check_threshold "Utilisation mémoire" 90 75 85 "%"
```

## Logique d'escalade

### Escalade automatique

**Système d'escalade** :
```bash
#!/bin/bash
# Système d'escalade

ESCALATION_LOG="/var/log/escalation.log"

escalate_alert() {
    local alert_id="$1"
    local level="$2"
    local message="$3"
    local current_escalation="${4:-0}"
    
    # Niveaux d'escalade
    # 0: Notification initiale
    # 1: Escalade niveau 1 (après 5 min)
    # 2: Escalade niveau 2 (après 15 min)
    # 3: Escalade niveau 3 (après 30 min)
    
    case "$current_escalation" in
        0)
            send_alert "$level" "$message"
            echo "$(date '+%Y-%m-%d %H:%M:%S') $alert_id ESCALATION_0" >> "$ESCALATION_LOG"
            ;;
        1)
            send_alert "$level" "$message [ESCALATION 1]"
            # Notifier le manager
            send_email "$MANAGER_EMAIL" "[ESCALATION] $message" "$message"
            echo "$(date '+%Y-%m-%d %H:%M:%S') $alert_id ESCALATION_1" >> "$ESCALATION_LOG"
            ;;
        2)
            send_alert "$level" "$message [ESCALATION 2]"
            # Appel téléphonique
            make_phone_call "$ON_CALL_NUMBER" "$message"
            echo "$(date '+%Y-%m-%d %H:%M:%S') $alert_id ESCALATION_2" >> "$ESCALATION_LOG"
            ;;
        3)
            send_alert "$level" "$message [ESCALATION 3 - URGENT]"
            # Toutes les notifications
            send_sms_twilio "$ON_CALL_NUMBER" "URGENT: $message"
            make_phone_call "$MANAGER_NUMBER" "$message"
            echo "$(date '+%Y-%m-%d %H:%M:%S') $alert_id ESCALATION_3" >> "$ESCALATION_LOG"
            ;;
    esac
}
```

## Déduplication et regroupement

### Déduplication d'alertes

**Système de déduplication** :
```bash
#!/bin/bash
# Déduplication d'alertes

ALERT_DEDUP_DB="/var/lib/alerting/dedup.db"

deduplicate_alert() {
    local alert_key="$1"
    local message="$2"
    local ttl="${3:-300}"  # 5 minutes par défaut
    
    # Créer la clé de déduplication
    local hash=$(echo -n "$alert_key" | sha256sum | cut -d' ' -f1)
    local timestamp=$(date +%s)
    
    # Vérifier si l'alerte existe déjà
    if [ -f "$ALERT_DEDUP_DB" ]; then
        local existing=$(grep "^$hash:" "$ALERT_DEDUP_DB" | cut -d: -f2)
        if [ -n "$existing" ]; then
            local age=$((timestamp - existing))
            if [ $age -lt $ttl ]; then
                echo "Alerte dupliquée ignorée (âge: ${age}s)"
                return 1
            fi
        fi
    fi
    
    # Enregistrer la nouvelle alerte
    mkdir -p "$(dirname "$ALERT_DEDUP_DB")"
    echo "$hash:$timestamp" >> "$ALERT_DEDUP_DB"
    
    # Nettoyer les anciennes entrées
    while IFS=: read -r h t; do
        local age=$((timestamp - t))
        if [ $age -gt $ttl ]; then
            sed -i "/^$h:/d" "$ALERT_DEDUP_DB"
        fi
    done < "$ALERT_DEDUP_DB"
    
    return 0
}
```

## Intégration avec les outils de monitoring

### Nagios

**Intégration Nagios** :
```bash
#!/bin/bash
# Intégration Nagios

send_nagios_notification() {
    local hostname="$1"
    local service="$2"
    local state="$3"  # OK, WARNING, CRITICAL, UNKNOWN
    local output="$4"
    
    # Utiliser send_nsca pour envoyer à Nagios
    echo -e "$hostname\t$service\t$state\t$output" | \
        send_nsca -H nagios-server -c /etc/send_nsca.cfg
}
```

### Zabbix

**Intégration Zabbix** :
```bash
#!/bin/bash
# Intégration Zabbix

send_zabbix_alert() {
    local hostname="$1"
    local key="$2"
    local value="$3"
    
    zabbix_sender -z zabbix-server -s "$hostname" -k "$key" -o "$value"
}
```

## Prometheus Alertmanager

### Configuration Alertmanager

**Configuration Alertmanager** :
```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/...'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'default'
  routes:
    - match:
        severity: critical
      receiver: 'critical'
      continue: true
    - match:
        severity: warning
      receiver: 'warning'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@example.com'
  
  - name: 'critical'
    slack_configs:
      - channel: '#critical-alerts'
    pagerduty_configs:
      - service_key: 'your-key'
  
  - name: 'warning'
    slack_configs:
      - channel: '#warnings'
```

## Sécurité et fiabilité

### Sécurisation des webhooks

**Sécurité** :
```bash
#!/bin/bash
# Sécurisation des webhooks

secure_webhook() {
    local url="$1"
    local payload="$2"
    local secret="$3"
    
    # Créer une signature HMAC
    local signature=$(echo -n "$payload" | openssl dgst -sha256 -hmac "$secret" | cut -d' ' -f2)
    
    curl -X POST \
        -H 'Content-Type: application/json' \
        -H "X-Signature: $signature" \
        --data "$payload" \
        "$url"
}
```

## Bonnes pratiques

### Recommandations

**Bonnes pratiques** :
```bash
#!/bin/bash
# Bonnes pratiques pour les alertes

# 1. Ne pas alerter pour tout
#    - Seuils appropriés
#    - Déduplication

# 2. Contextualiser les alertes
#    - Informations utiles
#    - Liens vers documentation

# 3. Escalade intelligente
#    - Pas d'escalade immédiate
#    - Délais appropriés

# 4. Canaux multiples
#    - Email pour historique
#    - Slack pour temps réel
#    - SMS pour urgence

# 5. Monitoring des alertes
#    - Taux d'alertes
#    - Temps de résolution
#    - Faux positifs
```

## Scripts d'automatisation

### Gestionnaire d'alertes complet

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire d'alertes complet

set -euo pipefail

ALERT_SYSTEM_HOME="/opt/alerting"
ALERT_CONFIG="${ALERT_SYSTEM_HOME}/config.conf"
ALERT_LOG="${ALERT_SYSTEM_HOME}/alerts.log"

# Charger la configuration
load_config() {
    if [ -f "$ALERT_CONFIG" ]; then
        source "$ALERT_CONFIG"
    fi
}

# Fonction principale
main() {
    local action="$1"
    shift
    
    load_config
    
    case "$action" in
        send)
            send_alert "$@"
            ;;
        check)
            check_and_alert "$@"
            ;;
        escalate)
            escalate_alert "$@"
            ;;
        *)
            echo "Usage: $0 {send|check|escalate}"
            exit 1
            ;;
    esac
}

# Utilisation
# main send CRITICAL "Message d'alerte"
# main check disk /var 80 90
# main escalate alert-123 CRITICAL "Message"
```

## Conclusion

Les alertes et notifications automatisées sont essentielles pour maintenir la visibilité sur l'état des systèmes. En utilisant plusieurs canaux (email, Slack, Telegram, SMS), en implémentant une logique d'escalade intelligente, et en intégrant avec les outils de monitoring existants, vous pouvez créer un système d'alertes robuste qui informe les bonnes personnes au bon moment.

Un système bien conçu évite la fatigue d'alerte en ne notifiant que lorsque c'est vraiment nécessaire, avec des informations contextuelles utiles pour une résolution rapide. La compréhension approfondie de ces mécanismes permet de créer des systèmes d'alertes qui s'adaptent aux besoins spécifiques de chaque environnement.

Dans le chapitre suivant, nous explorerons les expressions cron avancées, découvrant comment créer des planifications complexes pour l'automatisation des tâches.
