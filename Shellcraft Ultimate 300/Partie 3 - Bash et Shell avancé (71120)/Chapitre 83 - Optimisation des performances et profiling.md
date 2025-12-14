# Chapitre 83 - Optimisation des performances et profiling

> "La performance absolue n'est pas le but. L'optimisation intelligente l'est." - Donald Knuth

## Introduction : Quand la vitesse rencontre l'efficacité

Imaginez-vous au volant d'une Formule 1 dans les rues étroites de la ville : puissance maximale mais parfaitement maîtrisée. L'optimisation des performances en Bash n'est pas qu'une question de vitesse brute, mais d'efficacité intelligente. Vos scripts doivent non seulement fonctionner rapidement, mais aussi utiliser au mieux les ressources disponibles, anticiper les goulots d'étranglement, et s'adapter aux conditions changeantes.

Dans ce chapitre, nous allons transformer vos scripts lents et gourmands en véritables bolides : du profiling précis aux optimisations algorithmiques, en passant par la parallélisation intelligente et la gestion fine des ressources.

## Section 1 : Les bases du profiling en Bash

### 1.1 Outils de mesure intégrés

Bash offre plusieurs outils natifs pour mesurer les performances :

```bash
#!/bin/bash

# Outils de mesure intégrés
echo "=== Outils de mesure intégrés ==="

# Fonction de profiling simple
profile_command() {
    local cmd="$1"
    local description="${2:-Commande}"
    
    echo "--- Profiling: $description ---"
    echo "Commande: $cmd"
    
    # Mesure du temps CPU et réel
    local start_time=$(date +%s.%N)
    local start_cpu=$(ps -o pcpu= $$ | tr -d ' ')
    
    eval "$cmd"
    local exit_code=$?
    
    local end_time=$(date +%s.%N)
    local end_cpu=$(ps -o pcpu= $$ | tr -d ' ')
    
    local real_time=$(echo "$end_time - $start_time" | bc)
    local cpu_time=$(echo "$end_cpu - $start_cpu" | bc)
    
    echo "Temps réel: ${real_time}s"
    echo "Temps CPU: ${cpu_time}s"
    echo "Code de sortie: $exit_code"
    echo
}

# Tests de profiling
echo "Test profiling de différentes opérations:"

# Opération CPU-intensive
profile_command "for i in {1..10000}; do echo \$i > /dev/null; done" "Boucle simple"

# Opération I/O-intensive  
profile_command "find /tmp -name '*' -type f 2>/dev/null | head -100" "Recherche de fichiers"

# Opération réseau (si disponible)
if ping -c 1 -W 1 8.8.8.8 >/dev/null 2>&1; then
    profile_command "curl -s https://httpbin.org/get > /dev/null" "Requête HTTP"
else
    echo "Réseau non disponible pour le test HTTP"
fi
```

### 1.2 Time et ses options avancées

L'outil `time` offre des mesures détaillées :

```bash
#!/bin/bash

# Utilisation avancée de time
echo "=== Utilisation avancée de time ==="

# Fonction de benchmarking avec time
benchmark_with_time() {
    local cmd="$1"
    local iterations="${2:-1}"
    local description="${3:-Test}"
    
    echo "=== Benchmark: $description ==="
    echo "Commande: $cmd"
    echo "Itérations: $iterations"
    
    # Exécution avec time détaillé
    local time_output
    time_output=$( (time for ((i=1; i<=iterations; i++)); do eval "$cmd" >/dev/null 2>&1; done) 2>&1 )
    
    echo "Résultats time:"
    echo "$time_output"
    
    # Calcul des métriques par itération
    local total_real=$(echo "$time_output" | grep "real" | awk '{print $2}' | sed 's/m/*60+/;s/s//;s/+//')
    local total_user=$(echo "$time_output" | grep "user" | awk '{print $2}' | sed 's/m/*60+/;s/s//;s/+//')
    local total_sys=$(echo "$time_output" | grep "sys" | awk '{print $2}' | sed 's/m/*60+/;s/s//;s/+//')
    
    if (( iterations > 1 )); then
        local avg_real=$(echo "scale=3; $total_real / $iterations" | bc 2>/dev/null || echo "N/A")
        local avg_user=$(echo "scale=3; $total_user / $iterations" | bc 2>/dev/null || echo "N/A")
        local avg_sys=$(echo "scale=3; $total_sys / $iterations" | bc 2>/dev/null || echo "N/A")
        
        echo "Moyenne par itération:"
        echo "  Réel: ${avg_real}s"
        echo "  User: ${avg_user}s"
        echo "  Sys: ${avg_sys}s"
    fi
    
    echo
}

# Benchmarks comparatifs
echo "Comparaison d'algorithmes de tri:"

# Création de données de test
test_data=$(seq 1 100 | sort -R | tr '\n' ' ')

# Test avec sort
benchmark_with_time "echo '$test_data' | tr ' ' '\n' | sort -n" 5 "Tri avec sort"

# Test avec awk
benchmark_with_time "
echo '$test_data' | tr ' ' '\n' | awk '
BEGIN { split(\"\", arr) }
{ arr[NR] = \$1 }
END {
    for (i = 1; i <= NR; i++) {
        for (j = i + 1; j <= NR; j++) {
            if (arr[i] > arr[j]) {
                temp = arr[i]
                arr[i] = arr[j]
                arr[j] = temp
            }
        }
    }
    for (i = 1; i <= NR; i++) print arr[i]
}
'" 3 "Tri avec awk (bubble sort)"
```

