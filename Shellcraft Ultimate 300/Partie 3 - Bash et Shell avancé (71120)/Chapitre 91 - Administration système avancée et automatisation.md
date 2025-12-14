# Chapitre 91 - Administration système avancée et automatisation

> "L'administration système n'est pas un ensemble de tâches : c'est un état d'esprit où la proactivité prévient les problèmes avant qu'ils n'existent." - SysAdmin Proverbe

## Introduction : Le gardien vigilant du système

Imaginez-vous en tant que maître d'échecs anticipant les coups de votre adversaire avec plusieurs mouvements d'avance. L'administration système avancée en Bash, c'est exactement cela : anticiper les problèmes, automatiser les maintenances, durcir les systèmes, et maintenir la conformité sans effort apparent.

Dans ce chapitre, nous allons construire des systèmes d'administration proactive : durcissement automatique, vérification de conformité, remédiation intelligente, et maintenance prédictive qui transforme l'administration réactive en orchestration préventive.

## Section 1 : Durcissement automatique du système

### 1.1 Framework de durcissement de sécurité

Système complet de hardening automatique :

```bash
#!/bin/bash

# Framework de durcissement de sécurité
echo "=== Framework de durcissement de sécurité ==="

# Security Hardening Framework
SecurityHardener() {
    local self="$1"
    
    declare -a $self._checks
    declare -A $self._check_configs
    declare -A $self._check_results
    declare -A $self._remediations
    
    # Ajout d'une vérification de sécurité
    $self.add_security_check() {
        local check_id="$1"
        local description="$2"
        local check_command="$3"
        local severity="$4"
        local remediation="$5"
        
        $self._checks+=("$check_id")
        $self._check_configs["${check_id}_description"]="$description"
        $self._check_configs["${check_id}_command"]="$check_command"
        $self._check_configs["${check_id}_severity"]="$severity"
        $self._remediations["$check_id"]="$remediation"
        
        echo "✓ Vérification ajoutée: $check_id ($severity)"
    }
    
    # Exécution de l'audit de sécurité
    $self.run_security_audit() {
        echo "=== AUDIT DE SÉCURITÉ ==="
        echo "Date: $(date)"
        echo
        
        local total_checks=${#$self._checks[@]}
        local passed=0
        local failed=0
        local warnings=0
        
        for check_id in "${$self._checks[@]}"; do
            echo "--- Vérification: $check_id ---"
            
            local description="${$self._check_configs[${check_id}_description]}"
            local command="${$self._check_configs[${check_id}_command]}"
            local severity="${$self._check_configs[${check_id}_severity]}"
            
            echo "Description: $description"
            echo "Sévérité: $severity"
            
            if $self._execute_check "$command"; then
                echo "✓ RÉUSSI"
                $self._check_results["$check_id"]="passed"
                ((passed++))
            else
                local exit_code=$?
                echo "❌ ÉCHEC (code: $exit_code)"
                $self._check_results["$check_id"]="failed"
                ((failed++))
                
                # Affichage de la remédiation
                local remediation="${$self._remediations[$check_id]}"
                if [[ -n "$remediation" ]]; then
                    echo "🔧 Remédiation suggérée:"
                    echo "   $remediation"
                fi
                
                # Pour les sévérités critiques, on pourrait arrêter ici
                if [[ "$severity" == "CRITICAL" ]]; then
                    ((warnings++))
                fi
            fi
            
            echo
        done
        
        # Résumé
        echo "=== RÉSULTATS DE L'AUDIT ==="
        echo "Total vérifications: $total_checks"
        echo "Réussies: $passed"
        echo "Échouées: $failed"
        echo "Avertissements: $warnings"
        
        local compliance=$((passed * 100 / total_checks))
        echo "Score de conformité: ${compliance}%"
        
        if (( compliance >= 90 )); then
            echo "✅ Niveau de sécurité: EXCELLENT"
        elif (( compliance >= 75 )); then
            echo "⚠️  Niveau de sécurité: BON"
        elif (( compliance >= 60 )); then
            echo "⚠️  Niveau de sécurité: MOYEN"
        else
            echo "❌ Niveau de sécurité: CRITIQUE - Action requise immédiate"
        fi
        
        return $(( failed > 0 ))
    }
    
    # Exécution d'une vérification
    $self._execute_check() {
        local command="$1"
        
        # Exécution dans un sous-shell pour l'isolation
        (
            # Variables d'environnement sécurisées
            export PATH="/bin:/usr/bin:/usr/local/bin"
            
            # Exécution du test
            eval "$command"
        )
    }
    
    # Application automatique des remédiations
    $self.apply_remediations() {
        local auto_apply="${1:-false}"
        
        echo "=== APPLICATION DES REMÉDIATIONS ==="
        
        if [[ "$auto_apply" != "true" ]]; then
            echo "Mode interactif - Appuyez sur 'y' pour appliquer chaque remédiation"
        fi
        
        local applied=0
        local skipped=0
        
        for check_id in "${$self._checks[@]}"; do
            local status="${$self._check_results[$check_id]}"
            
            if [[ "$status" == "failed" ]]; then
                local description="${$self._check_configs[${check_id}_description]}"
                local severity="${$self._check_configs[${check_id}_severity]}"
                local remediation="${$self._remediations[$check_id]}"
                
                echo "--- Remédiation: $check_id ---"
                echo "Problème: $description"
                echo "Sévérité: $severity"
                echo "Action: $remediation"
                
                local apply=false
                
                if [[ "$auto_apply" == "true" ]]; then
                    apply=true
                    echo "Application automatique..."
                else
                    read -p "Appliquer cette remédiation ? (y/n): " -n 1 -r
                    echo
                    
                    if [[ $REPLY =~ ^[Yy]$ ]]; then
                        apply=true
                    fi
                fi
                
                if [[ "$apply" == "true" ]]; then
                    if $self._apply_remediation "$remediation"; then
                        echo "✓ Remédiation appliquée"
                        $self._check_results["$check_id"]="remediated"
                        ((applied++))
                    else
                        echo "❌ Échec de la remédiation"
                    fi
                else
                    echo "⚠️  Remédiation ignorée"
                    ((skipped++))
                fi
                
                echo
            fi
        done
        
        echo "Résumé des remédiations:"
        echo "  Appliquées: $applied"
        echo "  Ignorées: $skipped"
    }
    
    # Application d'une remédiation
    $self._apply_remediation() {
        local remediation="$1"
        
        echo "Exécution: $remediation"
        
        # Exécution avec privilèges si nécessaire
        if [[ "$remediation" =~ ^sudo ]]; then
            eval "$remediation"
        else
            eval "$remediation"
        fi
    }
    
    # Génération de rapport de sécurité
    $self.generate_security_report() {
        local output_file="${1:-security_audit_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT D'AUDIT DE SÉCURITÉ"
            echo "============================"
            echo "Généré le: $(date)"
            echo "Système: $(hostname)"
            echo
            
            echo "RÉSUMÉ EXÉCUTIF"
            echo "==============="
            
            local total_checks=${#$self._checks[@]}
            local passed=0 failed=0 remediated=0
            
            for check_id in "${$self._checks[@]}"; do
                case "${$self._check_results[$check_id]}" in
                    passed) ((passed++)) ;;
                    failed) ((failed++)) ;;
                    remediated) ((remediated++)) ;;
                esac
            done
            
            local compliance=$(( (passed + remediated) * 100 / total_checks ))
            
            echo "Total de vérifications: $total_checks"
            echo "Réussies: $passed"
            echo "Échouées: $failed"
            echo "Remédiées: $remediated"
            echo "Score de conformité: ${compliance}%"
            echo
            
            echo "DÉTAIL DES VÉRIFICATIONS"
            echo "========================"
            
            for check_id in "${$self._checks[@]}"; do
                local description="${$self._check_configs[${check_id}_description]}"
                local severity="${$self._check_configs[${check_id}_severity]}"
                local status="${$self._check_results[$check_id]}"
                local remediation="${$self._remediations[$check_id]}"
                
                echo "Vérification: $check_id"
                echo "  Description: $description"
                echo "  Sévérité: $severity"
                echo "  Statut: $status"
                
                if [[ "$status" == "failed" && -n "$remediation" ]]; then
                    echo "  Remédiation: $remediation"
                fi
                
                echo
            done
            
            echo "RECOMMANDATIONS"
            echo "==============="
            
            if (( compliance < 75 )); then
                echo "CRITIQUE: Le niveau de sécurité est insuffisant."
                echo "Actions recommandées:"
                echo "  - Appliquer immédiatement toutes les remédiations disponibles"
                echo "  - Revoir la configuration du système"
                echo "  - Consulter un expert sécurité"
            elif (( compliance < 90 )); then
                echo "ATTENTION: Améliorations de sécurité recommandées."
                echo "Actions recommandées:"
                echo "  - Appliquer les remédiations restantes"
                echo "  - Mettre en place une surveillance continue"
            else
                echo "EXCELLENT: Niveau de sécurité satisfaisant."
                echo "Actions recommandées:"
                echo "  - Maintenir la surveillance régulière"
                echo "  - Garder les systèmes à jour"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Surveillance continue de sécurité
    $self.continuous_monitoring() {
        local interval="${1:-300}"  # 5 minutes par défaut
        
        echo "=== SURVEILLANCE CONTINUE DE SÉCURITÉ ==="
        echo "Intervalle: ${interval}s"
        echo "Arrêt: Ctrl+C"
        echo
        
        local baseline_score=""
        
        while true; do
            echo "[$(date)] Exécution d'audit de sécurité..."
            
            # Audit rapide
            local current_failed=0
            
            for check_id in "${$self._checks[@]}"; do
                local command="${$self._check_configs[${check_id}_command]}"
                
                if ! $self._execute_check "$command" >/dev/null 2>&1; then
                    ((current_failed++))
                fi
            done
            
            local total_checks=${#$self._checks[@]}
            local current_score=$(( (total_checks - current_failed) * 100 / total_checks ))
            
            if [[ -z "$baseline_score" ]]; then
                baseline_score="$current_score"
                echo "Score de référence établi: ${baseline_score}%"
            elif (( current_score < baseline_score - 5 )); then
                echo "🚨 ALERTE: Dégradation de sécurité détectée!"
                echo "  Score actuel: ${current_score}% (référence: ${baseline_score}%)"
                echo "  Échecs détectés: $current_failed"
                
                # Ici on pourrait envoyer une notification
            else
                echo "✓ Sécurité stable - Score: ${current_score}%"
            fi
            
            sleep "$interval"
        done
    }
}

# Fonction de définition des vérifications de sécurité standard
define_standard_security_checks() {
    local hardener="$1"
    
    # Vérifications de comptes et authentification
    $hardener.add_security_check "root_login_disabled" \
        "Vérification que le login root est désactivé" \
        "grep -q '^PermitRootLogin no' /etc/ssh/sshd_config 2>/dev/null || grep -q '^PermitRootLogin prohibit-password' /etc/ssh/sshd_config 2>/dev/null" \
        "HIGH" \
        "sudo sed -i 's/^#PermitRootLogin.*/PermitRootLogin no/' /etc/ssh/sshd_config && sudo systemctl restart sshd"
    
    $hardener.add_security_check "password_policy" \
        "Vérification de la complexité des mots de passe" \
        "grep -q 'minlen.*12' /etc/security/pwquality.conf 2>/dev/null" \
        "MEDIUM" \
        "sudo sed -i 's/minlen.*/minlen = 12/' /etc/security/pwquality.conf"
    
    $hardener.add_security_check "sudo_no_password" \
        "Vérification que sudo nécessite un mot de passe" \
        "! grep -q 'NOPASSWD' /etc/sudoers 2>/dev/null" \
        "HIGH" \
        "sudo sed -i '/NOPASSWD/d' /etc/sudoers"
    
    # Vérifications réseau
    $hardener.add_security_check "ssh_protocol_v2_only" \
        "Vérification que SSH utilise seulement le protocole 2" \
        "grep -q '^Protocol 2' /etc/ssh/sshd_config 2>/dev/null" \
        "HIGH" \
        "sudo sed -i 's/^Protocol.*/Protocol 2/' /etc/ssh/sshd_config && sudo systemctl restart sshd"
    
    $hardener.add_security_check "unnecessary_services" \
        "Vérification qu'aucun service inutile n'est actif" \
        "! systemctl list-units --type=service --state=active | grep -qE '(telnet|ftp|rsh|rlogin)'" \
        "MEDIUM" \
        "Identifier et désactiver manuellement les services non nécessaires"
    
    # Vérifications système de fichiers
    $hardener.add_security_check "no_world_writable" \
        "Vérification de l'absence de fichiers world-writable critiques" \
        "find /etc -type f -perm -002 2>/dev/null | wc -l | grep -q '^0$'" \
        "MEDIUM" \
        "sudo chmod o-w /etc/*"
    
    $hardener.add_security_check "secure_umask" \
        "Vérification du umask sécurisé" \
        "umask | grep -q '0022'" \
        "LOW" \
        "umask 0022"
    
    # Vérifications de logging
    $hardener.add_security_check "auditd_active" \
        "Vérification que auditd est actif" \
        "systemctl is-active auditd >/dev/null 2>&1" \
        "MEDIUM" \
        "sudo systemctl enable auditd && sudo systemctl start auditd"
    
    # Vérifications de mises à jour
    $hardener.add_security_check "automatic_updates" \
        "Vérification des mises à jour automatiques" \
        "systemctl is-active unattended-upgrades >/dev/null 2>&1 || systemctl is-active dnf-automatic >/dev/null 2>&1" \
        "MEDIUM" \
        "sudo apt install unattended-upgrades && sudo dpkg-reconfigure unattended-upgrades"
}

# Démonstration du framework de durcissement
echo "--- Framework de durcissement de sécurité ---"

SecurityHardener "hardener"

# Définition des vérifications standard
define_standard_security_checks "hardener"

# Exécution de l'audit
hardener.run_security_audit

echo
echo "--- Application automatique des remédiations (simulation) ---"
hardener.apply_remediations "false"

echo
echo "--- Génération de rapport ---"
hardener.generate_security_report

# Nettoyage
rm -f security_audit_report_*.txt
```

