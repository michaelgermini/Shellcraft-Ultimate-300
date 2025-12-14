# Chapitre 53 - Utilisation avancée de awk et sed

## Table des matières
- [Introduction](#introduction)
- [Sed : éditeur de flux avancé](#sed--éditeur-de-flux-avancé)
- [Commandes sed avancées](#commandes-sed-avancées)
- [Scripts sed complexes](#scripts-sed-complexes)
- [Awk : langage de traitement](#awk--langage-de-traitement)
- [Programmation awk avancée](#programmation-awk-avancée)
- [Fonctions awk intégrées](#fonctions-awk-intégrées)
- [Arrays et structures de données](#arrays-et-structures-de-données)
- [Combinaison sed et awk](#combinaison-sed-et-awk)
- [Pipelines complexes](#pipelines-complexes)
- [Performance et optimisation](#performance-et-optimisation)
- [Applications pratiques](#applications-pratiques)
- [Traitement de logs](#traitement-de-logs)
- [Transformation de données](#transformation-de-données)
- [Débogage et bonnes pratiques](#débogage-et-bonnes-pratiques)
- [Alternatives modernes](#alternatives-modernes)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Sed et awk constituent les outils historiques de traitement de texte sous UNIX, offrant des capacités de manipulation de données qui vont bien au-delà des outils modernes. Leur puissance réside dans leur philosophie : traiter les données comme des flux à transformer selon des patterns complexes.

Imaginez sed et awk comme des usines de transformation automatisées : sed agit comme une chaîne de montage qui modifie les produits au fur et à mesure, tandis qu'awk fonctionne comme un atelier spécialisé qui analyse, trie, et transforme les matériaux selon des spécifications précises. Maîtriser ces outils permet de transformer des heures de travail manuel en quelques secondes d'automatisation.

## Sed : éditeur de flux avancé

### Principes fondamentaux

**Fonctionnement de sed** :
```bash
#!/bin/bash
# Sed traite les données ligne par ligne

# Format de base
sed 's/pattern/replacement/' file.txt

# Options courantes
sed -i 's/old/new/' file.txt        # Modification in-place
sed -n 'p' file.txt                  # Supprimer sortie par défaut
sed -e 's/old/new/' -e 's/old2/new2/' file.txt  # Plusieurs commandes
```

**Modes d'exécution** :
```bash
#!/bin/bash
# Modes d'exécution sed

# Mode standard (affiche toutes les lignes)
sed 's/old/new/' file.txt

# Mode silencieux (affiche seulement les lignes modifiées)
sed -n 's/old/new/p' file.txt

# Mode in-place (modifie le fichier)
sed -i 's/old/new/' file.txt

# Mode in-place avec backup
sed -i.bak 's/old/new/' file.txt
```

## Commandes sed avancées

### Substitution avancée

**Substitution avec groupes** :
```bash
#!/bin/bash
# Substitution avec groupes de capture

# Réorganiser les champs
echo "John Doe" | sed -E 's/(\w+) (\w+)/\2, \1/'
# Résultat: Doe, John

# Extraire et réorganiser des dates
echo "2024-01-15" | sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/'
# Résultat: 15/01/2024

# Remplacer avec référence arrière
echo "hello hello" | sed -E 's/(\w+) \1/repeat/'
# Résultat: repeat
```

**Substitution conditionnelle** :
```bash
#!/bin/bash
# Substitution conditionnelle

# Remplacer seulement sur certaines lignes
sed '/^#/! s/old/new/' file.txt  # Remplacer sauf sur les lignes commentées

# Remplacer entre deux patterns
sed '/start/,/end/ s/old/new/' file.txt

# Remplacer avec conditions multiples
sed -E '/pattern1/ s/old/new/; /pattern2/ s/old/new/' file.txt
```

### Commandes de manipulation

**Insertion et suppression** :
```bash
#!/bin/bash
# Commandes de manipulation avancées

# Insérer avant une ligne
sed '/pattern/i\Nouvelle ligne avant' file.txt

# Ajouter après une ligne
sed '/pattern/a\Nouvelle ligne après' file.txt

# Remplacer une ligne entière
sed '/pattern/c\Ligne remplacée' file.txt

# Supprimer des lignes
sed '/pattern/d' file.txt
sed '1,10d' file.txt              # Supprimer lignes 1-10
sed '/^$/d' file.txt               # Supprimer lignes vides

# Supprimer entre deux patterns
sed '/start/,/end/d' file.txt
```

**Transformation de lignes** :
```bash
#!/bin/bash
# Transformation de lignes

# Convertir en majuscules (GNU sed)
sed 's/.*/\U&/' file.txt

# Convertir en minuscules (GNU sed)
sed 's/.*/\L&/' file.txt

# Inverser l'ordre des lignes
sed '1!G;h;$!d' file.txt

# Numéroter les lignes
sed = file.txt | sed 'N;s/\n/\t/'
```

## Scripts sed complexes

### Scripts sed multi-lignes

**Script sed complexe** :
```bash
#!/bin/bash
# Script sed complexe pour nettoyer un fichier

cat > clean.sed << 'EOF'
# Supprimer les lignes vides multiples
/^$/N
/^\n$/d

# Supprimer les espaces en fin de ligne
s/[[:space:]]*$//

# Supprimer les espaces multiples
s/[[:space:]]\{2,\}/ /g

# Ajouter un point final si manquant
s/[^.!?]$/&./

# Capitaliser la première lettre de chaque ligne
s/^./\U&/
EOF

sed -f clean.sed file.txt
```

**Script sed pour transformation de format** :
```bash
#!/bin/bash
# Transformation CSV vers format personnalisé

cat > csv_transform.sed << 'EOF'
# CSV vers format clé=valeur
s/^([^,]+),([^,]+),([^,]+)$/field1=\1\nfield2=\2\nfield3=\3/
EOF

sed -E -f csv_transform.sed data.csv
```

## Awk : langage de traitement

### Structure d'un programme awk

**Format de base** :
```bash
#!/bin/bash
# Structure d'un programme awk

# Format: pattern { action }
awk 'pattern { action }' file.txt

# Exemples
awk '/pattern/ { print }' file.txt           # Lignes correspondantes
awk 'NR > 10 { print }' file.txt             # Après ligne 10
awk '{ print $1, $3 }' file.txt              # Colonnes 1 et 3
awk 'END { print NR }' file.txt              # Nombre total de lignes
```

**Blocs BEGIN et END** :
```bash
#!/bin/bash
# Blocs BEGIN et END

# Initialisation avant traitement
awk 'BEGIN { 
    print "Début du traitement"
    sum = 0
}
{
    sum += $1
}
END {
    print "Total:", sum
    print "Moyenne:", sum/NR
}' file.txt
```

## Programmation awk avancée

### Variables et opérateurs

**Variables awk** :
```bash
#!/bin/bash
# Variables awk

# Variables intégrées
awk '{
    print "Ligne:", NR           # Numéro de ligne
    print "Champ:", NF           # Nombre de champs
    print "Premier champ:", $1
    print "Dernier champ:", $NF
    print "Ligne complète:", $0
}' file.txt

# Variables personnalisées
awk '{
    count++
    total += $1
}
END {
    print "Nombre:", count
    print "Total:", total
    print "Moyenne:", total/count
}' file.txt
```

**Opérateurs avancés** :
```bash
#!/bin/bash
# Opérateurs awk

# Opérateurs arithmétiques
awk '{ print $1 + $2, $1 - $2, $1 * $2, $1 / $2 }' file.txt

# Opérateurs de comparaison
awk '$1 > 100 { print }' file.txt
awk '$1 == "value" { print }' file.txt
awk '$1 ~ /pattern/ { print }' file.txt     # Correspondance regex

# Opérateurs logiques
awk '$1 > 10 && $2 < 20 { print }' file.txt
awk '$1 > 10 || $2 < 20 { print }' file.txt
awk '!($1 == 0) { print }' file.txt
```

### Structures de contrôle

**Boucles et conditions** :
```bash
#!/bin/bash
# Structures de contrôle awk

# Condition if-else
awk '{
    if ($1 > 100) {
        print "Grand:", $1
    } else if ($1 > 50) {
        print "Moyen:", $1
    } else {
        print "Petit:", $1
    }
}' file.txt

# Boucle for
awk '{
    for (i = 1; i <= NF; i++) {
        print "Champ", i ":", $i
    }
}' file.txt

# Boucle while
awk '{
    i = 1
    while (i <= NF) {
        print "Champ", i ":", $i
        i++
    }
}' file.txt

# Switch (GNU awk)
awk '{
    switch ($1) {
        case "A":
            print "Type A"
            break
        case "B":
            print "Type B"
            break
        default:
            print "Autre"
    }
}' file.txt
```

## Fonctions awk intégrées

### Fonctions de chaînes

**Manipulation de chaînes** :
```bash
#!/bin/bash
# Fonctions de chaînes awk

# Longueur
awk '{ print length($1) }' file.txt

# Substring
awk '{ print substr($1, 1, 3) }' file.txt

# Index (position)
awk '{ print index($1, "pattern") }' file.txt

# Split
awk '{ 
    n = split($1, arr, ":")
    for (i = 1; i <= n; i++) print arr[i]
}' file.txt

# Match et remplacement
awk '{ 
    gsub(/old/, "new", $1)
    print $1
}' file.txt

# Toupper et tolower
awk '{ print toupper($1), tolower($2) }' file.txt
```

### Fonctions mathématiques

**Calculs mathématiques** :
```bash
#!/bin/bash
# Fonctions mathématiques awk

# Fonctions de base
awk '{
    print "Sin:", sin($1)
    print "Cos:", cos($1)
    print "Exp:", exp($1)
    print "Log:", log($1)
    print "Sqrt:", sqrt($1)
    print "Int:", int($1)
    print "Rand:", rand()
}' file.txt

# Arrondi
awk '{ print sprintf("%.2f", $1) }' file.txt
```

## Arrays et structures de données

### Arrays associatifs

**Utilisation d'arrays** :
```bash
#!/bin/bash
# Arrays associatifs awk

# Compter les occurrences
awk '{
    count[$1]++
}
END {
    for (key in count) {
        print key, count[key]
    }
}' file.txt

# Groupement et agrégation
awk '{
    sum[$1] += $2
    count[$1]++
}
END {
    for (key in sum) {
        print key, sum[key], sum[key]/count[key]
    }
}' file.txt

# Tri des arrays
awk '{
    arr[$1] = $2
}
END {
    n = asorti(arr, sorted)
    for (i = 1; i <= n; i++) {
        print sorted[i], arr[sorted[i]]
    }
}' file.txt
```

### Arrays multidimensionnels

**Simulation d'arrays multidimensionnels** :
```bash
#!/bin/bash
# Arrays multidimensionnels (simulés)

awk '{
    # Utiliser une clé composite
    key = $1 "," $2
    sum[key] += $3
    count[key]++
}
END {
    for (k in sum) {
        split(k, parts, ",")
        print parts[1], parts[2], sum[k], count[k]
    }
}' file.txt
```

## Combinaison sed et awk

### Pipelines efficaces

**Combinaison optimale** :
```bash
#!/bin/bash
# Combinaison sed et awk

# sed pour nettoyer, awk pour analyser
sed 's/[[:space:]]*$//' file.txt | \
awk '{
    if (NF > 0) {
        print $1, $NF
    }
}'

# sed pour extraire, awk pour transformer
sed -n '/pattern/p' file.txt | \
awk '{ print $2, $1 }'

# sed pour formater, awk pour calculer
sed 's/old/new/g' file.txt | \
awk '{ total += $1 } END { print total }'
```

## Pipelines complexes

### Traitement multi-étapes

**Pipeline complexe** :
```bash
#!/bin/bash
# Pipeline complexe avec sed et awk

# Nettoyer, filtrer, transformer, analyser
cat file.txt | \
sed 's/[[:space:]]*$//' | \           # Nettoyer espaces
sed '/^#/d' | \                        # Supprimer commentaires
awk 'NF > 0 { print }' | \             # Supprimer lignes vides
awk '{
    if ($1 > threshold) {
        print $1, $2
    }
}' threshold=100 | \
awk '{ 
    sum += $1
    count++
} 
END {
    print "Moyenne:", sum/count
}'
```

## Performance et optimisation

### Optimisation sed

**Techniques d'optimisation** :
```bash
#!/bin/bash
# Optimisation sed

# Utiliser -n pour éviter l'affichage inutile
sed -n '/pattern/p' file.txt

# Combiner plusieurs substitutions en une commande
sed -e 's/old1/new1/' -e 's/old2/new2/' file.txt

# Utiliser des patterns spécifiques
sed '/^pattern/ s/old/new/' file.txt  # Plus rapide que s/old/new/

# Éviter les regex complexes quand possible
sed 's/old/new/' file.txt              # Plus rapide que sed 's/.*old.*/new/'
```

### Optimisation awk

**Techniques d'optimisation awk** :
```bash
#!/bin/bash
# Optimisation awk

# Utiliser des patterns pour filtrer tôt
awk '/pattern/ { process }' file.txt   # Plus rapide que awk '{ if (/pattern/) process }'

# Éviter les regex dans les boucles
awk '{
    # Mauvais: regex dans boucle
    for (i = 1; i <= NF; i++) {
        if ($i ~ /pattern/) { ... }
    }
    
    # Bon: regex une fois
    if ($0 ~ /pattern/) {
        for (i = 1; i <= NF; i++) { ... }
    }
}' file.txt

# Utiliser des variables pour éviter les recalculs
awk '{
    field1 = $1
    field2 = $2
    # Utiliser field1 et field2 au lieu de $1 et $2
}' file.txt
```

## Applications pratiques

### Traitement de logs

**Analyse de logs avec awk** :
```bash
#!/bin/bash
# Analyse de logs Apache

# Top 10 IPs
awk '{ print $1 }' access.log | sort | uniq -c | sort -rn | head -10

# Statistiques par code de statut
awk '{
    status[$9]++
}
END {
    for (s in status) {
        print s, status[s]
    }
}' access.log

# Requêtes par heure
awk '{
    split($4, dt, ":")
    hour[dt[2]]++
}
END {
    for (h in hour) {
        print h ":00", hour[h]
    }
}' access.log | sort
```

**Nettoyage de logs avec sed** :
```bash
#!/bin/bash
# Nettoyage de logs

# Supprimer les lignes d'erreur anciennes
sed -i '/ERROR/d' old.log

# Formater les timestamps
sed -E 's/([0-9]{4})-([0-9]{2})-([0-9]{2})/\3\/\2\/\1/' log.txt

# Extraire seulement les erreurs critiques
sed -n '/CRITICAL/p' log.txt
```

## Transformation de données

### CSV vers autres formats

**Transformation CSV** :
```bash
#!/bin/bash
# Transformation CSV

# CSV vers JSON
awk -F',' 'BEGIN {
    print "["
}
{
    if (NR > 1) print ","
    print "  {"
    print "    \"field1\": \"" $1 "\","
    print "    \"field2\": \"" $2 "\""
    print "  }"
}
END {
    print "]"
}' data.csv

# CSV vers format personnalisé
awk -F',' '{
    print "Nom:", $1
    print "Email:", $2
    print "---"
}' contacts.csv
```

### Extraction de données

**Extraction avec awk** :
```bash
#!/bin/bash
# Extraction de données

# Extraire des champs spécifiques
awk '/pattern/ { print $2, $5 }' file.txt

# Extraire entre deux patterns
awk '/start/,/end/ { print }' file.txt

# Extraire avec conditions
awk '$3 > 100 && $4 < 50 { print $1, $2 }' file.txt
```

## Débogage et bonnes pratiques

### Débogage awk

**Techniques de débogage** :
```bash
#!/bin/bash
# Débogage awk

# Mode debug (GNU awk)
awk --debug -f script.awk file.txt

# Ajouter des print de debug
awk '{
    print "DEBUG: Ligne", NR > "/dev/stderr"
    print "DEBUG: Champs", NF > "/dev/stderr"
    # Traitement normal
    print $1
}' file.txt

# Vérifier les variables
awk '{
    print "Variable:", variable > "/dev/stderr"
}' file.txt
```

### Bonnes pratiques

**Recommandations** :
```bash
#!/bin/bash
# Bonnes pratiques

# 1. Toujours tester sur un petit échantillon
head -10 file.txt | sed 's/old/new/'

# 2. Sauvegarder avant modification in-place
cp file.txt file.txt.bak
sed -i 's/old/new/' file.txt

# 3. Utiliser des patterns spécifiques
sed '/^pattern/ s/old/new/' file.txt  # Plus sûr que s/old/new/

# 4. Commenter les scripts complexes
sed -f script.sed file.txt            # Script dans fichier séparé

# 5. Valider les résultats
sed 's/old/new/' file.txt | head -5   # Vérifier avant application

# 6. Utiliser des délimiteurs non-conflictuels
sed 's|/old/path|/new/path|g' file.txt  # Éviter les échappements
```

## Alternatives modernes

### jq pour JSON

**jq vs awk pour JSON** :
```bash
#!/bin/bash
# jq pour JSON

# Avec awk (complexe)
awk '/"key":/ { print }' file.json

# Avec jq (simple)
jq '.key' file.json
jq '.[] | select(.status == "active")' file.json
```

### miller pour CSV

**miller vs awk pour CSV** :
```bash
#!/bin/bash
# miller pour CSV

# Avec awk
awk -F',' '$3 > 100 { print }' data.csv

# Avec miller
mlr --csv filter '$3 > 100' data.csv
mlr --csv stats1 -a sum -f field1 data.csv
```

## Scripts d'automatisation

### Gestionnaire de traitement

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de traitement de texte

set -euo pipefail

process_text() {
    local action="$1"
    local file="$2"
    shift 2
    
    case "$action" in
        clean)
            sed 's/[[:space:]]*$//' "$file" | \
            sed '/^$/d' | \
            sed 's/[[:space:]]\{2,\}/ /g'
            ;;
        extract)
            local pattern="$1"
            awk "/$pattern/ { print }" "$file"
            ;;
        transform)
            local old="$1"
            local new="$2"
            sed "s/$old/$new/g" "$file"
            ;;
        analyze)
            awk '{
                sum += $1
                count++
            }
            END {
                print "Total:", sum
                print "Moyenne:", sum/count
                print "Lignes:", count
            }' "$file"
            ;;
        *)
            echo "Usage: $0 {clean|extract|transform|analyze}"
            exit 1
            ;;
    esac
}

# Utilisation
# process_text clean file.txt
# process_text extract "pattern" file.txt
# process_text transform "old" "new" file.txt
# process_text analyze file.txt
```

## Conclusion

Sed et awk restent des outils essentiels pour le traitement de texte sous Linux, offrant une puissance et une flexibilité inégalées. Leur maîtrise permet de transformer des tâches répétitives en automatisations élégantes et efficaces.

Un script bien écrit utilise sed pour les transformations simples et awk pour les analyses complexes, en combinant les deux outils dans des pipelines optimisés. La compréhension approfondie de ces outils permet de créer des solutions qui traitent efficacement de grandes quantités de données textuelles.

Dans le chapitre suivant, nous explorerons les chapitres avancés Linux (57-70), découvrant des domaines spécialisés qui transforment un administrateur compétent en expert de classe mondiale.
