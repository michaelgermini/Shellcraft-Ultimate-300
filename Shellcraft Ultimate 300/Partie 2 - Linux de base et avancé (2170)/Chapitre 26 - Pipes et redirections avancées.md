# Chapitre 26 - Pipes et redirections avancées

## Table des matières
- [Introduction](#introduction)
- [Rappels sur les pipes et redirections de base](#rappels-sur-les-pipes-et-redirections-de-base)
- [Pipes nommés (FIFOs)](#pipes-nommés-fifos)
- [Redirections avancées avec exec](#redirections-avancées-avec-exec)
- [Substitution de commandes](#substitution-de-commandes)
- [Substitution de processus](#substitution-de-processus)
- [Redirections conditionnelles](#redirections-conditionnelles)
- [Chaînage complexe de commandes](#chaînage-complexe-de-commandes)
- [Gestion avancée des descripteurs](#gestion-avancée-des-descripteurs)
- [Coproc : Coprocessus](#coproc--coprocessus)
- [Débogage des pipelines](#débogage-des-pipelines)
- [Optimisation des performances](#optimisation-des-performances)
- [Conclusion](#conclusion)

## Introduction

Les pipes et redirections avancées représentent l'essence de la programmation shell sous Linux. Au-delà des simples redirections d'entrée/sortie, ces mécanismes permettent de construire des chaînes de traitement de données complexes, de gérer des communications inter-processus sophistiquées, et de créer des workflows automatisés puissants.

Imaginez les pipes avancés comme un système de plomberie industrielle : les pipes nommés sont des conduites permanentes connectant différents ateliers, les substitutions de processus sont des pompes qui injectent ou extraient des fluides selon les besoins, et les coprocessus sont des sous-stations de traitement qui opèrent en parallèle. Ensemble, ils forment un réseau complexe où les données circulent librement entre les composants.

## Rappels sur les pipes et redirections de base

### Pipes anonymes

**Pipe simple** :
```bash
command1 | command2
# Sortie de command1 → entrée de command2
```

**Pipe avec redirection** :
```bash
command1 2>&1 | command2
# Sortie standard ET erreurs vers command2
```

### Redirections standard

**Fichiers** :
```bash
command > output.txt    # STDOUT vers fichier (écrase)
command >> output.txt   # STDOUT vers fichier (ajoute)
command < input.txt     # STDIN depuis fichier
command 2> error.txt    # STDERR vers fichier
```

**Combinaisons** :
```bash
command > output.txt 2>&1    # Tout vers fichier
command &> output.txt        # Version moderne (Bash 4+)
command 2>&1 | tee output.txt  # Écran ET fichier
```

## Pipes nommés (FIFOs)

### Création et utilisation

**Création d'un pipe nommé** :
```bash
# Créer un pipe nommé
mkfifo mypipe

# Vérifier
ls -l mypipe
# prw-r--r-- 1 user user 0 Jan 15 10:30 mypipe

# Le 'p' indique un pipe nommé
```

**Utilisation basique** :
```bash
# Terminal 1 : écrire dans le pipe
echo "Hello World" > mypipe

# Terminal 2 : lire depuis le pipe
cat < mypipe
# Affiche: Hello World
```

### Communication inter-processus

**Producteur-consommateur** :
```bash
#!/bin/bash
# producteur.sh
PIPE="/tmp/mydata.pipe"

# Créer le pipe s'il n'existe pas
[ -p "$PIPE" ] || mkfifo "$PIPE"

# Écrire des données en continu
while true; do
    date '+%Y-%m-%d %H:%M:%S'
    sleep 1
done > "$PIPE"
```

```bash
#!/bin/bash
# consommateur.sh
PIPE="/tmp/mydata.pipe"

# Lire et traiter les données
while read line; do
    echo "Reçu: $line"
    # Traitement...
done < "$PIPE"
```

**Lancement simultané** :
```bash
# Terminal 1
./consommateur.sh

# Terminal 2
./producteur.sh
```

### Gestion avancée des pipes nommés

**Pipe avec timeout** :
```bash
#!/bin/bash
PIPE="/tmp/timed_pipe"
mkfifo "$PIPE"

# Lire avec timeout
timeout 10 cat < "$PIPE" || echo "Timeout atteint"

# Nettoyer
rm -f "$PIPE"
```

**Pipe bidirectionnel simulé** :
```bash
#!/bin/bash
# Simulation de communication bidirectionnelle
REQUEST_PIPE="/tmp/request.pipe"
RESPONSE_PIPE="/tmp/response.pipe"

mkfifo "$REQUEST_PIPE" "$RESPONSE_PIPE"

# Serveur
(
    while true; do
        read request < "$REQUEST_PIPE"
        response="Réponse à: $request"
        echo "$response" > "$RESPONSE_PIPE"
    done
) &

# Client
echo "Hello Server" > "$REQUEST_PIPE"
read response < "$RESPONSE_PIPE"
echo "Réponse du serveur: $response"

# Nettoyer
kill %1 2>/dev/null
rm -f "$REQUEST_PIPE" "$RESPONSE_PIPE"
```

## Redirections avancées avec exec

### Redirection globale du shell

**Redirection de tout le shell** :
```bash
# Rediriger toute la sortie du shell
exec > logfile.txt 2>&1

# Maintenant toutes les commandes vont dans logfile.txt
echo "Ceci va dans le log"
ls /nonexistent  # Erreur aussi dans le log
date

# Restaurer la sortie normale
exec > /dev/tty
```

**Redirection d'un descripteur spécifique** :
```bash
# Créer un descripteur personnalisé
exec 3> debug.log

# Utiliser le descripteur 3
echo "Debug: $(date)" >&3
echo "Info: Script démarré" >&3

# Fermer le descripteur
exec 3>&-
```

### Gestion des descripteurs multiples

**Ouverture de fichiers multiples** :
```bash
#!/bin/bash
# Script avec logs séparés
LOGFILE="script.log"
ERRORFILE="script.err"
DEBUGFILE="script.debug"

# Ouvrir les descripteurs
exec 3> "$LOGFILE"     # Log général
exec 4> "$ERRORFILE"   # Erreurs
exec 5> "$DEBUGFILE"   # Debug

# Fonctions de logging
log() { echo "$(date): $*" >&3; }
error() { echo "$(date): ERROR: $*" >&4; }
debug() { echo "$(date): DEBUG: $*" >&5; }

# Utilisation
log "Script démarré"
debug "Variable X = $X"
error "Fichier manquant: $MISSING_FILE"

# Fermeture propre
exec 3>&- 4>&- 5>&-
```

### Redirections conditionnelles

**Redirections selon les conditions** :
```bash
#!/bin/bash
LOGFILE="app.log"

# Rediriger seulement si le fichier existe
[ -f "$LOGFILE" ] && exec >> "$LOGFILE" 2>&1

# Ou créer s'il n'existe pas
[ -f "$LOGFILE" ] || touch "$LOGFILE"
exec >> "$LOGFILE" 2>&1

echo "Toutes les sorties suivantes vont dans $LOGFILE"
```

## Substitution de commandes

### Substitution de commande basique

**Syntaxe et utilisation** :
```bash
# $(command) - méthode moderne
files=$(ls *.txt)
echo "Fichiers trouvés: $files"

# `command` - méthode ancienne (obsolète)
files=`ls *.txt`
echo "Fichiers trouvés: $files"
```

**Calculs arithmétiques** :
```bash
# Calcul simple
result=$((5 + 3))
echo "5 + 3 = $result"

# Avec variables
a=10
b=5
echo "a + b = $((a + b))"
echo "a * b = $((a * b))"
```

### Substitution imbriquée

**Commandes dans des commandes** :
```bash
# Nombre de fichiers dans le répertoire le plus rempli
largest_dir=$(ls -d */ | xargs -I{} sh -c 'echo "$(find "{}" -type f | wc -l) {}"' | sort -nr | head -1 | cut -d' ' -f2)

echo "Répertoire avec le plus de fichiers: $largest_dir"
```

**Substitution avec traitement** :
```bash
# Liste des utilisateurs connectés, triée par nombre de sessions
users_connected=$(who | cut -d' ' -f1 | sort | uniq -c | sort -nr | awk '{print $2}')

echo "Utilisateurs connectés:"
echo "$users_connected"
```

### Gestion des erreurs dans la substitution

**Substitution avec gestion d'erreur** :
```bash
# Substitution sûre
user_home=$(getent passwd "$USERNAME" | cut -d: -f6) || user_home="/tmp"

# Avec valeur par défaut
backup_dir=${BACKUP_DIR:-/tmp/backup}

# Substitution conditionnelle
log_level=${LOG_LEVEL:-INFO}
```

## Substitution de processus

### Process substitution basique

**Syntaxe** :
```bash
# <(command) - traiter la sortie comme un fichier
# >(command) - envoyer l'entrée à une commande

# Comparaison de fichiers
diff <(ls dir1) <(ls dir2)

# Traitement parallèle
tar -cf - /source | tee >(md5sum > source.md5) | gzip > archive.tar.gz
```

### Applications pratiques

**Diff de commandes** :
```bash
# Comparer la sortie de deux commandes
diff <(ps aux | grep apache) <(ps aux | grep nginx)

# Comparer des fichiers distants
diff <(ssh server1 cat /etc/passwd) <(ssh server2 cat /etc/passwd)
```

**Traitement en parallèle** :
```bash
# Compresser et calculer le hash simultanément
tar -cf - /bigdata | tee >(sha256sum > archive.sha256) >(md5sum > archive.md5) | xz > archive.tar.xz

# Surveiller et logger simultanément
tail -f /var/log/apache2/access.log | tee >(grep " 404 " > errors.log) >(wc -l > line_count.txt)
```

### Substitution bidirectionnelle

**Communication complexe** :
```bash
# Redirection bidirectionnelle simulée
exec 3> >(grep "ERROR")
exec 4> >(grep "WARNING")

echo "This is an ERROR message" >&3
echo "This is a WARNING message" >&4
echo "This is normal output"

# Fermer
exec 3>&- 4>&-
```

## Redirections conditionnelles

### Redirections selon le succès

**Syntaxe conditionnelle** :
```bash
# command1 && command2 : command2 seulement si command1 réussit
# command1 || command2 : command2 seulement si command1 échoue

# Redirections conditionnelles
successful_backup && mv backup.tar.gz /mnt/backup/ || echo "Échec de la sauvegarde" >&2
```

### Redirections avancées

**Redirections avec vérification** :
```bash
# Créer le répertoire si nécessaire
mkdir -p /path/to/logs && exec >> /path/to/logs/app.log 2>&1

# Rediriger selon l'environnement
if [ -t 1 ]; then
    # Terminal interactif
    command | less
else
    # Redirection ou pipe
    command
fi
```

## Chaînage complexe de commandes

### Opérateurs de contrôle avancés

**Chaînage conditionnel** :
```bash
# Pipeline avec conditions
command1 | command2 && echo "Pipeline réussi" || echo "Erreur dans le pipeline" >&2

# Exécution conditionnelle
[ -f config.ini ] && source config.ini || create_default_config
```

**Groupement de commandes** :
```bash
# Groupe comme une seule commande
{ command1; command2; command3; } | final_command

# Sous-shell (variables isolées)
(command1; command2; export VAR="value"; command3) | final_command
```

### Pipelines complexes

**Traitement de données multi-étapes** :
```bash
# Analyse de logs complexe
tail -f /var/log/apache2/access.log |
grep -v "bot\|crawler" |
awk '{print $1, $7}' |
sort |
uniq -c |
sort -nr |
head -10 |
while read count ip page; do
    hostname=$(dig -x "$ip" +short 2>/dev/null | head -1)
    echo "$count $ip (${hostname:-unknown}) $page"
done
```

**Traitement parallèle avec named pipes** :
```bash
#!/bin/bash
# Traitement parallèle avec pipes nommés

# Créer les pipes
mkfifo pipe1 pipe2 pipe3

# Traitements parallèles
grep "ERROR" < pipe1 > errors.log &
grep "WARNING" < pipe2 > warnings.log &
grep "INFO" < pipe3 > info.log &

# Alimentation des pipes
tail -f /var/log/app.log |
tee pipe1 pipe2 pipe3 >/dev/null

# Nettoyer
rm -f pipe1 pipe2 pipe3
```

## Gestion avancée des descripteurs

### Descripteurs personnalisés

**Ouverture et manipulation** :
```bash
# Ouvrir un fichier en lecture/écriture
exec 3<> myfile.txt

# Lire depuis le descripteur 3
read line <&3
echo "Lu: $line"

# Écrire dans le descripteur 3
echo "Nouvelle ligne" >&3

# Positionnement
exec 3>&-  # Fermer
```

### Redirections de fonctions

**Redirections locales aux fonctions** :
```bash
my_function() {
    # Redirection locale à la fonction
    echo "Ceci va dans la sortie normale"
} >&2  # Toute la fonction vers stderr

my_function  # Sortie vers stderr
```

### Gestion des descripteurs ouverts

**Inspection et débogage** :
```bash
# Lister les descripteurs ouverts du shell
ls -l /proc/$$/fd/

# Compter les descripteurs ouverts
ls /proc/$$/fd/ | wc -l

# Vérifier si un descripteur est ouvert
if [ -e /proc/$$/fd/3 ]; then
    echo "Descripteur 3 ouvert"
fi
```

## Coproc : Coprocessus

### Principe des coprocessus

**Coprocessus** : Processus en arrière-plan avec communication bidirectionnelle
```bash
# Lancement d'un coprocessus
coproc MYPROC { bc -l; }

# MYPROC_PID : PID du coprocessus
# MYPROC[0] : Descripteur en lecture (sortie du coproc)
# MYPROC[1] : Descripteur en écriture (entrée du coproc)
```

### Utilisation basique

**Communication simple** :
```bash
#!/bin/bash
# Calculatrice interactive en arrière-plan

coproc CALC { bc -l; }

# Envoyer des calculs
echo "5 + 3" >&${CALC[1]}
read result <&${CALC[0]}
echo "5 + 3 = $result"

echo "sqrt(16)" >&${CALC[1]}
read result <&${CALC[0]}
echo "sqrt(16) = $result"

# Fermer
kill $CALC_PID 2>/dev/null
```

### Applications avancées

**Filtrage en temps réel** :
```bash
#!/bin/bash
# Filtrage interactif

coproc FILTER { grep "ERROR"; }

# Simulation de log stream
(
    echo "INFO: Application started"
    sleep 1
    echo "ERROR: Database connection failed"
    sleep 1
    echo "WARNING: High memory usage"
    sleep 1
    echo "ERROR: File not found"
) >&${FILTER[1]} &

# Lire les erreurs filtrées
while read line <&${FILTER[0]}; do
    echo "ALERTE: $line"
done

kill $FILTER_PID 2>/dev/null
```

**Coprocessus multiples** :
```bash
#!/bin/bash
# Traitement parallèle avec coprocessus

coproc SUM { awk '{sum += $1} END {print sum}'; }
coproc COUNT { wc -l; }

# Traiter des données
cat data.txt | tee >&${SUM[1]} >&${COUNT[1]} >/dev/null

# Récupérer les résultats
read total_sum <&${SUM[0]}
read line_count <&${COUNT[0]}

echo "Somme: $total_sum"
echo "Nombre de lignes: $line_count"

# Nettoyer
kill $SUM_PID $COUNT_PID 2>/dev/null
```

## Débogage des pipelines

### Outils de diagnostic

**set -x : Mode debug** :
```bash
#!/bin/bash
set -x  # Tracer l'exécution

# Pipeline avec debug
echo "Hello" | tr 'a-z' 'A-Z' | rev

set +x  # Fin du debug
```

**PIPESTATUS : Codes de retour** :
```bash
# Vérifier chaque commande du pipeline
echo "test" | grep "pattern" | wc -l
echo "Codes de retour: ${PIPESTATUS[@]}"
# Codes de retour: 0 1 0
# grep n'a pas trouvé "pattern" (code 1)
```

### Isolation des problèmes

**Test étape par étape** :
```bash
#!/bin/bash
# Débogage de pipeline

debug_pipeline() {
    local pipeline="$1"
    
    echo "=== DÉBOGAGE DU PIPELINE ==="
    echo "Pipeline: $pipeline"
    
    # Tester chaque commande individuellement
    IFS='|' read -ra commands <<< "$pipeline"
    
    local i=1
    for cmd in "${commands[@]}"; do
        cmd=$(echo "$cmd" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
        echo "--- Commande $i: $cmd ---"
        
        # Exécuter et capturer sortie/erreur
        if eval "$cmd" 2>&1; then
            echo "✓ Réussi (code: $?)"
        else
            echo "✗ Échoué (code: $?)"
        fi
        
        ((i++))
    done
}

# Utilisation
debug_pipeline "ls -la | grep '\.txt$' | sort -k5 -nr"
```

### Monitoring des ressources

**Profilage des pipelines** :
```bash
# Temps d'exécution
time (command1 | command2 | command3)

# Utilisation mémoire
/usr/bin/time -v command1 | command2

# Profiling avec strace
strace -c command1 | command2
```

## Optimisation des performances

### Éviter les sous-shells inutiles

**Sous-shells coûteux** :
```bash
# INEFFICACE : sous-shell à chaque itération
for file in $(ls *.txt); do
    process_file "$file"
done

# EFFICACE : éviter le sous-shell
for file in *.txt; do
    process_file "$file"
done
```

**Optimisation des redirections** :
```bash
# INEFFICACE : cat inutile
cat fichier.txt | grep "pattern"

# EFFICACE : redirection directe
grep "pattern" < fichier.txt

# INEFFICACE : multiple passes
cat fichier.txt | grep "pattern" | wc -l

# EFFICACE : grep seul
grep -c "pattern" fichier.txt
```

### Buffering et performance

**Contrôle du buffering** :
```bash
# Forcer le flush immédiat (debug)
stdbuf -oL command1 | stdbuf -iL command2

# Buffering par défaut (performance)
command1 | command2
```

**Parallélisation** :
```bash
# Traitement parallèle avec xargs
find . -name "*.jpg" -print0 | xargs -0 -P 4 -n 10 convert -resize 50% {}

# Coprocessus pour parallélisation
coproc WORKER1 { process_type_A; }
coproc WORKER2 { process_type_B; }

# Distribuer le travail
echo "data_A" >&${WORKER1[1]}
echo "data_B" >&${WORKER2[1]}
```

### Optimisations avancées

**Utilisation de RAM disks** :
```bash
# Créer un RAM disk temporaire
mkdir /tmp/ramdisk
mount -t tmpfs -o size=1G tmpfs /tmp/ramdisk

# Utiliser pour les données temporaires
command1 | tee /tmp/ramdisk/temp_data | command2

# Nettoyer
umount /tmp/ramdisk
```

**Optimisation des gros volumes** :
```bash
# Utiliser des pipes nommés pour éviter les fichiers temporaires
mkfifo /tmp/data_pipe

# Producteur
command1 > /tmp/data_pipe &

# Consommateur
command2 < /tmp/data_pipe

# Nettoyer
rm /tmp/data_pipe
```

## Conclusion

Les pipes et redirections avancées transforment le shell d'un simple interpréteur de commandes en un langage de programmation sophistiqué pour le traitement de données. Les pipes nommés permettent des communications inter-processus persistantes, les substitutions de processus offrent une flexibilité incroyable, et les coprocessus permettent un parallélisme élégant.

La maîtrise de ces mécanismes permet de construire des chaînes de traitement complexes où les données circulent efficacement entre les composants, créant des workflows automatisés qui rivalisent avec les programmes compilés en termes de puissance et d'efficacité.

L'optimisation de ces pipelines - éviter les sous-shells inutiles, contrôler le buffering, utiliser le parallélisme - transforme des scripts fonctionnels en solutions performantes capables de traiter des volumes de données importants.

Dans le chapitre suivant, nous explorerons la recherche avancée avec find, locate, grep, et expressions régulières, complétant notre arsenal d'outils pour la manipulation de fichiers et de données sous Linux.