### 1.2 Durcissement automatique par profil

Configuration de sécurité adaptative selon les rôles :

```bash
#!/bin/bash

# Durcissement automatique par profil
echo "=== Durcissement automatique par profil ==="

# Profile-Based Hardener
ProfileHardener() {
    local self="$1"
    
    declare -A $self._profiles
    declare -A $self._current_hardening
    
    # Définition d'un profil de sécurité
    $self.define_security_profile() {
        local profile_name="$1"
        local description="$2"
        local hardening_rules="$3"
        
        $self._profiles["${profile_name}_description"]="$description"
        $self._profiles["${profile_name}_rules"]="$hardening_rules"
        
        echo "✓ Profil défini: $profile_name"
    }
    
    # Application d'un profil
    $self.apply_profile() {
        local profile_name="$1"
        local dry_run="${2:-false}"
        
        echo "=== APPLICATION DU PROFIL: $profile_name ==="
        
        if [[ "$dry_run" == "true" ]]; then
            echo "🔍 DRY RUN - Simulation uniquement"
        fi
        
        local description="${$self._profiles[${profile_name}_description]}"
        local rules="${$self._profiles[${profile_name}_rules]}"
        
        echo "Description: $description"
        echo
        
        # Parsing et application des règles
        local applied=0
        local failed=0
        
        while IFS=';' read -ra rule_list; do
            for rule in "${rule_list[@]}"; do
                if [[ "$rule" =~ ^([^:]+):(.*)$ ]]; then
                    local rule_type="${BASH_REMATCH[1]}"
                    local rule_config="${BASH_REMATCH[2]}"
                    
                    echo "--- Règle: $rule_type ---"
                    
                    if [[ "$dry_run" == "true" ]]; then
                        echo "Simulation: $rule_config"
                        ((applied++))
                    else
                        if $self._apply_hardening_rule "$rule_type" "$rule_config"; then
                            $self._current_hardening["${profile_name}_${rule_type}"]="applied"
                            ((applied++))
                            echo "✓ Appliqué"
                        else
                            $self._current_hardening["${profile_name}_${rule_type}"]="failed"
                            ((failed++))
                            echo "❌ Échec"
                        fi
                    fi
                fi
            done
        done <<< "$rules"
        
        echo
        echo "Résumé du profil $profile_name:"
        echo "  Règles appliquées: $applied"
        echo "  Échecs: $failed"
        
        if [[ "$dry_run" == "false" ]]; then
            $self._current_hardening["${profile_name}_status"]=$([[ $failed -eq 0 ]] && echo "completed" || echo "partial")
        fi
        
        return $(( failed > 0 ))
    }
    
    # Application d'une règle de durcissement
    $self._apply_hardening_rule() {
        local rule_type="$1"
        local rule_config="$2"
        
        case "$rule_type" in
            disable_service)
                local service_name="$rule_config"
                echo "Désactivation du service: $service_name"
                # systemctl disable "$service_name" && systemctl stop "$service_name"
                ;;
                
            enable_service)
                local service_name="$rule_config"
                echo "Activation du service: $service_name"
                # systemctl enable "$service_name" && systemctl start "$service_name"
                ;;
                
            set_sysctl)
                local param value
                IFS='=' read param value <<< "$rule_config"
                echo "Configuration sysctl: $param = $value"
                # sysctl -w "$param=$value"
                ;;
                
            set_firewall)
                echo "Configuration firewall: $rule_config"
                # Application des règles firewall
                ;;
                
            secure_file)
                local file_path perms owner
                IFS='|' read file_path perms owner <<< "$rule_config"
                echo "Sécurisation fichier: $file_path ($perms, $owner)"
                # chmod "$perms" "$file_path" && chown "$owner" "$file_path"
                ;;
                
            remove_package)
                local package="$rule_config"
                echo "Suppression du package: $package"
                # apt remove "$package" || yum remove "$package"
                ;;
                
            kernel_module)
                local action module
                IFS='|' read action module <<< "$rule_config"
                case "$action" in
                    blacklist)
                        echo "Blacklist module kernel: $module"
                        # echo "blacklist $module" >> /etc/modprobe.d/blacklist.conf
                        ;;
                    disable)
                        echo "Désactivation module: $module"
                        # echo "install $module /bin/false" >> /etc/modprobe.d/disable.conf
                        ;;
                esac
                ;;
                
            audit_rule)
                echo "Ajout règle audit: $rule_config"
                # auditctl -a always,exit -F arch=b64 -S "$rule_config"
                ;;
                
            *)
                echo "Type de règle inconnu: $rule_type"
                return 1
                ;;
        esac
        
        # Simulation de succès (en vrai, vérifier le résultat)
        return 0
    }
    
    # Vérification de conformité au profil
    $self.verify_profile_compliance() {
        local profile_name="$1"
        
        echo "=== VÉRIFICATION CONFORMITÉ PROFIL: $profile_name ==="
        
        local rules="${$self._profiles[${profile_name}_rules]}"
        local compliant=true
        
        while IFS=';' read -ra rule_list; do
            for rule in "${rule_list[@]}"; do
                if [[ "$rule" =~ ^([^:]+):(.*)$ ]]; then
                    local rule_type="${BASH_REMATCH[1]}"
                    local rule_config="${BASH_REMATCH[2]}"
                    
                    if ! $self._verify_rule_compliance "$rule_type" "$rule_config"; then
                        echo "❌ Non conforme: $rule_type - $rule_config"
                        compliant=false
                    else
                        echo "✓ Conforme: $rule_type"
                    fi
                fi
            done
        done <<< "$rules"
        
        if [[ "$compliant" == "true" ]]; then
            echo "✅ Profil $profile_name conforme"
            return 0
        else
            echo "❌ Non-conformités détectées dans $profile_name"
            return 1
        fi
    }
    
    # Vérification de conformité d'une règle
    $self._verify_rule_compliance() {
        local rule_type="$1"
        local rule_config="$2"
        
        case "$rule_type" in
            disable_service)
                local service="$rule_config"
                # systemctl is-enabled "$service" >/dev/null 2>&1 && return 1
                return 0  # Simulation
                ;;
                
            enable_service)
                local service="$rule_config"
                # systemctl is-enabled "$service" >/dev/null 2>&1 && systemctl is-active "$service" >/dev/null 2>&1
                return 0  # Simulation
                ;;
                
            set_sysctl)
                local param value
                IFS='=' read param value <<< "$rule_config"
                # [[ "$(sysctl -n "$param")" == "$value" ]]
                return 0  # Simulation
                ;;
                
            secure_file)
                local file_path perms owner
                IFS='|' read file_path perms owner <<< "$rule_config"
                # [[ "$(stat -c '%a %U' "$file_path")" == "$perms $owner" ]]
                return 0  # Simulation
                ;;
                
            *)
                return 0  # Simulation de conformité
                ;;
        esac
    }
    
    # Rollback d'un profil
    $self.rollback_profile() {
        local profile_name="$1"
        
        echo "=== ROLLBACK PROFIL: $profile_name ==="
        
        # Ici on pourrait implémenter la logique de rollback
        # Pour chaque règle appliquée, annuler les changements
        
        echo "Rollback simulé terminé"
        $self._current_hardening["${profile_name}_status"]="rolled_back"
    }
    
    # Liste des profils
    $self.list_profiles() {
        echo "=== PROFILS DE SÉCURITÉ DISPONIBLES ==="
        
        for profile_key in "${!$self._profiles[@]}"; do
            if [[ "$profile_key" =~ _description$ ]]; then
                local profile_name="${profile_key%_description}"
                local description="${$self._profiles[$profile_key]}"
                local status="${$self._current_hardening[${profile_name}_status]:-not_applied}"
                
                echo "Profil: $profile_name"
                echo "  Description: $description"
                echo "  Statut: $status"
                echo
            fi
        done
    }
    
    # Recommandation de profil selon le rôle
    $self.recommend_profile() {
        local system_role="$1"
        
        echo "=== RECOMMANDATION DE PROFIL ==="
        echo "Rôle système détecté: $system_role"
        
        case "$system_role" in
            web_server)
                echo "✅ Profil recommandé: web_server_secure"
                echo "Raison: Optimisé pour les serveurs web avec sécurité renforcée"
                ;;
                
            database_server)
                echo "✅ Profil recommandé: database_server_secure"
                echo "Raison: Focus sur la protection des données sensibles"
                ;;
                
            workstation)
                echo "✅ Profil recommandé: workstation_balanced"
                echo "Raison: Équilibre entre sécurité et utilisabilité"
                ;;
                
            development)
                echo "⚠️  Profil recommandé: development_lenient"
                echo "Raison: Sécurité réduite pour faciliter le développement"
                ;;
                
            *)
                echo "❓ Profil recommandé: standard_secure"
                echo "Raison: Profil de sécurité standard pour serveurs"
                ;;
        esac
    }
}

# Définition des profils de sécurité
define_security_profiles() {
    local hardener="$1"
    
    # Profil serveur web
    $hardener.define_security_profile "web_server_secure" \
        "Sécurité renforcée pour serveurs web" \
        "disable_service:telnet;disable_service:ftp;enable_service:ufw;set_firewall:allow 80,443;secure_file:/etc/nginx/nginx.conf|644|root;kernel_module:blacklist|usb_storage;audit_rule:execve"
    
    # Profil serveur de base de données
    $hardener.define_security_profile "database_server_secure" \
        "Sécurité pour serveurs de base de données" \
        "disable_service:telnet;disable_service:ftp;enable_service:ufw;set_firewall:allow 3306 from 192.168.1.0/24;secure_file:/etc/mysql/mysql.conf.d/mysqld.cnf|644|mysql;kernel_module:disable|bluetooth;audit_rule:write"
    
    # Profil poste de travail
    $hardener.define_security_profile "workstation_balanced" \
        "Sécurité équilibrée pour postes de travail" \
        "enable_service:ufw;set_firewall:allow 22;secure_file:/etc/ssh/sshd_config|644|root;kernel_module:blacklist|firewire;audit_rule:login"
    
    # Profil développement
    $hardener.define_security_profile "development_lenient" \
        "Sécurité réduite pour développement" \
        "enable_service:ssh;enable_service:ufw;set_firewall:allow 22,80,443,3000-4000"
    
    # Profil serveur standard
    $hardener.define_security_profile "standard_secure" \
        "Sécurité standard pour serveurs" \
        "disable_service:telnet;enable_service:ufw;secure_file:/etc/ssh/sshd_config|644|root;audit_rule:execve"
}

# Démonstration du durcissement par profil
echo "--- Durcissement automatique par profil ---"

ProfileHardener "profile_hardener"

# Définition des profils
define_security_profiles "profile_hardener"

# Liste des profils
profile_hardener.list_profiles

# Recommandation
profile_hardener.recommend_profile "web_server"

echo
echo "--- Application d'un profil (simulation) ---"
profile_hardener.apply_profile "web_server_secure" "true"

echo
echo "--- Vérification de conformité ---"
profile_hardener.verify_profile_compliance "web_server_secure"

echo
echo "--- Rollback simulé ---"
profile_hardener.rollback_profile "web_server_secure"
```

