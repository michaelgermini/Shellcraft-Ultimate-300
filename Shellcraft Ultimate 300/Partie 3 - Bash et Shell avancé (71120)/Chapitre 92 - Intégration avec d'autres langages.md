# Chapitre 92 - Intégration avec d'autres langages

> "Bash n'est pas une île : c'est le chef d'orchestre d'un symphony linguistique où chaque langage joue sa partition avec excellence." - Scripting Polyglot Proverbe

## Introduction : Le symphony polyglotte

Imaginez-vous comme un maestro dirigeant un orchestre où chaque instrument - Python, Ruby, Perl, JavaScript, voire C - joue sa partition parfaite. L'intégration avec d'autres langages en Bash transforme le shell d'un outil isolé en un chef d'orchestre capable d'orchestrer les forces de chaque langage pour la tâche idéale.

Dans ce chapitre, nous construirons des ponts entre Bash et d'autres langages : interfaces de fonctions étrangères, appels de programmes externes, formats d'échange de données, et scripts polyglottes qui exploitent le meilleur de chaque monde.

## Section 1 : Interfaces de fonctions étrangères (FFI)

### 1.1 Architecture FFI en Bash

Système d'interfaçage avec fonctions externes via des wrappers intelligents :

```bash
#!/bin/bash

# Interfaces de fonctions étrangères (FFI) en Bash
echo "=== Interfaces de fonctions étrangères (FFI) ==="

# Foreign Function Interface Framework
FFIFramework() {
    local self="$1"
    
    declare -A $self._ffi_bindings
    declare -A $self._type_mappings
    declare -A $self._language_runtimes
    
    # Enregistrement d'un binding FFI
    $self.register_ffi_binding() {
        local binding_name="$1"
        local language="$2"
        local function_name="$3"
        local signature="$4"
        local implementation="$5"
        
        $self._ffi_bindings["${binding_name}_language"]="$language"
        $self._ffi_bindings["${binding_name}_function"]="$function_name"
        $self._ffi_bindings["${binding_name}_signature"]="$signature"
        $self._ffi_bindings["${binding_name}_implementation"]="$implementation"
        
        echo "✓ Binding FFI enregistré: $binding_name ($language)"
    }
    
    # Définition des mappings de types
    $self.define_type_mappings() {
        local language="$1"
        
        case "$language" in
            python)
                $self._type_mappings["${language}_int"]="int"
                $self._type_mappings["${language}_float"]="float"
                $self._type_mappings["${language}_string"]="str"
                $self._type_mappings["${language}_array"]="list"
                $self._type_mappings["${language}_hash"]="dict"
                $self._type_mappings["${language}_bool"]="bool"
                ;;
                
            ruby)
                $self._type_mappings["${language}_int"]="Integer"
                $self._type_mappings["${language}_float"]="Float"
                $self._type_mappings["${language}_string"]="String"
                $self._type_mappings["${language}_array"]="Array"
                $self._type_mappings["${language}_hash"]="Hash"
                $self._type_mappings["${language}_bool"]="Boolean"
                ;;
                
            perl)
                $self._type_mappings["${language}_int"]=""
                $self._type_mappings["${language}_float"]=""
                $self._type_mappings["${language}_string"]=""
                $self._type_mappings["${language}_array"]="ARRAY"
                $self._type_mappings["${language}_hash"]="HASH"
                $self._type_mappings["${language}_bool"]=""
                ;;
                
            javascript|node)
                $self._type_mappings["${language}_int"]="number"
                $self._type_mappings["${language}_float"]="number"
                $self._type_mappings["${language}_string"]="string"
                $self._type_mappings["${language}_array"]="Array"
                $self._type_mappings["${language}_hash"]="Object"
                $self._type_mappings["${language}_bool"]="boolean"
                ;;
        esac
    }
    
    # Appel d'une fonction FFI
    $self.call_ffi_function() {
        local binding_name="$1"
        shift
        
        local language="${$self._ffi_bindings[${binding_name}_language]}"
        local function_name="${$self._ffi_bindings[${binding_name}_function]}"
        local signature="${$self._ffi_bindings[${binding_name}_signature]}"
        local implementation="${$self._ffi_bindings[${binding_name}_implementation]}"
        
        if [[ -z "$language" ]]; then
            echo "❌ Binding FFI introuvable: $binding_name" >&2
            return 1
        fi
        
        echo "Appel FFI: $binding_name ($language.$function_name)"
        
        # Conversion des arguments selon la signature
        local converted_args
        converted_args="$($self._convert_arguments "$language" "$signature" "$@")"
        
        # Exécution selon le langage
        case "$language" in
            python)
                $self._call_python_function "$function_name" "$implementation" "$converted_args"
                ;;
                
            ruby)
                $self._call_ruby_function "$function_name" "$implementation" "$converted_args"
                ;;
                
            perl)
                $self._call_perl_function "$function_name" "$implementation" "$converted_args"
                ;;
                
            javascript|node)
                $self._call_nodejs_function "$function_name" "$implementation" "$converted_args"
                ;;
                
            *)
                echo "❌ Langage non supporté: $language" >&2
                return 1
                ;;
        esac
    }
    
    # Conversion des arguments
    $self._convert_arguments() {
        local language="$1"
        local signature="$2"
        shift 2
        local -a args=("$@")
        
        # Parsing de la signature (ex: "int,string->array")
        local input_types output_type
        input_types="$(echo "$signature" | cut -d'-' -f1)"
        output_type="$(echo "$signature" | cut -d'-' -f2 | cut -d'>' -f2)"
        
        # Conversion des types d'entrée
        local -a converted_args=()
        local i=0
        
        IFS=',' read -ra types <<< "$input_types"
        for type in "${types[@]}"; do
            local value="${args[$i]}"
            local converted_value
            
            converted_value="$($self._convert_value "$language" "$type" "$value")"
            converted_args+=("$converted_value")
            ((i++))
        done
        
        # Formatage selon le langage
        case "$language" in
            python)
                echo "[${converted_args[*]}]"
                ;;
                
            ruby)
                echo "[${converted_args[*]}]"
                ;;
                
            perl)
                echo "(${converted_args[*]})"
                ;;
                
            javascript|node)
                echo "[${converted_args[*]}]"
                ;;
        esac
    }
    
    # Conversion d'une valeur
    $self._convert_value() {
        local language="$1"
        local type="$2"
        local value="$3"
        
        case "$language" in
            python)
                case "$type" in
                    int) echo "$value" ;;
                    float) echo "$value" ;;
                    string) echo "\"$value\"" ;;
                    bool) [[ "$value" == "true" ]] && echo "True" || echo "False" ;;
                    array) echo "$value" ;;  # Assume déjà formaté
                    *) echo "\"$value\"" ;;
                esac
                ;;
                
            ruby)
                case "$type" in
                    int) echo "$value" ;;
                    float) echo "$value" ;;
                    string) echo "\"$value\"" ;;
                    bool) [[ "$value" == "true" ]] && echo "true" || echo "false" ;;
                    array) echo "$value" ;;
                    *) echo "\"$value\"" ;;
                esac
                ;;
                
            *)
                echo "\"$value\""
                ;;
        esac
    }
    
    # Appel de fonction Python
    $self._call_python_function() {
        local function_name="$1"
        local implementation="$2"
        local args="$3"
        
        # Création d'un script Python temporaire
        local temp_script
        temp_script="$(mktemp).py"
        
        cat > "$temp_script" << EOF
import sys
import json

$implementation

# Appel de la fonction
try:
    result = $function_name(*$args)
    if isinstance(result, (list, dict)):
        print(json.dumps(result))
    else:
        print(result)
except Exception as e:
    print(f"Erreur Python: {e}", file=sys.stderr)
    sys.exit(1)
EOF
        
        # Exécution
        if python3 "$temp_script" 2>&1; then
            rm -f "$temp_script"
            return 0
        else
            local exit_code=$?
            rm -f "$temp_script"
            return $exit_code
        fi
    }
    
    # Appel de fonction Ruby
    $self._call_ruby_function() {
        local function_name="$1"
        local implementation="$2"
        local args="$3"
        
        local temp_script
        temp_script="$(mktemp).rb"
        
        cat > "$temp_script" << EOF
require 'json'

$implementation

begin
    result = $function_name(*$args)
    if result.is_a?(Array) || result.is_a?(Hash)
        puts JSON.generate(result)
    else
        puts result
    end
rescue => e
    STDERR.puts "Erreur Ruby: #{e}"
    exit 1
end
EOF
        
        if ruby "$temp_script" 2>&1; then
            rm -f "$temp_script"
            return 0
        else
            local exit_code=$?
            rm -f "$temp_script"
            return $exit_code
        fi
    }
    
    # Appel de fonction Perl
    $self._call_perl_function() {
        local function_name="$1"
        local implementation="$2"
        local args="$3"
        
        local temp_script
        temp_script="$(mktemp).pl"
        
        cat > "$temp_script" << EOF
use strict;
use warnings;
use JSON;

$implementation

eval {
    my \$result = $function_name(@{\$args});
    if (ref(\$result) eq 'ARRAY' || ref(\$result) eq 'HASH') {
        print encode_json(\$result);
    } else {
        print \$result;
    }
};
if (\$@) {
    print STDERR "Erreur Perl: \$@";
    exit 1;
}
EOF
        
        if perl "$temp_script" 2>&1; then
            rm -f "$temp_script"
            return 0
        else
            local exit_code=$?
            rm -f "$temp_script"
            return $exit_code
        fi
    }
    
    # Appel de fonction Node.js
    $self._call_nodejs_function() {
        local function_name="$1"
        local implementation="$2"
        local args="$3"
        
        local temp_script
        temp_script="$(mktemp).js"
        
        cat > "$temp_script" << EOF
$implementation

try {
    const result = $function_name(...$args);
    if (Array.isArray(result) || typeof result === 'object') {
        console.log(JSON.stringify(result));
    } else {
        console.log(result);
    }
} catch (e) {
    console.error('Erreur JavaScript:', e.message);
    process.exit(1);
}
EOF
        
        if node "$temp_script" 2>&1; then
            rm -f "$temp_script"
            return 0
        else
            local exit_code=$?
            rm -f "$temp_script"
            return $exit_code
        fi
    }
    
    # Création d'un wrapper FFI automatique
    $self.create_ffi_wrapper() {
        local binding_name="$1"
        local output_file="$2"
        
        local language="${$self._ffi_bindings[${binding_name}_language]}"
        local function_name="${$self._ffi_bindings[${binding_name}_function]}"
        local signature="${$self._ffi_bindings[${binding_name}_signature]}"
        
        # Génération du wrapper Bash
        {
            echo "#!/bin/bash"
            echo "# Wrapper FFI généré automatiquement pour $binding_name"
            echo "# Langage: $language"
            echo "# Fonction: $function_name"
            echo "# Signature: $signature"
            echo
            echo "# Fonction wrapper"
            echo "${binding_name}() {"
            echo "    # Appel FFI via le framework"
            echo "    ffi_framework.call_ffi_function \"$binding_name\" \"\$@\""
            echo "}"
            echo
            echo "# Si exécuté directement"
            echo "if [[ \"\${BASH_SOURCE[0]}\" == \"\$0\" ]]; then"
            echo "    $binding_name \"\$@\""
            echo "fi"
        } > "$output_file"
        
        chmod +x "$output_file"
        echo "✓ Wrapper FFI généré: $output_file"
    }
    
    # Benchmarking des appels FFI
    $self.benchmark_ffi_calls() {
        local binding_name="$1"
        local iterations="${2:-100}"
        
        echo "=== BENCHMARK FFI: $binding_name ==="
        echo "Itérations: $iterations"
        
        local start_time
        start_time="$(date +%s.%N)"
        
        local i
        for ((i=1; i<=iterations; i++)); do
            $self.call_ffi_function "$binding_name" "test_arg_$i" >/dev/null 2>&1
        done
        
        local end_time
        end_time="$(date +%s.%N)"
        
        local duration
        duration="$(echo "$end_time - $start_time" | bc -l)"
        
        local avg_time
        avg_time="$(echo "scale=4; $duration / $iterations" | bc -l)"
        
        echo "Temps total: ${duration}s"
        echo "Temps moyen par appel: ${avg_time}s"
        echo "Appels par seconde: $(echo "scale=0; $iterations / $duration" | bc -l)"
    }
}

# Fonctions d'exemple pour les bindings
python_example_functions() {
    cat << 'EOF'
def calculate_fibonacci(n):
    """Calcule le n-ième nombre de Fibonacci"""
    if n <= 1:
        return n
    else:
        return calculate_fibonacci(n-1) + calculate_fibonacci(n-2)

def process_text(text, operation):
    """Traite du texte selon l'opération spécifiée"""
    if operation == "uppercase":
        return text.upper()
    elif operation == "lowercase":
        return text.lower()
    elif operation == "reverse":
        return text[::-1]
    else:
        return text

def analyze_data(numbers):
    """Analyse une liste de nombres"""
    if not numbers:
        return {"error": "Liste vide"}
    
    return {
        "count": len(numbers),
        "sum": sum(numbers),
        "average": sum(numbers) / len(numbers),
        "min": min(numbers),
        "max": max(numbers)
    }
EOF
}

ruby_example_functions() {
    cat << 'EOF'
def encrypt_string(text, shift = 3)
    text.chars.map do |char|
        if char.match?(/[a-z]/)
            ((char.ord - 'a'.ord + shift) % 26 + 'a'.ord).chr
        elsif char.match?(/[A-Z]/)
            ((char.ord - 'A'.ord + shift) % 26 + 'A'.ord).chr
        else
            char
        end
    end.join
end

def validate_email(email)
    email.match?(/\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i)
end

def generate_password(length = 12)
    chars = ('a'..'z').to_a + ('A'..'Z').to_a + ('0'..'9').to_a + ['!', '@', '#', '$', '%', '^', '&', '*']
    (0...length).map { chars[rand(chars.length)] }.join
end
EOF
}

perl_example_functions() {
    cat << 'EOF'
sub calculate_checksum {
    my ($data) = @_;
    my $sum = 0;
    foreach my $byte (unpack('C*', $data)) {
        $sum += $byte;
    }
    return $sum % 256;
}

sub format_json_pretty {
    my ($json_text) = @_;
    eval {
        require JSON;
        my $data = JSON->new->decode($json_text);
        return JSON->new->pretty->encode($data);
    };
    if ($@) {
        return "Erreur JSON: $@";
    }
}

sub find_duplicates {
    my @array = @_;
    my %seen;
    my @duplicates;
    foreach my $item (@array) {
        $seen{$item}++;
    }
    foreach my $item (keys %seen) {
        if ($seen{$item} > 1) {
            push @duplicates, $item;
        }
    }
    return \@duplicates;
}
EOF
}

nodejs_example_functions() {
    cat << 'EOF'
function hash_string(text, algorithm = 'sha256') {
    const crypto = require('crypto');
    return crypto.createHash(algorithm).update(text).digest('hex');
}

function parse_csv(csvText) {
    const lines = csvText.trim().split('\n');
    const headers = lines[0].split(',');
    const data = [];
    
    for (let i = 1; i < lines.length; i++) {
        const values = lines[i].split(',');
        const row = {};
        headers.forEach((header, index) => {
            row[header.trim()] = values[index].trim();
        });
        data.push(row);
    }
    
    return data;
}

function http_request(url, method = 'GET') {
    const http = require('http');
    const https = require('https');
    
    return new Promise((resolve, reject) => {
        const client = url.startsWith('https://') ? https : http;
        const req = client.request(url, { method }, (res) => {
            let data = '';
            res.on('data', (chunk) => data += chunk);
            res.on('end', () => resolve({ status: res.statusCode, data }));
        });
        
        req.on('error', reject);
        req.end();
    });
}
EOF
}

# Démonstration du framework FFI
echo "--- Framework FFI ---"

FFIFramework "ffi_framework"

# Définition des mappings de types
ffi_framework.define_type_mappings "python"
ffi_framework.define_type_mappings "ruby"
ffi_framework.define_type_mappings "perl"
ffi_framework.define_type_mappings "node"

# Enregistrement des bindings FFI
echo "Enregistrement des bindings FFI..."

# Python bindings
ffi_framework.register_ffi_binding "fibonacci" "python" "calculate_fibonacci" "int->int" "$(python_example_functions | grep -A 3 "def calculate_fibonacci")"
ffi_framework.register_ffi_binding "text_processor" "python" "process_text" "string,string->string" "$(python_example_functions | grep -A 5 "def process_text")"
ffi_framework.register_ffi_binding "data_analyzer" "python" "analyze_data" "array->hash" "$(python_example_functions | grep -A 10 "def analyze_data")"

# Ruby bindings
ffi_framework.register_ffi_binding "caesar_cipher" "ruby" "encrypt_string" "string,int->string" "$(ruby_example_functions | grep -A 10 "def encrypt_string")"
ffi_framework.register_ffi_binding "email_validator" "ruby" "validate_email" "string->bool" "$(ruby_example_functions | grep -A 2 "def validate_email")"
ffi_framework.register_ffi_binding "password_generator" "ruby" "generate_password" "int->string" "$(ruby_example_functions | grep -A 3 "def generate_password")"

# Perl bindings
ffi_framework.register_ffi_binding "checksum_calculator" "perl" "calculate_checksum" "string->int" "$(perl_example_functions | grep -A 6 "sub calculate_checksum")"
ffi_framework.register_ffi_binding "json_formatter" "perl" "format_json_pretty" "string->string" "$(perl_example_functions | grep -A 8 "sub format_json_pretty")"
ffi_framework.register_ffi_binding "duplicate_finder" "perl" "find_duplicates" "array->array" "$(perl_example_functions | grep -A 10 "sub find_duplicates")"

# Node.js bindings
ffi_framework.register_ffi_binding "string_hasher" "node" "hash_string" "string,string->string" "$(nodejs_example_functions | grep -A 2 "function hash_string")"
ffi_framework.register_ffi_binding "csv_parser" "node" "parse_csv" "string->array" "$(nodejs_example_functions | grep -A 12 "function parse_csv")"

echo
echo "--- Tests des appels FFI ---"

# Test Python
echo "Test Fibonacci (Python):"
ffi_framework.call_ffi_function "fibonacci" "10"

echo
echo "Test traitement de texte (Python):"
ffi_framework.call_ffi_function "text_processor" "Hello World" "uppercase"

echo
echo "Test analyse de données (Python):"
ffi_framework.call_ffi_function "data_analyzer" "[1,2,3,4,5]"

echo
echo "Test chiffrement César (Ruby):"
ffi_framework.call_ffi_function "caesar_cipher" "Hello" "3"

echo
echo "Test validation email (Ruby):"
ffi_framework.call_ffi_function "email_validator" "test@example.com"

echo
echo "Test calcul de checksum (Perl):"
ffi_framework.call_ffi_function "checksum_calculator" "Hello World"

echo
echo "Test formatage JSON (Perl):"
ffi_framework.call_ffi_function "json_formatter" '{"name":"test","value":123}'

echo
echo "Test hash de chaîne (Node.js):"
ffi_framework.call_ffi_function "string_hasher" "password123" "sha256"

echo
echo "--- Génération de wrappers ---"
ffi_framework.create_ffi_wrapper "fibonacci" "/tmp/fib_wrapper.sh"
ffi_framework.create_ffi_wrapper "caesar_cipher" "/tmp/caesar_wrapper.sh"

echo
echo "Wrappers générés:"
ls -la /tmp/*_wrapper.sh

echo
echo "--- Benchmarking ---"
ffi_framework.benchmark_ffi_calls "fibonacci" "10"

# Nettoyage
rm -f /tmp/*_wrapper.sh
```

