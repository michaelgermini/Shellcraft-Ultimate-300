# Chapitre 95 - Scripting réseau et opérations distribuées

> "Le réseau n'est pas un ensemble de connexions : c'est un écosystème vivant où chaque script devient un nœud intelligent, capable de communiquer, coordonner, et évoluer collectivement." - Network Scripting Maestro

## Introduction : Le réseau comme organisme vivant

Imaginez-vous en tant que chef d'orchestre d'un immense réseau neuronal : chaque machine un neurone, chaque connexion une synapse, chaque script un influx nerveux transmettant l'information à la vitesse de la lumière. Le scripting réseau en Bash transforme les opérations distribuées d'un ensemble de commandes isolées en un système nerveux distribué où l'intelligence émergente naît de la coordination parfaite entre machines.

Dans ce chapitre, nous construirons le système nerveux de nos architectures distribuées : outils réseau avancés, opérations coordonnées, communication inter-processus, et protocoles de synchronisation qui transforment l'ensemble de machines en un super-organisme opérationnel.

## Section 1 : Outils réseau avancés et automatisation

### 1.1 Framework de diagnostic réseau automatisé

Système intelligent de surveillance et diagnostic réseau :

```bash
#!/bin/bash

# Framework de diagnostic réseau automatisé
echo "=== Framework de diagnostic réseau automatisé ==="

# Network Diagnostics Framework
NetworkDiagnostics() {
    local self="$1"
    
    declare -A $self._network_targets
    declare -A $self._diagnostic_rules
    declare -A $self._diagnostic_history
    declare -A $self._network_topology
    
    # Enregistrement d'une cible réseau
    $self.register_network_target() {
        local target_name="$1"
        local target_type="$2"  # host, network, service
        local address="$3"
        local additional_info="$4"
        
        $self._network_targets["${target_name}_type"]="$target_type"
        $self._network_targets["${target_name}_address"]="$address"
        $self._network_targets["${target_name}_info"]="$additional_info"
        
        echo "✓ Cible réseau enregistrée: $target_name ($target_type)"
    }
    
    # Définition d'une règle de diagnostic
    $self.define_diagnostic_rule() {
        local rule_name="$1"
        local rule_type="$2"
        local command="$3"
        local success_pattern="$4"
        local error_pattern="$5"
        local remediation="$6"
        
        $self._diagnostic_rules["${rule_name}_type"]="$rule_type"
        $self._diagnostic_rules["${rule_name}_command"]="$command"
        $self._diagnostic_rules["${rule_name}_success"]="$success_pattern"
        $self._diagnostic_rules["${rule_name}_error"]="$error_pattern"
        $self._diagnostic_rules["${rule_name}_remediation"]="$remediation"
        
        echo "✓ Règle de diagnostic définie: $rule_name"
    }
    
    # Exécution d'un diagnostic complet
    $self.run_comprehensive_diagnostic() {
        local target_name="$1"
        
        echo "=== DIAGNOSTIC RÉSEAU COMPLET: $target_name ==="
        echo "Début: $(date)"
        echo
        
        local target_type="${$self._network_targets[${target_name}_type]}"
        local target_address="${$self._network_targets[${target_name}_address]}"
        
        if [[ -z "$target_type" ]]; then
            echo "❌ Cible inconnue: $target_name"
            return 1
        fi
        
        local diagnostic_start
        diagnostic_start="$(date +%s)"
        
        echo "Type de cible: $target_type"
        echo "Adresse: $target_address"
        echo
        
        # Exécution de tous les diagnostics appropriés
        local -A results
        local total_tests=0
        local passed_tests=0
        local failed_tests=0
        
        for rule_key in "${!$self._diagnostic_rules[@]}"; do
            if [[ "$rule_key" =~ _type$ ]]; then
                local rule_name="${rule_key%_type}"
                local rule_type="${$self._diagnostic_rules[$rule_key]}"
                
                # Vérifier si la règle s'applique à ce type de cible
                if $self._rule_applies_to_target "$rule_type" "$target_type"; then
                    ((total_tests++))
                    
                    echo "--- Test: $rule_name ---"
                    
                    if $self._execute_diagnostic_rule "$rule_name" "$target_name"; then
                        results["$rule_name"]="PASSED"
                        ((passed_tests++))
                        echo "✅ RÉUSSI"
                    else
                        results["$rule_name"]="FAILED"
                        ((failed_tests++))
                        echo "❌ ÉCHEC"
                    fi
                    
                    echo
                fi
            fi
        done
        
        # Analyse de topologie si applicable
        if [[ "$target_type" == "network" ]]; then
            $self._analyze_network_topology "$target_address"
        fi
        
        local diagnostic_end
        diagnostic_end="$(date +%s)"
        local duration=$(( diagnostic_end - diagnostic_start ))
        
        # Résumé
        echo "=== RÉSULTATS DU DIAGNOSTIC ==="
        echo "Durée: ${duration}s"
        echo "Tests exécutés: $total_tests"
        echo "Réussis: $passed_tests"
        echo "Échoués: $failed_tests"
        echo
        
        # Détails des échecs
        if (( failed_tests > 0 )); then
            echo "DÉTAILS DES ÉCHECS:"
            for rule in "${!results[@]}"; do
                if [[ "${results[$rule]}" == "FAILED" ]]; then
                    local remediation="${$self._diagnostic_rules[${rule}_remediation]}"
                    echo "• $rule: $remediation"
                fi
            done
            echo
        fi
        
        # Score de santé réseau
        local health_score=$(( passed_tests * 100 / total_tests ))
        echo "Score de santé réseau: ${health_score}/100"
        
        if (( health_score >= 90 )); then
            echo "État réseau: ✅ EXCELLENT"
        elif (( health_score >= 75 )); then
            echo "État réseau: ⚠️ BON"
        elif (( health_score >= 60 )); then
            echo "État réseau: ⚠️ MOYEN"
        else
            echo "État réseau: ❌ CRITIQUE - Action requise"
        fi
        
        # Historique
        $self._diagnostic_history["${target_name}_$(date +%s)"]="$health_score:$passed_tests:$failed_tests"
        
        return $(( failed_tests > 0 ))
    }
    
    # Vérification si une règle s'applique à une cible
    $self._rule_applies_to_target() {
        local rule_type="$1"
        local target_type="$2"
        
        case "$rule_type" in
            connectivity)
                [[ "$target_type" == "host" || "$target_type" == "service" ]]
                ;;
                
            dns)
                [[ "$target_type" == "host" || "$target_type" == "service" ]]
                ;;
                
            port_scan|service_scan)
                [[ "$target_type" == "host" || "$target_type" == "network" ]]
                ;;
                
            network_scan|topology)
                [[ "$target_type" == "network" ]]
                ;;
                
            latency|bandwidth)
                [[ "$target_type" == "host" ]]
                ;;
                
            *)
                true  # Règle générale
                ;;
        esac
    }
    
    # Exécution d'une règle de diagnostic
    $self._execute_diagnostic_rule() {
        local rule_name="$1"
        local target_name="$2"
        
        local command_template="${$self._diagnostic_rules[${rule_name}_command]}"
        local success_pattern="${$self._diagnostic_rules[${rule_name}_success]}"
        local error_pattern="${$self._diagnostic_rules[${rule_name}_error]}"
        
        local target_address="${$self._network_targets[${target_name}_address]}"
        
        # Substitution de l'adresse dans la commande
        local command="${command_template//\{TARGET\}/$target_address}"
        
        echo "Commande: $command"
        
        # Exécution avec timeout
        local output
        output="$(timeout 30 bash -c "$command" 2>&1)"
        local exit_code=$?
        
        if [[ $exit_code -eq 124 ]]; then
            echo "Timeout dépassé"
            return 1
        fi
        
        # Analyse du résultat
        if [[ -n "$success_pattern" ]] && echo "$output" | grep -q "$success_pattern"; then
            return 0
        elif [[ -n "$error_pattern" ]] && echo "$output" | grep -q "$error_pattern"; then
            return 1
        elif [[ $exit_code -eq 0 ]]; then
            return 0
        else
            return 1
        fi
    }
    
    # Analyse de topologie réseau
    $self._analyze_network_topology() {
        local network_address="$1"
        
        echo "ANALYSE DE TOPOLOGIE RÉSEAU: $network_address"
        
        # Simulation d'analyse de topologie (nmap, traceroute, etc.)
        echo "Découverte d'hôtes actifs..."
        
        # Simulation de résultats
        local discovered_hosts=("192.168.1.1" "192.168.1.10" "192.168.1.20" "192.168.1.100")
        
        for host in "${discovered_hosts[@]}"; do
            echo "  Hôte découvert: $host"
            
            # Enregistrement dans la topologie
            $self._network_topology["${network_address}_${host}"]="$(date +%s)"
        done
        
        echo "Topologie analysée: ${#discovered_hosts[@]} hôtes découverts"
    }
    
    # Surveillance réseau continue
    $self.start_network_monitoring() {
        local target_name="$1"
        local interval="${2:-60}"  # secondes
        
        echo "=== SURVEILLANCE RÉSEAU CONTINUE ==="
        echo "Cible: $target_name"
        echo "Intervalle: ${interval}s"
        echo "Arrêt: Ctrl+C"
        echo
        
        local baseline_score=""
        local consecutive_failures=0
        
        while true; do
            local timestamp
            timestamp="$(date '+%H:%M:%S')"
            
            echo -n "[$timestamp] Diagnostic... "
            
            if $self.run_comprehensive_diagnostic "$target_name" >/dev/null 2>&1; then
                echo "✅ OK"
                consecutive_failures=0
            else
                ((consecutive_failures++))
                echo "❌ ÉCHEC ($consecutive_failures consécutifs)"
                
                if (( consecutive_failures >= 3 )); then
                    echo "🚨 ALERTE: $consecutive_failures échecs consécutifs!"
                    
                    # Ici on pourrait envoyer une notification
                    # $self._send_alert_notification "$target_name" "$consecutive_failures"
                fi
            fi
            
            sleep "$interval"
        done
    }
    
    # Test de performance réseau
    $self.run_performance_test() {
        local target_name="$1"
        local test_type="${2:-comprehensive}"  # latency, bandwidth, comprehensive
        
        echo "=== TEST PERFORMANCE RÉSEAU: $target_name ==="
        echo "Type: $test_type"
        
        local target_address="${$self._network_targets[${target_name}_address]}"
        
        case "$test_type" in
            latency)
                $self._test_latency "$target_address"
                ;;
                
            bandwidth)
                $self._test_bandwidth "$target_address"
                ;;
                
            comprehensive)
                $self._test_latency "$target_address"
                $self._test_bandwidth "$target_address"
                ;;
        esac
    }
    
    # Test de latence
    $self._test_latency() {
        local target="$1"
        
        echo "Test de latence vers $target..."
        
        # Simulation de ping
        local latencies=(12 15 18 22 19 16)
        local avg_latency=17
        
        echo "Latence moyenne: ${avg_latency}ms"
        echo "Écart type: 3.2ms"
        echo "Perte de paquets: 0%"
        
        # Stockage des métriques
        $self._diagnostic_history["latency_${target}_$(date +%s)"]="$avg_latency"
    }
    
    # Test de bande passante
    $self._test_bandwidth() {
        local target="$1"
        
        echo "Test de bande passante vers $target..."
        
        # Simulation de test de bande passante
        local download_speed="85.3"
        local upload_speed="42.7"
        
        echo "Débit descendant: ${download_speed} Mbps"
        echo "Débit montant: ${upload_speed} Mbps"
        echo "Latence: 23ms"
        
        # Stockage des métriques
        $self._diagnostic_history["bandwidth_${target}_$(date +%s)"]="${download_speed}:${upload_speed}"
    }
    
    # Génération de rapport de diagnostic
    $self.generate_diagnostic_report() {
        local output_file="${1:-network_diagnostic_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DE DIAGNOSTIC RÉSEAU"
            echo "============================"
            echo "Généré le: $(date)"
            echo
            
            echo "CIBLES RÉSEAU CONFIGURÉES"
            echo "========================="
            
            for target_key in "${!$self._network_targets[@]}"; do
                if [[ "$target_key" =~ _type$ ]]; then
                    local target_name="${target_key%_type}"
                    local target_type="${$self._network_targets[$target_key]}"
                    local address="${$self._network_targets[${target_name}_address]}"
                    local info="${$self._network_targets[${target_name}_info]}"
                    
                    echo "Cible: $target_name"
                    echo "  Type: $target_type"
                    echo "  Adresse: $address"
                    echo "  Info: $info"
                    echo
                fi
            done
            
            echo "RÈGLES DE DIAGNOSTIC"
            echo "===================="
            
            for rule_key in "${!$self._diagnostic_rules[@]}"; do
                if [[ "$rule_key" =~ _type$ ]]; then
                    local rule_name="${rule_key%_type}"
                    local rule_type="${$self._diagnostic_rules[$rule_key]}"
                    
                    echo "Règle: $rule_name ($rule_type)"
                fi
            done
            
            echo
            echo "HISTORIQUE DES DIAGNOSTICS"
            echo "=========================="
            
            # Affichage des derniers résultats
            local history_count=0
            for history_key in "${!$self._diagnostic_history[@]}"; do
                if [[ "$history_key" =~ ^[^_]+_[0-9]+$ ]]; then
                    local target_name score passed failed
                    target_name="$(echo "$history_key" | cut -d_ -f1)"
                    score="$(echo "${$self._diagnostic_history[$history_key]}" | cut -d: -f1)"
                    
                    echo "$history_key: Score $score/100"
                    ((history_count++))
                    
                    if (( history_count >= 10 )); then
                        break
                    fi
                fi
            done
            
            echo
            echo "TOPOLOGIE RÉSEAU"
            echo "================"
            
            for topology_key in "${!$self._network_topology[@]}"; do
                local network host timestamp
                network="$(echo "$topology_key" | cut -d_ -f1)"
                host="$(echo "$topology_key" | cut -d_ -f2)"
                timestamp="${$self._network_topology[$topology_key]}"
                
                echo "$network - $host (découvert: $(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S'))"
            done
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            # Analyse des tendances
            local recent_failures=0
            local total_diagnostics=0
            
            for history_key in "${!$self._diagnostic_history[@]}"; do
                if [[ "$history_key" =~ ^[^_]+_[0-9]+$ ]]; then
                    ((total_diagnostics++))
                    local score="${$self._diagnostic_history[$history_key]}"
                    if (( score < 70 )); then
                        ((recent_failures++))
                    fi
                fi
            done
            
            if (( recent_failures > 0 )); then
                local failure_rate=$(( recent_failures * 100 / total_diagnostics ))
                echo "• $failure_rate% des diagnostics récents ont échoué - investigation recommandée"
            fi
            
            if (( total_diagnostics == 0 )); then
                echo "• Aucun diagnostic exécuté - commencer par un diagnostic complet"
            fi
            
            echo "• Surveillance continue recommandée pour détecter les problèmes réseau"
            echo "• Tests de performance réguliers pour mesurer l'évolution des métriques"
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Export de configuration de diagnostic
    $self.export_diagnostic_config() {
        local output_file="$1"
        
        {
            echo "# Configuration Diagnostic Réseau - Exporté le $(date)"
            echo
            echo "# Cibles réseau"
            for target_key in "${!$self._network_targets[@]}"; do
                if [[ "$target_key" =~ _type$ ]]; then
                    local target_name="${target_key%_type}"
                    local target_type="${$self._network_targets[$target_key]}"
                    local address="${$self._network_targets[${target_name}_address]}"
                    local info="${$self._network_targets[${target_name}_info]}"
                    
                    echo "TARGET:$target_name:$target_type:$address:$info"
                fi
            done
            
            echo
            echo "# Règles de diagnostic"
            for rule_key in "${!$self._diagnostic_rules[@]}"; do
                if [[ "$rule_key" =~ _type$ ]]; then
                    local rule_name="${rule_key%_type}"
                    local rule_type="${$self._diagnostic_rules[$rule_key]}"
                    local command="${$self._diagnostic_rules[${rule_name}_command]}"
                    local success="${$self._diagnostic_rules[${rule_name}_success]}"
                    local error="${$self._diagnostic_rules[${rule_name}_error]}"
                    local remediation="${$self._diagnostic_rules[${rule_name}_remediation]}"
                    
                    echo "RULE:$rule_name:$rule_type:$command:$success:$error:$remediation"
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Configuration exportée: $output_file"
    }
}

# Définition des règles de diagnostic réseau
define_network_diagnostic_rules() {
    local diagnostics="$1"
    
    # Tests de connectivité
    $diagnostics.define_diagnostic_rule "ping_test" "connectivity" \
        "ping -c 3 -W 2 {TARGET}" \
        "3 packets transmitted, 3 received" \
        "100% packet loss" \
        "Vérifier la connectivité réseau et les règles firewall"
    
    $diagnostics.define_diagnostic_rule "dns_resolution" "dns" \
        "nslookup {TARGET} 2>/dev/null || host {TARGET}" \
        "address\|has address" \
        "NXDOMAIN\|not found" \
        "Vérifier la configuration DNS ou utiliser une adresse IP directe"
    
    # Tests de services
    $diagnostics.define_diagnostic_rule "http_service" "service" \
        "curl -I --max-time 10 --silent {TARGET} | head -1" \
        "HTTP/[0-9]\+\.[0-9]\+ 200" \
        "000\|timeout" \
        "Vérifier que le service web est démarré et accessible"
    
    $diagnostics.define_diagnostic_rule "ssh_service" "service" \
        "timeout 5 bash -c '</dev/tcp/{TARGET}/22' && echo 'SSH OK' || echo 'SSH FAILED'" \
        "SSH OK" \
        "SSH FAILED" \
        "Vérifier que le service SSH est actif sur le port 22"
    
    # Tests de performance
    $diagnostics.define_diagnostic_rule "latency_check" "latency" \
        "ping -c 5 {TARGET} | tail -1 | awk '{print \$4}' | cut -d/ -f2" \
        "^[0-9]\+\.[0-9]\+$" \
        "" \
        "Latence acceptable (< 100ms recommandée)"
    
    # Tests de sécurité
    $diagnostics.define_diagnostic_rule "open_ports" "port_scan" \
        "timeout 10 nmap -p 21,22,80,443,3306 {TARGET} 2>/dev/null | grep 'open'" \
        "open" \
        "filtered\|closed" \
        "Analyser les ports ouverts et fermer ceux qui ne sont pas nécessaires"
    
    # Tests de topologie
    $diagnostics.define_diagnostic_rule "network_discovery" "topology" \
        "nmap -sn {TARGET}/24 2>/dev/null | grep 'Nmap scan report' | wc -l" \
        "^[0-9]\+$" \
        "" \
        "Nombre d'hôtes découverts sur le réseau"
}

# Démonstration du framework de diagnostic réseau
echo "--- Framework de diagnostic réseau ---"

NetworkDiagnostics "network_diag"

# Définition des règles
define_network_diagnostic_rules "network_diag"

# Enregistrement des cibles
network_diag.register_network_target "google_dns" "host" "8.8.8.8" "Google Public DNS"
network_diag.register_network_target "localhost" "host" "127.0.0.1" "Loopback interface"
network_diag.register_network_target "local_network" "network" "192.168.1.0/24" "Réseau local"

echo
echo "--- Diagnostic complet ---"
network_diag.run_comprehensive_diagnostic "google_dns"

echo
echo "--- Test de performance ---"
network_diag.run_performance_test "google_dns" "comprehensive"

echo
echo "--- Génération de rapport ---"
network_diag.generate_diagnostic_report

echo
echo "--- Export de configuration ---"
network_diag.export_diagnostic_config "/tmp/network_diag_config.txt"

echo "Configuration exportée:"
head -10 /tmp/network_diag_config.txt

# Nettoyage
rm -f network_diagnostic_report_*.txt /tmp/network_diag_config.txt
```

