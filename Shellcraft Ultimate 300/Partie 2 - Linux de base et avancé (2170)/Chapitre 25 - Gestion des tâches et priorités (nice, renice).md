# Chapitre 25 - Gestion des tâches et priorités (nice, renice)

## Table des matières
- [Introduction](#introduction)
- [Comprendre la priorité des processus](#comprendre-la-priorité-des-processus)
- [nice : Lancer des processus avec une priorité définie](#nice--lancer-des-processus-avec-une-priorité-définie)
- [renice : Modifier la priorité des processus existants](#renice--modifier-la-priorité-des-processus-existants)
- [chrt : Ordonnancement temps réel](#chrt--ordonnancement-temps-réel)
- [taskset : Contrôle d'affinité CPU](#taskset--contrôle-daffinité-cpu)
- [cgcreate/cgset : Groupes de contrôle](#cgcreatecgset--groupes-de-contrôle)
- [ionice : Priorités d'I/O](#ionice--priorités-dio)
- [Stratégies de gestion des priorités](#stratégies-de-gestion-des-priorités)
- [Monitoring et débogage des priorités](#monitoring-et-débogage-des-priorités)
- [Automatisation de la gestion des priorités](#automatisation-de-la-gestion-des-priorités)
- [Cas d'usage pratiques](#cas-dusage-pratiques)
- [Conclusion](#conclusion)

## Introduction

La gestion des priorités est l'art subtil de répartir équitablement les ressources système entre les différents processus. Dans un environnement multi-tâches, tous les processus ne se valent pas : certains sont critiques pour l'expérience utilisateur, d'autres peuvent tourner en arrière-plan sans impact perceptible.

Imaginez les priorités comme les files d'attente dans un supermarché : les clients avec des achats urgents (processus haute priorité) passent devant ceux qui font leurs courses hebdomadaires (processus basse priorité). Mais contrairement au supermarché, le système d'exploitation ajuste dynamiquement ces priorités pour maintenir des performances optimales et une réactivité acceptable.

## Comprendre la priorité des processus

### Le système de priorité Linux

**Nice value** :
- Plage : -20 (priorité maximale) à +19 (priorité minimale)
- Défaut : 0 pour la plupart des processus
- Unité : Relative (pas de valeur absolue)

**Calcul de la priorité effective** :
```bash
# Priorité = (20 + nice_value) + priorité_statique
# nice -20 → priorité 0 (max)
# nice 0 → priorité 20 (normal)
# nice +19 → priorité 39 (min)
```

### Types d'ordonnancement

**Politiques d'ordonnancement** :
- **SCHED_OTHER** (défaut) : Ordonnancement proportionnel fair (CFS)
- **SCHED_FIFO** : Temps réel, premier arrivé premier servi
- **SCHED_RR** : Temps réel, round-robin
- **SCHED_BATCH** : Batch processing (minimise changements de contexte)
- **SCHED_IDLE** : Très basse priorité, seulement quand CPU inactif

### Facteurs influençant les priorités

**Priorité dynamique** :
```bash
# Le scheduler ajuste automatiquement selon :
# - Temps d'attente (priorité augmente si attente longue)
# - Utilisation CPU récente
# - Interactions utilisateur (processus X11/Wayland prioritaires)
# - I/O en cours
```

**Priorité statique (temps réel)** :
```bash
# Priorité fixe 1-99 (99 = maximale)
# Non affectée par le nice value
# Réservée aux processus critiques
```

## nice : Lancer des processus avec une priorité définie

### Syntaxe de base

**Lancement avec priorité** :
```bash
# Syntaxe générale
nice [OPTION] [NICE_VALUE] COMMAND [ARGS...]

# Priorité plus basse (plus "gentil")
nice -n 10 long_backup.sh

# Priorité plus haute (root seulement)
sudo nice -n -5 critical_process

# Valeur par défaut
nice command  # Équivalent à nice -n 10
```

### Exemples pratiques

**Tâches d'arrière-plan** :
```bash
# Compression en basse priorité
nice -n 15 tar -czf backup.tar.gz /home/user/

# Synchronisation réseau
nice -n 10 rsync -av /source /destination/

# Compilation lourde
nice -n 10 make -j$(nproc)
```

**Processus interactifs prioritaires** :
```bash
# Éditeur de texte prioritaire (root seulement)
sudo nice -n -10 vim important_file.txt

# Navigateur web réactif
sudo nice -n -5 firefox

# Terminal réactif
sudo nice -n -5 gnome-terminal
```

### Gestion des erreurs

**Limitations de nice** :
```bash
# Utilisateur normal ne peut qu'augmenter nice
nice -n -5 command  # Échouera pour utilisateur normal

# Root peut définir n'importe quelle valeur
sudo nice -n -20 critical_command

# Vérification des permissions
ulimit -e  # Limite de nice pour l'utilisateur
```

## renice : Modifier la priorité des processus existants

### Syntaxe et options

**Modification de priorité** :
```bash
# Syntaxe générale
renice [OPTIONS] PRIORITY [[-p] PID...] [[-g] PGRP...] [[-u] USER...]

# Par PID
renice +10 1234

# Par nom de processus
renice +5 -u alice

# Par groupe de processus
renice -5 -g developers
```

### Utilisations courantes

**Ajustement en cours d'exécution** :
```bash
# Identifier processus gourmand
ps aux --sort=-%cpu | head -5

# Diminuer sa priorité
renice +10 $(pgrep heavy_process)

# Augmenter priorité processus critique
sudo renice -10 $(pgrep critical_service)
```

**Gestion par utilisateur** :
```bash
# Tous les processus d'un utilisateur
renice +15 -u backup_user

# Utilisateur de développement prioritaire
sudo renice -5 -u developer

# Utilisateur invité basse priorité
sudo renice +15 -u guest
```

### Options avancées

**Mode verbose et vérification** :
```bash
# Afficher les changements
renice -v +10 1234

# Ne pas changer si nouvelle priorité plus basse
renice -n +5 1234  # Seulement si actuel > 5
```

## chrt : Ordonnancement temps réel

### Politiques temps réel

**SCHED_FIFO (First In, First Out)** :
```bash
# Processus temps réel haute priorité
sudo chrt --fifo -p 50 1234

# Lancer avec priorité temps réel
sudo chrt --fifo 50 critical_process
```

**SCHED_RR (Round Robin)** :
```bash
# Temps réel avec partage équitable
sudo chrt --rr -p 30 1234

# Intervalle de quantum personnalisé
sudo chrt --rr -T 10000000 30 process  # 10ms quantum
```

### Gestion avancée

**Inspection des politiques** :
```bash
# Voir la politique d'un processus
chrt -p 1234

# Lister tous les processus temps réel
ps -eo pid,comm,cls,pri | grep -E "(RR|FF)"
```

**Changement de politique** :
```bash
# Passer en batch processing
chrt --batch -p 0 1234

# Retour à l'ordonnancement normal
chrt --other -p 0 1234
```

## taskset : Contrôle d'affinité CPU

### Comprendre l'affinité CPU

**Affinité CPU** : Liaison d'un processus à des cœurs spécifiques
```bash
# Avantages :
# - Cache locality (meilleures performances)
# - Isolation (éviter interférence)
# - Débogage (processus sur CPU dédié)
```

### Utilisation de taskset

**Afficher l'affinité actuelle** :
```bash
# Affinité d'un processus
taskset -p 1234

# Affinité de tous les processus
ps -eo pid,comm,psr | head -10  # PSR = CPU actuel
```

**Définir l'affinité** :
```bash
# Lier à CPU 0 seulement
taskset -p 0x1 1234

# Lier à CPU 0 et 1
taskset -p 0x3 1234

# Masque hexadécimal :
# 0x1 = CPU 0
# 0x2 = CPU 1
# 0x4 = CPU 2
# 0x8 = CPU 3
# 0xF = CPU 0-3
```

### Applications pratiques

**Isolation de processus critiques** :
```bash
# Serveur web sur CPU dédiés
sudo taskset -c 0-3 systemctl start apache2

# Processus de backup sur CPU faible
taskset -c 4-7 nice -n 15 backup_script.sh
```

**Débogage de performance** :
```bash
# Tester sur CPU spécifique
taskset -c 2 perf record -p 1234

# Migration forcée
taskset -p 0x4 1234  # Déplacer vers CPU 2
```

## cgcreate/cgset : Groupes de contrôle

### Principe des cgroups

**Control Groups (cgroups)** : Hiérarchie de groupes de contrôle des ressources
```bash
# Ressources contrôlables :
# - CPU (shares, quota)
# - Mémoire (limits, swappiness)
# - I/O (priorités, limites)
# - Réseau (bandwidth, classes)
```

### Création et configuration

**Création de groupes** :
```bash
# Installation des outils
sudo apt install cgroup-tools

# Créer un groupe
sudo cgcreate -g cpu,memory:/mygroup

# Vérifier
ls /sys/fs/cgroup/cpu/mygroup/
ls /sys/fs/cgroup/memory/mygroup/
```

**Configuration des limites** :
```bash
# Partage CPU (1024 = 100%)
sudo cgset -r cpu.shares=512 mygroup  # 50% de CPU

# Limite mémoire
sudo cgset -r memory.limit_in_bytes=1G mygroup

# Limite I/O
sudo cgset -r blkio.weight=500 mygroup  # 50% de priorité I/O
```

### Utilisation des groupes

**Assignation de processus** :
```bash
# Ajouter un processus existant
sudo cgclassify -g cpu,memory:mygroup 1234

# Lancer dans un groupe
sudo cgexec -g cpu,memory:mygroup my_command

# Lister les processus d'un groupe
cat /sys/fs/cgroup/cpu/mygroup/tasks
```

### Gestion hiérarchique

**Sous-groupes** :
```bash
# Créer une hiérarchie
sudo cgcreate -g cpu:/parent
sudo cgcreate -g cpu:/parent/child1
sudo cgcreate -g cpu:/parent/child2

# Configuration héritée
sudo cgset -r cpu.shares=1024 parent
sudo cgset -r cpu.shares=512 parent/child1  # Hérite et partage
sudo cgset -r cpu.shares=512 parent/child2
```

## ionice : Priorités d'I/O

### Comprendre les I/O priorities

**Classes d'I/O** :
- **Idle** : Très basse priorité, seulement quand disque inactif
- **Best-effort** (défaut) : Priorité proportionnelle (0-7)
- **Real-time** : Haute priorité (0-7)

### Utilisation d'ionice

**Installation et syntaxe** :
```bash
# Installation
sudo apt install util-linux

# Syntaxe générale
ionice [OPTIONS] COMMAND

# Classes :
# -c 0 : None (pas de priorité)
# -c 1 : Real-time
# -c 2 : Best-effort
# -c 3 : Idle
```

### Exemples pratiques

**Processus I/O intensifs** :
```bash
# Backup en basse priorité I/O
ionice -c 3 nice -n 15 tar -czf backup.tar.gz /data/

# Synchronisation idle
ionice -c 3 rsync -av /source /destination/

# Processus critique haute priorité
sudo ionice -c 1 -n 0 critical_backup.sh
```

**Modification de processus existants** :
```bash
# Changer la classe d'un processus
sudo ionice -c 1 -p 1234

# Vérifier la priorité I/O
ionice -p 1234

# Lister par classe
ps ax -o pid,comm | xargs -n1 ionice -p 2>/dev/null | grep "best-effort"
```

## Stratégies de gestion des priorités

### Politiques par type d'usage

**Serveur web** :
```bash
# Processus web haute priorité
sudo chrt --rr -p 20 $(pgrep apache2)
sudo ionice -c 1 -p $(pgrep apache2)

# Base de données haute priorité CPU, normale I/O
sudo chrt --batch -p 0 $(pgrep mysqld)
sudo ionice -c 2 -p $(pgrep mysqld)
```

**Station de travail** :
```bash
# Applications interactives prioritaires
sudo nice -n -10 $(pgrep firefox)
sudo ionice -c 1 -p $(pgrep firefox)

# Tâches d'arrière-plan basses priorités
nice -n 15 ionice -c 3 backup_script.sh &
```

### Gestion automatique

**Script de configuration des priorités** :
```bash
#!/bin/bash
# set_priorities.sh

# Applications interactives
interactive_apps="firefox thunderbird code"
for app in $interactive_apps; do
    pids=$(pgrep "$app")
    if [ -n "$pids" ]; then
        sudo renice -5 $pids 2>/dev/null
        sudo ionice -c 1 -p $pids 2>/dev/null
    fi
done

# Services système
system_services="sshd systemd-journald"
for service in $system_services; do
    pids=$(pgrep "$service")
    if [ -n "$pids" ]; then
        sudo chrt --batch -p 0 $pids 2>/dev/null
    fi
done

# Tâches de fond
background_tasks="updatedb mlocate"
for task in $background_tasks; do
    pids=$(pgrep "$task")
    if [ -n "$pids" ]; then
        renice +10 $pids 2>/dev/null
        ionice -c 3 -p $pids 2>/dev/null
    fi
done
```

### Optimisation selon la charge système

**Adaptation dynamique** :
```bash
#!/bin/bash
# adaptive_priorities.sh

# Charger système (1 minute)
load=$(uptime | awk '{print $NF}')

# Ajustement selon la charge
if (( $(echo "$load > 2.0" | bc -l) )); then
    # Système chargé : prioriser processus critiques
    sudo renice -10 $(pgrep -f "apache|nginx")
    renice +5 $(pgrep -f "backup|sync")
elif (( $(echo "$load < 0.5" | bc -l) )); then
    # Système peu chargé : permettre aux tâches de fond
    renice 0 $(pgrep -f "backup|maintenance")
fi
```

## Monitoring et débogage des priorités

### Outils d'inspection

**Vérification des priorités** :
```bash
# Nice values
ps -eo pid,ni,comm | sort -k2 -n

# Politiques d'ordonnancement
ps -eo pid,comm,cls,pri | grep -v "TS"

# Affinité CPU
taskset -p $(pgrep process_name)

# Groupes de contrôle
cat /sys/fs/cgroup/cpu/user.slice/tasks

# Priorités I/O
ionice -p $(pgrep process_name)
```

### Diagnostic de performance

**Analyse des goulots** :
```bash
# Processus attendant CPU
ps aux | awk 'NR>1 {if($8=="D") print $0}'

# Utilisation CPU par priorité
ps -eo ni,%cpu,comm | sort -k1 -n | awk '{sum[$1]+=$2} END {for(i in sum) print i, sum[i]}'

# Latence d'I/O
sudo iotop -b -n 1 | head -10
```

### Logs et traçage

**Suivi des changements** :
```bash
#!/bin/bash
# priority_audit.sh

LOGFILE="/var/log/priority_changes.log"

log_priority_change() {
    local pid="$1"
    local old_nice="$2"
    local new_nice="$3"
    local user="$4"
    
    echo "$(date): PID $pid ($user): nice $old_nice → $new_nice" >> "$LOGFILE"
}

# Hook pour renice (nécessite modification de renice ou wrapper)
renice() {
    local new_nice="$1"
    shift
    
    for pid in "$@"; do
        local old_nice=$(ps -o ni -p "$pid" --no-headers 2>/dev/null || echo "N/A")
        local user=$(ps -o user -p "$pid" --no-headers 2>/dev/null || echo "N/A")
        
        /usr/bin/renice "$new_nice" "$pid" 2>/dev/null
        
        if [ $? -eq 0 ]; then
            log_priority_change "$pid" "$old_nice" "$new_nice" "$user"
        fi
    done
}
```

## Automatisation de la gestion des priorités

### Systemd et configuration persistante

**Configuration systemd** :
```bash
# /etc/systemd/system/high-priority.service
[Unit]
Description=High priority service
After=network.target

[Service]
Type=simple
User=myuser
Nice=-10
IOSchedulingClass=1
IOSchedulingPriority=0
CPUSchedulingPolicy=rr
CPUSchedulingPriority=20
ExecStart=/usr/local/bin/my_service

[Install]
WantedBy=multi-user.target
```

**cgroups persistants** :
```bash
# Création persistante
sudo cgcreate -g cpu,memory:/persistent_group
sudo cgset -r cpu.shares=512 persistent_group
sudo cgset -r memory.limit_in_bytes=2G persistent_group

# Configuration dans /etc/cgconfig.conf
group persistent_group {
    cpu {
        cpu.shares = 512;
    }
    memory {
        memory.limit_in_bytes = 2147483648;  # 2GB
    }
}
```

### Scripts de gestion automatique

**Démon de gestion des priorités** :
```bash
#!/bin/bash
# priority_manager.sh

CONFIG_FILE="/etc/priority_rules.conf"
INTERVAL=60  # Vérification chaque minute

# Fonction d'application des règles
apply_rules() {
    while IFS=: read -r pattern nice_value io_class io_priority; do
        # Ignorer commentaires et lignes vides
        [[ "$pattern" =~ ^# ]] && continue
        [ -z "$pattern" ] && continue
        
        # Trouver les PIDs correspondants
        pids=$(pgrep -f "$pattern")
        
        for pid in $pids; do
            # Appliquer nice
            if [ "$nice_value" != "-" ]; then
                renice "$nice_value" "$pid" 2>/dev/null
            fi
            
            # Appliquer I/O priority
            if [ "$io_class" != "-" ] && [ "$io_priority" != "-" ]; then
                ionice -c "$io_class" -n "$io_priority" -p "$pid" 2>/dev/null
            fi
        done
    done < "$CONFIG_FILE"
}

# Boucle principale
while true; do
    apply_rules
    sleep "$INTERVAL"
done
```

**Fichier de configuration** :
```bash
# /etc/priority_rules.conf
# pattern:nice:io_class:io_priority

# Applications interactives prioritaires
firefox:-5:1:0
code:-5:1:0

# Services système
apache2:-10:1:0
mysqld:-5:2:4

# Tâches de fond basses priorités
backup_script:15:3:7
updatedb:19:3:7
```

## Cas d'usage pratiques

### Serveur de développement

**Configuration optimisée** :
```bash
# IDE et outils de développement prioritaires
sudo renice -10 $(pgrep code)
sudo ionice -c 1 $(pgrep code)

# Services de développement
sudo chrt --batch -p 0 $(pgrep docker)
sudo taskset -c 0-3 $(pgrep docker)

# Compilation en arrière-plan basse priorité
nice -n 15 ionice -c 3 make -j$(nproc) &
```

### Serveur de production

**Politiques de production** :
```bash
# Services critiques
sudo chrt --rr -p 50 $(pgrep nginx)
sudo cgclassify -g cpu,memory:critical $(pgrep nginx)

# Base de données
sudo chrt --batch -p 0 $(pgrep mysqld)
sudo cgset -r memory.limit_in_bytes=8G database_group

# Monitoring et logging basse priorité
nice -n 10 ionice -c 3 $(pgrep rsyslog)
```

### Station de travail personnelle

**Équilibre interactivité/performance** :
```bash
# Applications utilisateur prioritaires
sudo nice -n -5 $(pgrep -f "chrome|firefox")
sudo ionice -c 1 $(pgrep -f "chrome|firefox")

# Jeux et applications multimédia
sudo chrt --rr -p 30 $(pgrep steam)

# Tâches de maintenance nocturnes
# Crontab
0 2 * * * nice -n 19 ionice -c 3 /usr/local/bin/daily_maintenance.sh
```

## Conclusion

La gestion des priorités sous Linux est un équilibre délicat entre performance, réactivité et équité. Les outils `nice`, `renice`, `chrt`, `taskset`, et les cgroups offrent un contrôle granulaire sur la répartition des ressources système.

Une stratégie de priorisation efficace nécessite de comprendre les besoins de chaque processus : les applications interactives nécessitent une réponse rapide, les tâches de fond peuvent tourner avec des priorités plus basses, et les services critiques demandent souvent des garanties temps réel.

L'automatisation de ces politiques à travers des scripts et des configurations systemd permet de maintenir des performances optimales sans intervention manuelle constante. Cette approche proactive transforme l'administration système d'une gestion réactive en un pilotage intelligent des ressources.

