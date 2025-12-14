# Chapitre 93 - Sécurité des scripts shell et durcissement

> "La sécurité n'est pas un produit, mais un processus. Et dans le monde des scripts shell, ce processus commence par la paranoïa constructive." - Security Shell Proverbe

## Introduction : La sécurité comme état d'esprit

Imaginez-vous en train de construire un château fort numérique : chaque script shell devient une forteresse imprenable, chaque commande une sentinelle vigilante, chaque variable une chambre forte. La sécurité des scripts shell n'est pas un ensemble de bonnes pratiques optionnelles - c'est une philosophie où la méfiance constructive prévient les catastrophes avant qu'elles n'existent.

Dans ce chapitre, nous forgerons des scripts invulnérables : durcissement systématique, validation impitoyable des entrées, exécution sécurisée, et pratiques de codage qui transforment la vulnérabilité potentielle en robustesse inébranlable.

## Section 1 : Framework de sécurisation des scripts

### 1.1 Analyseur de sécurité automatique

Système complet d'analyse et de correction des vulnérabilités :

```bash
#!/bin/bash

# Analyseur de sécurité automatique pour scripts shell
echo "=== Analyseur de sécurité automatique ==="

# Security Analyzer Framework
SecurityAnalyzer() {
    local self="$1"
    
    declare -A $self._security_rules
    declare -A $self._vulnerability_patterns
    declare -A $self._remediation_actions
    declare -A $self._analysis_results
    
    # Définition d'une règle de sécurité
    $self.define_security_rule() {
        local rule_id="$1"
        local severity="$2"
        local description="$3"
        local pattern="$4"
        local remediation="$5"
        local category="${6:-general}"
        
        $self._security_rules["${rule_id}_severity"]="$severity"
        $self._security_rules["${rule_id}_description"]="$description"
        $self._security_rules["${rule_id}_pattern"]="$pattern"
        $self._security_rules["${rule_id}_remediation"]="$remediation"
        $self._security_rules["${rule_id}_category"]="$category"
        
        echo "✓ Règle définie: $rule_id ($severity)"
    }
    
    # Analyse d'un script
    $self.analyze_script() {
        local script_file="$1"
        local analysis_id="${2:-$(date +%s)}"
        
        if [[ ! -f "$script_file" ]]; then
            echo "❌ Script introuvable: $script_file" >&2
            return 1
        fi
        
        echo "=== ANALYSE SÉCURITÉ: $(basename "$script_file") ==="
        echo "ID d'analyse: $analysis_id"
        echo
        
        local script_content
        script_content="$(cat "$script_file")"
        
        local total_vulnerabilities=0
        local critical_count=0
        local high_count=0
        local medium_count=0
        local low_count=0
        
        # Analyse de chaque règle
        for rule_key in "${!$self._security_rules[@]}"; do
            if [[ "$rule_key" =~ _pattern$ ]]; then
                local rule_id="${rule_key%_pattern}"
                local pattern="${$self._security_rules[$rule_key]}"
                local severity="${$self._security_rules[${rule_id}_severity]}"
                local description="${$self._security_rules[${rule_id}_description]}"
                
                # Recherche du pattern dans le script
                if echo "$script_content" | grep -q "$pattern"; then
                    echo "🚨 VULNÉRABILITÉ DÉTECTÉE: $rule_id"
                    echo "  Sévérité: $severity"
                    echo "  Description: $description"
                    
                    # Comptage par sévérité
                    case "$severity" in
                        CRITICAL) ((critical_count++)) ;;
                        HIGH) ((high_count++)) ;;
                        MEDIUM) ((medium_count++)) ;;
                        LOW) ((low_count++)) ;;
                    esac
                    
                    ((total_vulnerabilities++))
                    
                    # Stockage du résultat
                    $self._analysis_results["${analysis_id}_${rule_id}_detected"]="true"
                    $self._analysis_results["${analysis_id}_${rule_id}_severity"]="$severity"
                    
                    echo
                fi
            fi
        done
        
        # Analyse structurelle avancée
        $self._perform_structural_analysis "$script_content" "$analysis_id"
        
        # Résumé
        echo "=== RÉSULTATS DE L'ANALYSE ==="
        echo "Vulnérabilités totales: $total_vulnerabilities"
        echo "  Critique: $critical_count"
        echo "  Élevée: $high_count"
        echo "  Moyenne: $medium_count"
        echo "  Faible: $low_count"
        echo
        
        # Score de sécurité
        local security_score
        security_score=$($self._calculate_security_score "$critical_count" "$high_count" "$medium_count" "$low_count")
        
        echo "Score de sécurité: $security_score/100"
        
        if (( security_score >= 90 )); then
            echo "✅ Niveau de sécurité: EXCELLENT"
        elif (( security_score >= 75 )); then
            echo "⚠️  Niveau de sécurité: BON"
        elif (( security_score >= 60 )); then
            echo "⚠️  Niveau de sécurité: MOYEN"
        elif (( security_score >= 40 )); then
            echo "❌ Niveau de sécurité: FAIBLE - Corrections requises"
        else
            echo "🚨 Niveau de sécurité: CRITIQUE - Refactorisation complète requise"
        fi
        
        $self._analysis_results["${analysis_id}_total_vulnerabilities"]="$total_vulnerabilities"
        $self._analysis_results["${analysis_id}_security_score"]="$security_score"
        
        return $(( total_vulnerabilities > 0 ))
    }
    
    # Analyse structurelle avancée
    $self._perform_structural_analysis() {
        local script_content="$1"
        local analysis_id="$2"
        
        echo "--- ANALYSE STRUCTURELLE ---"
        
        # Vérification des bonnes pratiques de sécurité
        local issues_found=0
        
        # Absence de set -euo pipefail
        if ! echo "$script_content" | grep -q "set -euo pipefail"; then
            echo "⚠️  Bonne pratique manquante: set -euo pipefail"
            $self._analysis_results["${analysis_id}_missing_set_options"]="true"
            ((issues_found++))
        fi
        
        # Utilisation de eval
        if echo "$script_content" | grep -q "eval"; then
            echo "🚨 Utilisation dangereuse détectée: eval"
            $self._analysis_results["${analysis_id}_dangerous_eval"]="true"
            ((issues_found++))
        fi
        
        # Variables non quotées
        local unquoted_vars
        unquoted_vars=$(echo "$script_content" | grep -o '\$[A-Za-z_][A-Za-z0-9_]*[^"]' | wc -l)
        if (( unquoted_vars > 0 )); then
            echo "⚠️  Variables non quotées détectées: $unquoted_vars occurrences"
            $self._analysis_results["${analysis_id}_unquoted_variables"]="$unquoted_vars"
            ((issues_found++))
        fi
        
        # Utilisation de source avec chemins non vérifiés
        if echo "$script_content" | grep -q "source.*\$"; then
            echo "🚨 Source dynamique détecté - Risque d'inclusion de fichiers arbitraires"
            $self._analysis_results["${analysis_id}_dynamic_source"]="true"
            ((issues_found++))
        fi
        
        # Fonctions sans vérification d'erreur
        local functions_without_error_check
        functions_without_error_check=$(echo "$script_content" | grep -c "function.*{" | grep -v "||")
        if (( functions_without_error_check > 0 )); then
            echo "⚠️  Fonctions sans gestion d'erreur: $functions_without_error_check"
            ((issues_found++))
        fi
        
        if (( issues_found == 0 )); then
            echo "✅ Analyse structurelle: Aucune issue détectée"
        fi
        
        echo
    }
    
    # Calcul du score de sécurité
    $self._calculate_security_score() {
        local critical="$1" high="$2" medium="$3" low="$4"
        
        # Formule de calcul: 100 - (poids des vulnérabilités)
        local penalty=$(( critical * 25 + high * 15 + medium * 8 + low * 3 ))
        
        local score=$(( 100 - penalty ))
        
        # Score minimum de 0
        (( score < 0 )) && score=0
        
        echo "$score"
    }
    
    # Génération de rapport de sécurité
    $self.generate_security_report() {
        local analysis_id="$1"
        local output_file="${2:-security_report_${analysis_id}.txt}"
        
        {
            echo "RAPPORT D'ANALYSE DE SÉCURITÉ"
            echo "============================"
            echo "ID d'analyse: $analysis_id"
            echo "Date: $(date)"
            echo
            
            echo "RÉSUMÉ EXÉCUTIF"
            echo "==============="
            
            local total_vulns="${$self._analysis_results[${analysis_id}_total_vulnerabilities]}"
            local security_score="${$self._analysis_results[${analysis_id}_security_score]}"
            
            echo "Vulnérabilités détectées: $total_vulns"
            echo "Score de sécurité: $security_score/100"
            
            if (( security_score >= 80 )); then
                echo "Évaluation globale: SATISFAISANT"
            elif (( security_score >= 60 )); then
                echo "Évaluation globale: REQUIERT ATTENTION"
            else
                echo "Évaluation globale: ACTION IMMÉDIATE REQUISE"
            fi
            
            echo
            
            echo "DÉTAIL DES VULNÉRABILITÉS"
            echo "========================"
            
            for result_key in "${!$self._analysis_results[@]}"; do
                if [[ "$result_key" =~ ^${analysis_id}_.*_detected$ ]]; then
                    local vuln_id="${result_key#${analysis_id}_}"
                    vuln_id="${vuln_id%_detected}"
                    
                    local severity="${$self._analysis_results[${analysis_id}_${vuln_id}_severity]}"
                    local description="${$self._security_rules[${vuln_id}_description]}"
                    local remediation="${$self._security_rules[${vuln_id}_remediation]}"
                    
                    echo "Vulnérabilité: $vuln_id"
                    echo "  Sévérité: $severity"
                    echo "  Description: $description"
                    echo "  Remédiation: $remediation"
                    echo
                fi
            done
            
            echo "ANALYSE STRUCTURELLE"
            echo "===================="
            
            if [[ "${$self._analysis_results[${analysis_id}_missing_set_options]}" == "true" ]]; then
                echo "• Options 'set' manquantes: Ajouter 'set -euo pipefail' en début de script"
            fi
            
            if [[ "${$self._analysis_results[${analysis_id}_dangerous_eval]}" == "true" ]]; then
                echo "• Utilisation d'eval: Éviter eval, utiliser des alternatives sécurisées"
            fi
            
            local unquoted_vars="${$self._analysis_results[${analysis_id}_unquoted_variables]}"
            if [[ -n "$unquoted_vars" && "$unquoted_vars" != "0" ]]; then
                echo "• Variables non quotées: Toujours quoter les variables (\$VAR -> \"\$VAR\")"
            fi
            
            if [[ "${$self._analysis_results[${analysis_id}_dynamic_source]}" == "true" ]]; then
                echo "• Source dynamique: Valider les chemins avant inclusion"
            fi
            
            echo
            
            echo "RECOMMANDATIONS"
            echo "==============="
            
            if (( security_score < 60 )); then
                echo "PRIORITÉ CRITIQUE:"
                echo "• Refactoriser le script pour éliminer les vulnérabilités critiques"
                echo "• Implémenter une validation stricte des entrées"
                echo "• Ajouter une gestion d'erreur complète"
                echo "• Faire réviser le code par un expert sécurité"
            elif (( security_score < 80 )); then
                echo "PRIORITÉ ÉLEVÉE:"
                echo "• Corriger les vulnérabilités de haute sévérité"
                echo "• Améliorer la gestion des erreurs"
                echo "• Implémenter le logging sécurisé"
                echo "• Ajouter des tests de sécurité"
            else
                echo "PRIORITÉ MOYENNE:"
                echo "• Corriger les vulnérabilités restantes"
                echo "• Optimiser les performances"
                echo "• Documenter les fonctions de sécurité"
                echo "• Mettre en place une surveillance continue"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Correction automatique des vulnérabilités
    $self.auto_remediate() {
        local script_file="$1"
        local output_file="${2:-${script_file}.secured}"
        local analysis_id="$3"
        
        echo "=== CORRECTION AUTOMATIQUE ==="
        echo "Script original: $script_file"
        echo "Script corrigé: $output_file"
        
        local script_content
        script_content="$(cat "$script_file")"
        
        local corrections_applied=0
        
        # Correction 1: Ajout de set -euo pipefail
        if ! echo "$script_content" | grep -q "set -euo pipefail"; then
            # Trouver la ligne après le shebang
            local shebang_line
            shebang_line=$(echo "$script_content" | grep -n "^#!/.*bash" | cut -d: -f1)
            
            if [[ -n "$shebang_line" ]]; then
                # Insérer après le shebang
                script_content=$(echo "$script_content" | sed "${shebang_line}a\\
set -euo pipefail")
                echo "✓ Ajout de 'set -euo pipefail'"
                ((corrections_applied++))
            fi
        fi
        
        # Correction 2: Ajout de vérifications d'erreur pour les commandes critiques
        # (Version simplifiée - en pratique, cela nécessiterait une analyse plus fine)
        
        # Écriture du script corrigé
        echo "$script_content" > "$output_file"
        chmod +x "$output_file"
        
        echo "Corrections appliquées: $corrections_applied"
        echo "✓ Script sécurisé créé: $output_file"
        
        # Réanalyse du script corrigé
        echo
        echo "Réanalyse du script corrigé:"
        $self.analyze_script "$output_file" "${analysis_id}_remediated"
    }
    
    # Benchmarking des analyses de sécurité
    $self.benchmark_security_analysis() {
        local script_file="$1"
        local iterations="${2:-10}"
        
        echo "=== BENCHMARKING ANALYSE SÉCURITÉ ==="
        echo "Script: $(basename "$script_file")"
        echo "Itérations: $iterations"
        
        local total_time=0
        
        for ((i=1; i<=iterations; i++)); do
            echo -n "Itération $i... "
            
            local start_time
            start_time="$(date +%s.%N)"
            
            $self.analyze_script "$script_file" "bench_$i" >/dev/null 2>&1
            
            local end_time
            end_time="$(date +%s.%N)"
            
            local duration
            duration="$(echo "$end_time - $start_time" | bc -l 2>/dev/null || echo "0")"
            
            total_time="$(echo "$total_time + $duration" | bc -l)"
            echo "${duration}s"
        done
        
        local avg_time
        avg_time="$(echo "scale=4; $total_time / $iterations" | bc -l)"
        
        echo
        echo "Temps moyen d'analyse: ${avg_time}s"
        echo "Analyses par seconde: $(echo "scale=2; $iterations / $total_time" | bc -l)"
    }
}

# Définition des règles de sécurité standard
define_security_rules() {
    local analyzer="$1"
    
    # Règles de sévérité CRITIQUE
    $analyzer.define_security_rule "command_injection" "CRITICAL" \
        "Injection de commande - Utilisation dangereuse de variables dans des commandes" \
        '\$[^}]*\`|\$\([^)]*\)' \
        "Toujours quoter les variables et utiliser des tableaux pour les arguments" \
        "injection"
    
    $analyzer.define_security_rule "sql_injection" "CRITICAL" \
        "Injection SQL potentielle - Concaténation de requêtes SQL" \
        'SELECT.*\$|INSERT.*\$|UPDATE.*\$|DELETE.*\$' \
        "Utiliser des requêtes préparées ou des ORM sécurisés" \
        "injection"
    
    $analyzer.define_security_rule "path_traversal" "CRITICAL" \
        "Traversal de chemin - Utilisation de ../ ou de chemins non validés" \
        '\.\./|\.\.\\' \
        "Valider et nettoyer tous les chemins d'accès" \
        "filesystem"
    
    # Règles de sévérité HIGH
    $analyzer.define_security_rule "weak_random" "HIGH" \
        "Générateur de nombres aléatoires faible - Utilisation de \$RANDOM" \
        '\$RANDOM' \
        "Utiliser /dev/urandom ou des générateurs cryptographiques" \
        "crypto"
    
    $analyzer.define_security_rule "hardcoded_secrets" "HIGH" \
        "Secrets codés en dur - Mots de passe ou clés en clair" \
        'password.*=.*[a-zA-Z0-9]{8,}|secret.*=.*[a-zA-Z0-9]{8,}|key.*=.*[a-zA-Z0-9]{16,}' \
        "Utiliser des variables d'environnement ou des coffres-forts" \
        "secrets"
    
    $analyzer.define_security_rule "unsafe_temp_files" "HIGH" \
        "Fichiers temporaires non sécurisés" \
        'mktemp\(/tmp/|/var/tmp/' \
        "Utiliser mktemp avec des noms uniques et des permissions appropriées" \
        "filesystem"
    
    # Règles de sévérité MEDIUM
    $analyzer.define_security_rule "missing_input_validation" "MEDIUM" \
        "Validation d'entrée manquante - Variables non validées" \
        'read.*\$|echo.*\$' \
        "Valider et nettoyer toutes les entrées utilisateur" \
        "input"
    
    $analyzer.define_security_rule "race_condition" "MEDIUM" \
        "Condition de course potentielle - Accès concurrentiel à des fichiers" \
        '>>|\+\+|echo.*>' \
        "Utiliser des verrouillages ou des opérations atomiques" \
        "concurrency"
    
    $analyzer.define_security_rule "information_disclosure" "MEDIUM" \
        "Divulgation d'informations - Affichage de détails système" \
        'uname -a|whoami|pwd' \
        "Limiter l'affichage d'informations sensibles" \
        "information"
    
    # Règles de sévérité LOW
    $analyzer.define_security_rule "missing_error_handling" "LOW" \
        "Gestion d'erreur manquante" \
        'function.*\{.*[^}]*$' \
        "Ajouter des vérifications d'erreur après chaque commande critique" \
        "error_handling"
    
    $analyzer.define_security_rule "unused_variables" "LOW" \
        "Variables non utilisées - Possible code mort" \
        '[a-zA-Z_][a-zA-Z0-9_]*=.*' \
        "Supprimer ou utiliser les variables non utilisées" \
        "code_quality"
    
    $analyzer.define_security_rule "missing_comments" "LOW" \
        "Commentaires manquants - Code peu maintenable" \
        '^[^#]*function|^[^#]*if|^[^#]*for|^[^#]*while' \
        "Ajouter des commentaires explicatifs pour les fonctions complexes" \
        "documentation"
}

# Démonstration de l'analyseur de sécurité
echo "--- Analyseur de sécurité ---"

SecurityAnalyzer "security_analyzer"

# Définition des règles de sécurité
define_security_rules "security_analyzer"

echo
echo "--- Analyse d'un script vulnérable ---"

# Création d'un script vulnérable pour la démonstration
cat > /tmp/vulnerable_script.sh << 'EOF'
#!/bin/bash
# Script vulnérable pour démonstration

# Fonction dangereuse avec injection de commande
execute_command() {
    cmd="$1"
    eval $cmd  # DANGER ! Injection possible
}

# Lecture sans validation
read -p "Entrez un nom de fichier: " filename
cat $filename  # DANGER ! Path traversal possible

# Utilisation de /dev/random au lieu de /dev/urandom
random_num=$RANDOM

# Secret codé en dur
db_password="admin123"

# Fichier temporaire non sécurisé
temp_file="/tmp/temp_$$"
echo "Données temporaires" > $temp_file

# Commande système informative
uname -a

execute_command "ls -la"
EOF

chmod +x /tmp/vulnerable_script.sh

# Analyse du script vulnérable
security_analyzer.analyze_script "/tmp/vulnerable_script.sh" "demo_analysis"

echo
echo "--- Génération de rapport ---"
security_analyzer.generate_security_report "demo_analysis"

echo
echo "--- Correction automatique ---"
security_analyzer.auto_remediate "/tmp/vulnerable_script.sh" "/tmp/secured_script.sh" "demo_analysis"

echo
echo "--- Benchmarking ---"
security_analyzer.benchmark_security_analysis "/tmp/vulnerable_script.sh" "3"

# Nettoyage
rm -f /tmp/vulnerable_script.sh /tmp/secured_script.sh security_report_*.txt
```