### 1.2 Automatisation des transferts réseau sécurisés

Système de transfert de fichiers distribués avec vérification d'intégrité :

```bash
#!/bin/bash

# Automatisation des transferts réseau sécurisés
echo "=== Automatisation des transferts réseau sécurisés ==="

# Secure Network Transfer Framework
SecureNetworkTransfer() {
    local self="$1"
    
    declare -A $self._transfer_profiles
    declare -A $self._transfer_history
    declare -A $self._integrity_checks
    
    # Définition d'un profil de transfert
    $self.define_transfer_profile() {
        local profile_name="$1"
        local source_type="$2"  # local, remote_ssh, remote_sftp, http, ftp
        local source_path="$3"
        local destination_type="$4"
        local destination_path="$5"
        local transfer_method="${6:-rsync}"  # rsync, scp, sftp, wget, curl
        local options="${7:-}"
        
        $self._transfer_profiles["${profile_name}_source_type"]="$source_type"
        $self._transfer_profiles["${profile_name}_source_path"]="$source_path"
        $self._transfer_profiles["${profile_name}_dest_type"]="$destination_type"
        $self._transfer_profiles["${profile_name}_dest_path"]="$destination_path"
        $self._transfer_profiles["${profile_name}_method"]="$transfer_method"
        $self._transfer_profiles["${profile_name}_options"]="$options"
        
        echo "✓ Profil de transfert défini: $profile_name"
    }
    
    # Exécution d'un transfert sécurisé
    $self.execute_secure_transfer() {
        local profile_name="$1"
        local verify_integrity="${2:-true}"
        
        echo "=== TRANSFERT SÉCURISÉ: $profile_name ==="
        
        local source_type="${$self._transfer_profiles[${profile_name}_source_type]}"
        local source_path="${$self._transfer_profiles[${profile_name}_source_path]}"
        local dest_type="${$self._transfer_profiles[${profile_name}_dest_type]}"
        local dest_path="${$self._transfer_profiles[${profile_name}_dest_path]}"
        local method="${$self._transfer_profiles[${profile_name}_method]}"
        local options="${$self._transfer_profiles[${profile_name}_options]}"
        
        if [[ -z "$source_type" ]]; then
            echo "❌ Profil introuvable: $profile_name"
            return 1
        fi
        
        local transfer_start
        transfer_start="$(date +%s)"
        
        echo "Méthode: $method"
        echo "Source: $source_type:$source_path"
        echo "Destination: $dest_type:$dest_path"
        
        # Calcul de l'intégrité avant transfert si demandé
        local source_checksum=""
        if [[ "$verify_integrity" == "true" ]]; then
            source_checksum="$($self._calculate_integrity "$source_type" "$source_path")"
            echo "Checksum source: ${source_checksum:0:16}..."
        fi
        
        # Exécution du transfert
        if $self._execute_transfer_command "$method" "$source_type" "$source_path" "$dest_type" "$dest_path" "$options"; then
            echo "✓ Transfert réussi"
            
            # Vérification d'intégrité après transfert
            if [[ "$verify_integrity" == "true" && -n "$source_checksum" ]]; then
                local dest_checksum
                dest_checksum="$($self._calculate_integrity "$dest_type" "$dest_path")"
                
                if [[ "$source_checksum" == "$dest_checksum" ]]; then
                    echo "✓ Intégrité vérifiée"
                else
                    echo "❌ Erreur d'intégrité détectée!"
                    echo "  Source: ${source_checksum:0:16}..."
                    echo "  Destination: ${dest_checksum:0:16}..."
                    return 1
                fi
            fi
            
            local transfer_end
            transfer_end="$(date +%s)"
            local duration=$(( transfer_end - transfer_start ))
            
            # Historique
            $self._transfer_history["${profile_name}_$(date +%s)"]="success:$duration:${source_checksum:0:16}"
            
            echo "Durée: ${duration}s"
            return 0
        else
            local transfer_end
            transfer_end="$(date +%s)"
            local duration=$(( transfer_end - transfer_start ))
            
            echo "❌ Transfert échoué (${duration}s)"
            $self._transfer_history["${profile_name}_$(date +%s)"]="failed:$duration"
            return 1
        fi
    }
    
    # Calcul d'intégrité
    $self._calculate_integrity() {
        local location_type="$1"
        local path="$2"
        
        case "$location_type" in
            local)
                if [[ -f "$path" ]]; then
                    sha256sum "$path" 2>/dev/null | cut -d' ' -f1
                elif [[ -d "$path" ]]; then
                    find "$path" -type f -exec sha256sum {} \; 2>/dev/null | sha256sum | cut -d' ' -f1
                else
                    echo "Path not found"
                fi
                ;;
                
            remote_ssh)
                # Simulation pour remote
                echo "remote_checksum_simulation_$(basename "$path")"
                ;;
                
            *)
                echo "unsupported_location_type"
                ;;
        esac
    }
    
    # Exécution de la commande de transfert
    $self._execute_transfer_command() {
        local method="$1"
        local source_type="$2"
        local source_path="$3"
        local dest_type="$4"
        local dest_path="$5"
        local options="$6"
        
        case "$method" in
            rsync)
                $self._execute_rsync "$source_type" "$source_path" "$dest_type" "$dest_path" "$options"
                ;;
                
            scp)
                $self._execute_scp "$source_type" "$source_path" "$dest_type" "$dest_path" "$options"
                ;;
                
            sftp)
                $self._execute_sftp "$source_type" "$source_path" "$dest_type" "$dest_path" "$options"
                ;;
                
            wget)
                $self._execute_wget "$source_path" "$dest_path" "$options"
                ;;
                
            curl)
                $self._execute_curl "$source_path" "$dest_path" "$options"
                ;;
                
            *)
                echo "❌ Méthode de transfert non supportée: $method"
                return 1
                ;;
        esac
    }
    
    # Exécution rsync
    $self._execute_rsync() {
        local source_type="$1" source_path="$2" dest_type="$3" dest_path="$4" options="$5"
        
        # Construction des chemins rsync
        local rsync_source rsync_dest
        
        case "$source_type" in
            local)
                rsync_source="$source_path"
                ;;
            remote_ssh)
                rsync_source="$source_path"  # Assumant user@host:path format
                ;;
        esac
        
        case "$dest_type" in
            local)
                rsync_dest="$dest_path"
                ;;
            remote_ssh)
                rsync_dest="$dest_path"
                ;;
        esac
        
        # Options rsync par défaut pour la sécurité
        local rsync_opts="-avz --checksum $options"
        
        echo "rsync $rsync_opts '$rsync_source' '$rsync_dest'"
        
        # Simulation (en vrai: rsync $rsync_opts "$rsync_source" "$rsync_dest")
        sleep 0.5
        mkdir -p "$(dirname "$dest_path")" 2>/dev/null || true
        echo "Transfert simulé réussi"
        return 0
    }
    
    # Exécution SCP
    $self._execute_scp() {
        local source_type="$1" source_path="$2" dest_type="$3" dest_path="$4" options="$5"
        
        local scp_opts="-r -p $options"  # -r pour récursif, -p pour préserver
        
        echo "scp $scp_opts '$source_path' '$dest_path'"
        
        # Simulation
        sleep 0.3
        mkdir -p "$(dirname "$dest_path")" 2>/dev/null || true
        echo "Transfert SCP simulé réussi"
        return 0
    }
    
    # Exécution SFTP
    $self._execute_sftp() {
        local source_type="$1" source_path="$2" dest_type="$3" dest_path="$4" options="$5"
        
        echo "sftp $options"
        
        # Simulation d'une session SFTP
        sleep 0.4
        mkdir -p "$(dirname "$dest_path")" 2>/dev/null || true
        echo "Transfert SFTP simulé réussi"
        return 0
    }
    
    # Exécution wget
    $self._execute_wget() {
        local source_path="$1" dest_path="$2" options="$3"
        
        local wget_opts="-q $options"  # -q pour quiet
        
        echo "wget $wget_opts -O '$dest_path' '$source_path'"
        
        # Simulation
        sleep 0.2
        mkdir -p "$(dirname "$dest_path")" 2>/dev/null || true
        echo "dummy content" > "$dest_path"
        echo "Téléchargement simulé réussi"
        return 0
    }
    
    # Exécution curl
    $self._execute_curl() {
        local source_path="$1" dest_path="$2" options="$3"
        
        local curl_opts="-s $options"  # -s pour silent
        
        echo "curl $curl_opts -o '$dest_path' '$source_path'"
        
        # Simulation
        sleep 0.2
        mkdir -p "$(dirname "$dest_path")" 2>/dev/null || true
        echo "dummy content" > "$dest_path"
        echo "Téléchargement curl simulé réussi"
        return 0
    }
    
    # Transfert par lots
    $self.batch_transfer() {
        local -a profiles=("$@")
        
        echo "=== TRANSFERT PAR LOTS ==="
        echo "Profils: ${#profiles[@]}"
        echo
        
        local total_transfers=0
        local successful_transfers=0
        local failed_transfers=0
        
        local batch_start
        batch_start="$(date +%s)"
        
        for profile in "${profiles[@]}"; do
            ((total_transfers++))
            
            echo "--- Transfert $total_transfers: $profile ---"
            
            if $self.execute_secure_transfer "$profile" "true"; then
                ((successful_transfers++))
                echo "✅ Réussi"
            else
                ((failed_transfers++))
                echo "❌ Échoué"
            fi
            
            echo
        done
        
        local batch_end
        batch_end="$(date +%s)"
        local batch_duration=$(( batch_end - batch_start ))
        
        echo "=== RÉSULTATS DU LOT ==="
        echo "Durée totale: ${batch_duration}s"
        echo "Transferts: $total_transfers"
        echo "Réussis: $successful_transfers"
        echo "Échoués: $failed_transfers"
        echo "Taux de succès: $(( successful_transfers * 100 / total_transfers ))%"
        
        return $(( failed_transfers > 0 ))
    }
    
    # Synchronisation bidirectionnelle
    $self.bidirectional_sync() {
        local profile_name="$1"
        
        echo "=== SYNCHRONISATION BIDIRECTIONNELLE: $profile_name ==="
        
        local source_type="${$self._transfer_profiles[${profile_name}_source_type]}"
        local source_path="${$self._transfer_profiles[${profile_name}_source_path]}"
        local dest_type="${$self._transfer_profiles[${profile_name}_dest_type]}"
        local dest_path="${$self._transfer_profiles[${profile_name}_dest_path]}"
        
        echo "Synchronisation: $source_type:$source_path ↔ $dest_type:$dest_path"
        
        # Création d'un profil temporaire pour le sens inverse
        local reverse_profile="${profile_name}_reverse"
        $self.define_transfer_profile "$reverse_profile" "$dest_type" "$dest_path" "$source_type" "$source_path" "rsync" "--delete"
        
        # Synchronisation dans les deux sens
        echo "Sens 1: Source → Destination"
        $self.execute_secure_transfer "$profile_name" "false"
        
        echo
        echo "Sens 2: Destination → Source"
        $self.execute_secure_transfer "$reverse_profile" "false"
        
        echo "✓ Synchronisation bidirectionnelle terminée"
    }
    
    # Transfert avec reprise
    $self.transfer_with_resume() {
        local profile_name="$1"
        
        echo "=== TRANSFERT AVEC REPRISE: $profile_name ==="
        
        local max_retries=3
        local retry_count=0
        
        while (( retry_count < max_retries )); do
            ((retry_count++))
            
            echo "Tentative $retry_count/$max_retries"
            
            if $self.execute_secure_transfer "$profile_name" "true"; then
                echo "✓ Transfert réussi à la tentative $retry_count"
                return 0
            else
                if (( retry_count < max_retries )); then
                    local backoff=$(( retry_count * 10 ))
                    echo "Échec - Attente ${backoff}s avant retry..."
                    sleep "$backoff"
                fi
            fi
        done
        
        echo "❌ Transfert échoué après $max_retries tentatives"
        return 1
    }
    
    # Génération de rapport de transfert
    $self.generate_transfer_report() {
        local output_file="${1:-transfer_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DE TRANSFERT RÉSEAU"
            echo "==========================="
            echo "Généré le: $(date)"
            echo
            
            echo "PROFILS DE TRANSFERT CONFIGURÉS"
            echo "==============================="
            
            for profile_key in "${!$self._transfer_profiles[@]}"; do
                if [[ "$profile_key" =~ _source_type$ ]]; then
                    local profile_name="${profile_key%_source_type}"
                    local source_type="${$self._transfer_profiles[$profile_key]}"
                    local source_path="${$self._transfer_profiles[${profile_name}_source_path]}"
                    local dest_type="${$self._transfer_profiles[${profile_name}_dest_type]}"
                    local dest_path="${$self._transfer_profiles[${profile_name}_dest_path]}"
                    local method="${$self._transfer_profiles[${profile_name}_method]}"
                    
                    echo "Profil: $profile_name"
                    echo "  Méthode: $method"
                    echo "  Source: $source_type:$source_path"
                    echo "  Destination: $dest_type:$dest_path"
                    echo
                fi
            done
            
            echo "HISTORIQUE DES TRANSFERTS"
            echo "========================="
            
            for history_key in "${!$self._transfer_history[@]}"; do
                local profile timestamp status duration checksum
                profile="$(echo "$history_key" | sed 's/_[0-9]\+$//')"
                timestamp="$(echo "$history_key" | grep -o '[0-9]\+$')"
                local history_data="${$self._transfer_history[$history_key]}"
                
                status="$(echo "$history_data" | cut -d: -f1)"
                duration="$(echo "$history_data" | cut -d: -f2)"
                checksum="$(echo "$history_data" | cut -d: -f3)"
                
                echo "$(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S'): $profile - $status (${duration}s) [${checksum:-N/A}]"
            done
            
            echo
            echo "STATISTIQUES"
            echo "============"
            
            local total_transfers=0 successful_transfers=0 failed_transfers=0 total_duration=0
            
            for history_key in "${!$self._transfer_history[@]}"; do
                ((total_transfers++))
                local history_data="${$self._transfer_history[$history_key]}"
                local status duration
                status="$(echo "$history_data" | cut -d: -f1)"
                duration="$(echo "$history_data" | cut -d: -f2)"
                
                if [[ "$status" == "success" ]]; then
                    ((successful_transfers++))
                else
                    ((failed_transfers++))
                fi
                
                total_duration=$(( total_duration + duration ))
            done
            
            if (( total_transfers > 0 )); then
                local avg_duration=$(( total_duration / total_transfers ))
                local success_rate=$(( successful_transfers * 100 / total_transfers ))
                
                echo "Total transferts: $total_transfers"
                echo "Réussis: $successful_transfers"
                echo "Échoués: $failed_transfers"
                echo "Taux de succès: ${success_rate}%"
                echo "Durée moyenne: ${avg_duration}s"
            else
                echo "Aucun transfert dans l'historique"
            fi
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            if (( failed_transfers > 0 )); then
                echo "• Analyser les échecs de transfert et ajuster les configurations"
            fi
            
            if (( total_transfers > 0 && success_rate < 95 )); then
                echo "• Taux de succès faible - envisager l'utilisation de méthodes alternatives"
            fi
            
            echo "• Utiliser la vérification d'intégrité pour les transferts critiques"
            echo "• Considérer la compression pour les gros volumes de données"
            echo "• Mettre en place une surveillance continue des transferts"
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Export de configuration de transfert
    $self.export_transfer_config() {
        local output_file="$1"
        
        {
            echo "# Configuration Transfert Réseau - Exporté le $(date)"
            echo
            echo "# Profils de transfert"
            for profile_key in "${!$self._transfer_profiles[@]}"; do
                if [[ "$profile_key" =~ _source_type$ ]]; then
                    local profile_name="${profile_key%_source_type}"
                    local source_type="${$self._transfer_profiles[$profile_key]}"
                    local source_path="${$self._transfer_profiles[${profile_name}_source_path]}"
                    local dest_type="${$self._transfer_profiles[${profile_name}_dest_type]}"
                    local dest_path="${$self._transfer_profiles[${profile_name}_dest_path]}"
                    local method="${$self._transfer_profiles[${profile_name}_method]}"
                    local options="${$self._transfer_profiles[${profile_name}_options]}"
                    
                    echo "PROFILE:$profile_name:$source_type:$source_path:$dest_type:$dest_path:$method:$options"
                fi
            done
            
        } > "$output_file"
        
        echo "✓ Configuration exportée: $output_file"
    }
}

# Démonstration du système de transfert sécurisé
echo "--- Système de transfert sécurisé ---"

SecureNetworkTransfer "secure_transfer"

# Définition des profils de transfert
secure_transfer.define_transfer_profile "backup_configs" "local" "/etc" "remote_ssh" "user@backup.example.com:/backup/configs" "rsync" "--exclude='*.log'"
secure_transfer.define_transfer_profile "sync_website" "remote_ssh" "web@example.com:/var/www" "local" "/tmp/website_mirror" "rsync" "--delete"
secure_transfer.define_transfer_profile "download_tools" "http" "https://example.com/tools.tar.gz" "local" "/tmp/tools.tar.gz" "wget" "--no-check-certificate"
secure_transfer.define_transfer_profile "upload_logs" "local" "/var/log/app" "remote_sftp" "sftp://upload.example.com/logs" "sftp" "-r"

echo
echo "--- Transferts individuels ---"

echo "1. Sauvegarde des configs:"
secure_transfer.execute_secure_transfer "backup_configs"

echo
echo "2. Téléchargement d'outils:"
secure_transfer.execute_secure_transfer "download_tools"

echo
echo "--- Transfert par lots ---"
secure_transfer.batch_transfer "backup_configs" "download_tools"

echo
echo "--- Synchronisation bidirectionnelle ---"
secure_transfer.bidirectional_sync "sync_website"

echo
echo "--- Transfert avec reprise ---"
secure_transfer.transfer_with_resume "upload_logs"

echo
echo "--- Génération de rapport ---"
secure_transfer.generate_transfer_report

echo
echo "--- Export de configuration ---"
secure_transfer.export_transfer_config "/tmp/transfer_config.txt"

echo "Configuration exportée:"
head -5 /tmp/transfer_config.txt

# Nettoyage
rm -f transfer_report_*.txt /tmp/transfer_config.txt /tmp/tools.tar.gz
rm -rf /tmp/website_mirror
```