### 1.3 Profiling avec strace et ltrace

Analyse détaillée des appels système :

```bash
#!/bin/bash

# Profiling avec strace et ltrace
echo "=== Profiling avec strace et ltrace ==="

# Fonction de profiling système
system_profile() {
    local cmd="$1"
    local output_file="${2:-/tmp/system_profile.txt}"
    
    echo "=== Profiling système ==="
    echo "Commande: $cmd"
    echo "Résultats dans: $output_file"
    
    # Profiling avec strace (appels système)
    if command -v strace >/dev/null 2>&1; then
        echo "Profiling des appels système..."
        strace -c -o "${output_file}.strace" bash -c "$cmd" 2>/dev/null
        
        echo "Résumé strace:"
        cat "${output_file}.strace" | tail -10
    else
        echo "strace non disponible"
    fi
    
    # Profiling avec ltrace (appels de bibliothèque)
    if command -v ltrace >/dev/null 2>&1; then
        echo "Profiling des appels de bibliothèque..."
        ltrace -c -o "${output_file}.ltrace" bash -c "$cmd" 2>/dev/null
        
        echo "Résumé ltrace:"
        cat "${output_file}.ltrace" | tail -10
    else
        echo "ltrace non disponible"
    fi
    
    echo
}

# Tests de profiling système
test_file="/tmp/profiling_test.txt"

# Création d'un fichier de test
echo "Ligne de test pour profiling" > "$test_file"
for i in {1..100}; do echo "Ligne $i avec du contenu" >> "$test_file"; done

# Profiling de différentes opérations
system_profile "cat '$test_file' > /dev/null"
system_profile "grep 'Ligne' '$test_file' > /dev/null"
system_profile "sort '$test_file' > /dev/null"

# Nettoyage
rm -f "$test_file" /tmp/system_profile.txt*
```

## Section 2 : Optimisations algorithmiques

### 2.1 Éviter les pièges algorithmiques courants

Les erreurs qui tuent les performances :

```bash
#!/bin/bash

# Pièges algorithmiques courants
echo "=== Pièges algorithmiques courants ==="

# Fonction de démonstration des pièges
demonstrate_traps() {
    local n="${1:-1000}"
    
    echo "Comparaison pour n=$n"
    
    # Piège 1: Boucles imbriquées inefficaces
    echo "--- Piège 1: Boucles O(n²) ---"
    local start=$(date +%s.%N)
    local count=0
    for ((i=1; i<=n; i++)); do
        for ((j=1; j<=n; j++)); do
            ((count++))
        done
    done
    local end=$(date +%s.%N)
    local time1=$(echo "$end - $start" | bc)
    echo "Boucles imbriquées: ${time1}s (opérations: $count)"
    
    # Version optimisée
    start=$(date +%s.%N)
    count=$((n * n))
    end=$(date +%s.%N)
    local time2=$(echo "$end - $start" | bc)
    echo "Calcul direct: ${time2}s (opérations: $count)"
    
    # Piège 2: Recherche linéaire répétée
    echo "--- Piège 2: Recherche inefficace ---"
    local -a data
    for ((i=1; i<=n; i++)); do
        data+=($i)
    done
    
    # Recherche inefficace
    start=$(date +%s.%N)
    local found=0
    for ((i=1; i<=n; i++)); do
        if [[ "${data[i]}" == "$n" ]]; then
            found=$i
            break
        fi
    done
    end=$(date +%s.%N)
    local time3=$(echo "$end - $start" | bc)
    echo "Recherche linéaire: ${time3}s"
    
    # Version directe (optimisée)
    start=$(date +%s.%N)
    found=$n  # On sait que c'est le dernier élément
    end=$(date +%s.%N)
    local time4=$(echo "$end - $start" | bc)
    echo "Accès direct: ${time4}s"
    
    echo "Amélioration: $(echo "scale=1; $time1 / $time2" | bc)x pour les boucles"
    echo "Amélioration: $(echo "scale=1; $time3 / $time4" | bc)x pour la recherche"
    echo
}

# Tests avec différentes tailles
demonstrate_traps 100
demonstrate_traps 500
```

