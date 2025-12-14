# Chapitre 52 - Gestion des processus zombies

## Table des matières
- [Introduction](#introduction)
- [Nature des processus zombies](#nature-des-processus-zombies)
- [Cycle de vie et création](#cycle-de-vie-et-création)
- [Pourquoi les zombies apparaissent](#pourquoi-les-zombies-apparaissent)
- [Diagnostic et identification](#diagnostic-et-identification)
- [Détection automatique](#détection-automatique)
- [Prévention des zombies](#prévention-des-zombies)
- [Gestion correcte des processus enfants](#gestion-correcte-des-processus-enfants)
- [Nettoyage automatique](#nettoyage-automatique)
- [Gestionnaire de signaux SIGCHLD](#gestionnaire-de-signaux-sigchld)
- [Impact sur le système](#impact-sur-le-système)
- [Limites et ressources](#limites-et-ressources)
- [Outils de monitoring](#outils-de-monitoring)
- [Scripts de diagnostic](#scripts-de-diagnostic)
- [Résolution de problèmes](#résolution-de-problèmes)
- [Bonnes pratiques](#bonnes-pratiques)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Les processus zombies constituent l'un des phénomènes les plus mystérieux et problématiques de l'administration système UNIX. Ces processus "morts-vivants" peuvent accumuler des ressources système et créer des problèmes de performance si ils ne sont pas gérés correctement.

Imaginez les processus zombies comme des fantômes dans une maison hantée : ils sont techniquement morts mais refusent de disparaître complètement, occupant de l'espace et effrayant les autres processus jusqu'à ce qu'ils soient correctement exorcisés. Dans un système Linux, ces zombies sont des processus qui ont terminé leur exécution mais dont l'entrée dans la table des processus n'a pas été nettoyée car le processus parent n'a pas encore lu leur code de sortie via `wait()` ou `waitpid()`.

## Nature des processus zombies

### Définition technique

**Qu'est-ce qu'un zombie ?** :
```bash
#!/bin/bash
# Un processus zombie est un processus qui :
# 1. A terminé son exécution (exit)
# 2. N'a plus de code exécutable en mémoire
# 3. Conserve son entrée dans la table des processus
# 4. Attend que le parent lise son code de sortie

# État dans ps : Z (zombie) ou <defunct>
# Ne peut pas être tué (déjà mort)
# Consomme une entrée PID mais pas de CPU/mémoire
```

**Caractéristiques** :
- État : `Z` (zombie) dans `ps`
- Marqué comme `<defunct>` dans certains affichages
- Ne consomme pas de CPU ni de mémoire
- Consomme une entrée dans la table des processus
- Ne peut pas être tué (déjà terminé)
- Disparaît quand le parent appelle `wait()`

## Cycle de vie et création

### Création d'un zombie

**Scénario typique** :
```bash
#!/bin/bash
# Création intentionnelle d'un zombie pour démonstration

create_zombie() {
    # Créer un processus enfant
    (
        # Processus enfant
        sleep 1
        exit 0
    ) &
    
    local child_pid=$!
    echo "Processus enfant créé: $child_pid"
    
    # Ne pas attendre l'enfant (créer un zombie)
    sleep 2
    
    # Vérifier l'état
    ps -p "$child_pid" -o pid,stat,cmd
    # Statut devrait être Z (zombie)
}

# Exécuter
create_zombie
```

**Cycle de vie normal** :
```bash
#!/bin/bash
# Cycle de vie normal d'un processus

normal_lifecycle() {
    # 1. Création (fork)
    (
        # 2. Exécution
        echo "Processus enfant en cours d'exécution"
        sleep 1
        
        # 3. Terminaison
        exit 0
    ) &
    
    local child_pid=$!
    
    # 4. Parent attend l'enfant (évite le zombie)
    wait "$child_pid"
    local exit_code=$?
    
    echo "Processus enfant terminé avec le code: $exit_code"
    # Pas de zombie car wait() a été appelé
}
```

## Pourquoi les zombies apparaissent

### Causes communes

**Raisons principales** :
```bash
#!/bin/bash
# Causes communes des processus zombies

# 1. Parent ne gère pas SIGCHLD
#    - Signal envoyé quand un enfant termine
#    - Si ignoré, le parent ne sait pas que l'enfant est mort

# 2. Parent trop occupé
#    - Parent dans une boucle infinie
#    - Ne prend pas le temps d'appeler wait()

# 3. Parent mal programmé
#    - Oublie d'appeler wait()
#    - Gestion d'erreurs incomplète

# 4. Parent crashé avant wait()
#    - Enfant devient orphelin
#    - Adopté par init (PID 1)
#    - init nettoie automatiquement les zombies

# 5. Scripts shell mal écrits
#    - Pas de wait pour les processus en arrière-plan
#    - Pas de gestion de SIGCHLD
```

**Exemple de script problématique** :
```bash
#!/bin/bash
# Script qui crée des zombies

# ❌ MAUVAIS : Pas de wait()
for i in {1..10}; do
    (
        echo "Tâche $i"
        sleep 1
    ) &
done

# Les processus enfants terminent mais deviennent zombies
# car le parent ne les attend pas
```

## Diagnostic et identification

### Détection avec ps

**Commandes de base** :
```bash
#!/bin/bash
# Détecter les processus zombies

# Lister tous les zombies
ps aux | awk '$8 ~ /^Z/ {print}'

# Compter les zombies
zombie_count() {
    ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}'
}

# Détails sur les zombies
show_zombies() {
    echo "=== Processus zombies ==="
    ps aux | awk '$8 ~ /^Z/ {
        printf "PID: %s, PPID: %s, CMD: %s\n", $2, $3, $11
    }'
}

# Zombies avec leur parent
show_zombies_with_parent() {
    ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/ {
        printf "Zombie PID: %s, Parent PID: %s, CMD: %s\n", $1, $2, $4
    }'
}
```

### Détection avec /proc

**Inspection via /proc** :
```bash
#!/bin/bash
# Inspection des zombies via /proc

check_zombie_in_proc() {
    local pid="$1"
    
    if [ ! -d "/proc/$pid" ]; then
        echo "Processus $pid n'existe pas"
        return 1
    fi
    
    # Lire l'état
    local state=$(cat "/proc/$pid/stat" 2>/dev/null | awk '{print $3}')
    
    if [ "$state" = "Z" ]; then
        echo "Processus $pid est un zombie"
        
        # Informations supplémentaires
        local ppid=$(cat "/proc/$pid/stat" 2>/dev/null | awk '{print $4}')
        echo "Parent PID: $ppid"
        
        # Vérifier si le parent existe
        if [ -d "/proc/$ppid" ]; then
            local parent_cmd=$(cat "/proc/$ppid/comm" 2>/dev/null)
            echo "Parent: $parent_cmd"
        else
            echo "Parent n'existe plus (orphelin)"
        fi
        
        return 0
    else
        echo "Processus $pid n'est pas un zombie (état: $state)"
        return 1
    fi
}
```

## Détection automatique

### Script de monitoring

**Moniteur de zombies** :
```bash
#!/bin/bash
# Moniteur automatique de zombies

monitor_zombies() {
    local threshold="${1:-5}"  # Seuil d'alerte
    local interval="${2:-60}"  # Intervalle de vérification en secondes
    
    while true; do
        local zombie_count=$(ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}')
        
        if [ "$zombie_count" -gt "$threshold" ]; then
            echo "[$(date)] ALERTE: $zombie_count processus zombie(s) détecté(s)"
            show_zombies
            
            # Optionnel: Envoyer une alerte
            # send_alert "Zombie count: $zombie_count"
        else
            echo "[$(date)] OK: $zombie_count processus zombie(s)"
        fi
        
        sleep "$interval"
    done
}

# Exécuter en arrière-plan
# monitor_zombies 5 60 &
```

## Prévention des zombies

### Gestion correcte dans Bash

**Patterns corrects** :
```bash
#!/bin/bash
# Patterns corrects pour éviter les zombies

# Pattern 1: Attendre explicitement
correct_pattern_1() {
    (
        echo "Tâche en cours"
        sleep 1
    ) &
    
    local child_pid=$!
    wait "$child_pid"  # Attendre l'enfant
    echo "Tâche terminée"
}

# Pattern 2: Attendre tous les enfants
correct_pattern_2() {
    local -a pids=()
    
    for i in {1..10}; do
        (
            echo "Tâche $i"
            sleep 1
        ) &
        pids+=($!)
    done
    
    # Attendre tous les enfants
    for pid in "${pids[@]}"; do
        wait "$pid"
    done
    
    echo "Toutes les tâches terminées"
}

# Pattern 3: Gestionnaire SIGCHLD
correct_pattern_3() {
    # Installer un gestionnaire SIGCHLD
    trap 'wait -n' CHLD
    
    # Créer des enfants
    for i in {1..10}; do
        (
            echo "Tâche $i"
            sleep 1
        ) &
    done
    
    # Attendre que tous terminent
    wait
}
```

### Gestion avec wait -n

**wait -n (Bash 4.3+)** :
```bash
#!/bin/bash
# Utilisation de wait -n pour nettoyer les zombies

cleanup_children() {
    local -a pids=()
    
    # Créer plusieurs enfants
    for i in {1..5}; do
        (
            sleep $((RANDOM % 5))
            echo "Tâche $i terminée"
        ) &
        pids+=($!)
    done
    
    # Attendre chaque enfant au fur et à mesure
    while [ ${#pids[@]} -gt 0 ]; do
        wait -n  # Attend le prochain enfant qui termine
        # Retirer le PID terminé de la liste
        local new_pids=()
        for pid in "${pids[@]}"; do
            if kill -0 "$pid" 2>/dev/null; then
                new_pids+=("$pid")
            fi
        done
        pids=("${new_pids[@]}")
    done
    
    echo "Tous les enfants nettoyés"
}
```

## Gestionnaire de signaux SIGCHLD

### Implémentation complète

**Gestionnaire SIGCHLD robuste** :
```bash
#!/bin/bash
# Gestionnaire SIGCHLD complet

set -euo pipefail

# Compteur d'enfants
declare -i CHILD_COUNT=0

# Fonction de nettoyage
cleanup_children() {
    local pid
    local exit_code
    
    # Nettoyer tous les enfants terminés
    while true; do
        # wait -n retourne immédiatement si aucun enfant n'est terminé
        if pid=$(wait -n 2>/dev/null); then
            exit_code=$?
            CHILD_COUNT=$((CHILD_COUNT - 1))
            echo "Enfant $pid terminé avec le code $exit_code"
        else
            # Plus d'enfants à nettoyer
            break
        fi
    done
}

# Installer le gestionnaire SIGCHLD
trap cleanup_children CHLD

# Créer des enfants
spawn_children() {
    local count="${1:-5}"
    
    for i in $(seq 1 "$count"); do
        (
            echo "Enfant $i démarré"
            sleep $((RANDOM % 5 + 1))
            echo "Enfant $i terminé"
            exit $((i % 2))  # Code de sortie alterné
        ) &
        CHILD_COUNT=$((CHILD_COUNT + 1))
    done
    
    echo "$count enfants créés"
}

# Utilisation
spawn_children 10

# Attendre que tous les enfants terminent
while [ $CHILD_COUNT -gt 0 ]; do
    sleep 1
done

echo "Tous les enfants terminés et nettoyés"
```

## Nettoyage automatique

### Script de nettoyage

**Nettoyeur automatique** :
```bash
#!/bin/bash
# Nettoyeur automatique de zombies

cleanup_zombies() {
    local cleaned=0
    
    # Trouver tous les zombies
    local zombies=$(ps aux | awk '$8 ~ /^Z/ {print $2}')
    
    for zombie_pid in $zombies; do
        # Trouver le parent
        local ppid=$(ps -o ppid= -p "$zombie_pid" 2>/dev/null | tr -d ' ')
        
        if [ -n "$ppid" ]; then
            # Vérifier si le parent existe
            if [ -d "/proc/$ppid" ]; then
                local parent_cmd=$(ps -o cmd= -p "$ppid" 2>/dev/null)
                echo "Zombie $zombie_pid a un parent vivant: $ppid ($parent_cmd)"
                echo "  → Le parent doit appeler wait() pour nettoyer"
            else
                echo "Zombie $zombie_pid est orphelin (parent $ppid n'existe plus)"
                echo "  → Devrait être adopté par init et nettoyé automatiquement"
            fi
        fi
        
        cleaned=$((cleaned + 1))
    done
    
    if [ $cleaned -eq 0 ]; then
        echo "Aucun zombie trouvé"
    else
        echo "Total: $cleaned zombie(s) détecté(s)"
    fi
}

# Note: On ne peut pas tuer un zombie directement
# Il faut que le parent appelle wait() ou que le parent meure
```

### Nettoyage via init

**Adoption par init** :
```bash
#!/bin/bash
# Comprendre l'adoption par init

# Si un processus parent meurt avant d'appeler wait():
# 1. L'enfant devient orphelin
# 2. init (PID 1) devient son nouveau parent
# 3. init appelle automatiquement wait() pour tous ses enfants
# 4. Le zombie est nettoyé automatiquement

# Forcer l'adoption en tuant le parent
force_adoption() {
    local parent_pid="$1"
    
    echo "Parent PID: $parent_pid"
    
    # Lister les enfants
    local children=$(ps -o pid= --ppid "$parent_pid" 2>/dev/null)
    
    if [ -n "$children" ]; then
        echo "Enfants avant: $children"
        
        # Tuer le parent (les enfants deviendront orphelins)
        kill "$parent_pid" 2>/dev/null || true
        
        sleep 1
        
        # Vérifier que les enfants sont maintenant enfants de init
        for child in $children; do
            local new_ppid=$(ps -o ppid= -p "$child" 2>/dev/null | tr -d ' ')
            if [ "$new_ppid" = "1" ]; then
                echo "Enfant $child adopté par init"
            fi
        done
    fi
}
```

## Impact sur le système

### Ressources consommées

**Impact des zombies** :
```bash
#!/bin/bash
# Analyser l'impact des zombies

analyze_zombie_impact() {
    echo "=== Impact des processus zombies ==="
    echo ""
    
    # Compter les zombies
    local zombie_count=$(ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}')
    echo "Nombre de zombies: $zombie_count"
    
    # Limite de PIDs
    local pid_max=$(cat /proc/sys/kernel/pid_max 2>/dev/null || echo "32768")
    echo "Limite de PIDs: $pid_max"
    
    # PIDs utilisés
    local used_pids=$(ps -eo pid --no-headers | wc -l)
    echo "PIDs utilisés: $used_pids"
    
    # Impact
    if [ "$zombie_count" -gt 100 ]; then
        echo ""
        echo "⚠️  ALERTE: Nombre élevé de zombies"
        echo "   - Consommation d'entrées dans la table des processus"
        echo "   - Risque d'épuisement des PIDs disponibles"
    elif [ "$zombie_count" -gt 0 ]; then
        echo ""
        echo "ℹ️  Impact limité mais surveillance recommandée"
    else
        echo ""
        echo "✓ Aucun impact"
    fi
}
```

## Limites et ressources

### Limites système

**Vérification des limites** :
```bash
#!/bin/bash
# Vérifier les limites système

check_system_limits() {
    echo "=== Limites système ==="
    echo ""
    
    # Limite de PIDs
    echo "PID max: $(cat /proc/sys/kernel/pid_max)"
    
    # Limite de processus par utilisateur
    ulimit -u
    echo "Limite de processus (ulimit -u): $(ulimit -u)"
    
    # Processus actuels
    local process_count=$(ps -eo pid --no-headers | wc -l)
    echo "Processus actuels: $process_count"
    
    # Zombies
    local zombie_count=$(ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}')
    echo "Zombies: $zombie_count"
    
    # Calcul du pourcentage
    local pid_max=$(cat /proc/sys/kernel/pid_max)
    local usage_percent=$((process_count * 100 / pid_max))
    echo "Utilisation: $usage_percent%"
}
```

## Outils de monitoring

### Scripts de surveillance

**Surveillance complète** :
```bash
#!/bin/bash
# Système de surveillance des zombies

zombie_monitor() {
    local log_file="${1:-/var/log/zombie_monitor.log}"
    local alert_threshold="${2:-10}"
    
    while true; do
        local zombie_count=$(ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}')
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        
        {
            echo "[$timestamp] Zombies: $zombie_count"
            
            if [ "$zombie_count" -gt 0 ]; then
                echo "Détails:"
                ps aux | awk '$8 ~ /^Z/ {
                    printf "  PID: %s, PPID: %s, CMD: %s\n", $2, $3, $11
                }'
            fi
            
            if [ "$zombie_count" -gt "$alert_threshold" ]; then
                echo "ALERTE: Seuil dépassé!"
            fi
            
            echo ""
        } >> "$log_file"
        
        sleep 60
    done
}
```

## Scripts de diagnostic

### Diagnostic complet

**Outil de diagnostic** :
```bash
#!/bin/bash
# Outil de diagnostic des zombies

diagnose_zombies() {
    echo "=== Diagnostic des processus zombies ==="
    echo ""
    
    # Compter
    local zombie_count=$(ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}')
    echo "Nombre total: $zombie_count"
    echo ""
    
    if [ "$zombie_count" -eq 0 ]; then
        echo "✓ Aucun zombie détecté"
        return 0
    fi
    
    # Analyser par parent
    echo "=== Analyse par parent ==="
    ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/ {
        parents[$2]++
        zombie_info[$1] = $2 " " $4
    }
    END {
        for (ppid in parents) {
            print "Parent PID " ppid ": " parents[ppid] " zombie(s)"
            for (zpid in zombie_info) {
                split(zombie_info[zpid], info)
                if (info[1] == ppid) {
                    print "  - Zombie PID: " zpid ", CMD: " info[2]
                }
            }
        }
    }'
    
    echo ""
    echo "=== Recommandations ==="
    
    # Vérifier les parents
    ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/ {
        ppid = $2
        if (system("test -d /proc/" ppid) == 0) {
            print "Parent " ppid " existe - doit appeler wait()"
        } else {
            print "Parent " ppid " n'existe plus - devrait être adopté par init"
        }
    }'
}
```

## Résolution de problèmes

### Solutions pratiques

**Résoudre les zombies** :
```bash
#!/bin/bash
# Solutions pour résoudre les zombies

# Solution 1: Redémarrer le parent
restart_parent() {
    local zombie_pid="$1"
    local ppid=$(ps -o ppid= -p "$zombie_pid" 2>/dev/null | tr -d ' ')
    
    if [ -n "$ppid" ] && [ -d "/proc/$ppid" ]; then
        local parent_cmd=$(ps -o cmd= -p "$ppid" 2>/dev/null | awk '{print $1}')
        echo "Redémarrer le parent $ppid ($parent_cmd)"
        echo "  kill $ppid"
        echo "  # Puis redémarrer le service/processus"
    fi
}

# Solution 2: Forcer l'adoption par init
force_init_adoption() {
    local zombie_pid="$1"
    local ppid=$(ps -o ppid= -p "$zombie_pid" 2>/dev/null | tr -d ' ')
    
    if [ -n "$ppid" ] && [ "$ppid" != "1" ]; then
        echo "Tuer le parent $ppid pour forcer l'adoption par init"
        echo "  kill $ppid"
        echo "  # L'enfant sera adopté par init et nettoyé automatiquement"
    fi
}

# Solution 3: Corriger le code source
fix_source_code() {
    echo "Corriger le code source du parent:"
    echo "  1. Ajouter wait() après fork()"
    echo "  2. Installer un gestionnaire SIGCHLD"
    echo "  3. Utiliser wait -n pour nettoyer au fur et à mesure"
}
```

## Bonnes pratiques

### Recommandations

**Bonnes pratiques** :
```bash
#!/bin/bash
# Bonnes pratiques pour éviter les zombies

# 1. Toujours attendre les enfants
#    - Utiliser wait() ou wait -n
#    - Gérer tous les codes de sortie

# 2. Installer un gestionnaire SIGCHLD
#    - Nettoyer automatiquement les enfants terminés
#    - Utiliser wait -n dans le gestionnaire

# 3. Limiter le nombre d'enfants simultanés
#    - Utiliser un pool de processus
#    - Attendre avant de créer de nouveaux enfants

# 4. Gérer les erreurs
#    - Vérifier les codes de sortie
#    - Logger les erreurs

# 5. Tester avec des charges élevées
#    - Créer beaucoup d'enfants
#    - Vérifier qu'il n'y a pas de zombies

# 6. Monitoring
#    - Surveiller le nombre de zombies
#    - Alerter si le seuil est dépassé
```

## Scripts d'automatisation

### Gestionnaire complet

**Gestionnaire de zombies** :
```bash
#!/bin/bash
# Gestionnaire complet de zombies

set -euo pipefail

ZOMBIE_MANAGER_LOG="/var/log/zombie_manager.log"

# Fonction principale
main() {
    local action="${1:-status}"
    
    case "$action" in
        status)
            show_status
            ;;
        monitor)
            monitor_zombies
            ;;
        diagnose)
            diagnose_zombies
            ;;
        cleanup)
            attempt_cleanup
            ;;
        *)
            echo "Usage: $0 {status|monitor|diagnose|cleanup}"
            exit 1
            ;;
    esac
}

show_status() {
    local zombie_count=$(ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}')
    
    echo "=== État des zombies ==="
    echo "Nombre: $zombie_count"
    
    if [ "$zombie_count" -gt 0 ]; then
        echo ""
        echo "Détails:"
        ps aux | awk '$8 ~ /^Z/ {
            printf "  PID: %s, PPID: %s, CMD: %s\n", $2, $3, $11
        }'
    fi
}

monitor_zombies() {
    local threshold="${2:-10}"
    local interval="${3:-60}"
    
    echo "Surveillance démarrée (seuil: $threshold, intervalle: ${interval}s)"
    
    while true; do
        local zombie_count=$(ps aux | awk '$8 ~ /^Z/ {count++} END {print count+0}')
        local timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        
        echo "[$timestamp] Zombies: $zombie_count" | tee -a "$ZOMBIE_MANAGER_LOG"
        
        if [ "$zombie_count" -gt "$threshold" ]; then
            echo "ALERTE: Seuil dépassé!" | tee -a "$ZOMBIE_MANAGER_LOG"
            diagnose_zombies | tee -a "$ZOMBIE_MANAGER_LOG"
        fi
        
        sleep "$interval"
    done
}

attempt_cleanup() {
    echo "Tentative de nettoyage..."
    
    # Trouver les parents des zombies
    ps -eo pid,ppid,stat,cmd | awk '$3 ~ /^Z/ {
        ppid = $2
        if (system("test -d /proc/" ppid) == 0) {
            print "Parent " ppid " existe - action requise"
            # Note: On ne peut pas forcer le parent à appeler wait()
            # Il faut corriger le code source ou redémarrer le parent
        }
    }'
    
    echo "Note: Les zombies ne peuvent pas être tués directement"
    echo "      Le parent doit appeler wait() ou être redémarré"
}

# Utilisation
# zombie_manager.sh status
# zombie_manager.sh monitor 10 60
# zombie_manager.sh diagnose
# zombie_manager.sh cleanup
```

## Conclusion

La gestion des processus zombies est essentielle pour maintenir un système Linux sain et performant. En comprenant leur nature, en les détectant rapidement, et en implémentant des pratiques de prévention appropriées, vous pouvez éviter l'accumulation de zombies qui peuvent consommer des ressources système précieuses.

Un système bien conçu utilise toujours `wait()` ou `wait -n` pour nettoyer les processus enfants, installe des gestionnaires SIGCHLD appropriés, et surveille régulièrement le nombre de zombies. La compréhension approfondie de ces mécanismes permet de créer des applications robustes qui gèrent correctement leurs processus enfants.

Dans le chapitre suivant, nous explorerons l'utilisation avancée de awk et sed, découvrant comment ces outils puissants peuvent transformer et analyser des données textuelles de manière efficace.
