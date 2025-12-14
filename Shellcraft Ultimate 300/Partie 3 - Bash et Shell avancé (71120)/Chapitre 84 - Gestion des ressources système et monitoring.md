# Chapitre 84 - Gestion des ressources système et monitoring

> "Dans un système complexe, la surveillance n'est pas une option : c'est une nécessité." - Anonyme

## Introduction : Gardien vigilant du système

Imaginez-vous en tant que chef d'orchestre d'un immense opéra numérique. Chaque instrument (CPU, mémoire, disque, réseau) doit jouer sa partition à la perfection, sans accaprer la scène ni laisser les autres dans l'ombre. La gestion des ressources système en Bash, c'est exactement cela : surveiller, contrôler, et orchestrer les composants pour une harmonie parfaite.

Dans ce chapitre, nous allons explorer les techniques avancées de monitoring système et de gestion des ressources : des limites `ulimit` aux contrôleurs cgroups, en passant par l'intégration systemd et les stratégies de supervision proactive.

## Section 1 : Les bases du monitoring système

### 1.1 Collecte des métriques essentielles

Les indicateurs vitaux du système :

```bash
#!/bin/bash

# Collecte des métriques système essentielles
echo "=== Métriques système essentielles ==="

# Fonction de collecte CPU
get_cpu_metrics() {
    echo "--- Métriques CPU ---"
    
    # Charge système (load average)
    local load
    load=$(uptime | awk -F'load average:' '{ print $2 }' | sed 's/,//g')
    echo "Charge système: $load"
    
    # Utilisation CPU détaillée
    if command -v mpstat >/dev/null 2>&1; then
        echo "Utilisation CPU détaillée:"
        mpstat 1 1 | tail -1
    else
        # Fallback avec top
        local cpu_idle
        cpu_idle=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/")
        local cpu_usage=$(echo "100 - $cpu_idle" | bc)
        echo "Utilisation CPU: ${cpu_usage}%"
    fi
    
    # Nombre de cœurs
    local cores
    cores=$(nproc 2>/dev/null || grep -c '^processor' /proc/cpuinfo 2>/dev/null || echo "N/A")
    echo "Cœurs disponibles: $cores"
}

# Fonction de collecte mémoire
get_memory_metrics() {
    echo "--- Métriques mémoire ---"
    
    if [[ -f /proc/meminfo ]]; then
        # Mémoire totale
        local mem_total
        mem_total=$(grep "MemTotal" /proc/meminfo | awk '{print $2}')
        echo "Mémoire totale: $((mem_total / 1024)) MB"
        
        # Mémoire disponible
        local mem_available
        mem_available=$(grep "MemAvailable" /proc/meminfo | awk '{print $2}')
        echo "Mémoire disponible: $((mem_available / 1024)) MB"
        
        # Utilisation
        local mem_used=$((mem_total - mem_available))
        local mem_usage_percent=$((mem_used * 100 / mem_total))
        echo "Utilisation mémoire: ${mem_usage_percent}%"
        
        # Swap
        local swap_total
        local swap_free
        swap_total=$(grep "SwapTotal" /proc/meminfo | awk '{print $2}')
        swap_free=$(grep "SwapFree" /proc/meminfo | awk '{print $2}')
        
        if (( swap_total > 0 )); then
            local swap_used=$((swap_total - swap_free))
            local swap_usage=$((swap_used * 100 / swap_total))
            echo "Utilisation swap: ${swap_usage}%"
        fi
    else
        # Fallback avec free
        free -h
    fi
}

# Fonction de collecte disque
get_disk_metrics() {
    echo "--- Métriques disque ---"
    
    echo "Utilisation des systèmes de fichiers:"
    df -h | grep -E '^/dev/' | while read -r fs size used avail use mount; do
        echo "  $mount: ${use} utilisé (${used}/${size})"
        
        # Alerte si utilisation > 85%
        local usage_percent
        usage_percent=$(echo "$use" | sed 's/%//')
        if (( usage_percent > 85 )); then
            echo "    ⚠️  ALERTE: Utilisation élevée!"
        fi
    done
    
    # I/O disque (si iostat disponible)
    if command -v iostat >/dev/null 2>&1; then
        echo "I/O disque (dernières 5 secondes):"
        iostat -x 5 1 | tail -5
    fi
}

# Fonction de collecte réseau
get_network_metrics() {
    echo "--- Métriques réseau ---"
    
    # Interfaces réseau
    echo "Interfaces réseau:"
    ip addr show 2>/dev/null | grep -E '^[0-9]+:' | while read -r line; do
        local iface
        iface=$(echo "$line" | awk '{print $2}' | sed 's/://')
        echo "  $iface: $(echo "$line" | grep -o 'state [A-Z]*' | cut -d' ' -f2 || echo 'UNKNOWN')"
    done
    
    # Statistiques réseau
    if command -v ss >/dev/null 2>&1; then
        local connections
        connections=$(ss -tun | wc -l)
        echo "Connexions réseau actives: $((connections - 1))"  # -1 pour l'en-tête
    fi
    
    # Trafic réseau (si vnstat disponible)
    if command -v vnstat >/dev/null 2>&1; then
        echo "Trafic réseau (24h):"
        vnstat -d 1 2>/dev/null | tail -1 || echo "  Données non disponibles"
    fi
}

# Collecte complète
get_cpu_metrics
echo
get_memory_metrics
echo
get_disk_metrics
echo
get_network_metrics
```

### 1.2 Monitoring en temps réel

Surveillance continue avec mise à jour périodique :