### 1.2 Intégration avec C via wrappers

Connexion avec du code C compilé pour des performances optimales :

```bash
#!/bin/bash

# Intégration avec C via wrappers
echo "=== Intégration avec C via wrappers ==="

# C Integration Framework
CIntegration() {
    local self="$1"
    
    declare -A $self._c_functions
    declare -A $self._c_libraries
    
    # Enregistrement d'une fonction C
    $self.register_c_function() {
        local func_name="$1"
        local signature="$2"
        local c_code="$3"
        local libraries="${4:-}"
        
        $self._c_functions["${func_name}_signature"]="$signature"
        $self._c_functions["${func_name}_code"]="$c_code"
        $self._c_functions["${func_name}_libraries"]="$libraries"
        
        echo "✓ Fonction C enregistrée: $func_name"
    }
    
    # Compilation et chargement d'une fonction C
    $self.compile_c_function() {
        local func_name="$1"
        
        local signature="${$self._c_functions[${func_name}_signature]}"
        local c_code="${$self._c_functions[${func_name}_code]}"
        local libraries="${$self._c_functions[${func_name}_libraries]}"
        
        if [[ -z "$signature" ]]; then
            echo "❌ Fonction C introuvable: $func_name" >&2
            return 1
        fi
        
        # Création du programme C complet
        local c_program
        c_program="$(mktemp).c"
        
        cat > "$c_program" << EOF
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

$c_code

int main(int argc, char *argv[]) {
    // Parsing des arguments selon la signature
    $signature
    
    return 0;
}
EOF
        
        # Compilation
        local binary
        binary="$(mktemp)_${func_name}"
        
        local compile_cmd="gcc -o '$binary' '$c_program'"
        if [[ -n "$libraries" ]]; then
            compile_cmd="$compile_cmd $libraries"
        fi
        
        echo "Compilation: $compile_cmd"
        
        if eval "$compile_cmd" 2>&1; then
            chmod +x "$binary"
            $self._c_functions["${func_name}_binary"]="$binary"
            rm -f "$c_program"
            echo "✓ Compilation réussie: $binary"
            return 0
        else
            rm -f "$c_program"
            echo "❌ Échec de compilation" >&2
            return 1
        fi
    }
    
    # Appel d'une fonction C
    $self.call_c_function() {
        local func_name="$1"
        shift
        
        local binary="${$self._c_functions[${func_name}_binary]}"
        
        if [[ -z "$binary" ]]; then
            echo "❌ Binaire non trouvé pour $func_name, tentative de compilation..." >&2
            if ! $self.compile_c_function "$func_name"; then
                return 1
            fi
            binary="${$self._c_functions[${func_name}_binary]}"
        fi
        
        echo "Exécution: $binary $@"
        "$binary" "$@"
    }
    
    # Création d'un wrapper persistant
    $self.create_c_wrapper() {
        local func_name="$1"
        local wrapper_file="$2"
        
        local signature="${$self._c_functions[${func_name}_signature]}"
        
        {
            echo "#!/bin/bash"
            echo "# Wrapper C généré pour $func_name"
            echo
            echo "${func_name}() {"
            echo "    # Appel de la fonction C compilée"
            echo "    c_integration.call_c_function \"$func_name\" \"\$@\""
            echo "}"
            echo
            echo "if [[ \"\${BASH_SOURCE[0]}\" == \"\$0\" ]]; then"
            echo "    $func_name \"\$@\""
            echo "fi"
        } > "$wrapper_file"
        
        chmod +x "$wrapper_file"
        echo "✓ Wrapper C créé: $wrapper_file"
    }
    
    # Cache des binaires compilés
    $self.cache_compiled_binaries() {
        local cache_dir="${1:-$HOME/.c_wrappers_cache}"
        
        mkdir -p "$cache_dir"
        
        echo "Cache des binaires dans: $cache_dir"
        
        for func_key in "${!$self._c_functions[@]}"; do
            if [[ "$func_key" =~ _binary$ ]]; then
                local func_name="${func_key%_binary}"
                local binary="${$self._c_functions[$func_key]}"
                
                if [[ -f "$binary" ]]; then
                    local cached_binary="$cache_dir/${func_name}"
                    cp "$binary" "$cached_binary"
                    $self._c_functions["${func_name}_binary"]="$cached_binary"
                    echo "✓ $func_name mis en cache"
                fi
            fi
        done
    }
    
    # Nettoyage des binaires temporaires
    $self.cleanup_temp_binaries() {
        for func_key in "${!$self._c_functions[@]}"; do
            if [[ "$func_key" =~ _binary$ ]]; then
                local binary="${$self._c_functions[$func_key]}"
                
                if [[ "$binary" =~ ^/tmp/ ]]; then
                    rm -f "$binary"
                    echo "✓ Nettoyé: $binary"
                fi
            fi
        done
    }
}

# Fonctions C d'exemple
c_example_functions() {
    cat << 'EOF'
/* Fonction de calcul de factorielle */
unsigned long long factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);
}

/* Fonction de recherche dans un tableau */
int linear_search(int arr[], int size, int target) {
    for (int i = 0; i < size; i++) {
        if (arr[i] == target) return i;
    }
    return -1;
}

/* Fonction de tri rapide */
void quicksort(int arr[], int low, int high) {
    if (low < high) {
        int pivot = arr[high];
        int i = low - 1;
        
        for (int j = low; j < high; j++) {
            if (arr[j] < pivot) {
                i++;
                int temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
        
        int temp = arr[i + 1];
        arr[i + 1] = arr[high];
        arr[high] = temp;
        
        quicksort(arr, low, i);
        quicksort(arr, i + 2, high);
    }
}

/* Fonction de calcul de nombre premier */
int is_prime(int n) {
    if (n <= 1) return 0;
    if (n <= 3) return 1;
    if (n % 2 == 0 || n % 3 == 0) return 0;
    
    for (int i = 5; i * i <= n; i += 6) {
        if (n % i == 0 || n % (i + 2) == 0) return 0;
    }
    
    return 1;
}
EOF
}

# Signatures d'appel pour les fonctions C
c_signatures() {
    cat << 'EOF'
    // Signature pour factorial
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <n>\n", argv[0]);
        return 1;
    }
    int n = atoi(argv[1]);
    unsigned long long result = factorial(n);
    printf("%llu\n", result);

    // Signature pour linear_search
    if (argc < 3) {
        fprintf(stderr, "Usage: %s <array_elements> <target>\n", argv[0]);
        return 1;
    }
    int size = argc - 2;
    int *arr = malloc(size * sizeof(int));
    for (int i = 0; i < size; i++) {
        arr[i] = atoi(argv[i + 1]);
    }
    int target = atoi(argv[argc - 1]);
    int index = linear_search(arr, size, target);
    printf("%d\n", index);
    free(arr);

    // Signature pour quicksort
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <array_elements>\n", argv[0]);
        return 1;
    }
    int sort_size = argc - 1;
    int *sort_arr = malloc(sort_size * sizeof(int));
    for (int i = 0; i < sort_size; i++) {
        sort_arr[i] = atoi(argv[i + 1]);
    }
    quicksort(sort_arr, 0, sort_size - 1);
    for (int i = 0; i < sort_size; i++) {
        printf("%d ", sort_arr[i]);
    }
    printf("\n");
    free(sort_arr);

    // Signature pour is_prime
    if (argc != 2) {
        fprintf(stderr, "Usage: %s <n>\n", argv[0]);
        return 1;
    }
    int prime_n = atoi(argv[1]);
    int result = is_prime(prime_n);
    printf("%s\n", result ? "true" : "false");
EOF
}

# Démonstration de l'intégration C
echo "--- Intégration C ---"

CIntegration "c_integration"

# Enregistrement des fonctions C
echo "Enregistrement des fonctions C..."

# Fonction factorielle
c_integration.register_c_function "factorial" \
    "$(c_signatures | sed -n '/factorial/,/printf/p' | head -n -1)" \
    "$(c_example_functions | sed -n '/factorial/,/^$/p' | head -n -1)"

# Fonction recherche linéaire
c_integration.register_c_function "linear_search" \
    "$(c_signatures | sed -n '/linear_search/,/free/p' | head -n -1)" \
    "$(c_example_functions | sed -n '/linear_search/,/^$/p' | head -n -1)"

# Fonction tri rapide
c_integration.register_c_function "quicksort" \
    "$(c_signatures | sed -n '/quicksort/,/free/p' | head -n -1)" \
    "$(c_example_functions | sed -n '/quicksort/,/^$/p' | head -n -1)"

# Fonction nombre premier
c_integration.register_c_function "is_prime" \
    "$(c_signatures | sed -n '/is_prime/,/printf/p' | head -n -1)" \
    "$(c_example_functions | sed -n '/is_prime/,/^$/p' | head -n -1)"

echo
echo "--- Tests des fonctions C ---"

echo "Test factorielle:"
c_integration.call_c_function "factorial" "5"

echo
echo "Test recherche linéaire:"
c_integration.call_c_function "linear_search" "10" "20" "30" "40" "50" "30"

echo
echo "Test tri rapide:"
c_integration.call_c_function "quicksort" "64" "34" "25" "12" "22" "11" "90"

echo
echo "Test nombre premier:"
c_integration.call_c_function "is_prime" "17"
c_integration.call_c_function "is_prime" "18"

echo
echo "--- Création de wrappers ---"
c_integration.create_c_wrapper "factorial" "/tmp/factorial_wrapper.sh"
c_integration.create_c_wrapper "is_prime" "/tmp/prime_wrapper.sh"

echo "Wrappers créés:"
ls -la /tmp/*_wrapper.sh

echo
echo "--- Cache des binaires ---"
c_integration.cache_compiled_binaries "/tmp/c_cache"

echo "Binaires en cache:"
ls -la /tmp/c_cache/

# Nettoyage
c_integration.cleanup_temp_binaries
rm -rf /tmp/c_cache /tmp/*_wrapper.sh
```