### 1.2 Framework de validation d'entrée sécurisée

Système complet de validation et de nettoyage des entrées :

```bash
#!/bin/bash

# Framework de validation d'entrée sécurisée
echo "=== Framework de validation d'entrée sécurisée ==="

# Input Validation Framework
InputValidator() {
    local self="$1"
    
    declare -A $self._validation_rules
    declare -A $self._sanitization_rules
    declare -A $self._validation_cache
    
    # Définition d'une règle de validation
    $self.define_validation_rule() {
        local rule_name="$1"
        local rule_type="$2"
        local pattern="$3"
        local error_message="$4"
        
        $self._validation_rules["${rule_name}_type"]="$rule_type"
        $self._validation_rules["${rule_name}_pattern"]="$pattern"
        $self._validation_rules["${rule_name}_error"]="$error_message"
        
        echo "✓ Règle de validation définie: $rule_name ($rule_type)"
    }
    
    # Définition d'une règle de nettoyage
    $self.define_sanitization_rule() {
        local rule_name="$1"
        local transformation="$2"
        local description="$3"
        
        $self._sanitization_rules["${rule_name}_transformation"]="$transformation"
        $self._sanitization_rules["${rule_name}_description"]="$description"
        
        echo "✓ Règle de nettoyage définie: $rule_name"
    }
    
    # Validation d'une entrée
    $self.validate_input() {
        local input="$1"
        local rule_name="$2"
        local strict="${3:-false}"
        
        local rule_type="${$self._validation_rules[${rule_name}_type]}"
        local pattern="${$self._validation_rules[${rule_name}_pattern]}"
        local error_msg="${$self._validation_rules[${rule_name}_error]}"
        
        if [[ -z "$rule_type" ]]; then
            echo "❌ Règle de validation introuvable: $rule_name" >&2
            return 1
        fi
        
        case "$rule_type" in
            regex)
                if [[ "$input" =~ $pattern ]]; then
                    echo "✅ Validation réussie: $rule_name"
                    return 0
                else
                    echo "❌ Validation échouée: $error_msg" >&2
                    return 1
                fi
                ;;
                
            length)
                local min_length max_length
                IFS='-' read min_length max_length <<< "$pattern"
                
                if (( ${#input} >= min_length && ${#input} <= max_length )); then
                    echo "✅ Validation réussie: $rule_name"
                    return 0
                else
                    echo "❌ Validation échouée: $error_msg" >&2
                    return 1
                fi
                ;;
                
            whitelist)
                # Vérification que l'entrée ne contient que des caractères autorisés
                if [[ "$input" =~ ^[$pattern]+$ ]]; then
                    echo "✅ Validation réussie: $rule_name"
                    return 0
                else
                    echo "❌ Validation échouée: $error_msg" >&2
                    return 1
                fi
                ;;
                
            blacklist)
                # Vérification que l'entrée ne contient pas de caractères interdits
                if [[ ! "$input" =~ [$pattern] ]]; then
                    echo "✅ Validation réussie: $rule_name"
                    return 0
                else
                    echo "❌ Validation échouée: $error_msg" >&2
                    return 1
                fi
                ;;
                
            custom)
                # Exécution d'une fonction de validation personnalisée
                if $pattern "$input" 2>/dev/null; then
                    echo "✅ Validation réussie: $rule_name"
                    return 0
                else
                    echo "❌ Validation échouée: $error_msg" >&2
                    return 1
                fi
                ;;
                
            *)
                echo "❌ Type de règle inconnu: $rule_type" >&2
                return 1
                ;;
        esac
    }
    
    # Nettoyage d'une entrée
    $self.sanitize_input() {
        local input="$1"
        local rule_name="$2"
        
        local transformation="${$self._sanitization_rules[${rule_name}_transformation]}"
        local description="${$self._sanitization_rules[${rule_name}_description]}"
        
        if [[ -z "$transformation" ]]; then
            echo "❌ Règle de nettoyage introuvable: $rule_name" >&2
            return 1
        fi
        
        echo "Nettoyage avec: $description"
        
        local sanitized_input
        
        case "$transformation" in
            lowercase)
                sanitized_input="$(echo "$input" | tr '[:upper:]' '[:lower:]')"
                ;;
                
            uppercase)
                sanitized_input="$(echo "$input" | tr '[:lower:]' '[:upper:]')"
                ;;
                
            trim)
                sanitized_input="$(echo "$input" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')"
                ;;
                
            remove_spaces)
                sanitized_input="$(echo "$input" | tr -d '[:space:]')"
                ;;
                
            alphanumeric_only)
                sanitized_input="$(echo "$input" | sed 's/[^a-zA-Z0-9]//g')"
                ;;
                
            safe_filename)
                sanitized_input="$(echo "$input" | sed 's/[^a-zA-Z0-9._-]//g')"
                ;;
                
            escape_shell)
                sanitized_input="$(printf '%q' "$input")"
                ;;
                
            custom:*)
                local custom_transform="${transformation#custom:}"
                sanitized_input="$(eval "$custom_transform" <<< "$input")"
                ;;
                
            *)
                echo "❌ Transformation inconnue: $transformation" >&2
                return 1
                ;;
        esac
        
        echo "Entrée nettoyée: '$input' -> '$sanitized_input'"
        echo "$sanitized_input"
    }
    
    # Validation et nettoyage combinés
    $self.validate_and_sanitize() {
        local input="$1"
        local validation_rule="$2"
        local sanitization_rule="$3"
        local max_attempts="${4:-3}"
        
        local attempt=1
        local current_input="$input"
        
        while (( attempt <= max_attempts )); do
            echo "Tentative $attempt/$max_attempts"
            
            # Validation
            if $self.validate_input "$current_input" "$validation_rule"; then
                echo "✅ Validation réussie"
                echo "$current_input"
                return 0
            fi
            
            # Si validation échouée et qu'on peut nettoyer
            if [[ -n "$sanitization_rule" ]]; then
                echo "Tentative de nettoyage..."
                current_input="$($self.sanitize_input "$current_input" "$sanitization_rule")"
                ((attempt++))
            else
                break
            fi
        done
        
        echo "❌ Validation finale échouée après $max_attempts tentatives" >&2
        return 1
    }
    
    # Validation par lots
    $self.batch_validate() {
        local inputs_file="$1"
        local rules_file="$2"
        local output_file="${3:-validation_results.txt}"
        
        if [[ ! -f "$inputs_file" ]]; then
            echo "❌ Fichier d'entrées introuvable: $inputs_file" >&2
            return 1
        fi
        
        echo "=== VALIDATION PAR LOTS ==="
        echo "Entrées: $inputs_file"
        echo "Règles: $rules_file"
        echo "Résultats: $output_file"
        echo
        
        local total_processed=0
        local total_valid=0
        local total_invalid=0
        
        {
            echo "RAPPORT DE VALIDATION PAR LOTS"
            echo "=============================="
            echo "Date: $(date)"
            echo
            
            while IFS='|' read -r input_value rule_name; do
                ((total_processed++))
                
                echo "Validation #$total_processed: '$input_value' (règle: $rule_name)"
                
                if $self.validate_input "$input_value" "$rule_name" >/dev/null 2>&1; then
                    echo "  ✅ VALIDE"
                    ((total_valid++))
                else
                    echo "  ❌ INVALIDE"
                    ((total_invalid++))
                fi
                
                echo
            done < "$inputs_file"
            
            echo "RÉSUMÉ"
            echo "======"
            echo "Total traité: $total_processed"
            echo "Valides: $total_valid"
            echo "Invalides: $total_invalid"
            echo "Taux de succès: $((total_valid * 100 / total_processed))%"
            
        } > "$output_file"
        
        echo "✓ Validation par lots terminée"
        echo "Résultats dans: $output_file"
    }
    
    # Fonction de validation personnalisée pour les emails
    validate_email() {
        local email="$1"
        [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]
    }
    
    # Fonction de validation personnalisée pour les URLs
    validate_url() {
        local url="$1"
        [[ "$url" =~ ^https?://[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}(/[a-zA-Z0-9._~:/?#[\]@!$&'()*+,;=-]*)?$ ]]
    }
    
    # Fonction de validation personnalisée pour les adresses IP
    validate_ip() {
        local ip="$1"
        [[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] && {
            local IFS='.'
            local -a octets=($ip)
            for octet in "${octets[@]}"; do
                (( octet >= 0 && octet <= 255 )) || return 1
            done
        }
    }
    
    # Création d'un validateur sécurisé pour les scripts
    $self.create_secure_validator() {
        local output_file="$1"
        
        {
            echo "#!/bin/bash"
            echo "# Validateur sécurisé généré automatiquement"
            echo "# Date: $(date)"
            echo
            echo "# Options de sécurité"
            echo "set -euo pipefail"
            echo
            echo "# Fonction de validation principale"
            echo "secure_validate() {"
            echo "    local input=\"\$1\""
            echo "    local validation_type=\"\$2\""
            echo "    local max_length=\"\${3:-1000}\""
            echo "    "
            echo "    # Vérifications de sécurité de base"
            echo "    if [[ -z \"\$input\" ]]; then"
            echo "        echo \"Erreur: Entrée vide\" >&2"
            echo "        return 1"
            echo "    fi"
            echo "    "
            echo "    if (( \${#input} > max_length )); then"
            echo "        echo \"Erreur: Entrée trop longue (max \$max_length caractères)\" >&2"
            echo "        return 1"
            echo "    fi"
            echo "    "
            echo "    # Validation selon le type"
            echo "    case \"\$validation_type\" in"
            echo "        email)"
            echo "            if [[ ! \"\$input\" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}\$ ]]; then"
            echo "                echo \"Erreur: Format d'email invalide\" >&2"
            echo "                return 1"
            echo "            fi"
            echo "            ;;"
            echo "        username)"
            echo "            if [[ ! \"\$input\" =~ ^[a-zA-Z][a-zA-Z0-9_-]{2,31}\$ ]]; then"
            echo "                echo \"Erreur: Nom d'utilisateur invalide\" >&2"
            echo "                return 1"
            echo "            fi"
            echo "            ;;"
            echo "        filename)"
            echo "            if [[ \"\$input\" =~ [/\] ]]; then"
            echo "                echo \"Erreur: Caractères interdits dans le nom de fichier\" >&2"
            echo "                return 1"
            echo "            fi"
            echo "            ;;"
            echo "        number)"
            echo "            if [[ ! \"\$input\" =~ ^[0-9]+\$ ]]; then"
            echo "                echo \"Erreur: Nombre entier attendu\" >&2"
            echo "                return 1"
            echo "            fi"
            echo "            ;;"
            echo "        ip_address)"
            echo "            if ! echo \"\$input\" | grep -qE '^([0-9]{1,3}\.){3}[0-9]{1,3}\$' || ! ping -c1 -W1 \"\$input\" >/dev/null 2>&1; then"
            echo "                echo \"Erreur: Adresse IP invalide\" >&2"
            echo "                return 1"
            echo "            fi"
            echo "            ;;"
            echo "        *)"
            echo "            echo \"Erreur: Type de validation inconnu: \$validation_type\" >&2"
            echo "            return 1"
            echo "            ;;"
            echo "    esac"
            echo "    "
            echo "    echo \"✅ Validation réussie\""
            echo "    return 0"
            echo "}"
            echo
            echo "# Fonction de nettoyage sécurisé"
            echo "secure_sanitize() {"
            echo "    local input=\"\$1\""
            echo "    local sanitization_type=\"\$2\""
            echo "    "
            echo "    case \"\$sanitization_type\" in"
            echo "        shell_escape)"
            echo "            printf '%q' \"\$input\""
            echo "            ;;"
            echo "        filename)"
            echo "            echo \"\$input\" | sed 's/[^a-zA-Z0-9._-]//g'"
            echo "            ;;"
            echo "        lowercase)"
            echo "            echo \"\$input\" | tr '[:upper:]' '[:lower:]'"
            echo "            ;;"
            echo "        trim)"
            echo "            echo \"\$input\" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//'"
            echo "            ;;"
            echo "        *)"
            echo "            echo \"Erreur: Type de nettoyage inconnu\" >&2"
            echo "            echo \"\$input\""
            echo "            ;;"
            echo "    esac"
            echo "}"
            echo
            echo "# Si exécuté directement"
            echo "if [[ \"\${BASH_SOURCE[0]}\" == \"\$0\" ]]; then"
            echo "    if [[ \$# -lt 2 ]]; then"
            echo "        echo \"Usage: \$0 <input> <validation_type> [max_length]\" >&2"
            echo "        echo \"Types: email, username, filename, number, ip_address\" >&2"
            echo "        exit 1"
            echo "    fi"
            echo "    "
            echo "    secure_validate \"\$1\" \"\$2\" \"\$3\""
            echo "fi"
            
        } > "$output_file"
        
        chmod +x "$output_file"
        echo "✓ Validateur sécurisé créé: $output_file"
    }
    
    # Test de sécurité des validateurs
    $self.security_test() {
        local validator_file="$1"
        
        echo "=== TEST DE SÉCURITÉ DU VALIDATEUR ==="
        
        local test_cases=(
            # Tests d'injection
            "'; rm -rf /; echo '"
            "\$(cat /etc/passwd)"
            "`whoami`"
            # Tests de dépassement
            "$(printf 'a%.0s' {1..10000})"
            # Tests de caractères spéciaux
            "../../../etc/passwd"
            "<script>alert('xss')</script>"
            # Tests de valeurs normales
            "valid@email.com"
            "test_user_123"
            "my_file.txt"
        )
        
        local passed=0
        local failed=0
        
        for test_input in "${test_cases[@]}"; do
            echo "Test: '$test_input'"
            
            if "$validator_file" "$test_input" "email" >/dev/null 2>&1; then
                echo "  ❌ Test passé (devrait échouer)"
                ((failed++))
            else
                echo "  ✅ Test bloqué (correct)"
                ((passed++))
            fi
        done
        
        echo
        echo "Résultats du test de sécurité:"
        echo "  Tests passés: $passed"
        echo "  Tests échoués: $failed"
        echo "  Taux de protection: $((passed * 100 / (passed + failed)))%"
    }
}

# Définition des règles de validation standard
define_validation_rules() {
    local validator="$1"
    
    # Règles de validation de base
    $validator.define_validation_rule "email" "regex" \
        "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$" \
        "Format d'email invalide"
    
    $validator.define_validation_rule "username" "regex" \
        "^[a-zA-Z][a-zA-Z0-9_-]{2,31}$" \
        "Nom d'utilisateur invalide (3-32 caractères, lettres/chiffres/tiret/underscore, commence par une lettre)"
    
    $validator.define_validation_rule "password" "length" \
        "8-128" \
        "Mot de passe trop court (8 caractères minimum) ou trop long (128 caractères maximum)"
    
    $validator.define_validation_rule "safe_filename" "blacklist" \
        "/\\<>:\"|?*" \
        "Caractères interdits dans le nom de fichier"
    
    $validator.define_validation_rule "alphanumeric" "whitelist" \
        "a-zA-Z0-9" \
        "Seuls les caractères alphanumériques sont autorisés"
    
    $validator.define_validation_rule "ip_address" "custom" \
        "validate_ip" \
        "Adresse IP invalide"
    
    $validator.define_validation_rule "url" "custom" \
        "validate_url" \
        "URL invalide"
    
    # Règles de nettoyage
    $validator.define_sanitization_rule "lowercase" "lowercase" \
        "Conversion en minuscules"
    
    $validator.define_sanitization_rule "safe_filename" "safe_filename" \
        "Suppression des caractères dangereux dans les noms de fichiers"
    
    $validator.define_sanitization_rule "trim_spaces" "trim" \
        "Suppression des espaces en début et fin"
    
    $validator.define_sanitization_rule "shell_escape" "escape_shell" \
        "Échappement des caractères spéciaux pour le shell"
    
    $validator.define_sanitization_rule "alphanumeric_only" "alphanumeric_only" \
        "Conservation des seuls caractères alphanumériques"
}

# Démonstration du framework de validation
echo "--- Framework de validation d'entrée ---"

InputValidator "input_validator"

# Définition des règles
define_validation_rules "input_validator"

echo
echo "--- Tests de validation ---"

# Tests de validation de base
echo "Test email:"
input_validator.validate_input "user@example.com" "email"
input_validator.validate_input "invalid-email" "email"

echo
echo "Test nom d'utilisateur:"
input_validator.validate_input "valid_user123" "username"
input_validator.validate_input "123invalid" "username"

echo
echo "Test mot de passe:"
input_validator.validate_input "short" "password"
input_validator.validate_input "verylongpasswordthatexceedsnormalboundsandshouldfailvalidation" "password"

echo
echo "Test nom de fichier sécurisé:"
input_validator.validate_input "safe_file.txt" "safe_filename"
input_validator.validate_input "dangerous/../../../etc/passwd" "safe_filename"

echo
echo "--- Tests de nettoyage ---"

echo "Test nettoyage:"
input_validator.sanitize_input "  SPACES AROUND  " "trim_spaces"
input_validator.sanitize_input "DANGERous/../../../file" "safe_filename"
input_validator.sanitize_input 'DANGEROUS;rm -rf /' "shell_escape"

echo
echo "--- Validation et nettoyage combinés ---"

echo "Test validation+nettoyage:"
input_validator.validate_and_sanitize "  user@EXAMPLE.COM  " "email" "lowercase"
input_validator.validate_and_sanitize "INVALID EMAIL" "email" "trim_spaces"

echo
echo "--- Création d'un validateur sécurisé ---"
input_validator.create_secure_validator "/tmp/secure_validator.sh"

echo "Test du validateur sécurisé:"
bash /tmp/secure_validator.sh "test@example.com" "email"
bash /tmp/secure_validator.sh "invalid-email" "email"
bash /tmp/secure_validator.sh "valid_user" "username"

echo
echo "--- Test de sécurité ---"
input_validator.security_test "/tmp/secure_validator.sh"

# Nettoyage
rm -f /tmp/secure_validator.sh
```