```bash
#!/bin/bash

# Monitoring en temps réel
echo "=== Monitoring en temps réel ==="

# Fonction de surveillance continue
real_time_monitor() {
    local interval="${1:-5}"
    local duration="${2:-60}"
    
    echo "Monitoring en temps réel - Intervalle: ${interval}s, Durée: ${duration}s"
    echo "Appuyez sur Ctrl+C pour arrêter"
    echo
    
    local start_time=$(date +%s)
    local count=0
    
    # En-têtes
    printf "%-8s %-8s %-8s %-8s %-8s %-8s\n" "Temps" "CPU%" "RAM%" "SWAP%" "DISK%" "LOAD"
    printf "%-8s %-8s %-8s %-8s %-8s %-8s\n" "--------" "--------" "--------" "--------" "--------" "--------"
    
    while (( $(date +%s) - start_time < duration )); do
        # Collecte des métriques
        local cpu_usage
        local mem_usage
        local swap_usage
        local disk_usage
        local load_avg
        
        # CPU
        cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}' | xargs printf "%.0f")
        
        # Mémoire
        mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
        
        # Swap
        swap_usage=$(free | grep Swap | awk '{if ($2 > 0) printf "%.0f", $3/$2 * 100.0; else print "0"}')
        
        # Disque (partition root)
        disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
        
        # Charge système
        load_avg=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)
        
        # Affichage
        printf "%-8s %-8s %-8s %-8s %-8s %-8s\n" \
               "$(date +%H:%M:%S)" \
               "${cpu_usage}%" \
               "${mem_usage}%" \
               "${swap_usage}%" \
               "${disk_usage}%" \
               "$load_avg"
        
        ((count++))
        sleep "$interval"
    done
    
    echo
    echo "Monitoring terminé - $count mesures collectées"
}

# Fonction de surveillance avec seuils d'alerte
monitor_with_alerts() {
    local interval="${1:-10}"
    
    echo "Monitoring avec alertes - Intervalle: ${interval}s"
    echo "Seuls les événements notables seront affichés"
    echo
    
    local cpu_threshold=80
    local mem_threshold=85
    local disk_threshold=90
    
    while true; do
        local alerts=()
        
        # Vérification CPU
        local cpu_usage
        cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
        if (( $(echo "$cpu_usage > $cpu_threshold" | bc -l) )); then
            alerts+=("CPU: ${cpu_usage}% (seuil: ${cpu_threshold}%)")
        fi
        
        # Vérification mémoire
        local mem_usage
        mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
        if (( mem_usage > mem_threshold )); then
            alerts+=("RAM: ${mem_usage}% (seuil: ${mem_threshold}%)")
        fi
        
        # Vérification disque
        local disk_usage
        disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
        if (( disk_usage > disk_threshold )); then
            alerts+=("DISK: ${disk_usage}% (seuil: ${disk_threshold}%)")
        fi
        
        # Affichage des alertes
        if (( ${#alerts[@]} > 0 )); then
            echo "[$(date)] ALERTES:"
            for alert in "${alerts[@]}"; do
                echo "  ⚠️  $alert"
            done
            echo
        fi
        
        sleep "$interval"
    done
}

# Test du monitoring temps réel (court pour la démo)
real_time_monitor 2 10
echo

# Démarrage du monitoring avec alertes (en arrière-plan pour la démo)
monitor_with_alerts 5 &
monitor_pid=$!

# Laisser tourner quelques secondes
sleep 15

# Arrêt du monitoring
kill "$monitor_pid" 2>/dev/null
echo "Monitoring arrêté"
```

## Section 2 : Gestion des limites avec ulimit

### 2.1 Comprendre et configurer ulimit

Contrôler les ressources allouées aux processus :