## Section 2 : Formats d'échange de données et sérialisation

### 2.1 Framework d'échange de données polyglotte

Système unifié pour l'échange de données entre langages :

```bash
#!/bin/bash

# Framework d'échange de données polyglotte
echo "=== Framework d'échange de données polyglotte ==="

# Data Interchange Framework
DataInterchange() {
    local self="$1"
    
    declare -A $self._serializers
    declare -A $self._deserializers
    declare -A $self._converters
    
    # Enregistrement d'un sérialiseur
    $self.register_serializer() {
        local format="$1"
        local language="$2"
        local serialize_code="$3"
        
        $self._serializers["${format}_${language}"]="$serialize_code"
        echo "✓ Sérialiseur enregistré: $format ($language)"
    }
    
    # Enregistrement d'un désérialiseur
    $self.register_deserializer() {
        local format="$1"
        local language="$2"
        local deserialize_code="$3"
        
        $self._deserializers["${format}_${language}"]="$deserialize_code"
        echo "✓ Désérialiseur enregistré: $format ($language)"
    }
    
    # Conversion de données entre formats
    $self.convert_data() {
        local source_format="$1"
        local target_format="$2"
        local language="$3"
        local data="$4"
        
        echo "Conversion: $source_format -> $target_format ($language)"
        
        # Désérialisation depuis le format source
        local intermediate_data
        intermediate_data="$($self.deserialize "$source_format" "$language" "$data")"
        
        if [[ $? -ne 0 ]]; then
            echo "❌ Échec de désérialisation" >&2
            return 1
        fi
        
        # Sérialisation vers le format cible
        local result
        result="$($self.serialize "$target_format" "$language" "$intermediate_data")"
        
        if [[ $? -eq 0 ]]; then
            echo "$result"
            return 0
        else
            echo "❌ Échec de sérialisation" >&2
            return 1
        fi
    }
    
    # Sérialisation de données
    $self.serialize() {
        local format="$1"
        local language="$2"
        local data="$3"
        
        local serializer_key="${format}_${language}"
        local serializer_code="${$self._serializers[$serializer_key]}"
        
        if [[ -z "$serializer_code" ]]; then
            echo "❌ Sérialiseur introuvable: $format ($language)" >&2
            return 1
        fi
        
        case "$language" in
            python)
                $self._python_serialize "$format" "$data"
                ;;
                
            ruby)
                $self._ruby_serialize "$format" "$data"
                ;;
                
            perl)
                $self._perl_serialize "$format" "$data"
                ;;
                
            javascript)
                $self._javascript_serialize "$format" "$data"
                ;;
                
            bash)
                $self._bash_serialize "$format" "$data"
                ;;
                
            *)
                echo "❌ Langage non supporté: $language" >&2
                return 1
                ;;
        esac
    }
    
    # Désérialisation de données
    $self.deserialize() {
        local format="$1"
        local language="$2"
        local data="$3"
        
        local deserializer_key="${format}_${language}"
        local deserializer_code="${$self._deserializers[$deserializer_key]}"
        
        if [[ -z "$deserializer_code" ]]; then
            echo "❌ Désérialiseur introuvable: $format ($language)" >&2
            return 1
        fi
        
        case "$language" in
            python)
                $self._python_deserialize "$format" "$data"
                ;;
                
            ruby)
                $self._ruby_deserialize "$format" "$data"
                ;;
                
            perl)
                $self._perl_deserialize "$format" "$data"
                ;;
                
            javascript)
                $self._javascript_deserialize "$format" "$data"
                ;;
                
            bash)
                $self._bash_deserialize "$format" "$data"
                ;;
                
            *)
                echo "❌ Langage non supporté: $language" >&2
                return 1
                ;;
        esac
    }
    
    # Sérialisation Python
    $self._python_serialize() {
        local format="$1"
        local data="$2"
        
        local temp_script
        temp_script="$(mktemp).py"
        
        cat > "$temp_script" << EOF
import sys
import json
import pickle
import yaml

data = $data

try:
    if '$format' == 'json':
        result = json.dumps(data)
    elif '$format' == 'yaml':
        import yaml
        result = yaml.dump(data)
    elif '$format' == 'pickle':
        result = pickle.dumps(data).hex()
    else:
        result = str(data)
    
    print(result)
except Exception as e:
    print(f"Erreur sérialisation: {e}", file=sys.stderr)
    sys.exit(1)
EOF
        
        python3 "$temp_script" 2>&1 && rm -f "$temp_script" || { rm -f "$temp_script"; return 1; }
    }
    
    # Désérialisation Python
    $self._python_deserialize() {
        local format="$1"
        local data="$2"
        
        local temp_script
        temp_script="$(mktemp).py"
        
        cat > "$temp_script" << EOF
import sys
import json
import pickle
import yaml

data = """$data"""

try:
    if '$format' == 'json':
        result = json.loads(data)
    elif '$format' == 'yaml':
        import yaml
        result = yaml.safe_load(data)
    elif '$format' == 'pickle':
        result = pickle.loads(bytes.fromhex(data))
    else:
        result = data
    
    print(repr(result))
except Exception as e:
    print(f"Erreur désérialisation: {e}", file=sys.stderr)
    sys.exit(1)
EOF
        
        python3 "$temp_script" 2>&1 && rm -f "$temp_script" || { rm -f "$temp_script"; return 1; }
    }
    
    # Sérialisation Ruby
    $self._ruby_serialize() {
        local format="$1"
        local data="$2"
        
        local temp_script
        temp_script="$(mktemp).rb"
        
        cat > "$temp_script" << EOF
require 'json'
require 'yaml'

data = $data

begin
    case '$format'
    when 'json'
        result = JSON.generate(data)
    when 'yaml'
        result = YAML.dump(data)
    else
        result = data.to_s
    end
    
    puts result
rescue => e
    STDERR.puts "Erreur sérialisation: #{e}"
    exit 1
end
EOF
        
        ruby "$temp_script" 2>&1 && rm -f "$temp_script" || { rm -f "$temp_script"; return 1; }
    }
    
    # Désérialisation Ruby
    $self._ruby_deserialize() {
        local format="$1"
        local data="$2"
        
        local temp_script
        temp_script="$(mktemp).rb"
        
        cat > "$temp_script" << EOF
require 'json'
require 'yaml'

data = <<EOF_DATA
$data
EOF_DATA

begin
    case '$format'
    when 'json'
        result = JSON.parse(data)
    when 'yaml'
        result = YAML.safe_load(data)
    else
        result = data
    end
    
    puts result.inspect
rescue => e
    STDERR.puts "Erreur désérialisation: #{e}"
    exit 1
end
EOF
        
        ruby "$temp_script" 2>&1 && rm -f "$temp_script" || { rm -f "$temp_script"; return 1; }
    }
    
    # Sérialisation JavaScript
    $self._javascript_serialize() {
        local format="$1"
        local data="$2"
        
        local temp_script
        temp_script="$(mktemp).js"
        
        cat > "$temp_script" << EOF
const data = $data;

try {
    let result;
    switch ('$format') {
        case 'json':
            result = JSON.stringify(data);
            break;
        case 'yaml':
            // Simple YAML-like output
            result = JSON.stringify(data, null, 2);
            break;
        default:
            result = String(data);
    }
    
    console.log(result);
} catch (e) {
    console.error('Erreur sérialisation:', e.message);
    process.exit(1);
}
EOF
        
        node "$temp_script" 2>&1 && rm -f "$temp_script" || { rm -f "$temp_script"; return 1; }
    }
    
    # Désérialisation JavaScript
    $self._javascript_deserialize() {
        local format="$1"
        local data="$2"
        
        local temp_script
        temp_script="$(mktemp).js"
        
        cat > "$temp_script" << EOF
const data = `$data`;

try {
    let result;
    switch ('$format') {
        case 'json':
            result = JSON.parse(data);
            break;
        default:
            result = data;
    }
    
    console.log(JSON.stringify(result));
} catch (e) {
    console.error('Erreur désérialisation:', e.message);
    process.exit(1);
}
EOF
        
        node "$temp_script" 2>&1 && rm -f "$temp_script" || { rm -f "$temp_script"; return 1; }
    }
    
    # Sérialisation Bash
    $self._bash_serialize() {
        local format="$1"
        local data="$2"
        
        case "$format" in
            json)
                # Conversion simple en JSON
                echo "{"
                echo "  \"data\": \"$data\""
                echo "}"
                ;;
                
            yaml)
                echo "data: $data"
                ;;
                
            *)
                echo "$data"
                ;;
        esac
    }
    
    # Désérialisation Bash
    $self._bash_deserialize() {
        local format="$1"
        local data="$2"
        
        case "$format" in
            json)
                # Extraction simple depuis JSON
                echo "$data" | grep -o '"data": "[^"]*"' | cut -d'"' -f4
                ;;
                
            yaml)
                echo "$data" | grep "data:" | cut -d: -f2 | xargs
                ;;
                
            *)
                echo "$data"
                ;;
        esac
    }
    
    # Pipeline de conversion automatique
    $self.create_conversion_pipeline() {
        local pipeline_name="$1"
        shift
        local -a steps=("$@")
        
        $self._converters["${pipeline_name}_steps"]="${steps[*]}"
        echo "✓ Pipeline créé: $pipeline_name (${#steps[@]} étapes)"
    }
    
    # Exécution d'un pipeline
    $self.execute_pipeline() {
        local pipeline_name="$1"
        local initial_data="$2"
        
        local steps="${$self._converters[${pipeline_name}_steps]}"
        
        if [[ -z "$steps" ]]; then
            echo "❌ Pipeline introuvable: $pipeline_name" >&2
            return 1
        fi
        
        echo "=== EXÉCUTION PIPELINE: $pipeline_name ==="
        
        local current_data="$initial_data"
        local step_num=1
        
        for step in $steps; do
            echo "Étape $step_num: $step"
            
            # Parsing de l'étape (format: source_format->target_format:language)
            local source_format target_format language
            source_format="$(echo "$step" | cut -d'-' -f1)"
            target_format="$(echo "$step" | cut -d'>' -f1 | cut -d: -f1)"
            language="$(echo "$step" | cut -d: -f2)"
            
            current_data="$($self.convert_data "$source_format" "$target_format" "$language" "$current_data")"
            
            if [[ $? -ne 0 ]]; then
                echo "❌ Échec à l'étape $step_num" >&2
                return 1
            fi
            
            echo "Données après étape $step_num:"
            echo "$current_data"
            echo
            
            ((step_num++))
        done
        
        echo "✓ Pipeline terminé"
        echo "Résultat final:"
        echo "$current_data"
    }
    
    # Validation de format de données
    $self.validate_data_format() {
        local format="$1"
        local language="$2"
        local data="$3"
        
        echo "Validation format: $format ($language)"
        
        # Tentative de désérialisation pour validation
        local test_result
        test_result="$($self.deserialize "$format" "$language" "$data" 2>/dev/null)"
        
        if [[ $? -eq 0 ]]; then
            echo "✓ Format valide"
            return 0
        else
            echo "❌ Format invalide"
            return 1
        fi
    }
    
    # Métriques d'échange de données
    $self.get_exchange_metrics() {
        local format="$1"
        local language="$2"
        local data="$3"
        
        echo "=== MÉTRIQUES ÉCHANGE: $format ($language) ==="
        
        # Taille des données
        local original_size="${#data}"
        echo "Taille originale: $original_size octets"
        
        # Taille sérialisée
        local serialized
        serialized="$($self.serialize "$format" "$language" "$data" 2>/dev/null)"
        local serialized_size="${#serialized}"
        echo "Taille sérialisée: $serialized_size octets"
        
        # Ratio de compression
        if (( serialized_size > 0 )); then
            local ratio
            ratio="$(echo "scale=2; $original_size * 100 / $serialized_size" | bc -l 2>/dev/null || echo "N/A")"
            echo "Ratio de compression: ${ratio}%"
        fi
        
        # Temps de sérialisation
        local start_time end_time duration
        
        start_time="$(date +%s.%N)"
        $self.serialize "$format" "$language" "$data" >/dev/null 2>&1
        end_time="$(date +%s.%N)"
        duration="$(echo "$end_time - $start_time" | bc -l 2>/dev/null || echo "0")"
        echo "Temps sérialisation: ${duration}s"
        
        # Temps de désérialisation
        start_time="$(date +%s.%N)"
        $self.deserialize "$format" "$language" "$serialized" >/dev/null 2>&1
        end_time="$(date +%s.%N)"
        duration="$(echo "$end_time - $start_time" | bc -l 2>/dev/null || echo "0")"
        echo "Temps désérialisation: ${duration}s"
    }
}

# Démonstration du framework d'échange de données
echo "--- Framework d'échange de données ---"

DataInterchange "data_framework"

# Enregistrement des sérialiseurs/désérialiseurs
echo "Configuration des sérialiseurs..."

# JSON
data_framework.register_serializer "json" "python" "json.dumps"
data_framework.register_deserializer "json" "python" "json.loads"

data_framework.register_serializer "json" "ruby" "JSON.generate"
data_framework.register_deserializer "json" "ruby" "JSON.parse"

data_framework.register_serializer "json" "javascript" "JSON.stringify"
data_framework.register_deserializer "json" "javascript" "JSON.parse"

# YAML
data_framework.register_serializer "yaml" "python" "yaml.dump"
data_framework.register_deserializer "yaml" "python" "yaml.safe_load"

data_framework.register_serializer "yaml" "ruby" "YAML.dump"
data_framework.register_deserializer "yaml" "ruby" "YAML.safe_load"

echo
echo "--- Tests de sérialisation/désérialisation ---"

# Données de test
test_data="{\"name\": \"Alice\", \"age\": 30, \"city\": \"Paris\", \"hobbies\": [\"reading\", \"coding\"]}"

echo "Données de test:"
echo "$test_data"

echo
echo "Sérialisation JSON (Python):"
data_framework.serialize "json" "python" "$test_data"

echo
echo "Désérialisation JSON (Ruby):"
json_data='{"users": [{"name": "Bob", "role": "admin"}, {"name": "Charlie", "role": "user"}]}'
data_framework.deserialize "json" "ruby" "$json_data"

echo
echo "--- Conversions entre formats ---"

echo "JSON -> YAML (Python):"
data_framework.convert_data "json" "yaml" "python" "$test_data"

echo
echo "YAML -> JSON (Ruby):"
yaml_data="name: David\nage: 25\nskills:\n  - bash\n  - python\n  - docker"
data_framework.convert_data "yaml" "json" "ruby" "$yaml_data"

echo
echo "--- Pipelines de conversion ---"

# Création d'un pipeline complexe
data_framework.create_conversion_pipeline "complex_conversion" \
    "json->yaml:python" \
    "yaml->json:javascript" \
    "json->yaml:ruby"

echo
echo "Exécution du pipeline complexe:"
data_framework.execute_pipeline "complex_conversion" "$test_data"

echo
echo "--- Validation de formats ---"

echo "Validation JSON:"
data_framework.validate_data_format "json" "python" "$test_data"

echo "Validation JSON invalide:"
invalid_json='{"name": "Test", "incomplete": '
data_framework.validate_data_format "json" "python" "$invalid_json"

echo
echo "--- Métriques d'échange ---"
data_framework.get_exchange_metrics "json" "python" "$test_data"
```