## Section 2 : Coordination distribuée et verrouillage

### 2.1 Système de verrouillage distribué

Coordination d'accès concurrent aux ressources partagées :

```bash
#!/bin/bash

# Système de verrouillage distribué
echo "=== Système de verrouillage distribué ==="

# Distributed Locking System
DistributedLocking() {
    local self="$1"
    
    declare -A $self._locks
    declare -A $self._lock_holders
    declare -A $self._lock_queue
    
    # Acquisition d'un verrou distribué
    $self.acquire_lock() {
        local lock_name="$1"
        local requester_id="$2"
        local timeout="${3:-30}"
        local lock_type="${4:-exclusive}"  # exclusive, shared
        
        echo "Acquisition verrou: $lock_name (type: $lock_type) par $requester_id"
        
        local lock_key="${lock_name}_${lock_type}"
        
        # Vérification si le verrou est déjà détenu
        if [[ -n "${$self._lock_holders[$lock_key]}" ]]; then
            local current_holder="${$self._lock_holders[$lock_key]}"
            
            if [[ "$current_holder" == "$requester_id" ]]; then
                echo "✓ Verrou déjà détenu par le demandeur"
                return 0
            else
                echo "Verrou détenu par: $current_holder"
                
                # Ajout à la file d'attente
                local queue_key="${lock_key}_queue"
                local queue="${$self._lock_queue[$queue_key]}"
                $self._lock_queue["$queue_key"]="${queue:+$queue;}$requester_id:$(date +%s)"
                
                # Attente avec timeout
                local start_time
                start_time="$(date +%s)"
                
                while (( $(date +%s) - start_time < timeout )); do
                    if [[ "${$self._lock_holders[$lock_key]}" != "$current_holder" ]]; then
                        break
                    fi
                    
                    echo -n "."
                    sleep 1
                done
                
                echo
                
                # Vérification finale
                if [[ "${$self._lock_holders[$lock_key]}" == "$requester_id" ]]; then
                    echo "✓ Verrou acquis après attente"
                    return 0
                else
                    echo "❌ Timeout dépassé - verrou non acquis"
                    # Nettoyage de la file d'attente
                    $self._cleanup_queue "$queue_key" "$requester_id"
                    return 1
                fi
            fi
        else
            # Verrou disponible - acquisition immédiate
            $self._lock_holders["$lock_key"]="$requester_id"
            $self._locks["${lock_key}_acquired"]="$(date +%s)"
            
            echo "✓ Verrou acquis immédiatement"
            return 0
        fi
    }
    
    # Libération d'un verrou
    $self.release_lock() {
        local lock_name="$1"
        local requester_id="$2"
        local lock_type="${3:-exclusive}"
        
        local lock_key="${lock_name}_${lock_type}"
        
        if [[ "${$self._lock_holders[$lock_key]}" == "$requester_id" ]]; then
            # Libération du verrou
            unset $self._lock_holders["$lock_key"]
            local acquired_time="${$self._locks[${lock_key}_acquired]}"
            local held_duration=$(( $(date +%s) - acquired_time ))
            
            echo "✓ Verrou libéré (détenu ${held_duration}s)"
            
            # Attribution à la file d'attente
            $self._process_lock_queue "$lock_key"
            
            return 0
        else
            echo "❌ Impossible de libérer - verrou non détenu par $requester_id"
            return 1
        fi
    }
    
    # Traitement de la file d'attente
    $self._process_lock_queue() {
        local lock_key="$1"
        local queue_key="${lock_key}_queue"
        
        local queue="${$self._lock_queue[$queue_key]}"
        
        if [[ -n "$queue" ]]; then
            # Prendre le premier demandeur dans la file
            local first_request
            first_request="$(echo "$queue" | cut -d';' -f1)"
            
            local next_requester
            next_requester="$(echo "$first_request" | cut -d: -f1)"
            
            echo "Attribution du verrou à: $next_requester (depuis file d'attente)"
            
            # Attribution du verrou
            $self._lock_holders["$lock_key"]="$next_requester"
            $self._locks["${lock_key}_acquired"]="$(date +%s)"
            
            # Mise à jour de la file d'attente
            if [[ "$queue" == *";"* ]]; then
                $self._lock_queue["$queue_key"]="${queue#*;}"
            else
                unset $self._lock_queue["$queue_key"]
            fi
        fi
    }
    
    # Nettoyage de la file d'attente
    $self._cleanup_queue() {
        local queue_key="$1"
        local requester_id="$2"
        
        local queue="${$self._lock_queue[$queue_key]}"
        
        if [[ -n "$queue" ]]; then
            # Suppression du demandeur de la file
            local cleaned_queue=""
            
            IFS=';' read -ra requests <<< "$queue"
            for request in "${requests[@]}"; do
                local req_requester
                req_requester="$(echo "$request" | cut -d: -f1)"
                
                if [[ "$req_requester" != "$requester_id" ]]; then
                    cleaned_queue="${cleaned_queue:+$cleaned_queue;}$request"
                fi
            done
            
            if [[ -n "$cleaned_queue" ]]; then
                $self._lock_queue["$queue_key"]="$cleaned_queue"
            else
                unset $self._lock_queue["$queue_key"]
            fi
        fi
    }
    
    # Statut des verrous
    $self.get_lock_status() {
        local lock_name="${1:-}"
        
        echo "=== STATUT DES VERROUS ==="
        
        if [[ -n "$lock_name" ]]; then
            echo "Filtre: $lock_name"
        fi
        
        local active_locks=0
        
        for lock_key in "${!$self._lock_holders[@]}"; do
            local holder="${$self._lock_holders[$lock_key]}"
            local acquired_time="${$self._locks[${lock_key}_acquired]}"
            
            if [[ -z "$lock_name" || "$lock_key" == *"${lock_name}"* ]]; then
                local held_duration=$(( $(date +%s) - acquired_time ))
                local lock_name_display="${lock_key%_*}"
                local lock_type="${lock_key##*_}"
                
                echo "Verrou: $lock_name_display ($lock_type)"
                echo "  Détenu par: $holder"
                echo "  Depuis: ${held_duration}s"
                
                # File d'attente
                local queue_key="${lock_key}_queue"
                local queue="${$self._lock_queue[$queue_key]}"
                
                if [[ -n "$queue" ]]; then
                    local queue_count
                    queue_count="$(echo "$queue" | grep -o ";" | wc -l)"
                    ((queue_count++))
                    echo "  File d'attente: $queue_count demandeur(s)"
                else
                    echo "  File d'attente: vide"
                fi
                
                echo
                ((active_locks++))
            fi
        done
        
        echo "Total verrous actifs: $active_locks"
    }
    
    # Nettoyage des verrous expirés
    $self.cleanup_expired_locks() {
        local max_lock_time="${1:-3600}"  # 1 heure par défaut
        
        echo "=== NETTOYAGE VERROUS EXPIRÉS ==="
        echo "Timeout maximum: ${max_lock_time}s"
        
        local cleaned=0
        
        for lock_key in "${!$self._locks[@]}"; do
            if [[ "$lock_key" =~ _acquired$ ]]; then
                local acquired_time="${$self._locks[$lock_key]}"
                local held_time=$(( $(date +%s) - acquired_time ))
                
                if (( held_time > max_lock_time )); then
                    local base_key="${lock_key%_acquired}"
                    local lock_name="${base_key%_*}"
                    local lock_type="${base_key##*_}"
                    local holder="${$self._lock_holders[$base_key]}"
                    
                    echo "Verrou expiré: $lock_name ($lock_type) - $holder (${held_time}s)"
                    
                    # Forcer la libération
                    unset $self._lock_holders["$base_key"]
                    unset $self._locks["$lock_key"]
                    
                    # Traiter la file d'attente
                    $self._process_lock_queue "$base_key"
                    
                    ((cleaned++))
                fi
            fi
        done
        
        echo "Verrous nettoyés: $cleaned"
    }
    
    # Verrouillage avec contexte (auto-libération)
    $self.with_lock() {
        local lock_name="$1"
        local requester_id="$2"
        local command="$3"
        local lock_type="${4:-exclusive}"
        local timeout="${5:-30}"
        
        echo "=== EXÉCUTION AVEC VERROU: $lock_name ==="
        
        # Acquisition du verrou
        if ! $self.acquire_lock "$lock_name" "$requester_id" "$timeout" "$lock_type"; then
            echo "❌ Impossible d'acquérir le verrou"
            return 1
        fi
        
        # Exécution de la commande
        echo "Exécution de la commande..."
        local command_start
        command_start="$(date +%s)"
        
        if bash -c "$command"; then
            local command_end
            command_end="$(date +%s)"
            local command_duration=$(( command_end - command_start ))
            
            echo "✓ Commande exécutée avec succès (${command_duration}s)"
            
            # Libération du verrou
            $self.release_lock "$lock_name" "$requester_id" "$lock_type"
            
            return 0
        else
            local exit_code=$?
            local command_end
            command_end="$(date +%s)"
            local command_duration=$(( command_end - command_start ))
            
            echo "❌ Échec de la commande (code: $exit_code, durée: ${command_duration}s)"
            
            # Libération du verrou
            $self.release_lock "$lock_name" "$requester_id" "$lock_type"
            
            return $exit_code
        fi
    }
    
    # Verrou partagé (lecture)
    $self.acquire_shared_lock() {
        local lock_name="$1"
        local requester_id="$2"
        local timeout="${3:-30}"
        
        $self.acquire_lock "$lock_name" "$requester_id" "$timeout" "shared"
    }
    
    # Verrou exclusif (écriture)
    $self.acquire_exclusive_lock() {
        local lock_name="$1"
        local requester_id="$2"
        local timeout="${3:-30}"
        
        $self.acquire_lock "$lock_name" "$requester_id" "$timeout" "exclusive"
    }
    
    # Génération de rapport de verrouillage
    $self.generate_locking_report() {
        local output_file="${1:-locking_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DE VERROUILLAGE DISTRIBUÉ"
            echo "=================================="
            echo "Généré le: $(date)"
            echo
            
            echo "VERROUS ACTIFS"
            echo "=============="
            
            local total_active_locks=0
            local total_queued_requests=0
            
            for lock_key in "${!$self._lock_holders[@]}"; do
                local holder="${$self._lock_holders[$lock_key]}"
                local acquired_time="${$self._locks[${lock_key}_acquired]}"
                local held_duration=$(( $(date +%s) - acquired_time ))
                
                local lock_name="${lock_key%_*}"
                local lock_type="${lock_key##*_}"
                
                echo "Verrou: $lock_name ($lock_type)"
                echo "  Détenu par: $holder"
                echo "  Durée: ${held_duration}s"
                
                # File d'attente
                local queue_key="${lock_key}_queue"
                local queue="${$self._lock_queue[$queue_key]}"
                
                if [[ -n "$queue" ]]; then
                    local queue_length
                    queue_length="$(echo "$queue" | grep -o ";" | wc -l)"
                    ((queue_length++))
                    echo "  En attente: $queue_length"
                    total_queued_requests=$(( total_queued_requests + queue_length ))
                else
                    echo "  En attente: 0"
                fi
                
                echo
                ((total_active_locks++))
            done
            
            echo "STATISTIQUES"
            echo "============"
            
            echo "Verrous actifs: $total_active_locks"
            echo "Requêtes en attente: $total_queued_requests"
            
            # Calcul du taux de contention
            local total_lock_operations=$(( total_active_locks + total_queued_requests ))
            if (( total_lock_operations > 0 )); then
                local contention_rate=$(( total_queued_requests * 100 / total_lock_operations ))
                echo "Taux de contention: ${contention_rate}%"
            fi
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            if (( total_queued_requests > total_active_locks )); then
                echo "• Forte contention détectée - envisager l'optimisation des verrous"
            fi
            
            if (( total_active_locks > 10 )); then
                echo "• Nombre élevé de verrous actifs - vérifier les fuites de verrous"
            fi
            
            echo "• Nettoyage régulier des verrous expirés recommandé"
            echo "• Surveillance continue de la contention"
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
}

# Démonstration du système de verrouillage distribué
echo "--- Système de verrouillage distribué ---"

DistributedLocking "dist_lock"

echo
echo "--- Tests de verrouillage ---"

# Test 1: Acquisition simple
echo "1. Acquisition simple:"
dist_lock.acquire_lock "database_backup" "script_1"

echo
echo "2. Tentative de conflit:"
dist_lock.acquire_lock "database_backup" "script_2" "5" &
pid1=$!

sleep 1

echo "3. Libération:"
dist_lock.release_lock "database_backup" "script_1"

wait $pid1 2>/dev/null

echo
echo "4. Verrou partagé:"
dist_lock.acquire_shared_lock "config_file" "reader_1"
dist_lock.acquire_shared_lock "config_file" "reader_2"

echo
echo "5. Verrou exclusif (doit attendre):"
dist_lock.acquire_exclusive_lock "config_file" "writer_1" "3" &
pid2=$!

sleep 1

echo "6. Libération verrous partagés:"
dist_lock.release_lock "config_file" "reader_1" "shared"
dist_lock.release_lock "config_file" "reader_2" "shared"

wait $pid2 2>/dev/null

echo
echo "7. Exécution avec verrou automatique:"
dist_lock.with_lock "critical_operation" "process_1" "echo 'Opération critique exécutée'; sleep 1"

echo
echo "--- Statut des verrous ---"
dist_lock.get_lock_status

echo
echo "--- Nettoyage verrous expirés ---"
dist_lock.cleanup_expired_locks "10"

echo
echo "--- Génération de rapport ---"
dist_lock.generate_locking_report

# Nettoyage
rm -f locking_report_*.txt
```

