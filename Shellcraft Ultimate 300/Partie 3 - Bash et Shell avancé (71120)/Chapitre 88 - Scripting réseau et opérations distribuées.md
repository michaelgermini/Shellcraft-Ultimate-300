# Chapitre 88 - Scripting réseau et opérations distribuées

> "Dans un monde distribué, la coordination n'est pas un luxe : c'est une nécessité." - Proverbe réseau

## Introduction : Le réseau comme extension du shell

Imaginez-vous chef d'orchestre d'un immense réseau de musiciens, chacun sur sa propre scène, jouant sa partition tout en restant parfaitement synchronisé avec les autres. Le scripting réseau en Bash transforme vos scripts locaux en véritables systèmes distribués capables de coordonner des opérations à travers des réseaux complexes.

Dans ce chapitre, nous allons explorer les techniques avancées de scripting réseau : exécution distante, synchronisation distribuée, gestion de clusters, et automatisation réseau à grande échelle.

## Section 1 : Exécution distante et gestion de clusters

### 1.1 Framework d'exécution distante

Système complet pour l'exécution de commandes sur des hôtes distants :

```bash
#!/bin/bash

# Framework d'exécution distante
echo "=== Framework d'exécution distante ==="

# Gestionnaire d'exécution distante
RemoteExecutor() {
    local self="$1"
    
    declare -a $self._hosts
    declare -A $self._host_configs
    declare -A $self._results
    
    # Ajout d'un hôte
    $self.add_host() {
        local alias="$1"
        local connection_string="$2"
        local properties="${3:-}"
        
        $self._hosts+=("$alias")
        $self._host_configs["${alias}_connection"]="$connection_string"
        $self._host_configs["${alias}_properties"]="$properties"
        
        echo "✓ Hôte ajouté: $alias ($connection_string)"
    }
    
    # Exécution sur un hôte
    $self.execute_on_host() {
        local host_alias="$1"
        local command="$2"
        local timeout="${3:-30}"
        
        local connection="${$self._host_configs[${host_alias}_connection]}"
        
        if [[ -z "$connection" ]]; then
            echo "❌ Hôte inconnu: $host_alias" >&2
            return 1
        fi
        
        echo "--- Exécution sur $host_alias ---"
        echo "Commande: $command"
        
        local result_key="${host_alias}_$(date +%s)"
        
        # Exécution avec timeout et capture
        if [[ "$connection" == "local" ]]; then
            # Exécution locale pour test
            local output
            output=$(timeout "$timeout" bash -c "$command" 2>&1)
            local exit_code=$?
            
            $self._results["${result_key}_output"]="$output"
            $self._results["${result_key}_exit_code"]="$exit_code"
            $self._results["${result_key}_host"]="$host_alias"
            
            echo "Résultat: $( (( exit_code == 0 )) && echo "SUCCÈS" || echo "ÉCHEC ($exit_code)" )"
            return $exit_code
        else
            # Exécution SSH (simulation pour la démo)
            echo "SSH simulé vers: $connection"
            local simulated_output="Résultat simulé de $command sur $host_alias"
            local simulated_exit_code=$(( RANDOM % 2 ))  # 0 ou 1 aléatoire
            
            $self._results["${result_key}_output"]="$simulated_output"
            $self._results["${result_key}_exit_code"]="$simulated_exit_code"
            $self._results["${result_key}_host"]="$host_alias"
            
            echo "Résultat simulé: $( (( simulated_exit_code == 0 )) && echo "SUCCÈS" || echo "ÉCHEC ($simulated_exit_code)" )"
            return $simulated_exit_code
        fi
    }
    
    # Exécution parallèle sur tous les hôtes
    $self.execute_parallel() {
        local command="$1"
        local timeout="${2:-30}"
        
        echo "=== EXÉCUTION PARALLÈLE ==="
        echo "Commande: $command"
        echo "Hôtes: ${$self._hosts[*]}"
        
        local pids=()
        local host_pids=()
        
        # Lancement en parallèle
        for host in "${$self._hosts[@]}"; do
            (
                $self.execute_on_host "$host" "$command" "$timeout"
            ) &
            pids+=($!)
            host_pids["$!"]="$host"
        done
        
        # Collecte des résultats
        local success_count=0
        local failure_count=0
        
        for pid in "${pids[@]}"; do
            wait "$pid"
            local exit_code=$?
            local host="${host_pids[$pid]}"
            
            if (( exit_code == 0 )); then
                ((success_count++))
            else
                ((failure_count++))
                echo "❌ Échec sur $host (code: $exit_code)"
            fi
        done
        
        echo "Résumé: $success_count succès, $failure_count échecs"
        return $(( failure_count > 0 ))
    }
    
    # Exécution avec stratégie (premier succès, majorité, etc.)
    $self.execute_with_strategy() {
        local command="$1"
        local strategy="$2"
        local timeout="${3:-30}"
        
        echo "=== EXÉCUTION AVEC STRATÉGIE: $strategy ==="
        
        case "$strategy" in
            first_success)
                echo "Stratégie: Premier succès"
                for host in "${$self._hosts[@]}"; do
                    echo "Tentative sur: $host"
                    if $self.execute_on_host "$host" "$command" "$timeout"; then
                        echo "✓ Succès trouvé sur $host"
                        return 0
                    fi
                done
                echo "❌ Aucun hôte n'a réussi"
                return 1
                ;;
                
            majority)
                echo "Stratégie: Majorité"
                local total_hosts=${#$self._hosts[@]}
                local required_successes=$(( (total_hosts + 1) / 2 ))
                local successes=0
                
                for host in "${$self._hosts[@]}"; do
                    if $self.execute_on_host "$host" "$command" "$timeout"; then
                        ((successes++))
                        if (( successes >= required_successes )); then
                            echo "✓ Majorité atteinte ($successes/$total_hosts)"
                            return 0
                        fi
                    fi
                done
                
                echo "❌ Majorité non atteinte ($successes/$total_hosts)"
                return 1
                ;;
                
            all_or_nothing)
                echo "Stratégie: Tout ou rien"
                local failures=0
                
                for host in "${$self._hosts[@]}"; do
                    if ! $self.execute_on_host "$host" "$command" "$timeout"; then
                        ((failures++))
                    fi
                done
                
                if (( failures == 0 )); then
                    echo "✓ Tous les hôtes ont réussi"
                    return 0
                else
                    echo "❌ $failures échecs - rollback nécessaire"
                    return 1
                fi
                ;;
                
            *)
                echo "❌ Stratégie inconnue: $strategy"
                return 1
                ;;
        esac
    }
    
    # Collecte des résultats
    $self.get_results() {
        local host_filter="${1:-}"
        
        echo "=== RÉSULTATS D'EXÉCUTION ==="
        
        for result_key in "${!$self._results[@]}"; do
            if [[ "$result_key" =~ _output$ ]]; then
                local base_key="${result_key%_output}"
                local host="${$self._results[${base_key}_host]}"
                
                if [[ -z "$host_filter" || "$host" == "$host_filter" ]]; then
                    local output="${$self._results[${result_key}]}"
                    local exit_code="${$self._results[${base_key}_exit_code]}"
                    
                    echo "--- Résultat de $host ---"
                    echo "Code de sortie: $exit_code"
                    echo "Sortie:"
                    echo "$output"
                    echo
                fi
            fi
        done
    }
    
    # Tests de connectivité
    $self.test_connectivity() {
        echo "=== TESTS DE CONNECTIVITÉ ==="
        
        local reachable=0
        local unreachable=0
        
        for host in "${$self._hosts[@]}"; do
            local connection="${$self._host_configs[${host}_connection]}"
            
            if [[ "$connection" == "local" ]]; then
                echo "✓ $host: local (toujours accessible)"
                ((reachable++))
            else
                # Test ping simulé
                local random_result=$(( RANDOM % 4 ))
                if (( random_result < 3 )); then
                    echo "✓ $host: accessible"
                    ((reachable++))
                else
                    echo "❌ $host: inaccessible"
                    ((unreachable++))
                fi
            fi
        done
        
        echo "Résumé: $reachable accessibles, $unreachable inaccessibles"
        return $(( unreachable > 0 ))
    }
}

# Démonstration du framework d'exécution distante
echo "--- Framework d'exécution distante ---"

RemoteExecutor "remote_exec"

# Ajout d'hôtes
remote_exec.add_host "local" "local" "env=test,type=local"
remote_exec.add_host "web01" "user@web01.example.com" "env=prod,type=web"
remote_exec.add_host "web02" "user@web02.example.com" "env=prod,type=web"
remote_exec.add_host "db01" "user@db01.example.com" "env=prod,type=database"

# Tests de connectivité
remote_exec.test_connectivity

echo
echo "--- Exécution parallèle ---"
remote_exec.execute_parallel "echo 'Uptime: '; uptime"

echo
echo "--- Exécution avec stratégie ---"
remote_exec.execute_with_strategy "echo 'Test stratégique'" "first_success"

echo
echo "--- Résultats ---"
remote_exec.get_results
```

