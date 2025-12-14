# Chapitre 97 - Métaprogrammation et génération de code

> "La métaprogrammation n'est pas de la magie : c'est l'art de faire écrire le code par le code lui-même, où les programmes deviennent capables de s'étendre, de s'adapter, et d'évoluer au-delà de leurs limites initiales." - MetaProgramming Sage

## Introduction : Le code qui s'écrit lui-même

Imaginez-vous en tant qu'architecte d'un bâtiment capable de se reconstruire lui-même : chaque script Bash devient un méta-programme capable de générer du code, de s'étendre dynamiquement, et d'évoluer au-delà de sa forme initiale. La métaprogrammation en Bash transcende la programmation traditionnelle - elle permet aux scripts de s'analyser eux-mêmes, de se modifier, et de créer de nouveaux programmes à la volée.

Dans ce chapitre, nous construirons les fondations de la programmation autoréférentielle : exécution dynamique de code, génération de programmes, templates intelligents, et systèmes capables d'écrire leur propre évolution.

## Section 1 : Exécution dynamique de code

### 1.1 Framework d'évaluation sécurisée

Système d'exécution de code dynamique avec sandboxing et validation :

```bash
#!/bin/bash

# Framework d'évaluation sécurisée
echo "=== Framework d'évaluation sécurisée ==="

# Secure Evaluation Framework
SecureEvaluator() {
    local self="$1"
    
    declare -A $self._allowed_commands
    declare -A $self._security_policies
    declare -A $self._evaluation_history
    declare -A $self._code_templates
    
    # Enregistrement d'une commande autorisée
    $self.allow_command() {
        local command_name="$1"
        local allowed_args="$2"
        local risk_level="${3:-low}"  # low, medium, high
        
        $self._allowed_commands["$command_name"]="$allowed_args"
        $self._security_policies["${command_name}_risk"]="$risk_level"
        
        echo "✓ Commande autorisée: $command_name (risque: $risk_level)"
    }
    
    # Définition d'une politique de sécurité
    $self.set_security_policy() {
        local policy_name="$1"
        local restrictions="$2"
        
        $self._security_policies["$policy_name"]="$restrictions"
        echo "✓ Politique définie: $policy_name"
    }
    
    # Évaluation sécurisée d'une expression
    $self.safe_eval() {
        local expression="$1"
        local context="${2:-default}"
        local timeout="${3:-10}"
        
        echo "Évaluation sécurisée: $expression"
        echo "Contexte: $context | Timeout: ${timeout}s"
        
        # Validation de sécurité
        if ! $self._validate_expression "$expression" "$context"; then
            echo "❌ Expression rejetée pour raisons de sécurité"
            $self._log_evaluation "$expression" "rejected" "security_violation"
            return 1
        fi
        
        # Exécution avec timeout et capture
        local result=""
        local exit_code=0
        
        # Création d'un sous-shell isolé pour l'évaluation
        result="$(timeout "$timeout" bash -c "
            # Contexte d'exécution limité
            export PATH='/bin:/usr/bin'
            unset BASH_ENV
            unset ENV
            
            # Évaluation de l'expression
            eval \"$expression\"
        " 2>&1)" || exit_code=$?
        
        if [[ $exit_code -eq 124 ]]; then
            echo "❌ Timeout dépassé"
            $self._log_evaluation "$expression" "timeout" ""
            return 124
        elif [[ $exit_code -ne 0 ]]; then
            echo "❌ Erreur d'évaluation (code: $exit_code)"
            $self._log_evaluation "$expression" "error" "$exit_code"
            return $exit_code
        fi
        
        echo "✓ Résultat: $result"
        $self._log_evaluation "$expression" "success" "$result"
        
        # Retourner le résultat
        echo "$result"
    }
    
    # Validation d'une expression
    $self._validate_expression() {
        local expression="$1"
        local context="$2"
        
        # Liste des patterns dangereux
        local dangerous_patterns=(
            'rm[[:space:]]\+.*-rf'
            'mkfs|fdisk|dd'
            'chmod[[:space:]]\+777'
            'curl.*\|\|.*bash'
            'wget.*\|\|.*sh'
            'eval.*rm'
            'source.*\/dev'
            '\$\(.*rm.*\)'
            '>\s*/dev'
            'pkill.*-9.*-f'
        )
        
        # Vérification des patterns dangereux
        for pattern in "${dangerous_patterns[@]}"; do
            if echo "$expression" | grep -qE "$pattern"; then
                echo "Pattern dangereux détecté: $pattern" >&2
                return 1
            fi
        done
        
        # Validation selon le contexte
        case "$context" in
            math)
                # Contexte mathématique - seulement les opérations autorisées
                if ! echo "$expression" | grep -qE '^[0-9+\-*/().\s]+$'; then
                    echo "Expression mathématique invalide" >&2
                    return 1
                fi
                ;;
                
            file_ops)
                # Opérations sur fichiers - seulement ls, stat, etc.
                local allowed_file_cmds="ls|stat|file|basename|dirname|readlink"
                if ! echo "$expression" | grep -qE "^($allowed_file_cmds)"; then
                    echo "Commande fichier non autorisée" >&2
                    return 1
                fi
                ;;
                
            safe)
                # Contexte très restrictif
                local allowed_safe="echo|printf|date|whoami|pwd|id"
                if ! echo "$expression" | grep -qE "^($allowed_safe)"; then
                    echo "Commande non autorisée en mode safe" >&2
                    return 1
                fi
                ;;
        esac
        
        return 0
    }
    
    # Journalisation des évaluations
    $self._log_evaluation() {
        local expression="$1"
        local status="$2"
        local details="$3"
        
        local timestamp
        timestamp="$(date +%s)"
        
        $self._evaluation_history["${timestamp}"]="$expression|$status|$details"
    }
    
    # Exécution de template de code
    $self.evaluate_template() {
        local template_name="$1"
        shift
        local -a variables=("$@")
        
        local template="${$self._code_templates[$template_name]}"
        
        if [[ -z "$template" ]]; then
            echo "❌ Template introuvable: $template_name" >&2
            return 1
        fi
        
        echo "Évaluation du template: $template_name"
        
        # Substitution des variables
        local code="$template"
        local i=0
        for var in "${variables[@]}"; do
            code="${code//\{$i\}/$var}"
            ((i++))
        done
        
        echo "Code généré:"
        echo "$code"
        echo
        
        # Évaluation sécurisée
        $self.safe_eval "$code" "safe"
    }
    
    # Définition d'un template de code
    $self.define_code_template() {
        local template_name="$1"
        local template_code="$2"
        
        $self._code_templates["$template_name"]="$template_code"
        echo "✓ Template défini: $template_name"
    }
    
    # Génération de code à partir de spécifications
    $self.generate_code_from_spec() {
        local spec_file="$1"
        local output_file="$2"
        
        if [[ ! -f "$spec_file" ]]; then
            echo "❌ Fichier de spécification introuvable: $spec_file" >&2
            return 1
        fi
        
        echo "Génération de code depuis: $spec_file"
        
        # Parsing simple du fichier de spécification
        local generated_code="#!/bin/bash
# Code généré automatiquement depuis $spec_file
# Généré le: $(date)

"
        
        while IFS=':' read -r directive params; do
            case "$directive" in
                FUNCTION)
                    local func_name func_body
                    IFS='|' read func_name func_body <<< "$params"
                    generated_code="${generated_code}
# Fonction générée: $func_name
$func_name() {
    $func_body
}
"
                    ;;
                    
                VARIABLE)
                    local var_name var_value
                    IFS='=' read var_name var_value <<< "$params"
                    generated_code="${generated_code}
# Variable générée: $var_name
${var_name}=\"$var_value\"
"
                    ;;
                    
                COMMAND)
                    generated_code="${generated_code}
# Commande générée
$params
"
                    ;;
            esac
        done < "$spec_file"
        
        # Ajout du code d'initialisation
        generated_code="${generated_code}
# Code d'initialisation
main() {
    echo \"Programme généré exécuté\"
    # Ajoutez ici la logique principale
}

# Point d'entrée
if [[ \"\${BASH_SOURCE[0]}\" == \"\$0\" ]]; then
    main \"\$@\"
fi
"
        
        echo "$generated_code" > "$output_file"
        chmod +x "$output_file"
        
        echo "✓ Code généré: $output_file"
    }
    
    # Analyse statique de code
    $self.static_code_analysis() {
        local code_file="$1"
        
        if [[ ! -f "$code_file" ]]; then
            echo "❌ Fichier introuvable: $code_file" >&2
            return 1
        fi
        
        echo "=== ANALYSE STATIQUE: $(basename "$code_file") ==="
        
        local code_content
        code_content="$(cat "$code_file")"
        
        local issues_found=0
        
        # Analyse des patterns problématiques
        if echo "$code_content" | grep -q "eval"; then
            echo "⚠️  Utilisation d'eval détectée - Risque de sécurité"
            ((issues_found++))
        fi
        
        if echo "$code_content" | grep -q "source.*\$"; then
            echo "⚠️  Source dynamique détecté - Risque d'inclusion arbitraire"
            ((issues_found++))
        fi
        
        if ! echo "$code_content" | grep -q "set -euo pipefail"; then
            echo "ℹ️  Options de sécurité manquantes (set -euo pipefail)"
            ((issues_found++))
        fi
        
        local unquoted_vars
        unquoted_vars=$(echo "$code_content" | grep -c '\$[A-Za-z_][A-Za-z0-9_]*[^"]')
        if (( unquoted_vars > 0 )); then
            echo "⚠️  Variables non quotées: $unquoted_vars occurrences"
            ((issues_found++))
        fi
        
        local functions_without_docs
        functions_without_docs=$(echo "$code_content" | grep -c "function.*{" | grep -v "#" | wc -l)
        if (( functions_without_docs > 0 )); then
            echo "ℹ️  Fonctions sans commentaires: $functions_without_docs"
            ((issues_found++))
        fi
        
        echo
        echo "Résumé de l'analyse: $issues_found problèmes détectés"
        
        return $(( issues_found > 0 ))
    }
    
    # Optimisation de code
    $self.optimize_code() {
        local input_file="$1"
        local output_file="$2"
        
        if [[ ! -f "$input_file" ]]; then
            echo "❌ Fichier introuvable: $input_file" >&2
            return 1
        fi
        
        echo "Optimisation du code: $input_file -> $output_file"
        
        local code_content
        code_content="$(cat "$input_file")"
        
        # Optimisations simples
        local optimized_code="$code_content"
        
        # Suppression des commentaires vides
        optimized_code="$(echo "$optimized_code" | sed '/^[[:space:]]*#[[:space:]]*$/d')"
        
        # Fusion des lignes vides consécutives
        optimized_code="$(echo "$optimized_code" | sed '/^$/N;/^\n$/d')"
        
        # Optimisation des echo successifs
        optimized_code="$(echo "$optimized_code" | sed ':a;N;$!ba;s/echo "\([^"]*\)"\necho "\([^"]*\)"/echo "\1\2"/g')"
        
        # Suppression des variables inutilisées (détection simple)
        # Note: Cette optimisation est basique et peut produire des faux positifs
        
        echo "$optimized_code" > "$output_file"
        chmod +x "$output_file"
        
        local original_size="${#code_content}"
        local optimized_size="${#optimized_code}"
        local reduction=$(( (original_size - optimized_size) * 100 / original_size ))
        
        echo "✓ Code optimisé: ${reduction}% de réduction"
    }
    
    # Génération de rapport d'évaluation
    $self.generate_evaluation_report() {
        local output_file="${1:-evaluation_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT D'ÉVALUATION SÉCURISÉE"
            echo "==============================="
            echo "Généré le: $(date)"
            echo
            
            echo "COMMANDES AUTORISÉES"
            echo "===================="
            
            for cmd in "${!$self._allowed_commands[@]}"; do
                if [[ "$cmd" != *_risk ]]; then
                    local args="${$self._allowed_commands[$cmd]}"
                    local risk="${$self._security_policies[${cmd}_risk]}"
                    echo "$cmd ($risk): $args"
                fi
            done
            
            echo
            echo "HISTORIQUE DES ÉVALUATIONS"
            echo "=========================="
            
            local total_evaluations=0 successful_evals=0 failed_evals=0
            
            for timestamp in "${!$self._evaluation_history[@]}"; do
                local entry="${$self._evaluation_history[$timestamp]}"
                local expression status details
                IFS='|' read expression status details <<< "$entry"
                
                echo "$(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S'): $status"
                echo "  Expression: ${expression:0:50}..."
                echo "  Détails: $details"
                echo
                
                ((total_evaluations++))
                
                case "$status" in
                    success) ((successful_evals++)) ;;
                    rejected|timeout|error) ((failed_evals++)) ;;
                esac
            done
            
            echo "STATISTIQUES"
            echo "============"
            
            echo "Total évaluations: $total_evaluations"
            echo "Réussies: $successful_evals"
            echo "Échouées: $failed_evals"
            
            if (( total_evaluations > 0 )); then
                local success_rate=$(( successful_evals * 100 / total_evaluations ))
                echo "Taux de succès: ${success_rate}%"
            fi
            
            echo
            echo "RECOMMANDATIONS"
            echo "==============="
            
            if (( failed_evals > total_evaluations / 2 )); then
                echo "• Taux d'échec élevé - vérifier les politiques de sécurité"
            fi
            
            local dangerous_patterns=("eval" "source.*\$" "rm.*-rf")
            local dangerous_found=0
            
            for timestamp in "${!$self._evaluation_history[@]}"; do
                local entry="${$self._evaluation_history[$timestamp]}"
                local expression status
                IFS='|' read expression status <<< "$entry"
                
                for pattern in "${dangerous_patterns[@]}"; do
                    if echo "$expression" | grep -q "$pattern" && [[ "$status" == "rejected" ]]; then
                        ((dangerous_found++))
                    fi
                done
            done
            
            if (( dangerous_found > 0 )); then
                echo "• $dangerous_found tentatives de code dangereux bloquées"
            fi
            
            echo "• Maintenir les politiques de sécurité à jour"
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Exécution de code avec profiling
    $self.profile_code_execution() {
        local code="$1"
        local iterations="${2:-10}"
        
        echo "=== PROFILING CODE ==="
        echo "Itérations: $iterations"
        
        local total_time=0
        local min_time=999999
        local max_time=0
        
        for ((i=1; i<=iterations; i++)); do
            echo -n "Itération $i... "
            
            local start_time
            start_time="$(date +%s.%N)"
            
            $self.safe_eval "$code" "safe" "30" >/dev/null 2>&1
            
            local end_time
            end_time="$(date +%s.%N)"
            
            local duration
            duration="$(echo "$end_time - $start_time" | bc -l 2>/dev/null || echo "0")"
            
            total_time="$(echo "$total_time + $duration" | bc -l)"
            
            if (( $(echo "$duration < $min_time" | bc -l) )); then
                min_time="$duration"
            fi
            
            if (( $(echo "$duration > $max_time" | bc -l) )); then
                max_time="$duration"
            fi
            
            echo "${duration}s"
        done
        
        local avg_time
        avg_time="$(echo "scale=4; $total_time / $iterations" | bc -l)"
        
        echo
        echo "Résultats du profiling:"
        echo "  Temps moyen: ${avg_time}s"
        echo "  Temps minimum: ${min_time}s"
        echo "  Temps maximum: ${max_time}s"
        echo "  Temps total: ${total_time}s"
    }
}

# Définition des commandes autorisées
define_allowed_commands() {
    local evaluator="$1"
    
    $evaluator.allow_command "echo" "text" "low"
    $evaluator.allow_command "printf" "format string" "low"
    $evaluator.allow_command "date" "+format" "low"
    $evaluator.allow_command "whoami" "" "low"
    $evaluator.allow_command "pwd" "" "low"
    $evaluator.allow_command "ls" "-la directory" "medium"
    $evaluator.allow_command "grep" "pattern file" "medium"
    $evaluator.allow_command "awk" "program file" "high"
    $evaluator.allow_command "sed" "expression file" "high"
}

# Définition des templates de code
define_code_templates() {
    local evaluator="$1"
    
    $evaluator.define_code_template "simple_function" "
# Template: Fonction simple
my_function_{0}() {
    local param=\"\$1\"
    echo \"Fonction {0} appelée avec: \$param\"
    {1}
}
"
    
    $evaluator.define_code_template "error_handler" "
# Template: Gestionnaire d'erreur
handle_error_{0}() {
    local error_code=\"\$?\"
    local error_message=\"\$1\"
    
    if [[ \$error_code -ne 0 ]]; then
        echo \"Erreur dans {0}: \$error_message\" >&2
        {1}
        exit \$error_code
    fi
}
"
    
    $evaluator.define_code_template "config_reader" "
# Template: Lecteur de configuration
read_config_{0}() {
    local config_file=\"\$1\"
    local -A config=()
    
    while IFS='=' read -r key value; do
        [[ \$key =~ ^[[:space:]]*# ]] && continue
        config[\"\$key\"]=\"\$value\"
    done < \"\$config_file\"
    
    {1}
}
"
}

# Démonstration du framework d'évaluation sécurisée
echo "--- Framework d'évaluation sécurisée ---"

SecureEvaluator "secure_eval"

# Définition des commandes autorisées
define_allowed_commands "secure_eval"

# Définition des politiques de sécurité
secure_eval.set_security_policy "max_execution_time" "30"
secure_eval.set_security_policy "forbidden_patterns" "rm.*-rf|mkfs|dd.*if=/dev"

# Définition des templates
define_code_templates "secure_eval"

echo
echo "--- Tests d'évaluation sécurisée ---"

# Tests d'expressions sûres
echo "1. Expression mathématique:"
secure_eval.safe_eval "echo \$((2 + 3 * 4))" "math"

echo
echo "2. Commande système sûre:"
secure_eval.safe_eval "echo 'Hello, World!'" "safe"

echo
echo "3. Expression dangereuse (devrait être rejetée):"
secure_eval.safe_eval "rm -rf /tmp/*" "safe"

echo
echo "4. Expression avec timeout:"
secure_eval.safe_eval "sleep 2 && echo 'Done'" "safe" "1"

echo
echo "--- Évaluation de templates ---"

echo "5. Template de fonction:"
secure_eval.evaluate_template "simple_function" "greet" "echo \"Hello from generated function\""

echo
echo "6. Template de gestion d'erreur:"
secure_eval.evaluate_template "error_handler" "database" "logger \"Database error\""

echo
echo "--- Génération de code depuis spécifications ---"

# Création d'un fichier de spécifications
cat > /tmp/code_spec.txt << 'EOF'
FUNCTION:backup_files|echo "Sauvegarde des fichiers: $1"; cp "$1" "$1.backup"
VARIABLE:BACKUP_DIR=/tmp/backups
COMMAND:mkdir -p "$BACKUP_DIR"
FUNCTION:cleanup_backups|find "$BACKUP_DIR" -name "*.backup" -mtime +7 -delete
EOF

secure_eval.generate_code_from_spec "/tmp/code_spec.txt" "/tmp/generated_script.sh"

echo "Script généré:"
head -20 /tmp/generated_script.sh

echo
echo "--- Analyse statique ---"
secure_eval.static_code_analysis "/tmp/generated_script.sh"

echo
echo "--- Optimisation de code ---"
secure_eval.optimize_code "/tmp/generated_script.sh" "/tmp/optimized_script.sh"

echo
echo "--- Profiling ---"
secure_eval.profile_code_execution "echo 'test'" "5"

echo
echo "--- Génération de rapport ---"
secure_eval.generate_evaluation_report

# Nettoyage
rm -f /tmp/code_spec.txt /tmp/generated_script.sh /tmp/optimized_script.sh evaluation_report_*.txt
```