```bash
#!/bin/bash

# Gestion des limites avec ulimit
echo "=== Gestion des limites avec ulimit ==="

# Fonction d'affichage des limites actuelles
show_limits() {
    echo "--- Limites actuelles ---"
    
    echo "Limites soft (actuelles):"
    ulimit -a | sed 's/^/  /'
    echo
    
    echo "Limites hard (maximales):"
    ulimit -Ha | sed 's/^/  /'
}

# Fonction de configuration des limites
set_limits() {
    local resource="$1"
    local soft_limit="$2"
    local hard_limit="${3:-$soft_limit}"
    
    echo "Configuration de $resource (soft: $soft_limit, hard: $hard_limit)"
    
    case "$resource" in
        cpu)
            ulimit -t "$soft_limit"  # CPU time (seconds)
            ;;
        memory|data)
            ulimit -d "$((soft_limit * 1024 * 1024))"  # Data segment (MB to KB)
            ;;
        stack)
            ulimit -s "$((soft_limit * 1024))"  # Stack size (MB to KB)
            ;;
        files)
            ulimit -n "$soft_limit"  # Open files
            ;;
        processes)
            ulimit -u "$soft_limit"  # Max user processes
            ;;
        *)
            echo "Ressource inconnue: $resource"
            return 1
            ;;
    esac
}

# Fonction de test des limites
test_limits() {
    local test_type="$1"
    
    echo "--- Test des limites: $test_type ---"
    
    case "$test_type" in
        memory)
            echo "Test de consommation mémoire..."
            local -a big_array
            
            # Tenter de consommer beaucoup de mémoire
            for ((i=0; i<100000; i++)); do
                big_array+=("Élément $i avec du contenu pour consommer de la mémoire")
                
                # Vérifier si on approche des limites
                if (( i % 10000 == 0 )); then
                    local mem_usage
                    mem_usage=$(ps -o rss= $$ 2>/dev/null || echo "0")
                    echo "  Itération $i: ${mem_usage}KB utilisés"
                fi
            done
            ;;
            
        files)
            echo "Test d'ouverture de fichiers..."
            local files=()
            
            for ((i=0; i<100; i++)); do
                local temp_file
                temp_file="/tmp/test_limits_$$_$i.txt"
                echo "Test content $i" > "$temp_file"
                files+=("$temp_file")
                
                if (( i % 10 == 0 )); then
                    echo "  $i fichiers ouverts"
                fi
            done
            
            # Nettoyage
            for file in "${files[@]}"; do
                rm -f "$file"
            done
            ;;
            
        processes)
            echo "Test de création de processus..."
            local pids=()
            
            for ((i=0; i<20; i++)); do
                sleep 60 &  # Processus dormant
                pids+=($!)
                
                echo "  $i processus créés"
                sleep 0.1
            done
            
            # Attendre un peu puis tuer
            sleep 1
            for pid in "${pids[@]}"; do
                kill "$pid" 2>/dev/null
            done
            ;;
    esac
}

# Affichage des limites par défaut
show_limits
echo

# Configuration de limites restrictives pour les tests
echo "Configuration de limites restrictives..."
set_limits "files" "50" "100"
set_limits "processes" "20" "50"
set_limits "memory" "10" "20"  # 10MB soft, 20MB hard

echo
show_limits
echo

# Tests avec limites
test_limits "files"
echo
test_limits "processes"

# Nettoyage
echo "--- Restauration des limites par défaut ---"
ulimit -S -c unlimited  # Core dumps
ulimit -S -d unlimited  # Data segment
ulimit -S -f unlimited  # File size
ulimit -S -l unlimited  # Locked memory
ulimit -S -m unlimited  # Resident set size
ulimit -S -n 1024       # Open files
ulimit -S -p 8          # Pipe size
ulimit -S -q 819200     # POSIX message queues
ulimit -S -r 0          # Real-time priority
ulimit -S -s 8192       # Stack size
ulimit -S -t unlimited  # CPU time
ulimit -S -u 4096       # Max user processes
ulimit -S -v unlimited  # Virtual memory
ulimit -S -x unlimited  # File locks
```

### 2.2 Gestion avancée des limites

Scripts avec auto-limitation et récupération :

```bash
#!/bin/bash

# Gestion avancée des limites
echo "=== Gestion avancée des limites ==="

# Fonction de limitation automatique basée sur la charge système
auto_limit_based_on_load() {
    local max_load="${1:-2.0}"
    
    echo "Auto-limitation basée sur la charge système (max: $max_load)"
    
    while true; do
        # Mesure de la charge actuelle
        local current_load
        current_load=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)
        
        if (( $(echo "$current_load > $max_load" | bc -l) )); then
            echo "[$(date)] Charge élevée détectée ($current_load > $max_load)"
            echo "  Réduction des limites pour protéger le système"
            
            # Réduction des limites
            ulimit -u 50  # Max 50 processus
            ulimit -n 100  # Max 100 fichiers ouverts
            
            echo "  Limites réduites - attente de baisse de charge..."
            
            # Attendre que la charge baisse
            while (( $(echo "$current_load > $max_load" | bc -l) )); do
                sleep 30
                current_load=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)
            done
            
            echo "[$(date)] Charge normalisée ($current_load) - restauration des limites"
            
            # Restauration des limites
            ulimit -u 1024
            ulimit -n 1024
            
        fi
        
        sleep 60  # Vérification chaque minute
    done
}

# Fonction de limitation basée sur l'heure
time_based_limits() {
    echo "Limitation basée sur l'heure de la journée"
    
    while true; do
        local hour
        hour=$(date +%H)
        
        if (( hour >= 9 && hour <= 17 )); then
            # Heures de bureau - limites normales
            ulimit -u 500
            ulimit -n 500
            echo "[$(date)] Heures de bureau - limites normales"
        else
            # Hors heures - limites réduites pour économie
            ulimit -u 100
            ulimit -n 100
            echo "[$(date)] Hors heures - limites réduites"
        fi
        
        sleep 3600  # Vérification chaque heure
    done
}

# Fonction de monitoring des limites
monitor_limits() {
    echo "Monitoring des limites de ressources"
    echo "PID: $$"
    echo
    
    while true; do
        echo "[$(date)] === État des limites ==="
        
        # Limites actuelles
        echo "Limites soft:"
        ulimit -a | head -10
        
        # Ressources utilisées
        echo "Ressources utilisées:"
        if [[ -f /proc/$$/status ]]; then
            grep -E "VmSize|VmRSS|Threads" /proc/$$/status | sed 's/^/  /'
        fi
        
        # Nombre de fichiers ouverts
        local open_files
        open_files=$(lsof -p $$ 2>/dev/null | wc -l)
        echo "  Fichiers ouverts: $open_files"
        
        echo
        sleep 300  # Toutes les 5 minutes
    done
}

# Fonction de limitation avec récupération automatique
safe_execute_with_limits() {
    local soft_limit="$1"
    local hard_limit="$2"
    shift 2
    local command="$*"
    
    echo "Exécution avec limites (soft: $soft_limit, hard: $hard_limit)"
    echo "Commande: $command"
    
    # Sauvegarde des limites actuelles
    local saved_limits
    saved_limits=$(ulimit -a)
    
    # Configuration des limites
    (
        ulimit -v "$((soft_limit * 1024 * 1024))"  # Mémoire virtuelle en KB
        ulimit -t "$hard_limit"  # Temps CPU en secondes
        
        echo "Limites configurées pour le sous-shell"
        ulimit -v -t
        
        # Exécution de la commande
        if timeout "$hard_limit" bash -c "$command"; then
            echo "Commande exécutée avec succès"
        else
            echo "Commande interrompue (limite atteinte)"
        fi
    )
    
    # Restauration implicite à la sortie du sous-shell
    echo "Retour aux limites normales"
}

# Tests des fonctionnalités avancées
echo "--- Test de limitation avec récupération ---"
safe_execute_with_limits 50 10 "echo 'Test avec limites'; sleep 5"

echo
echo "--- Test de monitoring (court) ---"
monitor_limits &
monitor_pid=$!

# Laisser tourner 5 secondes
sleep 5

kill "$monitor_pid" 2>/dev/null
echo "Monitoring arrêté"
```

