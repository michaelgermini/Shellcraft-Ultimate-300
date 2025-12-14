# Chapitre 28 - Expressions régulières

## Table des matières
- [Introduction](#introduction)
- [Concepts fondamentaux](#concepts-fondamentaux)
- [Expressions régulières de base (BRE)](#expressions-régulières-de-base-bre)
- [Expressions régulières étendues (ERE)](#expressions-régulières-étendues-ere)
- [Expressions régulières compatibles Perl (PCRE)](#expressions-régulières-compatibles-perl-pcre)
- [Classes de caractères et ensembles](#classes-de-caractères-et-ensembles)
- [Ancres et limites](#ancres-et-limites)
- [Quantificateurs et répétitions](#quantificateurs-et-répétitions)
- [Groupes et références arrières](#groupes-et-références-arrières)
- [Alternatives et choix](#alternatives-et-choix)
- [Assertions lookahead/lookbehind](#assertions-lookaheadlookbehind)
- [Optimisation et performance](#optimisation-et-performance)
- [Débogage et test](#débogage-et-test)
- [Applications pratiques](#applications-pratiques)
- [Outils avancés](#outils-avancés)
- [Conclusion](#conclusion)

## Introduction

Les expressions régulières constituent l'un des outils les plus puissants pour le traitement de texte dans l'écosystème UNIX/Linux. Elles permettent de décrire des patterns complexes de texte avec une concision remarquable, transformant des tâches de recherche et de manipulation de texte qui seraient autrement laborieuses en opérations élégantes et efficaces.

Imaginez les expressions régulières comme le langage de reconnaissance de formes pour le texte : au lieu de chercher des chaînes exactes, vous décrivez des motifs abstraits qui peuvent matcher une infinité de variations. C'est comme donner une description à un artiste plutôt qu'un dessin précis - "un animal avec des rayures, quatre pattes et une crinière" au lieu de "un tigre spécifique".

## Concepts fondamentaux

### Qu'est-ce qu'une regex ?

**Définition** : Une expression régulière est une chaîne de caractères qui décrit un ensemble de chaînes de caractères possibles.

**Exemples simples** :
```bash
# Motif pour des adresses email
[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}

# Ce motif décrit :
# - Lettres, chiffres, points, tirets, pourcents, plus
# - Suivi d'un @
# - Suivi de lettres, chiffres, points, tirets
# - Suivi d'un point
# - Suivi de 2+ lettres
```

### Métacaractères vs littéraux

**Caractères littéraux** : Matchent exactement eux-mêmes
```bash
# "cat" matche exactement "cat"
echo "cat" | grep "cat"  # Trouvé
echo "dog" | grep "cat"  # Non trouvé
```

**Métacaractères** : Ont une signification spéciale
```bash
# . matche n'importe quel caractère
echo "cat" | grep "c.t"   # Trouvé (c + n'importe quoi + t)
echo "cot" | grep "c.t"   # Trouvé
echo "ct" | grep "c.t"    # Non trouvé
```

### Greediness (gourmandise)

**Quantificateurs gourmands** : Matchent le plus possible
```bash
# .* matche tout ce qu'il peut
echo "abc123def" | grep -o ".*[0-9].*"
# Résultat : abc123def (tout)

# .*? serait non-gourmand (si supporté)
```

## Expressions régulières de base (BRE)

### grep standard (sans -E)

**Métacaractères BRE** :
```bash
# . : n'importe quel caractère
grep "c.t" fichier.txt

# * : zéro ou plus du caractère précédent
grep "ab*c" fichier.txt  # abc, abbc, abbbc, ac

# ^ : début de ligne
grep "^ERROR" log.txt

# $ : fin de ligne
grep "END$" log.txt

# [abc] : un des caractères listés
grep "[aeiou]" fichier.txt

# [^abc] : aucun des caractères listés
grep "[^0-9]" fichier.txt

# \(abc\) : groupe (pour référence arrière)
grep "\(abc\)\1" fichier.txt  # abcabc
```

### Échappement dans BRE

**Caractères spéciaux à échapper** :
```bash
# + et ? ne sont pas spéciaux en BRE
grep "ab\+c" fichier.txt  # Recherche littérale "ab+c"

# Pour utiliser + et ? en BRE, les échapper
grep "ab\+c" fichier.txt   # ab+c (littéral)
grep "ab\\+c" fichier.txt  # ab suivi de un ou plus c
```

### Exemples BRE courants

**Validation de formats** :
```bash
# Numéros de téléphone US (BRE)
grep '([0-9]\{3\}) [0-9]\{3\}-[0-9]\{4\}' contacts.txt

# Adresses IP simples
grep '[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}' access.log
```

## Expressions régulières étendues (ERE)

### grep avec -E ou egrep

**Métacaractères ERE supplémentaires** :
```bash
# + : un ou plus
grep -E "ab+c" fichier.txt  # abc, abbc, abbbc (pas ac)

# ? : zéro ou un
grep -E "colou?r" fichier.txt  # color ou colour

# | : alternative
grep -E "error|warning|info" log.txt

# () : groupement sans échappement
grep -E "(abc)+" fichier.txt  # abc, abcabc, etc.

# {} : quantificateurs sans échappement
grep -E "[0-9]{3}-[0-9]{3}-[0-9]{4}" phones.txt
```

### Comparaison BRE vs ERE

**Même motif, syntaxes différentes** :
```bash
# BRE (avec échappement)
grep '^\([0-9]\{3\}\) [0-9]\{3\}-[0-9]\{4\}$' phones.txt

# ERE (naturel)
grep -E '^\([0-9]{3}\) [0-9]{3}-[0-9]{4}$' phones.txt

# Ou encore plus simple
grep -E '^\(\d{3}\) \d{3}-\d{4}$' phones.txt
```

### Avantages d'ERE

**Lisibilité** :
- Pas d'échappement pour + ? | ( )
- Syntaxe plus intuitive
- Moins d'erreurs de frappe

**Performance** :
- Souvent plus rapide que BRE
- Meilleur support des optimisations

## Expressions régulières compatibles Perl (PCRE)

### grep avec -P

**Fonctionnalités avancées** :
```bash
# \d \w \s : classes abrégées
grep -P '\d{3}-\d{3}-\d{4}' phones.txt

# \b : frontière de mot
grep -P '\berror\b' log.txt

# Lookahead positif
grep -P 'error(?=.*critical)' log.txt

# Lookbehind négatif
grep -P '(?<!not )error' log.txt
```

### Classes abrégées PCRE

**Raccourcis pratiques** :
```bash
# \d : chiffre [0-9]
# \D : non-chiffre [^0-9]
# \w : caractère de mot [a-zA-Z0-9_]
# \W : non-caractère de mot [^a-zA-Z0-9_]
# \s : espace [\t\n\f\r ]
# \S : non-espace [^\t\n\f\r ]

# Exemple d'utilisation
grep -P '\w+@\w+\.\w+' emails.txt
```

## Classes de caractères et ensembles

### Classes de base

**Ensembles personnalisés** :
```bash
# Voyelles
grep "[aeiou]" fichier.txt

# Consonnes
grep "[^aeiou]" fichier.txt  # Tout sauf voyelles

# Plage de caractères
grep "[a-z]" fichier.txt     # Lettres minuscules
grep "[A-Z]" fichier.txt     # Lettres majuscules
grep "[0-9]" fichier.txt     # Chiffres

# Combinaisons
grep "[a-zA-Z0-9_]" fichier.txt  # Comme \w en PCRE
```

### Classes POSIX

**Classes standardisées** :
```bash
# [:alnum:] : alphanumérique
grep "[[:alnum:]]" fichier.txt

# [:alpha:] : alphabétique
grep "[[:alpha:]]" fichier.txt

# [:digit:] : chiffres
grep "[[:digit:]]" fichier.txt

# [:space:] : espaces
grep "[[:space:]]" fichier.txt

# [:upper:] : majuscules
grep "[[:upper:]]" fichier.txt

# Combinaisons
grep "[[:alpha:]][[:digit:]]*" fichier.txt  # Lettre suivie de chiffres optionnels
```

### Ensembles complexes

**Ensembles avec exclusions** :
```bash
# Tout sauf chiffres et voyelles
grep "[^0-9aeiou]" fichier.txt

# Caractères de ponctuation
grep "[[:punct:]]" fichier.txt

# Caractères de contrôle
grep "[[:cntrl:]]" fichier.txt
```

## Ancres et limites

### Ancres de ligne

**Position dans la ligne** :
```bash
# ^ : début de ligne
grep "^ERROR" log.txt

# $ : fin de ligne
grep "END$" log.txt

# Ligne complète
grep "^exactement cette ligne$" fichier.txt

# Lignes vides
grep "^$" fichier.txt
```

### Frontières de mots

**Limites de mots (PCRE)** :
```bash
# \b : frontière de mot
grep -P '\berror\b' log.txt     # "error" mais pas "terror"
grep -P '\b\d{3}\b' fichier.txt # Nombre de 3 chiffres exactement

# \< \> : frontières de mots (vim/sed)
grep -E '\<error\>' log.txt
```

### Ancres avancées

**Début/fin de chaîne** :
```bash
# \A : début absolu (PCRE)
# \Z : fin absolue (PCRE)

# Multiligne
grep -P '(?m)^ERROR' log.txt  # ^ matche début de ligne en multiligne
```

## Quantificateurs et répétitions

### Quantificateurs de base

**Répétitions** :
```bash
# * : zéro ou plus
grep -E "ab*c" fichier.txt  # ac, abc, abbc, abbbc...

# + : un ou plus
grep -E "ab+c" fichier.txt  # abc, abbc, abbbc... (pas ac)

# ? : zéro ou un
grep -E "colou?r" fichier.txt  # color, colour

# {n} : exactement n fois
grep -E "\d{3}" fichier.txt  # exactement 3 chiffres

# {n,} : n fois ou plus
grep -E "\d{3,}" fichier.txt  # 3 chiffres ou plus

# {n,m} : entre n et m fois
grep -E "\d{2,4}" fichier.txt  # 2 à 4 chiffres
```

### Quantificateurs gourmands vs non-gourmands

**Gourmands (par défaut)** :
```bash
echo "abc123def456" | grep -o ".*[0-9]"
# abc123def456 (tout, car .* prend tout)

echo "abc123def456" | grep -o "[0-9]+.*[0-9]+"
# 123def456 (le + gourmand prend le plus possible)
```

**Non-gourmands (PCRE)** :
```bash
echo "abc123def456" | grep -Po ".*?[0-9]"
# abc123 (s'arrête au premier chiffre)

echo "abc123def456" | grep -Po "[0-9]+?.*[0-9]+?"
# 123def456 (mais avec quantification non-gourmande)
```

## Groupes et références arrières

### Groupes capturants

**Capture et référence** :
```bash
# Groupe capturant
grep -E "([0-9]{3})-([0-9]{3})-([0-9]{4})" phones.txt

# Référence arrière (\1, \2, \3)
grep -E "(abc)\1" fichier.txt  # abcabc

# Référence dans la même regex
grep -E "([a-z])\1" fichier.txt  # aa, bb, cc...
```

### Groupes non-capturants

**Groupement sans capture (PCRE)** :
```bash
# (?:pattern) : groupe non-capturant
grep -P '(?:error|warning|info): .+' log.txt

# Plus efficace si on n'a pas besoin de référencer
```

### Références arrières complexes

**Réutilisation de groupes** :
```bash
# Mots répétés
grep -P '(\b\w+\b) .*\1' fichier.txt

# Balises HTML simples
grep -P '<([a-z]+)>.*</\1>' fichier.html

# Numéros répétés
grep -P '([0-9]{3})-[0-9]{3}-\1' fichier.txt  # Même préfixe
```

## Alternatives et choix

### Alternatives simples

**OU logique** :
```bash
# |
grep -E 'error|warning|critical' log.txt

# Plusieurs alternatives
grep -E 'cat|dog|bird' animaux.txt

# Avec groupes
grep -E '(error|warning) message' log.txt
```

### Alternatives complexes

**Choix conditionnels** :
```bash
# Mots avec ou sans suffixe
grep -E 'color|colour' fichier.txt

# Formats de date
grep -E '\d{1,2}/\d{1,2}/\d{4}|\d{4}-\d{1,2}-\d{1,2}' dates.txt
```

## Assertions lookahead/lookbehind

### Lookahead positif

**Condition devant** :
```bash
# Mot suivi de "error"
grep -P 'error(?=.*critical)' log.txt

# Mot avant un nombre
grep -P '\w+(?=\d)' fichier.txt

# Validation de mot de passe (au moins 8 caractères)
grep -P '(?=^.{8,}$)' passwords.txt
```

### Lookahead négatif

**Condition non présente devant** :
```bash
# "error" non suivi de "debug"
grep -P 'error(?!.*debug)' log.txt

# Mot de passe sans chiffres consécutifs
grep -P '(?!.*\d{3})' passwords.txt
```

### Lookbehind

**Condition derrière** :
```bash
# Mot précédé de "not"
grep -P '(?<=not )error' log.txt

# Prix en euros
grep -P '\d+(?<=€)' fichier.txt
```

### Lookbehind négatif

**Condition non présente derrière** :
```bash
# "error" non précédé de "not"
grep -P '(?<!not )error' log.txt
```

## Optimisation et performance

### Patterns inefficaces

**Backtracking coûteux** :
```bash
# Problématique : nested quantifiers
grep -E "(a+)+" fichier.txt  # Beaucoup de backtracking

# Mieux : évite les imbriquations
grep -E "a+" fichier.txt
```

### Optimisations pratiques

**Ancres pour limiter la recherche** :
```bash
# Mauvais : scanne toute la ligne
grep "ERROR" huge_file.txt

# Bon : utilise les ancres
grep "^ERROR" huge_file.txt
```

**Classes de caractères spécifiques** :
```bash
# Mieux que .*
grep -P '^[^:]*: ERROR:' log.txt
```

### Benchmarking

**Comparaison de performances** :
```bash
# Test de différentes regex
time grep -E 'pattern1' huge_file.txt
time grep -P 'pattern2' huge_file.txt

# Avec profiling
perf record -g grep -P 'complex_pattern' huge_file.txt
perf report
```

## Débogage et test

### Outils de test

**Regex testing online** :
- regex101.com
- regexr.com
- regextester.com

**Test en ligne de commande** :
```bash
# Test simple
echo "test string" | grep -E "pattern"

# Avec explication (si disponible)
echo "test string" | pcregrep -d "pattern"
```

### Debugging de regex

**Étapes de débogage** :
```bash
# 1. Test avec chaîne simple
echo "abc123" | grep -E "[a-z]+\d+"

# 2. Ajout progressif
echo "abc123" | grep -E "[a-z]+"      # Fonctionne
echo "abc123" | grep -E "[a-z]+\d+"   # Fonctionne

# 3. Test avec données réelles
head -5 data.txt | grep -E "[a-z]+\d+"

# 4. Vérification des échappements
grep -E '\$[0-9]+\.[0-9]{2}' prices.txt
```

### Erreurs courantes

**Problèmes fréquents** :
```bash
# Oubli d'échappement
grep "(abc)" fichier.txt     # Recherche littérale (abc)

# Quantificateur sur rien
grep "a*" fichier.txt        # a* signifie "zéro ou plus a"

# Ancres oubliées
grep "ERROR" fichier.txt     # Trouve "ERROR" n'importe où
grep "^ERROR" fichier.txt    # Trouve "ERROR" en début de ligne
```

## Applications pratiques

### Traitement de logs

**Extraction d'informations** :
```bash
# Adresses IP des erreurs
grep -P '^(\d{1,3}\.){3}\d{1,3}' error.log | cut -d' ' -f1 | sort | uniq -c | sort -nr

# Timestamps des erreurs critiques
grep -P '^(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) .* CRITICAL' app.log

# Utilisateurs avec échecs répétés
grep "Failed password" /var/log/auth.log | grep -oP '(?<=for )[^ ]+' | sort | uniq -c | sort -nr | head -10
```

### Validation de données

**Formats courants** :
```bash
# Emails
grep -P '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$' emails.txt

# URLs
grep -P 'https?://[^\s]+' webpage.html

# Numéros de carte de crédit (format basique)
grep -P '\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b' payments.txt
```

### Manipulation de texte

**Reformatage** :
```bash
# CSV vers format personnalisé
grep -P '^([^,]+),([^,]+),([^,]+)$' data.csv | 
sed -E 's/^([^,]+),([^,]+),([^,]+)$/Nom: \1\nPrénom: \2\nEmail: \3\n---/'

# Extraction de données structurées
grep -P 'name="([^"]+)" value="([^"]+)"' form.html |
sed -E 's/.*name="([^"]+)".*value="([^"]+)".*/\1 = \2/'
```

## Outils avancés

### sed avec regex

**Substitution avancée** :
```bash
# Remplacement avec groupes
echo "John Doe" | sed -E 's/(\w+) (\w+)/\2, \1/'

# Modification conditionnelle
sed -E '/^#/! s/error/warning/g' fichier.txt
```

### awk avec regex

**Traitement par champs** :
```bash
# Lignes où le deuxième champ est un email
awk '$2 ~ /^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$/' data.txt

# Somme des valeurs valides
awk '$3 ~ /^[0-9]+\.[0-9]{2}$/ {sum += $3} END {print "Total:", sum}' transactions.txt
```

### Perl one-liners

**Scripts rapides** :
```bash
# Recherche avec Perl regex
perl -ne 'print if /\berror\b/i' logfile.txt

# Remplacement complexe
perl -pe 's/(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})/$4.$3.$2.$1/g' ips.txt
```

### ripgrep (rg)

**Recherche ultra-rapide** :
```bash
# Recherche récursive avec regex
rg '\berror\b' /var/log/

# Avec contexte
rg -C 3 'function \w+' src/

# Recherche par type de fichier
rg --type py 'class \w+' .
```

## Conclusion

Les expressions régulières constituent un langage de programmation à part entière pour la reconnaissance et la manipulation de patterns dans le texte. De la simple recherche de mots à l'extraction complexe de données structurées, elles offrent une puissance et une flexibilité incomparables.

La clé de la maîtrise réside dans la compréhension progressive : commencer par les bases (BRE), maîtriser les extensions (ERE), puis explorer les fonctionnalités avancées (PCRE). Chaque niveau ajoute de la puissance tout en maintenant la lisibilité.

L'optimisation des regex - éviter le backtracking coûteux, utiliser les ancres appropriées, choisir les bons quantificateurs - transforme des patterns lents en recherches ultra-efficaces.

Dans le chapitre suivant, nous explorerons la gestion des logs et la surveillance système, appliquant ces techniques de regex à l'analyse des journaux système et à la surveillance proactive de l'infrastructure.

