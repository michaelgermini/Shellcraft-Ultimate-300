# Chapitre 35 - Swap et mémoire

## Table des matières
- [Introduction](#introduction)
- [Architecture de la mémoire Linux](#architecture-de-la-mémoire-linux)
- [Mémoire virtuelle et swap](#mémoire-virtuelle-et-swap)
- [Configuration du swap](#configuration-du-swap)
- [Gestion avancée du swap](#gestion-avancée-du-swap)
- [Optimisation de la mémoire](#optimisation-de-la-mémoire)
- [Monitoring et diagnostic](#monitoring-et-diagnostic)
- [Paramètres du kernel](#paramètres-du-kernel)
- [Swap avancé et techniques spéciales](#swap-avancé-et-techniques-spéciales)
- [Dépannage mémoire](#dépannage-mémoire)
- [Performance et tuning](#performance-et-tuning)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La gestion de la mémoire et du swap constitue un aspect critique de l'administration système. Comprendre comment Linux gère la mémoire virtuelle, optimise les ressources, et utilise l'espace swap permet de maintenir des systèmes stables et performants même sous charge importante.

Imaginez la mémoire comme une bibliothèque universitaire : les livres fréquemment consultés restent sur les étagères principales (RAM), tandis que les ouvrages moins utilisés sont stockés dans les archives (swap), prêts à être rappelés rapidement en cas de besoin. Dans un système Linux moderne, cette métaphore s'étend : le système utilise des algorithmes sophistiqués pour décider quoi garder en mémoire, quand utiliser le swap, et comment optimiser les performances globales.

## Architecture de la mémoire Linux

### Types de mémoire

**Mémoire physique (RAM)** :
```bash
#!/bin/bash
# Comprendre la mémoire physique

# Informations détaillées depuis /proc/meminfo
cat /proc/meminfo

# Mémoire totale
grep MemTotal /proc/meminfo

# Mémoire disponible
grep MemAvailable /proc/meminfo

# Mémoire utilisée
grep MemUsed /proc/meminfo || {
    mem_total=$(grep MemTotal /proc/meminfo | awk '{print $2}')
    mem_available=$(grep MemAvailable /proc/meminfo | awk '{print $2}')
    mem_used=$((mem_total - mem_available))
    echo "MemUsed: $mem_used kB"
}

# Mémoire libre
grep MemFree /proc/meminfo
```

**Mémoire virtuelle** :
```bash
#!/bin/bash
# Mémoire virtuelle

# Taille totale de la mémoire virtuelle
grep VmallocTotal /proc/meminfo

# Mémoire virtuelle utilisée
grep VmallocUsed /proc/meminfo

# Mémoire virtuelle disponible
grep VmallocChunk /proc/meminfo
```

### Zones de mémoire

**Zones mémoire du kernel** :
```bash
#!/bin/bash
# Zones de mémoire

# Informations sur les zones
cat /proc/zoneinfo

# Résumé par zone
cat /proc/zoneinfo | grep -E "Node|zone|pages free|pages managed"

# Statistiques de mémoire
cat /proc/vmstat
```

## Mémoire virtuelle et swap

### Concept de swap

**Qu'est-ce que le swap ?** :
```bash
#!/bin/bash
# Comprendre le swap

# Swap est un espace disque utilisé comme extension de la RAM
# Permet au système de "swapper" des pages mémoire vers le disque
# Libère de la RAM pour les processus actifs

# Avantages :
# - Permet d'exécuter plus de processus que la RAM ne le permet
# - Évite l'OOM (Out Of Memory) killer
# - Permet l'hibernation

# Inconvénients :
# - Plus lent que la RAM (disque)
# - Peut dégrader les performances si utilisé excessivement
```

**Types de swap** :
```bash
#!/bin/bash
# Types de swap

# 1. Fichier swap
# Créé comme fichier normal sur un système de fichiers
# Avantage : Facile à créer/modifier
# Inconvénient : Plus lent (traverse le système de fichiers)

# 2. Partition swap
# Partition dédiée sur le disque
# Avantage : Plus rapide, pas de fragmentation
# Inconvénient : Nécessite partitionnement

# 3. Swap sur LVM
# Volume logique LVM dédié au swap
# Avantage : Flexibilité, peut être redimensionné
# Inconvénient : Légèrement plus lent

# Vérifier le type de swap actuel
swapon --show
cat /proc/swaps
```

## Configuration du swap

### Création d'un fichier swap

**Créer un fichier swap** :
```bash
#!/bin/bash
# Création d'un fichier swap

create_swap_file() {
    local swap_file="${1:-/swapfile}"
    local swap_size="${2:-2G}"
    
    # Créer le fichier (fallocate plus rapide que dd)
    sudo fallocate -l "$swap_size" "$swap_file"
    
    # Sécuriser les permissions
    sudo chmod 600 "$swap_file"
    
    # Formater comme swap
    sudo mkswap "$swap_file"
    
    # Activer le swap
    sudo swapon "$swap_file"
    
    # Vérifier
    swapon --show
    
    echo "Fichier swap créé et activé: $swap_file"
}

# Alternative avec dd (si fallocate non disponible)
create_swap_file_dd() {
    local swap_file="${1:-/swapfile}"
    local swap_size_gb="${2:-2}"
    
    # Calculer la taille en blocs (1GB = 1024*1024*1024 / 1024 = 1048576 blocs de 1KB)
    local blocks=$((swap_size_gb * 1048576))
    
    sudo dd if=/dev/zero of="$swap_file" bs=1024 count="$blocks"
    sudo chmod 600 "$swap_file"
    sudo mkswap "$swap_file"
    sudo swapon "$swap_file"
}
```

**Persistance du swap** :
```bash
#!/bin/bash
# Rendre le swap persistant

# Ajouter à /etc/fstab
add_swap_to_fstab() {
    local swap_file="${1:-/swapfile}"
    
    # Vérifier si déjà présent
    if grep -q "$swap_file" /etc/fstab; then
        echo "Swap déjà dans /etc/fstab"
        return 0
    fi
    
    # Ajouter l'entrée
    echo "$swap_file none swap sw 0 0" | sudo tee -a /etc/fstab
    
    echo "Swap ajouté à /etc/fstab"
    echo "Vérifier avec: sudo mount -a"
}

# Vérifier la configuration fstab
verify_fstab_swap() {
    echo "=== Entrées swap dans /etc/fstab ==="
    grep swap /etc/fstab
}
```

### Création d'une partition swap

**Partition swap** :
```bash
#!/bin/bash
# Création d'une partition swap

create_swap_partition() {
    local device="${1:-/dev/sdb1}"
    local label="${2:-swap1}"
    
    # Formater comme swap
    sudo mkswap -L "$label" "$device"
    
    # Activer
    sudo swapon "$device"
    
    # Ajouter à fstab
    local uuid=$(sudo blkid -s UUID -o value "$device")
    echo "UUID=$uuid none swap sw 0 0" | sudo tee -a /etc/fstab
    
    echo "Partition swap créée: $device"
}
```

## Gestion avancée du swap

### swapon et swapoff

**Gestion du swap** :
```bash
#!/bin/bash
# Gestion avancée du swap

# Activer un swap
sudo swapon /swapfile
sudo swapon /dev/sdb1

# Activer tous les swaps de fstab
sudo swapon -a

# Désactiver un swap
sudo swapoff /swapfile
sudo swapoff /dev/sdb1

# Désactiver tous les swaps
sudo swapoff -a

# Afficher les swaps actifs
swapon --show
swapon --summary

# Informations détaillées
swapon --show=NAME,TYPE,SIZE,USED,PRIO
```

**Priorité du swap** :
```bash
#!/bin/bash
# Gestion de la priorité du swap

# Plus la priorité est élevée, plus le swap est utilisé en premier
# Plage : -1 à 32767
# Par défaut : -1 (priorité basse)

# Activer avec priorité
sudo swapon -p 10 /swapfile1  # Priorité haute
sudo swapon -p 5 /swapfile2   # Priorité moyenne
sudo swapon -p 1 /swapfile3   # Priorité basse

# Vérifier les priorités
swapon --show=NAME,PRIO

# Le système utilisera d'abord swapfile1, puis swapfile2, puis swapfile3
```

### Redimensionnement du swap

**Modifier la taille du swap** :
```bash
#!/bin/bash
# Redimensionnement du swap

resize_swap_file() {
    local swap_file="${1:-/swapfile}"
    local new_size="${2:-4G}"
    
    # Désactiver le swap
    sudo swapoff "$swap_file"
    
    # Supprimer l'ancien fichier
    sudo rm "$swap_file"
    
    # Créer le nouveau fichier
    sudo fallocate -l "$new_size" "$swap_file"
    sudo chmod 600 "$swap_file"
    sudo mkswap "$swap_file"
    
    # Réactiver
    sudo swapon "$swap_file"
    
    echo "Swap redimensionné à $new_size"
}

# Redimensionner une partition swap (nécessite LVM)
resize_swap_partition() {
    local lv_path="${1:-/dev/myvg/swap}"
    local new_size="${2:-8G}"
    
    # Désactiver
    sudo swapoff "$lv_path"
    
    # Redimensionner le LV
    sudo lvreduce -L "$new_size" "$lv_path"
    # ou
    sudo lvextend -L "$new_size" "$lv_path"
    
    # Reformater
    sudo mkswap "$lv_path"
    
    # Réactiver
    sudo swapon "$lv_path"
}
```

## Optimisation de la mémoire

### free : affichage de la mémoire

**Utilisation de free** :
```bash
#!/bin/bash
# Utilisation de free

# Affichage de base
free

# Format humain
free -h

# Format en MB
free -m

# Format en GB
free -g

# Affichage continu
free -h -s 2  # Toutes les 2 secondes

# Nombre d'itérations
free -h -s 1 -c 10  # 10 fois toutes les secondes

# Affichage détaillé
free -h -w  # Colonnes larges
```

**Scripts avec free** :
```bash
#!/bin/bash
# Scripts utilitaires avec free

# Mémoire disponible en pourcentage
get_memory_usage() {
    free | grep Mem | awk '{
        total = $2
        used = $3
        available = $7
        usage = (used / total) * 100
        printf "Mémoire utilisée: %.1f%% (%d MB / %d MB)\n", usage, used/1024, total/1024
        printf "Mémoire disponible: %d MB\n", available/1024
    }'
}

# Swap utilisé en pourcentage
get_swap_usage() {
    free | grep Swap | awk '{
        if ($2 > 0) {
            usage = ($3 / $2) * 100
            printf "Swap utilisé: %.1f%% (%d MB / %d MB)\n", usage, $3/1024, $2/1024
        } else {
            print "Swap non configuré"
        }
    }'
}

# Alertes mémoire
check_memory_threshold() {
    local mem_threshold="${1:-80}"
    local swap_threshold="${2:-50}"
    
    local mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
    local swap_usage=$(free | grep Swap | awk '{if ($2 > 0) printf "%.0f", $3/$2 * 100; else print 0}')
    
    if [ "$mem_usage" -gt "$mem_threshold" ]; then
        echo "⚠️  Mémoire élevée: ${mem_usage}%"
    fi
    
    if [ "$swap_usage" -gt "$swap_threshold" ]; then
        echo "⚠️  Swap élevé: ${swap_usage}%"
    fi
}
```

### vmstat : statistiques virtuelles

**Utilisation de vmstat** :
```bash
#!/bin/bash
# Utilisation de vmstat

# Statistiques instantanées
vmstat

# Statistiques avec délai
vmstat 2  # Toutes les 2 secondes

# Statistiques avec nombre d'itérations
vmstat 2 10  # 10 fois toutes les 2 secondes

# Statistiques détaillées
vmstat -s  # Statistiques résumées
vmstat -d  # Statistiques disque
vmstat -m  # Statistiques slab
vmstat -a  # Mémoire active/inactive

# Format personnalisé
vmstat -n 2  # Pas d'en-têtes répétés
```

**Interprétation de vmstat** :
```bash
#!/bin/bash
# Interprétation de vmstat

# Colonnes importantes :
# r : Processus en attente d'exécution
# b : Processus en attente I/O
# swpd : Swap utilisé (KB)
# free : Mémoire libre (KB)
# buff : Buffer cache (KB)
# cache : Page cache (KB)
# si : Pages swap in (par seconde)
# so : Pages swap out (par seconde)
# bi : Blocs lus depuis disque (par seconde)
# bo : Blocs écrits vers disque (par seconde)

# Script d'analyse
analyze_vmstat() {
    vmstat 1 5 | awk '
    NR > 2 {
        swpd_sum += $3
        free_sum += $4
        si_sum += $7
        so_sum += $8
        count++
    }
    END {
        if (count > 0) {
            printf "Swap moyen utilisé: %.0f KB\n", swpd_sum/count
            printf "Mémoire libre moyenne: %.0f KB\n", free_sum/count
            printf "Swap in moyen: %.2f pages/s\n", si_sum/count
            printf "Swap out moyen: %.2f pages/s\n", so_sum/count
            
            if (si_sum/count > 0 || so_sum/count > 0) {
                print "⚠️  Activité swap détectée - système peut être sous pression mémoire"
            }
        }
    }'
}
```

## Monitoring et diagnostic

### /proc/meminfo : informations détaillées

**Analyse de /proc/meminfo** :
```bash
#!/bin/bash
# Analyse complète de /proc/meminfo

analyze_meminfo() {
    echo "=== Analyse de la mémoire ==="
    echo ""
    
    # Mémoire totale et disponible
    local mem_total=$(grep MemTotal /proc/meminfo | awk '{print $2}')
    local mem_available=$(grep MemAvailable /proc/meminfo | awk '{print $2}')
    local mem_free=$(grep MemFree /proc/meminfo | awk '{print $2}')
    
    echo "Mémoire totale: $((mem_total / 1024)) MB"
    echo "Mémoire disponible: $((mem_available / 1024)) MB"
    echo "Mémoire libre: $((mem_free / 1024)) MB"
    echo ""
    
    # Mémoire utilisée
    local mem_used=$((mem_total - mem_available))
    local mem_usage_percent=$((mem_used * 100 / mem_total))
    echo "Mémoire utilisée: $((mem_used / 1024)) MB ($mem_usage_percent%)"
    echo ""
    
    # Cache et buffers
    local cached=$(grep "^Cached:" /proc/meminfo | awk '{print $2}')
    local buffers=$(grep "^Buffers:" /proc/meminfo | awk '{print $2}')
    local slab=$(grep "^Slab:" /proc/meminfo | awk '{print $2}')
    
    echo "Cache: $((cached / 1024)) MB"
    echo "Buffers: $((buffers / 1024)) MB"
    echo "Slab: $((slab / 1024)) MB"
    echo ""
    
    # Swap
    local swap_total=$(grep SwapTotal /proc/meminfo | awk '{print $2}')
    local swap_free=$(grep SwapFree /proc/meminfo | awk '{print $2}')
    
    if [ "$swap_total" -gt 0 ]; then
        local swap_used=$((swap_total - swap_free))
        local swap_usage_percent=$((swap_used * 100 / swap_total))
        echo "Swap total: $((swap_total / 1024)) MB"
        echo "Swap utilisé: $((swap_used / 1024)) MB ($swap_usage_percent%)"
    else
        echo "Swap non configuré"
    fi
}
```

### Monitoring en temps réel

**Scripts de monitoring** :
```bash
#!/bin/bash
# Monitoring mémoire en temps réel

monitor_memory() {
    local interval="${1:-5}"
    
    while true; do
        clear
        echo "=== Monitoring mémoire ==="
        echo "Temps: $(date)"
        echo ""
        
        free -h
        echo ""
        
        echo "=== Top processus par mémoire ==="
        ps aux --sort=-%mem | head -6
        echo ""
        
        echo "=== Activité swap ==="
        vmstat 1 2 | tail -1 | awk '{printf "Swap in: %d pages/s, Swap out: %d pages/s\n", $7, $8}'
        
        sleep "$interval"
    done
}

# Monitoring avec alertes
monitor_memory_with_alerts() {
    local mem_threshold="${1:-85}"
    local swap_threshold="${2:-50}"
    local interval="${3:-10}"
    
    while true; do
        local mem_usage=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
        local swap_usage=$(free | grep Swap | awk '{if ($2 > 0) printf "%.0f", $3/$2 * 100; else print 0}')
        
        if [ "$mem_usage" -gt "$mem_threshold" ]; then
            echo "[$(date)] ⚠️  ALERTE: Mémoire à ${mem_usage}%"
        fi
        
        if [ "$swap_usage" -gt "$swap_threshold" ]; then
            echo "[$(date)] ⚠️  ALERTE: Swap à ${swap_usage}%"
        fi
        
        sleep "$interval"
    done
}
```

## Paramètres du kernel

### sysctl : paramètres de mémoire

**Paramètres importants** :
```bash
#!/bin/bash
# Paramètres sysctl pour la mémoire

# Afficher tous les paramètres mémoire
sysctl -a | grep vm.

# Paramètres importants :

# swappiness : Tendance à utiliser le swap (0-100)
# 0 = Éviter swap autant que possible
# 100 = Utiliser swap agressivement
# Défaut : 60
sysctl vm.swappiness
sudo sysctl -w vm.swappiness=10  # Réduire l'utilisation du swap

# dirty_ratio : Pourcentage de RAM avant écriture synchrone des données sales
sysctl vm.dirty_ratio
sudo sysctl -w vm.dirty_ratio=15

# dirty_background_ratio : Pourcentage avant écriture asynchrone
sysctl vm.dirty_background_ratio
sudo sysctl -w vm.dirty_background_ratio=5

# overcommit_memory : Politique de sur-allocation
# 0 = Heuristique (défaut)
# 1 = Toujours sur-allouer
# 2 = Ne jamais sur-allouer
sysctl vm.overcommit_memory

# overcommit_ratio : Pourcentage de RAM + swap pour sur-allocation
sysctl vm.overcommit_ratio
```

**Configuration persistante** :
```bash
#!/bin/bash
# Configuration persistante sysctl

# Fichier de configuration : /etc/sysctl.conf
# ou fichiers dans /etc/sysctl.d/*.conf

configure_memory_sysctl() {
    local config_file="/etc/sysctl.d/99-memory.conf"
    
    sudo tee "$config_file" << 'EOF'
# Optimisation mémoire
vm.swappiness=10
vm.dirty_ratio=15
vm.dirty_background_ratio=5
vm.overcommit_memory=0
vm.overcommit_ratio=50

# Cache et buffers
vm.vfs_cache_pressure=50
vm.dirty_expire_centisecs=3000
vm.dirty_writeback_centisecs=500
EOF
    
    # Appliquer immédiatement
    sudo sysctl -p "$config_file"
    
    echo "Configuration sauvegardée dans $config_file"
}
```

## Swap avancé et techniques spéciales

### Swap sur plusieurs disques

**Swap distribué** :
```bash
#!/bin/bash
# Configuration de swap sur plusieurs disques

# Avantage : Parallélisation des I/O swap
# Améliore les performances si swap utilisé

setup_multiple_swaps() {
    # Créer plusieurs fichiers swap
    for i in {1..3}; do
        sudo fallocate -l 2G "/swapfile$i"
        sudo chmod 600 "/swapfile$i"
        sudo mkswap "/swapfile$i"
        sudo swapon -p "$i" "/swapfile$i"  # Priorités différentes
    done
    
    # Vérifier
    swapon --show
}
```

### Swap compressé (zswap)

**zswap : swap compressé en mémoire** :
```bash
#!/bin/bash
# Configuration zswap

# zswap compresse les pages avant de les envoyer au swap
# Réduit l'utilisation du swap disque
# Améliore les performances

# Vérifier si zswap est activé
cat /sys/module/zswap/parameters/enabled

# Activer zswap (au démarrage)
# Ajouter au kernel : zswap.enabled=1
# ou dans /etc/default/grub :
# GRUB_CMDLINE_LINUX="zswap.enabled=1"

# Paramètres zswap
cat /sys/module/zswap/parameters/*
```

## Dépannage mémoire

### OOM Killer

**Comprendre l'OOM Killer** :
```bash
#!/bin/bash
# Gestion de l'OOM Killer

# OOM Killer tue des processus quand la mémoire est épuisée
# Vérifier les logs
sudo dmesg | grep -i "out of memory"
sudo journalctl -k | grep -i "oom"

# Vérifier les scores OOM
check_oom_scores() {
    echo "=== Scores OOM des processus ==="
    ps aux | awk 'NR>1 {
        pid = $2
        cmd = $11
        oom_score = "N/A"
        
        if (system("test -f /proc/" pid "/oom_score") == 0) {
            "cat /proc/" pid "/oom_score" | getline oom_score
        }
        
        printf "PID: %-6s OOM Score: %-6s %s\n", pid, oom_score, cmd
    }' | sort -k3 -rn | head -20
}

# Ajuster le score OOM d'un processus
adjust_oom_score() {
    local pid="$1"
    local score="${2:-0}"  # -1000 à 1000
    
    echo "$score" | sudo tee "/proc/$pid/oom_score_adj"
    echo "Score OOM ajusté pour PID $pid"
}
```

### Fuites mémoire

**Détection de fuites mémoire** :
```bash
#!/bin/bash
# Détection de fuites mémoire

# Surveiller la croissance mémoire d'un processus
monitor_process_memory() {
    local pid="$1"
    local interval="${2:-5}"
    
    while kill -0 "$pid" 2>/dev/null; do
        local rss=$(ps -o rss= -p "$pid" 2>/dev/null)
        if [ -n "$rss" ]; then
            echo "$(date): RSS: $((rss / 1024)) MB"
        fi
        sleep "$interval"
    done
}

# Trouver les processus consommant le plus de mémoire
find_memory_hogs() {
    ps aux --sort=-%mem | head -10 | \
    awk 'NR==1 {print $0} NR>1 {
        rss_mb = $6 / 1024
        printf "%-8s %6.1f%% %8.1f MB %s\n", $2, $4, rss_mb, $11
    }'
}
```

## Performance et tuning

### Optimisation du swap

**Tuning du swap** :
```bash
#!/bin/bash
# Optimisation du swap

# 1. Réduire swappiness pour systèmes avec beaucoup de RAM
optimize_swappiness() {
    local target="${1:-10}"
    
    sudo sysctl -w vm.swappiness="$target"
    echo "vm.swappiness=$target" | sudo tee -a /etc/sysctl.d/99-swap.conf
    echo "Swappiness réglé à $target"
}

# 2. Utiliser swapfile sur SSD (plus rapide)
# 3. Éviter le swap si possible (systèmes avec RAM suffisante)
# 4. Monitorer l'utilisation du swap régulièrement

# Script d'optimisation complète
optimize_memory_settings() {
    local swappiness="${1:-10}"
    local dirty_ratio="${2:-15}"
    local dirty_bg_ratio="${3:-5}"
    
    cat << EOF | sudo tee /etc/sysctl.d/99-memory-optimization.conf
# Optimisation mémoire
vm.swappiness=$swappiness
vm.dirty_ratio=$dirty_ratio
vm.dirty_background_ratio=$dirty_bg_ratio
vm.vfs_cache_pressure=50
EOF
    
    sudo sysctl -p /etc/sysctl.d/99-memory-optimization.conf
    echo "Paramètres mémoire optimisés"
}
```

### Cache et buffers

**Gestion du cache** :
```bash
#!/bin/bash
# Gestion du cache

# Vider le cache de pages (attention : peut ralentir temporairement)
clear_page_cache() {
    echo "Vidage du cache de pages..."
    sudo sync
    echo 3 | sudo tee /proc/sys/vm/drop_caches
    
    echo "Cache vidé"
    free -h
}

# Vider seulement le cache de pages (sans buffers/inodes)
clear_page_cache_only() {
    sudo sync
    echo 1 | sudo tee /proc/sys/vm/drop_caches
}

# Vider buffers et cache
clear_buffers_and_cache() {
    sudo sync
    echo 2 | sudo tee /proc/sys/vm/drop_caches
}

# Statistiques du cache
cache_statistics() {
    echo "=== Statistiques du cache ==="
    grep -E "^(Cached|Buffers|Dirty|Writeback)" /proc/meminfo
}
```

## Scripts d'automatisation

### Script complet de gestion

**Gestionnaire de mémoire et swap** :
```bash
#!/bin/bash
# Gestionnaire complet de mémoire et swap

set -euo pipefail

memory_manager() {
    local action="$1"
    shift
    
    case "$action" in
        status)
            show_memory_status
            ;;
        swap-create)
            create_swap "$@"
            ;;
        swap-remove)
            remove_swap "$@"
            ;;
        optimize)
            optimize_memory "$@"
            ;;
        monitor)
            monitor_memory_continuous "$@"
            ;;
        *)
            echo "Usage: memory_manager {status|swap-create|swap-remove|optimize|monitor}"
            exit 1
            ;;
    esac
}

show_memory_status() {
    echo "=== État de la mémoire ==="
    free -h
    echo ""
    echo "=== Swap actif ==="
    swapon --show
    echo ""
    echo "=== Paramètres sysctl ==="
    sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio
}

create_swap() {
    local size="${1:-2G}"
    local file="${2:-/swapfile}"
    
    if [ -f "$file" ]; then
        echo "Fichier swap existe déjà: $file"
        return 1
    fi
    
    echo "Création du swap $size dans $file..."
    sudo fallocate -l "$size" "$file"
    sudo chmod 600 "$file"
    sudo mkswap "$file"
    sudo swapon "$file"
    
    # Ajouter à fstab
    if ! grep -q "$file" /etc/fstab; then
        echo "$file none swap sw 0 0" | sudo tee -a /etc/fstab
    fi
    
    echo "Swap créé et activé"
}

remove_swap() {
    local file="${1:-/swapfile}"
    
    if [ ! -f "$file" ]; then
        echo "Fichier swap non trouvé: $file"
        return 1
    fi
    
    echo "Suppression du swap $file..."
    sudo swapoff "$file"
    sudo sed -i "\|$file|d" /etc/fstab
    sudo rm "$file"
    
    echo "Swap supprimé"
}

# Utilisation
# memory_manager status
# memory_manager swap-create 4G /swapfile
# memory_manager optimize
```

## Conclusion

La gestion de la mémoire et du swap est essentielle pour maintenir un système Linux performant et stable. En comprenant comment Linux gère la mémoire virtuelle, en configurant correctement le swap, et en optimisant les paramètres système, vous pouvez créer un environnement qui utilise efficacement les ressources disponibles.

Un système bien configuré utilise la RAM de manière optimale, n'utilise le swap que lorsque nécessaire, et évite les problèmes de mémoire qui pourraient dégrader les performances ou causer des crashes. La surveillance régulière et l'optimisation proactive garantissent que votre système reste performant même sous charge importante.

Dans le chapitre suivant, nous explorerons la gestion des services système avec systemctl, découvrant comment administrer les services Linux modernes de manière efficace et professionnelle.