## Section 3 : Contrôleurs de ressources cgroups

### 3.1 Introduction aux cgroups

Les cgroups (Control Groups) pour une gestion fine des ressources :

```bash
#!/bin/bash

# Introduction aux cgroups
echo "=== Introduction aux cgroups ==="

# Vérification du support cgroups
check_cgroups_support() {
    echo "--- Vérification du support cgroups ---"
    
    if [[ ! -d /sys/fs/cgroup ]]; then
        echo "❌ cgroups v2 non disponible (pas de /sys/fs/cgroup)"
        return 1
    fi
    
    if mount | grep -q cgroup; then
        echo "✓ cgroups v1 détecté"
    else
        echo "✓ cgroups v2 détecté"
    fi
    
    # Vérification des contrôleurs disponibles
    local controllers
    controllers=$(cat /sys/fs/cgroup/cgroup.controllers 2>/dev/null || echo "cpu memory pids")
    echo "Contrôleurs disponibles: $controllers"
    
    # Droits d'écriture
    if [[ -w /sys/fs/cgroup ]]; then
        echo "✓ Droits d'administration disponibles"
    else
        echo "⚠️  Droits d'administration limités (nécessite sudo)"
    fi
}

# Fonction de création d'un cgroup
create_cgroup() {
    local name="$1"
    local controllers="${2:-cpu memory}"
    
    echo "--- Création du cgroup: $name ---"
    
    local cgroup_path="/sys/fs/cgroup/$name"
    
    # Création du répertoire
    if ! mkdir -p "$cgroup_path" 2>/dev/null; then
        echo "❌ Impossible de créer le cgroup (droits insuffisants ?)"
        echo "Essayez avec sudo ou vérifiez le montage cgroup"
        return 1
    fi
    
    echo "✓ Cgroup créé: $cgroup_path"
    
    # Configuration des contrôleurs
    for controller in $controllers; do
        if [[ -f "$cgroup_path/cgroup.subtree_control" ]]; then
            # cgroups v2
            echo "+$controller" > "$cgroup_path/cgroup.subtree_control" 2>/dev/null || true
        fi
    done
    
    echo "Contrôleurs configurés: $controllers"
}

# Fonction de configuration des limites
configure_cgroup_limits() {
    local name="$1"
    local limits="$2"  # Format: "cpu.max=50000 memory.max=100M"
    
    echo "--- Configuration des limites pour $name ---"
    
    local cgroup_path="/sys/fs/cgroup/$name"
    
    if [[ ! -d "$cgroup_path" ]]; then
        echo "❌ Cgroup $name inexistant"
        return 1
    fi
    
    # Parsing et application des limites
    IFS=' ' read -ra limit_array <<< "$limits"
    for limit in "${limit_array[@]}"; do
        local key="${limit%%=*}"
        local value="${limit#*=}"
        
        case "$key" in
            cpu.max)
                echo "$value" > "$cgroup_path/cpu.max" 2>/dev/null && echo "✓ CPU max: $value" || echo "❌ Échec configuration CPU"
                ;;
            memory.max)
                # Conversion en bytes si nécessaire
                if [[ "$value" =~ M$ ]]; then
                    value=$(( ${value%M} * 1024 * 1024 ))
                elif [[ "$value" =~ G$ ]]; then
                    value=$(( ${value%G} * 1024 * 1024 * 1024 ))
                fi
                echo "$value" > "$cgroup_path/memory.max" 2>/dev/null && echo "✓ Mémoire max: $value bytes" || echo "❌ Échec configuration mémoire"
                ;;
            pids.max)
                echo "$value" > "$cgroup_path/pids.max" 2>/dev/null && echo "✓ PIDs max: $value" || echo "❌ Échec configuration PIDs"
                ;;
            *)
                echo "⚠️  Limite inconnue: $key"
                ;;
        esac
    done
}

# Fonction d'ajout de processus à un cgroup
add_process_to_cgroup() {
    local name="$1"
    local pid="$2"
    
    echo "--- Ajout du processus $pid au cgroup $name ---"
    
    local cgroup_path="/sys/fs/cgroup/$name"
    local cgroup_procs="$cgroup_path/cgroup.procs"
    
    if [[ ! -f "$cgroup_procs" ]]; then
        echo "❌ Fichier cgroup.procs introuvable"
        return 1
    fi
    
    if echo "$pid" > "$cgroup_procs" 2>/dev/null; then
        echo "✓ Processus $pid ajouté au cgroup $name"
    else
        echo "❌ Échec ajout processus (droits insuffisants ?)"
        return 1
    fi
}

# Fonction de monitoring d'un cgroup
monitor_cgroup() {
    local name="$1"
    local duration="${2:-10}"
    
    echo "--- Monitoring du cgroup $name ($duration secondes) ---"
    
    local cgroup_path="/sys/fs/cgroup/$name"
    
    if [[ ! -d "$cgroup_path" ]]; then
        echo "❌ Cgroup $name inexistant"
        return 1
    fi
    
    local count=0
    while (( count < duration )); do
        echo "[$(date +%H:%M:%S)] === État du cgroup $name ==="
        
        # Processus
        if [[ -f "$cgroup_path/cgroup.procs" ]]; then
            local proc_count
            proc_count=$(wc -l < "$cgroup_path/cgroup.procs")
            echo "Processus: $proc_count"
        fi
        
        # CPU
        if [[ -f "$cgroup_path/cpu.stat" ]]; then
            echo "CPU stat:"
            cat "$cgroup_path/cpu.stat" | sed 's/^/  /'
        fi
        
        # Mémoire
        if [[ -f "$cgroup_path/memory.current" ]]; then
            local mem_current
            mem_current=$(cat "$cgroup_path/memory.current")
            echo "Mémoire utilisée: $((mem_current / 1024 / 1024)) MB"
        fi
        
        sleep 2
        ((count += 2))
        echo
    done
}

# Test des cgroups (nécessite des droits d'administration)
check_cgroups_support

if [[ -w /sys/fs/cgroup ]]; then
    echo
    echo "--- Tests pratiques ---"
    
    # Création d'un cgroup de test
    create_cgroup "test_group" "cpu memory pids"
    
    # Configuration des limites
    configure_cgroup_limits "test_group" "cpu.max=50000 memory.max=50M pids.max=10"
    
    # Ajout du shell courant au cgroup
    add_process_to_cgroup "test_group" "$$"
    
    # Lancement d'un processus de test
    echo "--- Lancement d'un processus de test ---"
    (
        echo "Processus de test démarré (PID: $$)"
        
        # Simulation de charge
        for ((i=0; i<1000000; i++)); do
            true
        done
        
        echo "Processus de test terminé"
    ) &
    test_pid=$!
    
    # Ajout au cgroup
    add_process_to_cgroup "test_group" "$test_pid"
    
    # Monitoring court
    monitor_cgroup "test_group" 6 &
    monitor_bg_pid=$!
    
    # Attendre la fin du test
    wait "$test_pid" 2>/dev/null
    wait "$monitor_bg_pid" 2>/dev/null
    
    # Nettoyage
    rmdir "/sys/fs/cgroup/test_group" 2>/dev/null
    echo "✓ Nettoyage terminé"
else
    echo
    echo "⚠️  Tests cgroups ignorés (droits insuffisants)"
    echo "Pour tester, exécutez avec sudo ou configurez les permissions cgroup"
fi
```