### 2.2 Optimisations avec les outils externes

Utiliser les bons outils pour les bonnes tâches :

```bash
#!/bin/bash

# Optimisations avec outils externes
echo "=== Optimisations avec outils externes ==="

# Fonction de comparaison d'approches
compare_approaches() {
    local test_file="/tmp/performance_test.txt"
    local pattern="test.*ligne"
    
    # Création du fichier de test
    for i in {1..10000}; do
        echo "Ceci est une ligne de test numéro $i" >> "$test_file"
        echo "Ceci est une autre ligne $i" >> "$test_file"
    done
    
    echo "Fichier de test: $(wc -l < "$test_file") lignes"
    
    # Approche 1: Boucle while en Bash pur
    echo "--- Approche 1: Bash pur ---"
    local start=$(date +%s.%N)
    local count=0
    while IFS= read -r line; do
        if [[ "$line" =~ $pattern ]]; then
            ((count++))
        fi
    done < "$test_file"
    local end=$(date +%s.%N)
    local time1=$(echo "$end - $start" | bc)
    echo "Bash pur: ${time1}s, résultats: $count"
    
    # Approche 2: grep
    echo "--- Approche 2: grep ---"
    start=$(date +%s.%N)
    count=$(grep -c "$pattern" "$test_file")
    end=$(date +%s.%N)
    local time2=$(echo "$end - $start" | bc)
    echo "grep: ${time2}s, résultats: $count"
    
    # Approche 3: awk
    echo "--- Approche 3: awk ---"
    start=$(date +%s.%N)
    count=$(awk "/$pattern/ {count++} END {print count}" "$test_file")
    end=$(date +%s.%N)
    local time3=$(echo "$end - $start" | bc)
    echo "awk: ${time3}s, résultats: $count"
    
    # Approche 4: sed + wc
    echo "--- Approche 4: sed + wc ---"
    start=$(date +%s.%N)
    count=$(sed -n "/$pattern/p" "$test_file" | wc -l)
    end=$(date +%s.%N)
    local time4=$(echo "$end - $start" | bc)
    echo "sed + wc: ${time4}s, résultats: $count"
    
    echo "Performance relative à Bash pur:"
    [[ "$time2" != "0" ]] && echo "  grep: $(echo "scale=1; $time1 / $time2" | bc)x plus rapide"
    [[ "$time3" != "0" ]] && echo "  awk: $(echo "scale=1; $time1 / $time3" | bc)x plus rapide"
    [[ "$time4" != "0" ]] && echo "  sed+wc: $(echo "scale=1; $time1 / $time4" | bc)x plus rapide"
    
    # Nettoyage
    rm -f "$test_file"
    echo
}

compare_approaches
```

### 2.3 Cache intelligent et mémoïsation

Éviter les recalculs inutiles :

```bash
#!/bin/bash

# Cache intelligent et mémoïsation
echo "=== Cache intelligent et mémoïsation ==="

# Fonction avec mémoïsation
declare -A fibonacci_cache

fibonacci_memoized() {
    local n="$1"
    
    # Vérification du cache
    if [[ -n "${fibonacci_cache[$n]}" ]]; then
        echo "${fibonacci_cache[$n]}"
        return 0
    fi
    
    # Calcul récursif avec mémoïsation
    if (( n <= 1 )); then
        result="$n"
    else
        local f1
        local f2
        f1=$(fibonacci_memoized $((n-1)))
        f2=$(fibonacci_memoized $((n-2)))
        result=$((f1 + f2))
    fi
    
    # Mise en cache
    fibonacci_cache[$n]="$result"
    echo "$result"
}

# Fonction sans mémoïsation (pour comparaison)
fibonacci_naive() {
    local n="$1"
    
    if (( n <= 1 )); then
        echo "$n"
    else
        local f1
        local f2
        f1=$(fibonacci_naive $((n-1)))
        f2=$(fibonacci_naive $((n-2)))
        echo $((f1 + f2))
    fi
}

# Comparaison de performance
compare_fibonacci() {
    local n="$1"
    
    echo "Calcul de Fibonacci($n):"
    
    # Avec mémoïsation
    echo "--- Avec mémoïsation ---"
    local start=$(date +%s.%N)
    local result1
    result1=$(fibonacci_memoized "$n")
    local end=$(date +%s.%N)
    local time1=$(echo "$end - $start" | bc)
    echo "Résultat: $result1, Temps: ${time1}s"
    
    # Sans mémoïsation
    echo "--- Sans mémoïsation ---"
    start=$(date +%s.%N)
    local result2
    result2=$(fibonacci_naive "$n")
    end=$(date +%s.%N)
    local time2=$(echo "$end - $start" | bc)
    echo "Résultat: $result2, Temps: ${time2}s"
    
    # Comparaison
    if [[ "$time2" != "0" ]]; then
        local speedup=$(echo "scale=1; $time2 / $time1" | bc)
        echo "Accélération: ${speedup}x"
    fi
    
    echo "Entrées en cache: ${#fibonacci_cache[@]}"
    echo
}

# Tests
compare_fibonacci 10
compare_fibonacci 20
compare_fibonacci 30
```