### 1.2 Gestion de clusters et load balancing

Distribution intelligente des tâches dans un cluster :

```bash
#!/bin/bash

# Gestion de clusters et load balancing
echo "=== Gestion de clusters et load balancing ==="

# Gestionnaire de cluster
ClusterManager() {
    local self="$1"
    
    declare -a $self._nodes
    declare -A $self._node_stats
    declare -A $self._tasks
    
    # Ajout d'un nœud au cluster
    $self.add_node() {
        local node_id="$1"
        local capacity="${2:-1}"
        local properties="${3:-}"
        
        $self._nodes+=("$node_id")
        $self._node_stats["${node_id}_capacity"]="$capacity"
        $self._node_stats["${node_id}_load"]="0"
        $self._node_stats["${node_id}_active_tasks"]="0"
        $self._node_stats["${node_id}_properties"]="$properties"
        
        echo "✓ Nœud ajouté: $node_id (capacité: $capacity)"
    }
    
    # Mise à jour des statistiques d'un nœud
    $self.update_node_stats() {
        local node_id="$1"
        
        # Simulation de récupération de métriques
        local cpu_load=$(( RANDOM % 100 ))
        local mem_load=$(( RANDOM % 100 ))
        local avg_load=$(( (cpu_load + mem_load) / 2 ))
        
        $self._node_stats["${node_id}_cpu_load"]="$cpu_load"
        $self._node_stats["${node_id}_mem_load"]="$mem_load"
        $self._node_stats["${node_id}_avg_load"]="$avg_load"
        
        echo "✓ Stats mises à jour pour $node_id: CPU=${cpu_load}%, MEM=${mem_load}%, AVG=${avg_load}%"
    }
    
    # Sélection d'un nœud selon la stratégie
    $self.select_node() {
        local strategy="${1:-round_robin}"
        
        case "$strategy" in
            round_robin)
                $self._select_round_robin
                ;;
            least_loaded)
                $self._select_least_loaded
                ;;
            weighted_random)
                $self._select_weighted_random
                ;;
            resource_based)
                $self._select_resource_based
                ;;
            *)
                echo "❌ Stratégie inconnue: $strategy" >&2
                return 1
                ;;
        esac
    }
    
    # Round-robin
    $self._select_round_robin() {
        local -i current_index="${$self._node_stats[round_robin_index]:-0}"
        
        if (( current_index >= ${#$self._nodes[@]} )); then
            current_index=0
        fi
        
        local selected_node="${$self._nodes[$current_index]}"
        $self._node_stats["round_robin_index"]=$(( current_index + 1 ))
        
        echo "$selected_node"
    }
    
    # Moins chargé
    $self._select_least_loaded() {
        local best_node=""
        local best_load=999
        
        for node in "${$self._nodes[@]}"; do
            $self.update_node_stats "$node"
            local load="${$self._node_stats[${node}_avg_load]}"
            
            if (( load < best_load )); then
                best_load="$load"
                best_node="$node"
            fi
        done
        
        echo "$best_node"
    }
    
    # Aléatoire pondéré
    $self._select_weighted_random() {
        local total_weight=0
        
        # Calcul du poids total (inversement proportionnel à la charge)
        for node in "${$self._nodes[@]}"; do
            $self.update_node_stats "$node"
            local load="${$self._node_stats[${node}_avg_load]}"
            local capacity="${$self._node_stats[${node}_capacity]}"
            
            # Poids = capacité / (1 + charge)
            local weight=$(( capacity * 100 / (1 + load) ))
            $self._node_stats["${node}_weight"]="$weight"
            total_weight=$(( total_weight + weight ))
        done
        
        # Sélection aléatoire pondérée
        local random=$(( RANDOM % total_weight ))
        local cumulative=0
        
        for node in "${$self._nodes[@]}"; do
            local weight="${$self._node_stats[${node}_weight]}"
            cumulative=$(( cumulative + weight ))
            
            if (( random < cumulative )); then
                echo "$node"
                return 0
            fi
        done
    }
    
    # Basé sur les ressources
    $self._select_resource_based() {
        local best_node=""
        local best_score=0
        
        for node in "${$self._nodes[@]}"; do
            $self.update_node_stats "$node"
            
            local cpu="${$self._node_stats[${node}_cpu_load]}"
            local mem="${$self._node_stats[${node}_mem_load]}"
            local capacity="${$self._node_stats[${node}_capacity]}"
            local active_tasks="${$self._node_stats[${node}_active_tasks]}"
            
            # Score = capacité * (100 - charge moyenne) / (1 + tâches actives)
            local avg_load=$(( (cpu + mem) / 2 ))
            local score=$(( capacity * (100 - avg_load) / (1 + active_tasks) ))
            
            if (( score > best_score )); then
                best_score="$score"
                best_node="$node"
            fi
        done
        
        echo "$best_node"
    }
    
    # Assignation d'une tâche
    $self.assign_task() {
        local task_id="$1"
        local task_command="$2"
        local strategy="${3:-round_robin}"
        
        local selected_node
        selected_node=$($self.select_node "$strategy")
        
        if [[ -z "$selected_node" ]]; then
            echo "❌ Aucun nœud disponible pour la tâche $task_id"
            return 1
        fi
        
        # Enregistrement de la tâche
        $self._tasks["${task_id}_node"]="$selected_node"
        $self._tasks["${task_id}_command"]="$task_command"
        $self._tasks["${task_id}_status"]="assigned"
        $self._tasks["${task_id}_start_time"]="$(date +%s)"
        
        # Mise à jour des stats du nœud
        local active_tasks="${$self._node_stats[${selected_node}_active_tasks]}"
        $self._node_stats["${selected_node}_active_tasks"]=$(( active_tasks + 1 ))
        
        echo "✓ Tâche $task_id assignée à $selected_node (stratégie: $strategy)"
        
        # Simulation d'exécution
        (
            sleep 2  # Simulation du temps de traitement
            $self._complete_task "$task_id"
        ) &
    }
    
    # Finalisation d'une tâche
    $self._complete_task() {
        local task_id="$1"
        
        local node="${$self._tasks[${task_id}_node]}"
        local active_tasks="${$self._node_stats[${node}_active_tasks]}"
        
        $self._node_stats["${node}_active_tasks"]=$(( active_tasks - 1 ))
        $self._tasks["${task_id}_status"]="completed"
        $self._tasks["${task_id}_end_time"]="$(date +%s)"
        
        echo "✓ Tâche $task_id terminée sur $node"
    }
    
    # État du cluster
    $self.cluster_status() {
        echo "=== ÉTAT DU CLUSTER ==="
        
        for node in "${$self._nodes[@]}"; do
            $self.update_node_stats "$node"
            
            local capacity="${$self._node_stats[${node}_capacity]}"
            local active_tasks="${$self._node_stats[${node}_active_tasks]}"
            local cpu="${$self._node_stats[${node}_cpu_load]}"
            local mem="${$self._node_stats[${node}_mem_load]}"
            
            echo "Nœud $node:"
            echo "  Capacité: $capacity"
            echo "  Tâches actives: $active_tasks"
            echo "  Charge CPU: ${cpu}%"
            echo "  Charge mémoire: ${mem}%"
            echo
        done
        
        # Statistiques des tâches
        local total_tasks=0
        local completed_tasks=0
        
        for task_key in "${!$self._tasks[@]}"; do
            if [[ "$task_key" =~ _status$ ]]; then
                ((total_tasks++))
                local status="${$self._tasks[$task_key]}"
                if [[ "$status" == "completed" ]]; then
                    ((completed_tasks++))
                fi
            fi
        done
        
        echo "Statistiques des tâches:"
        echo "  Total: $total_tasks"
        echo "  Terminées: $completed_tasks"
        echo "  En cours: $((total_tasks - completed_tasks))"
    }
    
    # Rééquilibrage du cluster
    $self.rebalance() {
        echo "=== RÉÉQUILIBRAGE DU CLUSTER ==="
        
        local total_capacity=0
        local total_load=0
        
        # Calcul des métriques globales
        for node in "${$self._nodes[@]}"; do
            $self.update_node_stats "$node"
            local capacity="${$self._node_stats[${node}_capacity]}"
            local load="${$self._node_stats[${node}_avg_load]}"
            
            total_capacity=$(( total_capacity + capacity ))
            total_load=$(( total_load + load ))
        done
        
        local avg_load=$(( total_load / ${#$self._nodes[@]} ))
        
        echo "Charge moyenne cible: ${avg_load}%"
        
        # Identification des nœuds surchargés et sous-chargés
        for node in "${$self._nodes[@]}"; do
            local load="${$self._node_stats[${node}_avg_load]}"
            local capacity="${$self._node_stats[${node}_capacity]}"
            
            if (( load > avg_load + 10 )); then
                echo "⚠️  $node surchargé (${load}%)"
            elif (( load < avg_load - 10 )); then
                echo "ℹ️  $node sous-chargé (${load}%)"
            else
                echo "✓ $node équilibré (${load}%)"
            fi
        done
        
        echo "Rééquilibrage simulé terminé"
    }
}

# Démonstration du gestionnaire de cluster
echo "--- Gestionnaire de cluster ---"

ClusterManager "cluster"

# Ajout de nœuds avec différentes capacités
cluster.add_node "node01" "2" "cpu=4,mem=8GB"
cluster.add_node "node02" "1" "cpu=2,mem=4GB"
cluster.add_node "node03" "3" "cpu=8,mem=16GB"
cluster.add_node "node04" "1" "cpu=2,mem=4GB"

# État initial
cluster.cluster_status

echo
echo "--- Assignation de tâches ---"

# Assignation avec différentes stratégies
cluster.assign_task "task001" "process_data.sh input1.dat" "round_robin"
cluster.assign_task "task002" "analyze_logs.sh /var/log/app.log" "least_loaded"
cluster.assign_task "task003" "backup_database.sh" "weighted_random"
cluster.assign_task "task004" "send_notifications.sh" "resource_based"

# Attente de la fin des tâches
echo "Attente de la fin des traitements..."
sleep 3

echo
echo "--- État final ---"
cluster.cluster_status

echo
echo "--- Rééquilibrage ---"
cluster.rebalance
```

