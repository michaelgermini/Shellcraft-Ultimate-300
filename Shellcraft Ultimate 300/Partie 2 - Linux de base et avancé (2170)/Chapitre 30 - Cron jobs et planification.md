# Chapitre 30 - Cron jobs et planification

## Table des matières
- [Introduction](#introduction)
- [Fonctionnement de cron](#fonctionnement-de-cron)
- [Syntaxe des crontabs](#syntaxe-des-crontabs)
- [Gestion des crontabs](#gestion-des-crontabs)
- [Exemples pratiques](#exemples-pratiques)
- [Variables d'environnement dans cron](#variables-denvironnement-dans-cron)
- [Gestion des erreurs et logging](#gestion-des-erreurs-et-logging)
- [Sécurité et permissions](#sécurité-et-permissions)
- [Alternatives à cron](#alternaires-à-cron)
- [Débogage des tâches cron](#débogage-des-tâches-cron)
- [Bonnes pratiques](#bonnes-pratiques)
- [Automatisation avancée](#automatisation-avancée)
- [Conclusion](#conclusion)

## Introduction

Cron constitue le système de planification historique d'UNIX/Linux, permettant l'exécution automatique de tâches à des moments précis ou selon des intervalles réguliers. De la sauvegarde nocturne aux vérifications de sécurité, cron transforme des tâches manuelles en processus automatisés fiables.

Imaginez cron comme un secrétaire personnel infatigable : il se souvient de vos rendez-vous (tâches planifiées), exécute vos tâches routinières aux bons moments, et vous alerte si quelque chose ne va pas. Contrairement à un assistant humain, cron ne dort jamais, n'oublie rien, et exécute vos instructions avec une précision parfaite.

## Fonctionnement de cron

### Architecture de cron

**Composants principaux** :
```bash
# Daemon cron
/usr/sbin/cron          # ou /usr/sbin/crond

# Configuration système
/etc/crontab           # Tâches système
/etc/cron.d/           # Configurations supplémentaires
/etc/cron.hourly/      # Scripts exécutés toutes les heures
/etc/cron.daily/       # Scripts quotidiens
/etc/cron.weekly/      # Scripts hebdomadaires
/etc/cron.monthly/     # Scripts mensuels

# Crontabs utilisateurs
/var/spool/cron/crontabs/  # Stockage des crontabs utilisateur
```

### Processus cron

**Cycle d'exécution** :
1. **Lecture** : Cron lit les crontabs toutes les minutes
2. **Planification** : Calcule les prochaines exécutions
3. **Exécution** : Lance les commandes aux moments prévus
4. **Logging** : Enregistre les exécutions et erreurs

**Gestion des ressources** :
- Une instance unique par système
- Exécution en arrière-plan
- Gestion automatique des processus enfant
- Nettoyage des processus zombies

## Syntaxe des crontabs

### Format de base

**Structure d'une ligne crontab** :
```
* * * * * command_to_execute
│ │ │ │ │
│ │ │ │ └─ Jour de la semaine (0-7, 0/7=dimanche)
│ │ │ └─ Mois (1-12)
│ │ └─ Jour du mois (1-31)
│ └─ Heure (0-23)
└─ Minute (0-59)
```

**Exemples concrets** :
```bash
# Toutes les minutes
* * * * * /usr/local/bin/check_status.sh

# Toutes les heures, à la 30ème minute
30 * * * * /usr/local/bin/hourly_backup.sh

# Tous les jours à 2h du matin
0 2 * * * /usr/local/bin/daily_maintenance.sh

# Tous les lundis à 9h
0 9 * * 1 /usr/local/bin/weekly_report.sh

# Le 1er de chaque mois à minuit
0 0 1 * * /usr/local/bin/monthly_cleanup.sh
```

### Raccourcis spéciaux

**Macros pratiques** :
```bash
# Exécution toutes les heures
@hourly /usr/local/bin/hourly_task.sh

# Quotidienne (2h du matin)
@daily /usr/local/bin/daily_backup.sh

# Hebdomadaire (dimanche 0h)
@weekly /usr/local/bin/weekly_maintenance.sh

# Mensuelle (1er du mois 0h)
@monthly /usr/local/bin/monthly_report.sh

# Au redémarrage
@reboot /usr/local/bin/startup_tasks.sh
```

### Expressions complexes

**Listes et intervalles** :
```bash
# Plusieurs valeurs spécifiques
0 9,12,18 * * * /usr/local/bin/check_status.sh  # 9h, 12h, 18h

# Intervalles
*/15 * * * * /usr/local/bin/quarter_hourly.sh   # Toutes les 15 minutes
0 9-17 * * 1-5 /usr/local/bin/business_hours.sh # 9h-17h, lun-ven

# Combinaisons
0 8,12,18 * * 1,3,5 /usr/local/bin/odd_days.sh  # Lundi, mercredi, vendredi
```

**Jours de la semaine** :
```bash
# Noms ou numéros
0 9 * * mon /usr/local/bin/monday_task.sh    # Lundi
0 9 * * 1 /usr/local/bin/monday_task.sh      # Même chose

# Week-end
0 10 * * 6,7 /usr/local/bin/weekend_task.sh  # Samedi et dimanche
```

## Gestion des crontabs

### Édition des crontabs

**Pour l'utilisateur courant** :
```bash
# Éditer sa propre crontab
crontab -e

# Lister ses tâches
crontab -l

# Supprimer toutes ses tâches
crontab -r

# Sauvegarder avant suppression
crontab -l > my_crontab_backup.txt
crontab -r
```

**Pour d'autres utilisateurs (root)** :
```bash
# Éditer la crontab d'un utilisateur
crontab -u alice -e

# Lister les tâches d'un utilisateur
crontab -u alice -l

# Supprimer les tâches d'un utilisateur
crontab -u alice -r
```

### Crontab système

**Configuration globale** :
```bash
# /etc/crontab - Format avec utilisateur
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# Exemple de tâche système
*/10 * * * * root /usr/local/bin/system_check.sh
0 2 * * * root /usr/local/bin/daily_backup.sh
```

**Répertoire cron.d** :
```bash
# /etc/cron.d/myapp - Format comme /etc/crontab
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Tâches pour myapp
*/5 * * * * myapp /usr/local/myapp/bin/health_check.sh
0 3 * * * myapp /usr/local/myapp/bin/daily_maintenance.sh
```

## Exemples pratiques

### Sauvegarde automatique

**Sauvegarde quotidienne** :
```bash
# Crontab utilisateur
0 2 * * * /home/alice/scripts/backup_home.sh

# Script de sauvegarde
#!/bin/bash
SOURCE="/home/alice"
DEST="/mnt/backup/$(date +%Y%m%d)"
LOGFILE="/home/alice/logs/backup.log"

mkdir -p "$DEST"
rsync -av --delete "$SOURCE" "$DEST" >> "$LOGFILE" 2>&1

if [ $? -eq 0 ]; then
    echo "$(date): Sauvegarde réussie" >> "$LOGFILE"
else
    echo "$(date): ERREUR de sauvegarde" >> "$LOGFILE"
fi
```

### Maintenance système

**Nettoyage quotidien** :
```bash
# /etc/cron.daily/cleanup
#!/bin/bash
# Nettoyage quotidien du système

# Temporaires vieux
find /tmp -type f -mtime +7 -delete 2>/dev/null || true

# Cache utilisateur
find /home -name ".cache" -type d -exec rm -rf {}/* \; 2>/dev/null || true

# Logs archivés
find /var/log -name "*.gz" -mtime +30 -delete 2>/dev/null || true

# Paquets obsolètes
apt autoremove -y >/dev/null 2>&1 || true
apt autoclean -y >/dev/null 2>&1 || true
```

### Monitoring et alertes

**Vérifications périodiques** :
```bash
# /etc/cron.d/monitoring
*/5 * * * * root /usr/local/bin/check_services.sh
*/15 * * * * root /usr/local/bin/check_disk_space.sh
0 * * * * root /usr/local/bin/hourly_report.sh
0 6 * * * root /usr/local/bin/daily_summary.sh
```

**Script de vérification de services** :
```bash
#!/bin/bash
# check_services.sh

SERVICES=("sshd" "apache2" "mysql" "postgresql")
ADMIN_EMAIL="admin@example.com"
LOGFILE="/var/log/service_checks.log"

for service in "${SERVICES[@]}"; do
    if ! systemctl is-active --quiet "$service"; then
        MESSAGE="$service est arrêté sur $(hostname)"
        echo "$(date): $MESSAGE" >> "$LOGFILE"
        echo "$MESSAGE" | mail -s "Service arrêté: $service" "$ADMIN_EMAIL"
        
        # Tentative de redémarrage
        if systemctl restart "$service" 2>/dev/null; then
            echo "$(date): $service redémarré automatiquement" >> "$LOGFILE"
        fi
    fi
done
```

## Variables d'environnement dans cron

### Variables par défaut

**Configuration des variables** :
```bash
# Dans crontab -e
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=alice@example.com
HOME=/home/alice

# Variables personnalisées
MYAPP_CONFIG=/etc/myapp/config.ini
BACKUP_DEST=/mnt/external/backup
```

### Gestion des chemins

**PATH dans cron** :
```bash
# Problème courant : PATH limité dans cron
# Solution : définir explicitement

# Mauvais (risque d'échec)
* * * * * my_script.sh

# Bon
* * * * * /usr/local/bin/my_script.sh

# Ou définir PATH
PATH=/usr/local/bin:/usr/bin:/bin
* * * * * my_script.sh
```

### Variables dynamiques

**Calculs dans cron** :
```bash
# Utiliser des commandes pour les valeurs dynamiques
0 2 * * * /usr/local/bin/backup.sh $(date +\%Y\%m\%d) /mnt/backup

# Échappement des %
0 2 * * * /usr/local/bin/backup.sh $(date +\%Y\%m\%d) /mnt/backup
```

## Gestion des erreurs et logging

### Redirections de sortie

**Capturer la sortie** :
```bash
# Rediriger stdout et stderr
* * * * * /usr/local/bin/script.sh >> /var/log/cron.log 2>&1

# Fichiers séparés
* * * * * /usr/local/bin/script.sh 1>> /var/log/cron_output.log 2>> /var/log/cron_errors.log

# Silencieux sauf erreurs
* * * * * /usr/local/bin/script.sh >/dev/null 2>> /var/log/cron_errors.log
```

### Gestion des échecs

**Réessayer en cas d'échec** :
```bash
# Script avec retry
#!/bin/bash
# retry_script.sh

MAX_RETRIES=3
RETRY_DELAY=60

for ((i=1; i<=MAX_RETRIES; i++)); do
    if /usr/local/bin/unreliable_command.sh; then
        exit 0
    fi
    
    if [ $i -lt $MAX_RETRIES ]; then
        sleep $RETRY_DELAY
    fi
done

exit 1
```

**Notification d'échec** :
```bash
# Cron avec notification
MAILTO=admin@example.com
* * * * * /usr/local/bin/critical_task.sh || echo "Échec de critical_task.sh à $(date)" | mail -s "Alerte cron" $MAILTO
```

### Logging structuré

**Logs avec contexte** :
```bash
# Script avec logging
#!/bin/bash
LOGFILE="/var/log/cron_tasks.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [CRON] $*" >> "$LOGFILE"
}

log "Début de la tâche"
# ... traitement ...
if [ $? -eq 0 ]; then
    log "Tâche terminée avec succès"
else
    log "ERREUR: Échec de la tâche"
    exit 1
fi
```

## Sécurité et permissions

### Permissions des scripts

**Scripts exécutables** :
```bash
# Permissions appropriées
chmod 755 /usr/local/bin/my_script.sh

# Propriétaire correct
chown root:root /usr/local/bin/system_script.sh
chown alice:alice /home/alice/scripts/personal_script.sh
```

### Utilisateur approprié

**Tâches système** :
```bash
# /etc/crontab - Tâches root
* * * * * root /usr/local/bin/system_monitor.sh

# Tâches utilisateur
* * * * * alice /home/alice/scripts/personal_backup.sh
```

### Restrictions de sécurité

**Limiter l'accès** :
```bash
# /etc/cron.allow - Utilisateurs autorisés
alice
bob
admin

# /etc/cron.deny - Utilisateurs interdits (si cron.allow n'existe pas)
# Liste des utilisateurs non autorisés
```

### Sudo dans cron

**Utilisation de sudo** :
```bash
# Éviter sudo dans cron (préférer les permissions directes)
# Si nécessaire :
* * * * * alice sudo /usr/local/bin/admin_task.sh

# Avec NOPASSWD dans /etc/sudoers
alice ALL=(root) NOPASSWD: /usr/local/bin/admin_task.sh
```

## Alternatives à cron

### Systemd timers

**Timers modernes** :
```bash
# /etc/systemd/system/myapp.service
[Unit]
Description=My App Service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/myapp_task.sh

# /etc/systemd/system/myapp.timer
[Unit]
Description=Run myapp task every 10 minutes

[Timer]
OnBootSec=5min
OnUnitActiveSec=10min

[Install]
WantedBy=timers.target

# Gestion
sudo systemctl enable myapp.timer
sudo systemctl start myapp.timer
sudo systemctl list-timers
```

### Anacron

**Pour les machines non permanentes** :
```bash
# Installation
sudo apt install anacron

# Configuration /etc/anacrontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# period delay job-id command
1       5       cron.daily      nice run-parts /etc/cron.daily
7       10      cron.weekly     nice run-parts /etc/cron.weekly
@monthly 15     cron.monthly    nice run-parts /etc/cron.monthly
```

### Fcron

**Cron amélioré** :
```bash
# Installation
sudo apt install fcron

# Fonctionnalités avancées :
# - Exécution basée sur la charge système
# - Dépendances entre tâches
# - Options avancées de logging
# - Interface utilisateur
```

## Débogage des tâches cron

### Diagnostic des problèmes

**Vérification de l'exécution** :
```bash
# Logs de cron
grep CRON /var/log/syslog

# Logs spécifiques
journalctl -u cron

# Vérifier si cron tourne
systemctl status cron
```

**Test manuel** :
```bash
# Exécuter la commande manuellement
/usr/local/bin/my_script.sh

# Avec les mêmes variables d'environnement
env -i SHELL=/bin/bash PATH=/usr/bin:/bin /usr/local/bin/my_script.sh

# Simuler l'environnement cron
env -i HOME=/home/alice SHELL=/bin/bash PATH=/usr/bin:/bin LOGNAME=alice /usr/local/bin/my_script.sh
```

### Debugging avancé

**Script de débogage** :
```bash
#!/bin/bash
# debug_cron.sh - Tester une commande cron

COMMAND="$1"

if [ -z "$COMMAND" ]; then
    echo "Usage: $0 'command to test'"
    exit 1
fi

echo "=== Test de la commande cron ==="
echo "Commande: $COMMAND"
echo "Utilisateur: $(whoami)"
echo "Répertoire: $(pwd)"
echo "Date: $(date)"
echo ""

echo "=== Variables d'environnement ==="
env | grep -E "(SHELL|PATH|HOME|USER|LOGNAME)" | sort
echo ""

echo "=== Exécution de test ==="
eval "$COMMAND"
EXIT_CODE=$?

echo ""
echo "=== Résultat ==="
echo "Code de sortie: $EXIT_CODE"

if [ $EXIT_CODE -eq 0 ]; then
    echo "✓ Commande exécutée avec succès"
else
    echo "✗ Échec de la commande"
fi
```

### Problèmes courants

**Chemin incorrect** :
```bash
# Problème
* * * * * my_script.sh  # Ne trouve pas le script

# Solution
* * * * * /usr/local/bin/my_script.sh
```

**Variables manquantes** :
```bash
# Problème
* * * * * /usr/local/bin/script.sh  # Utilise des variables non définies

# Solution
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
* * * * * /usr/local/bin/script.sh
```

**Permissions insuffisantes** :
```bash
# Vérifier
ls -l /usr/local/bin/my_script.sh
# Doit être exécutable et accessible
```

## Bonnes pratiques

### Organisation des tâches

**Structure recommandée** :
```bash
# /etc/cron.d/ - Un fichier par application
# myapp
*/5 * * * * myapp /usr/local/myapp/bin/health_check.sh
0 2 * * * myapp /usr/local/myapp/bin/daily_backup.sh

# monitoring
*/10 * * * * root /usr/local/bin/system_monitor.sh
0 6 * * * root /usr/local/bin/daily_report.sh
```

### Nommage et documentation

**Crontabs documentées** :
```bash
# Crontab personnelle d'Alice Dupont
# Créé: 2024-01-15
# Dernière modification: 2024-01-20

# Sauvegarde quotidienne à 2h
0 2 * * * /home/alice/scripts/backup.sh

# Synchronisation des mails toutes les heures
0 * * * * /home/alice/scripts/sync_mail.sh

# Nettoyage hebdomadaire le dimanche
0 3 * * 0 /home/alice/scripts/weekly_cleanup.sh
```

### Gestion des ressources

**Éviter la surcharge** :
```bash
# Échelonner les tâches lourdes
0 2 * * * /usr/local/bin/backup_database.sh
0 3 * * * /usr/local/bin/backup_files.sh    # 1h plus tard
0 4 * * * /usr/local/bin/backup_configs.sh  # 1h plus tard

# Utiliser nice pour les tâches non critiques
0 2 * * * nice -n 10 /usr/local/bin/heavy_backup.sh
```

### Maintenance

**Révision régulière** :
```bash
# Lister toutes les tâches cron
for user in $(cut -f1 -d: /etc/passwd); do
    echo "=== $user ==="
    crontab -u "$user" -l 2>/dev/null || echo "Aucune tâche"
    echo ""
done

# Sauvegarder les configurations
crontab -l > ~/crontab_backup_$(date +%Y%m%d).txt
```

## Automatisation avancée

### Génération dynamique de crontabs

**Script de déploiement** :
```bash
#!/bin/bash
# deploy_cron.sh

APP_NAME="$1"
APP_USER="${2:-$APP_NAME}"
APP_SCRIPTS_DIR="/usr/local/$APP_NAME/bin"

# Générer la configuration cron
cat > "/etc/cron.d/$APP_NAME" << EOF
# Tâches automatiques pour $APP_NAME
# Généré le $(date) par $0

SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=admin@example.com

# Health check toutes les 5 minutes
*/5 * * * * $APP_USER $APP_SCRIPTS_DIR/health_check.sh

# Backup quotidien à 2h
0 2 * * * $APP_USER $APP_SCRIPTS_DIR/daily_backup.sh

# Maintenance hebdomadaire dimanche 3h
0 3 * * 0 $APP_USER $APP_SCRIPTS_DIR/weekly_maintenance.sh

# Rapport mensuel le 1er à 6h
0 6 1 * * $APP_USER $APP_SCRIPTS_DIR/monthly_report.sh
EOF

# Permissions appropriées
chmod 644 "/etc/cron.d/$APP_NAME"

# Recharger cron
systemctl reload cron

echo "Configuration cron déployée pour $APP_NAME"
```

### Monitoring des tâches cron

**Surveillance des exécutions** :
```bash
#!/bin/bash
# monitor_cron.sh

LOGFILE="/var/log/cron_monitor.log"
ADMIN_EMAIL="admin@example.com"

# Fonction de logging
log() {
    echo "$(date): $*" >> "$LOGFILE"
}

# Vérifier les tâches en échec récent
check_failed_crons() {
    local failed_tasks=$(grep -E "exit code [1-9]" /var/log/syslog | tail -10)
    
    if [ -n "$failed_tasks" ]; then
        log "Tâches cron en échec détectées"
        echo "$failed_tasks" | mail -s "Alertes cron" "$ADMIN_EMAIL"
    fi
}

# Vérifier les tâches non exécutées
check_missing_executions() {
    # Liste des tâches attendues aujourd'hui
    local expected_tasks="/tmp/expected_cron_tasks.txt"
    
    # Générer la liste des tâches attendues
    crontab -l | grep -v '^#' | grep -v '^$' > "$expected_tasks"
    
    # Vérifier les logs récents
    local recent_logs=$(grep CRON /var/log/syslog | tail -50)
    
    # TODO: Logique de vérification des exécutions manquées
}

# Exécution
check_failed_crons
check_missing_executions
```

## Conclusion

Cron représente l'automatisation temporelle par excellence sous Linux, permettant de transformer des tâches manuelles en processus fiables et prévisibles. De la simple sauvegarde nocturne aux vérifications de sécurité complexes, cron offre la flexibilité nécessaire pour maintenir des systèmes sains et efficaces.

La puissance de cron réside dans sa simplicité : une syntaxe claire, des options puissantes, et une intégration parfaite avec l'écosystème Linux. Cependant, cette puissance nécessite une gestion rigoureuse des permissions, des chemins, et des variables d'environnement.

Dans le chapitre suivant, nous explorerons le scripting Bash avancé, qui permettra de créer des scripts plus sophistiqués pour les tâches cron, incluant la gestion d'erreurs, les fonctions modulaires, et l'interaction avec l'utilisateur.

