# Chapitre 80 - Tests et validation avancée

> "Le code sans tests est comme une voiture sans freins : vous espérez qu'elle s'arrête quand vous le voulez." - Anonyme

## Introduction : De l'artisanat au génie logiciel

Imaginez-vous en train de construire une fusée spatiale. Vous pourriez créer les moteurs les plus puissants, les circuits les plus sophistiqués, mais sans tests rigoureux, le premier vol se terminerait en catastrophe. Les scripts Bash, bien que plus terre-à-terre, méritent la même rigueur. Dans ce chapitre, nous allons transformer vos scripts de simples outils en systèmes fiables et testables.

Nous explorerons les stratégies de test avancées : des tests unitaires pour valider chaque fonction aux tests d'intégration qui simulent les pires conditions possibles. Vous apprendrez à créer des frameworks de test, à simuler les échecs, et à garantir que vos scripts survivent aux situations extrêmes.

## Section 1 : Principes fondamentaux des tests en Bash

### 1.1 Pourquoi tester les scripts Bash ?

Contrairement à une idée reçue, les scripts Bash ne sont pas "trop simples" pour être testés. Au contraire, leur utilisation dans des environnements critiques (déploiement, sauvegarde, monitoring) exige une fiabilité absolue.

**Analogies pédagogiques :**
- Un script non testé est comme un parachute que vous n'avez jamais ouvert
- Les tests sont l'assurance qualité de votre code, comme les crash tests pour les voitures

### 1.2 Les différents types de tests

1. **Tests unitaires** : Validation des fonctions individuelles
2. **Tests d'intégration** : Validation des interactions entre composants
3. **Tests de régression** : Assurance que les corrections n'introduisent pas de bugs
4. **Tests de performance** : Validation des contraintes temporelles
5. **Tests de robustesse** : Validation face aux conditions extrêmes

### 1.3 Le framework de test minimal

Commençons par créer un framework de test simple et extensible :

```bash
#!/bin/bash

# Framework de test minimal pour Bash
TEST_PASSED=0
TEST_FAILED=0

assert_equals() {
    local expected="$1"
    local actual="$2"
    local message="${3:-Test}"
    
    if [[ "$expected" == "$actual" ]]; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        echo "  Attendu: '$expected'"
        echo "  Obtenu:  '$actual'"
        ((TEST_FAILED++))
        return 1
    fi
}

assert_true() {
    local condition="$1"
    local message="${2:-Condition}"
    
    if eval "$condition"; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        ((TEST_FAILED++))
        return 1
    fi
}

assert_false() {
    local condition="$1"
    local message="${2:-Condition}"
    
    if ! eval "$condition"; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        ((TEST_FAILED++))
        return 1
    fi
}

print_test_results() {
    local total=$((TEST_PASSED + TEST_FAILED))
    echo
    echo "=== RÉSULTATS DES TESTS ==="
    echo "Total: $total"
    echo "Réussis: $TEST_PASSED"
    echo "Échoués: $TEST_FAILED"
    
    if (( TEST_FAILED > 0 )); then
        echo "❌ ÉCHEC GLOBAL"
        return 1
    else
        echo "✅ SUCCÈS GLOBAL"
        return 0
    fi
}

# Configuration pour les tests
set -euo pipefail  # Arrêt strict sur erreur
```

## Section 2 : Tests unitaires pour les fonctions

### 2.1 Fonctions assert_* spécialisées

Créons des fonctions d'assertion spécialisées pour les cas d'usage courants :