## Section 2 : Synchronisation et communication distribuée

### 2.1 Systèmes de synchronisation de fichiers

Synchronisation avancée de fichiers dans des environnements distribués :

```bash
#!/bin/bash

# Systèmes de synchronisation de fichiers
echo "=== Systèmes de synchronisation de fichiers ==="

# Synchronisateur de fichiers distribué
FileSynchronizer() {
    local self="$1"
    
    declare -a $self._peers
    declare -A $self._file_hashes
    declare -A $self._sync_status
    
    # Ajout d'un pair
    $self.add_peer() {
        local peer_id="$1"
        local connection="$2"
        local sync_dir="$3"
        
        $self._peers+=("$peer_id")
        $self._sync_status["${peer_id}_connection"]="$connection"
        $self._sync_status["${peer_id}_sync_dir"]="$sync_dir"
        $self._sync_status["${peer_id}_last_sync"]="never"
        
        echo "✓ Pair ajouté: $peer_id ($connection:$sync_dir)"
    }
    
    # Calcul des empreintes des fichiers locaux
    $self._calculate_hashes() {
        local directory="$1"
        
        echo "Calcul des empreintes pour: $directory"
        
        local -A local_hashes
        
        while IFS= read -r -d '' file; do
            local relative_path="${file#$directory/}"
            
            if [[ -f "$file" ]]; then
                local hash
                hash=$(sha256sum "$file" 2>/dev/null | cut -d' ' -f1)
                local_hashes["$relative_path"]="$hash"
            fi
        done < <(find "$directory" -type f -print0 2>/dev/null)
        
        # Sauvegarde des hashes
        for file in "${!local_hashes[@]}"; do
            $self._file_hashes["local_$file"]="${local_hashes[$file]}"
        done
        
        echo "✓ ${#local_hashes[@]} fichiers analysés"
    }
    
    # Comparaison avec un pair
    $self._compare_with_peer() {
        local peer_id="$1"
        local local_dir="$2"
        
        local peer_connection="${$self._sync_status[${peer_id}_connection]}"
        local peer_dir="${$self._sync_status[${peer_id}_sync_dir]}"
        
        echo "Comparaison avec $peer_id..."
        
        # Simulation de récupération des hashes du pair
        declare -A peer_hashes
        local simulated_files=("config.txt" "data.dat" "logs/app.log")
        
        for file in "${simulated_files[@]}"; do
            # Simulation de hash différent pour certains fichiers
            if [[ "$file" == "config.txt" ]]; then
                peer_hashes["$file"]="different_hash_$(date +%s)"
            else
                peer_hashes["$file"]="same_hash_$file"
            fi
        done
        
        # Comparaison
        declare -a to_sync_from_peer
        declare -a to_sync_to_peer
        
        # Fichiers locaux à envoyer
        for file in "${!self._file_hashes[@]}"; do
            if [[ "$file" =~ ^local_ ]]; then
                local relative_path="${file#local_}"
                local local_hash="${$self._file_hashes[$file]}"
                local peer_hash="${peer_hashes[$relative_path]}"
                
                if [[ -n "$peer_hash" && "$local_hash" != "$peer_hash" ]]; then
                    to_sync_to_peer+=("$relative_path")
                fi
            fi
        done
        
        # Fichiers à récupérer du pair
        for file in "${!peer_hashes[@]}"; do
            local peer_hash="${peer_hashes[$file]}"
            local local_hash="${$self._file_hashes[local_$file]}"
            
            if [[ -z "$local_hash" || "$local_hash" != "$peer_hash" ]]; then
                to_sync_from_peer+=("$file")
            fi
        done
        
        echo "À envoyer vers $peer_id: ${to_sync_to_peer[*]}"
        echo "À récupérer de $peer_id: ${to_sync_from_peer[*]}"
        
        # Mise à jour du statut
        $self._sync_status["${peer_id}_to_send"]="${#to_sync_to_peer[@]}"
        $self._sync_status["${peer_id}_to_receive"]="${#to_sync_from_peer[@]}"
        $self._sync_status["${peer_id}_last_sync"]="$(date)"
        
        return 0
    }
    
    # Synchronisation bidirectionnelle
    $self.sync_with_peer() {
        local peer_id="$1"
        local local_dir="$2"
        
        echo "=== SYNCHRONISATION AVEC $peer_id ==="
        
        # Calcul des hashes locaux
        $self._calculate_hashes "$local_dir"
        
        # Comparaison
        $self._compare_with_peer "$peer_id" "$local_dir"
        
        # Synchronisation simulée
        local to_send="${$self._sync_status[${peer_id}_to_send]}"
        local to_receive="${$self._sync_status[${peer_id}_to_receive]}"
        
        if (( to_send > 0 || to_receive > 0 )); then
            echo "Synchronisation en cours..."
            
            # Simulation rsync
            echo "rsync -avz $local_dir/ ${$self._sync_status[${peer_id}_connection]}:${$self._sync_status[${peer_id}_sync_dir]}/"
            sleep 1
            
            echo "✓ Synchronisation terminée"
            echo "  Envoyé: $to_send fichiers"
            echo "  Reçu: $to_receive fichiers"
        else
            echo "✓ Rien à synchroniser - déjà à jour"
        fi
    }
    
    # Synchronisation complète du cluster
    $self.cluster_sync() {
        local local_dir="$1"
        
        echo "=== SYNCHRONISATION DU CLUSTER ==="
        echo "Répertoire local: $local_dir"
        
        for peer in "${$self._peers[@]}"; do
            $self.sync_with_peer "$peer" "$local_dir"
            echo
        done
        
        echo "Synchronisation du cluster terminée"
    }
    
    # État de synchronisation
    $self.sync_status() {
        echo "=== ÉTAT DE SYNCHRONISATION ==="
        
        for peer in "${$self._peers[@]}"; do
            echo "Pair $peer:"
            echo "  Dernière sync: ${$self._sync_status[${peer}_last_sync]}"
            echo "  À envoyer: ${$self._sync_status[${peer}_to_send]:-0} fichiers"
            echo "  À recevoir: ${$self._sync_status[${peer}_to_receive]:-0} fichiers"
            echo
        done
        
        # Statistiques globales
        local total_to_send=0
        local total_to_receive=0
        
        for peer in "${$self._peers[@]}"; do
            total_to_send=$(( total_to_send + ${$self._sync_status[${peer}_to_send]:-0} ))
            total_to_receive=$(( total_to_receive + ${$self._sync_status[${peer}_to_receive]:-0} ))
        done
        
        echo "Statistiques globales:"
        echo "  Total à envoyer: $total_to_send fichiers"
        echo "  Total à recevoir: $total_to_receive fichiers"
        echo "  Pairs synchronisés: ${#$self._peers[@]}"
    }
    
    # Nettoyage des anciens fichiers
    $self.cleanup_old_files() {
        local local_dir="$1"
        local max_age_days="${2:-30}"
        
        echo "=== NETTOYAGE DES ANCIENS FICHIERS ==="
        echo "Répertoire: $local_dir"
        echo "Âge maximum: $max_age_days jours"
        
        local deleted=0
        
        while IFS= read -r -d '' file; do
            local file_age_days=$(( ($(date +%s) - $(stat -c %Y "$file")) / 86400 ))
            
            if (( file_age_days > max_age_days )); then
                echo "Suppression: $file (${file_age_days} jours)"
                rm -f "$file"
                ((deleted++))
            fi
        done < <(find "$local_dir" -type f -print0 2>/dev/null)
        
        echo "✓ $deleted fichiers supprimés"
        
        # Synchronisation du nettoyage
        $self.cluster_sync "$local_dir"
    }
}

# Démonstration du synchronisateur
echo "--- Synchronisateur de fichiers ---"

FileSynchronizer "file_sync"

# Configuration des pairs
file_sync.add_peer "server1" "user@server1.example.com" "/shared/data"
file_sync.add_peer "server2" "user@server2.example.com" "/shared/data"
file_sync.add_peer "backup" "user@backup.example.com" "/backup/data"

# Création de données de test
mkdir -p /tmp/sync_test
echo "Configuration principale" > /tmp/sync_test/config.txt
echo "Données importantes" > /tmp/sync_test/data.dat
mkdir -p /tmp/sync_test/logs
echo "Log d'application" > /tmp/sync_test/logs/app.log

# Synchronisation
file_sync.cluster_sync "/tmp/sync_test"

echo
file_sync.sync_status

echo
echo "--- Nettoyage ---"
file_sync.cleanup_old_files "/tmp/sync_test" "0"  # Supprime tout pour la démo

# Nettoyage
rm -rf /tmp/sync_test
```