### 3.2 Intégration avancée avec systemd

Utilisation des slices systemd pour la gestion des ressources :

```bash
#!/bin/bash

# Intégration avancée avec systemd
echo "=== Intégration avancée avec systemd ==="

# Fonction de vérification systemd
check_systemd_support() {
    echo "--- Vérification du support systemd ---"
    
    if ! command -v systemctl >/dev/null 2>&1; then
        echo "❌ systemctl non disponible"
        return 1
    fi
    
    if ! systemctl is-system-running >/dev/null 2>&1; then
        echo "❌ systemd non actif"
        return 1
    fi
    
    echo "✓ systemd disponible et actif"
    
    # Vérification des slices utilisateur
    if systemctl --user list-units --type=slice >/dev/null 2>&1; then
        echo "✓ Slices utilisateur supportés"
    else
        echo "⚠️  Slices utilisateur non disponibles"
    fi
}

# Fonction de création d'une slice systemd
create_systemd_slice() {
    local slice_name="$1"
    local cpu_limit="${2:-50%}"
    local memory_limit="${3:-100M}"
    
    echo "--- Création de la slice systemd: $slice_name ---"
    
    local slice_file="/etc/systemd/system/${slice_name}.slice"
    
    # Création du fichier slice
    cat > "$slice_file" << EOF
[Unit]
Description=Slice for $slice_name
Before=slices.target

[Slice]
CPUQuota=${cpu_limit}
MemoryLimit=${memory_limit}
EOF
    
    echo "✓ Slice créée: $slice_file"
    
    # Rechargement de systemd
    if systemctl daemon-reload; then
        echo "✓ Configuration rechargée"
    else
        echo "❌ Échec rechargement configuration"
        return 1
    fi
    
    # Démarrage de la slice
    if systemctl start "${slice_name}.slice"; then
        echo "✓ Slice démarrée"
    else
        echo "❌ Échec démarrage slice"
        return 1
    fi
}

# Fonction d'exécution dans une slice
run_in_slice() {
    local slice_name="$1"
    shift
    local command="$*"
    
    echo "--- Exécution dans la slice $slice_name ---"
    echo "Commande: $command"
    
    # Utilisation de systemd-run pour exécuter dans la slice
    if systemd-run --slice="${slice_name}.slice" --remain-after-exit bash -c "$command"; then
        echo "✓ Commande lancée dans la slice"
    else
        echo "❌ Échec lancement dans la slice"
        return 1
    fi
}

# Fonction de monitoring d'une slice
monitor_slice() {
    local slice_name="$1"
    
    echo "--- Monitoring de la slice: $slice_name ---"
    
    # État de la slice
    systemctl status "${slice_name}.slice" --no-pager
    
    # Propriétés
    echo
    echo "Propriétés de la slice:"
    systemctl show "${slice_name}.slice" | grep -E "(CPUQuota|MemoryLimit|ActiveState)" | sed 's/^/  /'
    
    # Processus dans la slice
    echo
    echo "Processus dans la slice:"
    systemctl status "${slice_name}.slice" -o cat | grep -A 50 "CGroup:" | grep -E "^[[:space:]]*[0-9]+" | head -10 || echo "  Aucun processus"
}

# Fonction de nettoyage des slices
cleanup_slice() {
    local slice_name="$1"
    
    echo "--- Nettoyage de la slice: $slice_name ---"
    
    # Arrêt de la slice
    systemctl stop "${slice_name}.slice" 2>/dev/null || true
    
    # Suppression du fichier
    rm -f "/etc/systemd/system/${slice_name}.slice"
    
    # Rechargement
    systemctl daemon-reload 2>/dev/null || true
    
    echo "✓ Slice nettoyée"
}

# Tests systemd (nécessite des droits d'administration)
check_systemd_support

if command -v systemctl >/dev/null 2>&1 && systemctl is-system-running >/dev/null 2>&1; then
    echo
    echo "--- Tests pratiques (nécessite sudo) ---"
    
    # Test de création de slice
    if sudo -n true 2>/dev/null; then
        echo "Droits sudo disponibles - tests activés"
        
        # Création d'une slice de test
        sudo bash -c "
            $(declare -f create_systemd_slice)
            create_systemd_slice 'test-slice' '30%' '50M'
        "
        
        # Exécution d'une commande dans la slice
        sudo systemd-run --slice=test-slice.slice --remain-after-exit bash -c 'echo "Test dans slice"; sleep 2'
        
        # Monitoring
        sudo bash -c "
            $(declare -f monitor_slice)
            monitor_slice 'test-slice'
        "
        
        # Nettoyage
        sudo bash -c "
            $(declare -f cleanup_slice)
            cleanup_slice 'test-slice'
        "
        
        echo "✓ Tests systemd terminés"
    else
        echo "⚠️  Tests systemd ignorés (sudo non disponible ou nécessite mot de passe)"
        echo "Pour tester: sudo $0"
    fi
else
    echo
    echo "⚠️  Tests systemd ignorés (systemd non disponible)"
fi
```