### 2.2 Scripts polyglottes avec shebang dynamique

Exécution de scripts multi-langages avec interprétation automatique :

```bash
#!/bin/bash

# Scripts polyglottes avec shebang dynamique
echo "=== Scripts polyglottes avec shebang dynamique ==="

# Polyglot Script Framework
PolyglotFramework() {
    local self="$1"
    
    declare -A $self._language_detectors
    declare -A $self._executors
    
    # Enregistrement d'un détecteur de langage
    $self.register_language_detector() {
        local language="$1"
        local detector_pattern="$2"
        
        $self._language_detectors["$language"]="$detector_pattern"
        echo "✓ Détecteur enregistré: $language"
    }
    
    # Enregistrement d'un exécuteur
    $self.register_executor() {
        local language="$1"
        local executor_command="$2"
        
        $self._executors["$language"]="$executor_command"
        echo "✓ Exécuteur enregistré: $language"
    }
    
    # Détection automatique du langage
    $self.detect_language() {
        local script_content="$1"
        
        echo "Détection du langage..."
        
        for language in "${!$self._language_detectors[@]}"; do
            local pattern="${$self._language_detectors[$language]}"
            
            if echo "$script_content" | grep -q "$pattern"; then
                echo "✓ Langage détecté: $language"
                echo "$language"
                return 0
            fi
        done
        
        echo "❌ Langage non détecté"
        return 1
    }
    
    # Exécution polyglotte
    $self.execute_polyglot_script() {
        local script_file="$1"
        shift
        local -a args=("$@")
        
        if [[ ! -f "$script_file" ]]; then
            echo "❌ Fichier script introuvable: $script_file" >&2
            return 1
        fi
        
        echo "=== EXÉCUTION POLYGLOTTE: $(basename "$script_file") ==="
        
        # Lecture du contenu du script
        local script_content
        script_content="$(cat "$script_file")"
        
        # Détection du langage
        local detected_language
        detected_language="$($self.detect_language "$script_content")"
        
        if [[ -z "$detected_language" ]]; then
            echo "❌ Impossible de détecter le langage du script" >&2
            return 1
        fi
        
        # Récupération de l'exécuteur
        local executor="${$self._executors[$detected_language]}"
        
        if [[ -z "$executor" ]]; then
            echo "❌ Aucun exécuteur défini pour: $detected_language" >&2
            return 1
        fi
        
        echo "Langage: $detected_language"
        echo "Exécuteur: $executor"
        echo "Arguments: ${args[*]}"
        echo
        
        # Création d'un script temporaire avec le bon shebang
        local temp_script
        temp_script="$(mktemp).${detected_language}"
        
        # Ajout du shebang approprié
        case "$detected_language" in
            python)
                echo "#!/usr/bin/env python3" > "$temp_script"
                ;;
                
            ruby)
                echo "#!/usr/bin/env ruby" > "$temp_script"
                ;;
                
            perl)
                echo "#!/usr/bin/env perl" > "$temp_script"
                ;;
                
            javascript|node)
                echo "#!/usr/bin/env node" > "$temp_script"
                ;;
                
            bash)
                echo "#!/bin/bash" > "$temp_script"
                ;;
                
            *)
                echo "#!/bin/bash" > "$temp_script"
                ;;
        esac
        
        # Ajout du contenu original
        echo "$script_content" >> "$temp_script"
        
        chmod +x "$temp_script"
        
        # Exécution avec les arguments
        echo "Exécution du script..."
        "$temp_script" "${args[@]}"
        local exit_code=$?
        
        # Nettoyage
        rm -f "$temp_script"
        
        echo
        echo "✓ Exécution terminée (code: $exit_code)"
        return $exit_code
    }
    
    # Création d'un script polyglotte
    $self.create_polyglot_script() {
        local output_file="$1"
        local -a languages=("$2")
        local script_content="$3"
        
        echo "Création du script polyglotte: $output_file"
        
        {
            echo "#!/bin/bash"
            echo "# Script polyglotte - Langages supportés: ${languages[*]}"
            echo "#"
            echo "# Ce script peut être exécuté dans différents langages"
            echo "# Utilisez: $0 --language <lang> [args...]"
            echo
            
            echo "# Fonction d'aide"
            echo "show_help() {"
            echo "    echo \"Usage: \$0 [--language LANG] [arguments...]\""
            echo "    echo \"Langages supportés: ${languages[*]}\""
            echo "    echo"
            echo "    echo \"Exemples:\""
            for lang in "${languages[@]}"; do
                echo "    echo \"  \$0 --language $lang arg1 arg2\""
            done
            echo "}"
            echo
            
            echo "# Parsing des arguments"
            echo "LANGUAGE=\"\""
            echo "ARGS=()"
            echo
            
            echo "while [[ \$# -gt 0 ]]; do"
            echo "    case \$1 in"
            echo "        --language|-l)"
            echo "            LANGUAGE=\"\$2\""
            echo "            shift 2"
            echo "            ;;"
            echo "        --help|-h)"
            echo "            show_help"
            echo "            exit 0"
            echo "            ;;"
            echo "        *)"
            echo "            ARGS+=(\"\$1\")"
            echo "            shift"
            echo "            ;;"
            echo "    esac"
            echo "done"
            echo
            
            echo "# Si aucun langage spécifié, détection automatique"
            echo "if [[ -z \"\$LANGUAGE\" ]]; then"
            echo "    echo \"Détection automatique du langage...\""
            echo "    # Ici viendrait la logique de détection"
            echo "    LANGUAGE=\"${languages[0]}\"  # Défaut"
            echo "fi"
            echo
            
            echo "# Exécution selon le langage"
            echo "case \"\$LANGUAGE\" in"
            
            for lang in "${languages[@]}"; do
                echo "    $lang)"
                echo "        echo \"Exécution en $lang...\""
                echo "        exec_$lang \"\${ARGS[@]}\""
                echo "        ;;"
            done
            
            echo "    *)"
            echo "        echo \"Langage non supporté: \$LANGUAGE\" >&2"
            echo "        echo \"Langages disponibles: ${languages[*]}\" >&2"
            echo "        exit 1"
            echo "        ;;"
            echo "esac"
            
            # Ajout des fonctions d'exécution pour chaque langage
            for lang in "${languages[@]}"; do
                echo
                echo "# Fonction d'exécution $lang"
                echo "exec_$lang() {"
                
                case "$lang" in
                    python)
                        echo "    # Code Python intégré"
                        echo "    python3 -c \""
                        echo "$script_content" | sed 's/^/    /'
                        echo "    \" \"\$@\""
                        ;;
                        
                    ruby)
                        echo "    # Code Ruby intégré"
                        echo "    ruby -e \""
                        echo "$script_content" | sed 's/^/    /'
                        echo "    \" \"\$@\""
                        ;;
                        
                    perl)
                        echo "    # Code Perl intégré"
                        echo "    perl -e \""
                        echo "$script_content" | sed 's/^/    /'
                        echo "    \" \"\$@\""
                        ;;
                        
                    javascript)
                        echo "    # Code JavaScript intégré"
                        echo "    node -e \""
                        echo "$script_content" | sed 's/^/    /'
                        echo "    \" \"\$@\""
                        ;;
                        
                    bash)
                        echo "    # Code Bash intégré"
                        echo "    $script_content"
                        ;;
                esac
                
                echo "}"
            done
            
        } > "$output_file"
        
        chmod +x "$output_file"
        echo "✓ Script polyglotte créé: $output_file"
    }
    
    # Conversion d'un script existant en polyglotte
    $self.convert_to_polyglot() {
        local source_file="$1"
        local target_languages="$2"
        local output_file="$3"
        
        if [[ ! -f "$source_file" ]]; then
            echo "❌ Fichier source introuvable: $source_file" >&2
            return 1
        fi
        
        echo "Conversion en polyglotte: $source_file"
        
        local source_content
        source_content="$(cat "$source_file")"
        
        # Détection du langage source
        local source_language
        source_language="$($self.detect_language "$source_content")"
        
        if [[ -z "$source_language" ]]; then
            echo "❌ Impossible de détecter le langage source" >&2
            return 1
        fi
        
        echo "Langage source détecté: $source_language"
        
        # Conversion selon le langage source
        local converted_content=""
        
        case "$source_language" in
            python)
                converted_content="$($self._convert_python_to_polyglot "$source_content" "$target_languages")"
                ;;
                
            ruby)
                converted_content="$($self._convert_ruby_to_polyglot "$source_content" "$target_languages")"
                ;;
                
            bash)
                converted_content="$($self._convert_bash_to_polyglot "$source_content" "$target_languages")"
                ;;
                
            *)
                echo "❌ Conversion depuis $source_language non supportée" >&2
                return 1
                ;;
        esac
        
        if [[ -n "$converted_content" ]]; then
            echo "$converted_content" > "$output_file"
            chmod +x "$output_file"
            echo "✓ Conversion terminée: $output_file"
            return 0
        else
            echo "❌ Échec de la conversion" >&2
            return 1
        fi
    }
    
    # Conversion Python vers polyglotte
    $self._convert_python_to_polyglot() {
        local python_code="$1"
        local target_langs="$2"
        
        # Code polyglotte basique
        cat << EOF
#!/bin/bash
# Script polyglotte (Python primary)

# Exécution Python
python3 -c "
$python_code
" "\$@"
EOF
    }
    
    # Profilage d'exécution polyglotte
    $self.profile_polyglot_execution() {
        local script_file="$1"
        local iterations="${2:-10}"
        
        echo "=== PROFILAGE POLYGLOTTE ==="
        echo "Script: $(basename "$script_file")"
        echo "Itérations: $iterations"
        
        local total_time=0
        local min_time=999999
        local max_time=0
        
        for ((i=1; i<=iterations; i++)); do
            echo -n "Itération $i... "
            
            local start_time
            start_time="$(date +%s.%N)"
            
            $self.execute_polyglot_script "$script_file" "test_arg_$i" >/dev/null 2>&1
            
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
        echo "Résultats du profilage:"
        echo "  Temps moyen: ${avg_time}s"
        echo "  Temps minimum: ${min_time}s"
        echo "  Temps maximum: ${max_time}s"
        echo "  Temps total: ${total_time}s"
    }
}

# Exemples de scripts dans différents langages
python_example_script() {
    cat << 'EOF'
import sys
import json

def main():
    if len(sys.argv) > 1:
        name = sys.argv[1]
    else:
        name = "World"
    
    data = {
        "message": f"Hello, {name}!",
        "language": "Python",
        "timestamp": __import__("time").time()
    }
    
    print(json.dumps(data, indent=2))

if __name__ == "__main__":
    main()
EOF
}

ruby_example_script() {
    cat << 'EOF'
require 'json'
require 'time'

def main
    name = ARGV[0] || "World"
    
    data = {
        "message" => "Hello, #{name}!",
        "language" => "Ruby",
        "timestamp" => Time.now.to_f
    }
    
    puts JSON.pretty_generate(data)
end

main if __FILE__ == $0
EOF
}

javascript_example_script() {
    cat << 'EOF'
const data = {
    message: `Hello, ${process.argv[2] || 'World'}!`,
    language: 'JavaScript',
    timestamp: Date.now() / 1000
};

console.log(JSON.stringify(data, null, 2));
EOF
}

# Démonstration des scripts polyglottes
echo "--- Scripts polyglottes ---"

PolyglotFramework "polyglot_framework"

# Enregistrement des détecteurs de langage
polyglot_framework.register_language_detector "python" "import "
polyglot_framework.register_language_detector "ruby" "require "
polyglot_framework.register_language_detector "perl" "use "
polyglot_framework.register_language_detector "javascript" "const |let |var "
polyglot_framework.register_language_detector "bash" "#!/bin/bash|function "

# Enregistrement des exécuteurs
polyglot_framework.register_executor "python" "python3"
polyglot_framework.register_executor "ruby" "ruby"
polyglot_framework.register_executor "perl" "perl"
polyglot_framework.register_executor "javascript" "node"
polyglot_framework.register_executor "bash" "bash"

echo
echo "--- Création de scripts d'exemple ---"

# Création des scripts de test
python_example_script > /tmp/test_python.py
ruby_example_script > /tmp/test_ruby.rb
javascript_example_script > /tmp/test_javascript.js

echo "Scripts de test créés"

echo
echo "--- Exécution polyglotte ---"

echo "Test Python:"
polyglot_framework.execute_polyglot_script "/tmp/test_python.py" "Alice"

echo
echo "Test Ruby:"
polyglot_framework.execute_polyglot_script "/tmp/test_ruby.rb" "Bob"

echo
echo "Test JavaScript:"
polyglot_framework.execute_polyglot_script "/tmp/test_javascript.js" "Charlie"

echo
echo "--- Création d'un script polyglotte ---"

polyglot_content='
def greet(name):
    return f"Hello, {name} from polyglot script!"

if __name__ == "__main__":
    import sys
    name = sys.argv[1] if len(sys.argv) > 1 else "World"
    print(greet(name))
'

polyglot_framework.create_polyglot_script "/tmp/polyglot_example.sh" "python ruby javascript" "$polyglot_content"

echo
echo "Script polyglotte créé. Test:"
bash /tmp/polyglot_example.sh --language python "Polyglot"

echo
echo "--- Profilage ---"
polyglot_framework.profile_polyglot_execution "/tmp/test_python.py" "5"

# Nettoyage
rm -f /tmp/test_*.py /tmp/test_*.rb /tmp/test_*.js /tmp/polyglot_example.sh
```