```bash
#!/bin/bash

# Assertions spécialisées pour les tests unitaires

assert_fails() {
    local cmd="$1"
    local message="${2:-Commande devrait échouer}"
    
    if ! eval "$cmd" 2>/dev/null; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        ((TEST_FAILED++))
        return 1
    fi
}

assert_succeeds() {
    local cmd="$1"
    local message="${2:-Commande devrait réussir}"
    
    if eval "$cmd" 2>/dev/null; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        ((TEST_FAILED++))
        return 1
    fi
}

assert_output_contains() {
    local cmd="$1"
    local expected="$2"
    local message="${3:-Sortie devrait contenir '$expected'}"
    
    local output
    output=$(eval "$cmd" 2>&1)
    
    if [[ "$output" == *"$expected"* ]]; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        echo "  Sortie obtenue: '$output'"
        ((TEST_FAILED++))
        return 1
    fi
}

assert_output_equals() {
    local cmd="$1"
    local expected="$2"
    local message="${3:-Sortie devrait être '$expected'}"
    
    local output
    output=$(eval "$cmd" 2>&1)
    
    if [[ "$output" == "$expected" ]]; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        echo "  Attendu: '$expected'"
        echo "  Obtenu:  '$output'"
        ((TEST_FAILED++))
        return 1
    fi
}

assert_file_exists() {
    local file="$1"
    local message="${2:-Fichier devrait exister: $file}"
    
    if [[ -f "$file" ]]; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        ((TEST_FAILED++))
        return 1
    fi
}

assert_file_not_exists() {
    local file="$1"
    local message="${2:-Fichier ne devrait pas exister: $file}"
    
    if [[ ! -f "$file" ]]; then
        echo "✓ $message"
        ((TEST_PASSED++))
        return 0
    else
        echo "✗ $message"
        ((TEST_FAILED++))
        return 1
    fi
}
```

### 2.2 Test d'une fonction simple

Appliquons nos assertions à une fonction réelle :

```bash
#!/bin/bash

# Fonction à tester
calculer_somme() {
    local a="$1"
    local b="$2"
    
    # Validation des paramètres
    if ! [[ "$a" =~ ^[0-9]+$ ]] || ! [[ "$b" =~ ^[0-9]+$ ]]; then
        echo "Erreur: paramètres doivent être des nombres" >&2
        return 1
    fi
    
    echo $((a + b))
}

# Tests unitaires
test_calculer_somme() {
    echo "=== Tests de calculer_somme ==="
    
    # Test cas normal
    assert_output_equals "calculer_somme 2 3" "5" "Somme 2+3=5"
    
    # Test paramètres invalides
    assert_fails "calculer_somme a b" "Devrait échouer avec paramètres non numériques"
    
    # Test valeurs limites
    assert_output_equals "calculer_somme 0 0" "0" "Somme 0+0=0"
    assert_output_equals "calculer_somme 100 200" "300" "Somme 100+200=300"
    
    # Test un seul paramètre (devrait échouer)
    assert_fails "calculer_somme 5" "Devrait échouer avec un seul paramètre"
}

# Exécution des tests
test_calculer_somme
```

### 2.3 Tests avec mocks et stubs

Pour isoler les fonctions, créez des mocks :

