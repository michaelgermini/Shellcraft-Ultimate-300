# Chapitre 24 - Processus

## Table des matières
- [Introduction](#introduction)
- [Concepts fondamentaux des processus](#concepts-fondamentaux-des-processus)
- [Commandes avancées de gestion des processus](#commandes-avancées-de-gestion-des-processus)
- [États des processus et transitions](#états-des-processus-et-transitions)
- [Hiérarchie et parenté des processus](#hiérarchie-et-parenté-des-processus)
- [Signaux et communication inter-processus](#signaux-et-communication-inter-processus)
- [Processus en arrière-plan et détachement](#processus-en-arrière-plan-et-détachement)
- [Limites et ressources des processus](#limites-et-ressources-des-processus)
- [Le système de fichiers /proc](#le-système-de-fichiers-proc)
- [cgroups et contrôle des ressources](#cgroups-et-contrôle-des-ressources)
- [Namespaces et isolation](#namespaces-et-isolation)
- [Dépannage et debugging avancé](#dépannage-et-debugging-avancé)
- [Monitoring et observabilité](#monitoring-et-observabilité)
- [Conclusion](#conclusion)

## Introduction

Les processus constituent l'âme d'un système Linux : chaque programme en cours d'exécution, chaque service système, chaque tâche automatisée est un processus. Comprendre leur fonctionnement, leur gestion et leur interaction est essentiel pour maîtriser l'administration système et le débogage avancé.

Imaginez les processus comme les acteurs d'une pièce de théâtre complexe : chacun joue son rôle, communique avec les autres, consomme des ressources, et peut être interrompu ou relancé selon les besoins de la performance globale. Mais dans un système Linux moderne, cette métaphore s'étend : les processus vivent dans des espaces isolés (namespaces), sont organisés en groupes (cgroups), et chaque action peut être tracée et analysée via le système de fichiers /proc.

## Concepts fondamentaux des processus

### Définition et caractéristiques

**Un processus est** :
- Une instance d'un programme en cours d'exécution
- Un conteneur pour le code, les données et l'état d'exécution
- Identifié de manière unique par un PID (Process ID)
- Possédant un PPID (Parent Process ID) qui indique son créateur

**Informations essentielles** :
```bash
#!/bin/bash
# Obtenir les informations de base d'un processus

get_process_info() {
    local pid="${1:-$$}"  # PID courant par défaut
    
    echo "=== Informations du processus $pid ==="
    echo "PID: $pid"
    echo "PPID: $(ps -o ppid= -p "$pid")"
    echo "Utilisateur: $(ps -o user= -p "$pid")"
    echo "Groupe: $(ps -o group= -p "$pid")"
    echo "Commande: $(ps -o cmd= -p "$pid")"
    echo "État: $(ps -o stat= -p "$pid")"
    echo "Priorité: $(ps -o nice= -p "$pid")"
    echo "Utilisation CPU: $(ps -o %cpu= -p "$pid")%"
    echo "Utilisation mémoire: $(ps -o %mem= -p "$pid")%"
    echo "Temps CPU: $(ps -o time= -p "$pid")"
    echo "Temps de démarrage: $(ps -o lstart= -p "$pid")"
}

# Utilisation
get_process_info
get_process_info 1  # Informations sur init/systemd
```

### PID et espace de noms

**PIDs uniques** :
```bash
#!/bin/bash
# Comprendre l'espace de noms des PIDs

# PID dans le namespace global
echo "PID global: $$"

# PID dans un namespace (avec unshare)
# sudo unshare --pid --fork --mount-proc bash
# echo "PID dans namespace: $$"  # Commence à 1 dans le namespace
```

## Commandes avancées de gestion des processus

### ps : options avancées

**ps avec format personnalisé** :
```bash
#!/bin/bash
# Utilisation avancée de ps

# Format personnalisé avec colonnes spécifiques
ps -eo pid,ppid,user,group,ni,pri,stat,time,cmd --sort=-%cpu | head -20

# Informations détaillées sur threads
ps -eLf | grep nginx

# Processus avec arbre
ps -ejH | head -30

# Processus avec informations de sécurité
ps -eo pid,user,group,cap,capbnd,cmd | head -20

# Processus avec informations réseau
ps -eo pid,user,cmd --sort=-%mem | head -10

# Processus zombies uniquement
ps aux | awk '$8 ~ /^Z/ {print}'

# Processus par utilisateur avec totaux
ps -eo user,%cpu,%mem --no-headers | awk '
{
    cpu[$1] += $2
    mem[$1] += $3
}
END {
    for (u in cpu) print u, cpu[u] "% CPU", mem[u] "% MEM"
}' | sort -k2 -rn
```

**Scripts utilitaires avec ps** :
```bash
#!/bin/bash
# Scripts utilitaires basés sur ps

# Trouver le processus consommant le plus de CPU
top_cpu_process() {
    ps -eo pid,%cpu,cmd --sort=-%cpu --no-headers | head -1 | awk '{print $1}'
}

# Trouver le processus consommant le plus de mémoire
top_mem_process() {
    ps -eo pid,%mem,cmd --sort=-%mem --no-headers | head -1 | awk '{print $1}'
}

# Lister tous les processus d'un utilisateur avec totaux
user_process_summary() {
    local user="${1:-$USER}"
    
    echo "=== Processus de $user ==="
    ps -u "$user" -o pid,%cpu,%mem,cmd --sort=-%cpu
    echo ""
    echo "Totaux:"
    ps -u "$user" -o %cpu,%mem --no-headers | awk '
    {
        cpu += $1
        mem += $2
    }
    END {
        printf "CPU: %.1f%%\nMémoire: %.1f%%\n", cpu, mem
    }'
}

# Trouver les processus enfants d'un PID
find_children() {
    local pid="$1"
    
    echo "=== Processus enfants de $pid ==="
    ps --ppid "$pid" -o pid,cmd
}

# Trouver tous les descendants (récursif)
find_all_descendants() {
    local pid="$1"
    
    function recursive_find() {
        local parent="$1"
        ps --ppid "$parent" -o pid --no-headers | while read -r child; do
            echo "$child"
            recursive_find "$child"
        done
    }
    
    echo "$pid"
    recursive_find "$pid"
}
```

### top et htop : monitoring avancé

**top avancé** :
```bash
#!/bin/bash
# Utilisation avancée de top

# top en mode batch (pour scripts)
top -b -n 1 | head -20

# top avec délai personnalisé
top -d 2  # Mise à jour toutes les 2 secondes

# top pour un utilisateur spécifique
top -u www-data

# Sauvegarder la sortie de top
top -b -n 1 > /tmp/top_snapshot.txt

# top avec tri personnalisé
# Dans top: appuyer sur 'O' puis choisir la colonne de tri

# Script pour monitorer un processus spécifique
monitor_process() {
    local pid="$1"
    local duration="${2:-60}"  # secondes
    
    top -b -d 1 -p "$pid" -n "$duration" | \
    awk -v pid="$pid" '
    /^%Cpu/ {cpu_line=$0}
    /^[ ]*'$pid'/ {
        print cpu_line
        print $0
        print ""
    }'
}
```

**htop avancé** :
```bash
#!/bin/bash
# Configuration et utilisation avancée de htop

# Configuration htop personnalisée
configure_htop() {
    mkdir -p ~/.config/htop
    
    cat > ~/.config/htop/htoprc << 'EOF'
# Configuration htop
hide_kernel_threads=0
hide_userland_threads=0
hide_threads=0
tree_view=1
sort_key=46
sort_direction=1
EOF
    
    echo "Configuration htop sauvegardée"
}

# htop avec filtres
# Dans htop: appuyer sur 'F4' pour filtrer par nom
# 'F5' pour vue en arbre
# 'F6' pour trier
# 'F9' pour tuer un processus
```

### pstree : visualisation de la hiérarchie

**pstree avancé** :
```bash
#!/bin/bash
# Utilisation avancée de pstree

# Arbre complet avec PIDs
pstree -p

# Arbre avec noms complets de commandes
pstree -a

# Arbre avec informations utilisateur
pstree -u

# Arbre pour un processus spécifique
pstree -p 1234

# Arbre avec limites de profondeur
pstree -L 3  # Limiter à 3 niveaux

# Arbre avec highlight d'un processus
pstree -H 1234

# Arbre en ASCII art
pstree -A

# Script pour analyser l'arbre
analyze_process_tree() {
    local pid="${1:-1}"  # systemd par défaut
    
    echo "=== Analyse de l'arbre de processus ==="
    echo ""
    echo "Profondeur maximale:"
    pstree -p "$pid" | awk -F'(' '{print NF-1}' | sort -rn | head -1
    echo ""
    echo "Nombre de processus:"
    pstree -p "$pid" | grep -o '([0-9]*' | wc -l
    echo ""
    echo "Processus avec le plus d'enfants:"
    pstree -p "$pid" | awk -F'(' '{print $1}' | sort | uniq -c | sort -rn | head -5
}
```

## États des processus et transitions

### Les états détaillés

**États Linux** :
```bash
#!/bin/bash
# Analyse détaillée des états de processus

# R : Running (en cours d'exécution)
# S : Sleeping (interruptible)
# D : Disk sleep (uninterruptible - attente I/O disque)
# T : Stopped (arrêté par signal)
# t : Tracing stop (arrêté par debugger)
# Z : Zombie (terminé mais pas nettoyé)
# X : Dead (en cours de terminaison)

analyze_process_states() {
    echo "=== Répartition des états de processus ==="
    ps -eo stat --no-headers | \
    awk '{
        state = substr($0, 1, 1)
        states[state]++
    }
    END {
        for (s in states) {
            switch(s) {
                case "R": print "Running:", states[s]; break
                case "S": print "Sleeping:", states[s]; break
                case "D": print "Disk Sleep:", states[s]; break
                case "T": print "Stopped:", states[s]; break
                case "Z": print "Zombie:", states[s]; break
                default: print "Other (" s "):", states[s]
            }
        }
    }'
}

# Processus en attente I/O (D state)
find_io_wait_processes() {
    ps aux | awk '$8 ~ /^D/ {print $0}'
    echo ""
    echo "Ces processus sont en attente I/O et ne peuvent pas être interrompus"
}

# Processus zombies
find_zombies() {
    local zombies=$(ps aux | awk '$8 ~ /^Z/ {print $2}' | wc -l)
    
    if [ "$zombies" -gt 0 ]; then
        echo "⚠️  $zombies processus zombie(s) détecté(s):"
        ps aux | awk '$8 ~ /^Z/ {print $0}'
    else
        echo "✓ Aucun processus zombie"
    fi
}
```

### Transitions d'états

**Cycle de vie complet** :
```bash
#!/bin/bash
# Visualiser les transitions d'états

# Création → Running → Sleeping → Running → Terminated
#     ↓         ↓         ↓         ↓         ↓
#   fork()   I/O      I/O       CPU      exit()
#                attente  complété

demonstrate_state_transitions() {
    # Créer un processus qui passe par différents états
    (
        echo "Processus créé (fork)"
        sleep 1 &
        echo "Processus en attente I/O (sleep)"
        cat /dev/zero > /dev/null &
        echo "Processus utilisant CPU"
        while true; do
            :  # Boucle CPU-intensive
        done &
    ) &
    
    local parent_pid=$!
    sleep 2
    
    echo "=== États des processus créés ==="
    ps -eo pid,stat,cmd | grep -E "($parent_pid|sleep|cat|while)"
    
    # Nettoyer
    pkill -P "$parent_pid"
    kill "$parent_pid" 2>/dev/null
}
```

## Hiérarchie et parenté des processus

### Relations parent-enfant avancées

**Analyse de la hiérarchie** :
```bash
#!/bin/bash
# Analyse avancée de la hiérarchie

# Trouver tous les ancêtres d'un processus
find_ancestors() {
    local pid="$1"
    
    echo "=== Ancêtres de $pid ==="
    while [ "$pid" != "1" ] && [ -n "$pid" ]; do
        local ppid=$(ps -o ppid= -p "$pid" 2>/dev/null | tr -d ' ')
        if [ -n "$ppid" ]; then
            echo "PID $pid -> PPID $ppid ($(ps -o cmd= -p "$ppid"))"
            pid="$ppid"
        else
            break
        fi
    done
}

# Trouver le processus racine d'une session
find_session_leader() {
    local pid="$1"
    
    # Remonter jusqu'au leader de session
    while true; do
        local sid=$(ps -o sid= -p "$pid" 2>/dev/null | tr -d ' ')
        local leader=$(ps -eo sid,pid,cmd | awk -v sid="$sid" '$1 == sid && $2 == sid {print $2}')
        
        if [ -n "$leader" ]; then
            echo "Leader de session: $leader"
            break
        fi
        
        local ppid=$(ps -o ppid= -p "$pid" 2>/dev/null | tr -d ' ')
        if [ "$ppid" = "1" ] || [ -z "$ppid" ]; then
            break
        fi
        pid="$ppid"
    done
}

# Analyser les processus orphelins
find_orphans() {
    echo "=== Processus orphelins (PPID != 1 mais parent inexistant) ==="
    ps -eo pid,ppid,cmd | awk '
    NR > 1 {
        pid = $1
        ppid = $2
        if (ppid != 1) {
            # Vérifier si le parent existe
            cmd = "ps -p " ppid " > /dev/null 2>&1"
            if (system(cmd) != 0) {
                print "Orphelin: PID", pid, "PPID", ppid, "inexistant"
            }
        }
    }'
}
```

### Sessions et groupes de processus

**Gestion des sessions** :
```bash
#!/bin/bash
# Gestion avancée des sessions

# Créer une nouvelle session
create_new_session() {
    setsid bash -c 'echo "Nouvelle session: $$ (SID: $(ps -o sid= -p $$))"'
}

# Lister les processus par session
list_by_session() {
    echo "=== Processus groupés par session ==="
    ps -eo sid,pid,cmd | awk '
    NR > 1 {
        sessions[$1] = sessions[$1] "\n  PID " $2 ": " substr($0, index($0,$3))
    }
    END {
        for (sid in sessions) {
            print "Session", sid ":" sessions[sid]
            print ""
        }
    }'
}

# Détacher un processus de son terminal
detach_from_terminal() {
    local pid="$1"
    
    # Utiliser disown ou nohup
    disown "$pid" 2>/dev/null || {
        echo "Utilisation de nohup pour détacher"
        nohup sleep 3600 > /dev/null 2>&1 &
        echo "Processus détaché: $!"
    }
}
```

## Signaux et communication inter-processus

### Signaux avancés

**Gestion complète des signaux** :
```bash
#!/bin/bash
# Gestion avancée des signaux

# Liste complète des signaux
list_all_signals() {
    kill -l | tr ' ' '\n' | nl -v 1
}

# Envoyer un signal à un processus
send_signal() {
    local signal="$1"  # Nom ou numéro
    local pid="$2"
    
    kill -s "$signal" "$pid" 2>/dev/null && \
        echo "Signal $signal envoyé à PID $pid" || \
        echo "Échec d'envoi du signal"
}

# Attendre qu'un processus termine
wait_for_process() {
    local pid="$1"
    local timeout="${2:-60}"  # secondes
    
    local elapsed=0
    while kill -0 "$pid" 2>/dev/null && [ $elapsed -lt $timeout ]; do
        sleep 1
        elapsed=$((elapsed + 1))
        echo -n "."
    done
    echo ""
    
    if kill -0 "$pid" 2>/dev/null; then
        echo "Timeout: processus toujours actif"
        return 1
    else
        echo "Processus terminé"
        return 0
    fi
}

# Gestion propre des signaux dans un script
signal_handler_example() {
    # Fonction de nettoyage
    cleanup() {
        echo ""
        echo "Nettoyage en cours..."
        # Fermer les fichiers, arrêter les services, etc.
        exit 0
    }
    
    # Capturer les signaux
    trap cleanup SIGINT SIGTERM EXIT
    
    echo "Script en cours (Ctrl+C pour arrêter proprement)"
    while true; do
        sleep 1
    done
}
```

### Communication inter-processus avancée

**Signaux personnalisés** :
```bash
#!/bin/bash
# Utilisation de signaux personnalisés

# SIGUSR1 et SIGUSR2 pour communication personnalisée
setup_custom_signal_handler() {
    handler() {
        case "$1" in
            SIGUSR1)
                echo "Signal personnalisé 1 reçu"
                # Action personnalisée
                ;;
            SIGUSR2)
                echo "Signal personnalisé 2 reçu"
                # Autre action personnalisée
                ;;
        esac
    }
    
    trap 'handler SIGUSR1' SIGUSR1
    trap 'handler SIGUSR2' SIGUSR2
    
    echo "Handlers configurés. PID: $$"
    echo "Envoyer: kill -USR1 $$ ou kill -USR2 $$"
    
    # Attendre les signaux
    while true; do
        sleep 1
    done
}
```

## Processus en arrière-plan et détachement

### Gestion avancée des jobs

**Jobs et contrôle** :
```bash
#!/bin/bash
# Gestion avancée des jobs

# Lister tous les jobs avec détails
list_jobs_detailed() {
    jobs -l
}

# Envoyer un job en arrière-plan
background_job() {
    long_running_task &
    local job_pid=$!
    echo "Job démarré en arrière-plan: $job_pid"
    disown "$job_pid"  # Détacher du shell
}

# Gérer plusieurs jobs
manage_multiple_jobs() {
    # Démarrer plusieurs jobs
    for i in {1..5}; do
        sleep 10 &
        echo "Job $i démarré: $!"
    done
    
    # Attendre tous les jobs
    wait
    
    echo "Tous les jobs terminés"
}

# Jobs avec timeout
job_with_timeout() {
    local timeout="${1:-10}"
    local command="${2:-sleep 100}"
    
    (
        eval "$command" &
        local pid=$!
        
        sleep "$timeout"
        
        if kill -0 "$pid" 2>/dev/null; then
            echo "Timeout atteint, arrêt du processus"
            kill "$pid" 2>/dev/null
            wait "$pid" 2>/dev/null
            return 124  # Code de retour timeout
        else
            wait "$pid"
            return $?
        fi
    )
}
```

### Détachement complet

**Détachement avec nohup et screen/tmux** :
```bash
#!/bin/bash
# Techniques de détachement avancées

# Détachement avec nohup
detach_with_nohup() {
    nohup long_running_script.sh > output.log 2>&1 &
    echo "Processus détaché avec nohup: $!"
    echo "Sortie dans output.log"
}

# Détachement avec screen
detach_with_screen() {
    screen -dmS mysession bash -c 'long_running_script.sh; exec bash'
    echo "Session screen créée: mysession"
    echo "Rattacher avec: screen -r mysession"
}

# Détachement avec tmux
detach_with_tmux() {
    tmux new-session -d -s mysession 'long_running_script.sh'
    echo "Session tmux créée: mysession"
    echo "Rattacher avec: tmux attach -t mysession"
}

# Détachement complet (double fork)
complete_detach() {
    (
        (
            # Double fork pour détachement complet
            long_running_daemon.sh &
        ) &
    ) &
    
    echo "Processus complètement détaché"
}
```

## Limites et ressources des processus

### ulimit et limites système

**Gestion des limites** :
```bash
#!/bin/bash
# Gestion avancée des limites

# Afficher toutes les limites
show_all_limits() {
    ulimit -a
}

# Définir des limites spécifiques
set_limits() {
    # Limite de fichiers ouverts
    ulimit -n 4096
    
    # Limite de processus
    ulimit -u 512
    
    # Limite de mémoire virtuelle (KB)
    ulimit -v 2097152  # 2GB
    
    # Limite de taille de fichier (KB)
    ulimit -f 1048576  # 1GB
    
    echo "Limites définies"
}

# Vérifier les limites d'un processus
check_process_limits() {
    local pid="$1"
    
    echo "=== Limites du processus $pid ==="
    cat "/proc/$pid/limits"
}

# Augmenter les limites pour un script
increase_limits_for_script() {
    ulimit -n 8192
    ulimit -u 1024
    
    # Exécuter le script avec les nouvelles limites
    "$@"
}
```

### Monitoring des ressources

**Surveillance des ressources** :
```bash
#!/bin/bash
# Surveillance avancée des ressources

# Utilisation mémoire détaillée
monitor_memory() {
    local pid="$1"
    
    while kill -0 "$pid" 2>/dev/null; do
        local mem=$(ps -o rss= -p "$pid" 2>/dev/null | tr -d ' ')
        local mem_mb=$((mem / 1024))
        echo "$(date): Mémoire: ${mem_mb}MB"
        sleep 5
    done
}

# Utilisation CPU détaillée
monitor_cpu() {
    local pid="$1"
    local duration="${2:-60}"
    
    local start_time=$(date +%s)
    local start_cpu=$(ps -o time= -p "$pid" | awk -F: '{print $1*3600+$2*60+$3}')
    
    sleep "$duration"
    
    local end_time=$(date +%s)
    local end_cpu=$(ps -o time= -p "$pid" | awk -F: '{print $1*3600+$2*60+$3}')
    
    local elapsed=$((end_time - start_time))
    local cpu_used=$((end_cpu - start_cpu))
    local cpu_percent=$((cpu_used * 100 / elapsed))
    
    echo "Utilisation CPU sur ${duration}s: ${cpu_percent}%"
}

# Fichiers ouverts
list_open_files() {
    local pid="$1"
    
    echo "=== Fichiers ouverts par PID $pid ==="
    lsof -p "$pid" 2>/dev/null || {
        echo "Utilisation de /proc:"
        ls -l "/proc/$pid/fd/" 2>/dev/null | tail -n +2
    }
}
```

## Le système de fichiers /proc

### Exploration de /proc

**Informations détaillées depuis /proc** :
```bash
#!/bin/bash
# Exploration avancée de /proc

# Informations complètes d'un processus
get_proc_info() {
    local pid="$1"
    local proc_dir="/proc/$pid"
    
    if [ ! -d "$proc_dir" ]; then
        echo "Processus $pid n'existe pas"
        return 1
    fi
    
    echo "=== Informations depuis /proc/$pid ==="
    echo ""
    echo "Commande: $(cat "$proc_dir/comm")"
    echo "Commande complète: $(cat "$proc_dir/cmdline" | tr '\0' ' ')"
    echo "État: $(cat "$proc_dir/status" | grep State)"
    echo "PPID: $(cat "$proc_dir/status" | grep PPid | awk '{print $2}')"
    echo "UID/GID: $(cat "$proc_dir/status" | grep -E '^Uid|^Gid')"
    echo "Mémoire:"
    cat "$proc_dir/status" | grep -E 'VmSize|VmRSS|VmData|VmStk'
    echo ""
    echo "Limites:"
    cat "$proc_dir/limits"
    echo ""
    echo "Statistiques CPU:"
    cat "$proc_dir/stat" | awk '{print "PID:", $1, "État:", $3, "utime:", $14, "stime:", $15}'
}

# Environnement d'un processus
get_process_environment() {
    local pid="$1"
    
    echo "=== Environnement du processus $pid ==="
    cat "/proc/$pid/environ" | tr '\0' '\n' | sort
}

# CWD d'un processus
get_process_cwd() {
    local pid="$1"
    
    readlink "/proc/$pid/cwd" 2>/dev/null || echo "Non accessible"
}

# Mappings mémoire
get_memory_mappings() {
    local pid="$1"
    
    echo "=== Mappings mémoire du processus $pid ==="
    cat "/proc/$pid/maps" | head -20
}

# Statistiques système
get_system_stats() {
    echo "=== Statistiques système depuis /proc ==="
    echo ""
    echo "Uptime: $(cat /proc/uptime | awk '{print $1 " secondes"}')"
    echo "Load average: $(cat /proc/loadavg)"
    echo ""
    echo "Mémoire:"
    cat /proc/meminfo | grep -E 'MemTotal|MemFree|MemAvailable|SwapTotal|SwapFree'
    echo ""
    echo "CPU:"
    cat /proc/cpuinfo | grep -E 'processor|model name|cpu MHz' | head -6
}
```

## cgroups et contrôle des ressources

### cgroups v2

**Gestion avec cgroups v2** :
```bash
#!/bin/bash
# Gestion avancée des cgroups v2

# Vérifier si cgroups v2 est activé
check_cgroup_v2() {
    if [ -f /sys/fs/cgroup/cgroup.controllers ]; then
        echo "cgroups v2 activé"
        cat /sys/fs/cgroup/cgroup.controllers
    else
        echo "cgroups v1 ou non activé"
    fi
}

# Créer un cgroup personnalisé
create_custom_cgroup() {
    local cgroup_name="$1"
    local cgroup_path="/sys/fs/cgroup/$cgroup_name"
    
    sudo mkdir -p "$cgroup_path"
    echo "Cgroup créé: $cgroup_path"
}

# Limiter la CPU d'un processus
limit_cpu() {
    local pid="$1"
    local cpu_max="${2:-50000}"  # 50% par défaut (100000 = 100%)
    local cgroup_name="cpu_limit_$$"
    
    local cgroup_path="/sys/fs/cgroup/$cgroup_name"
    sudo mkdir -p "$cgroup_path"
    
    echo "$cpu_max" | sudo tee "$cgroup_path/cpu.max" > /dev/null
    echo "$pid" | sudo tee "$cgroup_path/cgroup.procs" > /dev/null
    
    echo "CPU limité à ${cpu_max}00% pour PID $pid"
}

# Limiter la mémoire
limit_memory() {
    local pid="$1"
    local mem_limit="${2:-512M}"
    local cgroup_name="memory_limit_$$"
    
    local cgroup_path="/sys/fs/cgroup/$cgroup_name"
    sudo mkdir -p "$cgroup_path"
    
    # Convertir en bytes
    local mem_bytes
    case "$mem_limit" in
        *K) mem_bytes=$((${mem_limit%K} * 1024));;
        *M) mem_bytes=$((${mem_limit%M} * 1024 * 1024));;
        *G) mem_bytes=$((${mem_limit%G} * 1024 * 1024 * 1024));;
        *) mem_bytes="$mem_limit";;
    esac
    
    echo "$mem_bytes" | sudo tee "$cgroup_path/memory.max" > /dev/null
    echo "$pid" | sudo tee "$cgroup_path/cgroup.procs" > /dev/null
    
    echo "Mémoire limitée à $mem_limit pour PID $pid"
}
```

## Namespaces et isolation

### Namespaces Linux

**Comprendre les namespaces** :
```bash
#!/bin/bash
# Exploration des namespaces

# Lister les namespaces d'un processus
list_process_namespaces() {
    local pid="$1"
    
    echo "=== Namespaces du processus $pid ==="
    ls -l "/proc/$pid/ns/"
}

# Entrer dans un namespace
enter_namespace() {
    local pid="$1"
    local ns_type="${2:-pid}"  # pid, net, mnt, etc.
    
    sudo nsenter -t "$pid" -"${ns_type:0:1}" bash
}

# Créer un namespace isolé
create_isolated_namespace() {
    # PID namespace isolé
    sudo unshare --pid --fork --mount-proc bash
    
    # Dans ce namespace:
    # - Le processus aura PID 1
    # - Ne verra que ses propres processus
    # - Système de fichiers /proc isolé
}
```

## Dépannage et debugging avancé

### strace et ltrace

**Tracing avancé** :
```bash
#!/bin/bash
# Utilisation avancée de strace

# Tracer les appels système d'un processus
trace_syscalls() {
    local pid="$1"
    
    sudo strace -p "$pid" -e trace=all
}

# Tracer seulement les appels de fichiers
trace_file_operations() {
    local command="$1"
    
    strace -e trace=open,openat,read,write "$command"
}

# Tracer avec statistiques
trace_with_stats() {
    local command="$1"
    
    strace -c "$command"
}

# Tracer les signaux
trace_signals() {
    local pid="$1"
    
    strace -p "$pid" -e trace=signal
}

# ltrace pour les appels de bibliothèque
trace_library_calls() {
    local command="$1"
    
    ltrace "$command"
}
```

### Debugging avec gdb

**Attacher gdb à un processus** :
```bash
#!/bin/bash
# Debugging avec gdb

# Attacher gdb à un processus en cours
attach_gdb() {
    local pid="$1"
    
    sudo gdb -p "$pid"
    
    # Dans gdb:
    # (gdb) info registers
    # (gdb) backtrace
    # (gdb) continue
}

# Analyser un core dump
analyze_core_dump() {
    local core_file="$1"
    local binary="$2"
    
    gdb "$binary" "$core_file"
    
    # Dans gdb:
    # (gdb) backtrace
    # (gdb) info registers
    # (gdb) print variable_name
}
```

## Monitoring et observabilité

### Scripts de monitoring complets

**Monitoring système complet** :
```bash
#!/bin/bash
# Script de monitoring complet des processus

monitor_all_processes() {
    local interval="${1:-5}"  # secondes
    local duration="${2:-60}"  # secondes
    
    local end_time=$(($(date +%s) + duration))
    
    while [ $(date +%s) -lt $end_time ]; do
        clear
        echo "=== Monitoring des processus ==="
        echo "Temps: $(date)"
        echo ""
        
        echo "Top 5 CPU:"
        ps -eo pid,%cpu,cmd --sort=-%cpu --no-headers | head -5
        echo ""
        
        echo "Top 5 Mémoire:"
        ps -eo pid,%mem,cmd --sort=-%mem --no-headers | head -5
        echo ""
        
        echo "Processus zombies:"
        ps aux | awk '$8 ~ /^Z/ {print $0}' || echo "Aucun"
        echo ""
        
        echo "Load average: $(uptime | awk -F'load average:' '{print $2}')"
        
        sleep "$interval"
    done
}

# Utilisation
# monitor_all_processes 5 60
```

## Conclusion

La maîtrise avancée des processus Linux nécessite une compréhension approfondie de leur cycle de vie, de leurs états, de leur hiérarchie, et des mécanismes de contrôle disponibles. En utilisant les outils avancés comme ps, top, htop, strace, et en explorant /proc et les cgroups, vous pouvez diagnostiquer, optimiser et contrôler efficacement l'exécution des processus dans votre système.

Les processus ne sont pas des entités isolées mais font partie d'un écosystème complexe où chaque interaction peut être observée, chaque ressource peut être contrôlée, et chaque problème peut être diagnostiqué avec les bons outils et la bonne compréhension.

Dans le chapitre suivant, nous explorerons la gestion avancée des priorités avec nice, renice, et les mécanismes d'ordonnancement temps réel, découvrant comment optimiser la répartition des ressources CPU entre les différents processus.