## Section 2 : Automatisation de la maintenance système

### 2.1 Système de maintenance prédictive

Maintenance automatisée basée sur l'analyse prédictive :

```bash
#!/bin/bash

# Système de maintenance prédictive
echo "=== Système de maintenance prédictive ==="

# Predictive Maintenance System
PredictiveMaintenance() {
    local self="$1"
    
    declare -A $self._metrics_history
    declare -A $self._maintenance_rules
    declare -A $self._maintenance_schedule
    
    # Collecte de métriques de santé système
    $self.collect_health_metrics() {
        local timestamp
        timestamp="$(date +%s)"
        
        echo "Collecte des métriques de santé système..."
        
        # Métriques CPU
        local cpu_usage
        cpu_usage="$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')"
        $self._store_metric "cpu_usage" "$cpu_usage" "$timestamp"
        
        # Métriques mémoire
        local mem_usage
        mem_usage="$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')"
        $self._store_metric "memory_usage" "$mem_usage" "$timestamp"
        
        # Métriques disque
        local disk_usage
        disk_usage="$(df / | tail -1 | awk '{print $5}' | sed 's/%//')"
        $self._store_metric "disk_usage" "$disk_usage" "$timestamp"
        
        # Métriques réseau
        local network_errors
        network_errors="$(ip -s link show | grep -A 1 "eth0\|enp" | tail -1 | awk '{print $3}')"
        $self._store_metric "network_errors" "${network_errors:-0}" "$timestamp"
        
        # Métriques de charge système
        local load_avg
        load_avg="$(uptime | awk -F'load average:' '{print $2}' | cut -d, -f1 | xargs)"
        $self._store_metric "system_load" "$load_avg" "$timestamp"
        
        echo "✓ Métriques collectées: CPU=${cpu_usage}%, RAM=${mem_usage}%, Disk=${disk_usage}%, Load=${load_avg}"
    }
    
    # Stockage d'une métrique
    $self._store_metric() {
        local metric_name="$1"
        local value="$2"
        local timestamp="$3"
        
        local history_key="${metric_name}_history"
        local -a history
        local existing_history="${$self._metrics_history[$history_key]}"
        
        if [[ -n "$existing_history" ]]; then
            history=($existing_history)
        fi
        
        # Ajout du nouveau point (timestamp:valeur)
        history+=("${timestamp}:${value}")
        
        # Limitation de l'historique (garder les 100 derniers points)
        if (( ${#history[@]} > 100 )); then
            history=("${history[@]: -100}")
        fi
        
        $self._metrics_history["$history_key"]="${history[*]}"
        $self._metrics_history["${metric_name}_latest"]="$value"
        $self._metrics_history["${metric_name}_last_update"]="$timestamp"
    }
    
    # Définition d'une règle de maintenance
    $self.add_maintenance_rule() {
        local rule_name="$1"
        local condition="$2"
        local action="$3"
        local priority="${4:-medium}"
        local description="$5"
        
        $self._maintenance_rules["${rule_name}_condition"]="$condition"
        $self._maintenance_rules["${rule_name}_action"]="$action"
        $self._maintenance_rules["${rule_name}_priority"]="$priority"
        $self._maintenance_rules["${rule_name}_description"]="$description"
        
        echo "✓ Règle de maintenance ajoutée: $rule_name ($priority)"
    }
    
    # Évaluation des règles de maintenance
    $self.evaluate_maintenance_rules() {
        echo "Évaluation des règles de maintenance..."
        
        local triggered_rules=()
        
        for rule_key in "${!$self._maintenance_rules[@]}"; do
            if [[ "$rule_key" =~ _condition$ ]]; then
                local rule_name="${rule_key%_condition}"
                local condition="${$self._maintenance_rules[$rule_key]}"
                
                if $self._evaluate_condition "$condition"; then
                    local priority="${$self._maintenance_rules[${rule_name}_priority]}"
                    local action="${$self._maintenance_rules[${rule_name}_action]}"
                    local description="${$self._maintenance_rules[${rule_name}_description]}"
                    
                    echo "🚨 MAINTENANCE DÉCLENCHÉE: $rule_name"
                    echo "  Priorité: $priority"
                    echo "  Description: $description"
                    echo "  Action recommandée: $action"
                    
                    triggered_rules+=("$rule_name:$priority:$action")
                    echo
                fi
            fi
        done
        
        # Tri par priorité et exécution
        if (( ${#triggered_rules[@]} > 0 )); then
            $self._prioritize_and_execute "${triggered_rules[@]}"
        else
            echo "✓ Aucune maintenance requise"
        fi
    }
    
    # Évaluation d'une condition
    $self._evaluate_condition() {
        local condition="$1"
        
        # Conditions prédéfinies
        case "$condition" in
            high_cpu_trend)
                $self._check_metric_trend "cpu_usage" 80 3
                ;;
                
            high_memory_usage)
                local mem_usage="${$self._metrics_history[memory_usage_latest]}"
                (( $(echo "$mem_usage > 85" | bc -l) ))
                ;;
                
            low_disk_space)
                local disk_usage="${$self._metrics_history[disk_usage_latest]}"
                (( disk_usage > 90 ))
                ;;
                
            network_errors_increasing)
                $self._check_metric_trend "network_errors" 10 2
                ;;
                
            high_system_load)
                local load_avg="${$self._metrics_history[system_load_latest]}"
                (( $(echo "$load_avg > 2.0" | bc -l) ))
                ;;
                
            custom:*)
                # Condition personnalisée
                local custom_condition="${condition#custom:}"
                eval "$custom_condition"
                ;;
                
            *)
                echo "Condition inconnue: $condition" >&2
                false
                ;;
        esac
    }
    
    # Vérification de tendance métrique
    $self._check_metric_trend() {
        local metric_name="$1"
        local threshold="$2"
        local consecutive="$3"
        
        local history_key="${metric_name}_history"
        local -a history
        local existing_history="${$self._metrics_history[$history_key]}"
        
        if [[ -z "$existing_history" ]]; then
            return 1
        fi
        
        history=($existing_history)
        
        # Vérification des N dernières valeurs
        local count=0
        local i
        for ((i = ${#history[@]} - 1; i >= 0 && count < consecutive; i--)); do
            local value
            value="$(echo "${history[$i]}" | cut -d: -f2)"
            
            if (( $(echo "$value > $threshold" | bc -l) )); then
                ((count++))
            else
                break
            fi
        done
        
        (( count >= consecutive ))
    }
    
    # Priorisation et exécution des maintenances
    $self._prioritize_and_execute() {
        local -a rules=("$@")
        
        # Tri par priorité (critical > high > medium > low)
        local sorted_rules
        sorted_rules="$(for rule in "${rules[@]}"; do
            local priority_num
            case "$(echo "$rule" | cut -d: -f2)" in
                critical) priority_num=4 ;;
                high) priority_num=3 ;;
                medium) priority_num=2 ;;
                low) priority_num=1 ;;
                *) priority_num=0 ;;
            esac
            echo "$priority_num:$rule"
        done | sort -nr | cut -d: -f2-)"
        
        echo "Exécution des maintenances par priorité:"
        
        local executed=0
        while IFS= read -r rule; do
            if [[ -n "$rule" ]]; then
                local rule_name action
                rule_name="$(echo "$rule" | cut -d: -f1)"
                action="$(echo "$rule" | cut -d: -f3-)"
                
                echo "--- Exécution: $rule_name ---"
                if $self._execute_maintenance_action "$action"; then
                    echo "✓ Maintenance exécutée"
                    ((executed++))
                else
                    echo "❌ Échec de maintenance"
                fi
            fi
        done <<< "$sorted_rules"
        
        echo "Maintenances exécutées: $executed"
    }
    
    # Exécution d'une action de maintenance
    $self._execute_maintenance_action() {
        local action="$1"
        
        case "$action" in
            restart_services)
                echo "Redémarrage des services surchargés..."
                # systemctl restart nginx php-fpm 2>/dev/null || true
                sleep 1
                ;;
                
            clear_cache)
                echo "Vidage des caches système..."
                # sync && echo 3 > /proc/sys/vm/drop_caches
                sleep 1
                ;;
                
            cleanup_disk)
                echo "Nettoyage de l'espace disque..."
                # find /tmp -type f -mtime +7 -delete 2>/dev/null || true
                sleep 1
                ;;
                
            network_diagnostic)
                echo "Diagnostic réseau..."
                # ping -c 3 8.8.8.8 >/dev/null
                sleep 1
                ;;
                
            kill_high_cpu)
                echo "Identification et gestion des processus haute CPU..."
                # ps aux --sort=-%cpu | head -6 | tail -5 | awk '{print $2}' | xargs -r kill -TERM 2>/dev/null || true
                sleep 1
                ;;
                
            custom:*)
                local custom_action="${action#custom:}"
                echo "Action personnalisée: $custom_action"
                eval "$custom_action"
                ;;
                
            *)
                echo "Action inconnue: $action"
                return 1
                ;;
        esac
        
        return 0
    }
    
    # Planification des maintenances préventives
    $self.schedule_preventive_maintenance() {
        local schedule_type="$1"
        
        echo "Planification de la maintenance préventive: $schedule_type"
        
        case "$schedule_type" in
            daily)
                $self._schedule_daily_maintenance
                ;;
                
            weekly)
                $self._schedule_weekly_maintenance
                ;;
                
            monthly)
                $self._schedule_monthly_maintenance
                ;;
                
            *)
                echo "Type de planification inconnu: $schedule_type"
                return 1
                ;;
        esac
    }
    
    # Maintenance quotidienne
    $self._schedule_daily_maintenance() {
        echo "Planification maintenance quotidienne:"
        
        # Nettoyage des logs anciens
        echo "  - Nettoyage des logs (>30 jours)"
        # find /var/log -name "*.log.*" -mtime +30 -delete 2>/dev/null || true
        
        # Mise à jour des bases de données de localisation
        echo "  - Mise à jour geoip"
        # geoipupdate 2>/dev/null || true
        
        # Vérification de l'intégrité des paquets
        echo "  - Vérification intégrité paquets"
        # dpkg --verify 2>/dev/null || rpm -Va 2>/dev/null || true
        
        echo "✓ Maintenance quotidienne planifiée"
    }
    
    # Maintenance hebdomadaire
    $self._schedule_weekly_maintenance() {
        echo "Planification maintenance hebdomadaire:"
        
        # Défragmentation si nécessaire
        echo "  - Vérification défragmentation"
        # e4defrag / 2>/dev/null || true
        
        # Nettoyage du cache des paquets
        echo "  - Nettoyage cache paquets"
        # apt clean 2>/dev/null || yum clean all 2>/dev/null || true
        
        # Vérification des systèmes de fichiers
        echo "  - Vérification systèmes de fichiers"
        # fsck -A -y 2>/dev/null || true
        
        echo "✓ Maintenance hebdomadaire planifiée"
    }
    
    # Maintenance mensuelle
    $self._schedule_monthly_maintenance() {
        echo "Planification maintenance mensuelle:"
        
        # Rotation et archivage des logs
        echo "  - Archivage des logs mensuels"
        # logrotate -f /etc/logrotate.conf 2>/dev/null || true
        
        # Audit de sécurité complet
        echo "  - Audit de sécurité"
        # lynis audit system 2>/dev/null || true
        
        # Vérification des mises à jour de sécurité
        echo "  - Vérification mises à jour sécurité"
        # unattended-upgrade 2>/dev/null || yum update --security 2>/dev/null || true
        
        echo "✓ Maintenance mensuelle planifiée"
    }
    
    # Rapport de santé système
    $self.generate_health_report() {
        local output_file="${1:-system_health_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DE SANTÉ SYSTÈME"
            echo "========================"
            echo "Généré le: $(date)"
            echo "Système: $(hostname)"
            echo
            
            echo "MÉTRIQUES ACTUELLES"
            echo "==================="
            
            for metric_key in "${!$self._metrics_history[@]}"; do
                if [[ "$metric_key" =~ _latest$ ]]; then
                    local metric_name="${metric_key%_latest}"
                    local value="${$self._metrics_history[$metric_key]}"
                    local last_update="${$self._metrics_history[${metric_name}_last_update]}"
                    
                    printf "%-20s %-10s (dernière maj: %s)\n" \
                           "$metric_name:" "$value" "$(date -d "@$last_update" '+%H:%M:%S')"
                fi
            done
            
            echo
            echo "TENDANCES"
            echo "========="
            
            # Analyse des tendances
            local cpu_trend
            cpu_trend="$($self._analyze_trend "cpu_usage")"
            echo "CPU: $cpu_trend"
            
            local mem_trend
            mem_trend="$($self._analyze_trend "memory_usage")"
            echo "Mémoire: $mem_trend"
            
            local disk_trend
            disk_trend="$($self._analyze_trend "disk_usage")"
            echo "Disque: $disk_trend"
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            # Recommandations basées sur les métriques
            local cpu_latest="${$self._metrics_history[cpu_usage_latest]}"
            local mem_latest="${$self._metrics_history[memory_usage_latest]}"
            local disk_latest="${$self._metrics_history[disk_usage_latest]}"
            
            if (( $(echo "$cpu_latest > 80" | bc -l) )); then
                echo "⚠️  CPU fréquemment surchargé - envisager upgrade ou optimisation"
            fi
            
            if (( mem_latest > 90 )); then
                echo "⚠️  Mémoire critique - ajouter RAM ou optimiser utilisation"
            fi
            
            if (( disk_latest > 85 )); then
                echo "⚠️  Espace disque faible - nettoyer ou ajouter stockage"
            fi
            
            if [[ "$cpu_trend" == *"augmente"* && "$mem_trend" == *"augmente"* ]]; then
                echo "⚠️  Tendance générale à la dégradation - maintenance préventive recommandée"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport de santé généré: $output_file"
    }
    
    # Analyse de tendance
    $self._analyze_trend() {
        local metric_name="$1"
        
        local history_key="${metric_name}_history"
        local -a history
        local existing_history="${$self._metrics_history[$history_key]}"
        
        if [[ -z "$existing_history" ]]; then
            echo "données insuffisantes"
            return
        fi
        
        history=($existing_history)
        
        if (( ${#history[@]} < 5 )); then
            echo "données insuffisantes"
            return
        fi
        
        # Calcul de la tendance (régression linéaire simple)
        local n="${#history[@]}"
        local sum_x=0 sum_y=0 sum_xy=0 sum_x2=0
        
        for ((i=0; i<n; i++)); do
            local value
            value="$(echo "${history[$i]}" | cut -d: -f2)"
            
            sum_x=$((sum_x + i))
            sum_y=$(echo "$sum_y + $value" | bc -l)
            sum_xy=$(echo "$sum_xy + ($i * $value)" | bc -l)
            sum_x2=$((sum_x2 + (i * i)))
        done
        
        local slope=$(echo "($n * $sum_xy - $sum_x * $sum_y) / ($n * $sum_x2 - $sum_x * $sum_x)" | bc -l)
        
        if (( $(echo "$slope > 0.5" | bc -l) )); then
            echo "en augmentation"
        elif (( $(echo "$slope < -0.5" | bc -l) )); then
            echo "en diminution"
        else
            echo "stable"
        fi
    }
}

# Définition des règles de maintenance prédictive
define_predictive_rules() {
    local maintenance="$1"
    
    $maintenance.add_maintenance_rule "cpu_overload_protection" \
        "high_cpu_trend" \
        "restart_services" \
        "high" \
        "Protection contre surcharge CPU persistante"
    
    $maintenance.add_maintenance_rule "memory_management" \
        "high_memory_usage" \
        "clear_cache" \
        "medium" \
        "Gestion automatique de la mémoire"
    
    $maintenance.add_maintenance_rule "disk_space_maintenance" \
        "low_disk_space" \
        "cleanup_disk" \
        "high" \
        "Maintenance de l'espace disque"
    
    $maintenance.add_maintenance_rule "network_stability" \
        "network_errors_increasing" \
        "network_diagnostic" \
        "medium" \
        "Stabilité du réseau"
    
    $maintenance.add_maintenance_rule "load_management" \
        "high_system_load" \
        "kill_high_cpu" \
        "high" \
        "Gestion de la charge système"
}

# Démonstration de la maintenance prédictive
echo "--- Système de maintenance prédictive ---"

PredictiveMaintenance "predictive_maint"

# Définition des règles
define_predictive_rules "predictive_maint"

# Collecte initiale de métriques
predictive_maint.collect_health_metrics

echo
echo "--- Évaluation des règles de maintenance ---"
predictive_maint.evaluate_maintenance_rules

echo
echo "--- Planification maintenance préventive ---"
predictive_maint.schedule_preventive_maintenance "daily"

echo
echo "--- Génération rapport de santé ---"
predictive_maint.generate_health_report

# Nettoyage
rm -f system_health_report_*.txt
```