### 1.2 Générateur de code auto-modifiant

Système capable de s'analyser et de se modifier lui-même :

```bash
#!/bin/bash

# Générateur de code auto-modifiant
echo "=== Générateur de code auto-modifiant ==="

# Self-Modifying Code Generator
SelfModifyingGenerator() {
    local self="$1"
    
    declare -A $self._code_reflection
    declare -A $self._modification_rules
    declare -A $self._evolution_history
    declare -A $self._code_patterns
    
    # Analyse de soi (réflexion)
    $self.reflect_on_self() {
        echo "=== RÉFLEXION SUR SOI ==="
        
        # Analyse du code source du générateur
        local self_source
        self_source="$(declare -f SelfModifyingGenerator)"
        
        # Extraction des fonctions
        local functions
        functions="$(echo "$self_source" | grep -E "^[[:space:]]*\$self\.[a-zA-Z_][a-zA-Z0-9_]*\(\)" | sed 's/.*\$self\.//;s/().*//')"
        
        echo "Fonctions découvertes:"
        echo "$functions" | while read -r func; do
            echo "  ✓ $func"
        done
        
        # Comptage des lignes
        local line_count
        line_count="$(echo "$self_source" | wc -l)"
        echo "Lignes de code: $line_count"
        
        # Analyse de complexité
        local complexity
        complexity="$(echo "$self_source" | grep -cE "(if|for|while|case)")"
        echo "Points de complexité: $complexity"
        
        # Stockage de la réflexion
        $self._code_reflection["function_count"]="$(echo "$functions" | wc -l)"
        $self._code_reflection["line_count"]="$line_count"
        $self._code_reflection["complexity"]="$complexity"
        $self._code_reflection["last_reflection"]="$(date +%s)"
    }
    
    # Définition d'une règle de modification
    $self.define_modification_rule() {
        local rule_name="$1"
        local trigger_condition="$2"
        local modification_action="$3"
        local description="$4"
        
        $self._modification_rules["${rule_name}_condition"]="$trigger_condition"
        $self._modification_rules["${rule_name}_action"]="$modification_action"
        $self._modification_rules["${rule_name}_description"]="$description"
        
        echo "✓ Règle de modification définie: $rule_name"
    }
    
    # Vérification et application des règles de modification
    $self.check_and_apply_modifications() {
        echo "=== VÉRIFICATION MODIFICATIONS ==="
        
        local modifications_applied=0
        
        for rule_key in "${!$self._modification_rules[@]}"; do
            if [[ "$rule_key" =~ _condition$ ]]; then
                local rule_name="${rule_key%_condition}"
                local condition="${$self._modification_rules[$rule_key]}"
                
                echo "Vérification règle: $rule_name"
                
                if $self._evaluate_modification_condition "$condition"; then
                    local action="${$self._modification_rules[${rule_name}_action]}"
                    local description="${$self._modification_rules[${rule_name}_description]}"
                    
                    echo "✓ Condition remplie: $description"
                    echo "Application de la modification..."
                    
                    if $self._apply_modification_action "$action"; then
                        echo "✓ Modification appliquée"
                        ((modifications_applied++))
                        
                        # Journalisation
                        $self._evolution_history["$(date +%s)"]="$rule_name|$description"
                    else
                        echo "❌ Échec de la modification"
                    fi
                else
                    echo "✗ Condition non remplie"
                fi
                
                echo
            fi
        done
        
        echo "Modifications appliquées: $modifications_applied"
        return $(( modifications_applied > 0 ))
    }
    
    # Évaluation d'une condition de modification
    $self._evaluate_modification_condition() {
        local condition="$1"
        
        case "$condition" in
            high_complexity)
                local complexity="${$self._code_reflection[complexity]}"
                (( complexity > 20 ))
                ;;
                
            many_functions)
                local func_count="${$self._code_reflection[function_count]}"
                (( func_count > 10 ))
                ;;
                
            old_reflection)
                local last_reflection="${$self._code_reflection[last_reflection]}"
                local age=$(( $(date +%s) - last_reflection ))
                (( age > 300 ))  # 5 minutes
                ;;
                
            error_prone)
                # Simulation de détection d'erreurs
                (( RANDOM % 10 < 3 ))  # 30% de chance
                ;;
                
            performance_issue)
                # Simulation de problème de performance
                (( RANDOM % 10 < 2 ))  # 20% de chance
                ;;
                
            *)
                false
                ;;
        esac
    }
    
    # Application d'une action de modification
    $self._apply_modification_action() {
        local action="$1"
        
        case "$action" in
            add_error_handling)
                echo "Ajout de gestion d'erreur globale..."
                # Simulation d'ajout de code
                $self._inject_code "error_handling" "
# Gestion d'erreur ajoutée automatiquement
trap 'echo \"Erreur détectée à la ligne \$LINENO\" >&2' ERR
set -e
"
                ;;
                
            optimize_performance)
                echo "Optimisation des performances..."
                $self._inject_code "performance" "
# Optimisations de performance ajoutées
export LC_ALL=C
shopt -s lastpipe
"
                ;;
                
            add_logging)
                echo "Ajout de logging..."
                $self._inject_code "logging" "
# Fonction de logging ajoutée
log_event() {
    echo \"[\$(date '+%Y-%m-%d %H:%M:%S')] \$*\" >> /tmp/self_modifying.log
}
"
                ;;
                
            refactor_functions)
                echo "Refactorisation des fonctions..."
                $self._inject_code "refactor" "
# Fonctions refactorisées pour plus de lisibilité
# (Code simplifié pour la démonstration)
"
                ;;
                
            add_self_test)
                echo "Ajout de tests automatiques..."
                $self._inject_code "testing" "
# Tests automatiques ajoutés
run_self_tests() {
    echo \"Exécution des tests automatiques...\"
    # Tests seraient implémentés ici
}
"
                ;;
                
            *)
                echo "Action inconnue: $action"
                return 1
                ;;
        esac
        
        return 0
    }
    
    # Injection de code (simulation)
    $self._inject_code() {
        local component="$1"
        local code="$2"
        
        echo "Injection de code dans le composant: $component"
        echo "Code injecté:"
        echo "$code"
        echo
        
        # En réalité, cela modifierait le code source du script
        # Ici, on simule seulement
        $self._code_reflection["injected_${component}"]="$(date +%s)"
    }
    
    # Évolution adaptative
    $self.adaptive_evolution() {
        local environment_factor="$1"
        
        echo "=== ÉVOLUTION ADAPTATIVE ==="
        echo "Facteur environnemental: $environment_factor"
        
        # Analyse de l'environnement
        case "$environment_factor" in
            high_load)
                echo "Environnement haute charge détecté"
                $self.define_modification_rule "load_optimization" "performance_issue" "optimize_performance" "Optimisation pour haute charge"
                ;;
                
            error_prone)
                echo "Environnement propice aux erreurs"
                $self.define_modification_rule "error_resilience" "error_prone" "add_error_handling" "Amélioration de la résilience"
                ;;
                
            development)
                echo "Environnement de développement"
                $self.define_modification_rule "dev_enhancements" "old_reflection" "add_logging" "Améliorations développement"
                ;;
                
            production)
                echo "Environnement de production"
                $self.define_modification_rule "prod_hardening" "high_complexity" "refactor_functions" "Durcissement production"
                ;;
        esac
        
        # Application des évolutions
        $self.check_and_apply_modifications
    }
    
    # Génération de code enfant (reproduction)
    $self.generate_offspring() {
        local offspring_name="$1"
        local modifications="$2"
        
        echo "=== GÉNÉRATION DE DESCENDANCE ==="
        echo "Nom: $offspring_name"
        
        # Création d'une copie modifiée
        local offspring_code="
#!/bin/bash
# Code généré automatiquement - Descendance de $(basename \"\$0\")
# Généré le: $(date)
# Modifications: $modifications

# Code de base (simplifié)
echo \"Programme enfant: $offspring_name\"
echo \"Modifications appliquées: $modifications\"

# Fonction héritée
inherited_function() {
    echo \"Fonction héritée du parent\"
}

# Point d'entrée
main() {
    echo \"Exécution du programme enfant\"
    inherited_function
}

if [[ \"\${BASH_SOURCE[0]}\" == \"\$0\" ]]; then
    main \"\$@\"
fi
"
        
        local offspring_file="/tmp/${offspring_name}.sh"
        echo "$offspring_code" > "$offspring_file"
        chmod +x "$offspring_file"
        
        echo "✓ Descendance générée: $offspring_file"
        
        # Test de l'enfant
        echo "Test de l'enfant:"
        bash "$offspring_file"
    }
    
    # Apprentissage par expérience
    $self.learn_from_experience() {
        local experience_type="$1"
        local outcome="$2"
        
        echo "=== APPRENTISSAGE EXPÉRIENTIEL ==="
        echo "Type: $experience_type | Résultat: $outcome"
        
        # Stockage de l'expérience
        local experience_key="${experience_type}_$(date +%s)"
        $self._evolution_history["experience_$experience_key"]="$outcome"
        
        # Apprentissage adaptatif
        case "$experience_type:$outcome" in
            modification:success)
                echo "✓ Apprentissage: Les modifications réussissent souvent"
                $self._code_reflection["modification_success_rate"]="high"
                ;;
                
            modification:failure)
                echo "⚠️ Apprentissage: Les modifications peuvent échouer"
                $self._code_reflection["modification_risk"]="high"
                ;;
                
            execution:success)
                echo "✓ Apprentissage: L'exécution est stable"
                $self._code_reflection["execution_stability"]="high"
                ;;
                
            execution:failure)
                echo "⚠️ Apprentissage: Problèmes d'exécution détectés"
                $self._code_reflection["execution_stability"]="low"
                ;;
        esac
        
        # Adaptation des règles basée sur l'apprentissage
        if [[ "${$self._code_reflection[modification_risk]}" == "high" ]]; then
            echo "Réduction de l'agressivité des modifications automatiques"
            # On pourrait désactiver certaines règles de modification
        fi
    }
    
    # Auto-documentation
    $self.generate_self_documentation() {
        local output_file="${1:-self_documentation_$(date +%Y%m%d_%H%M%S).md}"
        
        {
            echo "# Auto-Documentation du Générateur Auto-Modifiant"
            echo ""
            echo "Généré automatiquement le: $(date)"
            echo ""
            echo "## Métriques de Réflexion"
            echo ""
            echo "- **Lignes de code**: ${$self._code_reflection[line_count]}"
            echo "- **Fonctions**: ${$self._code_reflection[function_count]}"
            echo "- **Complexité**: ${$self._code_reflection[complexity]}"
            echo "- **Dernière réflexion**: $(date -d "@${$self._code_reflection[last_reflection]}" '+%Y-%m-%d %H:%M:%S')"
            echo ""
            echo "## Composants Injectés"
            echo ""
            
            for key in "${!$self._code_reflection[@]}"; do
                if [[ "$key" =~ ^injected_ ]]; then
                    local component="${key#injected_}"
                    local timestamp="${$self._code_reflection[$key]}"
                    echo "- **$component**: Injecté le $(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S')"
                fi
            done
            
            echo ""
            echo "## Historique Évolutionnaire"
            echo ""
            
            for timestamp in "${!$self._evolution_history[@]}"; do
                if [[ "$timestamp" =~ ^[0-9]+$ ]]; then
                    local entry="${$self._evolution_history[$timestamp]}"
                    local rule description
                    IFS='|' read rule description <<< "$entry"
                    echo "- **$(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S')**: $rule - $description"
                fi
            done
            
            echo ""
            echo "## Règles de Modification"
            echo ""
            
            for rule_key in "${!$self._modification_rules[@]}"; do
                if [[ "$rule_key" =~ _description$ ]]; then
                    local rule_name="${rule_key%_description}"
                    local description="${$self._modification_rules[$rule_key]}"
                    echo "- **$rule_name**: $description"
                fi
            done
            
            echo ""
            echo "## État d'Apprentissage"
            echo ""
            echo "- **Stabilité d'exécution**: ${$self._code_reflection[execution_stability]:-unknown}"
            echo "- **Taux de succès des modifications**: ${$self._code_reflection[modification_success_rate]:-unknown}"
            echo "- **Risque de modification**: ${$self._code_reflection[modification_risk]:-unknown}"
            
        } > "$output_file"
        
        echo "✓ Auto-documentation générée: $output_file"
    }
    
    # Cycle de vie complet
    $self.lifecycle_cycle() {
        echo "=== CYCLE DE VIE COMPLET ==="
        
        # Phase 1: Réflexion
        echo "Phase 1: Réflexion"
        $self.reflect_on_self
        
        # Phase 2: Apprentissage
        echo "Phase 2: Apprentissage"
        $self.learn_from_experience "execution" "success"
        
        # Phase 3: Évolution
        echo "Phase 3: Évolution adaptative"
        $self.adaptive_evolution "development"
        
        # Phase 4: Reproduction
        echo "Phase 4: Reproduction"
        $self.generate_offspring "child_$(date +%s)" "logging_error_handling"
        
        # Phase 5: Documentation
        echo "Phase 5: Auto-documentation"
        $self.generate_self_documentation
        
        echo "✓ Cycle de vie terminé"
    }
}

# Définition des règles de modification évolutive
define_evolution_rules() {
    local generator="$1"
    
    $generator.define_modification_rule "complexity_reduction" "high_complexity" "refactor_functions" "Réduction de la complexité cyclomatique"
    $generator.define_modification_rule "error_prevention" "error_prone" "add_error_handling" "Prévention des erreurs d'exécution"
    $generator.define_modification_rule "performance_boost" "performance_issue" "optimize_performance" "Amélioration des performances"
    $generator.define_modification_rule "self_improvement" "old_reflection" "add_self_test" "Amélioration continue de soi"
    $generator.define_modification_rule "observability" "many_functions" "add_logging" "Amélioration de l'observabilité"
}

# Démonstration du générateur auto-modifiant
echo "--- Générateur de code auto-modifiant ---"

SelfModifyingGenerator "self_modifier"

# Définition des règles d'évolution
define_evolution_rules "self_modifier"

echo
echo "--- Cycle de vie complet ---"
self_modifier.lifecycle_cycle

echo
echo "--- Tests individuels ---"

echo "1. Réflexion sur soi:"
self_modifier.reflect_on_self

echo
echo "2. Vérification des modifications:"
self_modifier.check_and_apply_modifications

echo
echo "3. Évolution adaptative:"
self_modifier.adaptive_evolution "production"

echo
echo "4. Apprentissage:"
self_modifier.learn_from_experience "modification" "success"

echo
echo "5. Génération d'un enfant:"
self_modifier.generate_offspring "test_child" "minimal_modifications"

# Nettoyage
rm -f /tmp/child_*.sh self_documentation_*.md
```

