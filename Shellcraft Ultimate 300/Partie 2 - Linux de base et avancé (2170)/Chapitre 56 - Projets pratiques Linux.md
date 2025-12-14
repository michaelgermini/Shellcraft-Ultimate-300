# Chapitre 56 - Projets pratiques Linux

## Table des matières
- [Introduction](#introduction)
- [Projet 1 : Serveur web complet](#projet-1-serveur-web-complet)
- [Projet 2 : Système de monitoring](#projet-2-système-de-monitoring)
- [Projet 3 : Automatisation de déploiement](#projet-3-automatisation-de-déploiement)
- [Projet 4 : Solution de backup](#projet-4-solution-de-backup)
- [Projet 5 : Cluster de calcul](#projet-5-cluster-de-calcul)
- [Méthodologie de projet](#méthodologie-de-projet)
- [Évaluation et optimisation](#évaluation-et-optimisation)
- [Conclusion](#conclusion)

## Introduction

Les projets pratiques constituent l'aboutissement de l'apprentissage : ils permettent d'appliquer l'ensemble des concepts théoriques dans des scénarios réels, de développer des compétences de résolution de problèmes, et de construire un portfolio concret de réalisations.

Imaginez ces projets comme les œuvres d'un artisan expérimenté : chaque projet démontre non seulement la maîtrise technique, mais aussi la capacité à concevoir des solutions élégantes, maintenables, et adaptées aux besoins réels des utilisateurs.

## Projet 1 : Serveur web complet

### Objectifs du projet

**Fonctionnalités requises** :
- Serveur web Nginx configuré et sécurisé
- Certificats SSL/TLS avec Let's Encrypt
- Application web déployée (ex: Node.js, Python)
- Base de données (PostgreSQL ou MySQL)
- Monitoring de base
- Sauvegardes automatiques

### Architecture

**Composants** :
```
Internet
   ↓
Nginx (Reverse Proxy + SSL)
   ↓
Application (Node.js/Python)
   ↓
Base de données (PostgreSQL)
```

### Implémentation

**Installation des composants** :
```bash
#!/bin/bash
# setup_web_server.sh

set -euo pipefail

# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation de Nginx
sudo apt install -y nginx

# Installation de PostgreSQL
sudo apt install -y postgresql postgresql-contrib

# Installation de Node.js (exemple)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Installation de Certbot pour SSL
sudo apt install -y certbot python3-certbot-nginx
```

**Configuration Nginx** :
```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name myapp.example.com;
    
    # Redirection vers HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.example.com;
    
    # Certificats SSL
    ssl_certificate /etc/letsencrypt/live/myapp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.example.com/privkey.pem;
    
    # Configuration SSL moderne
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;
    
    # Proxy vers l'application
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
    
    # Fichiers statiques
    location /static {
        alias /var/www/myapp/static;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }
    
    # Logs
    access_log /var/log/nginx/myapp_access.log;
    error_log /var/log/nginx/myapp_error.log;
}
```

**Activation du site** :
```bash
# Créer le lien symbolique
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/

# Tester la configuration
sudo nginx -t

# Recharger Nginx
sudo systemctl reload nginx
```

**Configuration SSL** :
```bash
# Obtenir le certificat
sudo certbot --nginx -d myapp.example.com

# Renouvellement automatique
sudo certbot renew --dry-run

# Ajouter au cron pour renouvellement automatique
# Crontab: 0 0 1 * * certbot renew --quiet
```

**Script de déploiement** :
```bash
#!/bin/bash
# deploy_app.sh

set -euo pipefail

APP_DIR="/var/www/myapp"
GIT_REPO="https://github.com/user/myapp.git"

# Cloner ou mettre à jour
if [ -d "$APP_DIR" ]; then
    cd "$APP_DIR"
    git pull origin main
else
    git clone "$GIT_REPO" "$APP_DIR"
    cd "$APP_DIR"
fi

# Installer les dépendances
npm install --production

# Construire l'application
npm run build

# Redémarrer l'application
sudo systemctl restart myapp

# Vérifier le statut
sudo systemctl status myapp
```

**Service systemd pour l'application** :
```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=MyApp Node.js Application
After=network.target postgresql.service

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/myapp
Environment=NODE_ENV=production
Environment=PORT=3000
ExecStart=/usr/bin/node /var/www/myapp/server.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

## Projet 2 : Système de monitoring

### Objectifs du projet

**Fonctionnalités** :
- Collecte de métriques système
- Alertes automatiques
- Tableaux de bord de visualisation
- Historique des métriques
- Monitoring multi-serveurs

### Architecture Prometheus + Grafana

**Composants** :
```
Prometheus (Collecte métriques)
   ↓
Node Exporter (Métriques système)
   ↓
Grafana (Visualisation)
   ↓
Alertmanager (Alertes)
```

### Implémentation

**Installation Prometheus** :
```bash
#!/bin/bash
# install_prometheus.sh

# Créer utilisateur
sudo useradd --no-create-home --shell /bin/false prometheus

# Télécharger Prometheus
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar -xzf prometheus-2.45.0.linux-amd64.tar.gz
cd prometheus-2.45.0.linux-amd64

# Installer les fichiers
sudo cp prometheus promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus*

# Créer les répertoires
sudo mkdir -p /etc/prometheus
sudo mkdir -p /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus
sudo chown prometheus:prometheus /var/lib/prometheus
```

**Configuration Prometheus** :
```yaml
# /etc/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'nginx'
    static_configs:
      - targets: ['localhost:9113']
```

**Service systemd** :
```ini
# /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus
After=network.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file /etc/prometheus/prometheus.yml \
    --storage.tsdb.path /var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

**Installation Node Exporter** :
```bash
# Télécharger et installer
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.6.1.linux-amd64.tar.gz
sudo cp node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter

# Service systemd
# /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

**Installation Grafana** :
```bash
# Ajouter le dépôt
sudo apt install -y software-properties-common
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Installer
sudo apt update
sudo apt install -y grafana

# Démarrer
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

**Règles d'alerte** :
```yaml
# /etc/prometheus/alert_rules.yml
groups:
  - name: system_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU usage is above 80%"

      - alert: LowDiskSpace
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space is below 10%"
```

## Projet 3 : Automatisation de déploiement

### Objectifs du projet

**Fonctionnalités** :
- Déploiement automatique depuis Git
- Tests automatiques avant déploiement
- Rollback en cas d'erreur
- Notifications de déploiement
- Environnements multiples (dev, staging, prod)

### Architecture CI/CD simple

**Workflow** :
```
Git Push → Webhook → Script de déploiement → Tests → Déploiement → Notification
```

### Implémentation

**Script de déploiement principal** :
```bash
#!/bin/bash
# deploy.sh

set -euo pipefail

ENVIRONMENT="${1:-staging}"
APP_DIR="/var/www/myapp"
BACKUP_DIR="/backups/myapp"
LOG_FILE="/var/log/deploy.log"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

deploy() {
    local env="$1"
    
    log "Début du déploiement en environnement: $env"
    
    # Créer une sauvegarde
    local backup_name="backup_$(date +%Y%m%d_%H%M%S).tar.gz"
    log "Création de la sauvegarde: $backup_name"
    tar -czf "${BACKUP_DIR}/${backup_name}" -C "$APP_DIR" .
    
    # Pull les changements
    cd "$APP_DIR"
    git fetch origin
    git checkout "$env"
    git pull origin "$env"
    
    # Installer les dépendances
    log "Installation des dépendances"
    npm install --production
    
    # Exécuter les tests
    log "Exécution des tests"
    if ! npm test; then
        log "ERREUR: Les tests ont échoué"
        # Rollback
        log "Rollback vers la sauvegarde"
        tar -xzf "${BACKUP_DIR}/${backup_name}" -C "$APP_DIR"
        exit 1
    fi
    
    # Construire l'application
    log "Construction de l'application"
    npm run build
    
    # Redémarrer le service
    log "Redémarrage du service"
    sudo systemctl restart myapp
    
    # Vérifier le déploiement
    sleep 5
    if sudo systemctl is-active --quiet myapp; then
        log "Déploiement réussi"
        notify_success "$env"
    else
        log "ERREUR: Le service n'a pas démarré"
        # Rollback
        tar -xzf "${BACKUP_DIR}/${backup_name}" -C "$APP_DIR"
        sudo systemctl restart myapp
        notify_failure "$env"
        exit 1
    fi
}

notify_success() {
    local env="$1"
    echo "Déploiement réussi en $env" | \
        mail -s "Déploiement $env: Succès" admin@example.com
}

notify_failure() {
    local env="$1"
    echo "Échec du déploiement en $env" | \
        mail -s "Déploiement $env: Échec" admin@example.com
}

deploy "$ENVIRONMENT"
```

**Webhook Git** :
```bash
#!/bin/bash
# git_webhook.sh

# Écouter sur un port
# Utiliser nginx pour proxy vers ce script

read -r payload

# Parser le payload JSON (nécessite jq)
BRANCH=$(echo "$payload" | jq -r '.ref' | sed 's/refs\/heads\///')

if [ "$BRANCH" = "main" ]; then
    /usr/local/bin/deploy.sh production
elif [ "$BRANCH" = "develop" ]; then
    /usr/local/bin/deploy.sh staging
fi
```

## Projet 4 : Solution de backup

### Objectifs du projet

**Fonctionnalités** :
- Sauvegardes automatiques multi-niveaux
- Rotation intelligente des sauvegardes
- Vérification d'intégrité
- Sauvegarde distante (rsync, cloud)
- Restauration facile

### Architecture de sauvegarde

**Stratégie 3-2-1** :
- 3 copies des données
- 2 types de stockage différents
- 1 copie hors site

### Implémentation

**Script de sauvegarde complet** :
```bash
#!/bin/bash
# comprehensive_backup.sh

set -euo pipefail

# Configuration
BACKUP_ROOT="/backups"
REMOTE_HOST="backup-server.example.com"
REMOTE_USER="backup"
RETENTION_DAILY=7
RETENTION_WEEKLY=4
RETENTION_MONTHLY=12

# Sources à sauvegarder
SOURCES=(
    "/home"
    "/etc"
    "/var/www"
    "/opt"
)

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a /var/log/backup.log
}

create_backup() {
    local source="$1"
    local dest="$2"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_file="${dest}/backup_$(basename "$source")_${timestamp}.tar.gz"
    
    log "Sauvegarde de $source vers $backup_file"
    
    # Créer la sauvegarde avec vérification
    if tar -czf "$backup_file" -C "$(dirname "$source")" "$(basename "$source")" && \
       tar -tzf "$backup_file" > /dev/null; then
        log "Sauvegarde réussie: $backup_file"
        
        # Créer le checksum
        sha256sum "$backup_file" > "${backup_file}.sha256"
        
        echo "$backup_file"
    else
        log "ERREUR: Échec de la sauvegarde de $source"
        return 1
    fi
}

rotate_backups() {
    local backup_dir="$1"
    local retention="$2"
    
    log "Rotation des sauvegardes dans $backup_dir (rétention: $retention jours)"
    
    find "$backup_dir" -name "backup_*.tar.gz" -mtime +$retention -delete
    find "$backup_dir" -name "backup_*.sha256" -mtime +$retention -delete
}

sync_to_remote() {
    local local_backup="$1"
    local remote_path="/backups/$(hostname)/$(date +%Y/%m)"
    
    log "Synchronisation vers $REMOTE_HOST:$remote_path"
    
    # Créer le répertoire distant
    ssh "$REMOTE_USER@$REMOTE_HOST" "mkdir -p $remote_path"
    
    # Synchroniser
    rsync -avz --progress "$local_backup" \
        "$REMOTE_USER@$REMOTE_HOST:$remote_path/"
    rsync -avz "${local_backup}.sha256" \
        "$REMOTE_USER@$REMOTE_HOST:$remote_path/"
}

main() {
    local backup_type="${1:-daily}"
    local backup_dir="${BACKUP_ROOT}/${backup_type}"
    
    mkdir -p "$backup_dir"
    
    log "Début de la sauvegarde de type: $backup_type"
    
    # Sauvegarder chaque source
    for source in "${SOURCES[@]}"; do
        if [ -e "$source" ]; then
            create_backup "$source" "$backup_dir"
            sync_to_remote "$(ls -t "${backup_dir}/backup_$(basename "$source")_"*.tar.gz | head -1)"
        else
            log "AVERTISSEMENT: $source n'existe pas"
        fi
    done
    
    # Rotation
    rotate_backups "$backup_dir" "$RETENTION_DAILY"
    
    log "Sauvegarde terminée"
}

main "${1:-daily}"
```

**Planification avec cron** :
```bash
# Crontab
# Sauvegarde quotidienne à 2h du matin
0 2 * * * /usr/local/bin/comprehensive_backup.sh daily

# Sauvegarde hebdomadaire le dimanche à 3h
0 3 * * 0 /usr/local/bin/comprehensive_backup.sh weekly

# Sauvegarde mensuelle le 1er du mois à 4h
0 4 1 * * /usr/local/bin/comprehensive_backup.sh monthly
```

## Projet 5 : Cluster de calcul

### Objectifs du projet

**Fonctionnalités** :
- Plusieurs nœuds de calcul
- Répartition de charge
- Gestion centralisée
- Monitoring des performances
- File d'attente de tâches

### Architecture simple

**Composants** :
```
Master Node (Orchestrateur)
   ↓
Worker Nodes (Calcul)
   ↓
Shared Storage (NFS)
```

### Implémentation

**Configuration NFS** :
```bash
# Sur le serveur NFS
sudo apt install nfs-kernel-server

# Exporter le répertoire
echo "/shared *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server

# Sur les clients
sudo apt install nfs-common
sudo mkdir -p /mnt/shared
sudo mount nfs-server:/shared /mnt/shared

# Montage automatique
echo "nfs-server:/shared /mnt/shared nfs defaults 0 0" | sudo tee -a /etc/fstab
```

**Script de distribution de tâches** :
```bash
#!/bin/bash
# distribute_tasks.sh

WORKERS=(
    "worker1.example.com"
    "worker2.example.com"
    "worker3.example.com"
)

TASKS_DIR="/mnt/shared/tasks"
RESULTS_DIR="/mnt/shared/results"

distribute_task() {
    local task_file="$1"
    local worker="$2"
    
    echo "Distribution de $task_file vers $worker"
    
    # Copier la tâche vers le worker
    scp "$task_file" "$worker:/tmp/task.sh"
    
    # Exécuter sur le worker
    ssh "$worker" "bash /tmp/task.sh > /tmp/result.txt 2>&1 && \
                   scp /tmp/result.txt master:${RESULTS_DIR}/result_$(basename "$task_file").txt"
}

# Distribuer les tâches
task_count=0
for task in "$TASKS_DIR"/*.sh; do
    worker="${WORKERS[$((task_count % ${#WORKERS[@]}))]}"
    distribute_task "$task" "$worker" &
    ((task_count++))
done

wait
echo "Toutes les tâches ont été distribuées"
```

## Méthodologie de projet

### Phases de développement

**1. Planification** :
- Définir les objectifs
- Identifier les contraintes
- Choisir les technologies
- Estimer le temps

**2. Conception** :
- Architecture système
- Schémas de données
- Flux de données
- Interfaces

**3. Implémentation** :
- Développement itératif
- Tests continus
- Documentation en cours
- Révision de code

**4. Déploiement** :
- Environnement de test
- Déploiement progressif
- Monitoring
- Rollback planifié

**5. Maintenance** :
- Monitoring continu
- Mises à jour régulières
- Optimisation
- Documentation

### Bonnes pratiques

**Documentation** :
```bash
# README.md pour chaque projet
# - Objectifs
# - Architecture
# - Installation
# - Configuration
# - Utilisation
# - Dépannage
```

**Versionnement** :
```bash
# Utiliser Git pour tout
git init
git add .
git commit -m "Initial commit"

# Tags pour les versions
git tag -a v1.0.0 -m "Version initiale"
```

**Tests** :
```bash
# Scripts de test
#!/bin/bash
# test_deployment.sh

# Vérifier les services
systemctl is-active nginx || exit 1
systemctl is-active myapp || exit 1

# Vérifier la connectivité
curl -f http://localhost/health || exit 1

echo "Tous les tests passent"
```

## Évaluation et optimisation

### Métriques de performance

**Monitoring des ressources** :
```bash
# Script de monitoring
#!/bin/bash
# monitor_resources.sh

echo "=== CPU ==="
top -bn1 | grep "Cpu(s)" | awk '{print $2}'

echo "=== Mémoire ==="
free -h

echo "=== Disque ==="
df -h

echo "=== Réseau ==="
ifstat -t 1 1
```

**Optimisation** :
```bash
# Optimiser les performances
# - Cache des requêtes fréquentes
# - Compression des réponses
# - Mise en cache des fichiers statiques
# - Optimisation des requêtes base de données
```

## Conclusion

Ces projets pratiques synthétisent toutes les compétences acquises dans la Partie 2, démontrant la capacité à concevoir, implémenter et maintenir des systèmes Linux complexes. Chaque projet représente un défi réel qui combine plusieurs domaines : administration système, réseau, sécurité, automatisation et monitoring.

La réussite de ces projets ne vient pas seulement de la maîtrise technique, mais aussi de la capacité à penser en systèmes, à anticiper les problèmes, et à créer des solutions robustes et maintenables. Ces compétences sont essentielles pour tout administrateur système moderne et tout professionnel DevOps.

Félicitations ! Vous avez complété la Partie 2 de Shellcraft Ultimate 300. Vous êtes maintenant prêt à explorer la Partie 3 : Bash et Shell avancé, où nous approfondirons les techniques de scripting avancées et l'automatisation complexe.