### 2.2 Communication inter-processus distribuée

Systèmes de messagerie et de coordination distribuée :

```bash
#!/bin/bash

# Communication inter-processus distribuée
echo "=== Communication inter-processus distribuée ==="

# Système de messagerie distribué
DistributedMessenger() {
    local self="$1"
    
    declare -a $self._nodes
    declare -A $self._message_queues
    declare -A $self._subscriptions
    
    # Ajout d'un nœud
    $self.add_node() {
        local node_id="$1"
        local endpoint="$2"
        
        $self._nodes+=("$node_id")
        $self._message_queues["${node_id}_queue"]=()
        $self._subscriptions["$node_id"]=()
        
        echo "✓ Nœud ajouté: $node_id ($endpoint)"
    }
    
    # Publication d'un message
    $self.publish() {
        local topic="$1"
        local message="$2"
        local publisher="${3:-$(hostname)}"
        
        local timestamp
        timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        local message_id="${publisher}_$(date +%s)_$RANDOM"
        
        local full_message="{
  \"id\": \"$message_id\",
  \"topic\": \"$topic\",
  \"content\": \"$message\",
  \"publisher\": \"$publisher\",
  \"timestamp\": \"$timestamp\"
}"
        
        echo "Publication sur topic '$topic': $message"
        
        # Distribution aux abonnés
        local delivered=0
        
        for node in "${$self._nodes[@]}"; do
            if $self._is_subscribed "$node" "$topic"; then
                # Ajout à la queue du nœud
                local -n node_queue=$self._message_queues["${node}_queue"]
                node_queue+=("$full_message")
                
                ((delivered++))
                echo "  ✓ Délivré à $node"
            fi
        done
        
        if (( delivered == 0 )); then
            echo "  ⚠️  Aucun abonné pour ce topic"
        else
            echo "  📨 Message délivré à $delivered nœud(s)"
        fi
        
        return 0
    }
    
    # Abonnement à un topic
    $self.subscribe() {
        local node_id="$1"
        local topic="$2"
        
        if ! $self._node_exists "$node_id"; then
            echo "❌ Nœud inconnu: $node_id"
            return 1
        fi
        
        local -n node_subs=$self._subscriptions["$node_id"]
        
        # Vérification si déjà abonné
        for existing_topic in "${node_subs[@]}"; do
            if [[ "$existing_topic" == "$topic" ]]; then
                echo "⚠️  $node_id déjà abonné à '$topic'"
                return 0
            fi
        done
        
        node_subs+=("$topic")
        echo "✓ $node_id abonné à '$topic'"
    }
    
    # Désabonnement
    $self.unsubscribe() {
        local node_id="$1"
        local topic="$2"
        
        local -n node_subs=$self._subscriptions["$node_id"]
        local -a new_subs
        
        for existing_topic in "${node_subs[@]}"; do
            if [[ "$existing_topic" != "$topic" ]]; then
                new_subs+=("$existing_topic")
            fi
        done
        
        $self._subscriptions["$node_id"]=("${new_subs[@]}")
        echo "✓ $node_id désabonné de '$topic'"
    }
    
    # Consommation de messages
    $self.consume_messages() {
        local node_id="$1"
        local max_messages="${2:-10}"
        
        if ! $self._node_exists "$node_id"; then
            echo "❌ Nœud inconnu: $node_id"
            return 1
        fi
        
        local -n node_queue=$self._message_queues["${node_id}_queue"]
        
        if (( ${#node_queue[@]} == 0 )); then
            echo "📭 Aucune message pour $node_id"
            return 0
        fi
        
        echo "=== MESSAGES POUR $node_id ==="
        
        local consumed=0
        while (( consumed < max_messages && ${#node_queue[@]} > 0 )); do
            local message="${node_queue[0]}"
            unset node_queue[0]
            node_queue=("${node_queue[@]}")  # Reindexer
            
            echo "--- Message $(printf "%02d" $((consumed + 1))) ---"
            
            # Parsing JSON simplifié
            local topic content timestamp
            topic=$(echo "$message" | grep '"topic"' | cut -d'"' -f4)
            content=$(echo "$message" | grep '"content"' | cut -d'"' -f4)
            timestamp=$(echo "$message" | grep '"timestamp"' | cut -d'"' -f4)
            
            echo "Topic: $topic"
            echo "Contenu: $content"
            echo "Timestamp: $timestamp"
            echo
            
            ((consumed++))
        done
        
        echo "✓ $consumed message(s) consommé(s)"
        echo "Messages restants en queue: ${#node_queue[@]}"
    }
    
    # Fonctions utilitaires
    $self._node_exists() {
        local node_id="$1"
        
        for node in "${$self._nodes[@]}"; do
            if [[ "$node" == "$node_id" ]]; then
                return 0
            fi
        done
        return 1
    }
    
    $self._is_subscribed() {
        local node_id="$1"
        local topic="$2"
        
        local -n node_subs=$self._subscriptions["$node_id"]
        
        for existing_topic in "${node_subs[@]}"; do
            if [[ "$existing_topic" == "$topic" ]]; then
                return 0
            fi
        done
        return 1
    }
    
    # État du système
    $self.status() {
        echo "=== ÉTAT DU SYSTÈME DE MESSAGERIE ==="
        
        for node in "${$self._nodes[@]}"; do
            local -n node_queue=$self._message_queues["${node}_queue"]
            local -n node_subs=$self._subscriptions["$node"]
            
            echo "Nœud $node:"
            echo "  Abonnements: ${node_subs[*]}"
            echo "  Messages en queue: ${#node_queue[@]}"
            echo
        done
        
        # Statistiques globales
        local total_messages=0
        local total_subscriptions=0
        
        for node in "${$self._nodes[@]}"; do
            local -n node_queue=$self._message_queues["${node}_queue"]
            local -n node_subs=$self._subscriptions["$node"]
            
            total_messages=$(( total_messages + ${#node_queue[@]} ))
            total_subscriptions=$(( total_subscriptions + ${#node_subs[@]} ))
        done
        
        echo "Statistiques globales:"
        echo "  Nœuds: ${#$self._nodes[@]}"
        echo "  Messages en attente: $total_messages"
        echo "  Abonnements actifs: $total_subscriptions"
    }
    
    # Nettoyage des anciens messages
    $self.cleanup_old_messages() {
        local max_age_seconds="${1:-3600}"  # 1 heure par défaut
        
        echo "=== NETTOYAGE DES ANCIENS MESSAGES ==="
        echo "Âge maximum: ${max_age_seconds} secondes"
        
        local cleaned=0
        local current_time=$(date +%s)
        
        for node in "${$self._nodes[@]}"; do
            local -n node_queue=$self._message_queues["${node}_queue"]
            local -a kept_messages
            
            for message in "${node_queue[@]}"; do
                local timestamp
                timestamp=$(echo "$message" | grep '"timestamp"' | cut -d'"' -f4)
                
                if [[ -n "$timestamp" ]]; then
                    local message_time
                    message_time=$(date -d "$timestamp" +%s 2>/dev/null)
                    
                    if (( current_time - message_time < max_age_seconds )); then
                        kept_messages+=("$message")
                    else
                        ((cleaned++))
                    fi
                else
                    kept_messages+=("$message")
                fi
            done
            
            $self._message_queues["${node}_queue"]=("${kept_messages[@]}")
        done
        
        echo "✓ $cleaned message(s) nettoyé(s)"
    }
}

# Démonstration du système de messagerie
echo "--- Système de messagerie distribué ---"

DistributedMessenger "messenger"

# Ajout de nœuds
messenger.add_node "web_server" "192.168.1.10:8080"
messenger.add_node "worker_01" "192.168.1.11:8081"
messenger.add_node "worker_02" "192.168.1.12:8082"
messenger.add_node "database" "192.168.1.13:5432"

# Abonnements
messenger.subscribe "web_server" "user_actions"
messenger.subscribe "web_server" "system_alerts"
messenger.subscribe "worker_01" "job_queue"
messenger.subscribe "worker_02" "job_queue"
messenger.subscribe "database" "data_updates"
messenger.subscribe "worker_01" "system_alerts"
messenger.subscribe "worker_02" "system_alerts"

# Publication de messages
echo
echo "--- Publication de messages ---"

messenger.publish "user_actions" "User login: alice@example.com"
messenger.publish "job_queue" "Process data batch #12345"
messenger.publish "system_alerts" "High CPU usage detected: 85%"
messenger.publish "data_updates" "New user registered: bob@example.com"
messenger.publish "maintenance" "System maintenance scheduled"  # Aucun abonné

# Consommation des messages
echo
echo "--- Consommation des messages ---"

messenger.consume_messages "web_server" 3
echo
messenger.consume_messages "worker_01" 2
echo
messenger.consume_messages "database" 5

# État du système
echo
messenger.status

# Nettoyage
echo
echo "--- Nettoyage ---"
messenger.cleanup_old_messages 1  # Supprime tout pour la démo
messenger.status
```