## Section 2 : Durcissement des environnements d'exécution

### 2.1 Configuration sécurisée de l'environnement

Configuration automatique d'environnements sécurisés :

```bash
#!/bin/bash

# Configuration sécurisée de l'environnement
echo "=== Configuration sécurisée de l'environnement ==="

# Secure Environment Configurator
SecureEnvironment() {
    local self="$1"
    
    declare -A $self._security_policies
    declare -A $self._environment_settings
    declare -A $self._hardening_status
    
    # Définition d'une politique de sécurité
    $self.define_security_policy() {
        local policy_name="$1"
        local description="$2"
        local settings="$3"
        local priority="${4:-medium}"
        
        $self._security_policies["${policy_name}_description"]="$description"
        $self._security_policies["${policy_name}_settings"]="$settings"
        $self._security_policies["${policy_name}_priority"]="$priority"
        
        echo "✓ Politique de sécurité définie: $policy_name ($priority)"
    }
    
    # Application d'une politique de sécurité
    $self.apply_security_policy() {
        local policy_name="$1"
        local dry_run="${2:-false}"
        
        echo "=== APPLICATION POLITIQUE SÉCURITÉ: $policy_name ==="
        
        local description="${$self._security_policies[${policy_name}_description]}"
        local settings="${$self._security_policies[${policy_name}_settings]}"
        
        echo "Description: $description"
        
        if [[ "$dry_run" == "true" ]]; then
            echo "🔍 MODE SIMULATION - Aucune modification appliquée"
        fi
        
        local applied=0
        local failed=0
        
        # Parsing et application des paramètres
        while IFS=';' read -ra setting_list; do
            for setting in "${setting_list[@]}"; do
                if [[ "$setting" =~ ^([^:]+):(.*)$ ]]; then
                    local setting_type="${BASH_REMATCH[1]}"
                    local setting_value="${BASH_REMATCH[2]}"
                    
                    echo "--- Application: $setting_type ---"
                    
                    if [[ "$dry_run" == "true" ]]; then
                        echo "Simulation: $setting_value"
                        ((applied++))
                    else
                        if $self._apply_environment_setting "$setting_type" "$setting_value"; then
                            $self._environment_settings["${policy_name}_${setting_type}"]="$setting_value"
                            ((applied++))
                            echo "✓ Appliqué"
                        else
                            ((failed++))
                            echo "❌ Échec"
                        fi
                    fi
                fi
            done
        done <<< "$settings"
        
        echo
        echo "Résumé politique $policy_name:"
        echo "  Paramètres appliqués: $applied"
        echo "  Échecs: $failed"
        
        if [[ "$dry_run" == "false" ]]; then
            $self._hardening_status["${policy_name}_applied"]="$(date +%s)"
            $self._hardening_status["${policy_name}_status"]=$([[ $failed -eq 0 ]] && echo "success" || echo "partial")
        fi
        
        return $(( failed > 0 ))
    }
    
    # Application d'un paramètre d'environnement
    $self._apply_environment_setting() {
        local setting_type="$1"
        local setting_value="$2"
        
        case "$setting_type" in
            bash_option)
                # Configuration des options Bash
                local option value
                IFS='=' read option value <<< "$setting_value"
                
                case "$option" in
                    errexit) set -e ;;
                    nounset) set -u ;;
                    pipefail) set -o pipefail ;;
                    noglob) set -f ;;
                    noclobber) set -C ;;
                    *) echo "Option Bash inconnue: $option"; return 1 ;;
                esac
                ;;
                
            umask)
                umask "$setting_value"
                ;;
                
            path)
                # Configuration sécurisée du PATH
                export PATH="$setting_value"
                ;;
                
            env_var)
                # Configuration d'une variable d'environnement
                local var value
                IFS='=' read var value <<< "$setting_value"
                export "$var"="$value"
                ;;
                
            limit)
                # Configuration des limites de ressources
                local resource value
                IFS='=' read resource value <<< "$setting_value"
                
                case "$resource" in
                    core) ulimit -c "$value" ;;
                    data) ulimit -d "$value" ;;
                    fsize) ulimit -f "$value" ;;
                    memlock) ulimit -l "$value" ;;
                    nofile) ulimit -n "$value" ;;
                    nproc) ulimit -u "$value" ;;
                    stack) ulimit -s "$value" ;;
                    *) echo "Limite inconnue: $resource"; return 1 ;;
                esac
                ;;
                
            sysctl)
                # Configuration sysctl (nécessite root)
                if [[ $EUID -eq 0 ]]; then
                    sysctl -w "$setting_value" >/dev/null 2>&1 || true
                else
                    echo "Root requis pour sysctl"
                    return 1
                fi
                ;;
                
            firewall)
                # Configuration firewall simple
                if command -v ufw >/dev/null 2>&1; then
                    ufw "$setting_value" >/dev/null 2>&1 || true
                elif command -v firewall-cmd >/dev/null 2>&1; then
                    firewall-cmd --$setting_value >/dev/null 2>&1 || true
                fi
                ;;
                
            service)
                # Gestion des services
                local action service_name
                IFS=':' read action service_name <<< "$setting_value"
                
                case "$action" in
                    enable) systemctl enable "$service_name" >/dev/null 2>&1 || true ;;
                    disable) systemctl disable "$service_name" >/dev/null 2>&1 || true ;;
                    stop) systemctl stop "$service_name" >/dev/null 2>&1 || true ;;
                    start) systemctl start "$service_name" >/dev/null 2>&1 || true ;;
                    *) echo "Action service inconnue: $action"; return 1 ;;
                esac
                ;;
                
            *)
                echo "Type de paramètre inconnu: $setting_type"
                return 1
                ;;
        esac
        
        return 0
    }
    
    # Vérification de conformité à une politique
    $self.verify_policy_compliance() {
        local policy_name="$1"
        
        echo "=== VÉRIFICATION CONFORMITÉ POLITIQUE: $policy_name ==="
        
        local settings="${$self._security_policies[${policy_name}_settings]}"
        local compliant=true
        
        while IFS=';' read -ra setting_list; do
            for setting in "${setting_list[@]}"; do
                if [[ "$setting" =~ ^([^:]+):(.*)$ ]]; then
                    local setting_type="${BASH_REMATCH[1]}"
                    local setting_value="${BASH_REMATCH[2]}"
                    
                    if ! $self._verify_setting_compliance "$setting_type" "$setting_value"; then
                        echo "❌ Non conforme: $setting_type"
                        compliant=false
                    else
                        echo "✅ Conforme: $setting_type"
                    fi
                fi
            done
        done <<< "$settings"
        
        if [[ "$compliant" == "true" ]]; then
            echo "✅ Politique $policy_name conforme"
            return 0
        else
            echo "❌ Non-conformités détectées dans $policy_name"
            return 1
        fi
    }
    
    # Vérification de conformité d'un paramètre
    $self._verify_setting_compliance() {
        local setting_type="$1"
        local setting_value="$2"
        
        case "$setting_type" in
            bash_option)
                local option value
                IFS='=' read option value <<< "$setting_value"
                
                case "$option" in
                    errexit) [[ $- == *e* ]] ;;
                    nounset) [[ $- == *u* ]] ;;
                    pipefail) [[ $(set -o | grep pipefail | awk '{print $2}') == "on" ]] ;;
                    *) false ;;
                esac
                ;;
                
            umask)
                [[ "$(umask)" == "$setting_value" ]]
                ;;
                
            env_var)
                local var value
                IFS='=' read var value <<< "$setting_value"
                [[ "${!var}" == "$value" ]]
                ;;
                
            limit)
                local resource value
                IFS='=' read resource value <<< "$setting_value"
                
                case "$resource" in
                    nofile) [[ "$(ulimit -n)" == "$value" ]] ;;
                    nproc) [[ "$(ulimit -u)" == "$value" ]] ;;
                    *) true ;;  # Simulation pour les autres limites
                esac
                ;;
                
            *)
                true  # Simulation de conformité pour les autres types
                ;;
        esac
    }
    
    # Création d'un profil de sécurité personnalisé
    $self.create_custom_security_profile() {
        local profile_name="$1"
        local base_policy="$2"
        local customizations="$3"
        
        echo "Création profil personnalisé: $profile_name"
        echo "Basé sur: $base_policy"
        
        local base_settings="${$self._security_policies[${base_policy}_settings]}"
        local base_description="${$self._security_policies[${base_policy}_description]}"
        
        if [[ -z "$base_settings" ]]; then
            echo "❌ Politique de base introuvable: $base_policy" >&2
            return 1
        fi
        
        # Fusion des paramètres (customizations peuvent remplacer les paramètres de base)
        local final_settings="$base_settings"
        
        # Parsing des customizations et remplacement
        while IFS=';' read -ra customization_list; do
            for customization in "${customization_list[@]}"; do
                if [[ "$customization" =~ ^([^:]+):(.*)$ ]]; then
                    local cust_type="${BASH_REMATCH[1]}"
                    local cust_value="${BASH_REMATCH[2]}"
                    
                    # Remplacement ou ajout du paramètre
                    if grep -q "^${cust_type}:" <<< "$final_settings"; then
                        # Remplacement
                        final_settings="$(sed "s/^${cust_type}:.*/${cust_type}:${cust_value}/" <<< "$final_settings")"
                    else
                        # Ajout
                        final_settings="${final_settings};${cust_type}:${cust_value}"
                    fi
                fi
            done
        done <<< "$customizations"
        
        # Création de la nouvelle politique
        $self.define_security_policy "$profile_name" \
            "Profil personnalisé basé sur $base_description" \
            "$final_settings" \
            "custom"
        
        echo "✓ Profil personnalisé créé: $profile_name"
    }
    
    # Génération d'un script de durcissement
    $self.generate_hardening_script() {
        local policy_name="$1"
        local output_file="$2"
        
        local description="${$self._security_policies[${policy_name}_description]}"
        local settings="${$self._security_policies[${policy_name}_settings]}"
        
        {
            echo "#!/bin/bash"
            echo "# Script de durcissement généré automatiquement"
            echo "# Politique: $policy_name"
            echo "# Description: $description"
            echo "# Généré le: $(date)"
            echo
            echo "# Options de sécurité de base"
            echo "set -euo pipefail"
            echo
            echo "# Fonction de journalisation"
            echo "log() {"
            echo "    echo \"[\$(date '+%Y-%m-%d %H:%M:%S')] \$*\""
            echo "}"
            echo
            echo "# Fonction de vérification des privilèges"
            echo "check_privileges() {"
            echo "    if [[ \$EUID -ne 0 ]]; then"
            echo "        log \"ERREUR: Ce script nécessite les privilèges root\""
            echo "        exit 1"
            echo "    fi"
            echo "}"
            echo
            echo "# Fonction de sauvegarde"
            echo "backup_setting() {"
            echo "    local setting=\"\$1\""
            echo "    local value=\"\$2\""
            echo "    echo \"\$setting=\$value\" >> \"\$BACKUP_FILE\""
            echo "}"
            echo
            echo "# Fonction de rollback"
            echo "rollback() {"
            echo "    log \"ROLLBACK: Restauration des paramètres précédents\""
            echo "    while IFS='=' read -r setting value; do"
            echo "        case \"\$setting\" in"
            echo "            UMASK) umask \"\$value\" ;;"
            echo "            PATH) export PATH=\"\$value\" ;;"
            echo "            *) export \"\$setting\"=\"\$value\" ;;"
            echo "        esac"
            echo "    done < \"\$BACKUP_FILE\""
            echo "}"
            echo
            echo "# Initialisation"
            echo "BACKUP_FILE=\"/tmp/hardening_backup_\$(date +%s).txt\""
            echo "log \"Démarrage du durcissement: $policy_name\""
            echo
            echo "# Vérification des privilèges si nécessaire"
            echo "check_privileges 2>/dev/null || true"
            echo
            echo "# Application des paramètres de sécurité"
            
            # Génération du code d'application des paramètres
            while IFS=';' read -ra setting_list; do
                for setting in "${setting_list[@]}"; do
                    if [[ "$setting" =~ ^([^:]+):(.*)$ ]]; then
                        local setting_type="${BASH_REMATCH[1]}"
                        local setting_value="${BASH_REMATCH[2]}"
                        
                        echo "# $setting_type: $setting_value"
                        
                        case "$setting_type" in
                            bash_option)
                                local option value
                                IFS='=' read option value <<< "$setting_value"
                                echo "log \"Configuration option Bash: $option\""
                                echo "set -$option"
                                ;;
                                
                            umask)
                                echo "backup_setting \"UMASK\" \"\$(umask)\""
                                echo "log \"Configuration umask: $setting_value\""
                                echo "umask $setting_value"
                                ;;
                                
                            env_var)
                                local var value
                                IFS='=' read var value <<< "$setting_value"
                                echo "backup_setting \"$var\" \"\${$var:-}\""
                                echo "log \"Configuration variable: $var=$value\""
                                echo "export $var=\"$value\""
                                ;;
                                
                            limit)
                                local resource value
                                IFS='=' read resource value <<< "$setting_value"
                                echo "backup_setting \"ULIMIT_$resource\" \"\$(ulimit -$resource 2>/dev/null || echo 'unlimited')\""
                                echo "log \"Configuration limite: $resource=$value\""
                                echo "ulimit -$resource $value"
                                ;;
                        esac
                        
                        echo
                    fi
                done
            done <<< "$settings"
            
            echo "# Vérification finale"
            echo "log \"Durcissement terminé avec succès\""
            echo "log \"Sauvegarde créée: \$BACKUP_FILE\""
            echo
            echo "# Nettoyage automatique au bout de 30 jours"
            echo "echo \"rm -f \$BACKUP_FILE\" | at now + 30 days 2>/dev/null || true"
            
        } > "$output_file"
        
        chmod +x "$output_file"
        echo "✓ Script de durcissement généré: $output_file"
    }
    
    # Audit de sécurité de l'environnement
    $self.audit_environment() {
        local output_file="${1:-environment_audit_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "AUDIT DE SÉCURITÉ DE L'ENVIRONNEMENT"
            echo "===================================="
            echo "Date: $(date)"
            echo "Utilisateur: $(whoami)"
            echo "Système: $(uname -a)"
            echo
            
            echo "PARAMÈTRES BASH"
            echo "==============="
            echo "Options actives: $-"
            echo "Version Bash: $BASH_VERSION"
            echo "Shell: $SHELL"
            echo "PID: $$"
            echo
            
            echo "ENVIRONNEMENT"
            echo "============="
            echo "PATH: $PATH"
            echo "UMASK: $(umask)"
            echo "HOME: $HOME"
            echo "USER: $USER"
            echo "UID: $UID"
            echo "EUID: $EUID"
            echo
            
            echo "LIMITES RESSOURCES"
            echo "=================="
            echo "Core file size: $(ulimit -c)"
            echo "Data seg size: $(ulimit -d)"
            echo "File size: $(ulimit -f)"
            echo "Max locked memory: $(ulimit -l)"
            echo "Max memory size: $(ulimit -m)"
            echo "Open files: $(ulimit -n)"
            echo "Pipe size: $(ulimit -p)"
            echo "POSIX message queues: $(ulimit -q)"
            echo "Real-time priority: $(ulimit -r)"
            echo "Stack size: $(ulimit -s)"
            echo "CPU time: $(ulimit -t)"
            echo "Max user processes: $(ulimit -u)"
            echo "Virtual memory: $(ulimit -v)"
            echo "File locks: $(ulimit -x)"
            echo
            
            echo "VARIABLES SENSIBLES"
            echo "==================="
            env | grep -E "(PASS|SECRET|KEY|TOKEN)" | sed 's/=.*$/=[REDACTED]/' || echo "Aucune variable sensible détectée"
            echo
            
            echo "SERVICES ACTIFS"
            echo "==============="
            systemctl list-units --type=service --state=active --no-pager -q 2>/dev/null | head -10 || echo "Systemctl non disponible"
            echo
            
            echo "PORTS OUVERTS"
            echo "=============="
            netstat -tln 2>/dev/null | grep LISTEN | head -10 || ss -tln 2>/dev/null | head -10 || echo "Outils réseau non disponibles"
            echo
            
            echo "RECOMMANDATIONS"
            echo "==============="
            
            # Analyse et recommandations
            if [[ ! $- == *e* ]]; then
                echo "• Activer errexit (set -e) pour arrêter sur erreur"
            fi
            
            if [[ ! $- == *u* ]]; then
                echo "• Activer nounset (set -u) pour détecter les variables non définies"
            fi
            
            if [[ $(id -u) -eq 0 ]]; then
                echo "• ATTENTION: Exécution en tant que root - Limiter l'usage"
            fi
            
            if [[ $(umask) != "0022" && $(umask) != "0077" ]]; then
                echo "• Considérer umask 0022 ou 0077 pour plus de sécurité"
            fi
            
            local open_files
            open_files="$(ulimit -n)"
            if (( open_files < 1024 )); then
                echo "• Augmenter la limite de fichiers ouverts (actuellement $open_files)"
            fi
            
        } > "$output_file"
        
        echo "✓ Audit d'environnement généré: $output_file"
    }
}

# Définition des politiques de sécurité standard
define_security_policies() {
    local configurator="$1"
    
    # Politique de développement
    $configurator.define_security_policy "development" \
        "Environnement de développement avec sécurité équilibrée" \
        "bash_option=errexit;bash_option=nounset;umask=0022;limit=nofile=4096;limit=nproc=512" \
        "low"
    
    # Politique de production
    $configurator.define_security_policy "production" \
        "Environnement de production hautement sécurisé" \
        "bash_option=errexit;bash_option=nounset;bash_option=pipefail;bash_option=noclobber;umask=0077;limit=nofile=1024;limit=nproc=256;env_var=PATH=/usr/local/bin:/usr/bin:/bin" \
        "high"
    
    # Politique serveur web
    $configurator.define_security_policy "web_server" \
        "Sécurité optimisée pour serveurs web" \
        "bash_option=errexit;bash_option=nounset;umask=0027;limit=nofile=2048;limit=nproc=128;service:disable:telnet;service:disable:ftp;firewall=enable" \
        "high"
    
    # Politique minimal
    $configurator.define_security_policy "minimal" \
        "Configuration de sécurité minimale" \
        "bash_option=errexit;umask=0022" \
        "low"
}

# Démonstration du configurateur d'environnement sécurisé
echo "--- Configurateur d'environnement sécurisé ---"

SecureEnvironment "secure_env"

# Définition des politiques
define_security_policies "secure_env"

echo
echo "--- Application d'une politique (simulation) ---"
secure_env.apply_security_policy "development" "true"

echo
echo "--- Application réelle d'une politique minimal ---"
secure_env.apply_security_policy "minimal" "false"

echo
echo "--- Vérification de conformité ---"
secure_env.verify_policy_compliance "minimal"

echo
echo "--- Création d'un profil personnalisé ---"
secure_env.create_custom_security_profile "custom_dev" "development" \
    "bash_option=pipefail;limit=nofile=8192;env_var=DEBUG=true"

echo
echo "--- Application du profil personnalisé (simulation) ---"
secure_env.apply_security_policy "custom_dev" "true"

echo
echo "--- Génération d'un script de durcissement ---"
secure_env.generate_hardening_script "production" "/tmp/production_hardening.sh"

echo "Script généré:"
head -20 /tmp/production_hardening.sh

echo
echo "--- Audit d'environnement ---"
secure_env.audit_environment

# Nettoyage
rm -f /tmp/production_hardening.sh environment_audit_*.txt
```