## Section 3 : Parallélisation et traitement concurrent

### 3.1 Parallélisation avec background jobs

Utiliser les jobs en arrière-plan pour paralléliser :

```bash
#!/bin/bash

# Parallélisation avec background jobs
echo "=== Parallélisation avec background jobs ==="

# Fonction qui simule un traitement coûteux
traitement_couteux() {
    local id="$1"
    local duration="${2:-2}"
    
    echo "[$id] Début du traitement (durée: ${duration}s)"
    sleep "$duration"
    echo "[$id] Traitement terminé"
    
    # Retourner un résultat
    echo "resultat_$id"
}

# Fonction de parallélisation simple
parallel_process() {
    local max_jobs="${1:-4}"
    shift
    
    local pids=()
    local results=()
    
    echo "Traitement parallèle de $# tâches (max $max_jobs jobs simultanés)"
    
    for task in "$@"; do
        # Attendre si on atteint la limite
        while (( ${#pids[@]} >= max_jobs )); do
            # Vérifier les jobs terminés
            for i in "${!pids[@]}"; do
                if ! kill -0 "${pids[$i]}" 2>/dev/null; then
                    # Récupérer le résultat
                    wait "${pids[$i]}"
                    results+=($?)
                    unset pids[$i]
                fi
            done
            pids=("${pids[@]}")  # Reindexer
            
            sleep 0.1  # Éviter la surcharge CPU
        done
        
        # Lancer la tâche
        (
            traitement_couteux "$task"
        ) &
        pids+=($!)
    done
    
    # Attendre la fin de tous les jobs
    for pid in "${pids[@]}"; do
        wait "$pid"
        results+=($?)
    done
    
    echo "Tous les traitements terminés"
    echo "Résultats: ${results[*]}"
}

# Test de parallélisation
echo "--- Test séquentiel ---"
start=$(date +%s.%N)
for i in {1..5}; do
    traitement_couteux "task_$i" 1
done
end=$(date +%s.%N)
time_seq=$(echo "$end - $start" | bc)
echo "Temps séquentiel: ${time_seq}s"

echo "--- Test parallèle ---"
start=$(date +%s.%N)
parallel_process 3 "task_1" "task_2" "task_3" "task_4" "task_5"
end=$(date +%s.%N)
time_par=$(echo "$end - $start" | bc)
echo "Temps parallèle: ${time_par}s"

echo "Accélération: $(echo "scale=2; $time_seq / $time_par" | bc)x"
```

### 3.2 Utilisation de GNU Parallel

L'outil `parallel` pour une parallélisation avancée :

```bash
#!/bin/bash

# Utilisation de GNU Parallel
echo "=== Utilisation de GNU Parallel ==="

# Vérification de la disponibilité
if ! command -v parallel >/dev/null 2>&1; then
    echo "GNU parallel n'est pas installé. Installation recommandée:"
    echo "  Ubuntu/Debian: sudo apt install parallel"
    echo "  CentOS/RHEL: sudo yum install parallel"
    echo "  macOS: brew install parallel"
    exit 1
fi

# Fonction à paralléliser
process_file() {
    local filename="$1"
    local size
    local lines
    local words
    
    size=$(stat -f%z "$filename" 2>/dev/null || stat -c%s "$filename")
    lines=$(wc -l < "$filename")
    words=$(wc -w < "$filename")
    
    echo "$filename: ${size} octets, ${lines} lignes, ${words} mots"
}

# Création de fichiers de test
echo "Création de fichiers de test..."
for i in {1..10}; do
    local filename="/tmp/test_parallel_$i.txt"
    local num_lines=$((RANDOM % 1000 + 100))
    
    for ((j=1; j<=num_lines; j++)); do
        echo "Ceci est la ligne $j du fichier $i avec du contenu aléatoire $RANDOM" >> "$filename"
    done
done

# Test séquentiel
echo "--- Traitement séquentiel ---"
start=$(date +%s.%N)
for file in /tmp/test_parallel_*.txt; do
    process_file "$file"
done
end=$(date +%s.%N)
time_seq=$(echo "$end - $start" | bc)
echo "Temps séquentiel: ${time_seq}s"

# Test parallèle avec GNU parallel
echo "--- Traitement parallèle (4 jobs) ---"
start=$(date +%s.%N)
find /tmp -name "test_parallel_*.txt" -print0 | 
    parallel -0 -j4 process_file
end=$(date +%s.%N)
time_par=$(echo "$end - $start" | bc)
echo "Temps parallèle: ${time_par}s"

echo "Accélération: $(echo "scale=2; $time_seq / $time_par" | bc)x"

# Test avec différentes options
echo "--- Test avec contrôle de charge ---"
# Limiter la charge CPU à 50%
find /tmp -name "test_parallel_*.txt" -print0 | 
    parallel --load 50% -0 process_file |
    head -5

# Nettoyage
rm -f /tmp/test_parallel_*.txt
```