## Section 3 : Automatisation réseau et orchestration

### 3.1 Gestion d'infrastructure réseau

Configuration et monitoring d'infrastructure réseau :

```bash
#!/bin/bash

# Gestion d'infrastructure réseau
echo "=== Gestion d'infrastructure réseau ==="

# Gestionnaire d'infrastructure réseau
NetworkManager() {
    local self="$1"
    
    declare -a $self._devices
    declare -A $self._device_configs
    declare -A $self._network_status
    
    # Ajout d'un périphérique réseau
    $self.add_device() {
        local device_id="$1"
        local device_type="$2"  # switch, router, firewall, server
        local ip_address="$3"
        local properties="${4:-}"
        
        $self._devices+=("$device_id")
        $self._device_configs["${device_id}_type"]="$device_type"
        $self._device_configs["${device_id}_ip"]="$ip_address"
        $self._device_configs["${device_id}_properties"]="$properties"
        $self._network_status["${device_id}_reachable"]="unknown"
        $self._network_status["${device_id}_last_check"]="never"
        
        echo "✓ Périphérique ajouté: $device_id ($device_type - $ip_address)"
    }
    
    # Test de connectivité
    $self.test_connectivity() {
        echo "=== TESTS DE CONNECTIVITÉ RÉSEAU ==="
        
        local reachable=0
        local unreachable=0
        
        for device in "${$self._devices[@]}"; do
            local ip="${$self._device_configs[${device}_ip]}"
            
            echo -n "Test $device ($ip): "
            
            # Test ping
            if ping -c 1 -W 2 "$ip" >/dev/null 2>&1; then
                echo "✓ accessible"
                $self._network_status["${device}_reachable"]="yes"
                ((reachable++))
            else
                echo "❌ inaccessible"
                $self._network_status["${device}_reachable"]="no"
                ((unreachable++))
            fi
            
            $self._network_status["${device}_last_check"]="$(date)"
        done
        
        echo
        echo "Résumé: $reachable accessibles, $unreachable inaccessibles"
        
        return $(( unreachable > 0 ))
    }
    
    # Configuration de VLANs
    $self.configure_vlans() {
        echo "=== CONFIGURATION DES VLANS ==="
        
        # Définition des VLANs
        declare -A vlans
        vlans["vlan10"]="management"
        vlans["vlan20"]="web_servers"
        vlans["vlan30"]="database"
        vlans["vlan40"]="backup"
        
        for vlan in "${!vlans[@]}"; do
            local name="${vlans[$vlan]}"
            echo "Configuration VLAN $vlan ($name)"
            
            # Simulation de configuration sur les switches
            for device in "${$self._devices[@]}"; do
                local type="${$self._device_configs[${device}_type]}"
                
                if [[ "$type" == "switch" ]]; then
                    echo "  Application sur $device..."
                    # Simulation: ssh $device "vlan $vlan; name $name; exit"
                    sleep 0.1
                fi
            done
        done
        
        echo "✓ Configuration VLAN terminée"
    }
    
    # Configuration de règles firewall
    $self.configure_firewall() {
        echo "=== CONFIGURATION FIREWALL ==="
        
        # Règles de sécurité
        declare -a firewall_rules
        firewall_rules+=("allow tcp 22 from 192.168.1.0/24")     # SSH depuis le réseau admin
        firewall_rules+=("allow tcp 80,443 from 0.0.0.0/0")      # HTTP/HTTPS depuis partout
        firewall_rules+=("allow tcp 3306 from 192.168.2.0/24")   # MySQL depuis les web servers
        firewall_rules+=("deny all")                              # Tout le reste refusé
        
        for device in "${$self._devices[@]}"; do
            local type="${$self._device_configs[${device}_type]}"
            
            if [[ "$type" == "firewall" ]]; then
                echo "Configuration du firewall $device:"
                
                for rule in "${firewall_rules[@]}"; do
                    echo "  Application règle: $rule"
                    # Simulation: ssh $device "firewall add $rule"
                    sleep 0.1
                done
                
                echo "  ✓ Firewall $device configuré"
            fi
        done
        
        echo "✓ Configuration firewall terminée"
    }
    
    # Configuration de routage
    $self.configure_routing() {
        echo "=== CONFIGURATION DU ROUTAGE ==="
        
        # Table de routage simulée
        declare -A routes
        routes["192.168.1.0/24"]="eth0"    # Réseau admin
        routes["192.168.2.0/24"]="eth1"    # Réseau web
        routes["192.168.3.0/24"]="eth2"    # Réseau data
        routes["0.0.0.0/0"]="gateway"      # Route par défaut
        
        for device in "${$self._devices[@]}"; do
            local type="${$self._device_configs[${device}_type]}"
            
            if [[ "$type" == "router" ]]; then
                echo "Configuration du routage sur $device:"
                
                for network in "${!routes[@]}"; do
                    local interface="${routes[$network]}"
                    echo "  Ajout route: $network via $interface"
                    # Simulation: ssh $device "ip route add $network dev $interface"
                    sleep 0.1
                done
                
                echo "  ✓ Routage configuré sur $device"
            fi
        done
        
        echo "✓ Configuration routage terminée"
    }
    
    # Monitoring réseau
    $self.network_monitor() {
        local duration="${1:-30}"
        
        echo "=== MONITORING RÉSEAU ($duration secondes) ==="
        
        local start_time=$(date +%s)
        local checks=0
        
        while (( $(date +%s) - start_time < duration )); do
            ((checks++))
            
            echo "[$(date +%H:%M:%S)] === Contrôle réseau #$checks ==="
            
            # Test de connectivité
            $self.test_connectivity
            
            # Métriques réseau (simulation)
            echo "Métriques réseau:"
            echo "  Bande passante utilisée: $((RANDOM % 100))%"
            echo "  Paquets perdus: $((RANDOM % 10))"
            echo "  Latence moyenne: $((RANDOM % 50 + 10))ms"
            
            # Alertes
            local bandwidth=$((RANDOM % 100))
            if (( bandwidth > 80 )); then
                echo "⚠️  ALERTE: Bande passante élevée ($bandwidth%)"
            fi
            
            echo
            sleep 5
        done
        
        echo "Monitoring terminé - $checks contrôles effectués"
    }
    
    # Sauvegarde de configuration
    $self.backup_configs() {
        local backup_dir="${1:-/tmp/network_backup_$(date +%Y%m%d_%H%M%S)}"
        
        echo "=== SAUVEGARDE DES CONFIGURATIONS ==="
        echo "Destination: $backup_dir"
        
        mkdir -p "$backup_dir"
        
        for device in "${$self._devices[@]}"; do
            local type="${$self._device_configs[${device}_type]}"
            local ip="${$self._device_configs[${device}_ip]}"
            
            echo "Sauvegarde de $device ($type - $ip)"
            
            # Simulation de sauvegarde
            cat > "$backup_dir/${device}_config.txt" << EOF
# Configuration de $device
# Type: $type
# IP: $ip
# Générée le: $(date)

# Configuration réseau
interface eth0
 ip address $ip 255.255.255.0
 no shutdown

# Configuration spécifique au type
EOF
            
            case "$type" in
                switch)
                    cat >> "$backup_dir/${device}_config.txt" << EOF
# Configuration switch
vlan 10
 name management
vlan 20
 name servers
EOF
                    ;;
                router)
                    cat >> "$backup_dir/${device}_config.txt" << EOF
# Configuration router
ip route 0.0.0.0 0.0.0.0 192.168.1.1
EOF
                    ;;
                firewall)
                    cat >> "$backup_dir/${device}_config.txt" << EOF
# Configuration firewall
allow tcp 22 from 192.168.1.0/24
allow tcp 80,443 from any
deny all
EOF
                    ;;
            esac
            
            echo "  ✓ Configuration sauvegardée: ${device}_config.txt"
        done
        
        # Archive
        local archive_name="${backup_dir}.tar.gz"
        tar -czf "$archive_name" -C "$(dirname "$backup_dir")" "$(basename "$backup_dir")"
        rm -rf "$backup_dir"
        
        echo "✓ Archive créée: $archive_name"
    }
    
    # État de l'infrastructure
    $self.infrastructure_status() {
        echo "=== ÉTAT DE L'INFRASTRUCTURE RÉSEAU ==="
        
        for device in "${$self._devices[@]}"; do
            local type="${$self._device_configs[${device}_type]}"
            local ip="${$self._device_configs[${device}_ip]}"
            local reachable="${$self._network_status[${device}_reachable]:-unknown}"
            local last_check="${$self._network_status[${device}_last_check]:-never}"
            
            echo "Périphérique $device:"
            echo "  Type: $type"
            echo "  IP: $ip"
            echo "  Accessible: $reachable"
            echo "  Dernière vérification: $last_check"
            echo
        done
        
        # Statistiques
        local total_devices=${#$self._devices[@]}
        local reachable_devices=0
        
        for device in "${$self._devices[@]}"; do
            if [[ "${$self._network_status[${device}_reachable]}" == "yes" ]]; then
                ((reachable_devices++))
            fi
        done
        
        echo "Statistiques:"
        echo "  Total périphériques: $total_devices"
        echo "  Périphériques accessibles: $reachable_devices"
        echo "  Taux de disponibilité: $(( reachable_devices * 100 / total_devices ))%"
    }
}

# Démonstration du gestionnaire réseau
echo "--- Gestionnaire d'infrastructure réseau ---"

NetworkManager "net_mgr"

# Ajout de périphériques
net_mgr.add_device "core_switch" "switch" "192.168.1.1" "ports=48,vendor=cisco"
net_mgr.add_device "router_gw" "router" "192.168.1.254" "interfaces=4,vendor=juniper"
net_mgr.add_device "firewall" "firewall" "192.168.1.253" "vendor=paloalto"
net_mgr.add_device "web_server" "server" "192.168.2.10" "os=ubuntu,services=nginx"
net_mgr.add_device "db_server" "server" "192.168.3.10" "os=ubuntu,services=mysql"

# Tests de connectivité
net_mgr.test_connectivity

echo
echo "--- Configuration réseau ---"
net_mgr.configure_vlans
echo
net_mgr.configure_firewall
echo
net_mgr.configure_routing

echo
echo "--- État de l'infrastructure ---"
net_mgr.infrastructure_status

echo
echo "--- Sauvegarde des configurations ---"
net_mgr.backup_configs

# Monitoring court
echo
echo "--- Monitoring réseau (court) ---"
net_mgr.network_monitor 10

# Nettoyage
rm -f /tmp/network_backup_*.tar.gz
```

