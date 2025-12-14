# Chapitre 58 - Optimisation des performances

## Table des matières
- [Introduction](#introduction)
- [Diagnostic des performances](#diagnostic-des-performances)
- [Optimisation du kernel](#optimisation-du-kernel)
- [Optimisation mémoire](#optimisation-mémoire)
- [Optimisation I/O](#optimisation-io)
- [Optimisation réseau](#optimisation-réseau)
- [Profiling et benchmarking](#profiling-et-benchmarking)
- [Outils de monitoring](#outils-de-monitoring)
- [Tuning système](#tuning-système)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

L'optimisation des performances est l'art de faire fonctionner un système Linux au maximum de son potentiel. Au-delà de la simple surveillance, il s'agit d'identifier les goulots d'étranglement, d'ajuster finement les paramètres système, et de mesurer l'impact des modifications pour atteindre des performances optimales.

Imaginez l'optimisation des performances comme le réglage d'un moteur de course : chaque paramètre compte, chaque ajustement peut faire la différence entre la victoire et la défaite. Dans un système Linux, cela signifie comprendre comment le kernel gère les ressources, comment les applications interagissent avec le matériel, et comment identifier les points d'amélioration.

## Diagnostic des performances

### Identification des goulots d'étranglement

**Outils de diagnostic** :
```bash
#!/bin/bash
# Diagnostic complet des performances

# CPU
top
htop
vmstat 1
mpstat -P ALL 1

# Mémoire
free -h
vmstat -s
cat /proc/meminfo

# I/O Disque
iostat -x 1
iotop
sar -d 1

# Réseau
iftop
nethogs
sar -n DEV 1
```

## Optimisation du kernel

### Paramètres sysctl

**Tuning kernel** :
```bash
#!/bin/bash
# Optimisation kernel via sysctl

# /etc/sysctl.conf

# Optimisation réseau
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Optimisation mémoire
vm.swappiness = 10
vm.dirty_ratio = 15
vm.dirty_background_ratio = 5

# Optimisation fichiers
fs.file-max = 2097152
fs.nr_open = 1048576

# Appliquer
sysctl -p
```

## Optimisation mémoire

### Gestion de la mémoire

**Tuning mémoire** :
```bash
#!/bin/bash
# Optimisation mémoire

# Swappiness
echo 10 > /proc/sys/vm/swappiness

# Cache pressure
echo 50 > /proc/sys/vm/vfs_cache_pressure

# Overcommit
echo 1 > /proc/sys/vm/overcommit_memory
```

## Optimisation I/O

### Tuning des systèmes de fichiers

**Optimisation I/O** :
```bash
#!/bin/bash
# Optimisation I/O

# Mount options pour performance
mount -o noatime,nodiratime /dev/sda1 /mnt

# I/O scheduler
echo deadline > /sys/block/sda/queue/scheduler

# Queue depth
echo 1024 > /sys/block/sda/queue/nr_requests
```

## Optimisation réseau

### Tuning réseau

**Optimisation réseau** :
```bash
#!/bin/bash
# Optimisation réseau

# TCP tuning
echo 'net.core.somaxconn = 1024' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_max_syn_backlog = 2048' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_fin_timeout = 30' >> /etc/sysctl.conf

sysctl -p
```

## Profiling et benchmarking

### Outils de profiling

**Profiling avec perf** :
```bash
#!/bin/bash
# Profiling avec perf

# Installer perf
sudo apt install linux-perf

# Profiler un processus
perf record -g ./myprogram
perf report

# Statistiques système
perf stat ./myprogram
```

## Outils de monitoring

### Monitoring continu

**Scripts de monitoring** :
```bash
#!/bin/bash
# Monitoring des performances

monitor_performance() {
    while true; do
        echo "=== $(date) ==="
        echo "CPU:"
        top -bn1 | grep "Cpu(s)"
        echo "Mémoire:"
        free -h
        echo "I/O:"
        iostat -x 1 1
        sleep 60
    done
}
```

## Tuning système

### Optimisation globale

**Script de tuning** :
```bash
#!/bin/bash
# Script de tuning système

apply_performance_tuning() {
    # Kernel parameters
    sysctl -w vm.swappiness=10
    sysctl -w vm.dirty_ratio=15
    
    # I/O scheduler
    for disk in /sys/block/sd*/queue/scheduler; do
        echo deadline > "$disk"
    done
    
    # Network
    sysctl -w net.core.somaxconn=1024
}
```

## Scripts d'automatisation

**Gestionnaire de performance** :
```bash
#!/bin/bash
# Gestionnaire de performance

performance_manager() {
    local action="$1"
    
    case "$action" in
        diagnose)
            diagnose_performance
            ;;
        optimize)
            apply_optimizations
            ;;
        monitor)
            monitor_performance
            ;;
        *)
            echo "Usage: $0 {diagnose|optimize|monitor}"
            ;;
    esac
}
```

## Conclusion

L'optimisation des performances est un processus continu qui nécessite une compréhension approfondie du système Linux et de ses interactions avec le matériel. En utilisant les bons outils de diagnostic, en ajustant finement les paramètres système, et en mesurant l'impact des modifications, vous pouvez transformer un système fonctionnel en une machine hautement performante.