### 3.3 Gestion des ressources et limitation

Éviter la surcharge système :

```bash
#!/bin/bash

# Gestion des ressources et limitation
echo "=== Gestion des ressources et limitation ==="

# Fonction de monitoring des ressources
monitor_resources() {
    local interval="${1:-1}"
    
    echo "=== Monitoring des ressources (intervalle: ${interval}s) ==="
    
    while true; do
        # CPU
        local cpu_usage
        cpu_usage=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
        
        # Mémoire
        local mem_usage
        mem_usage=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100.0}')
        
        # Disque
        local disk_usage
        disk_usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')
        
        echo "$(date '+%H:%M:%S'): CPU: ${cpu_usage}%, RAM: ${mem_usage}%, Disk: ${disk_usage}%"
        
        sleep "$interval"
    done
}

# Fonction avec contrôle de ressources
controlled_process() {
    local max_cpu="${1:-80}"
    local max_mem="${2:-90}"
    
    echo "Processus contrôlé (CPU max: ${max_cpu}%, RAM max: ${max_mem}%)"
    
    while true; do
        # Vérification des ressources
        local current_cpu
        current_cpu=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
        
        local current_mem
        current_mem=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
        
        if (( $(echo "$current_cpu > $max_cpu" | bc -l) )); then
            echo "CPU trop élevé (${current_cpu}%), pause..."
            sleep 5
            continue
        fi
        
        if (( current_mem > max_mem )); then
            echo "Mémoire trop élevée (${current_mem}%), pause..."
            sleep 5
            continue
        fi
        
        # Traitement simulé
        dd if=/dev/zero of=/dev/null bs=1M count=10 2>/dev/null
        sleep 1
    done
}

# Test de limitation
echo "Démarrage du monitoring en arrière-plan..."
monitor_resources 2 &
monitor_pid=$!

echo "Démarrage du processus contrôlé..."
controlled_process 70 85 &
controlled_pid=$!

# Laisser tourner quelques secondes
sleep 10

echo "Arrêt des processus..."
kill $monitor_pid $controlled_pid 2>/dev/null
```

## Section 4 : Optimisations I/O et mémoire

### 4.1 Buffering intelligent

Optimiser les opérations d'entrée/sortie :

```bash
#!/bin/bash

# Optimisations I/O et mémoire
echo "=== Optimisations I/O ==="

# Comparaison de méthodes de lecture
compare_reading_methods() {
    local test_file="/tmp/io_test.txt"
    
    # Création d'un gros fichier de test
    echo "Création du fichier de test..."
    for i in {1..10000}; do
        echo "Ligne numéro $i avec du contenu assez long pour tester les performances d'entrée/sortie" >> "$test_file"
    done
    
    echo "Fichier créé: $(wc -l < "$test_file") lignes, $(du -h "$test_file" | cut -f1)"
    
    # Méthode 1: while read (ligne par ligne)
    echo "--- Méthode 1: while read ---"
    start=$(date +%s.%N)
    local count1=0
    while IFS= read -r line; do
        if [[ "$line" == *"numéro"* ]]; then
            ((count1++))
        fi
    done < "$test_file"
    end=$(date +%s.%N)
    time1=$(echo "$end - $start" | bc)
    echo "while read: ${time1}s, résultats: $count1"
    
    # Méthode 2: grep
    echo "--- Méthode 2: grep ---"
    start=$(date +%s.%N)
    count2=$(grep -c "numéro" "$test_file")
    end=$(date +%s.%N)
    time2=$(echo "$end - $start" | bc)
    echo "grep: ${time2}s, résultats: $count2"
    
    # Méthode 3: awk
    echo "--- Méthode 3: awk ---"
    start=$(date +%s.%N)
    count3=$(awk '/numéro/ {count++} END {print count}' "$test_file")
    end=$(date +%s.%N)
    time3=$(echo "$end - $start" | bc)
    echo "awk: ${time3}s, résultats: $count3"
    
    # Méthode 4: Lecture complète en mémoire
    echo "--- Méthode 4: Lecture complète ---"
    start=$(date +%s.%N)
    content=$(cat "$test_file")
    count4=$(echo "$content" | grep -c "numéro")
    end=$(date +%s.%N)
    time4=$(echo "$end - $start" | bc)
    echo "Lecture complète: ${time4}s, résultats: $count4"
    
    echo "Performance relative à while read:"
    [[ "$time2" != "0" ]] && echo "  grep: $(echo "scale=1; $time1 / $time2" | bc)x plus rapide"
    [[ "$time3" != "0" ]] && echo "  awk: $(echo "scale=1; $time1 / $time3" | bc)x plus rapide"
    [[ "$time4" != "0" ]] && echo "  Lecture complète: $(echo "scale=1; $time1 / $time4" | bc)x plus rapide"
    
    rm -f "$test_file"
}

compare_reading_methods
```