```bash
#!/bin/bash

# Fonction qui utilise des commandes externes (difficile à tester)
backup_file() {
    local source="$1"
    local dest="$2"
    
    if [[ ! -f "$source" ]]; then
        echo "Erreur: fichier source inexistant" >&2
        return 1
    fi
    
    # Simulation de cp avec vérification
    if cp "$source" "$dest" 2>/dev/null; then
        echo "Sauvegarde réussie: $source -> $dest"
        return 0
    else
        echo "Erreur lors de la sauvegarde" >&2
        return 1
    fi
}

# Version mockable pour les tests
backup_file_mockable() {
    local source="$1"
    local dest="$2"
    
    # Utilisation d'une fonction mockable
    if mock_cp "$source" "$dest"; then
        echo "Sauvegarde réussie: $source -> $dest"
        return 0
    else
        echo "Erreur lors de la sauvegarde" >&2
        return 1
    fi
}

# Mock de cp pour les tests
mock_cp() {
    local source="$1"
    local dest="$2"
    
    # Simulation de différents scénarios
    case "$MOCK_CP_SCENARIO" in
        success)
            echo "Mock: cp $source $dest (succès)" >&2
            return 0
            ;;
        failure)
            echo "Mock: cp $source $dest (échec)" >&2
            return 1
            ;;
        permission_denied)
            echo "Mock: cp: permission denied" >&2
            return 1
            ;;
        *)
            # Comportement réel
            cp "$source" "$dest"
            ;;
    esac
}

# Tests avec mocks
test_backup_with_mocks() {
    echo "=== Tests de backup_file avec mocks ==="
    
    local test_file="/tmp/test_backup.txt"
    local backup_file="/tmp/test_backup.bak"
    
    # Setup
    echo "contenu test" > "$test_file"
    rm -f "$backup_file"
    
    # Test succès
    MOCK_CP_SCENARIO="success"
    assert_succeeds "backup_file_mockable '$test_file' '$backup_file'" "Sauvegarde devrait réussir"
    
    # Test échec
    MOCK_CP_SCENARIO="failure"
    assert_fails "backup_file_mockable '$test_file' '$backup_file'" "Sauvegarde devrait échouer"
    
    # Test permission denied
    MOCK_CP_SCENARIO="permission_denied"
    assert_fails "backup_file_mockable '$test_file' '$backup_file'" "Sauvegarde devrait échouer (permission)"
    assert_output_contains "backup_file_mockable '$test_file' '$backup_file'" "permission denied" "Message d'erreur approprié"
    
    # Cleanup
    rm -f "$test_file" "$backup_file"
    unset MOCK_CP_SCENARIO
}

test_backup_with_mocks
```

## Section 3 : Tests d'intégration et simulation d'erreurs

### 3.1 Simulation des pannes réseau

Les tests d'intégration doivent simuler les conditions réelles, y compris les pannes :

```bash
#!/bin/bash

# Fonction qui dépend du réseau
telecharger_fichier() {
    local url="$1"
    local dest="$2"
    
    if ! curl -s --max-time 10 "$url" -o "$dest"; then
        echo "Erreur: impossible de télécharger $url" >&2
        return 1
    fi
    
    echo "Téléchargement réussi: $url -> $dest"
}

# Tests d'intégration avec simulation de pannes réseau
test_telechargement_integration() {
    echo "=== Tests d'intégration téléchargement ==="
    
    local dest_file="/tmp/test_download.txt"
    rm -f "$dest_file"
    
    # Test avec URL valide (si réseau disponible)
    if ping -c 1 -W 1 8.8.8.8 >/dev/null 2>&1; then
        assert_succeeds "telecharger_fichier 'https://httpbin.org/get' '$dest_file'" "Téléchargement devrait réussir"
        assert_file_exists "$dest_file" "Fichier devrait être créé"
    else
        echo "! Test réseau ignoré (pas de connectivité)"
    fi
    
    # Test avec URL invalide
    assert_fails "telecharger_fichier 'https://url-invalide-qui-nexiste-pas.com/file.txt' '$dest_file'" "Téléchargement d'URL invalide devrait échouer"
    
    # Test timeout (simulation)
    # Note: curl --max-time devrait gérer cela
    
    rm -f "$dest_file"
}

# Simulation de pannes réseau avec iptables (nécessite sudo)
simulate_network_failure() {
    echo "=== Simulation de panne réseau ==="
    
    # Sauvegarde des règles
    local rules_backup="/tmp/iptables_backup.rules"
    sudo iptables-save > "$rules_backup"
    
    # Blocage des connexions sortantes
    sudo iptables -P OUTPUT DROP
    sudo iptables -A OUTPUT -p tcp --dport 80 -j ACCEPT  # HTTP seulement
    
    echo "Réseau simulé comme coupé (sauf HTTP)"
    echo "Appuyez sur Entrée pour restaurer..."
    read
    
    # Restauration
    sudo iptables-restore < "$rules_backup"
    rm -f "$rules_backup"
    echo "Réseau restauré"
}

test_telechargement_integration
# simulate_network_failure  # À utiliser avec précaution !
```