## Conclusion : Le réseau comme extension de l'automatisation

Le scripting réseau en Bash transforme vos scripts locaux en véritables orchestrateurs de systèmes distribués. Comme un chef d'orchestre coordonnant plusieurs formations musicales à distance, vous apprenez à gérer la complexité des communications, synchronisations, et automatisations à travers les réseaux.

**Points clés à retenir :**

1. **Exécution distante** : Frameworks complets pour exécuter des commandes sur des hôtes multiples avec stratégies de gestion d'erreurs
2. **Gestion de clusters** : Load balancing intelligent et distribution de tâches dans des environnements distribués
3. **Synchronisation de fichiers** : Systèmes avancés de synchronisation bidirectionnelle et gestion de conflits
4. **Communication distribuée** : Messagerie publish/subscribe pour la coordination inter-processus
5. **Infrastructure réseau** : Automatisation complète de la configuration VLAN, firewall, et routage

Dans le chapitre suivant, nous explorerons les techniques avancées de monitoring et observabilité, pour que vos systèmes distribués soient non seulement automatisés, mais aussi complètement observables et maintenables.

---

**Exercice pratique :** Créez un système d'orchestration réseau complet incluant :
- Découverte automatique des hôtes sur le réseau
- Déploiement automatique d'agents de monitoring
- Synchronisation de configurations à travers le cluster
- Système de messagerie pour les alertes et commandes distribuées
- Interface de management centralisée pour l'administration

**Réflexion :** Comment adapteriez-vous ces techniques réseau pour créer un système de déploiement continu (CD) distribué capable de gérer des déploiements zero-downtime sur des clusters géographiquement distribués ?