## Section 2 : Templates et génération de code

### 2.1 Moteur de templates intelligents

Système de génération de code basé sur templates auto-adaptatifs :

```bash
#!/bin/bash

# Moteur de templates intelligents
echo "=== Moteur de templates intelligents ==="

# Intelligent Template Engine
IntelligentTemplateEngine() {
    local self="$1"
    
    declare -A $self._templates
    declare -A $self._template_metadata
    declare -A $self._generation_history
    declare -A $self._template_dependencies
    
    # Définition d'un template intelligent
    $self.define_intelligent_template() {
        local template_name="$1"
        local template_content="$2"
        local metadata="$3"
        
        $self._templates["$template_name"]="$template_content"
        $self._template_metadata["${template_name}_metadata"]="$metadata"
        
        # Extraction automatique des variables
        local variables
        variables="$(echo "$template_content" | grep -oE '\{\{[a-zA-Z_][a-zA-Z0-9_]*\}\}' | sort | uniq | sed 's/[{}]//g')"
        $self._template_metadata["${template_name}_variables"]="$variables"
        
        echo "✓ Template intelligent défini: $template_name"
    }
    
    # Génération de code depuis template
    $self.generate_from_template() {
        local template_name="$1"
        local output_file="$2"
        shift 2
        local -A variables=("$@")
        
        local template="${$self._templates[$template_name]}"
        
        if [[ -z "$template" ]]; then
            echo "❌ Template introuvable: $template_name" >&2
            return 1
        fi
        
        echo "Génération depuis template: $template_name -> $output_file"
        
        # Substitution des variables
        local generated_code="$template"
        
        for var_name in "${!variables[@]}"; do
            local placeholder="{{${var_name}}}"
            local value="${variables[$var_name]}"
            generated_code="${generated_code//$placeholder/$value}"
        done
        
        # Vérification des variables non substituées
        local unsubstituted
        unsubstituted="$(echo "$generated_code" | grep -oE '\{\{[a-zA-Z_][a-zA-Z0-9_]*\}\}' | sort | uniq)"
        
        if [[ -n "$unsubstituted" ]]; then
            echo "⚠️ Variables non substituées détectées:"
            echo "$unsubstituted" | sed 's/^/  - /'
        fi
        
        # Optimisation du code généré
        generated_code="$($self._optimize_generated_code "$generated_code")"
        
        # Ajout d'en-tête
        local header="#!/bin/bash
# Code généré automatiquement depuis template: $template_name
# Généré le: $(date)
# Variables utilisées: ${!variables[*]}

"
        
        echo "${header}${generated_code}" > "$output_file"
        chmod +x "$output_file"
        
        # Historique
        $self._generation_history["$(date +%s)"]="$template_name|$output_file|${#generated_code}"
        
        echo "✓ Code généré: $output_file (${#generated_code} caractères)"
    }
    
    # Optimisation du code généré
    $self._optimize_generated_code() {
        local code="$1"
        
        # Optimisations basiques
        local optimized="$code"
        
        # Suppression des commentaires vides
        optimized="$(echo "$optimized" | sed '/^[[:space:]]*#[[:space:]]*$/d')"
        
        # Fusion des lignes vides
        optimized="$(echo "$optimized" | sed '/^$/N;/^\n$/d')"
        
        # Optimisation des conditions simples
        optimized="$(echo "$optimized" | sed 's/if \[ \([^]]*\) \]; then/if [[ \1 ]]; then/g')"
        
        echo "$optimized"
    }
    
    # Template conditionnel
    $self.conditional_template() {
        local template_name="$1"
        local condition="$2"
        local true_template="$3"
        local false_template="${4:-}"
        
        $self._template_metadata["${template_name}_condition"]="$condition"
        $self._template_metadata["${template_name}_true_template"]="$true_template"
        
        if [[ -n "$false_template" ]]; then
            $self._template_metadata["${template_name}_false_template"]="$false_template"
        fi
        
        echo "✓ Template conditionnel défini: $template_name"
    }
    
    # Génération avec conditions
    $self.generate_conditional() {
        local template_name="$1"
        local output_file="$2"
        shift 2
        local -A variables=("$@")
        
        local condition="${$self._template_metadata[${template_name}_condition]}"
        
        # Évaluation de la condition
        local condition_result
        if eval "$condition" 2>/dev/null; then
            condition_result="true"
        else
            condition_result="false"
        fi
        
        echo "Condition '$condition' évaluée à: $condition_result"
        
        # Sélection du template approprié
        local selected_template
        if [[ "$condition_result" == "true" ]]; then
            selected_template="${$self._template_metadata[${template_name}_true_template]}"
        else
            selected_template="${$self._template_metadata[${template_name}_false_template]}"
        fi
        
        if [[ -z "$selected_template" ]]; then
            echo "❌ Template conditionnel incomplet" >&2
            return 1
        fi
        
        # Génération depuis le template sélectionné
        $self.generate_from_template "$selected_template" "$output_file" "${variables[@]}"
    }
    
    # Template avec héritage
    $self.template_inheritance() {
        local child_template="$1"
        local parent_template="$2"
        local overrides="$3"
        
        $self._template_dependencies["${child_template}_parent"]="$parent_template"
        $self._template_dependencies["${child_template}_overrides"]="$overrides"
        
        echo "✓ Héritage défini: $child_template -> $parent_template"
    }
    
    # Génération avec héritage
    $self.generate_with_inheritance() {
        local template_name="$1"
        local output_file="$2"
        shift 2
        local -A variables=("$@")
        
        local parent="${$self._template_dependencies[${template_name}_parent]}"
        
        if [[ -z "$parent" ]]; then
            # Pas d'héritage, génération normale
            $self.generate_from_template "$template_name" "$output_file" "${variables[@]}"
            return $?
        fi
        
        echo "Génération avec héritage: $template_name -> $parent"
        
        # Fusion des templates
        local parent_content="${$self._templates[$parent]}"
        local child_content="${$self._templates[$template_name]}"
        local overrides="${$self._template_dependencies[${template_name}_overrides]}"
        
        # Application des overrides (simplifié)
        local merged_content="$parent_content"
        
        # Ici on pourrait implémenter une logique plus sophistiquée
        # Pour la démonstration, on concatène simplement
        if [[ -n "$child_content" ]]; then
            merged_content="${merged_content}
# Extensions depuis $template_name
${child_content}"
        fi
        
        # Création d'un template temporaire fusionné
        local merged_template="${template_name}_merged_$(date +%s)"
        $self._templates["$merged_template"]="$merged_content"
        
        # Génération
        $self.generate_from_template "$merged_template" "$output_file" "${variables[@]}"
        
        # Nettoyage
        unset $self._templates["$merged_template"]
    }
    
    # Template auto-adaptatif
    $self.adaptive_template() {
        local template_name="$1"
        local base_template="$2"
        local adaptation_rules="$3"
        
        $self._template_metadata["${template_name}_base"]="$base_template"
        $self._template_metadata["${template_name}_adaptations"]="$adaptation_rules"
        
        echo "✓ Template adaptatif défini: $template_name"
    }
    
    # Génération adaptative
    $self.generate_adaptive() {
        local template_name="$1"
        local output_file="$2"
        local context="$3"
        shift 3
        local -A variables=("$@")
        
        local base_template="${$self._template_metadata[${template_name}_base]}"
        local adaptations="${$self._template_metadata[${template_name}_adaptations]}"
        
        echo "Génération adaptative: $template_name (contexte: $context)"
        
        # Récupération du template de base
        local adapted_content="${$self._templates[$base_template]}"
        
        # Application des adaptations selon le contexte
        case "$context" in
            development)
                # Adaptations pour le développement
                adapted_content="$(echo "$adapted_content" | sed 's/{{log_level}}/DEBUG/g')"
                adapted_content="$(echo "$adapted_content" | sed 's/{{error_handling}}/set -x/g')"
                ;;
                
            production)
                # Adaptations pour la production
                adapted_content="$(echo "$adapted_content" | sed 's/{{log_level}}/ERROR/g')"
                adapted_content="$(echo "$adapted_content" | sed 's/{{error_handling}}/set -euo pipefail/g')"
                ;;
                
            testing)
                # Adaptations pour les tests
                adapted_content="$(echo "$adapted_content" | sed 's/{{log_level}}/INFO/g')"
                adapted_content="$(echo "$adapted_content" | sed 's/{{error_handling}}/set -e/g')"
                ;;
        esac
        
        # Création d'un template temporaire adapté
        local adapted_template="${template_name}_adapted_$(date +%s)"
        $self._templates["$adapted_template"]="$adapted_content"
        
        # Génération
        $self.generate_from_template "$adapted_template" "$output_file" "${variables[@]}"
        
        # Nettoyage
        unset $self._templates["$adapted_template"]
    }
    
    # Validation de template
    $self.validate_template() {
        local template_name="$1"
        
        local template="${$self._templates[$template_name]}"
        
        if [[ -z "$template" ]]; then
            echo "❌ Template introuvable: $template_name" >&2
            return 1
        fi
        
        echo "Validation du template: $template_name"
        
        local issues=0
        
        # Vérification de la syntaxe Bash de base
        if ! echo "$template" | bash -n 2>/dev/null; then
            echo "⚠️ Erreur de syntaxe détectée"
            ((issues++))
        fi
        
        # Vérification des variables non définies
        local undefined_vars
        undefined_vars="$(echo "$template" | grep -oE '\{\{[a-zA-Z_][a-zA-Z0-9_]*\}\}' | sort | uniq)"
        
        if [[ -n "$undefined_vars" ]]; then
            echo "Variables template détectées:"
            echo "$undefined_vars" | sed 's/^/  - /'
        fi
        
        # Vérification des bonnes pratiques
        if echo "$template" | grep -q "eval\|source.*\$"; then
            echo "⚠️ Code potentiellement dangereux détecté"
            ((issues++))
        fi
        
        if [[ $issues -eq 0 ]]; then
            echo "✅ Template valide"
            return 0
        else
            echo "⚠️ $issues problème(s) détecté(s)"
            return 1
        fi
    }
    
    # Génération de rapport de templates
    $self.generate_template_report() {
        local output_file="${1:-template_report_$(date +%Y%m%d_%H%M%S).txt}"
        
        {
            echo "RAPPORT DES TEMPLATES INTELLIGENTS"
            echo "==================================="
            echo "Généré le: $(date)"
            echo
            
            echo "TEMPLATES DÉFINIS"
            echo "================="
            
            for template_name in "${!$self._templates[@]}"; do
                local metadata="${$self._template_metadata[${template_name}_metadata]}"
                local variables="${$self._template_metadata[${template_name}_variables]}"
                
                echo "Template: $template_name"
                echo "  Métadonnées: $metadata"
                echo "  Variables: $variables"
                echo
            done
            
            echo "HISTORIQUE DE GÉNÉRATION"
            echo "========================"
            
            for timestamp in "${!$self._generation_history[@]}"; do
                local entry="${$self._generation_history[$timestamp]}"
                local template output size
                IFS='|' read template output size <<< "$entry"
                
                echo "$(date -d "@$timestamp" '+%Y-%m-%d %H:%M:%S'): $template -> $output (${size} octets)"
            done
            
            echo
            echo "DÉPENDANCES"
            echo "==========="
            
            for dep_key in "${!$self._template_dependencies[@]}"; do
                if [[ "$dep_key" =~ _parent$ ]]; then
                    local child="${dep_key%_parent}"
                    local parent="${$self._template_dependencies[$dep_key]}"
                    
                    echo "$child hérite de $parent"
                fi
            done
            
            echo
            echo "STATISTIQUES"
            echo "============"
            
            local total_templates="${#$self._templates[@]}"
            local total_generations="${#$self._generation_history[@]}"
            
            echo "Templates définis: $total_templates"
            echo "Générations effectuées: $total_generations"
            
            if (( total_generations > 0 )); then
                # Calcul de la taille moyenne générée
                local total_size=0
                for timestamp in "${!$self._generation_history[@]}"; do
                    local entry="${$self._generation_history[$timestamp]}"
                    local size
                    IFS='|' read _ _ size <<< "$entry"
                    total_size=$(( total_size + size ))
                done
                
                local avg_size=$(( total_size / total_generations ))
                echo "Taille moyenne générée: ${avg_size} octets"
            fi
            
        } > "$output_file"
        
        echo "✓ Rapport généré: $output_file"
    }
    
    # Auto-amélioration des templates
    $self.improve_templates() {
        echo "=== AMÉLIORATION AUTOMATIQUE DES TEMPLATES ==="
        
        for template_name in "${!$self._templates[@]}"; do
            local template="${$self._templates[$template_name]}"
            
            echo "Amélioration du template: $template_name"
            
            local improved_template="$template"
            local improvements=0
            
            # Amélioration 1: Ajout de commentaires manquants
            if ! echo "$template" | grep -q "^#.*function"; then
                improved_template="$(echo "$improved_template" | sed 's/^\([a-zA-Z_][a-zA-Z0-9_]*\)()/\n# Function: \1\nfunction \1/g')"
                ((improvements++))
            fi
            
            # Amélioration 2: Ajout de validation d'entrée
            if echo "$template" | grep -q '\$1' && ! echo "$template" | grep -q "if.*-z.*1"; then
                improved_template="$(echo "$improved_template" | sed 's/^\([a-zA-Z_][a-zA-Z0-9_]*\)()/\1() {\n    if [[ -z "\$1" ]]; then\n        echo "Erreur: paramètre requis" >&2\n        return 1\n    fi\n/g')"
                ((improvements++))
            fi
            
            # Amélioration 3: Optimisation des performances
            if echo "$template" | grep -q "for.*in.*\$\("; then
                improved_template="$(echo "$improved_template" | sed 's/for .* in \$(/while IFS= read -r /g')"
                ((improvements++))
            fi
            
            if [[ $improvements -gt 0 ]]; then
                $self._templates["$template_name"]="$improved_template"
                echo "✓ $improvements amélioration(s) appliquée(s)"
            else
                echo "✗ Aucune amélioration nécessaire"
            fi
            
            echo
        done
    }
}

# Définition des templates intelligents
define_smart_templates() {
    local engine="$1"
    
    # Template pour un script de sauvegarde
    $engine.define_intelligent_template "backup_script" '
#!/bin/bash
# Script de sauvegarde généré automatiquement

# Configuration
BACKUP_SOURCE="{{source_dir}}"
BACKUP_DEST="{{dest_dir}}"
COMPRESSION="{{compression}}"
RETENTION="{{retention_days}}"

# Fonction de sauvegarde
perform_backup() {
    local timestamp=$(date +%Y%m%d_%H%M%S)
    local backup_name="backup_${timestamp}.{{extension}}"
    local backup_path="${BACKUP_DEST}/${backup_name}"
    
    echo "Début de la sauvegarde: $timestamp"
    
    # Création du répertoire de destination
    mkdir -p "$BACKUP_DEST"
    
    # Compression selon le type
    case "{{compression}}" in
        gzip)
            tar -czf "$backup_path" -C "$BACKUP_SOURCE" .
            ;;
        bzip2)
            tar -cjf "$backup_path" -C "$BACKUP_SOURCE" .
            ;;
        xz)
            tar -cJf "$backup_path" -C "$BACKUP_SOURCE" .
            ;;
        none)
            cp -r "$BACKUP_SOURCE" "$backup_path"
            ;;
    esac
    
    # Vérification
    if [[ -f "$backup_path" ]] || [[ -d "$backup_path" ]]; then
        echo "✓ Sauvegarde créée: $backup_path"
        
        # Nettoyage des anciennes sauvegardes
        find "$BACKUP_DEST" -name "backup_*.{{extension}}" -mtime +{{retention_days}} -delete
        
        return 0
    else
        echo "❌ Échec de la sauvegarde"
        return 1
    fi
}

# Fonction de restauration
restore_backup() {
    local backup_file="$1"
    local restore_dest="${2:-{{restore_default}}}"
    
    if [[ ! -f "$backup_file" ]]; then
        echo "❌ Fichier de sauvegarde introuvable: $backup_file"
        return 1
    fi
    
    echo "Restauration depuis: $backup_file"
    
    mkdir -p "$restore_dest"
    
    case "{{compression}}" in
        gzip)
            tar -xzf "$backup_file" -C "$restore_dest"
            ;;
        bzip2)
            tar -xjf "$backup_file" -C "$restore_dest"
            ;;
        xz)
            tar -xJf "$backup_file" -C "$restore_dest"
            ;;
        none)
            cp -r "$backup_file" "$restore_dest"
            ;;
    esac
    
    echo "✓ Restauration terminée dans: $restore_dest"
}

# Point d\'entrée
main() {
    case "${1:-backup}" in
        backup)
            perform_backup
            ;;
        restore)
            restore_backup "$2" "$3"
            ;;
        list)
            echo "Sauvegardes disponibles:"
            ls -la "$BACKUP_DEST"/backup_*.{{extension}} 2>/dev/null || echo "Aucune sauvegarde trouvée"
            ;;
        *)
            echo "Usage: $0 {backup|restore|list} [args...]"
            exit 1
            ;;
    esac
}

# Exécution
if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
    {{error_handling}}
    main "$@"
fi
' "Script de sauvegarde configurable"
    
    # Template pour un service système
    $engine.define_intelligent_template "system_service" '
#!/bin/bash
# Service système généré automatiquement

SERVICE_NAME="{{service_name}}"
SERVICE_USER="{{service_user}}"
SERVICE_GROUP="{{service_group}}"
SERVICE_PIDFILE="/var/run/${SERVICE_NAME}.pid"
SERVICE_LOGFILE="/var/log/${SERVICE_NAME}.log"

# Fonction de démarrage
start_service() {
    if [[ -f "$SERVICE_PIDFILE" ]]; then
        echo "Service déjà démarré (PID: $(cat "$SERVICE_PIDFILE"))"
        return 1
    fi
    
    echo "Démarrage du service: $SERVICE_NAME"
    
    # Démarrage en arrière-plan
    nohup {{service_command}} >> "$SERVICE_LOGFILE" 2>&1 &
    local pid=$!
    
    # Attente de stabilité
    sleep 2
    
    if kill -0 "$pid" 2>/dev/null; then
        echo "$pid" > "$SERVICE_PIDFILE"
        echo "✓ Service démarré (PID: $pid)"
        return 0
    else
        echo "❌ Échec du démarrage du service"
        return 1
    fi
}

# Fonction d\'arrêt
stop_service() {
    if [[ ! -f "$SERVICE_PIDFILE" ]]; then
        echo "Service non démarré"
        return 1
    fi
    
    local pid=$(cat "$SERVICE_PIDFILE")
    echo "Arrêt du service: $SERVICE_NAME (PID: $pid)"
    
    kill "$pid"
    
    # Attente de l\'arrêt
    local count=0
    while kill -0 "$pid" 2>/dev/null && [[ $count -lt 10 ]]; do
        sleep 1
        ((count++))
    done
    
    if kill -0 "$pid" 2>/dev/null; then
        echo "Forçage de l\'arrêt..."
        kill -9 "$pid"
    fi
    
    rm -f "$SERVICE_PIDFILE"
    echo "✓ Service arrêté"
}

# Fonction de statut
status_service() {
    if [[ -f "$SERVICE_PIDFILE" ]]; then
        local pid=$(cat "$SERVICE_PIDFILE")
        if kill -0 "$pid" 2>/dev/null; then
            echo "Service en cours d\'exécution (PID: $pid)"
            return 0
        else
            echo "Service mort (PID file exists but process not running)"
            return 1
        fi
    else
        echo "Service arrêté"
        return 1
    fi
}

# Fonction de redémarrage
restart_service() {
    stop_service
    sleep 1
    start_service
}

# Point d\'entrée
main() {
    case "${1:-status}" in
        start)
            start_service
            ;;
        stop)
            stop_service
            ;;
        restart)
            restart_service
            ;;
        status)
            status_service
            ;;
        *)
            echo "Usage: $0 {start|stop|restart|status}"
            exit 1
            ;;
    esac
}

# Exécution
if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
    {{error_handling}}
    main "$@"
fi
' "Service système avec gestion PID"
    
    # Template pour un script de monitoring
    $engine.define_intelligent_template "monitoring_script" '
#!/bin/bash
# Script de monitoring généré automatiquement

MONITOR_TARGET="{{monitor_target}}"
MONITOR_INTERVAL="{{interval}}"
MONITOR_METRICS="{{metrics}}"
ALERT_THRESHOLD="{{alert_threshold}}"
ALERT_COMMAND="{{alert_command}}"

# Fonction de collecte de métriques
collect_metrics() {
    local timestamp=$(date +%s)
    
    echo "Collecte de métriques: $timestamp"
    
    # Métriques CPU
    if echo "{{metrics}}" | grep -q "cpu"; then
        local cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/")
        echo "cpu_usage $timestamp $cpu_usage" >> "{{data_file}}"
    fi
    
    # Métriques mémoire
    if echo "{{metrics}}" | grep -q "memory"; then
        local mem_usage=$(free | grep Mem | awk '\''{printf "%.1f", $3/$2 * 100.0}'\'')
        echo "memory_usage $timestamp $mem_usage" >> "{{data_file}}"
    fi
    
    # Métriques disque
    if echo "{{metrics}}" | grep -q "disk"; then
        local disk_usage=$(df / | tail -1 | awk '\''{print $5}'\'' | sed '\''s/%//'\'')
        echo "disk_usage $timestamp $disk_usage" >> "{{data_file}}"
    fi
    
    # Métriques réseau
    if echo "{{metrics}}" | grep -q "network"; then
        local rx_bytes=$(cat /proc/net/dev | grep -E "^[[:space:]]*eth0|^[[:space:]]*enp" | awk '\''{print $2}'\'')
        local tx_bytes=$(cat /proc/net/dev | grep -E "^[[:space:]]*eth0|^[[:space:]]*enp" | awk '\''{print $10}'\'')
        echo "network_rx $timestamp $rx_bytes" >> "{{data_file}}"
        echo "network_tx $timestamp $tx_bytes" >> "{{data_file}}"
    fi
}

# Fonction de vérification d\'alertes
check_alerts() {
    local metric_file="{{data_file}}"
    
    if [[ ! -f "$metric_file" ]]; then
        return
    fi
    
    # Vérification des seuils
    while read -r metric_name timestamp value; do
        case "$metric_name" in
            cpu_usage|memory_usage|disk_usage)
                if (( $(echo "$value > {{alert_threshold}}" | bc -l) )); then
                    echo "ALERTE: $metric_name = $value% (seuil: {{alert_threshold}}%)"
                    {{alert_command}}
                fi
                ;;
        esac
    done < "$metric_file"
}

# Fonction de reporting
generate_report() {
    local report_file="{{report_file}}"
    
    {
        echo "RAPPORT DE MONITORING"
        echo "====================="
        echo "Généré le: $(date)"
        echo "Cible: {{monitor_target}}"
        echo
        
        if [[ -f "{{data_file}}" ]]; then
            echo "DERNIÈRES MÉTRIQUES"
            echo "==================="
            tail -10 "{{data_file}}" | while read -r metric timestamp value; do
                echo "$(date -d "@$timestamp" '\''+%H:%M:%S'\''): $metric = $value"
            done
        fi
        
        echo
        echo "STATISTIQUES"
        echo "============"
        
        if [[ -f "{{data_file}}" ]]; then
            local total_metrics=$(wc -l < "{{data_file}}")
            echo "Total métriques collectées: $total_metrics"
        fi
    } > "$report_file"
    
    echo "Rapport généré: $report_file"
}

# Point d\'entrée
main() {
    case "${1:-monitor}" in
        monitor)
            while true; do
                collect_metrics
                check_alerts
                sleep {{interval}}
            done
            ;;
        report)
            generate_report
            ;;
        once)
            collect_metrics
            check_alerts
            ;;
        *)
            echo "Usage: $0 {monitor|report|once}"
            exit 1
            ;;
    esac
}

# Exécution
if [[ "${BASH_SOURCE[0]}" == "$0" ]]; then
    {{error_handling}}
    main "$@"
fi
' "Script de monitoring système"
}

# Démonstration du moteur de templates intelligents
echo "--- Moteur de templates intelligents ---"

IntelligentTemplateEngine "template_engine"

# Définition des templates
define_smart_templates "template_engine"

echo
echo "--- Génération depuis templates ---"

# Génération d'un script de sauvegarde
template_engine.generate_from_template "backup_script" "/tmp/backup_generated.sh" \
    "source_dir" "/home/user/documents" \
    "dest_dir" "/tmp/backups" \
    "compression" "gzip" \
    "retention_days" "30" \
    "extension" "tar.gz" \
    "restore_default" "/tmp/restore" \
    "error_handling" "set -euo pipefail"

echo
echo "Script généré:"
head -20 /tmp/backup_generated.sh

echo
echo "--- Template conditionnel ---"

# Définition d'un template conditionnel
template_engine.conditional_template "smart_backup" \
    '[[ -d "/home/user" ]]' \
    "backup_script"

echo
echo "Génération conditionnelle:"
template_engine.generate_conditional "smart_backup" "/tmp/smart_backup.sh" \
    "source_dir" "/home/user" \
    "dest_dir" "/tmp/smart_backups" \
    "compression" "bzip2" \
    "retention_days" "7" \
    "extension" "tar.bz2" \
    "restore_default" "/tmp/restore" \
    "error_handling" "set -e"

echo
echo "--- Template adaptatif ---"

# Définition d'un template adaptatif
template_engine.adaptive_template "adaptive_service" "system_service" "context_aware"

echo
echo "Génération adaptative pour développement:"
template_engine.generate_adaptive "adaptive_service" "/tmp/dev_service.sh" "development" \
    "service_name" "my_app_dev" \
    "service_user" "developer" \
    "service_group" "developers" \
    "service_command" "python3 -m http.server 8000" \
    "error_handling" "set -x"

echo
echo "Génération adaptative pour production:"
template_engine.generate_adaptive "adaptive_service" "/tmp/prod_service.sh" "production" \
    "service_name" "my_app_prod" \
    "service_user" "www-data" \
    "service_group" "www-data" \
    "service_command" "gunicorn --bind 0.0.0.0:8000 myapp:app" \
    "error_handling" "set -euo pipefail"

echo
echo "--- Validation de templates ---"
template_engine.validate_template "backup_script"

echo
echo "--- Amélioration automatique ---"
template_engine.improve_templates

echo
echo "--- Génération de rapport ---"
template_engine.generate_template_report

# Nettoyage
rm -f /tmp/backup_generated.sh /tmp/smart_backup.sh /tmp/dev_service.sh /tmp/prod_service.sh template_report_*.txt
```