### 3.2 Simulation de disque plein

Testez comment votre script réagit quand l'espace disque vient à manquer :

```bash
#!/bin/bash

# Fonction qui écrit des fichiers
creer_fichiers_logs() {
    local dir="$1"
    local count="${2:-10}"
    
    mkdir -p "$dir" || return 1
    
    for i in $(seq 1 "$count"); do
        local filename="$dir/log_$i.txt"
        echo "Log entry $i - $(date)" > "$filename"
        
        # Simulation d'écriture volumineuse
        dd if=/dev/zero of="$filename.tmp" bs=1M count=1 2>/dev/null || {
            echo "Erreur: disque plein lors de l'écriture $filename" >&2
            rm -f "$filename.tmp"
            return 1
        }
        rm -f "$filename.tmp"
    done
    
    echo "$count fichiers de log créés dans $dir"
}

# Tests avec simulation de disque plein
test_disk_full_simulation() {
    echo "=== Tests simulation disque plein ==="
    
    local test_dir="/tmp/test_logs_full"
    rm -rf "$test_dir"
    
    # Test normal
    assert_succeeds "creer_fichiers_logs '$test_dir' 3" "Création normale devrait réussir"
    
    # Simulation de disque plein avec un filesystem limité
    # Méthode 1: Utiliser un filesystem en RAM limité
    if command -v mount >/dev/null && command -v fallocate >/dev/null; then
        local ramdisk="/tmp/ramdisk_test"
        local ramdisk_size="5M"  # 5MB seulement
        
        mkdir -p "$ramdisk"
        sudo mount -t tmpfs -o size="$ramdisk_size" tmpfs "$ramdisk" 2>/dev/null && {
            echo "Test avec ramdisk de $ramdisk_size"
            
            # Remplir presque tout l'espace
            dd if=/dev/zero of="$ramdisk/bigfile" bs=1M count=4 2>/dev/null
            
            # Test avec espace limité
            if ! creer_fichiers_logs "$ramdisk/logs" 10 2>/dev/null; then
                echo "✓ Gestion correcte du disque plein"
                ((TEST_PASSED++))
            else
                echo "✗ Le script devrait échouer sur disque plein"
                ((TEST_FAILED++))
            fi
            
            sudo umount "$ramdisk"
        } || echo "! Test ramdisk ignoré (permissions ou outils manquants)"
    fi
    
    rm -rf "$test_dir"
}

test_disk_full_simulation
```

### 3.3 Simulation de problèmes de permissions

Testez les scénarios de permissions insuffisantes :

```bash
#!/bin/bash

# Fonction qui nécessite des permissions spécifiques
installer_logiciel() {
    local target_dir="$1"
    local user="${2:-$USER}"
    
    # Création du répertoire cible
    if ! sudo mkdir -p "$target_dir"; then
        echo "Erreur: impossible de créer $target_dir" >&2
        return 1
    fi
    
    # Changement de propriétaire
    if ! sudo chown "$user:$user" "$target_dir"; then
        echo "Erreur: impossible de changer propriétaire de $target_dir" >&2
        return 1
    fi
    
    # Création d'un fichier de configuration
    local config_file="$target_dir/config.txt"
    cat > "$config_file" << EOF
# Configuration générée automatiquement
date_installation=$(date)
utilisateur=$user
EOF
    
    if [[ ! -w "$config_file" ]]; then
        echo "Erreur: fichier config non accessible en écriture" >&2
        return 1
    fi
    
    echo "Installation réussie dans $target_dir"
}

# Tests de permissions
test_permissions_scenarios() {
    echo "=== Tests scénarios de permissions ==="
    
    local test_dir="/tmp/test_install_readonly"
    rm -rf "$test_dir"
    
    # Test normal (nécessite sudo)
    if sudo -n true 2>/dev/null; then
        assert_succeeds "installer_logiciel '$test_dir' '$USER'" "Installation normale devrait réussir"
        assert_file_exists "$test_dir/config.txt" "Fichier de config devrait être créé"
    else
        echo "! Tests sudo ignorés (pas de sudo disponible ou nécessite mot de passe)"
    fi
    
    # Simulation de permissions insuffisantes
    # Créer un répertoire en lecture seule
    mkdir -p "$test_dir"
    chmod 444 "$test_dir"  # Lecture seule pour tous
    
    # Test sans sudo (devrait échouer)
    assert_fails "installer_logiciel '$test_dir' '$USER'" "Installation sans sudo devrait échouer"
    
    # Cleanup
    chmod 755 "$test_dir"  # Restaurer permissions pour suppression
    rm -rf "$test_dir"
}

test_permissions_scenarios
```