### 2.2 Automatisation des sauvegardes intelligentes

Système de sauvegarde adaptatif avec déduplication et compression :

```bash
#!/bin/bash

# Automatisation des sauvegardes intelligentes
echo "=== Automatisation des sauvegardes intelligentes ==="

# Intelligent Backup System
IntelligentBackup() {
    local self="$1"
    
    declare -A $self._backup_configs
    declare -A $self._backup_history
    declare -a $self._backup_schedule
    
    # Configuration d'une sauvegarde
    $self.configure_backup() {
        local name="$1"
        local source="$2"
        local destination="$3"
        local schedule="$4"
        local options="$5"
        
        $self._backup_configs["${name}_source"]="$source"
        $self._backup_configs["${name}_destination"]="$destination"
        $self._backup_configs["${name}_schedule"]="$schedule"
        $self._backup_configs["${name}_options"]="$options"
        $self._backup_configs["${name}_last_run"]="never"
        $self._backup_configs["${name}_status"]="configured"
        
        echo "✓ Sauvegarde configurée: $name"
    }
    
    # Analyse des données à sauvegarder
    $self.analyze_backup_data() {
        local name="$1"
        
        local source="${$self._backup_configs[${name}_source]}"
        
        if [[ ! -d "$source" && ! -f "$source" ]]; then
            echo "❌ Source inexistante: $source"
            return 1
        fi
        
        echo "Analyse des données source: $source"
        
        # Statistiques de base
        local file_count dir_count total_size
        
        if [[ -d "$source" ]]; then
            file_count="$(find "$source" -type f | wc -l)"
            dir_count="$(find "$source" -type d | wc -l)"
            total_size="$(du -sb "$source" 2>/dev/null | cut -f1)"
        else
            file_count="1"
            dir_count="0"
            total_size="$(stat -f%z "$source" 2>/dev/null || stat -c%s "$source")"
        fi
        
        echo "  Fichiers: $file_count"
        echo "  Répertoires: $dir_count"
        echo "  Taille totale: $total_size octets"
        
        # Analyse des types de fichiers
        echo "  Types de fichiers les plus courants:"
        find "$source" -type f -exec basename {} \; | sed 's/.*\.//' | sort | uniq -c | sort -nr | head -5 | sed 's/^/    /'
        
        # Fichiers volumineux
        echo "  Fichiers les plus volumineux:"
        find "$source" -type f -exec ls -lh {} \; 2>/dev/null | sort -k5 -hr | head -3 | sed 's/^/    /'
        
        # Recommandations
        local compression_recommended="false"
        local deduplication_recommended="false"
        
        if (( total_size > 1073741824 )); then  # > 1GB
            compression_recommended="true"
        fi
        
        if (( file_count > 1000 )); then
            deduplication_recommended="true"
        fi
        
        echo "  Recommandations:"
        if [[ "$compression_recommended" == "true" ]]; then
            echo "    ✓ Compression recommandée (données volumineuses)"
        fi
        if [[ "$deduplication_recommended" == "true" ]]; then
            echo "    ✓ Déduplication recommandée (nombreux fichiers)"
        fi
        
        # Stockage des métriques
        $self._backup_configs["${name}_file_count"]="$file_count"
        $self._backup_configs["${name}_total_size"]="$total_size"
        $self._backup_configs["${name}_compression_recommended"]="$compression_recommended"
        $self._backup_configs["${name}_deduplication_recommended"]="$deduplication_recommended"
    }
    
    # Exécution d'une sauvegarde intelligente
    $self.execute_intelligent_backup() {
        local name="$1"
        
        echo "=== EXÉCUTION SAUVEGARDE INTELLIGENTE: $name ==="
        
        local source="${$self._backup_configs[${name}_source]}"
        local destination="${$self._backup_configs[${name}_destination]}"
        local options="${$self._backup_configs[${name}_options]}"
        
        # Analyse pré-sauvegarde
        $self.analyze_backup_data "$name"
        
        local start_time
        start_time="$(date +%s)"
        
        # Création du répertoire de destination
        mkdir -p "$destination"
        
        # Génération du nom de sauvegarde
        local backup_name="${name}_$(date +%Y%m%d_%H%M%S)"
        local backup_path="$destination/${backup_name}"
        
        echo "Création de la sauvegarde: $backup_path"
        
        # Stratégie de sauvegarde basée sur l'analyse
        local compression_recommended="${$self._backup_configs[${name}_compression_recommended]}"
        local deduplication_recommended="${$self._backup_configs[${name}_deduplication_recommended]}"
        local file_count="${$self._backup_configs[${name}_file_count]}"
        local total_size="${$self._backup_configs[${name}_total_size]}"
        
        local backup_command=""
        
        if [[ "$compression_recommended" == "true" ]]; then
            echo "✓ Utilisation de la compression (données volumineuses)"
            backup_command="tar -czf '${backup_path}.tar.gz' -C '$source' ."
        elif [[ "$deduplication_recommended" == "true" ]]; then
            echo "✓ Utilisation de rsync (optimisé pour nombreux fichiers)"
            backup_command="rsync -av --delete '$source/' '$backup_path/'"
        else
            echo "✓ Utilisation de tar standard"
            backup_command="tar -cf '${backup_path}.tar' -C '$source' ."
        fi
        
        # Exécution de la sauvegarde
        echo "Commande: $backup_command"
        
        if eval "$backup_command"; then
            local end_time
            end_time="$(date +%s)"
            local duration=$((end_time - start_time))
            
            # Calcul de la taille de la sauvegarde
            local backup_size
            if [[ -f "${backup_path}.tar.gz" ]]; then
                backup_size="$(stat -f%z "${backup_path}.tar.gz" 2>/dev/null || stat -c%s "${backup_path}.tar.gz")"
            elif [[ -d "$backup_path" ]]; then
                backup_size="$(du -sb "$backup_path" 2>/dev/null | cut -f1)"
            else
                backup_size="$(stat -f%z "${backup_path}.tar" 2>/dev/null || stat -c%s "${backup_path}.tar")"
            fi
            
            # Calcul du ratio de compression
            local compression_ratio="N/A"
            if [[ "$compression_recommended" == "true" ]]; then
                compression_ratio="$(echo "scale=2; $total_size * 100 / $backup_size" | bc)%)"
            fi
            
            echo "✓ Sauvegarde réussie"
            echo "  Durée: ${duration}s"
            echo "  Taille: $backup_size octets"
            echo "  Ratio de compression: $compression_ratio"
            
            # Mise à jour de l'historique
            $self._backup_configs["${name}_last_run"]="$(date +%s)"
            $self._backup_configs["${name}_status"]="completed"
            $self._backup_history["${name}_${start_time}"]="$backup_name:$duration:$backup_size:$compression_ratio"
            
            # Nettoyage des anciennes sauvegardes
            $self._cleanup_old_backups "$name" "$destination"
            
            return 0
        else
            echo "❌ Échec de la sauvegarde"
            $self._backup_configs["${name}_status"]="failed"
            return 1
        fi
    }
    
    # Nettoyage des anciennes sauvegardes
    $self._cleanup_old_backups() {
        local name="$1"
        local destination="$2"
        
        # Politique de rétention (garder les 7 dernières sauvegardes)
        local retention_count=7
        
        local backup_files
        backup_files="$(ls -t "${destination}/${name}_"* 2>/dev/null | tail -n +$((retention_count + 1)))"
        
        if [[ -n "$backup_files" ]]; then
            echo "Nettoyage des anciennes sauvegardes:"
            echo "$backup_files" | while read -r old_backup; do
                echo "  Suppression: $(basename "$old_backup")"
                rm -rf "$old_backup"
            done
        fi
    }
    
    # Vérification d'intégrité des sauvegardes
    $self.verify_backup_integrity() {
        local name="$1"
        
        echo "=== VÉRIFICATION INTÉGRITÉ SAUVEGARDE: $name ==="
        
        local destination="${$self._backup_configs[${name}_destination]}"
        local latest_backup
        latest_backup="$(ls -t "${destination}/${name}_"* 2>/dev/null | head -1)"
        
        if [[ -z "$latest_backup" ]]; then
            echo "❌ Aucune sauvegarde trouvée"
            return 1
        fi
        
        echo "Vérification de: $(basename "$latest_backup")"
        
        # Vérification selon le type de sauvegarde
        if [[ "$latest_backup" =~ \.tar\.gz$ ]]; then
            echo "Vérification archive compressée..."
            if tar -tzf "$latest_backup" >/dev/null 2>&1; then
                echo "✓ Archive intacte"
                return 0
            else
                echo "❌ Archive corrompue"
                return 1
            fi
        elif [[ -d "$latest_backup" ]]; then
            echo "Vérification répertoire..."
            local file_count
            file_count="$(find "$latest_backup" -type f | wc -l)"
            echo "✓ $file_count fichiers présents"
            return 0
        else
            echo "Vérification fichier tar..."
            if tar -tf "$latest_backup" >/dev/null 2>&1; then
                echo "✓ Archive intacte"
                return 0
            else
                echo "❌ Archive corrompue"
                return 1
            fi
        fi
    }
    
    # Restauration depuis sauvegarde
    $self.restore_from_backup() {
        local name="$1"
        local restore_destination="$2"
        local backup_timestamp="${3:-latest}"
        
        echo "=== RESTAURATION SAUVEGARDE: $name ==="
        echo "Destination: $restore_destination"
        
        local destination="${$self._backup_configs[${name}_destination]}"
        local backup_file=""
        
        if [[ "$backup_timestamp" == "latest" ]]; then
            backup_file="$(ls -t "${destination}/${name}_"* 2>/dev/null | head -1)"
        else
            backup_file="${destination}/${name}_${backup_timestamp}"
        fi
        
        if [[ ! -e "$backup_file" ]]; then
            echo "❌ Sauvegarde introuvable: $backup_file"
            return 1
        fi
        
        echo "Restauration depuis: $(basename "$backup_file")"
        
        mkdir -p "$restore_destination"
        
        # Restauration selon le type
        if [[ "$backup_file" =~ \.tar\.gz$ ]]; then
            if tar -xzf "$backup_file" -C "$restore_destination"; then
                echo "✓ Restauration réussie"
                return 0
            fi
        elif [[ -d "$backup_file" ]]; then
            if cp -r "$backup_file"/* "$restore_destination/"; then
                echo "✓ Restauration réussie"
                return 0
            fi
        else
            if tar -xf "$backup_file" -C "$restore_destination"; then
                echo "✓ Restauration réussie"
                return 0
            fi
        fi
        
        echo "❌ Échec de la restauration"
        return 1
    }
    
    # Planification automatique
    $self.schedule_automatic_backups() {
        echo "=== PLANIFICATION SAUVEGARDES AUTOMATIQUES ==="
        
        # Création de tâches cron
        local cron_file="/tmp/backup_cron"
        
        for backup_key in "${!$self._backup_configs[@]}"; do
            if [[ "$backup_key" =~ _schedule$ ]]; then
                local backup_name="${backup_key%_schedule}"
                local schedule="${$self._backup_configs[$backup_key]}"
                
                # Conversion du schedule en cron
                local cron_schedule=""
                case "$schedule" in
                    hourly)
                        cron_schedule="0 * * * *"
                        ;;
                    daily)
                        cron_schedule="0 2 * * *"
                        ;;
                    weekly)
                        cron_schedule="0 2 * * 0"
                        ;;
                    monthly)
                        cron_schedule="0 2 1 * *"
                        ;;
                    *)
                        continue
                        ;;
                esac
                
                # Ajout à crontab
                echo "$cron_schedule $0 --backup $backup_name" >> "$cron_file"
                echo "✓ Planification: $backup_name ($schedule)"
            fi
        done
        
        if [[ -f "$cron_file" ]]; then
            echo "Installation des tâches cron..."
            crontab "$cron_file" 2>/dev/null || echo "⚠️  Impossible d'installer crontab (droits insuffisants)"
            rm -f "$cron_file"
        fi
    }
    
    # Rapport des sauvegardes
    $self.generate_backup_report() {
        local output_file="${1:-backup_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DES SAUVEGARDES"
            echo "======================="
            echo "Généré le: $(date)"
            echo
            
            echo "CONFIGURATIONS"
            echo "=============="
            
            for backup_key in "${!$self._backup_configs[@]}"; do
                if [[ "$backup_key" =~ _source$ ]]; then
                    local backup_name="${backup_key%_source}"
                    local source="${$self._backup_configs[$backup_key]}"
                    local destination="${$self._backup_configs[${backup_name}_destination]}"
                    local schedule="${$self._backup_configs[${backup_name}_schedule]}"
                    local last_run="${$self._backup_configs[${backup_name}_last_run]}"
                    local status="${$self._backup_configs[${backup_name}_status]}"
                    
                    echo "Sauvegarde: $backup_name"
                    echo "  Source: $source"
                    echo "  Destination: $destination"
                    echo "  Planification: $schedule"
                    echo "  Dernière exécution: $(date -d "@$last_run" '+%Y-%m-%d %H:%M:%S' 2>/dev/null || echo 'jamais')"
                    echo "  Statut: $status"
                    echo
                fi
            done
            
            echo "HISTORIQUE DES EXÉCUTIONS"
            echo "========================"
            
            for history_key in "${!$self._backup_history[@]}"; do
                local backup_name="${history_key%%_*}"
                local timestamp="${history_key#*_}"
                local details="${$self._backup_history[$history_key]}"
                
                echo "$(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S'): $backup_name - $details"
            done
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            # Analyse des métriques
            local failed_backups=0
            local old_backups=0
            local current_time="$(date +%s)"
            
            for backup_key in "${!$self._backup_configs[@]}"; do
                if [[ "$backup_key" =~ _status$ ]]; then
                    local status="${$self._backup_configs[$backup_key]}"
                    if [[ "$status" == "failed" ]]; then
                        ((failed_backups++))
                    fi
                fi
                
                if [[ "$backup_key" =~ _last_run$ ]]; then
                    local last_run="${$self._backup_configs[$backup_key]}"
                    if [[ "$last_run" != "never" && $((current_time - last_run)) -gt 604800 ]]; then  # 7 jours
                        ((old_backups++))
                    fi
                fi
            done
            
            if (( failed_backups > 0 )); then
                echo "⚠️  $failed_backups sauvegarde(s) en échec - intervention requise"
            fi
            
            if (( old_backups > 0 )); then
                echo "⚠️  $old_backups sauvegarde(s) obsolète(s) - exécution recommandée"
            fi
            
            if (( failed_backups == 0 && old_backups == 0 )); then
                echo "✅ État des sauvegardes satisfaisant"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Liste des sauvegardes
    $self.list_backups() {
        echo "=== SAUVEGARDES CONFIGURÉES ==="
        
        for backup_key in "${!$self._backup_configs[@]}"; do
            if [[ "$backup_key" =~ _source$ ]]; then
                local backup_name="${backup_key%_source}"
                local source="${$self._backup_configs[$backup_key]}"
                local destination="${$self._backup_configs[${backup_name}_destination]}"
                local schedule="${$self._backup_configs[${backup_name}_schedule]}"
                
                echo "$backup_name:"
                echo "  Source: $source"
                echo "  Destination: $destination"
                echo "  Planification: $schedule"
                echo
            fi
        done
    }
}

# Démonstration du système de sauvegarde intelligent
echo "--- Automatisation des sauvegardes intelligentes ---"

IntelligentBackup "smart_backup"

# Configuration des sauvegardes
smart_backup.configure_backup "web_app" "/var/www/html" "/backup/web" "daily" "compression=yes"
smart_backup.configure_backup "database" "/var/lib/mysql" "/backup/db" "hourly" "deduplication=yes"
smart_backup.configure_backup "user_data" "/home" "/backup/users" "weekly" "encryption=yes"

# Liste des sauvegardes
smart_backup.list_backups

echo
echo "--- Analyse des données ---"
smart_backup.analyze_backup_data "web_app"

echo
echo "--- Exécution d'une sauvegarde ---"
mkdir -p /tmp/backup_demo
smart_backup.configure_backup "demo" "/tmp" "/tmp/backup_demo" "manual" "compression=no"

# Création de données de test
echo "Fichier de test pour la démonstration" > /tmp/test_file.txt
mkdir -p /tmp/test_dir
for i in {1..10}; do
    echo "Contenu du fichier $i" > "/tmp/test_dir/file_$i.txt"
done

smart_backup.execute_intelligent_backup "demo"

echo
echo "--- Vérification d'intégrité ---"
smart_backup.verify_backup_integrity "demo"

echo
echo "--- Restauration ---"
mkdir -p /tmp/restore_demo
smart_backup.restore_from_backup "demo" "/tmp/restore_demo"

echo "Contenu restauré:"
ls -la /tmp/restore_demo/

echo
echo "--- Rapport des sauvegardes ---"
smart_backup.generate_backup_report

# Nettoyage
rm -rf /tmp/backup_demo /tmp/restore_demo /tmp/test_file.txt /tmp/test_dir
rm -f backup_report_*.txt
```

## Conclusion : L'administration comme intelligence artificielle

L'administration système avancée en Bash transcende la simple exécution de commandes : elle devient une intelligence proactive capable d'anticiper les problèmes, d'optimiser automatiquement les performances, et de maintenir la stabilité sans intervention humaine constante.

**Points clés à retenir :**

1. **Durcissement automatique** : Frameworks de sécurité avec profils adaptatifs et remédiation automatique
2. **Maintenance prédictive** : Systèmes d'analyse de tendances et d'intervention préventive
3. **Sauvegardes intelligentes** : Stratégies adaptatives avec compression et déduplication

Dans le chapitre suivant, nous explorerons les techniques avancées de conformité et d'audit, pour que vos systèmes soient non seulement sécurisés, mais aussi entièrement conformes aux réglementations et standards.

---

**Exercice pratique :** Créez un système d'administration système complet incluant :
- Framework de durcissement automatique avec profils de sécurité
- Système de maintenance prédictive avec analyse de tendances
- Automatisation des sauvegardes avec stratégie adaptative
- Surveillance continue et génération de rapports

**Réflexion :** Comment adapteriez-vous ces techniques d'administration avancée pour gérer une flotte de serveurs cloud avec auto-scaling et déploiement continu ?
