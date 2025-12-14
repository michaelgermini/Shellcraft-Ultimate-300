# Chapitre 76 - Pipes, redirections et flux multiples

## Table des matières
- [Introduction](#introduction)
- [Redirections de base](#redirections-de-base)
- [Pipes et chaînes de commandes](#pipes-et-chaînes-de-commandes)
- [Gestion des descripteurs de fichiers](#gestion-des-descripteurs-de-fichiers)
- [Redirections avancées](#redirections-avancées)
- [Traitement de flux multiples](#traitement-de-flux-multiples)
- [Debugging des redirections](#debugging-des-redirections)
- [Optimisation et bonnes pratiques](#optimisation-et-bonnes-pratiques)
- [Conclusion](#conclusion)

## Introduction

Les pipes et redirections constituent le système circulatoire de Bash, permettant aux commandes de communiquer entre elles et avec le monde extérieur. Au-delà des simples `>` et `|`, Bash offre un système sophistiqué de manipulation des flux qui transforme les commandes individuelles en puissants pipelines de traitement de données.

Imaginez les redirections comme les valves et canaux d'un système d'irrigation complexe : elles dirigent l'eau (les données) exactement où elle doit aller, permettant à chaque plante (commande) de recevoir exactement ce dont elle a besoin pour prospérer.

## Redirections de base

### Flux standard (stdin, stdout, stderr)

**Les trois flux fondamentaux** :
```bash
# stdin (0) : entrée standard - ce que la commande lit
# stdout (1) : sortie standard - ce que la commande affiche normalement
# stderr (2) : sortie d'erreur - messages d'erreur et diagnostics

# Exemples concrets
echo "Ceci va vers stdout"          # Sortie normale
echo "Ceci est une erreur" >&2      # Sortie d'erreur explicite
cat fichier_inexistant 2>/dev/null  # Erreur redirigée vers /dev/null
```

**Redirections simples** :
```bash
# Redirection de la sortie standard vers un fichier
echo "Hello World" > fichier.txt

# Redirection de la sortie d'erreur
rm fichier_inexistant 2> erreurs.log

# Redirection des deux flux
commande > sortie.txt 2>&1

# Redirection combinée avec append
echo "Nouvelle ligne" >> fichier.txt
echo "Erreur supplémentaire" 2>> erreurs.log
```

### Syntaxe et opérateurs de redirection

**Opérateurs de base** :
```bash
# > : redirection de stdout vers fichier (écrase)
echo "contenu" > fichier.txt

# >> : redirection de stdout vers fichier (append)
echo "ligne 1" > fichier.txt
echo "ligne 2" >> fichier.txt

# < : redirection de fichier vers stdin
wc -l < fichier.txt

# 2> : redirection de stderr vers fichier
find /etc -name "*.conf" 2> erreurs.log

# 2>> : redirection de stderr vers fichier (append)
make 2>> build.log

# &> : redirection des deux flux (stdout et stderr) - syntaxe moderne
commande &> fichier.log

# >& : redirection des deux flux (syntaxe alternative)
commande >& fichier.log
```

**Gestion des fichiers spéciaux** :
```bash
# /dev/null : poubelle universelle
commande > /dev/null 2>&1  # Suppression complète de la sortie

# /dev/zero : source de zéros
dd if=/dev/zero of=fichier_vide bs=1M count=1

# /dev/urandom : source d'aléatoire
head -c 16 /dev/urandom > cle_secrete.bin
```

## Pipes et chaînes de commandes

### Pipe simple (|)

**Principe du pipeline** :
```bash
# La sortie d'une commande devient l'entrée de la suivante
cat /etc/passwd | grep bash | wc -l

# Équivalent sans pipe (beaucoup plus complexe)
temp_file=$(mktemp)
cat /etc/passwd > "$temp_file"
grep bash "$temp_file" > "${temp_file}.2"
wc -l "${temp_file}.2"
rm "$temp_file" "${temp_file}.2"
```

**Chaînes de pipes complexes** :
```bash
# Analyse des logs Apache
cat /var/log/apache2/access.log | \
grep "404" | \
cut -d' ' -f1 | \
sort | \
uniq -c | \
sort -nr | \
head -10

# Résultat : Top 10 des IPs avec erreurs 404
```

**Gestion des erreurs dans les pipes** :
```bash
# Pipefail : arrêt sur première erreur
set -o pipefail

# Sans pipefail (comportement par défaut)
echo "avant" | grep "motif_inexistant" | echo "après"
# Affiche "après" même si grep échoue

# Avec pipefail
set -o pipefail
echo "avant" | grep "motif_inexistant" | echo "après"
# Le script s'arrête si grep échoue

# Gestion fine des erreurs
if ! commande1 | commande2 | commande3; then
    echo "Erreur dans le pipeline"
    exit 1
fi
```

### Pipes nommés (FIFOs)

**Création et utilisation de pipes nommés** :
```bash
# Création d'un pipe nommé
mkfifo mon_pipe

# Dans un terminal : écriture dans le pipe
echo "Hello World" > mon_pipe

# Dans un autre terminal : lecture du pipe
cat mon_pipe

# Pipe nommé persistant
mkfifo /tmp/pipe_communication

# Écriture en arrière-plan
(echo "Message 1"; sleep 1; echo "Message 2") > /tmp/pipe_communication &

# Lecture séquentielle
while read ligne; do
    echo "Reçu : $ligne"
    # Traitement du message
done < /tmp/pipe_communication

# Nettoyage
rm /tmp/pipe_communication
```

**Applications pratiques des pipes nommés** :
```bash
# Communication inter-processus
mkfifo /tmp/request_pipe /tmp/response_pipe

# Serveur en arrière-plan
(
    while true; do
        read request < /tmp/request_pipe
        case "$request" in
            "date") date > /tmp/response_pipe ;;
            "uptime") uptime > /tmp/response_pipe ;;
            "quit") break ;;
            *) echo "Commande inconnue" > /tmp/response_pipe ;;
        esac
    done
) &

# Client
echo "date" > /tmp/request_pipe
read response < /tmp/response_pipe
echo "Réponse : $response"

# Nettoyage
echo "quit" > /tmp/request_pipe
rm /tmp/request_pipe /tmp/response_pipe
```

## Gestion des descripteurs de fichiers

### Descripteurs personnalisés

**Ouverture de descripteurs supplémentaires** :
```bash
# Ouverture de fichiers sur des descripteurs spécifiques
exec 3> fichier_sortie.txt    # Descripteur 3 en écriture
exec 4< fichier_entree.txt   # Descripteur 4 en lecture
exec 5<> fichier_rw.txt      # Descripteur 5 en lecture/écriture

# Utilisation
echo "Message vers fichier" >&3
read ligne <&4
echo "Nouvelle ligne" >&5
read autre_ligne <&5

# Fermeture
exec 3>&-  # Fermeture du descripteur 3
exec 4<&-  # Fermeture du descripteur 4
exec 5>&-  # Fermeture du descripteur 5
```

**Gestion automatique des descripteurs** :
```bash
# Fonction avec descripteurs automatiques
traiter_fichiers() {
    local fichier_entree="$1"
    local fichier_sortie="$2"
    local fichier_erreurs="$3"

    # Ouverture des descripteurs
    exec 3<"$fichier_entree"
    exec 4>"$fichier_sortie"
    exec 5>"$fichier_erreurs"

    # Traitement avec gestion d'erreurs
    while read ligne <&3; do
        if traiter_ligne "$ligne"; then
            echo "$ligne" >&4
        else
            echo "Erreur traitement ligne : $ligne" >&5
        fi
    done

    # Fermeture automatique (via trap)
    trap 'exec 3<&- 4>&- 5>&-' RETURN
}

# Utilisation
traiter_fichiers "entree.txt" "sortie.txt" "erreurs.txt"
```

### Redirections de descripteurs

**Duplication de descripteurs** :
```bash
# command > file : équivalent à command 1> file
# command 2> file : redirection explicite de stderr

# Duplication : faire pointer deux descripteurs vers la même destination
command 2>&1        # stderr vers stdout
command 1>&2        # stdout vers stderr

# Exemples pratiques
# Logs séparés
make 1> build.log 2> build_errors.log

# Tout dans le même fichier
make &> build.log

# Stderr vers stdout, puis stdout vers fichier
make 2>&1 > build.log

# Stdout vers fichier, stderr vers stdout (qui va vers le fichier)
make > build.log 2>&1
```

**Redirections conditionnelles** :
```bash
# Redirection seulement si le fichier n'existe pas
echo "contenu" >| fichier.txt

# Redirection seulement si la commande réussit
commande_succes && echo "OK" > status.log

# Redirection avec gestion d'erreurs
if commande; then
    echo "Succès" > status.log
else
    echo "Échec" > errors.log
fi
```

## Redirections avancées

### Here documents et here strings

**Here documents (<<)** :
```bash
# Document en ligne
cat << EOF
Ceci est un document en ligne.
Il peut contenir plusieurs lignes
et des variables : $HOME
EOF

# Document avec suppression des tabulations
cat <<- EOF
	Ceci est indenté
	Mais les tabulations seront supprimées
	EOF

# Utilisation dans des scripts
configurer_apache() {
    local domaine="$1"
    local port="$2"

    cat > "/etc/apache2/sites-available/${domaine}.conf" << EOF
<VirtualHost *:${port}>
    ServerName ${domaine}
    DocumentRoot /var/www/${domaine}
    ErrorLog \${APACHE_LOG_DIR}/${domaine}_error.log
    CustomLog \${APACHE_LOG_DIR}/${domaine}_access.log combined
</VirtualHost>
EOF
}

configurer_apache "example.com" "8080"
```

**Here strings (<<<)** :
```bash
# Chaîne simple
grep "motif" <<< "Ceci est une chaîne de test avec motif"

# Avec variables
nom="Alice"
echo "Bonjour $nom" | grep -o "Alice"  # Traditionnel
grep -o "Alice" <<< "Bonjour $nom"   # Avec here string

# Traitement de données structurées
donnees="nom:Alice,age:25,email:alice@example.com"
while IFS=: read champ valeur; do
    echo "Champ: $champ, Valeur: $valeur"
done <<< "$donnees"
```

### Redirections multiples et complexes

**Combinaisons avancées** :
```bash
# Redirections multiples
commande > fichier_sortie.txt \
         2> fichier_erreurs.txt \
         < fichier_entree.txt

# Avec pipes et redirections
commande1 < entree.txt \
         2> erreurs.log \
         | commande2 2>&1 \
         | commande3 > resultat.txt

# Redirections dans des sous-shells
(
    echo "Début traitement"
    traiter_donnees > resultats.txt
    echo "Fin traitement"
) 2>&1 | tee traitement.log
```

**Gestion des descripteurs dans les fonctions** :
```bash
# Fonction avec redirection interne
executer_avec_logs() {
    local commande="$1"
    local log_file="${2:-/tmp/commande.log}"

    # Exécution avec redirection complète
    {
        echo "=== Début exécution: $(date) ==="
        echo "Commande: $commande"
        echo "--- Sortie ---"

        # Exécution avec préservation des codes de retour
        if eval "$commande"; then
            echo "--- Succès ---"
            return 0
        else
            local code=$?
            echo "--- Échec (code: $code) ---"
            return $code
        fi
    } > "$log_file" 2>&1
}

# Utilisation
if executer_avec_logs "ls -la /tmp"; then
    echo "Commande exécutée avec succès"
fi
```

## Traitement de flux multiples

### Process substitution

**Substitution de processus (<())** :
```bash
# Lecture depuis la sortie d'une commande
while read ligne; do
    echo "Ligne : $ligne"
done < <(cat /etc/passwd | grep bash)

# Comparaison de fichiers
diff <(sort fichier1.txt) <(sort fichier2.txt)

# Recherche dans des archives
grep "motif" <(tar -tzf archive.tar.gz | grep "\.txt$")

# Statistiques en parallèle
paste <(cut -d: -f1 /etc/passwd) \
      <(cut -d: -f3 /etc/passwd) \
      <(cut -d: -f7 /etc/passwd)
```

**Substitution en écriture (>())** :
```bash
# Écriture vers plusieurs destinations
tee >(gzip > fichier1.gz) \
    >(bzip2 > fichier2.bz2) \
    > fichier3.txt \
    < gros_fichier.txt

# Logging avec timestamp
commande 2> >(sed 's/^/'"$(date '+%Y-%m-%d %H:%M:%S')"' /' >> erreurs.log)

# Envoi vers plusieurs processus
sort fichier.txt | tee >(head -5 > premiers.txt) \
                     >(tail -5 > derniers.txt) \
                     >(wc -l > nombre_lignes.txt) \
                     > tout.txt
```

### Gestion de flux simultanés

**Traitement parallèle de flux** :
```bash
# Lecture de plusieurs fichiers en parallèle
exec 3< fichier1.txt
exec 4< fichier2.txt

while true; do
    read -u 3 ligne1 || break
    read -u 4 ligne2 || break

    # Traitement des paires de lignes
    comparer_lignes "$ligne1" "$ligne2"
done

exec 3<&-
exec 4<&-
```

**Multiplexage de flux** :
```bash
# Fonction de multiplexage
multiplexer_flux() {
    local prefix="$1"
    shift

    # Lancement des commandes en arrière-plan
    local pids=()
    local -a temp_files=()

    for cmd in "$@"; do
        local temp_file=$(mktemp)
        temp_files+=("$temp_file")

        eval "$cmd" > "$temp_file" 2>&1 &
        pids+=($!)
    done

    # Attente et affichage multiplexé
    local i=0
    for pid in "${pids[@]}"; do
        wait "$pid"
        echo "=== $prefix ${prefix:+$i}: ==="
        cat "${temp_files[$i]}"
        echo
        ((i++))
    done

    # Nettoyage
    rm -f "${temp_files[@]}"
}

# Utilisation
multiplexer_flux "Test" \
    "ping -c 3 google.com" \
    "curl -s httpbin.org/ip" \
    "date"
```

## Debugging des redirections

### Outils de diagnostic

**Trace des redirections** :
```bash
# Mode verbose pour voir les redirections
set -x

# Exemple tracé
echo "Test" > fichier.txt
# + echo 'Test'
# + 1> fichier.txt

# Désactivation
set +x

# Fonction de débogage des redirections
debug_redirection() {
    local commande="$1"
    echo "=== Debug redirection ==="
    echo "Commande: $commande"
    echo "Descripteurs ouverts:"
    ls -l /proc/$$/fd/
    echo "Exécution:"
    set -x
    eval "$commande"
    set +x
    echo "=== Fin debug ==="
}

debug_redirection "ls -la > liste.txt 2> erreurs.txt"
```

**Vérification des descripteurs** :
```bash
# Fonction d'inspection des descripteurs
inspect_descriptors() {
    echo "=== Descripteurs ouverts ==="
    for fd in /proc/$$/fd/*; do
        if [[ -e "$fd" ]]; then
            local num=$(basename "$fd")
            local type=$(stat -c %F "$fd" 2>/dev/null || echo "inconnu")
            local target=$(readlink "$fd" 2>/dev/null || echo "N/A")
            printf "FD %2d: %s -> %s\n" "$num" "$type" "$target"
        fi
    done
}

# Avant et après redirections
inspect_descriptors
exec 3> test.txt
exec 4< /etc/passwd
inspect_descriptors
exec 3>&-
exec 4<&-
inspect_descriptors
```

### Gestion des erreurs de redirection

**Validation des redirections** :
```bash
# Fonction de redirection sécurisée
redirect_securise() {
    local source="$1"
    local destination="$2"
    local type="${3:-file}"  # file, directory, device

    # Validation de la source
    case "$type" in
        file)
            if [[ ! -f "$source" ]]; then
                echo "Erreur: fichier source inexistant: $source" >&2
                return 1
            fi
            ;;
        directory)
            if [[ ! -d "$source" ]]; then
                echo "Erreur: répertoire source inexistant: $source" >&2
                return 1
            fi
            ;;
    esac

    # Validation de la destination
    if [[ ! -w "$(dirname "$destination")" ]]; then
        echo "Erreur: pas de permission d'écriture pour: $destination" >&2
        return 1
    fi

    # Exécution de la redirection
    if cp "$source" "$destination"; then
        echo "Redirection réussie: $source -> $destination"
        return 0
    else
        echo "Erreur lors de la redirection" >&2
        return 1
    fi
}

# Utilisation
redirect_securise "/etc/passwd" "/tmp/passwd_backup"
redirect_securise "/var/log" "/tmp/logs_backup" "directory"
```

## Optimisation et bonnes pratiques

### Performance des pipelines

**Réduction du nombre de processus** :
```bash
# Inefficace : plusieurs processus pour une simple transformation
cat fichier.txt | grep "motif" | wc -l

# Plus efficace : utilisation des capacités intégrées
grep -c "motif" fichier.txt

# Optimisation avec xargs
# Mauvais : un grep par fichier
find . -name "*.txt" -exec grep "motif" {} \;

# Bon : traitement par lots
find . -name "*.txt" -print0 | xargs -0 grep "motif"
```

**Buffers et traitement par blocs** :
```bash
# Utilisation de stdbuf pour contrôler les buffers
stdbuf -oL commande_produisant_sortie | commande_consommant_sortie

# Exemple avec tail -f
stdbuf -oL tail -f /var/log/syslog | grep "ERROR"

# Traitement par blocs avec dd
dd if=/dev/zero bs=1M count=100 | gzip | dd of=zeros.gz
```

### Gestion des ressources

**Limitation des ressources dans les pipes** :
```bash
# Timeout sur les pipes
timeout 30s commande_lente | traitement

# Limitation de mémoire avec ulimit
(
    ulimit -v 100000  # 100MB max
    ulimit -t 60      # 60 secondes max
    commande_intensive | traitement
)

# Gestion des signaux dans les pipes
trap 'kill 0' INT TERM  # Tue tous les processus du groupe

commande1 | commande2 | commande3 &
pid_pipeline=$!

# Attente avec possibilité d'interruption
wait $pid_pipeline
```

### Patterns de redirections courants

**Pattern : logging complet** :
```bash
# Fonction de logging avec rotation
log_command() {
    local commande="$1"
    local log_file="${2:-command.log}"
    local max_size="${3:-1048576}"  # 1MB

    # Rotation du log si nécessaire
    if [[ -f "$log_file" ]] && [[ $(stat -f%z "$log_file") -gt $max_size ]]; then
        mv "$log_file" "${log_file}.old"
    fi

    # Exécution avec logging complet
    {
        echo "=== $(date) ==="
        echo "Commande: $commande"
        echo "--- Sortie ---"
        eval "$commande"
        echo "--- Code de retour: $? ---"
        echo
    } >> "$log_file" 2>&1
}

# Utilisation
log_command "ls -la /tmp" "/var/log/commandes.log"
```

**Pattern : traitement avec rollback** :
```bash
# Traitement avec possibilité de rollback
process_with_backup() {
    local fichier="$1"
    local traitement="$2"
    local backup="${fichier}.backup"

    # Création du backup
    cp "$fichier" "$backup"

    # Traitement avec redirection
    if eval "$traitement" < "$fichier" > "${fichier}.tmp" 2> "${fichier}.errors"; then
        # Succès : remplacement du fichier original
        mv "${fichier}.tmp" "$fichier"
        rm -f "${fichier}.errors"
        echo "Traitement réussi"
        return 0
    else
        # Échec : restauration du backup
        mv "$backup" "$fichier"
        echo "Échec du traitement, rollback effectué" >&2
        echo "Erreurs :" >&2
        cat "${fichier}.errors" >&2
        return 1
    fi
}

# Utilisation
process_with_backup "config.txt" "sed 's/old/new/g'"
```

## Conclusion

Les pipes et redirections constituent le langage fondamental de la composition de commandes en Bash. En maîtrisant les descripteurs de fichiers, les redirections complexes, les process substitutions, et les patterns de traitement de flux, vous pouvez créer des pipelines de traitement de données d'une puissance et d'une élégance remarquables.

Comme un maître plombier qui connaît intimement les canalisations de sa maison, le programmeur Bash expérimenté utilise chaque type de redirection à bon escient, créant des systèmes où les données circulent efficacement et où chaque commande reçoit exactement ce dont elle a besoin.

Dans le chapitre suivant, nous explorerons le logging et l'audit avancés, découvrant comment tracer, surveiller et analyser le comportement des scripts pour maintenir la qualité et la fiabilité des systèmes automatisés.