### 3.4 Tests de charge et performance

Évaluez les performances sous charge :

```bash
#!/bin/bash

# Fonction à tester sous charge
traiter_fichiers() {
    local dir="$1"
    local count="${2:-100}"
    
    for i in $(seq 1 "$count"); do
        local file="$dir/fichier_$i.txt"
        echo "Contenu du fichier $i" > "$file"
        
        # Simulation de traitement
        sleep 0.01
        
        # Calcul de hash (CPU intensif)
        sha256sum "$file" >/dev/null
    done
    
    echo "$count fichiers traités"
}

# Tests de performance
test_performance_under_load() {
    echo "=== Tests de performance ==="
    
    local test_dir="/tmp/test_perf"
    rm -rf "$test_dir"
    mkdir -p "$test_dir"
    
    # Test avec charge normale
    local start_time=$(date +%s.%N)
    assert_succeeds "traiter_fichiers '$test_dir' 10" "Traitement 10 fichiers devrait réussir"
    local end_time=$(date +%s.%N)
    local duration=$(echo "$end_time - $start_time" | bc)
    
    echo "Temps pour 10 fichiers: ${duration}s"
    
    if (( $(echo "$duration < 1.0" | bc -l) )); then
        echo "✓ Performance acceptable (< 1s pour 10 fichiers)"
        ((TEST_PASSED++))
    else
        echo "✗ Performance insuffisante (> 1s pour 10 fichiers)"
        ((TEST_FAILED++))
    fi
    
    # Test avec charge élevée
    start_time=$(date +%s.%N)
    assert_succeeds "traiter_fichiers '$test_dir' 50" "Traitement 50 fichiers devrait réussir"
    end_time=$(date +%s.%N)
    duration=$(echo "$end_time - $start_time" | bc)
    
    echo "Temps pour 50 fichiers: ${duration}s"
    
    # Vérifier que ce n'est pas linéairement plus lent
    local expected_max=$(echo "5.0" | bc)  # 5x plus de fichiers = max 5x plus de temps
    if (( $(echo "$duration < $expected_max" | bc -l) )); then
        echo "✓ Scaling acceptable"
        ((TEST_PASSED++))
    else
        echo "✗ Problème de scaling"
        ((TEST_FAILED++))
    fi
    
    rm -rf "$test_dir"
}

test_performance_under_load
```

## Section 4 : Framework de test avancé

### 4.1 Framework avec setup/teardown

Créez un framework complet avec configuration et nettoyage :