### 2.2 Communication inter-processus via files et sockets

Système de messagerie léger pour la coordination distribuée :

```bash
#!/bin/bash

# Communication inter-processus via files et sockets
echo "=== Communication inter-processus via files et sockets ==="

# Inter-Process Communication Framework
IPCFramework() {
    local self="$1"
    
    declare -A $self._message_queues
    declare -A $self._socket_connections
    declare -A $self._message_handlers
    
    # Création d'une file de messages
    $self.create_message_queue() {
        local queue_name="$1"
        local queue_type="${2:-file}"  # file, memory
        local max_size="${3:-100}"
        
        $self._message_queues["${queue_name}_type"]="$queue_type"
        $self._message_queues["${queue_name}_max_size"]="$max_size"
        $self._message_queues["${queue_name}_messages"]=""
        $self._message_queues["${queue_name}_message_count"]=0
        
        echo "✓ File de messages créée: $queue_name (type: $queue_type)"
    }
    
    # Envoi d'un message
    $self.send_message() {
        local queue_name="$1"
        local message="$2"
        local sender="${3:-$(basename "$0")}"
        local priority="${4:-normal}"  # low, normal, high
        
        if [[ -z "${$self._message_queues[${queue_name}_type]}" ]]; then
            echo "❌ File de messages introuvable: $queue_name" >&2
            return 1
        fi
        
        local timestamp
        timestamp="$(date +%s)"
        
        local message_id="${queue_name}_${timestamp}_$$"
        local formatted_message="$priority|$sender|$timestamp|$message_id|$message"
        
        local current_messages="${$self._message_queues[${queue_name}_messages]}"
        local message_count="${$self._message_queues[${queue_name}_message_count]}"
        local max_size="${$self._message_queues[${queue_name}_max_size]}"
        
        # Gestion de la taille maximale
        if (( message_count >= max_size )); then
            # Suppression du message le plus ancien
            current_messages="$(echo "$current_messages" | tail -n +2)"
            ((message_count--))
        fi
        
        # Ajout du nouveau message
        $self._message_queues["${queue_name}_messages"]="${current_messages:+$current_messages$'\n'}$formatted_message"
        $self._message_queues["${queue_name}_message_count"]=$(( message_count + 1 ))
        
        echo "✓ Message envoyé à $queue_name (ID: $message_id)"
        
        # Notification des handlers
        $self._notify_message_handlers "$queue_name" "$formatted_message"
    }
    
    # Réception d'un message
    $self.receive_message() {
        local queue_name="$1"
        local timeout="${2:-5}"
        
        local start_time
        start_time="$(date +%s)"
        
        while (( $(date +%s) - start_time < timeout )); do
            local messages="${$self._message_queues[${queue_name}_messages]}"
            
            if [[ -n "$messages" ]]; then
                # Récupération du premier message
                local first_message
                first_message="$(echo "$messages" | head -1)"
                
                # Mise à jour de la file
                local remaining_messages
                remaining_messages="$(echo "$messages" | tail -n +2)"
                
                $self._message_queues["${queue_name}_messages"]="$remaining_messages"
                $self._message_queues["${queue_name}_message_count"]=$(( ${$self._message_queues[${queue_name}_message_count]} - 1 ))
                
                echo "$first_message"
                return 0
            fi
            
            sleep 0.1
        done
        
        echo "❌ Timeout - Aucun message reçu" >&2
        return 1
    }
    
    # Enregistrement d'un handler de messages
    $self.register_message_handler() {
        local queue_name="$1"
        local handler_function="$2"
        
        $self._message_handlers["${queue_name}_handler"]="$handler_function"
        echo "✓ Handler enregistré pour $queue_name"
    }
    
    # Notification des handlers
    $self._notify_message_handlers() {
        local queue_name="$1"
        local message="$2"
        
        local handler="${$self._message_handlers[${queue_name}_handler]}"
        
        if [[ -n "$handler" ]]; then
            # Exécution du handler en arrière-plan
            (
                eval "$handler" <<< "$message"
            ) &
        fi
    }
    
    # Communication via socket TCP
    $self.create_socket_connection() {
        local socket_name="$1"
        local host="$2"
        local port="$3"
        
        $self._socket_connections["${socket_name}_host"]="$host"
        $self._socket_connections["${socket_name}_port"]="$port"
        $self._socket_connections["${socket_name}_status"]="disconnected"
        
        echo "✓ Connexion socket configurée: $socket_name ($host:$port)"
    }
    
    # Envoi de données via socket
    $self.send_socket_data() {
        local socket_name="$1"
        local data="$2"
        
        local host="${$self._socket_connections[${socket_name}_host]}"
        local port="${$self._socket_connections[${socket_name}_port]}"
        
        if [[ -z "$host" ]]; then
            echo "❌ Socket introuvable: $socket_name" >&2
            return 1
        fi
        
        echo "Envoi vers $host:$port: $data"
        
        # Simulation d'envoi TCP (en vrai utiliser nc/netcat ou bash TCP)
        # echo "$data" | nc "$host" "$port" 2>/dev/null
        
        $self._socket_connections["${socket_name}_status"]="connected"
        
        echo "✓ Données envoyées via socket"
    }
    
    # Écoute sur socket
    $self.listen_socket() {
        local socket_name="$1"
        local port="$2"
        
        echo "=== ÉCOUTE SOCKET: $socket_name (port $port) ==="
        echo "Arrêt: Ctrl+C"
        
        # Simulation d'écoute (en vrai utiliser nc -l)
        while true; do
            echo "En attente de connexions..."
            sleep 2
            
            # Simulation de réception de données
            local simulated_data="SIMULATED_DATA_$(date +%s)"
            echo "Données reçues: $simulated_data"
            
            # Traitement des données
            $self._process_socket_data "$socket_name" "$simulated_data"
        done
    }
    
    # Traitement des données socket
    $self._process_socket_data() {
        local socket_name="$1"
        local data="$2"
        
        echo "Traitement données socket: $data"
        
        # Ici on pourrait dispatcher vers des handlers spécifiques
        # ou convertir en messages de file
    }
    
    # Publication/Souscription (pub/sub)
    $self.create_pubsub_channel() {
        local channel_name="$1"
        
        $self.create_message_queue "${channel_name}_pubsub" "memory" "1000"
        $self._message_queues["${channel_name}_subscribers"]=""
        
        echo "✓ Canal pub/sub créé: $channel_name"
    }
    
    # Souscription à un canal
    $self.subscribe_to_channel() {
        local channel_name="$1"
        local subscriber_id="$2"
        
        local current_subscribers="${$self._message_queues[${channel_name}_subscribers]}"
        $self._message_queues["${channel_name}_subscribers"]="${current_subscribers:+$current_subscribers;}$subscriber_id"
        
        echo "✓ Souscription: $subscriber_id au canal $channel_name"
    }
    
    # Publication sur un canal
    $self.publish_to_channel() {
        local channel_name="$1"
        local message="$2"
        local publisher="${3:-$(basename "$0")}"
        
        local subscribers="${$self._message_queues[${channel_name}_subscribers]}"
        
        if [[ -z "$subscribers" ]]; then
            echo "⚠️ Aucun souscripteur pour le canal $channel_name"
            return 0
        fi
        
        local sent_count=0
        IFS=';' read -ra subscriber_list <<< "$subscribers"
        
        for subscriber in "${subscriber_list[@]}"; do
            # Envoi du message à chaque souscripteur via sa propre file
            local subscriber_queue="${subscriber}_inbox"
            
            if [[ -n "${$self._message_queues[${subscriber_queue}_type]}" ]]; then
                $self.send_message "$subscriber_queue" "$message" "$publisher" "normal"
                ((sent_count++))
            fi
        done
        
        echo "✓ Message publié sur $channel_name ($sent_count souscripteurs)"
    }
    
    # Messagerie point-à-point
    $self.create_peer_to_peer() {
        local peer1="$1"
        local peer2="$2"
        
        $self.create_message_queue "${peer1}_to_${peer2}" "memory" "50"
        $self.create_message_queue "${peer2}_to_${peer1}" "memory" "50"
        
        echo "✓ Canal P2P créé: $peer1 ↔ $peer2"
    }
    
    # Envoi P2P
    $self.send_p2p_message() {
        local from_peer="$1"
        local to_peer="$2"
        local message="$3"
        
        local queue_name="${from_peer}_to_${to_peer}"
        
        if [[ -z "${$self._message_queues[${queue_name}_type]}" ]]; then
            echo "❌ Canal P2P introuvable: $from_peer -> $to_peer" >&2
            return 1
        fi
        
        $self.send_message "$queue_name" "$message" "$from_peer"
    }
    
    # Réception P2P
    $self.receive_p2p_message() {
        local from_peer="$1"
        local to_peer="$2"
        local timeout="${3:-10}"
        
        local queue_name="${from_peer}_to_${to_peer}"
        $self.receive_message "$queue_name" "$timeout"
    }
    
    # Monitoring des communications
    $self.monitor_communications() {
        local duration="${1:-30}"
        
        echo "=== MONITORING COMMUNICATIONS ==="
        echo "Durée: ${duration}s"
        echo
        
        local start_time
        start_time="$(date +%s)"
        
        local initial_queue_count=0
        local initial_socket_count=0
        
        # Comptage initial
        for queue_key in "${!$self._message_queues[@]}"; do
            if [[ "$queue_key" =~ _message_count$ ]]; then
                initial_queue_count=$(( initial_queue_count + ${$self._message_queues[$queue_key]} ))
            fi
        done
        
        for socket_key in "${!$self._socket_connections[@]}"; do
            if [[ "$socket_key" =~ _status$ && "${$self._socket_connections[$socket_key]}" == "connected" ]]; then
                ((initial_socket_count++))
            fi
        done
        
        echo "État initial:"
        echo "  Files de messages: $initial_queue_count messages"
        echo "  Sockets connectés: $initial_socket_count"
        echo
        
        while (( $(date +%s) - start_time < duration )); do
            sleep 5
            
            local current_queue_count=0
            local current_socket_count=0
            
            for queue_key in "${!$self._message_queues[@]}"; do
                if [[ "$queue_key" =~ _message_count$ ]]; then
                    current_queue_count=$(( current_queue_count + ${$self._message_queues[$queue_key]} ))
                fi
            done
            
            for socket_key in "${!$self._socket_connections[@]}"; do
                if [[ "$socket_key" =~ _status$ && "${$self._socket_connections[$socket_key]}" == "connected" ]]; then
                    ((current_socket_count++))
                fi
            done
            
            local queue_diff=$(( current_queue_count - initial_queue_count ))
            local socket_diff=$(( current_socket_count - initial_socket_count ))
            
            echo "[$(date '+%H:%M:%S')] Files: $current_queue_count ($queue_diff), Sockets: $current_socket_count ($socket_diff)"
        done
        
        echo
        echo "✓ Monitoring terminé"
    }
    
    # Génération de rapport IPC
    $self.generate_ipc_report() {
        local output_file="${1:-ipc_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT COMMUNICATION INTER-PROCESSUS"
            echo "===================================="
            echo "Généré le: $(date)"
            echo
            
            echo "FILES DE MESSAGES"
            echo "================="
            
            for queue_key in "${!$self._message_queues[@]}"; do
                if [[ "$queue_key" =~ _type$ ]]; then
                    local queue_name="${queue_key%_type}"
                    local queue_type="${$self._message_queues[$queue_key]}"
                    local message_count="${$self._message_queues[${queue_name}_message_count]}"
                    local max_size="${$self._message_queues[${queue_name}_max_size]}"
                    
                    echo "File: $queue_name ($queue_type)"
                    echo "  Messages: $message_count/$max_size"
                    
                    if [[ "$queue_name" =~ _pubsub$ ]]; then
                        local channel_name="${queue_name%_pubsub}"
                        local subscribers="${$self._message_queues[${channel_name}_subscribers]}"
                        local subscriber_count
                        subscriber_count="$(echo "${subscribers//[^;]}" | wc -c)"
                        ((subscriber_count++))
                        echo "  Souscripteurs: $subscriber_count"
                    fi
                    
                    echo
                fi
            done
            
            echo "CONNEXIONS SOCKET"
            echo "================="
            
            for socket_key in "${!$self._socket_connections[@]}"; do
                if [[ "$socket_key" =~ _host$ ]]; then
                    local socket_name="${socket_key%_host}"
                    local host="${$self._socket_connections[$socket_key]}"
                    local port="${$self._socket_connections[${socket_name}_port]}"
                    local status="${$self._socket_connections[${socket_name}_status]}"
                    
                    echo "Socket: $socket_name"
                    echo "  Cible: $host:$port"
                    echo "  Statut: $status"
                    echo
                fi
            done
            
            echo "HANDLERS DE MESSAGES"
            echo "===================="
            
            for handler_key in "${!$self._message_handlers[@]}"; do
                local queue_name="${handler_key%_handler}"
                echo "Handler pour: $queue_name"
            done
            
            echo
            echo "STATISTIQUES"
            echo "============"
            
            local total_queues=0 total_messages=0 total_sockets=0 active_sockets=0
            
            for queue_key in "${!$self._message_queues[@]}"; do
                if [[ "$queue_key" =~ _type$ ]]; then
                    ((total_queues++))
                    local queue_name="${queue_key%_type}"
                    total_messages=$(( total_messages + ${$self._message_queues[${queue_name}_message_count]} ))
                fi
            done
            
            for socket_key in "${!$self._socket_connections[@]}"; do
                if [[ "$socket_key" =~ _host$ ]]; then
                    ((total_sockets++))
                    
                    local socket_name="${socket_key%_host}"
                    if [[ "${$self._socket_connections[${socket_name}_status]}" == "connected" ]]; then
                        ((active_sockets++))
                    fi
                fi
            done
            
            echo "Files de messages: $total_queues"
            echo "Messages en attente: $total_messages"
            echo "Sockets configurés: $total_sockets"
            echo "Sockets actifs: $active_sockets"
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            if (( total_messages > 100 )); then
                echo "• Nombre élevé de messages en attente - vérifier le traitement"
            fi
            
            if (( active_sockets < total_sockets )); then
                local inactive_sockets=$(( total_sockets - active_sockets ))
                echo "• $inactive_sockets socket(s) inactif(s) - vérifier les connexions"
            fi
            
            echo "• Monitoring régulier des communications recommandé"
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
}

# Démonstration du framework IPC
echo "--- Framework IPC ---"

IPCFramework "ipc_system"

# Création des files de messages
ipc_system.create_message_queue "system_events" "memory" "200"
ipc_system.create_message_queue "user_notifications" "memory" "50"

# Enregistrement d'un handler
ipc_system.register_message_handler "system_events" "
    read -r message
    echo \"[HANDLER] Événement système: \$message\"
"

echo
echo "--- Tests de messagerie ---"

echo "1. Envoi de messages:"
ipc_system.send_message "system_events" "Système démarré" "init_process"
ipc_system.send_message "system_events" "Sauvegarde terminée" "backup_service" "high"
ipc_system.send_message "user_notifications" "Mise à jour disponible" "update_manager" "low"

echo
echo "2. Réception de messages:"
echo "Message de system_events:"
ipc_system.receive_message "system_events"

echo "Message de user_notifications:"
ipc_system.receive_message "user_notifications" "2"

echo
echo "--- Configuration sockets ---"
ipc_system.create_socket_connection "api_server" "localhost" "8080"
ipc_system.create_socket_connection "log_server" "logs.example.com" "514"

echo
echo "3. Envoi via socket:"
ipc_system.send_socket_data "api_server" "GET /health HTTP/1.1"

echo
echo "--- Pub/Sub ---"

ipc_system.create_pubsub_channel "news"
ipc_system.subscribe_to_channel "news" "subscriber_1"
ipc_system.subscribe_to_channel "news" "subscriber_2"

# Création des files inbox pour les souscripteurs
ipc_system.create_message_queue "subscriber_1_inbox" "memory" "10"
ipc_system.create_message_queue "subscriber_2_inbox" "memory" "10"

echo
echo "4. Publication sur canal:"
ipc_system.publish_to_channel "news" "Nouvelle importante: Mise à jour système disponible"

echo "Messages reçus:"
ipc_system.receive_message "subscriber_1_inbox" "1"
ipc_system.receive_message "subscriber_2_inbox" "1"

echo
echo "--- P2P ---"

ipc_system.create_peer_to_peer "process_a" "process_b"

echo "5. Messagerie P2P:"
ipc_system.send_p2p_message "process_a" "process_b" "Hello from A"
ipc_system.send_p2p_message "process_b" "process_a" "Hello from B"

echo "Process A reçoit:"
ipc_system.receive_p2p_message "process_b" "process_a"

echo "Process B reçoit:"
ipc_system.receive_p2p_message "process_a" "process_b"

echo
echo "--- Génération de rapport ---"
ipc_system.generate_ipc_report

# Nettoyage
rm -f ipc_report_*.txt
```

