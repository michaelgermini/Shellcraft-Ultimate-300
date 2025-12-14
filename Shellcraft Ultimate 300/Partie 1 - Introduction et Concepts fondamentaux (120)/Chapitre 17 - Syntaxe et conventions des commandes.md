# Chapitre 17 - Syntaxe et conventions des commandes

## Table des matières
- [Introduction](#introduction)
- [Structure générale d'une commande](#structure-générale-dune-commande)
- [Arguments et paramètres](#arguments-et-paramètres)
- [Options courtes et longues](#options-courtes-et-longues)
- [Arguments obligatoires vs optionnels](#arguments-obligatoires-vs-optionnels)
- [Types de données des arguments](#types-de-données-des-arguments)
- [Conventions de nommage](#conventions-de-nommage)
- [Messages d'aide et documentation](#messages-daide-et-documentation)
- [Codes de retour](#codes-de-retour)
- [Gestion des erreurs](#gestion-des-erreurs)
- [Exemples de commandes canoniques](#exemples-de-commandes-canoniques)
- [Conclusion](#conclusion)

## Introduction

La syntaxe et les conventions des commandes représentent le langage structuré qui permet de communiquer avec les programmes UNIX. Comprendre ces règles transforme l'usage du terminal d'une suite de tentatives en une conversation précise et prévisible avec le système.

Imaginez les commandes UNIX comme les verbes d'une langue étrangère sophistiquée : chaque commande a sa conjugaison propre (options), ses compléments d'objet (arguments), et suit des règles grammaticales strictes (conventions POSIX). Un locuteur maîtrisant ces règles peut exprimer des idées complexes avec élégance et précision.

## Structure générale d'une commande

### Anatomie d'une commande

**Forme canonique** :
```
nom_commande [options] [arguments]
```

**Éléments constitutifs** :
- **nom_commande** : programme à exécuter (requis)
- **[options]** : modificateurs du comportement (optionnel)
- **[arguments]** : données d'entrée (optionnel)

**Exemples concrets** :
```bash
# Commande simple
ls

# Avec options
ls -l -a

# Avec arguments
cp source.txt destination.txt

# Combinaison complète
tar -czf archive.tar.gz -C /path/to/directory .
```

### Ordre des éléments

**Règles de syntaxe** :
1. Le nom de commande vient toujours en premier
2. Les options suivent immédiatement, souvent groupées
3. Les arguments viennent en dernier
4. L'ordre des options n'a généralement pas d'importance
5. L'ordre des arguments peut être significatif

**Exceptions notables** :
```bash
# find : options et arguments entremêlés
find /path -name "*.txt" -type f -exec rm {} \;

# dd : syntaxe spécifique
dd if=input.txt of=output.txt bs=1M
```

## Arguments et paramètres

### Arguments positionnels

**Arguments ordonnés** :
```bash
# cp source destination
cp fichier1.txt fichier2.txt

# mv source destination
mv ancien_nom.txt nouveau_nom.txt

# Arguments multiples
mkdir dir1 dir2 dir3
```

**Signification par position** :
- **Premier argument** : souvent la source/entrée
- **Deuxième argument** : souvent la destination/sortie
- **Arguments suivants** : éléments supplémentaires

### Arguments spéciaux

**Traits d'union pour les arguments** :
```bash
# Fichier commençant par -
cat ./-filename-with-dash

# Fin des options
grep -- -pattern file.txt

# Arguments vides
touch ""  # Crée un fichier nommé ""
```

**Expansion des arguments** :
```bash
# Globbing
ls *.txt

# Variables
echo "$HOME"

# Commandes
echo "$(date)"
```

## Options courtes et longues

### Options courtes (style POSIX)

**Syntaxe** :
```bash
# Option simple
ls -l

# Options groupées
ls -la

# Option avec argument
tar -czf archive.tar.gz source/
```

**Conventions communes** :
```bash
-h, --help     # Aide
-v, --verbose  # Mode verbeux
-q, --quiet    # Mode silencieux
-f, --force    # Forcer l'action
-i, --interactive  # Mode interactif
-r, --recursive    # Récursif
```

### Options longues (GNU)

**Syntaxe étendue** :
```bash
# Option longue
ls --all

# Option longue avec argument
du --human-readable --max-depth=1

# Mélange possible
ls -l --all --human-readable
```

**Avantages des options longues** :
- Plus lisibles : `--verbose` vs `-v`
- Auto-documentées
- Moins d'ambiguïtés
- Support de la complétion

### Gestion des options

**Parsing des options en script** :
```bash
#!/bin/bash
# Exemple de parsing d'options
while getopts "hf:v" opt; do
    case $opt in
        h) echo "Usage: $0 [-h] [-f file] [-v]"; exit 0 ;;
        f) filename="$OPTARG" ;;
        v) verbose=true ;;
        *) echo "Option invalide: -$OPTARG" >&2; exit 1 ;;
    esac
done

shift $((OPTIND-1))  # Supprimer les options parsées
```

## Arguments obligatoires vs optionnels

### Arguments requis

**Arguments essentiels** :
```bash
# cp nécessite source ET destination
cp source.txt destination.txt  # ✓ Correct
cp source.txt                 # ✗ Erreur : destination manquante

# mkdir nécessite au moins un nom
mkdir nouveau_dossier         # ✓ Correct
mkdir                         # ✗ Erreur : argument manquant
```

**Arguments par défaut** :
```bash
# Certaines commandes ont des valeurs par défaut
# head sans argument : 10 premières lignes
head fichier.txt    # Équivalent à head -n 10 fichier.txt
```

### Arguments optionnels

**Arguments supplémentaires** :
```bash
# ls peut prendre un chemin (optionnel)
ls               # Liste le répertoire courant
ls /tmp         # Liste /tmp
ls *.txt        # Liste les fichiers .txt

# grep peut prendre plusieurs fichiers
grep "pattern" fichier1.txt fichier2.txt
```

### Validation des arguments

**Vérification en script** :
```bash
#!/bin/bash
# Validation des arguments
if [ $# -lt 1 ]; then
    echo "Usage: $0 <fichier>" >&2
    exit 1
fi

if [ ! -f "$1" ]; then
    echo "Erreur: '$1' n'est pas un fichier" >&2
    exit 1
fi

# Traitement...
echo "Traitement de $1"
```

## Types de données des arguments

### Chaînes de caractères

**Arguments textuels** :
```bash
# Citations pour les espaces
echo "Hello World"
echo 'Hello $USER'  # Pas d'expansion

# Échappement
echo "Prix: \$5.99"
```

### Nombres

**Arguments numériques** :
```bash
# Lignes à afficher
head -n 50 fichier.txt
tail -n 20 fichier.txt

# Permissions
chmod 755 script.sh

# Taille de bloc
dd bs=1M if=input of=output
```

### Chemins et fichiers

**Arguments de chemins** :
```bash
# Chemins absolus
cp /home/user/file.txt /tmp/

# Chemins relatifs
cp ./file.txt ../backup/

# Globs
rm *.tmp
ls **/*.txt  # Zsh : récursif
```

### Expressions régulières

**Patterns complexes** :
```bash
# grep avec regex
grep "^[0-9]\{3\}-[0-9]\{3\}-[0-9]\{4\}$" phone_numbers.txt

# find avec patterns
find . -name "*.log" -mtime -7
```

## Conventions de nommage

### Noms de commandes

**Conventions POSIX** :
- Minuscules uniquement
- Mots séparés par des tirets : `long-option`
- Acronymes courants : `ls`, `cp`, `mv`, `rm`
- Noms descriptifs : `useradd`, `groupdel`

**Exemples canoniques** :
```bash
# Actions : create, delete, modify, show
useradd, userdel, usermod, usershow

# Préfixes : get, set, list
getent, setquota, listusers

# Contextes : sys, net, disk
sysctl, netstat, diskusage
```

### Options standardisées

**Options courtes réservées** :
```bash
-h  # help
-v  # verbose
-q  # quiet
-f  # force/file
-i  # interactive
-r  # recursive/reverse
-n  # number/no-action
-a  # all
-l  # long/list
-s  # silent/size
-t  # time/type
```

### Arguments standardisés

**Formats communs** :
```bash
# Fichiers : chemin absolu ou relatif
command /path/to/file

# Utilisateurs : nom ou UID
usermod alice
usermod 1000

# Tailles : avec unités
dd bs=1M
du -h  # Human readable

# Dates : multiples formats
find . -mtime -7  # 7 derniers jours
```

## Messages d'aide et documentation

### Aide intégrée

**Option help standard** :
```bash
command --help
command -h

# Exemples courants
ls --help
grep --help
tar --help
```

**Structure typique d'un message d'aide** :
```
Usage: command [OPTIONS] [ARGUMENTS]

DESCRIPTION
    Description brève de la commande

OPTIONS
    -o, --option[=ARG]    Description de l'option
    -v, --verbose         Mode verbeux
    -h, --help           Afficher cette aide

EXAMPLES
    command --option valeur argument
    command -o valeur arg

SEE ALSO
    related_command(1), other_command(1)
```

### Pages de manuel

**Accès aux man pages** :
```bash
# Page principale
man command

# Sections spécifiques
man 1 command    # Commandes utilisateur
man 5 passwd     # Format des fichiers
man 8 mount      # Administration système
```

**Navigation dans les man pages** :
```bash
# Recherche
/pattern

# Ligne suivante
n

# Quitter
q

# Aide
h
```

## Codes de retour

### Signification des codes

**Convention UNIX** :
- **0** : Succès
- **1** : Erreur générale
- **2** : Usage incorrect (mauvais arguments)
- **126** : Commande trouvée mais non exécutable
- **127** : Commande non trouvée
- **128+N** : Signal N a terminé le processus

**Exemples pratiques** :
```bash
# Test explicite
grep "pattern" file.txt
echo "Code retour: $?"

# Utilisation en script
if grep -q "error" logfile.txt; then
    echo "Erreurs trouvées"
fi
```

### Gestion des codes de retour

**Capture et traitement** :
```bash
#!/bin/bash
# Gestion fine des erreurs
command
retour=$?

case $retour in
    0) echo "Succès" ;;
    1) echo "Erreur générale" ;;
    2) echo "Mauvaise utilisation" ;;
    127) echo "Commande introuvable" ;;
    *) echo "Erreur inconnue: $retour" ;;
esac
```

**Codes spécifiques par commande** :
```bash
# diff
# 0: fichiers identiques
# 1: fichiers différents
# 2: erreur

# grep
# 0: motif trouvé
# 1: motif non trouvé
# 2: erreur
```

## Gestion des erreurs

### Messages d'erreur standardisés

**Format cohérent** :
```
command: error message
command: fichier: error message
command: option: error message
```

**Exemples courants** :
```bash
ls /nonexistent
# ls: cannot access '/nonexistent': No such file or directory

cp file1 file2
# cp: 'file1' and 'file2' are the same file

grep "pattern" nonexistent.txt
# grep: nonexistent.txt: No such file or directory
```

### Gestion des erreurs en script

**Stratégies de gestion** :
```bash
# Arrêt sur erreur
set -e

# Traiter les erreurs
command || echo "Commande échouée, continuation..."

# Gestion fine
if ! command; then
    echo "Erreur détectée, nettoyage..."
    cleanup
    exit 1
fi
```

**Redirections d'erreur** :
```bash
# Erreurs vers fichier
command 2> errors.log

# Erreurs vers /dev/null
command 2>/dev/null

# Erreurs avec succès
command 2>&1  # Tout ensemble
```

## Exemples de commandes canoniques

### Commandes de manipulation de fichiers

**cp - copie** :
```bash
cp [OPTIONS] SOURCE DEST
cp [OPTIONS] SOURCE... DIRECTORY

# Options courantes
-i  # Interactif
-r  # Récursif
-v  # Verbeux
-p  # Préserver attributs
```

**mv - déplacement** :
```bash
mv [OPTIONS] SOURCE DEST
mv [OPTIONS] SOURCE... DIRECTORY

# Options courantes
-i  # Interactif
-v  # Verbeux
```

### Commandes de traitement de texte

**grep - recherche** :
```bash
grep [OPTIONS] PATTERN [FILE...]

# Options courantes
-i  # Insensible à la casse
-v  # Inversion (lignes sans le motif)
-n  # Numéros de ligne
-r  # Récursif
-l  # Noms de fichiers seulement
```

**sed - édition de flux** :
```bash
sed [OPTIONS] SCRIPT [FILE...]
sed [OPTIONS] -e SCRIPT... [FILE...]

# Scripts courants
's/old/new/'           # Substitution
'/pattern/d'           # Suppression
'/pattern/p'           # Impression
```

### Commandes système

**ps - processus** :
```bash
ps [OPTIONS]

# Options courantes
-a  # Tous les processus
-u  # Format utilisateur
-x  # Processus sans terminal
-e  # Tous les processus
-f  # Format complet
```

**find - recherche de fichiers** :
```bash
find [PATH...] [EXPRESSION]

# Expressions courantes
-name PATTERN     # Par nom
-type TYPE        # Par type (f,d,l)
-mtime DAYS       # Par date de modification
-size SIZE        # Par taille
-exec COMMAND ;   # Exécuter commande
```

## Conclusion

La syntaxe et les conventions des commandes UNIX forment un langage cohérent et prévisible qui permet d'exprimer des intentions complexes avec précision. Comprendre ces règles transforme l'usage du terminal d'une expérience empirique en une communication structurée avec le système.

Les conventions établies - des options courtes aux codes de retour, en passant par la structure argument-option - créent un cadre stable qui permet l'automatisation et la composition de commandes. Cette uniformité est l'une des forces majeures de l'écosystème UNIX.

Dans le chapitre suivant, nous explorerons l'utilisation des manuels et de la documentation, qui fournit le contexte et les détails nécessaires à la maîtrise de chaque commande.