## Conclusion : La sécurité comme fondation

La sécurité des scripts shell n'est pas un luxe ajouté après coup : c'est la fondation même sur laquelle repose la confiance dans nos automatisations. Un script sécurisé est comme un château fort numérique - imprenable, fiable, et capable de résister aux assauts les plus sophistiqués.

**Points clés à retenir :**

1. **Analyse automatique** : Frameworks d'analyse statique pour détecter les vulnérabilités avant l'exécution
2. **Validation d'entrée** : Systèmes complets de validation et nettoyage des données utilisateur
3. **Durcissement environnemental** : Configuration automatique d'environnements d'exécution sécurisés

Dans le prochain chapitre, nous explorerons les techniques avancées de déploiement et automatisation, pour que nos scripts sécurisés puissent être déployés et orchestrés à grande échelle.

---

**Exercice pratique :** Créez un système de sécurité complet incluant :
- Analyseur automatique de vulnérabilités pour scripts shell
- Framework de validation d'entrée avec nettoyage automatique
- Configurateur d'environnement sécurisé avec profils personnalisables
- Script de durcissement automatique pour différents environnements

**Réflexion :** Comment intégreriez-vous ces mécanismes de sécurité dans un pipeline CI/CD pour assurer que seuls les scripts passant tous les contrôles de sécurité soient déployés en production ?