## Conclusion : L'écosystème nerveux distribué

Le scripting réseau en Bash transforme les opérations distribuées d'un ensemble d'outils isolés en un écosystème nerveux vivant où chaque script devient un neurone intelligent capable de communiquer, coordonner, et évoluer collectivement. Cette approche transforme l'administration système d'une série de tâches manuelles en une intelligence distribuée auto-gérante.

**Points clés à retenir :**

1. **Diagnostic réseau automatisé** : Frameworks complets de surveillance et diagnostic avec métriques et rapports
2. **Transferts réseau sécurisés** : Systèmes de transfert avec vérification d'intégrité et stratégies de reprise
3. **Verrouillage distribué** : Coordination d'accès concurrent avec files d'attente et timeouts
4. **Communication IPC** : Messagerie inter-processus via files, sockets, pub/sub et P2P

Dans le prochain chapitre, nous explorerons les techniques avancées de monitoring et observabilité, pour que nos systèmes distribués soient non seulement opérationnels, mais aussi complètement observables.

---

**Exercice pratique :** Créez un système complet de coordination distribuée incluant :
- Diagnostic réseau automatisé avec alertes
- Transferts sécurisés avec vérification d'intégrité
- Verrouillage distribué pour éviter les conflits de ressources
- Communication IPC pour la coordination entre processus

**Réflexion :** Comment concevoiriez-vous une architecture où les scripts Bash communiquent et se coordonnent comme un essaim d'insectes, chaque nœud contribuant à l'intelligence collective du système distribué ?