### 4.2 Optimisation de la mémoire

Réduire l'empreinte mémoire :

```bash
#!/bin/bash

# Optimisation de la mémoire
echo "=== Optimisation de la mémoire ==="

# Fonction de monitoring mémoire
memory_usage() {
    local pid="${1:-$$}"
    
    if [[ -f "/proc/$pid/status" ]]; then
        grep -E "VmRSS|VmSize" "/proc/$pid/status" | 
            sed 's/:/: /' | 
            awk '{printf "%s: %d kB (%d MB)\n", $1, $2, $2/1024}'
    else
        echo "Informations mémoire non disponibles (proc non monté)"
    fi
}

# Test d'optimisation mémoire
memory_optimization_test() {
    echo "=== Test d'optimisation mémoire ==="
    
    # Méthode 1: Stocker en variables (gourmand)
    echo "--- Méthode 1: Variables ---"
    local -a lines_array
    for i in {1..10000}; do
        lines_array+=("Ligne $i: $(date) - Contenu assez long pour consommer de la mémoire")
    done
    memory_usage $$
    echo "Éléments stockés: ${#lines_array[@]}"
    
    # Nettoyer
    unset lines_array
    
    # Méthode 2: Traitement en flux (économe)
    echo "--- Méthode 2: Traitement en flux ---"
    local count=0
    for i in {1..10000}; do
        # Traitement immédiat sans stockage
        if (( i % 100 == 0 )); then
            ((count++))
        fi
    done
    memory_usage $$
    echo "Traitements effectués: $count"
    
    # Méthode 3: Utilisation de fichiers temporaires
    echo "--- Méthode 3: Fichiers temporaires ---"
    local temp_file="/tmp/memory_test_$$.txt"
    for i in {1..10000}; do
        echo "Ligne $i" >> "$temp_file"
    done
    local file_count=$(wc -l < "$temp_file")
    memory_usage $$
    echo "Lignes dans fichier: $file_count"
    
    rm -f "$temp_file"
}

memory_optimization_test
```

### 4.3 Cache de fichiers et optimisation

Utiliser le cache système intelligemment :

```bash
#!/bin/bash

# Cache de fichiers et optimisation
echo "=== Cache de fichiers ==="

# Système de cache intelligent
declare -A file_cache
declare -A file_mtime

cached_file_read() {
    local file="$1"
    
    # Vérifier si le fichier existe
    if [[ ! -f "$file" ]]; then
        echo "Erreur: fichier $file inexistant" >&2
        return 1
    fi
    
    # Vérifier la date de modification
    local current_mtime
    current_mtime=$(stat -f%Fm "$file" 2>/dev/null || stat -c%Y "$file")
    
    local cached_mtime="${file_mtime[$file]}"
    
    # Utiliser le cache si valide
    if [[ -n "${file_cache[$file]}" && "$current_mtime" == "$cached_mtime" ]]; then
        echo "${file_cache[$file]}"
        return 0
    fi
    
    # Lire et mettre en cache
    file_cache["$file"]=$(cat "$file")
    file_mtime["$file"]="$current_mtime"
    
    echo "${file_cache[$file]}"
}

# Fonction de statistiques du cache
cache_stats() {
    echo "=== Statistiques du cache ==="
    echo "Fichiers en cache: ${#file_cache[@]}"
    echo "Utilisation mémoire estimée: $(echo "${#file_cache[*]}" | wc -c) caractères"
    
    local total_size=0
    for file in "${!file_cache[@]}"; do
        local size=${#file_cache[$file]}
        total_size=$((total_size + size))
        echo "  $file: ${size} caractères"
    done
    echo "Total: ${total_size} caractères"
}

# Test du cache
echo "--- Test du système de cache ---"

# Création de fichiers de test
echo "Contenu du fichier 1" > /tmp/cache_test1.txt
echo "Contenu plus long du fichier 2 avec beaucoup plus de texte" > /tmp/cache_test2.txt

# Première lecture (chargement en cache)
echo "Première lecture:"
cached_file_read "/tmp/cache_test1.txt" >/dev/null
cached_file_read "/tmp/cache_test2.txt" >/dev/null

cache_stats

# Modification d'un fichier
echo "Contenu modifié du fichier 1" > /tmp/cache_test1.txt

# Deuxième lecture (utilisation du cache)
echo "Deuxième lecture:"
cached_file_read "/tmp/cache_test1.txt" >/dev/null
cached_file_read "/tmp/cache_test2.txt" >/dev/null

cache_stats

# Nettoyage
rm -f /tmp/cache_test*.txt
unset file_cache file_mtime
```