```bash
#!/bin/bash

# Framework de test avancé avec setup/teardown
TEST_SUITE_NAME=""
TEST_CURRENT=""
TEST_SETUP=""
TEST_TEARDOWN=""

suite() {
    TEST_SUITE_NAME="$1"
    echo "=== SUITE DE TESTS: $TEST_SUITE_NAME ==="
}

setup() {
    TEST_SETUP="$1"
}

teardown() {
    TEST_TEARDOWN="$1"
}

test() {
    local test_name="$1"
    local test_function="$2"
    
    TEST_CURRENT="$test_name"
    echo "--- Test: $test_name ---"
    
    # Setup
    if [[ -n "$TEST_SETUP" ]]; then
        if ! eval "$TEST_SETUP"; then
            echo "✗ Setup échoué pour $test_name"
            ((TEST_FAILED++))
            return 1
        fi
    fi
    
    # Exécution du test
    if eval "$test_function"; then
        echo "✓ $test_name réussi"
        ((TEST_PASSED++))
    else
        echo "✗ $test_name échoué"
        ((TEST_FAILED++))
    fi
    
    # Teardown
    if [[ -n "$TEST_TEARDOWN" ]]; then
        if ! eval "$TEST_TEARDOWN"; then
            echo "! Teardown échoué pour $test_name" >&2
        fi
    fi
}

# Utilisation du framework
suite "Tests de calculatrice"

setup "
    mkdir -p /tmp/test_calc
    echo 'Test setup completed' > /tmp/test_calc/setup.log
"

teardown "
    rm -rf /tmp/test_calc
    echo 'Test teardown completed'
"

# Fonction à tester
addition() {
    echo $(( $1 + $2 ))
}

test "Addition simple" "
    result=\$(addition 2 3)
    assert_equals '5' \"\$result\" '2+3=5'
"

test "Addition avec zéro" "
    result=\$(addition 0 5)
    assert_equals '5' \"\$result\" '0+5=5'
"

test "Addition négative" "
    result=\$(addition -1 1)
    assert_equals '0' \"\$result\" '-1+1=0'
"

print_test_results
```

### 4.2 Tests parallèles et reporting

Pour les gros projets, exécutez les tests en parallèle :

```bash
#!/bin/bash

# Framework avec exécution parallèle
run_tests_parallel() {
    local max_jobs="${1:-4}"
    local test_functions=("${@:2}")
    local pids=()
    local results=()
    
    echo "Exécution de ${#test_functions[@]} tests avec $max_jobs jobs max"
    
    for test_func in "${test_functions[@]}"; do
        # Attendre si trop de jobs
        while (( ${#pids[@]} >= max_jobs )); do
            for i in "${!pids[@]}"; do
                if ! kill -0 "${pids[$i]}" 2>/dev/null; then
                    wait "${pids[$i]}"
                    results[$i]=$?
                    unset pids[$i]
                fi
            done
            pids=("${pids[@]}")  # Reindexer
            sleep 0.1
        done
        
        # Lancer le test en arrière-plan
        (
            echo "Démarrage: $test_func"
            if eval "$test_func"; then
                echo "✓ $test_func terminé avec succès"
                exit 0
            else
                echo "✗ $test_func terminé avec échec"
                exit 1
            fi
        ) &
        pids+=($!)
    done
    
    # Attendre la fin de tous les tests
    for pid in "${pids[@]}"; do
        wait "$pid"
        results+=($?)
    done
    
    # Compter les résultats
    local success=0 failure=0
    for result in "${results[@]}"; do
        if (( result == 0 )); then ((success++)); else ((failure++)); fi
    done
    
    echo "=== RÉSULTATS PARALLÈLES ==="
    echo "Succès: $success, Échecs: $failure"
    
    return $(( failure > 0 ))
}

# Tests individuels
test_arithmetic() {
    assert_equals "5" "$(echo $((2+3)))" "Addition basique"
    assert_equals "6" "$(echo $((2*3)))" "Multiplication"
}

test_strings() {
    local str="hello world"
    assert_output_contains "echo '$str'" "hello" "Contient hello"
    assert_output_contains "echo '$str'" "world" "Contient world"
}

test_files() {
    local test_file="/tmp/parallel_test.txt"
    echo "test content" > "$test_file"
    assert_file_exists "$test_file" "Fichier créé"
    rm -f "$test_file"
}

# Exécution parallèle
run_tests_parallel 2 test_arithmetic test_strings test_files
```

### 4.3 Intégration continue basique

Créez un script de CI/CD simple pour vos projets Bash :

