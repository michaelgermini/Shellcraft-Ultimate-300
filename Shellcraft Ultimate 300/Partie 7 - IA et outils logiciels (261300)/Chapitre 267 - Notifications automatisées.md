# Chapitre 267 - Notifications automatisées

## Table des matières
- [Introduction](#introduction)
- [Notifications par email](#notifications-par-email)
- [Notifications Slack/Discord](#notifications-slackdiscord)
- [Notifications SMS et téléphone](#notifications-sms-et-téléphone)
- [Notifications push et desktop](#notifications-push-et-desktop)
- [Système de notification intelligent](#système-de-notification-intelligent)
- [Conclusion](#conclusion)

## Introduction

Les notifications automatisées sont essentielles pour maintenir la visibilité sur l'état de vos systèmes et applications. Un système de notification bien conçu informe les bonnes personnes au bon moment avec les bonnes informations, permettant une réponse rapide aux incidents et une gestion proactive des problèmes.

Imaginez les notifications comme un système d'alarme intelligent : il ne sonne que quand c'est vraiment nécessaire, fournit des informations contextuelles utiles, et guide vers la résolution du problème plutôt que de simplement signaler qu'un problème existe.

## Notifications par email

### Configuration de base

**Envoi d'email simple** :
```bash
#!/bin/bash
# Envoi d'email de base

send_email() {
    local to="$1"
    local subject="$2"
    local body="$3"
    
    echo "$body" | mail -s "$subject" "$to"
}

# Utilisation
send_email "admin@example.com" "Alerte système" "Le serveur nécessite une attention"
```

**Email avec formatage** :
```bash
#!/bin/bash
# Email formaté avec HTML

send_formatted_email() {
    local to="$1"
    local subject="$2"
    local title="$3"
    local content="$4"
    local priority="${5:-normal}"  # low, normal, high, urgent
    
    local html_body=$(cat <<EOF
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .header { background-color: #4CAF50; color: white; padding: 10px; }
        .content { padding: 20px; }
        .footer { background-color: #f1f1f1; padding: 10px; font-size: 12px; }
        .urgent { border-left: 5px solid red; }
        .high { border-left: 5px solid orange; }
        .normal { border-left: 5px solid blue; }
    </style>
</head>
<body>
    <div class="header">
        <h2>$title</h2>
    </div>
    <div class="content $priority">
        <pre>$content</pre>
    </div>
    <div class="footer">
        <p>Généré automatiquement le $(date)</p>
    </div>
</body>
</html>
EOF
)
    
    echo "$html_body" | mail -a "Content-Type: text/html" \
        -s "$subject" "$to"
}
```

### Email avec pièces jointes

**Email avec fichiers attachés** :
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
        echo
        echo "--boundary123"
        echo "Content-Type: text/plain"
        echo
        echo "$body"
        echo
        echo "--boundary123"
        echo "Content-Type: application/octet-stream"
        echo "Content-Disposition: attachment; filename=\"$(basename "$attachment")\""
        echo
        cat "$attachment"
        echo
        echo "--boundary123--"
    ) | sendmail "$to"
}
```

### Système d'alerte par email

**Système d'alerte complet** :
```bash
#!/bin/bash
# Système d'alerte par email

ALERT_EMAIL="admin@example.com"
ALERT_LOG="/var/log/alerts.log"

send_alert() {
    local level="$1"  # INFO, WARNING, ERROR, CRITICAL
    local message="$2"
    local details="${3:-}"
    
    local subject="[${level}] $(hostname) - $(date +%Y-%m-%d\ %H:%M:%S)"
    
    # Logger l'alerte
    echo "[$(date)] [$level] $message" >> "$ALERT_LOG"
    
    # Envoyer selon le niveau
    case "$level" in
        CRITICAL|ERROR)
            send_formatted_email "$ALERT_EMAIL" "$subject" \
                "Alerte $level" "$message\n\n$details" "urgent"
            ;;
        WARNING)
            send_formatted_email "$ALERT_EMAIL" "$subject" \
                "Alerte $level" "$message\n\n$details" "high"
            ;;
        INFO)
            # Ne pas envoyer d'email pour INFO, juste logger
            ;;
    esac
}

# Utilisation
send_alert "CRITICAL" "Disque plein" "Utilisation: 95% sur /var"
send_alert "WARNING" "CPU élevé" "Utilisation: 85%"
```

## Notifications Slack/Discord

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
        echo "Erreur: SLACK_WEBHOOK_URL non définie"
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
        "footer": "$(hostname)",
        "ts": $(date +%s)
    }]
}
EOF
)
    
    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" \
        "$SLACK_WEBHOOK_URL"
}

# Utilisation
send_slack_message "#alerts" "Serveur nécessite attention" "danger" "Alerte Critique"
```

**Slack avec formatage avancé** :
```bash
#!/bin/bash
# Slack avec formatage riche

send_slack_rich() {
    local channel="$1"
    local title="$2"
    local fields="$3"  # JSON array de fields
    
    local payload=$(cat <<EOF
{
    "channel": "$channel",
    "attachments": [{
        "color": "danger",
        "title": "$title",
        "fields": $fields,
        "footer": "Shellcraft Monitoring",
        "ts": $(date +%s)
    }]
}
EOF
)
    
    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" \
        "$SLACK_WEBHOOK_URL"
}

# Exemple avec champs
send_slack_rich "#alerts" "Rapport système" \
    '[{"title": "CPU", "value": "85%", "short": true},
      {"title": "RAM", "value": "70%", "short": true},
      {"title": "Disque", "value": "60%", "short": true}]'
```

### Intégration Discord

**Webhook Discord** :
```bash
#!/bin/bash
# Notifications Discord

DISCORD_WEBHOOK_URL="${DISCORD_WEBHOOK_URL:-}"

send_discord_message() {
    local content="$1"
    local title="${2:-Notification}"
    local color="${3:-16711680}"  # Rouge par défaut
    
    if [ -z "$DISCORD_WEBHOOK_URL" ]; then
        echo "Erreur: DISCORD_WEBHOOK_URL non définie"
        return 1
    fi
    
    local payload=$(cat <<EOF
{
    "embeds": [{
        "title": "$title",
        "description": "$content",
        "color": $color,
        "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
        "footer": {
            "text": "$(hostname)"
        }
    }]
}
EOF
)
    
    curl -X POST -H 'Content-type: application/json' \
        --data "$payload" \
        "$DISCORD_WEBHOOK_URL"
}

# Utilisation
send_discord_message "Le système nécessite une attention immédiate" \
    "Alerte Critique" 16711680
```

## Notifications SMS et téléphone

### SMS via API

**SMS avec Twilio** :
```bash
#!/bin/bash
# Notifications SMS avec Twilio

TWILIO_ACCOUNT_SID="${TWILIO_ACCOUNT_SID:-}"
TWILIO_AUTH_TOKEN="${TWILIO_AUTH_TOKEN:-}"
TWILIO_FROM="${TWILIO_FROM:-}"

send_sms() {
    local to="$1"
    local message="$2"
    
    if [ -z "$TWILIO_ACCOUNT_SID" ] || [ -z "$TWILIO_AUTH_TOKEN" ]; then
        echo "Erreur: Credentials Twilio non configurés"
        return 1
    fi
    
    curl -X POST "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID/Messages.json" \
        --data-urlencode "From=$TWILIO_FROM" \
        --data-urlencode "To=$to" \
        --data-urlencode "Body=$message" \
        -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN"
}

# Utilisation pour alertes critiques
send_sms "+1234567890" "ALERTE: Serveur critique nécessite attention immédiate"
```

### Appels téléphoniques

**Appel avec Twilio** :
```bash
#!/bin/bash
# Appel téléphonique avec Twilio

make_phone_call() {
    local to="$1"
    local message="$2"
    
    # Créer un TwiML pour l'appel
    local twiml=$(cat <<EOF
<?xml version="1.0" encoding="UTF-8"?>
<Response>
    <Say voice="alice" language="fr-FR">$message</Say>
    <Pause length="2"/>
    <Say voice="alice" language="fr-FR">Répétition. $message</Say>
</Response>
EOF
)
    
    # Enregistrer le TwiML
    local twiml_url="https://your-server.com/twiml/alert.xml"
    echo "$twiml" > /var/www/twiml/alert.xml
    
    # Initier l'appel
    curl -X POST "https://api.twilio.com/2010-04-01/Accounts/$TWILIO_ACCOUNT_SID/Calls.json" \
        --data-urlencode "From=$TWILIO_FROM" \
        --data-urlencode "To=$to" \
        --data-urlencode "Url=$twiml_url" \
        -u "$TWILIO_ACCOUNT_SID:$TWILIO_AUTH_TOKEN"
}
```

## Notifications push et desktop

### Notifications desktop Linux

**Notifications système** :
```bash
#!/bin/bash
# Notifications desktop Linux

send_desktop_notification() {
    local title="$1"
    local message="$2"
    local urgency="${3:-normal}"  # low, normal, critical
    local icon="${4:-dialog-information}"
    
    if command -v notify-send &> /dev/null; then
        notify-send -u "$urgency" -i "$icon" "$title" "$message"
    elif command -v kdialog &> /dev/null; then
        kdialog --title "$title" --msgbox "$message"
    elif command -v zenity &> /dev/null; then
        zenity --info --title "$title" --text "$message"
    fi
}

# Utilisation
send_desktop_notification "Alerte système" \
    "Le serveur nécessite une attention" "critical" "dialog-warning"
```

### Notifications macOS

**Notifications macOS** :
```bash
#!/bin/bash
# Notifications macOS

send_macos_notification() {
    local title="$1"
    local message="$2"
    local subtitle="${3:-}"
    
    osascript -e "display notification \"$message\" with title \"$title\"${subtitle:+ subtitle \"$subtitle\"}"
}

# Utilisation
send_macos_notification "Alerte système" \
    "Le serveur nécessite une attention" "Monitoring"
```

### Notifications Windows

**Notifications Windows PowerShell** :
```powershell
# Notifications Windows

function Send-WindowsNotification {
    param(
        [string]$Title,
        [string]$Message,
        [string]$Icon = "Info"
    )
    
    [Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
    [Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom.XmlDocument, ContentType = WindowsRuntime] | Out-Null
    
    $template = @"
<toast>
    <visual>
        <binding template="ToastGeneric">
            <text>$Title</text>
            <text>$Message</text>
        </binding>
    </visual>
</toast>
"@
    
    $xml = New-Object Windows.Data.Xml.Dom.XmlDocument
    $xml.LoadXml($template)
    
    $toast = [Windows.UI.Notifications.ToastNotification]::new($xml)
    [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier("Shellcraft").Show($toast)
}
```

## Système de notification intelligent

### Routeur de notifications

**Système intelligent multi-canal** :
```bash
#!/bin/bash
# Routeur de notifications intelligent

NOTIFICATION_CONFIG="/etc/notifications.conf"

route_notification() {
    local level="$1"
    local message="$2"
    local context="${3:-}"
    
    # Déterminer les canaux selon le niveau et le contexte
    local channels=$(determine_channels "$level" "$context")
    
    # Envoyer sur chaque canal
    for channel in $channels; do
        case "$channel" in
            email)
                send_email "$ALERT_EMAIL" "[$level] $message" "$context"
                ;;
            slack)
                send_slack_message "#alerts" "$message" \
                    "$(get_color_for_level "$level")" "$level"
                ;;
            sms)
                if [[ "$level" == "CRITICAL" ]]; then
                    send_sms "$ADMIN_PHONE" "$message"
                fi
                ;;
            desktop)
                send_desktop_notification "$level" "$message" \
                    "$(get_urgency_for_level "$level")"
                ;;
        esac
    done
}

determine_channels() {
    local level="$1"
    local context="$2"
    
    case "$level" in
        CRITICAL)
            echo "email slack sms desktop"
            ;;
        ERROR)
            echo "email slack desktop"
            ;;
        WARNING)
            echo "email slack"
            ;;
        INFO)
            echo "slack"
            ;;
    esac
}
```

### Notifications avec IA

**Notifications intelligentes** :
```bash
#!/bin/bash
# Notifications avec analyse IA

send_intelligent_notification() {
    local event_data="$1"
    
    # Analyser l'événement avec IA
    local analysis=$(ai_analyze "Analyse cet événement système et génère une notification appropriée:
$event_data

La notification doit:
- Être claire et actionnable
- Inclure le contexte nécessaire
- Suggérer des actions de résolution
- Déterminer le niveau d'urgence approprié")
    
    # Extraire les informations
    local level=$(echo "$analysis" | grep -oP "Level: \K\w+")
    local message=$(echo "$analysis" | grep -oP "Message: \K.+")
    local actions=$(echo "$analysis" | grep -oP "Actions: \K.+")
    
    # Envoyer la notification enrichie
    route_notification "$level" "$message" "Actions suggérées: $actions"
}
```

## Conclusion

Les notifications automatisées sont essentielles pour maintenir la visibilité sur vos systèmes. En combinant différents canaux de notification et en utilisant l'intelligence artificielle pour déterminer quand et comment notifier, vous créez un système d'alerte efficace qui informe sans submerger.

Un bon système de notification équilibre la nécessité d'être informé avec le besoin de ne pas être distrait par des alertes non essentielles. L'IA peut aider à trouver cet équilibre en analysant le contexte et en déterminant la meilleure façon de communiquer chaque événement.

Dans le chapitre suivant, nous explorerons l'analyse de données et les scripts multi-plateforme, découvrant comment traiter et analyser des données complexes à travers différents systèmes d'exploitation.