## Conclusion : L'orchestration linguistique

L'intégration avec d'autres langages transforme Bash d'un simple shell en un orchestrateur puissant capable d'exploiter les forces de chaque langage de programmation. Cette approche polyglotte permet de choisir le bon outil pour chaque tâche, créant des solutions plus élégantes, performantes et maintenables.

**Points clés à retenir :**

1. **FFI (Foreign Function Interface)** : Frameworks pour appeler des fonctions d'autres langages de manière transparente
2. **Formats d'échange de données** : Systèmes unifiés pour la sérialisation/désérialisation entre langages
3. **Scripts polyglottes** : Exécution automatique selon le langage détecté

Dans le prochain chapitre, nous explorerons les techniques avancées de sécurisation des scripts shell, pour que vos intégrations polyglottes soient non seulement puissantes, mais aussi parfaitement sécurisées.

---

**Exercice pratique :** Créez un système d'intégration complet incluant :
- Framework FFI pour appeler des fonctions Python/Ruby/JavaScript depuis Bash
- Système d'échange de données avec conversion automatique entre JSON, YAML et XML
- Script polyglotte capable de s'exécuter dans au moins 3 langages différents

**Réflexion :** Comment structureriez-vous une architecture où Bash agit comme orchestrateur principal, déléguant les tâches spécialisées aux langages appropriés (traitement de données à Python, scripting web à JavaScript, manipulation de texte à Perl, etc.) ?