```bash
#!/bin/bash

# Script de CI/CD basique pour projets Bash
run_ci_pipeline() {
    local project_dir="$1"
    local log_file="${2:-/tmp/ci_log.txt}"
    
    echo "=== PIPELINE CI/CD ===" > "$log_file"
    echo "Projet: $project_dir" >> "$log_file"
    echo "Date: $(date)" >> "$log_file"
    
    cd "$project_dir" || {
        echo "Erreur: répertoire $project_dir inaccessible" | tee -a "$log_file"
        return 1
    }
    
    # Étape 1: Validation syntaxique
    echo "Étape 1: Validation syntaxique" | tee -a "$log_file"
    local syntax_errors=0
    for script in *.sh; do
        if [[ -f "$script" ]]; then
            if bash -n "$script" 2>>"$log_file"; then
                echo "✓ $script: syntaxe OK" | tee -a "$log_file"
            else
                echo "✗ $script: erreurs de syntaxe" | tee -a "$log_file"
                ((syntax_errors++))
            fi
        fi
    done
    
    if (( syntax_errors > 0 )); then
        echo "❌ ÉCHEC: $syntax_errors erreurs de syntaxe" | tee -a "$log_file"
        return 1
    fi
    
    # Étape 2: Exécution des tests
    echo "Étape 2: Exécution des tests" | tee -a "$log_file"
    if [[ -f "test.sh" ]]; then
        if ./test.sh >>"$log_file" 2>&1; then
            echo "✓ Tests réussis" | tee -a "$log_file"
        else
            echo "✗ Tests échoués" | tee -a "$log_file"
            return 1
        fi
    else
        echo "! Aucun fichier test.sh trouvé" | tee -a "$log_file"
    fi
    
    # Étape 3: Vérification des dépendances
    echo "Étape 3: Vérification des dépendances" | tee -a "$log_file"
    local missing_deps=()
    for cmd in curl jq awk sed; do
        if ! command -v "$cmd" >/dev/null 2>&1; then
            missing_deps+=("$cmd")
        fi
    done
    
    if (( ${#missing_deps[@]} > 0 )); then
        echo "! Dépendances manquantes: ${missing_deps[*]}" | tee -a "$log_file"
    else
        echo "✓ Toutes les dépendances présentes" | tee -a "$log_file"
    fi
    
    # Étape 4: Analyse statique (si shellcheck disponible)
    if command -v shellcheck >/dev/null 2>&1; then
        echo "Étape 4: Analyse statique" | tee -a "$log_file"
        local sc_warnings=0
        for script in *.sh; do
            if [[ -f "$script" ]]; then
                if shellcheck_output=$(shellcheck "$script" 2>&1); then
                    echo "✓ $script: shellcheck OK" | tee -a "$log_file"
                else
                    echo "! $script: avertissements shellcheck:" | tee -a "$log_file"
                    echo "$shellcheck_output" | tee -a "$log_file"
                    ((sc_warnings++))
                fi
            fi
        done
    fi
    
    echo "=== PIPELINE TERMINÉE ===" | tee -a "$log_file"
    return 0
}

# Utilisation
if [[ "${1:-}" == "--ci" ]]; then
    run_ci_pipeline "$(pwd)" "${2:-}"
fi
```

## Section 5 : Bonnes pratiques et pièges à éviter

### 5.1 Éviter les faux positifs

Assurez-vous que vos tests détectent vraiment les vrais problèmes :

```bash
#!/bin/bash

# Exemple de test problématique (faux positif)
test_problematique() {
    # Ce test peut réussir même si la fonction est cassée
    ma_fonction 2>&1 | grep -q "OK"  # Recherche seulement "OK"
}

# Version corrigée
test_corrige() {
    local output
    local status
    
    output=$(ma_fonction)
    status=$?
    
    # Vérifier à la fois le statut et le contenu
    assert_equals "0" "$status" "Fonction devrait réussir"
    assert_output_contains "echo '$output'" "OK" "Sortie devrait contenir OK"
}
```

### 5.2 Tests déterministes

Évitez les tests qui dépendent de facteurs externes :