## Section 4 : Supervision proactive et alertes

### 4.1 Système d'alertes intelligent

Alertes adaptatives basées sur les tendances :

```bash
#!/bin/bash

# Système d'alertes intelligent
echo "=== Système d'alertes intelligent ==="

# Structure de données pour l'historique
declare -a cpu_history
declare -a mem_history
declare -a disk_history

# Fonction de collecte d'historique
collect_history() {
    local samples="${1:-10}"
    
    echo "Collecte d'historique ($samples échantillons)..."
    
    for ((i=0; i<samples; i++)); do
        # CPU
        local cpu
        cpu=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
        cpu_history+=("$cpu")
        
        # Mémoire
        local mem
        mem=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
        mem_history+=("$mem")
        
        # Disque
        local disk
        disk=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
        disk_history+=("$disk")
        
        sleep 1
    done
    
    echo "✓ Historique collecté"
}

# Fonction de calcul de tendance
calculate_trend() {
    local -a history=("$@")
    local window="${#history[@]}"
    
    if (( window < 2 )); then
        echo "0"
        return
    fi
    
    local first_half_sum=0
    local second_half_sum=0
    local half=$((window / 2))
    
    for ((i=0; i<half; i++)); do
        first_half_sum=$(echo "$first_half_sum + ${history[$i]}" | bc -l)
    done
    
    for ((i=half; i<window; i++)); do
        second_half_sum=$(echo "$second_half_sum + ${history[$i]}" | bc -l)
    done
    
    local first_avg=$(echo "scale=2; $first_half_sum / $half" | bc -l)
    local second_avg=$(echo "scale=2; $second_half_sum / ($window - $half)" | bc -l)
    
    # Tendance: différence entre deuxième moitié et première moitié
    local trend=$(echo "scale=2; $second_avg - $first_avg" | bc -l)
    
    echo "$trend"
}

# Fonction d'analyse de tendance
analyze_trends() {
    echo "--- Analyse des tendances ---"
    
    local cpu_trend
    local mem_trend
    local disk_trend
    
    cpu_trend=$(calculate_trend "${cpu_history[@]}")
    mem_trend=$(calculate_trend "${mem_history[@]}")
    disk_trend=$(calculate_trend "${disk_history[@]}")
    
    echo "Tendances (variation moyenne):"
    echo "  CPU: ${cpu_trend}%"
    echo "  RAM: ${mem_trend}%"
    echo "  Disk: ${disk_trend}%"
    
    # Alertes basées sur les tendances
    local alerts=()
    
    if (( $(echo "$cpu_trend > 5" | bc -l) )); then
        alerts+=("CPU en augmentation rapide (+${cpu_trend}%)")
    fi
    
    if (( $(echo "$mem_trend > 3" | bc -l) )); then
        alerts+=("Mémoire en augmentation (+${mem_trend}%)")
    fi
    
    if (( $(echo "$disk_trend > 1" | bc -l) )); then
        alerts+=("Disque en augmentation (+${disk_trend}%)")
    fi
    
    # Tendances négatives (amélioration)
    if (( $(echo "$cpu_trend < -5" | bc -l) )); then
        alerts+=("CPU en baisse (${cpu_trend}%) - amélioration")
    fi
    
    if (( ${#alerts[@]} > 0 )); then
        echo
        echo "🚨 ALERTES DE TENDANCE:"
        for alert in "${alerts[@]}"; do
            echo "  $alert"
        done
    else
        echo "✅ Aucune tendance préoccupante détectée"
    fi
}

# Fonction d'alertes adaptatives
adaptive_alerts() {
    local metric="$1"
    local current_value="$2"
    local history=("${@:3}")
    
    # Calcul de la moyenne historique
    local sum=0
    for value in "${history[@]}"; do
        sum=$(echo "$sum + $value" | bc -l)
    done
    local avg=$(echo "scale=2; $sum / ${#history[@]}" | bc -l)
    
    # Calcul de l'écart-type
    local variance=0
    for value in "${history[@]}"; do
        local diff=$(echo "$value - $avg" | bc -l)
        variance=$(echo "$variance + ($diff * $diff)" | bc -l)
    done
    variance=$(echo "scale=2; $variance / ${#history[@]}" | bc -l)
    local stddev=$(echo "scale=2; sqrt($variance)" | bc -l)
    
    # Seuils adaptatifs (moyenne + 2 écarts-types)
    local threshold_high=$(echo "scale=2; $avg + (2 * $stddev)" | bc -l)
    local threshold_low=$(echo "scale=2; $avg - (2 * $stddev)" | bc -l)
    
    # Alerte si hors des seuils
    if (( $(echo "$current_value > $threshold_high" | bc -l) )); then
        echo "🚨 $metric ÉLEVÉ: $current_value% (seuil: ${threshold_high}%, moyenne: ${avg}%)"
        return 1
    elif (( $(echo "$current_value < $threshold_low" | bc -l) )); then
        echo "ℹ️  $metric FAIBLE: $current_value% (seuil: ${threshold_low}%, moyenne: ${avg}%)"
        return 0
    else
        return 0
    fi
}

# Système de supervision principal
supervision_system() {
    local interval="${1:-30}"
    local duration="${2:-300}"
    
    echo "=== SYSTÈME DE SUPERVISION ==="
    echo "Intervalle: ${interval}s, Durée: ${duration}s"
    echo
    
    # Collecte initiale d'historique
    collect_history 10
    
    local start_time=$(date +%s)
    
    while (( $(date +%s) - start_time < duration )); do
        # Mesures actuelles
        local cpu_current="${cpu_history[-1]}"
        local mem_current="${mem_history[-1]}"
        local disk_current="${disk_history[-1]}"
        
        echo "[$(date)] === Contrôles adaptatifs ==="
        
        # Alertes adaptatives
        adaptive_alerts "CPU" "$cpu_current" "${cpu_history[@]}"
        adaptive_alerts "RAM" "$mem_current" "${mem_history[@]}"
        adaptive_alerts "DISK" "$disk_current" "${disk_history[@]}"
        
        # Analyse de tendance périodique
        if (( ($(date +%s) - start_time) % 120 == 0 )); then
            echo
            analyze_trends
            echo
        fi
        
        # Collecte de nouvelles mesures
        local cpu
        cpu=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
        cpu_history+=("$cpu")
        
        local mem
        mem=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
        mem_history+=("$mem")
        
        local disk
        disk=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
        disk_history+=("$disk")
        
        # Limitation de la taille de l'historique
        if (( ${#cpu_history[@]} > 100 )); then
            cpu_history=("${cpu_history[@]:50}")
            mem_history=("${mem_history[@]:50}")
            disk_history=("${disk_history[@]:50}")
        fi
        
        sleep "$interval"
    done
    
    echo "Supervision terminée"
}

# Test du système de supervision (version courte)
echo "--- Test du système de supervision (court) ---"
supervision_system 5 30
```

