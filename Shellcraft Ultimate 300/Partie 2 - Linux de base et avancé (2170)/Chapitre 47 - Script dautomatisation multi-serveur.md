# Chapitre 47 - Script d'automatisation multi-serveur

## Table des matières
- [Introduction](#introduction)
- [Principes de l'automatisation multi-serveur](#principes-de-lautomatisation-multi-serveur)
- [Architecture des scripts multi-serveur](#architecture-des-scripts-multi-serveur)
- [Outils d'orchestration](#outils-dorchestration)
- [pssh : exécution parallèle SSH](#pssh--exécution-parallèle-ssh)
- [Scripts parallèles avec Bash](#scripts-parallèles-avec-bash)
- [Gestion des erreurs](#gestion-des-erreurs)
- [Configuration centralisée](#configuration-centralisée)
- [Déploiement distribué](#déploiement-distribué)
- [Monitoring et rapports](#monitoring-et-rapports)
- [Stratégies d'exécution](#stratégies-dexécution)
- [Sécurité et bonnes pratiques](#sécurité-et-bonnes-pratiques)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

L'automatisation multi-serveur constitue l'évolution naturelle de l'administration système moderne. Au lieu de gérer chaque serveur individuellement, il s'agit de créer des orchestrations qui déploient, configurent, et maintiennent des flottes de serveurs de manière coordonnée et fiable.

Imaginez l'automatisation multi-serveur comme un chef d'orchestre dirigeant un grand ensemble : chaque instrument (serveur) joue sa partition, mais l'harmonie globale dépend de la coordination parfaite et du timing précis du chef d'orchestre (le script d'automatisation). Dans un environnement moderne où les infrastructures comptent des dizaines ou des centaines de serveurs, l'automatisation multi-serveur n'est plus un luxe mais une nécessité.

## Principes de l'automatisation multi-serveur

### Concepts fondamentaux

**Principes essentiels** :
```bash
#!/bin/bash
# Principes de l'automatisation multi-serveur

# 1. Idempotence
# Les opérations doivent pouvoir être répétées sans effet secondaire

# 2. Parallélisation
# Exécuter les opérations en parallèle pour gagner du temps

# 3. Gestion d'erreurs
# Gérer les échecs de manière gracieuse

# 4. Configuration centralisée
# Une seule source de vérité pour la configuration

# 5. Reporting
# Traçabilité et rapports des opérations
```

**Avantages** :
- Gain de temps considérable
- Cohérence entre serveurs
- Réduction des erreurs humaines
- Scalabilité
- Traçabilité complète

## Architecture des scripts multi-serveur

### Composants essentiels

**Composants** :
```bash
#!/bin/bash
# Composants d'un système d'automatisation multi-serveur

# 1. Inventaire des serveurs
# Liste des serveurs à gérer

# 2. Gestionnaire de connexions
# Gestion des connexions SSH

# 3. Moteur d'exécution
# Exécution des commandes sur les serveurs

# 4. Gestionnaire d'erreurs
# Gestion des échecs

# 5. Système de reporting
# Rapports et logs
```

## Outils d'orchestration

### Outils disponibles

**Outils principaux** :
```bash
#!/bin/bash
# Outils d'orchestration multi-serveur

# pssh : Exécution parallèle SSH
# ansible : Orchestration complète
# fabric : Déploiement Python
# capistrano : Déploiement Ruby
# mcollective : Orchestration Puppet
# salt : Orchestration SaltStack
```

## pssh : exécution parallèle SSH

### Installation et utilisation

**Installation pssh** :
```bash
#!/bin/bash
# Installation pssh

# Debian/Ubuntu
sudo apt install pssh

# CentOS/RHEL
sudo yum install pssh

# macOS
brew install pssh
```

**Utilisation de base** :
```bash
#!/bin/bash
# Utilisation de base pssh

# Créer un fichier d'hôtes
cat > hosts.txt << EOF
server1.example.com
server2.example.com
server3.example.com
EOF

# Exécuter une commande sur tous les serveurs
pssh -h hosts.txt -i "uptime"

# Exécuter avec utilisateur spécifique
pssh -h hosts.txt -l user -i "whoami"

# Exécuter avec clé SSH spécifique
pssh -h hosts.txt -i -x "-i ~/.ssh/id_rsa" "hostname"
```

### Commandes pssh

**Commandes pssh** :
```bash
#!/bin/bash
# Commandes pssh

# pssh : Exécution parallèle
pssh -h hosts.txt -i "command"

# pscp : Copie parallèle
pscp -h hosts.txt local_file remote_path

# prsync : Synchronisation parallèle
prsync -h hosts.txt -r local_dir remote_dir

# pslurp : Récupération parallèle
pslurp -h hosts.txt remote_file local_dir

# pnuke : Arrêt de processus parallèle
pnuke -h hosts.txt process_name
```

**Options pssh** :
```bash
#!/bin/bash
# Options pssh

# -h : Fichier d'hôtes
pssh -h hosts.txt -i "command"

# -H : Hôte spécifique
pssh -H "server1.example.com" -i "command"

# -l : Utilisateur
pssh -h hosts.txt -l user -i "command"

# -i : Mode interactif (afficher sortie)
pssh -h hosts.txt -i "command"

# -o : Répertoire de sortie
pssh -h hosts.txt -o /tmp/output -i "command"

# -e : Répertoire d'erreurs
pssh -h hosts.txt -e /tmp/errors -i "command"

# -t : Timeout
pssh -h hosts.txt -t 30 -i "command"

# -p : Nombre de connexions parallèles
pssh -h hosts.txt -p 10 -i "command"

# -x : Options SSH supplémentaires
pssh -h hosts.txt -x "-o StrictHostKeyChecking=no" -i "command"
```

## Scripts parallèles avec Bash

### Exécution parallèle de base

**Parallélisation simple** :
```bash
#!/bin/bash
# Exécution parallèle simple

SERVERS=("server1.example.com" "server2.example.com" "server3.example.com")

# Exécution parallèle
for server in "${SERVERS[@]}"; do
    (
        echo "Exécution sur $server..."
        ssh "$server" "command"
        echo "✓ $server terminé"
    ) &
done

# Attendre la fin de tous les processus
wait

echo "Toutes les exécutions terminées"
```

### Gestion des résultats

**Collecte des résultats** :
```bash
#!/bin/bash
# Collecte des résultats

SERVERS=("server1" "server2" "server3")
declare -A RESULTS

for server in "${SERVERS[@]}"; do
    (
        if ssh "$server" "command"; then
            RESULTS["$server"]="success"
        else
            RESULTS["$server"]="failed"
        fi
    ) &
done

wait

# Afficher les résultats
for server in "${SERVERS[@]}"; do
    echo "$server: ${RESULTS[$server]}"
done
```

### Limitation du parallélisme

**Contrôle du parallélisme** :
```bash
#!/bin/bash
# Contrôle du parallélisme

SERVERS=("server1" "server2" "server3" "server4" "server5")
MAX_PARALLEL=3
RUNNING=0

for server in "${SERVERS[@]}"; do
    # Attendre si trop de processus en cours
    while [ $RUNNING -ge $MAX_PARALLEL ]; do
        sleep 0.1
        RUNNING=$(jobs -r | wc -l)
    done
    
    (
        ssh "$server" "command"
    ) &
    
    RUNNING=$((RUNNING + 1))
done

wait
```

## Gestion des erreurs

### Gestion d'erreurs robuste

**Gestion d'erreurs avancée** :
```bash
#!/bin/bash
# Gestion d'erreurs robuste

set -euo pipefail

execute_on_server() {
    local server="$1"
    local command="$2"
    local max_retries="${3:-3}"
    local retry=0
    
    while [ $retry -lt $max_retries ]; do
        if ssh "$server" "$command"; then
            return 0
        else
            retry=$((retry + 1))
            echo "Tentative $retry/$max_retries échouée sur $server"
            sleep 2
        fi
    done
    
    echo "ERREUR: Échec après $max_retries tentatives sur $server"
    return 1
}

# Exécution avec gestion d'erreurs
SERVERS=("server1" "server2" "server3")
FAILED_SERVERS=()

for server in "${SERVERS[@]}"; do
    if ! execute_on_server "$server" "command"; then
        FAILED_SERVERS+=("$server")
    fi
done

if [ ${#FAILED_SERVERS[@]} -gt 0 ]; then
    echo "Serveurs en échec : ${FAILED_SERVERS[*]}"
    exit 1
fi
```

## Configuration centralisée

### Fichier de configuration

**Configuration centralisée** :
```bash
#!/bin/bash
# Configuration centralisée

# Fichier de configuration
CONFIG_FILE="/etc/multi-server.conf"

load_config() {
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
    else
        echo "Fichier de configuration non trouvé"
        exit 1
    fi
}

# Exemple de fichier de configuration
create_config_example() {
    cat > "$CONFIG_FILE" << 'EOF'
# Serveurs
SERVERS=("server1.example.com" "server2.example.com" "server3.example.com")

# Utilisateur SSH
SSH_USER="deploy"

# Clé SSH
SSH_KEY="~/.ssh/deploy_key"

# Timeout
SSH_TIMEOUT=30

# Nombre de connexions parallèles
MAX_PARALLEL=5
EOF
}
```

### Inventaire dynamique

**Inventaire dynamique** :
```bash
#!/bin/bash
# Inventaire dynamique

# Depuis un fichier
load_servers_from_file() {
    local file="$1"
    SERVERS=()
    
    while IFS= read -r line; do
        # Ignorer les commentaires et lignes vides
        [[ "$line" =~ ^#.*$ ]] && continue
        [[ -z "$line" ]] && continue
        
        SERVERS+=("$line")
    done < "$file"
}

# Depuis DNS
load_servers_from_dns() {
    local domain="$1"
    
    SERVERS=()
    for ip in $(dig +short "$domain" | grep -E '^[0-9]'); do
        SERVERS+=("$ip")
    done
}

# Depuis un service de découverte
load_servers_from_service() {
    local service_url="$1"
    
    SERVERS=($(curl -s "$service_url" | jq -r '.[]'))
}
```

## Déploiement distribué

### Déploiement avec rsync

**Déploiement rsync parallèle** :
```bash
#!/bin/bash
# Déploiement distribué avec rsync

deploy_to_servers() {
    local source_dir="$1"
    local remote_dir="$2"
    local servers=("${@:3}")
    
    for server in "${servers[@]}"; do
        (
            echo "Déploiement sur $server..."
            
            # Synchroniser les fichiers
            rsync -avz --delete \
                --exclude='.git' \
                --exclude='node_modules' \
                "$source_dir/" "$server:$remote_dir/"
            
            # Exécuter les commandes post-déploiement
            ssh "$server" "cd $remote_dir && ./deploy.sh"
            
            echo "✓ $server déployé"
        ) &
    done
    
    wait
    echo "Tous les déploiements terminés"
}
```

### Déploiement avec tar

**Déploiement tar** :
```bash
#!/bin/bash
# Déploiement avec tar

deploy_with_tar() {
    local source_dir="$1"
    local remote_dir="$2"
    local servers=("${@:3}")
    
    # Créer l'archive
    local archive="/tmp/deploy_$(date +%Y%m%d_%H%M%S).tar.gz"
    tar -czf "$archive" -C "$source_dir" .
    
    # Déployer sur tous les serveurs
    for server in "${servers[@]}"; do
        (
            echo "Déploiement sur $server..."
            
            # Copier l'archive
            scp "$archive" "$server:/tmp/"
            
            # Extraire et déployer
            ssh "$server" "
                mkdir -p $remote_dir
                tar -xzf /tmp/$(basename $archive) -C $remote_dir
                cd $remote_dir && ./deploy.sh
                rm /tmp/$(basename $archive)
            "
            
            echo "✓ $server déployé"
        ) &
    done
    
    wait
    rm -f "$archive"
}
```

## Monitoring et rapports

### Collecte de rapports

**Système de reporting** :
```bash
#!/bin/bash
# Système de reporting

execute_with_reporting() {
    local server="$1"
    local command="$2"
    local report_dir="${3:-/tmp/reports}"
    
    mkdir -p "$report_dir"
    
    local report_file="${report_dir}/${server}_$(date +%Y%m%d_%H%M%S).log"
    
    if ssh "$server" "$command" > "$report_file" 2>&1; then
        echo "SUCCESS:$server" >> "${report_dir}/summary.log"
        return 0
    else
        echo "FAILED:$server" >> "${report_dir}/summary.log"
        return 1
    fi
}

# Générer un rapport consolidé
generate_consolidated_report() {
    local report_dir="$1"
    local output_file="${2:-consolidated_report.txt}"
    
    {
        echo "=== Rapport consolidé ==="
        echo "Date: $(date)"
        echo ""
        echo "=== Résumé ==="
        grep -c "SUCCESS" "${report_dir}/summary.log" | xargs echo "Succès:"
        grep -c "FAILED" "${report_dir}/summary.log" | xargs echo "Échecs:"
        echo ""
        echo "=== Détails ==="
        cat "${report_dir}/summary.log"
    } > "$output_file"
}
```

## Stratégies d'exécution

### Exécution séquentielle

**Exécution séquentielle** :
```bash
#!/bin/bash
# Exécution séquentielle

execute_sequential() {
    local servers=("$@")
    
    for server in "${servers[@]}"; do
        echo "Exécution sur $server..."
        ssh "$server" "command"
        
        if [ $? -ne 0 ]; then
            echo "ERREUR sur $server, arrêt"
            return 1
        fi
    done
}

# Avantage : Contrôle total, arrêt sur erreur
# Inconvénient : Lent
```

### Exécution parallèle

**Exécution parallèle** :
```bash
#!/bin/bash
# Exécution parallèle

execute_parallel() {
    local servers=("$@")
    local pids=()
    
    for server in "${servers[@]}"; do
        ssh "$server" "command" &
        pids+=($!)
    done
    
    # Attendre tous les processus
    local failed=0
    for pid in "${pids[@]}"; do
        wait "$pid" || failed=$((failed + 1))
    done
    
    return $failed
}

# Avantage : Rapide
# Inconvénient : Moins de contrôle
```

### Exécution en rolling

**Exécution rolling (déploiement progressif)** :
```bash
#!/bin/bash
# Exécution rolling

execute_rolling() {
    local servers=("$@")
    local batch_size="${2:-2}"
    
    local i=0
    while [ $i -lt ${#servers[@]} ]; do
        local batch=()
        
        # Créer un batch
        for ((j=0; j<batch_size && i<${#servers[@]}; j++)); do
            batch+=("${servers[$i]}")
            i=$((i + 1))
        done
        
        # Exécuter le batch
        echo "Exécution du batch : ${batch[*]}"
        execute_parallel "${batch[@]}"
        
        # Attendre entre les batches
        sleep 5
    done
}

# Avantage : Équilibre entre vitesse et contrôle
# Utile pour les déploiements sans interruption de service
```

## Sécurité et bonnes pratiques

### Bonnes pratiques

**Recommandations** :
```bash
#!/bin/bash
# Bonnes pratiques

# 1. Utiliser des clés SSH au lieu de mots de passe
# 2. Limiter les permissions SSH
# 3. Utiliser des utilisateurs dédiés pour l'automatisation
# 4. Logger toutes les opérations
# 5. Vérifier les serveurs avant exécution
# 6. Utiliser des timeouts
# 7. Valider les configurations avant déploiement
```

### Sécurité SSH

**Configuration SSH sécurisée** :
```bash
#!/bin/bash
# Configuration SSH sécurisée pour automatisation

# Utiliser des clés avec restrictions
# Dans ~/.ssh/authorized_keys sur le serveur
restrict_automation_key() {
    cat >> ~/.ssh/authorized_keys << 'EOF'
command="/usr/local/bin/automation-script.sh",no-port-forwarding,no-X11-forwarding ssh-ed25519 AAAAC3...
EOF
}
```

## Scripts d'automatisation

### Framework d'automatisation complet

**Framework complet** :
```bash
#!/bin/bash
# Framework d'automatisation multi-serveur

set -euo pipefail

# Configuration
CONFIG_FILE="${CONFIG_FILE:-/etc/multi-server.conf}"
LOG_FILE="${LOG_FILE:-/var/log/multi-server.log}"

# Charger la configuration
load_config() {
    if [ -f "$CONFIG_FILE" ]; then
        source "$CONFIG_FILE"
    fi
}

# Logger
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" | tee -a "$LOG_FILE"
}

# Exécuter sur un serveur
execute_on_server() {
    local server="$1"
    local command="$2"
    local timeout="${3:-30}"
    
    log "Exécution sur $server : $command"
    
    if timeout "$timeout" ssh "$server" "$command"; then
        log "✓ Succès sur $server"
        return 0
    else
        log "✗ Échec sur $server"
        return 1
    fi
}

# Exécuter sur tous les serveurs
execute_on_all() {
    local command="$1"
    local strategy="${2:-parallel}"
    local servers=("${@:3}")
    
    case "$strategy" in
        sequential)
            execute_sequential "${servers[@]}"
            ;;
        parallel)
            execute_parallel "${servers[@]}"
            ;;
        rolling)
            execute_rolling "${servers[@]}"
            ;;
        *)
            echo "Stratégie inconnue : $strategy"
            return 1
            ;;
    esac
}

# Déployer sur tous les serveurs
deploy_to_all() {
    local source_dir="$1"
    local remote_dir="$2"
    local servers=("${@:3}")
    
    log "Déploiement de $source_dir vers ${#servers[@]} serveurs"
    
    deploy_to_servers "$source_dir" "$remote_dir" "${servers[@]}"
    
    log "Déploiement terminé"
}

# Utilisation
# load_config
# execute_on_all "uptime" parallel "${SERVERS[@]}"
# deploy_to_all "./app" "/var/www/app" "${SERVERS[@]}"
```

## Conclusion

L'automatisation multi-serveur est essentielle pour gérer efficacement les infrastructures modernes. En utilisant des outils comme pssh, en créant des scripts parallèles robustes, et en implémentant une gestion d'erreurs appropriée, vous pouvez orchestrer des opérations complexes sur des dizaines ou des centaines de serveurs de manière fiable et efficace.

Un système bien automatisé utilise plusieurs stratégies d'exécution selon le contexte : parallèle pour la vitesse, séquentielle pour le contrôle, et rolling pour les déploiements sans interruption. La compréhension approfondie de ces mécanismes permet de créer des systèmes d'automatisation qui s'adaptent aux besoins spécifiques de chaque environnement.

Dans le chapitre suivant, nous explorerons la gestion avancée des logs, découvrant comment centraliser, analyser, et monitorer les logs de multiples serveurs pour maintenir la visibilité sur l'ensemble de l'infrastructure.