```bash
#!/bin/bash

# MAUVAIS : test non déterministe
test_date() {
    local current_date=$(date +%Y)
    assert_equals "2024" "$current_date" "Année devrait être 2024"
    # Ce test échouera en 2025 !
}

# BON : test déterministe
test_date_format() {
    local date_str=$(date +%Y-%m-%d)
    
    # Vérifier le format seulement
    if [[ "$date_str" =~ ^[0-9]{4}-[0-9]{2}-[0-9]{2}$ ]]; then
        echo "✓ Format de date correct"
        ((TEST_PASSED++))
    else
        echo "✗ Format de date incorrect: $date_str"
        ((TEST_FAILED++))
    fi
}

# BON : test avec données contrôlées
test_with_mock_date() {
    # Mock de date pour les tests
    date() {
        echo "2024-01-01"
    }
    
    local date_str=$(date +%Y-%m-%d)
    assert_equals "2024-01-01" "$date_str" "Date mockée correcte"
}
```

### 5.3 Organisation des tests

Structurez vos tests pour une maintenance facile :

```bash
#!/bin/bash

# Organisation recommandée des tests
# tests/
# ├── unit/           # Tests unitaires
# │   ├── functions.sh
# │   └── variables.sh
# ├── integration/    # Tests d'intégration
# │   ├── network.sh
# │   └── filesystem.sh
# ├── performance/    # Tests de performance
# │   └── benchmarks.sh
# └── fixtures/       # Données de test
#     ├── sample_files/
#     └── mock_data.sh

# Fonction d'organisation
run_test_suite() {
    local suite_name="$1"
    local suite_dir="$2"
    
    echo "=== SUITE: $suite_name ==="
    
    if [[ -d "$suite_dir" ]]; then
        for test_file in "$suite_dir"/*.sh; do
            if [[ -f "$test_file" ]]; then
                echo "--- Exécution: $(basename "$test_file") ---"
                if source "$test_file"; then
                    echo "✓ $(basename "$test_file") réussi"
                else
                    echo "✗ $(basename "$test_file") échoué"
                fi
            fi
        done
    else
        echo "! Répertoire de tests manquant: $suite_dir"
    fi
}

# Structure de répertoire simulée
run_test_suite "Tests unitaires" "tests/unit"
run_test_suite "Tests d'intégration" "tests/integration"
run_test_suite "Tests de performance" "tests/performance"
```

## Conclusion : La qualité comme standard

Les tests ne sont pas une option dans le développement moderne : ils sont le minimum acceptable pour tout script qui prétend à la fiabilité. Comme un pilote qui vérifie sa check-list avant le décollage, vos tests valident que chaque fonction, chaque interaction, chaque condition d'erreur est maîtrisée.

**Points clés à retenir :**

1. **Les assertions spécialisées** (`assert_fails`, `assert_succeeds`, `assert_output_contains`) permettent de tester précisément les comportements attendus
2. **Les mocks et stubs** isolent les fonctions pour des tests unitaires fiables
3. **La simulation d'erreurs** (réseau, disque, permissions) prépare vos scripts aux conditions réelles
4. **Les tests d'intégration** valident le comportement global du système
5. **Les frameworks organisés** avec setup/teardown facilitent la maintenance
6. **L'exécution parallèle** accélère les suites de tests volumineuses

Dans le chapitre suivant, nous explorerons les patterns de programmation avancés en Bash, pour écrire du code non seulement testé, mais aussi élégant et maintenable.

---

**Exercice pratique :** Créez une suite de tests complète pour un script de sauvegarde qui teste :
- La validation des paramètres
- Les cas d'erreur (fichier source inexistant, destination inaccessible)
- Les simulations de panne (disque plein, réseau coupé)
- Les tests de performance avec différentes tailles de fichiers
- L'intégration avec des outils externes (tar, rsync)

**Réflexion :** Comment adapteriez-vous ces techniques de test pour des scripts exécutés dans des environnements conteneurisés comme Docker ?