### 4.2 Intégration avec des outils de monitoring externes

Connexion avec Nagios, Zabbix, Prometheus :

```bash
#!/bin/bash

# Intégration avec des outils de monitoring externes
echo "=== Intégration avec outils de monitoring externes ==="

# Fonction d'export vers Prometheus
export_to_prometheus() {
    local metric_name="$1"
    local value="$2"
    local labels="${3:-}"
    
    # Format Prometheus
    local prometheus_line="${metric_name}{${labels}} $value"
    
    # Écriture dans un fichier ou envoi à pushgateway
    echo "$prometheus_line" >> /tmp/prometheus_metrics.txt
    
    echo "✓ Métrique exportée: $prometheus_line"
}

# Fonction de génération de rapports Nagios/Icinga
generate_nagios_report() {
    local service_name="$1"
    local status="$2"  # OK, WARNING, CRITICAL, UNKNOWN
    local message="$3"
    
    # Format Nagios
    local nagios_output="$service_name;$status;$message"
    
    echo "$nagios_output"
    
    # Écriture dans un fichier de rapport
    echo "[$(date)] $nagios_output" >> /tmp/nagios_reports.txt
}

# Fonction d'envoi d'alertes vers un webhook
send_webhook_alert() {
    local webhook_url="$1"
    local alert_data="$2"
    
    if command -v curl >/dev/null 2>&1; then
        curl -X POST "$webhook_url" \
             -H "Content-Type: application/json" \
             -d "$alert_data" \
             --max-time 10 \
             --silent \
             --show-error
        echo "✓ Alerte webhook envoyée"
    else
        echo "❌ curl non disponible pour webhook"
    fi
}

# Fonction principale d'export des métriques
export_system_metrics() {
    local prometheus_gateway="${1:-}"
    local webhook_url="${2:-}"
    
    echo "--- Export des métriques système ---"
    
    # Collecte des métriques
    local cpu_usage
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
    
    local mem_usage
    mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
    
    local disk_usage
    disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
    
    local load_avg
    load_avg=$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)
    
    # Export Prometheus
    export_to_prometheus "system_cpu_usage_percent" "$cpu_usage" "host=$(hostname)"
    export_to_prometheus "system_memory_usage_percent" "$mem_usage" "host=$(hostname)"
    export_to_prometheus "system_disk_usage_percent" "$disk_usage" "host=$(hostname),mount=/"
    export_to_prometheus "system_load_average" "$load_avg" "host=$(hostname)"
    
    # Vérifications Nagios
    if (( $(echo "$cpu_usage > 90" | bc -l) )); then
        generate_nagios_report "CPU_USAGE" "CRITICAL" "CPU usage is ${cpu_usage}%"
    elif (( $(echo "$cpu_usage > 80" | bc -l) )); then
        generate_nagios_report "CPU_USAGE" "WARNING" "CPU usage is ${cpu_usage}%"
    else
        generate_nagios_report "CPU_USAGE" "OK" "CPU usage is ${cpu_usage}%"
    fi
    
    if (( mem_usage > 95 )); then
        generate_nagios_report "MEMORY_USAGE" "CRITICAL" "Memory usage is ${mem_usage}%"
    elif (( mem_usage > 85 )); then
        generate_nagios_report "MEMORY_USAGE" "WARNING" "Memory usage is ${mem_usage}%"
    else
        generate_nagios_report "MEMORY_USAGE" "OK" "Memory usage is ${mem_usage}%"
    fi
    
    # Alertes webhook si seuils dépassés
    if [[ -n "$webhook_url" ]] && (( $(echo "$cpu_usage > 85" | bc -l) || mem_usage > 90 || disk_usage > 95 )); then
        local alert_json="{
            \"timestamp\": \"$(date -Iseconds)\",
            \"host\": \"$(hostname)\",
            \"alerts\": {
                \"cpu_usage\": ${cpu_usage},
                \"memory_usage\": ${mem_usage},
                \"disk_usage\": ${disk_usage}
            }
        }"
        
        send_webhook_alert "$webhook_url" "$alert_json"
    fi
    
    echo "✓ Métriques exportées"
}

# Fonction de configuration d'un collecteur continu
setup_continuous_exporter() {
    local interval="${1:-60}"
    local prometheus_gateway="${2:-}"
    local webhook_url="${3:-}"
    
    echo "Configuration de l'export continu (intervalle: ${interval}s)"
    
    # Création du script d'export
    cat > /tmp/metrics_exporter.sh << EOF
#!/bin/bash

while true; do
    $(declare -f export_system_metrics)
    $(declare -f export_to_prometheus)
    $(declare -f generate_nagios_report)
    $(declare -f send_webhook_alert)
    
    export_system_metrics "$prometheus_gateway" "$webhook_url"
    
    # Envoi vers pushgateway si configuré
    if [[ -n "$prometheus_gateway" ]] && command -v curl >/dev/null 2>&1; then
        echo "# Métriques générées le $(date)" | cat - /tmp/prometheus_metrics.txt | 
            curl --data-binary @- "$prometheus_gateway/metrics/job/system_monitor/host/$(hostname)"
    fi
    
    sleep $interval
done
EOF
    
    chmod +x /tmp/metrics_exporter.sh
    echo "✓ Script d'export créé: /tmp/metrics_exporter.sh"
    echo "Pour démarrer: /tmp/metrics_exporter.sh &"
}

# Démonstration
echo "--- Test d'export des métriques ---"
export_system_metrics

echo
echo "--- Configuration de l'export continu ---"
setup_continuous_exporter 30

echo
echo "Fichiers générés:"
ls -la /tmp/*metrics* /tmp/*reports* 2>/dev/null || echo "Aucun fichier généré"

# Nettoyage
rm -f /tmp/prometheus_metrics.txt /tmp/nagios_reports.txt /tmp/metrics_exporter.sh
```