## Conclusion : Le code comme organisme vivant

La métaprogrammation en Bash transforme l'écriture de code d'un acte statique en un processus dynamique où les programmes peuvent s'analyser, s'étendre, et évoluer. Cette approche permet de créer des systèmes qui s'adaptent à leur environnement, apprennent de leur expérience, et se reproduisent pour former des écosystèmes de code auto-gérants.

**Points clés à retenir :**

1. **Exécution dynamique sécurisée** : Frameworks d'évaluation de code avec sandboxing, validation, et profiling
2. **Code auto-modifiant** : Systèmes capables de s'analyser et de se modifier selon des règles d'évolution
3. **Templates intelligents** : Moteurs de génération de code avec héritage, conditions, et adaptation contextuelle

Dans le prochain chapitre, nous conclurons notre périple à travers l'univers Bash avec une réflexion sur les patterns avancés de programmation et l'avenir de l'automatisation en shell.

---

**Exercice pratique :** Créez un système métaprogrammé complet incluant :
- Évaluateur sécurisé avec validation et profiling
- Générateur auto-modifiant avec évolution adaptative
- Moteur de templates avec héritage et adaptation contextuelle

**Réflexion :** Imaginez un monde où vos scripts Bash deviennent conscients d'eux-mêmes, capables de détecter leurs propres bogues, de s'optimiser automatiquement, et de collaborer avec d'autres programmes pour résoudre des problèmes complexes. Comment concevriez-vous l'architecture de cette intelligence artificielle distribuée basée sur des scripts shell ?