## Section 5 : Benchmarking et tests de performance

### 5.1 Framework de benchmarking complet

Créer un framework pour mesurer et comparer les performances :

```bash
#!/bin/bash

# Framework de benchmarking complet
echo "=== Framework de benchmarking ==="

# Structure de données pour les résultats
declare -A benchmark_results

# Fonction de benchmark
run_benchmark() {
    local name="$1"
    local command="$2"
    local iterations="${3:-10}"
    local warmup="${4:-1}"
    
    echo "=== Benchmark: $name ==="
    echo "Commande: $command"
    echo "Itérations: $iterations"
    echo "Échauffement: $warmup"
    
    # Échauffement
    if (( warmup > 0 )); then
        echo "Échauffement..."
        for ((i=1; i<=warmup; i++)); do
            eval "$command" >/dev/null 2>&1
        done
    fi
    
    # Mesures
    local times=()
    local total_time=0
    
    for ((i=1; i<=iterations; i++)); do
        local start=$(date +%s.%N)
        eval "$command" >/dev/null 2>&1
        local end=$(date +%s.%N)
        
        local duration=$(echo "$end - $start" | bc)
        times+=("$duration")
        total_time=$(echo "$total_time + $duration" | bc)
        
        printf "."
    done
    echo
    
    # Statistiques
    local avg_time=$(echo "scale=4; $total_time / $iterations" | bc)
    local min_time=${times[0]}
    local max_time=${times[0]}
    
    for time in "${times[@]}"; do
        if (( $(echo "$time < $min_time" | bc -l) )); then
            min_time="$time"
        fi
        if (( $(echo "$time > $max_time" | bc -l) )); then
            max_time="$time"
        fi
    done
    
    # Calcul de l'écart-type
    local variance=0
    for time in "${times[@]}"; do
        local diff=$(echo "$time - $avg_time" | bc)
        local squared=$(echo "$diff * $diff" | bc)
        variance=$(echo "$variance + $squared" | bc)
    done
    variance=$(echo "scale=4; $variance / $iterations" | bc)
    local stddev=$(echo "scale=4; sqrt($variance)" | bc)
    
    # Stockage des résultats
    benchmark_results["${name}_avg"]="$avg_time"
    benchmark_results["${name}_min"]="$min_time"
    benchmark_results["${name}_max"]="$max_time"
    benchmark_results["${name}_stddev"]="$stddev"
    
    # Affichage
    printf "Temps moyen: %.4fs\n" "$avg_time"
    printf "Temps min: %.4fs\n" "$min_time"
    printf "Temps max: %.4fs\n" "$max_time"
    printf "Écart-type: %.4fs\n" "$stddev"
    echo
}

# Fonction de comparaison
compare_benchmarks() {
    local baseline="$1"
    shift
    local others=("$@")
    
    echo "=== Comparaison de performances ==="
    echo "Base: $baseline"
    
    local baseline_avg="${benchmark_results[${baseline}_avg]}"
    
    for other in "${others[@]}"; do
        local other_avg="${benchmark_results[${other}_avg]}"
        
        if [[ -n "$other_avg" && -n "$baseline_avg" ]]; then
            local ratio=$(echo "scale=2; $baseline_avg / $other_avg" | bc)
            printf "%-20s : %.2fx %s\n" "$other" "$ratio" "$( (( $(echo "$ratio > 1" | bc -l) )) && echo "plus rapide" || echo "plus lent" )"
        fi
    done
    echo
}

# Exemples de benchmarks
run_benchmark "echo_simple" "echo 'test'"
run_benchmark "seq_100" "seq 1 100 >/dev/null"
run_benchmark "find_temp" "find /tmp -maxdepth 1 -name '*' 2>/dev/null | head -10 >/dev/null"

compare_benchmarks "echo_simple" "seq_100" "find_temp"
```