## Conclusion : L'harmonie dans la complexité

La gestion des ressources système en Bash transforme vos scripts de simples automatisations en véritables orchestrateurs système. Comme un chef d'orchestre qui ajuste constamment le volume de chaque instrument, vous apprenez à équilibrer CPU, mémoire, disque et réseau pour des performances optimales.

**Points clés à retenir :**

1. **Monitoring continu** : Collectez les métriques essentielles (CPU, RAM, disque, réseau) en temps réel
2. **Limites ulimit** : Contrôlez finement les ressources allouées aux processus avec ulimit
3. **cgroups avancés** : Utilisez les contrôleurs de groupes Linux pour une isolation parfaite
4. **systemd intégré** : Profitez des slices systemd pour une gestion déclarative des ressources
5. **Alertes proactives** : Anticipez les problèmes avec des seuils adaptatifs et l'analyse de tendance

Dans le chapitre suivant, nous explorerons les techniques de déploiement et d'automatisation à grande échelle, pour que vos scripts Bash puissent gérer des infrastructures complexes et distribuées.

---

**Exercice pratique :** Créez un système de supervision complet qui :
- Monitor les ressources système en temps réel
- Applique des limites adaptatives selon la charge
- Envoie des alertes multi-canaux (email, webhook, syslog)
- Génère des rapports de tendance quotidiens
- S'auto-optimise en fonction des patterns de charge observés

**Réflexion :** Comment adapteriez-vous ces techniques de monitoring pour un environnement cloud où les ressources sont dynamiques et payantes ?
