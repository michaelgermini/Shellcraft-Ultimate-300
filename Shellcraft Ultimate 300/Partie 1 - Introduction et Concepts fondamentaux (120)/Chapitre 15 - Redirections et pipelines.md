# Chapitre 15 - Redirections et pipelines

## Table des matières
- [Introduction](#introduction)
- [Descripteurs de fichiers](#descripteurs-de-fichiers)
- [Redirections de base](#redirections-de-base)
- [Redirections avancées](#redirections-avancées)
- [Pipelines : la philosophie UNIX](#pipelines--la-philosophie-unix)
- [Chaînage de commandes](#chaînage-de-commandes)
- [Gestion des erreurs](#gestion-des-erreurs)
- [Redirections nommées](#redirections-nommées)
- [Heredoc et herestrings](#heredoc-et-herestrings)
- [Exemples pratiques](#exemples-pratiques)
- [Débogage des redirections](#débogage-des-redirections)
- [Conclusion](#conclusion)

## Introduction

Les redirections et pipelines représentent le cœur de la puissance du shell UNIX. Ils permettent de connecter les programmes entre eux, de rediriger leurs entrées et sorties, et de construire des chaînes de traitement de données complexes à partir d'outils simples.

Imaginez les redirections comme les tuyaux et vannes d'une usine de traitement d'eau : les programmes sont les stations de traitement, les pipelines sont les canalisations qui les relient, et les redirections sont les vannes qui dirigent le flux vers les bons réservoirs. Une bonne compréhension de ces mécanismes transforme un ensemble d'outils isolés en une chaîne de production automatisée.

## Descripteurs de fichiers

### Les trois descripteurs standard

**STDIN (0)** : Entrée standard
- Par défaut : clavier
- Notation : `0<` ou simplement `<`

**STDOUT (1)** : Sortie standard
- Par défaut : écran
- Notation : `1>` ou simplement `>`

**STDERR (2)** : Sortie d'erreur
- Par défaut : écran
- Notation : `2>`

### Visualisation des descripteurs

**Voir les descripteurs ouverts** :
```bash
# Lister les descripteurs d'un processus
ls -l /proc/$$/fd/

# Output typique :
# lrwx------ 1 user user 64 Jan 15 10:30 0 -> /dev/pts/0
# lrwx------ 1 user user 64 Jan 15 10:30 1 -> /dev/pts/0
# lrwx------ 1 user user 64 Jan 15 10:30 2 -> /dev/pts/0
```

**Création de descripteurs supplémentaires** :
```bash
# Ouvrir un fichier sur le descripteur 3
exec 3< input.txt

# Lire depuis le descripteur 3
read line <&3

# Fermer le descripteur
exec 3<&-
```

## Redirections de base

### Redirection de sortie

**Écraser un fichier** :
```bash
# STDOUT vers fichier (écrase)
echo "Hello World" > output.txt

# STDERR vers fichier
ls /nonexistent 2> error.log

# Les deux séparément
command > output.txt 2> error.txt
```

**Ajouter à un fichier** :
```bash
# STDOUT en ajout (append)
echo "Nouvelle ligne" >> output.txt

# STDERR en ajout
command 2>> error.log
```

### Redirection d'entrée

**Fichier vers STDIN** :
```bash
# Lire depuis un fichier
sort < unsorted.txt

# Même chose avec cat
cat unsorted.txt | sort
```

**Here document** :
```bash
# Entrée multiligne
cat << EOF
Ceci est un document
sur plusieurs lignes
EOF
```

### Redirections combinées

**Sortie et erreur ensemble** :
```bash
# Les deux vers le même fichier
command > output.txt 2>&1

# Version moderne (Bash 4+)
command &> output.txt

# Ajout pour les deux
command >> output.txt 2>&1
```

**Redirections multiples** :
```bash
# STDOUT vers fichier, STDERR à l'écran
command > output.txt

# STDOUT à l'écran, STDERR vers fichier
command 2> error.txt
```

## Redirections avancées

### Redirections conditionnelles

**Noclobber : éviter l'écrasement** :
```bash
# Activer noclobber
set -o noclobber

# Échoue si le fichier existe
echo "test" > existing_file.txt  # Erreur

# Forcer l'écrasement
echo "test" >| existing_file.txt

# Désactiver
set +o noclobber
```

**Redirections avec vérification** :
```bash
# Seulement si le fichier n'existe pas
[ ! -f output.txt ] && echo "Création" > output.txt

# Avec commande moderne
echo "Contenu" > output.txt 2>/dev/null || echo "Erreur d'écriture"
```

### Redirections de processus

**Process substitution** :
```bash
# Commande comme fichier
diff <(ls dir1) <(ls dir2)

# Écriture dans plusieurs commandes
tee >(grep "error" > errors.log) >(grep "warning" > warnings.log) > all.log

# Lecture depuis une commande
while read line; do
    echo "Traitement: $line"
done < <(command)
```

## Pipelines : la philosophie UNIX

### Principe fondamental

**"Do one thing and do it well"** - Doug McIlroy

Chaque outil UNIX :
- Fait une tâche spécifique
- Lit depuis STDIN
- Écrit vers STDOUT
- Traite le texte comme format universel

### Construction de pipelines

**Pipeline simple** :
```bash
# Lister, filtrer, compter
ls -la | grep "\.txt$" | wc -l
```

**Pipeline complexe** :
```bash
# Analyse de logs
tail -f /var/log/apache2/access.log |
grep " 404 " |
cut -d' ' -f1 |
sort |
uniq -c |
sort -nr |
head -10
```

### Gestion des erreurs dans les pipelines

**Pipeline avec gestion d'erreur** :
```bash
# Le pipeline s'arrête à la première erreur
set -e
command1 | command2 | command3

# Pipeline qui continue malgré les erreurs
command1 | command2 || true | command3
```

## Chaînage de commandes

### Opérateurs de contrôle

**Point-virgule (;)** : Séquentiel inconditionnel
```bash
command1 ; command2  # command2 s'exécute toujours
```

**ET logique (&&)** : Conditionnel de succès
```bash
command1 && command2  # command2 seulement si command1 réussit
mkdir /tmp/test && cd /tmp/test
```

**OU logique (||)** : Conditionnel d'échec
```bash
command1 || command2  # command2 seulement si command1 échoue
ping -c1 google.com || echo "Pas de connexion Internet"
```

### Combinaisons avancées

**Chaînage complexe** :
```bash
# Tentative avec fallback
command1 && echo "Succès" || (echo "Échec, tentative alternative" ; command2)

# Installation avec vérification
which wget >/dev/null || (echo "Installation de wget..." ; sudo apt install wget)
```

**Boucles avec chaînage** :
```bash
# Traiter une liste avec gestion d'erreur
for file in *.txt; do
    process_file "$file" && echo "✓ $file traité" || echo "✗ Erreur sur $file"
done
```

## Gestion des erreurs

### Codes de retour et STDERR

**Vérification explicite** :
```bash
#!/bin/bash
command
if [ $? -eq 0 ]; then
    echo "Succès"
else
    echo "Erreur: code $?"
fi
```

**PIPESTATUS pour les pipelines** :
```bash
# Vérifier chaque commande du pipeline
command1 | command2 | command3
echo "Codes de retour: ${PIPESTATUS[@]}"
# Output: Codes de retour: 0 0 0 (tous réussis)
```

### Redirections d'erreur

**Capturer les erreurs séparément** :
```bash
# Erreurs vers un fichier, succès vers un autre
command > success.log 2> error.log

# Erreurs vers la sortie standard
command 2>&1

# Tout vers le même fichier
command &> all.log
```

**Filtrage des erreurs** :
```bash
# Traiter seulement les erreurs
command 2>&1 >/dev/null | grep "ERROR"

# Inverser : erreurs à l'écran, succès vers fichier
command 2>&1 > success.log
```

## Redirections nommées

### Descripteurs personnalisés

**Création de descripteurs nommés** :
```bash
# Créer un fichier temporaire comme descripteur
exec 3<> temp_file.txt

# Écrire dans le descripteur 3
echo "Ligne 1" >&3

# Lire depuis le descripteur 3
read line <&3
echo "Lu: $line"

# Fermer
exec 3>&-
```

### Applications pratiques

**Logs séparés par niveau** :
```bash
#!/bin/bash
exec 3> info.log    # Informations
exec 4> error.log   # Erreurs
exec 5> debug.log   # Debug

echo "Démarrage du programme" >&3
echo "Erreur critique détectée" >&4
echo "Variable x=42" >&5
```

**Communication inter-processus** :
```bash
# Tube nommé temporaire
mkfifo /tmp/my_pipe

# Un processus écrit
echo "Message" > /tmp/my_pipe &

# Un autre lit
cat < /tmp/my_pipe

# Nettoyage
rm /tmp/my_pipe
```

## Heredoc et herestrings

### Here documents

**Syntaxe de base** :
```bash
# Document multiligne
cat << EOF
Ceci est un document
sur plusieurs lignes
avec des variables: $HOME
EOF
```

**Options avancées** :
```bash
# Supprimer l'indentation (Bash 2.05+)
cat <<- EOF
    Ligne indentée
    Autre ligne indentée
EOF

# Supprimer l'expansion des variables
cat << 'EOF'
Variable non expansée: $HOME
Commande non exécutée: $(date)
EOF
```

### Here strings

**Chaîne simple comme entrée** :
```bash
# Au lieu de echo "string" | command
command <<< "string"

# Exemples pratiques
grep "motif" <<< "Ceci est une chaîne de test"
base64 <<< "Hello World"
```

**Comparaison avec les alternatives** :
```bash
# Here string
grep "error" <<< "$log_content"

# Pipeline équivalent
echo "$log_content" | grep "error"

# Here document pour multiligne
grep "error" << EOF
$log_line1
$log_line2
EOF
```

## Exemples pratiques

### Traitement de données

**Analyse de logs** :
```bash
# Extraire et analyser les erreurs
grep "ERROR" /var/log/app.log |
cut -d' ' -f1,3 |
sort |
uniq -c |
sort -nr > error_summary.txt
```

**Traitement CSV** :
```bash
# Calculer des statistiques
tail -n +2 data.csv |  # Sauter l'en-tête
cut -d',' -f3 |        # Colonne des valeurs
sort -n |              # Tri numérique
awk '{sum+=$1} END {print "Moyenne:", sum/NR}' |
tee statistics.txt
```

### Automatisation système

**Sauvegarde avec logging** :
```bash
#!/bin/bash
LOGFILE="/var/log/backup.log"
ERRORFILE="/var/log/backup_errors.log"

rsync -av /home /backup 2>> "$ERRORFILE" >> "$LOGFILE"

if [ $? -eq 0 ]; then
    echo "Sauvegarde réussie" >> "$LOGFILE"
else
    echo "Erreur de sauvegarde" | tee -a "$LOGFILE" >&2
fi
```

**Monitoring réseau** :
```bash
# Ping continu avec timestamp
ping -i 5 google.com 2>&1 |
while read line; do
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $line"
done >> ping_log.txt
```

## Débogage des redirections

### Outils de diagnostic

**set -x : mode debug** :
```bash
#!/bin/bash
set -x  # Activer le debug

echo "Début" > output.txt
ls /nonexistent 2> error.txt
command1 | command2

set +x  # Désactiver
```

**Vérification des descripteurs** :
```bash
# Voir où pointent les descripteurs
ls -l /proc/$$/fd/

# Tester une redirection
echo "test" > /tmp/test && echo "OK" || echo "Erreur d'écriture"
```

### Erreurs courantes

**Problèmes classiques** :
```bash
# ERREUR : espace manquant
echo "test">file.txt  # Marche, mais laid

# ERREUR : oubli du &
command 2>1 error.txt  # Crée un fichier "1"

# CORRECT
command 2> error.txt
command > output.txt 2>&1
```

**Débogage de pipelines** :
```bash
# Tester étape par étape
echo "Étape 1: $(command1)"
echo "Étape 2: $(echo "input" | command2)"

# Utiliser tee pour inspecter
command1 | tee debug1.txt | command2 | tee debug2.txt | command3
```

## Conclusion

Les redirections et pipelines constituent le langage de composition des outils UNIX, permettant de construire des programmes complexes à partir d'éléments simples. Cette approche modulaire est à la base de la puissance et de la flexibilité du shell.

Maîtriser ces mécanismes transforme l'utilisation du terminal d'une série de commandes isolées en une programmation fluide où les données circulent naturellement entre les outils. Les pipelines deviennent alors des programmes visuels, où chaque commande est une étape de transformation des données.

Dans le chapitre suivant, nous explorerons l'historique, l'autocomplétion et les raccourcis, qui permettent d'accélérer et de fiabiliser l'interaction avec le shell.