### 5.2 Profilage mémoire et détection de fuites

Outils pour analyser l'utilisation mémoire :

```bash
#!/bin/bash

# Profilage mémoire et détection de fuites
echo "=== Profilage mémoire ==="

# Fonction de profilage mémoire
memory_profile() {
    local command="$1"
    local duration="${2:-5}"
    
    echo "=== Profilage mémoire ==="
    echo "Commande: $command"
    echo "Durée: ${duration}s"
    
    # Mesures initiales
    local mem_before
    mem_before=$(ps -o rss= $$ 2>/dev/null || echo "0")
    
    # Lancement de la commande en arrière-plan
    eval "$command" &
    local pid=$!
    
    # Collecte des mesures
    local measurements=()
    local times=()
    
    local start_time=$(date +%s)
    while (( $(date +%s) - start_time < duration )); do
        if kill -0 "$pid" 2>/dev/null; then
            local mem_current
            mem_current=$(ps -o rss= "$pid" 2>/dev/null || echo "0")
            measurements+=("$mem_current")
            times+=($(date +%s))
        else
            break
        fi
        sleep 0.1
    done
    
    # Mesures finales
    local mem_after
    mem_after=$(ps -o rss= "$pid" 2>/dev/null || echo "0")
    
    # Attendre la fin du processus
    wait "$pid" 2>/dev/null
    
    # Analyse
    if (( ${#measurements[@]} > 0 )); then
        local min_mem=${measurements[0]}
        local max_mem=${measurements[0]}
        local total_mem=0
        
        for mem in "${measurements[@]}"; do
            if (( mem < min_mem )); then min_mem=$mem; fi
            if (( mem > max_mem )); then max_mem=$mem; fi
            total_mem=$((total_mem + mem))
        done
        
        local avg_mem=$((total_mem / ${#measurements[@]}))
        
        echo "Mémoire - Min: ${min_mem}KB, Moy: ${avg_mem}KB, Max: ${max_mem}KB"
        echo "Évolution: $((mem_after - mem_before))KB"
        
        # Détection de fuite potentielle
        if (( max_mem - min_mem > 10000 )); then  # Plus de 10MB de variation
            echo "⚠️  Possible fuite mémoire détectée"
        else
            echo "✅ Utilisation mémoire stable"
        fi
    else
        echo "Aucune mesure collectée"
    fi
    
    echo
}

# Tests de profilage mémoire
memory_profile "sleep 3" 3
memory_profile "for i in {1..1000}; do echo \$i >/dev/null; done" 2

# Fonction qui consomme de la mémoire
memory_hog() {
    local -a big_array
    for i in {1..10000}; do
        big_array+=("Élément $i avec beaucoup de contenu pour consommer de la mémoire")
    done
    sleep 2  # Maintenir en mémoire
}

memory_profile "memory_hog" 3
```

## Conclusion : L'optimisation, un art et une science

L'optimisation des performances en Bash n'est pas qu'une question de vitesse : c'est un équilibre subtil entre efficacité, maintenabilité et robustesse. Comme un pilote de course qui ajuste constamment les réglages de sa voiture, vous devez profiler, mesurer, et itérer pour atteindre les meilleures performances.

**Points clés à retenir :**

1. **Profilez avant d'optimiser** : Utilisez `time`, `strace`, et des benchmarks pour identifier les vrais goulots d'étranglement
2. **Choisissez les bons algorithmes** : Évitez les O(n²) quand O(n) suffit, utilisez les outils externes appropriés
3. **Parallélisez intelligemment** : Utilisez GNU parallel et les background jobs, mais respectez les limites système
4. **Optimisez I/O et mémoire** : Privilégiez le traitement en flux, utilisez le cache, surveillez la consommation
5. **Benchmarkez systématiquement** : Créez des frameworks de test pour mesurer et comparer objectivement

Dans le chapitre suivant, nous explorerons les techniques avancées de gestion des ressources système, pour que vos scripts Bash soient non seulement rapides, mais aussi respectueux de leur environnement d'exécution.

---

**Exercice pratique :** Créez un script d'optimisation automatique qui :
- Profile les performances actuelles d'un script donné
- Identifie les goulots d'étranglement (I/O, CPU, mémoire)
- Applique des optimisations automatiques (parallélisation, cache, etc.)
- Compare les performances avant/après optimisation
- Génère un rapport détaillé des améliorations

**Réflexion :** Comment adapteriez-vous ces techniques d'optimisation pour des environnements cloud avec des contraintes de coût (CPU credits, mémoire allouée) ?
