# Chapitre 36 - Gestion des services système (systemctl)

## Table des matières
- [Introduction](#introduction)
- [Systemd et architecture des services](#systemd-et-architecture-des-services)
- [Types d'unités systemd](#types-dunités-systemd)
- [Commandes systemctl essentielles](#commandes-systemctl-essentielles)
- [Gestion du cycle de vie des services](#gestion-du-cycle-de-vie-des-services)
- [Configuration des services](#configuration-des-services)
- [Fichiers unit et structure](#fichiers-unit-et-structure)
- [Dépendances et ordonnancement](#dépendances-et-ordonnancement)
- [Targets et niveaux d'exécution](#targets-et-niveaux-dexécution)
- [Monitoring et débogage](#monitoring-et-débogage)
- [journalctl : gestion des logs systemd](#journalctl--gestion-des-logs-systemd)
- [Services personnalisés](#services-personnalisés)
- [Timers systemd (alternative à cron)](#timers-systemd-alternative-à-cron)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Systemd constitue le cœur de la gestion des services dans les distributions Linux modernes. Au-delà d'un simple gestionnaire de démarrage, c'est un système complexe qui orchestre le cycle de vie de tous les processus système, gère les dépendances, et fournit des outils puissants de monitoring et de contrôle.

Imaginez systemd comme le chef d'orchestre d'un grand opéra : il coordonne l'entrée de chaque musicien au moment précis, s'assure que chacun joue sa partition, et peut intervenir instantanément si un instrument déraille. Dans un système Linux moderne, systemd ne gère pas seulement les services, mais aussi les montages de fichiers, les sockets réseau, les timers, et bien plus encore, créant un système intégré et cohérent.

## Systemd et architecture des services

### Concepts fondamentaux

**Systemd** :
- Gestionnaire de système et de services
- PID 1 (remplace init)
- Gère le démarrage, l'arrêt, et le monitoring des services
- Fournit journald pour les logs
- Gère les montages, sockets, timers, etc.

**Architecture** :
```bash
#!/bin/bash
# Architecture systemd

# Processus principal
ps -p 1
# systemd (PID 1)

# Répertoires importants
echo "=== Répertoires systemd ==="
echo "/etc/systemd/system/     : Unités système personnalisées"
echo "/usr/lib/systemd/system/ : Unités installées par packages"
echo "/run/systemd/system/     : Unités runtime"
echo "/var/lib/systemd/        : État persistant"
```

### Avantages de systemd

**Avantages principaux** :
- Démarrage parallèle des services
- Gestion des dépendances automatique
- Monitoring intégré des processus
- Logs centralisés avec journald
- Configuration déclarative
- Gestion des cgroups intégrée
- Activation à la demande (socket activation)

## Types d'unités systemd

### Types d'unités

**Types principaux** :
```bash
#!/bin/bash
# Types d'unités systemd

# .service : Services (programmes)
# .socket  : Sockets réseau/Unix
# .target  : Groupes d'unités (comme runlevels)
# .mount   : Points de montage
# .automount : Montage automatique
# .timer   : Tâches planifiées (alternative cron)
# .path    : Surveillance de chemins
# .slice   : Groupes de contrôle (cgroups)
# .scope   : Processus externes

# Lister tous les types
systemctl list-units --type=service
systemctl list-units --type=socket
systemctl list-units --type=timer
systemctl list-units --type=mount
```

## Commandes systemctl essentielles

### Commandes de base

**Gestion des services** :
```bash
#!/bin/bash
# Commandes systemctl de base

# Statut d'un service
systemctl status apache2
systemctl status nginx

# Démarrer un service
sudo systemctl start apache2

# Arrêter un service
sudo systemctl stop apache2

# Redémarrer un service
sudo systemctl restart apache2

# Recharger la configuration (si supporté)
sudo systemctl reload apache2

# Activer au démarrage
sudo systemctl enable apache2

# Désactiver au démarrage
sudo systemctl disable apache2

# Activer et démarrer en une commande
sudo systemctl enable --now apache2

# Désactiver et arrêter
sudo systemctl disable --now apache2
```

**Commandes d'information** :
```bash
#!/bin/bash
# Commandes d'information

# Lister tous les services
systemctl list-units --type=service

# Lister seulement les services actifs
systemctl list-units --type=service --state=running

# Lister les services échoués
systemctl list-units --type=service --state=failed

# Lister les services activés
systemctl list-unit-files --type=service --state=enabled

# Lister les services désactivés
systemctl list-unit-files --type=service --state=disabled

# Vérifier si un service est actif
systemctl is-active apache2

# Vérifier si un service est activé
systemctl is-enabled apache2

# Vérifier si un service a échoué
systemctl is-failed apache2
```

## Gestion du cycle de vie des services

### États des services

**États possibles** :
```bash
#!/bin/bash
# États des services

# active (running) : Service en cours d'exécution
# active (exited)  : Service terminé avec succès (one-shot)
# active (waiting) : Service en attente d'événement
# inactive         : Service arrêté
# activating       : Service en cours de démarrage
# deactivating     : Service en cours d'arrêt
# failed           : Service en échec

# Vérifier l'état détaillé
systemctl show apache2 -p ActiveState -p SubState

# Vérifier l'état de tous les services
systemctl list-units --type=service --all
```

### Cycle de vie complet

**Transitions d'état** :
```bash
#!/bin/bash
# Cycle de vie d'un service

# 1. Service désactivé et arrêté
systemctl status apache2
# inactive (dead)

# 2. Activation
sudo systemctl enable apache2
# enabled

# 3. Démarrage
sudo systemctl start apache2
# active (running)

# 4. Vérification
systemctl is-active apache2
# active

# 5. Arrêt
sudo systemctl stop apache2
# inactive (dead)

# 6. Redémarrage
sudo systemctl restart apache2
# active (running)
```

## Configuration des services

### Fichiers unit et structure

**Localisation des fichiers** :
```bash
#!/bin/bash
# Localisation des fichiers unit

# Unités système (priorité haute)
ls /etc/systemd/system/

# Unités installées par packages
ls /usr/lib/systemd/system/

# Unités runtime (temporaires)
ls /run/systemd/system/

# Ordre de priorité :
# 1. /run/systemd/system/ (runtime, priorité maximale)
# 2. /etc/systemd/system/ (système)
# 3. /usr/lib/systemd/system/ (packages)
```

**Structure d'un fichier unit** :
```bash
#!/bin/bash
# Structure d'un fichier unit

# Exemple de fichier service
cat > /tmp/example.service << 'EOF'
[Unit]
Description=Service Example
After=network.target
Wants=network-online.target

[Service]
Type=simple
User=www-data
ExecStart=/usr/bin/my-service
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

### Sections d'un fichier unit

**Section [Unit]** :
```bash
#!/bin/bash
# Section [Unit]

# Description : Description du service
# Documentation : URLs de documentation
# Requires : Dépendances strictes (si échoue, le service échoue)
# Wants : Dépendances souples (si échoue, continue quand même)
# After : Démarre après ces unités
# Before : Démarre avant ces unités
# Conflicts : Conflit avec ces unités
# ConditionPathExists : Condition sur existence de fichier
# ConditionPathIsDirectory : Condition sur répertoire

# Exemple
cat > /tmp/unit-example.service << 'EOF'
[Unit]
Description=My Service
Documentation=man:myservice(8)
After=network.target mysql.service
Wants=network-online.target
Requires=mysql.service
Conflicts=old-service.service
ConditionPathExists=/etc/myapp/config.conf
EOF
```

**Section [Service]** :
```bash
#!/bin/bash
# Section [Service]

# Type : Type de service
# - simple : Commande principale (défaut)
# - forking : Fork et exit (parent meurt)
# - oneshot : Exécution unique puis exit
# - notify : Notifie systemd quand prêt
# - idle : Attend que tous les jobs soient terminés

# ExecStart : Commande de démarrage
# ExecStop : Commande d'arrêt
# ExecReload : Commande de rechargement
# Restart : Politique de redémarrage
# - no : Ne pas redémarrer
# - always : Toujours redémarrer
# - on-success : Redémarrer seulement si succès
# - on-failure : Redémarrer seulement si échec

# User/Group : Utilisateur/groupe d'exécution
# WorkingDirectory : Répertoire de travail
# Environment : Variables d'environnement

cat > /tmp/service-example.service << 'EOF'
[Service]
Type=simple
User=myuser
Group=mygroup
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/myapp --config /etc/myapp.conf
ExecStop=/bin/kill -TERM $MAINPID
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5
Environment="APP_ENV=production"
EnvironmentFile=/etc/myapp/environment
EOF
```

**Section [Install]** :
```bash
#!/bin/bash
# Section [Install]

# WantedBy : Target qui veut ce service
# RequiredBy : Target qui requiert ce service
# Also : Unités à activer/désactiver avec celle-ci
# Alias : Alias pour le service

cat > /tmp/install-example.service << 'EOF'
[Install]
WantedBy=multi-user.target
RequiredBy=graphical.target
Also=myapp-socket.service
Alias=myapp.service
EOF
```

## Dépendances et ordonnancement

### Gestion des dépendances

**Dépendances** :
```bash
#!/bin/bash
# Gestion des dépendances

# Lister les dépendances d'un service
systemctl list-dependencies apache2

# Lister avec arbre inversé (qui dépend de ce service)
systemctl list-dependencies --reverse apache2

# Lister toutes les dépendances (récursif)
systemctl list-dependencies --all apache2

# Vérifier les dépendances avant démarrage
systemctl show apache2 -p Requires -p Wants -p After -p Before
```

**Création de dépendances** :
```bash
#!/bin/bash
# Créer des dépendances

# Service qui nécessite MySQL
create_service_with_dependency() {
    cat > /tmp/myapp.service << 'EOF'
[Unit]
Description=My Application
After=mysql.service network.target
Requires=mysql.service
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/myapp
Restart=always

[Install]
WantedBy=multi-user.target
EOF
    
    sudo cp /tmp/myapp.service /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl enable myapp.service
}
```

## Targets et niveaux d'exécution

### Comprendre les targets

**Targets système** :
```bash
#!/bin/bash
# Targets systemd

# Targets équivalents aux runlevels :
# runlevel 0 → poweroff.target
# runlevel 1 → rescue.target
# runlevel 2,3,4 → multi-user.target
# runlevel 5 → graphical.target
# runlevel 6 → reboot.target

# Lister les targets
systemctl list-units --type=target

# Vérifier le target actuel
systemctl get-default

# Changer le target par défaut
sudo systemctl set-default multi-user.target
sudo systemctl set-default graphical.target

# Changer de target (équivalent init 3, init 5)
sudo systemctl isolate multi-user.target
sudo systemctl isolate graphical.target
```

**Targets personnalisés** :
```bash
#!/bin/bash
# Créer un target personnalisé

create_custom_target() {
    cat > /tmp/my-custom.target << 'EOF'
[Unit]
Description=My Custom Target
Requires=multi-user.target
After=multi-user.target
EOF
    
    sudo cp /tmp/my-custom.target /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl set-default my-custom.target
}
```

## Monitoring et débogage

### Commandes de monitoring

**Monitoring avancé** :
```bash
#!/bin/bash
# Monitoring avancé avec systemctl

# Informations détaillées sur un service
systemctl show apache2

# Propriétés spécifiques
systemctl show apache2 -p ActiveState -p SubState -p MainPID

# Toutes les propriétés
systemctl show apache2 --all

# Informations sur les processus
systemctl status apache2 -l --no-pager

# Suivi en temps réel
systemctl status apache2 -f

# Vérifier les services échoués
systemctl --failed

# Réinitialiser un service échoué
sudo systemctl reset-failed apache2
```

**Scripts de monitoring** :
```bash
#!/bin/bash
# Scripts de monitoring

# Monitorer tous les services
monitor_all_services() {
    while true; do
        clear
        echo "=== Services actifs ==="
        systemctl list-units --type=service --state=running --no-pager | head -20
        echo ""
        echo "=== Services échoués ==="
        systemctl --failed --no-pager
        sleep 5
    done
}

# Vérifier la santé des services critiques
check_critical_services() {
    local services=("apache2" "mysql" "ssh")
    
    for service in "${services[@]}"; do
        if systemctl is-active --quiet "$service"; then
            echo "✓ $service : actif"
        else
            echo "✗ $service : inactif ou échoué"
            systemctl status "$service" --no-pager -l | head -5
        fi
    done
}
```

## journalctl : gestion des logs systemd

### Utilisation de base

**Commandes journalctl** :
```bash
#!/bin/bash
# Utilisation de journalctl

# Tous les logs
sudo journalctl

# Logs d'un service spécifique
sudo journalctl -u apache2

# Logs depuis un temps
sudo journalctl --since "1 hour ago"
sudo journalctl --since "2024-01-15 10:00:00"
sudo journalctl --since yesterday
sudo journalctl --since today

# Logs jusqu'à un temps
sudo journalctl --until "1 hour ago"

# Suivi en temps réel
sudo journalctl -f

# Suivi d'un service
sudo journalctl -u apache2 -f

# Logs depuis le dernier boot
sudo journalctl -b

# Logs d'un boot spécifique
sudo journalctl -b -1  # Boot précédent
sudo journalctl -b -2  # Avant-dernier boot
```

### Options avancées journalctl

**Filtrage et recherche** :
```bash
#!/bin/bash
# Options avancées journalctl

# Par priorité
sudo journalctl -p err      # Erreurs et plus critiques
sudo journalctl -p warning  # Warnings et plus critiques
sudo journalctl -p info     # Info et plus critiques
sudo journalctl -p debug    # Tout

# Par PID
sudo journalctl _PID=1234

# Par utilisateur
sudo journalctl _UID=1000

# Par exécutable
sudo journalctl /usr/bin/apache2

# Recherche de texte
sudo journalctl -u apache2 | grep "error"
sudo journalctl --grep="error"

# Format de sortie
sudo journalctl -o json      # JSON
sudo journalctl -o json-pretty  # JSON formaté
sudo journalctl -o cat       # Format brut
sudo journalctl -o verbose  # Format détaillé

# Limiter le nombre de lignes
sudo journalctl -n 50        # 50 dernières lignes
sudo journalctl -n 50 -f     # Suivre les 50 dernières

# Inverser l'ordre
sudo journalctl -r           # Plus récent en premier

# Exclure certains champs
sudo journalctl --no-pager   # Pas de pagination
```

**Scripts avec journalctl** :
```bash
#!/bin/bash
# Scripts avec journalctl

# Analyser les erreurs d'un service
analyze_service_errors() {
    local service="$1"
    
    echo "=== Erreurs pour $service ==="
    sudo journalctl -u "$service" -p err --since "24 hours ago" | \
    awk '{print $1, $2, $3, substr($0, index($0,$5))}' | \
    sort | uniq -c | sort -rn | head -10
}

# Statistiques de logs
log_statistics() {
    local service="${1:-}"
    
    if [ -n "$service" ]; then
        local total=$(sudo journalctl -u "$service" --since "24 hours ago" | wc -l)
        local errors=$(sudo journalctl -u "$service" -p err --since "24 hours ago" | wc -l)
        local warnings=$(sudo journalctl -u "$service" -p warning --since "24 hours ago" | wc -l)
    else
        local total=$(sudo journalctl --since "24 hours ago" | wc -l)
        local errors=$(sudo journalctl -p err --since "24 hours ago" | wc -l)
        local warnings=$(sudo journalctl -p warning --since "24 hours ago" | wc -l)
    fi
    
    echo "=== Statistiques logs (24h) ==="
    echo "Total: $total"
    echo "Erreurs: $errors"
    echo "Warnings: $warnings"
    if [ "$total" -gt 0 ]; then
        echo "Taux d'erreur: $((errors * 100 / total))%"
    fi
}
```

## Services personnalisés

### Création d'un service simple

**Service de base** :
```bash
#!/bin/bash
# Créer un service simple

create_simple_service() {
    local service_name="$1"
    local command="$2"
    local user="${3:-$USER}"
    
    cat > "/tmp/${service_name}.service" << EOF
[Unit]
Description=My Custom Service
After=network.target

[Service]
Type=simple
User=$user
ExecStart=$command
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
    
    sudo cp "/tmp/${service_name}.service" "/etc/systemd/system/"
    sudo systemctl daemon-reload
    sudo systemctl enable "${service_name}.service"
    
    echo "Service créé: ${service_name}.service"
    echo "Démarrer avec: sudo systemctl start ${service_name}"
}
```

### Service avec script

**Service avec script shell** :
```bash
#!/bin/bash
# Créer un service avec script

create_script_service() {
    local service_name="$1"
    local script_path="$2"
    
    # Créer le script s'il n'existe pas
    if [ ! -f "$script_path" ]; then
        cat > "$script_path" << 'SCRIPT'
#!/bin/bash
# Script de service

while true; do
    echo "$(date): Service running"
    sleep 60
done
SCRIPT
        chmod +x "$script_path"
    fi
    
    # Créer le fichier unit
    cat > "/tmp/${service_name}.service" << EOF
[Unit]
Description=Script Service
After=network.target

[Service]
Type=simple
ExecStart=$script_path
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
    
    sudo cp "/tmp/${service_name}.service" "/etc/systemd/system/"
    sudo systemctl daemon-reload
    sudo systemctl enable "${service_name}.service"
    
    echo "Service créé avec script: $script_path"
}
```

### Service avec environnement

**Service avec variables d'environnement** :
```bash
#!/bin/bash
# Service avec environnement

create_service_with_env() {
    local service_name="$1"
    local env_file="${2:-/etc/myapp/environment}"
    
    # Créer le fichier d'environnement
    sudo mkdir -p "$(dirname "$env_file")"
    sudo tee "$env_file" << 'EOF'
APP_ENV=production
APP_DEBUG=false
DATABASE_URL=postgresql://localhost/mydb
EOF
    
    # Créer le service
    cat > "/tmp/${service_name}.service" << EOF
[Unit]
Description=Service with Environment
After=network.target

[Service]
Type=simple
EnvironmentFile=$env_file
ExecStart=/usr/bin/myapp
Restart=always

[Install]
WantedBy=multi-user.target
EOF
    
    sudo cp "/tmp/${service_name}.service" "/etc/systemd/system/"
    sudo systemctl daemon-reload
}
```

## Timers systemd (alternative à cron)

### Création de timers

**Timer de base** :
```bash
#!/bin/bash
# Créer un timer systemd

create_timer() {
    local timer_name="$1"
    local service_name="$2"
    local schedule="${3:-daily}"
    
    # Créer le service
    cat > "/tmp/${service_name}.service" << EOF
[Unit]
Description=Timer Service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/my-script.sh
EOF
    
    # Créer le timer
    cat > "/tmp/${timer_name}.timer" << EOF
[Unit]
Description=Timer for ${service_name}
Requires=${service_name}.service

[Timer]
OnCalendar=$schedule
Persistent=true

[Install]
WantedBy=timers.target
EOF
    
    sudo cp "/tmp/${service_name}.service" "/etc/systemd/system/"
    sudo cp "/tmp/${timer_name}.timer" "/etc/systemd/system/"
    sudo systemctl daemon-reload
    sudo systemctl enable "${timer_name}.timer"
    sudo systemctl start "${timer_name}.timer"
    
    echo "Timer créé: ${timer_name}.timer"
}

# Exemples de schedules :
# OnCalendar=daily          # Tous les jours à minuit
# OnCalendar=hourly         # Toutes les heures
# OnCalendar=weekly         # Toutes les semaines
# OnCalendar=monthly        # Tous les mois
# OnCalendar=*-*-* 02:00:00 # Tous les jours à 2h
# OnCalendar=Mon..Fri 09:00 # Lun-Ven à 9h
# OnCalendar=*-*-1 00:00   # Premier jour du mois
```

**Gestion des timers** :
```bash
#!/bin/bash
# Gestion des timers

# Lister les timers
systemctl list-timers

# Lister tous les timers (actifs et inactifs)
systemctl list-timers --all

# Statut d'un timer
systemctl status mytimer.timer

# Voir le prochain déclenchement
systemctl list-timers mytimer.timer

# Déclencher manuellement
sudo systemctl start mytimer.service

# Activer/désactiver un timer
sudo systemctl enable mytimer.timer
sudo systemctl disable mytimer.timer
```

## Scripts d'automatisation

### Script complet de gestion

**Gestionnaire de services** :
```bash
#!/bin/bash
# Gestionnaire complet de services systemd

set -euo pipefail

service_manager() {
    local action="$1"
    shift
    
    case "$action" in
        list)
            list_services "$@"
            ;;
        status)
            show_status "$@"
            ;;
        start|stop|restart|reload)
            control_service "$action" "$@"
            ;;
        enable|disable)
            toggle_service "$action" "$@"
            ;;
        logs)
            show_logs "$@"
            ;;
        create)
            create_service "$@"
            ;;
        *)
            echo "Usage: service_manager {list|status|start|stop|restart|reload|enable|disable|logs|create}"
            exit 1
            ;;
    esac
}

list_services() {
    local filter="${1:-}"
    
    if [ -n "$filter" ]; then
        systemctl list-units --type=service --state="$filter"
    else
        systemctl list-units --type=service
    fi
}

show_status() {
    local service="$1"
    
    systemctl status "$service" -l --no-pager
}

control_service() {
    local action="$1"
    shift
    local service="$1"
    
    sudo systemctl "$action" "$service"
    echo "Service $service : $action"
}

toggle_service() {
    local action="$1"
    shift
    local service="$1"
    
    sudo systemctl "$action" "$service"
    echo "Service $service : $action"
}

show_logs() {
    local service="$1"
    local lines="${2:-50}"
    
    sudo journalctl -u "$service" -n "$lines" --no-pager
}

create_service() {
    local service_name="$1"
    local command="$2"
    local user="${3:-root}"
    
    cat > "/tmp/${service_name}.service" << EOF
[Unit]
Description=Service $service_name
After=network.target

[Service]
Type=simple
User=$user
ExecStart=$command
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
    
    sudo cp "/tmp/${service_name}.service" "/etc/systemd/system/"
    sudo systemctl daemon-reload
    sudo systemctl enable "${service_name}.service"
    
    echo "Service créé: ${service_name}.service"
}

# Utilisation
# service_manager list
# service_manager status apache2
# service_manager start apache2
# service_manager logs apache2 100
```

## Conclusion

Systemd et systemctl ont révolutionné la gestion des services Linux, offrant un système unifié, puissant et flexible. En maîtrisant les commandes systemctl, la création de fichiers unit, et l'utilisation de journalctl, vous pouvez administrer efficacement tous les aspects des services système.

Un système bien administré utilise systemd pour orchestrer le démarrage, gérer les dépendances, surveiller les services, et maintenir la stabilité du système. La compréhension approfondie de systemd permet non seulement de gérer les services existants, mais aussi de créer des services personnalisés robustes et professionnels.

Dans le chapitre suivant, nous explorerons la gestion du réseau avec les commandes modernes ip, ifconfig, et netstat, découvrant comment administrer efficacement les interfaces réseau et les connexions dans un système Linux.
