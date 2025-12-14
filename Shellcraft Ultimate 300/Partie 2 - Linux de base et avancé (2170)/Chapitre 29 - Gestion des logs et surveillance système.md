# Chapitre 29 - Gestion des logs et surveillance système

## Table des matières
- [Introduction](#introduction)
- [Architecture des logs système](#architecture-des-logs-système)
- [Outils de visualisation des logs](#outils-de-visualisation-des-logs)
- [Filtrage et recherche dans les logs](#filtrage-et-recherche-dans-les-logs)
- [Rotation et archivage des logs](#rotation-et-archivage-des-logs)
- [Surveillance en temps réel](#surveillance-en-temps-réel)
- [Alertes et notifications](#alertes-et-notifications)
- [Analyse et rapports](#analyse-et-rapports)
- [Logs d'application personnalisés](#logs-dapplication-personnalisés)
- [Centralisation des logs](#centralisation-des-logs)
- [Sécurité et audit des logs](#sécurité-et-audit-des-logs)
- [Dépannage avancé](#dépannage-avancé)
- [Automatisation de la surveillance](#automatisation-de-la-surveillance)
- [Conclusion](#conclusion)

## Introduction

Les logs constituent la mémoire du système Linux, enregistrant chaque événement significatif depuis le démarrage jusqu'aux opérations courantes. Une bonne gestion des logs permet non seulement de diagnostiquer les problèmes passés, mais aussi de surveiller l'état du système en temps réel et d'anticiper les défaillances futures.

Imaginez les logs comme le journal de bord d'un navire : ils enregistrent la météo (état système), les manœuvres (opérations), les incidents (erreurs), et permettent au capitaine (administrateur) de comprendre ce qui s'est passé et d'ajuster le cap pour éviter les tempêtes à venir.

## Architecture des logs système

### Emplacements standards

**Répertoire principal** :
```bash
/var/log/
├── syslog          # Logs système généraux
├── auth.log        # Authentifications
├── kern.log        # Noyau
├── daemon.log      # Services système
├── messages        # Messages généraux
├── dmesg           # Buffer du noyau (dernier boot)
├── boot.log        # Démarrage du système
├── cron.log        # Tâches planifiées
├── mail.log        # Serveur de mail
└── apache2/        # Logs spécifiques aux applications
    ├── access.log
    └── error.log
```

### Types de logs

**Logs système** :
- **syslog** : Standard pour les messages système
- **journald** : Système moderne (systemd)
- **rsyslog** : Centralisation et routage

**Logs d'application** :
- **Apache/Nginx** : Accès web et erreurs
- **MySQL/PostgreSQL** : Requêtes et erreurs DB
- **Application personnalisée** : Logs métier

### Format des logs

**Format syslog standard** :
```
<timestamp> <hostname> <program>[<pid>]: <message>
Jan 15 10:30:15 server sshd[1234]: Accepted password for user from 192.168.1.100
├──┼─────────┼────┼─────┼─────────────────────────────────────────────────────
│  │         │    │     └─ Message
│  │         │    └─ PID du processus
│  │         └─ Programme source
│  └─ Nom d'hôte
└─ Timestamp
```

## Outils de visualisation des logs

### Commandes de base

**Visualisation simple** :
```bash
# Dernières lignes
tail /var/log/syslog

# Premières lignes
head /var/log/syslog

# Nombre spécifique de lignes
tail -50 /var/log/auth.log
head -20 /var/log/syslog

# Pagination
less /var/log/syslog
```

### Outils spécialisés

**journalctl (systemd)** :
```bash
# Logs récents
journalctl

# Logs d'un service
journalctl -u apache2

# Logs depuis un temps
journalctl --since "1 hour ago"
journalctl --since yesterday

# Suivi en temps réel
journalctl -f

# Par priorité
journalctl -p err  # Erreurs seulement
```

**dmesg (messages noyau)** :
```bash
# Messages du noyau
dmesg

# Avec timestamps lisibles
dmesg -T

# Niveau de verbosité
dmesg -l err      # Erreurs seulement
dmesg -l warn,err # Warnings et erreurs

# Suivi en temps réel
dmesg -w
```

## Filtrage et recherche dans les logs

### Recherche basique

**grep dans les logs** :
```bash
# Recherche simple
grep "ERROR" /var/log/syslog

# Recherche insensible à la casse
grep -i "error" /var/log/apache2/error.log

# Recherche dans plusieurs fichiers
grep "failed" /var/log/auth.log /var/log/syslog

# Comptage des occurrences
grep -c "ERROR" /var/log/app.log
```

### Expressions régulières avancées

**Patterns courants** :
```bash
# Adresses IP dans les logs
grep -E '\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b' /var/log/access.log

# Timestamps
grep -E '\b\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\b' /var/log/app.log

# Niveaux de log
grep -E '\b(DEBUG|INFO|WARN|ERROR|FATAL)\b' /var/log/app.log

# Utilisateurs dans auth.log
grep "session opened" /var/log/auth.log | grep -oP '(?<=for user )\w+'
```

### Filtrage par critères

**Filtrage temporel** :
```bash
# Aujourd'hui seulement
grep "$(date +%b\ %e)" /var/log/syslog

# Dernière heure
journalctl --since "1 hour ago"

# Plage spécifique
journalctl --since "2024-01-15 10:00" --until "2024-01-15 11:00"
```

**Filtrage par source** :
```bash
# Par programme
grep "sshd" /var/log/auth.log

# Par PID
journalctl _PID=1234

# Par unité systemd
journalctl -u nginx
```

## Rotation et archivage des logs

### logrotate

**Configuration de base** :
```bash
# /etc/logrotate.conf
# Rotation hebdomadaire
weekly

# Garder 4 semaines
rotate 4

# Créer nouveaux logs vides
create

# Compresser les archives
compress

# Inclure configurations spécifiques
include /etc/logrotate.d
```

**Configuration par application** :
```bash
# /etc/logrotate.d/apache2
/var/log/apache2/*.log {
    daily
    missingok
    rotate 52
    compress
    delaycompress
    notifempty
    create 644 www-data www-data
    postrotate
        systemctl reload apache2
    endscript
}
```

### Gestion manuelle

**Rotation manuelle** :
```bash
# Sauvegarde et réinitialisation
cp /var/log/app.log /var/log/app.log.1
cat /dev/null > /var/log/app.log

# Avec compression
gzip /var/log/app.log.1
```

**Nettoyage automatique** :
```bash
# Supprimer les logs vieux de 30 jours
find /var/log -name "*.gz" -mtime +30 -delete

# Archiver les anciens logs
tar -czf /backup/logs_$(date +%Y%m%d).tar.gz /var/log/*.gz
```

## Surveillance en temps réel

### tail et suivi

**Suivi continu** :
```bash
# Suivre un log
tail -f /var/log/syslog

# Plusieurs fichiers
tail -f /var/log/apache2/access.log /var/log/apache2/error.log

# Avec numéros de ligne
tail -f -n 50 /var/log/app.log

# Suivre avec grep
tail -f /var/log/app.log | grep "ERROR"
```

### multitail

**Surveillance multiple** :
```bash
# Installation
sudo apt install multitail

# Plusieurs logs dans des fenêtres séparées
multitail /var/log/syslog /var/log/auth.log

# Avec couleurs et filtres
multitail -c /var/log/apache2/error.log -e "ERROR" -i /var/log/apache2/access.log
```

### lnav (Log Navigator)

**Navigateur de logs intelligent** :
```bash
# Installation
sudo apt install lnav

# Navigation intelligente
lnav /var/log/syslog

# Fonctionnalités :
# - Coloration automatique
# - Filtrage temps réel
# - Recherche floue
# - Statistiques
```

## Alertes et notifications

### Alertes par email

**Configuration mail** :
```bash
# Installation d'un MTA simple
sudo apt install mailutils

# Script d'alerte
#!/bin/bash
LOGFILE="/var/log/app.log"
ADMIN_EMAIL="admin@example.com"

# Vérifier les erreurs récentes
if grep -q "CRITICAL" "$LOGFILE"; then
    SUBJECT="Alerte critique détectée"
    MESSAGE="Des erreurs CRITIQUES ont été détectées dans $LOGFILE"
    echo "$MESSAGE" | mail -s "$SUBJECT" "$ADMIN_EMAIL"
fi
```

### Notifications système

**notify-send** :
```bash
# Notification desktop
notify-send "Erreur détectée" "Vérifier les logs système" --icon=dialog-error

# Avec actions
notify-send "Sauvegarde terminée" "Cliquez pour ouvrir le rapport" --action=open:xdg-open /path/to/report
```

### Intégration avec monitoring

**Nagios/Icinga** :
```bash
# Plugin de vérification de logs
#!/bin/bash
LOGFILE="$1"
PATTERN="$2"
MAX_OCCURRENCES="$3"

count=$(grep -c "$PATTERN" "$LOGFILE" 2>/dev/null || echo "0")

if [ "$count" -gt "$MAX_OCCURRENCES" ]; then
    echo "CRITICAL: $count occurrences de '$PATTERN' trouvées"
    exit 2
elif [ "$count" -gt 0 ]; then
    echo "WARNING: $count occurrences de '$PATTERN' trouvées"
    exit 1
else
    echo "OK: Aucune occurrence de '$PATTERN'"
    exit 0
fi
```

## Analyse et rapports

### Statistiques de base

**Analyse des logs** :
```bash
# Nombre d'erreurs par heure
grep "ERROR" /var/log/app.log | cut -d' ' -f1 | cut -d':' -f1 | sort | uniq -c

# Top des IP dans access.log
cut -d' ' -f1 /var/log/apache2/access.log | sort | uniq -c | sort -nr | head -10

# Erreurs par type
grep "ERROR" /var/log/app.log | sed 's/.*ERROR: //' | sort | uniq -c | sort -nr
```

### Outils d'analyse avancés

**goaccess (analyse web)** :
```bash
# Installation
sudo apt install goaccess

# Analyse d'access.log
goaccess /var/log/apache2/access.log -o report.html --log-format=COMBINED

# Interface temps réel
goaccess /var/log/apache2/access.log --real-time-html
```

**awstats** :
```bash
# Installation
sudo apt install awstats

# Configuration pour Apache
# Génère des rapports détaillés
```

### Rapports personnalisés

**Script de rapport quotidien** :
```bash
#!/bin/bash
REPORT_DIR="/var/log/reports"
TODAY=$(date +%Y%m%d)
REPORT_FILE="$REPORT_DIR/daily_report_$TODAY.txt"

mkdir -p "$REPORT_DIR"

{
    echo "=== RAPPORT QUOTIDIEN DU $(date) ==="
    echo ""
    
    echo "=== STATISTIQUES SYSTÈME ==="
    uptime
    df -h
    echo ""
    
    echo "=== ERREURS DANS LES LOGS ==="
    for logfile in /var/log/syslog /var/log/auth.log; do
        if [ -f "$logfile" ]; then
            echo "--- $logfile ---"
            grep -c "ERROR\|CRITICAL\|FAILED" "$logfile" 2>/dev/null || echo "0"
        fi
    done
    echo ""
    
    echo "=== CONNEXIONS SSH ==="
    grep "sshd" /var/log/auth.log | grep -c "Accepted" 2>/dev/null || echo "0"
    echo ""
    
    echo "=== UTILISATION DISQUE ==="
    du -sh /home/* 2>/dev/null | sort -hr | head -5
    
} > "$REPORT_FILE"

echo "Rapport généré: $REPORT_FILE"
```

## Logs d'application personnalisés

### Configuration syslog

**Application avec syslog** :
```bash
# En shell
logger -p local0.info "Application démarrée"
logger -p local0.error "Erreur de connexion base de données"
logger -t myapp "Message personnalisé"

# En Python
import syslog
syslog.syslog(syslog.LOG_INFO, "Application démarrée")
syslog.syslog(syslog.LOG_ERR, "Erreur critique")
```

### Configuration rsyslog

**Règles personnalisées** :
```bash
# /etc/rsyslog.d/myapp.conf
# Logs de l'application vers fichier dédié
local0.*    /var/log/myapp.log

# Avec rotation
local0.*    -/var/log/myapp.log

# Vers serveur distant
local0.*    @logserver.example.com:514
```

### Framework de logging

**Script de logging standardisé** :
```bash
#!/bin/bash
# logger.sh - Framework de logging pour scripts

LOGFILE="/var/log/myapp.log"
LOGLEVEL=${LOGLEVEL:-INFO}

log() {
    local level="$1"
    local message="$2"
    local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    # Niveaux: DEBUG, INFO, WARN, ERROR
    case "$LOGLEVEL:$level" in
        DEBUG:*|INFO:INFO|INFO:WARN|INFO:ERROR|WARN:WARN|WARN:ERROR|ERROR:ERROR)
            echo "[$timestamp] [$level] $message" >> "$LOGFILE"
            ;;
    esac
}

# Utilisation
log "INFO" "Script démarré"
log "DEBUG" "Variable X = $X"
log "ERROR" "Échec de l'opération"
```

## Centralisation des logs

### rsyslog comme collecteur

**Configuration serveur** :
```bash
# /etc/rsyslog.conf
# Accepter les logs distants
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")

# Règles pour les logs distants
if $fromhost-ip != '127.0.0.1' then {
    /var/log/remote/$fromhost.log
    stop
}
```

**Configuration client** :
```bash
# /etc/rsyslog.d/remote.conf
# Envoyer tous les logs au serveur central
*.* @logserver.example.com:514

# Avec TCP pour fiabilité
*.* @@logserver.example.com:514
```

### ELK Stack

**Elasticsearch + Logstash + Kibana** :
```bash
# Logstash configuration basique
input {
  file {
    path => "/var/log/syslog"
    start_position => "beginning"
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
  }
}
```

### Alternatives modernes

**Loki + Promtail + Grafana** :
- Collecte légère avec Promtail
- Stockage efficient avec Loki
- Visualisation avec Grafana
- Intégration Prometheus

## Sécurité et audit des logs

### Protection des logs

**Permissions appropriées** :
```bash
# Logs système
sudo chown root:adm /var/log/syslog
sudo chmod 640 /var/log/syslog

# Logs d'application
sudo chown appuser:appgroup /var/log/myapp.log
sudo chmod 644 /var/log/myapp.log
```

**Intégrité des logs** :
```bash
# Vérification avec logcheck
sudo apt install logcheck

# Surveillance des modifications
sudo chattr +a /var/log/auth.log  # Append-only
```

### Audit système

**auditd pour la surveillance** :
```bash
# Installation
sudo apt install auditd

# Règles d'audit
sudo auditctl -w /var/log -p wa -k log_modification

# Consultation
sudo ausearch -k log_modification
```

### Détection d'intrusions

**Surveillance des anomalies** :
```bash
# Échecs d'authentification répétés
grep "Failed password" /var/log/auth.log | 
awk '{print $9}' | sort | uniq -c | sort -nr | 
awk '$1 > 5 {print "Alerte: " $2 " (" $1 " tentatives)"}'

# Accès root suspects
grep "session opened" /var/log/auth.log | 
grep -v "for user root" | 
grep "by (uid=0)"
```

## Dépannage avancé

### Analyse de performance

**Logs lents** :
```bash
# Identifier les goulots
journalctl --since "1 hour ago" | 
grep -E "(slow|timeout|delay)" | 
cut -d' ' -f1-3 | sort | uniq -c | sort -nr
```

**Mémoire pleine** :
```bash
# Recherche d'alertes OOM
grep -i "out of memory\|oom" /var/log/syslog

# Analyse de la mémoire
journalctl -k | grep -i memory
```

### Diagnostic réseau

**Logs réseau** :
```bash
# Connexions échouées
grep "connection refused\|connection timed out" /var/log/syslog

# Interfaces réseau
journalctl -u NetworkManager

# DHCP
grep "dhcp" /var/log/syslog
```

### Debugging d'application

**Logs d'erreurs détaillés** :
```bash
# Pile d'appels
journalctl -u myapp --since "1 hour ago" | grep -A 10 -B 5 "Exception"

# Variables d'environnement au crash
journalctl -u myapp --since "1 hour ago" | grep -A 20 "CRASH"
```

## Automatisation de la surveillance

### Scripts de monitoring

**Surveillance complète** :
```bash
#!/bin/bash
# system_monitor.sh

LOGDIR="/var/log/monitoring"
mkdir -p "$LOGDIR"

# Fonctions de vérification
check_disk_space() {
    local usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
    if [ "$usage" -gt 90 ]; then
        echo "$(date): ALERT: Disk usage ${usage}%" >> "$LOGDIR/disk.log"
        return 1
    fi
    return 0
}

check_services() {
    local services=("sshd" "apache2" "mysql")
    for service in "${services[@]}"; do
        if ! systemctl is-active --quiet "$service"; then
            echo "$(date): ALERT: Service $service is down" >> "$LOGDIR/services.log"
            return 1
        fi
    done
    return 0
}

check_logs() {
    local critical_count=$(grep -c "CRITICAL\|FATAL" /var/log/app.log 2>/dev/null || echo "0")
    if [ "$critical_count" -gt 0 ]; then
        echo "$(date): ALERT: $critical_count erreurs critiques" >> "$LOGDIR/app.log"
        return 1
    fi
    return 0
}

# Exécution
errors=0
check_disk_space || ((errors++))
check_services || ((errors++))
check_logs || ((errors++))

if [ "$errors" -gt 0 ]; then
    echo "$(date): $errors problème(s) détecté(s)" | mail -s "Alerte système" admin@example.com
fi
```

### Intégration cron

**Planification automatique** :
```bash
# /etc/cron.d/system_monitor
*/5 * * * * root /usr/local/bin/system_monitor.sh

# Rapport quotidien
0 6 * * * root /usr/local/bin/generate_daily_report.sh

# Nettoyage hebdomadaire
0 3 * * 0 root find /var/log -name "*.gz" -mtime +30 -delete
```

### Monitoring avec systemd

**Timers systemd** :
```bash
# /etc/systemd/system/log-monitor.service
[Unit]
Description=Log monitoring service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/log_monitor.sh

# /etc/systemd/system/log-monitor.timer
[Unit]
Description=Run log monitoring every 10 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=10min

[Install]
WantedBy=timers.target
```

## Conclusion

La gestion des logs et la surveillance système transforment un système réactif en plateforme proactive capable d'anticiper les problèmes et de fournir des diagnostics précis. Des outils de base comme `tail` et `grep` aux infrastructures complexes comme ELK Stack, chaque niveau ajoute de la visibilité et du contrôle.

La surveillance efficace nécessite trois piliers : la collecte systématique des données, l'analyse intelligente des patterns, et l'automatisation des réponses. Cette approche transforme les logs de simples enregistrements en outils de pilotage système.

Dans le chapitre suivant, nous explorerons les tâches planifiées avec cron, qui permettent d'automatiser l'exécution des scripts de surveillance et de maintenance à des moments précis.

