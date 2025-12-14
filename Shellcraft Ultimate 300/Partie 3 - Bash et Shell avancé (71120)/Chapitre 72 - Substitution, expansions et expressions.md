# Chapitre 72 - Substitution, expansions et expressions

## Table des matières
- [Introduction](#introduction)
- [Expansion de commandes](#expansion-de-commandes)
- [Expansion arithmétique](#expansion-arithmétique)
- [Expansion de paramètres](#expansion-de-paramètres)
- [Expansion de chemins (pathname expansion)](#expansion-de-chemins-pathname-expansion)
- [Expansion d'accolades](#expansion-daccolades)
- [Ordre des expansions](#ordre-des-expansions)
- [Expressions conditionnelles](#expressions-conditionnelles)
- [Expressions régulières](#expressions-régulières)
- [Conclusion](#conclusion)

## Introduction

Les substitutions et expansions constituent le cœur de la puissance de Bash, transformant de simples chaînes de caractères en programmes dynamiques capables de s'adapter à leur environnement. Comprendre l'ordre et les mécanismes des expansions permet d'écrire des scripts qui s'adaptent intelligemment aux données et aux contextes.

Imaginez les expansions comme un orchestre sophistiqué où chaque instrument (type d'expansion) joue sa partition dans un ordre précis, créant une symphonie de traitement de données qui transforme l'entrée brute en sortie raffinée.

## Expansion de commandes

### Substitution de commandes avec backticks

**Syntaxe traditionnelle** :
```bash
# Récupération de la date
date_actuelle=`date`
echo "Nous sommes le $date_actuelle"

# Nombre de fichiers dans le répertoire
nb_fichiers=`ls -1 | wc -l`
echo "Il y a $nb_fichiers fichiers ici"

# Utilisateur connecté
user=`whoami`
echo "Connecté en tant que: $user"
```

**Utilisation dans des calculs** :
```bash
# Espace disque utilisé
usage=`df / | tail -1 | awk '{print $5}'`
echo "Utilisation disque: $usage"

# Dernière modification d'un fichier
modif=`stat -c %y fichier.txt | cut -d'.' -f1`
echo "Dernière modification: $modif"
```

### Substitution moderne avec $()

**Syntaxe recommandée** :
```bash
# Plus lisible et nestable
date_actuelle=$(date +"%Y-%m-%d %H:%M:%S")
echo "Timestamp: $date_actuelle"

# Imbrication de commandes
resultat=$(echo "Scale=2; $(cat nombres.txt | paste -sd+ | bc) / $(wc -l < nombres.txt)" | bc)
echo "Moyenne: $resultat"
```

**Avantages de la syntaxe $()** :
```bash
# Imbrication complexe
chemin=$(basename $(dirname $(pwd)))
echo "Répertoire parent: $chemin"

# Combinaison avec d'autres expansions
fichiers=$(ls $(echo "$HOME"/*.txt | head -3))
echo "Trois premiers fichiers txt: $fichiers"

# Utilisation dans des conditions
if [[ $(id -u) -eq 0 ]]; then
    echo "Exécution en root"
else
    echo "Utilisateur normal"
fi
```

### Gestion des erreurs et statuts

**Capture du statut de sortie** :
```bash
# Capture de la sortie ET du statut
if output=$(commande_risquée 2>&1); then
    echo "Succès: $output"
else
    echo "Erreur: $output"
    exit 1
fi

# Statut seulement
commande_importante
statut=$?
if (( statut != 0 )); then
    echo "La commande a échoué avec le code $statut"
    exit $statut
fi
```

**Substitution dans des pipelines** :
```bash
# Génération dynamique de commandes
commandes=$(cat liste_commandes.txt)
for cmd in $commandes; do
    echo "Exécution: $cmd"
    result=$(eval "$cmd" 2>&1)
    echo "Résultat: $result"
done
```

## Expansion arithmétique

### Syntaxe de base avec $(())

**Opérations arithmétiques** :
```bash
# Calculs simples
resultat=$(( 5 + 3 ))
echo "5 + 3 = $resultat"

# Variables dans les calculs
a=10
b=3
somme=$(( a + b ))
difference=$(( a - b ))
produit=$(( a * b ))
quotient=$(( a / b ))
modulo=$(( a % b ))

echo "Opérations: somme=$somme, différence=$difference, produit=$produit"
```

**Opérateurs avancés** :
```bash
# Puissance
puissance=$(( 2 ** 8 ))  # 256

# Opérateurs bit à bit
et_bit=$(( 12 & 10 ))   # 8  (1100 & 1010 = 1000)
ou_bit=$(( 12 | 10 ))   # 14 (1100 | 1010 = 1110)
xor_bit=$(( 12 ^ 10 ))  # 6  (1100 ^ 1010 = 0110)

# Décalages
gauche=$(( 8 << 2 ))    # 32 (8 * 4)
droite=$(( 32 >> 2 ))   # 8  (32 / 4)
```

### Variables et assignation

**Incrémentation et décrémentation** :
```bash
# Pré et post-incrémentation
compteur=5
echo "Pré-incrémentation: $(( ++compteur ))"  # 6, puis compteur=6
echo "Post-incrémentation: $(( compteur++ ))"  # 6, puis compteur=7

# Même chose pour la décrémentation
valeur=10
echo "Pré-décrémentation: $(( --valeur ))"   # 9
echo "Post-décrémentation: $(( valeur-- ))"  # 9
```

**Assignation composée** :
```bash
# Opérateurs d'assignation
x=5
(( x += 3 ))   # x = x + 3 → 8
(( x *= 2 ))   # x = x * 2 → 16
(( x /= 4 ))   # x = x / 4 → 4
(( x %= 3 ))   # x = x % 3 → 1

echo "Résultat final: $x"
```

### Expressions conditionnelles arithmétiques

**Évaluation booléenne** :
```bash
# Comparaisons
a=5
b=10

# Résultats: 1=vrai, 0=faux
echo "a < b:  $(( a < b ))"   # 1
echo "a > b:  $(( a > b ))"   # 0
echo "a == b: $(( a == b ))"  # 0
echo "a != b: $(( a != b ))"  # 1

# Conditions composées
echo "a < b ET a > 0: $(( (a < b) && (a > 0) ))"  # 1
echo "a > b OU a > 0: $(( (a > b) || (a > 0) ))"  # 1
```

**Utilisation dans des conditions** :
```bash
# Dans des structures de contrôle
if (( $(date +%H) >= 9 && $(date +%H) <= 17 )); then
    echo "Heures de bureau"
else
    echo "Hors heures de bureau"
fi

# Vérification de plages
port=8080
if (( port >= 1024 && port <= 65535 )); then
    echo "Port non privilégié valide"
fi
```

### Fonctions mathématiques

**Calculs avancés** :
```bash
# Utilisation de bc pour les flottants
pi=$(echo "scale=10; 4*a(1)" | bc -l)  # Calcul de π
echo "π ≈ $pi"

# Racine carrée
racine=$(echo "scale=2; sqrt(25)" | bc)
echo "√25 = $racine"

# Fonctions trigonométriques
sin30=$(echo "scale=4; s(30*3.14159/180)" | bc -l)
echo "sin(30°) ≈ $sin30"
```

**Génération de nombres aléatoires** :
```bash
# Nombre aléatoire entre 1 et 100
aleatoire=$(( RANDOM % 100 + 1 ))
echo "Nombre aléatoire: $aleatoire"

# Plusieurs dés
for i in {1..3}; do
    de=$(( RANDOM % 6 + 1 ))
    echo "Dé $i: $de"
done
```

## Expansion de paramètres

### Valeurs par défaut et alternatives

**Opérateur :- (valeur par défaut)** :
```bash
# Si la variable est vide ou non définie
nom=${NOM_UTILISATEUR:-"Anonyme"}
echo "Bienvenue $nom"

# Avec des chemins
config_file=${CONFIG_FILE:-"/etc/default/app.conf"}
log_dir=${LOG_DIR:-"/var/log/app"}

# Dans des fonctions
afficher_message() {
    local niveau=${1:-"info"}
    local message=${2:-"Message par défaut"}
    echo "[$niveau] $message"
}
```

**Opérateur := (assignation par défaut)** :
```bash
# Assigne la valeur si non définie
${CACHE_DIR:="/tmp/cache"}
echo "Cache: $CACHE_DIR"

# Utile pour l'initialisation
${INITIALISE:=false}
if [[ "$INITIALISE" == false ]]; then
    echo "Première exécution - initialisation..."
    INITIALISE=true
fi
```

### Gestion des valeurs non définies

**Opérateur :? (erreur si non défini)** :
```bash
# Lève une erreur si non défini
: ${DATABASE_URL:?"DATABASE_URL doit être défini"}

# Plus explicite
check_required_var() {
    local var_name="$1"
    local var_value="${!var_name}"

    if [[ -z "${var_value+x}" ]]; then
        echo "Erreur: Variable $var_name requise mais non définie"
        exit 1
    fi
}

check_required_var "API_KEY"
check_required_var "SECRET_TOKEN"
```

**Opérateur :+ (valeur si défini)** :
```bash
# Valeur seulement si la variable existe
prefixe=${ENV_PREFIX:+"${ENV_PREFIX}_"}
table_name="${prefixe}utilisateurs"
echo "Table: $table_name"

# Configuration conditionnelle
debug_options=${DEBUG:+"--verbose --trace"}
commande="mon_app $debug_options"
```

### Manipulations de chaînes

**Longueur de chaîne** :
```bash
chaine="Hello World"
longueur=${#chaine}
echo "Longueur: $longueur"  # 11

# Pour les tableaux
declare -a tableau=("un" "deux" "trois")
echo "Éléments: ${#tableau[@]}"
echo "Premier élément: ${#tableau[0]}"
```

**Sous-chaînes** :
```bash
texte="Bash Scripting Avancé"

# Extraction (position, longueur)
debut=${texte:0:4}      # "Bash"
milieu=${texte:5:10}    # "Scripting "
fin=${texte:16}         # "Avancé"

# Suppression depuis le début
sans_debut=${texte#Bash }     # "Scripting Avancé"
sans_motif=${texte##* }       # "Avancé"

# Suppression depuis la fin
sans_fin=${texte% *}          # "Bash Scripting"
sans_extension=${texte%% *}   # "Bash"
```

### Transformations avancées

**Remplacement de motifs** :
```bash
url="http://example.com/api/v1/users"

# Remplacement du premier occurrence
nouvelle_url=${url/v1/v2}           # http://example.com/api/v2/users

# Remplacement de toutes les occurrences
url_https=${url/#http/https}        # https://example.com/api/v1/users

# Remplacement avec motif
chemin="/home/user/documents/file.txt"
backup=${chemin/.txt/.bak}          # /home/user/documents/file.bak
```

**Transformations de casse** :
```bash
texte="Hello World Bash"

# Minuscules
minuscules=${texte,,}                # hello world bash

# Majuscules
majuscules=${texte^^}                # HELLO WORLD BASH

# Première lettre
titre=${texte^}                      # Hello World Bash

# Casse alternative (uniquement en anglais)
alternative=${texte~~}               # hELLO wORLD bASH
```

## Expansion de chemins (pathname expansion)

### Globbing de base

**Motifs simples** :
```bash
# Tous les fichiers .txt
echo *.txt

# Tous les fichiers commençant par "test"
echo test*

# Tous les fichiers avec un chiffre
echo *[0-9]*

# Fichiers avec une lettre spécifique
echo *a*
```

**Classes de caractères** :
```bash
# N'importe quel caractère
echo fichier?.txt    # fichier1.txt, fichierA.txt

# Classes POSIX
echo *[[:upper:]]*  # Fichiers avec majuscules
echo *[[:digit:]]*  # Fichiers avec chiffres
echo *[[:alpha:]]*  # Fichiers avec lettres

# Combinaisons
echo *[[:upper:]][[:lower:]]*
```

### Globbing étendu (extglob)

**Activation des extglob** :
```bash
# Activer les extended globbing
shopt -s extglob

# Désactiver si nécessaire
# shopt -u extglob
```

**Motifs étendus** :
```bash
# Zéro ou plusieurs occurrences
echo !(README).md    # Tous les .md sauf README.md

# Une ou plusieurs occurrences
echo +(fichier).txt  # fichier.txt, fichier1.txt, etc.

# Zéro ou une occurrence
echo ?(test).sh      # test.sh ou .sh (si existe)

# Exactement n occurrences
echo @(un|deux|trois).txt  # un.txt, deux.txt, ou trois.txt
```

**Applications pratiques** :
```bash
# Sauvegarde sélective
cp !(temp|cache)* /backup/

# Nettoyage de logs
rm log.@(err|warn|info).*

# Recherche de fichiers
ls -la /etc/rc.?(S|d)/*
```

### Gestion des cas particuliers

**Échappement et protection** :
```bash
# Désactiver temporairement le globbing
set -f
echo *.txt    # Littéralement "*.txt"
set +f        # Réactiver

# Protection avec des guillemets
echo "*.txt"  # Littéralement "*.txt"
echo '*.txt'  # Littéralement "*.txt"

# Utilisation de noglob
noglob echo *.txt
```

**Globbing récursif** :
```bash
# Avec ** (nécessite shopt -s globstar)
shopt -s globstar

# Tous les .py récursivement
echo **/*.py

# Tous les fichiers dans les sous-répertoires
echo **/*
```

## Expansion d'accolades

### Génération de chaînes

**Listes simples** :
```bash
# Génération de mots
echo {un,deux,trois}          # un deux trois

# Combinaisons
echo fichier{1,2,3}.txt       # fichier1.txt fichier2.txt fichier3.txt

# Préfixes et suffixes
echo {pré-,post-}fixe         # pré-fixe post-fixe
```

**Séquences numériques** :
```bash
# Séquences croissantes
echo {1..10}                  # 1 2 3 4 5 6 7 8 9 10

# Avec pas
echo {1..10..2}               # 1 3 5 7 9

# Lettres
echo {a..z}                   # a b c d ... z
echo {A..F}                   # A B C D E F
```

### Applications pratiques

**Création de répertoires** :
```bash
# Structure de projet
mkdir -p projet/{src,tests,docs,scripts}

# Arborescence complète
mkdir -p app/{controllers/{api,v1},models,views,config}

# Sauvegardes datées
mkdir sauvegarde_$(date +%Y%m%d)_{matin,midi,soir}
```

**Génération de fichiers** :
```bash
# Fichiers de configuration
touch config_{dev,staging,prod}.yml

# Logs rotatifs
touch app.log{,.1,.2,.3,.4,.5}

# Tests unitaires
touch test_{utilisateur,produit,commande}.py
```

### Expansion imbriquée

**Combinaisons complexes** :
```bash
# Produit cartésien
echo {a,b,c}{1,2,3}           # a1 a2 a3 b1 b2 b3 c1 c2 c3

# Avec des chaînes
echo serveur{web,db}{01..03}  # serveurweb01 serveurweb02 ... serveurbd03

# Structures hiérarchiques
echo {dev/{frontend,backend},prod/{api,worker}}/{config,logs}
```

**Utilisation dans des scripts** :
```bash
# Génération de commandes
for env in {dev,staging,prod}; do
    for service in {api,worker,web}; do
        echo "docker build -t myapp/${env}-${service} ."
    done
done

# Tests paramétrés
run_tests() {
    local browser=$1
    local os=$2
    echo "Test de $browser sur $os"
}

# Appel pour toutes les combinaisons
run_tests {firefox,chrome,safari} {linux,mac,windows}
```

## Ordre des expansions

### Séquence d'expansions Bash

**Ordre officiel** :
1. **Expansion d'accolades** `{a,b}` → `a b`
2. **Expansion de chemins** `*.txt` → `fichier1.txt fichier2.txt`
3. **Expansion de commandes** `` `commande` `` ou `$(commande)`
4. **Expansion arithmétique** `$((expression))`
5. **Expansion de paramètres** `${variable}`
6. **Découpage en mots** (word splitting)
7. **Expansion des noms de fichiers** (filename expansion)

**Exemple illustratif** :
```bash
# Ligne originale
echo $HOME/*.txt $(date +%Y) {a,b}

# 1. Expansion d'accolades
echo $HOME/*.txt $(date +%Y) a b

# 2. Expansion de chemins
echo $HOME/fichier1.txt $HOME/fichier2.txt $(date +%Y) a b

# 3. Expansion de commandes
echo $HOME/fichier1.txt $HOME/fichier2.txt 2024 a b

# 4. Expansion arithmétique (aucune ici)
# 5. Expansion de paramètres
echo /home/user/fichier1.txt /home/user/fichier2.txt 2024 a b
```

### Contrôle de l'ordre

**Protection contre les expansions** :
```bash
# Guillemets simples - aucune expansion
echo '$HOME/*.txt $(date)'    # Littéralement: $HOME/*.txt $(date)

# Guillemets doubles - expansions sauf accolades et chemins
echo "$HOME/*.txt $(date)"    # Expansions de variable et commande

# Échappement
echo \$HOME                      # Littéralement: $HOME
echo \*.txt                      # Littéralement: *.txt
```

**Forcer ou inhiber des expansions** :
```bash
# Désactiver temporairement le globbing
set -f
echo *.txt    # Littéralement: *.txt
set +f        # Réactiver

# Désactiver les extglob
shopt -u extglob

# Forcer l'expansion arithmétique
echo $(( 2 + 2 ))    # 4

# Expansion conditionnelle
[[ -f "$fichier" ]] && echo "Fichier existe" || echo "Fichier absent"
```

## Expressions conditionnelles

### Test avec [[ ]]

**Syntaxe moderne** :
```bash
# Comparaisons de chaînes
if [[ "$chaine1" == "$chaine2" ]]; then
    echo "Chaînes identiques"
fi

# Tests numériques (avec -eq, -ne, -lt, -le, -gt, -ge)
if [[ $nombre -gt 10 ]]; then
    echo "Nombre supérieur à 10"
fi

# Tests sur les fichiers
if [[ -f "$fichier" ]]; then
    echo "Fichier régulier"
elif [[ -d "$repertoire" ]]; then
    echo "Répertoire"
elif [[ -x "$executable" ]]; then
    echo "Fichier exécutable"
fi
```

**Opérateurs logiques** :
```bash
# ET logique
if [[ $age -ge 18 && "$pays" == "FR" ]]; then
    echo "Éligible au vote"
fi

# OU logique
if [[ "$role" == "admin" || "$role" == "moderator" ]]; then
    echo "Accès administrateur"
fi

# Négation
if [[ ! -z "$variable" ]]; then
    echo "Variable non vide"
fi
```

### Test avec [ ] (POSIX)

**Syntaxe classique** :
```bash
# Compatible avec sh
if [ "$chaine1" = "$chaine2" ]; then
    echo "Égalité POSIX"
fi

# Attention aux espaces obligatoires
if [ $nombre -gt 10 ]; then  # Erreur si nombre vide!
    echo "Test dangereux"
fi

# Bonne pratique
if [ "$nombre" -gt 10 ]; then
    echo "Test sécurisé"
fi
```

### Expressions régulières

**Test avec =~** :
```bash
# Validation d'email simple
if [[ "$email" =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]; then
    echo "Email valide"
fi

# Extraction de parties
if [[ "$url" =~ ^https?://([^/]+) ]]; then
    domaine="${BASH_REMATCH[1]}"
    echo "Domaine: $domaine"
fi

# Validation de numéro de téléphone français
if [[ "$phone" =~ ^(\+33|0)[1-9][0-9]{8}$ ]]; then
    echo "Numéro français valide"
fi
```

**Utilisation dans des scripts** :
```bash
valider_ip() {
    local ip="$1"
    local regex='^([0-9]{1,3}\.){3}[0-9]{1,3}$'

    if [[ $ip =~ $regex ]]; then
        # Vérifier chaque octet
        IFS='.' read -ra octets <<< "$ip"
        for octet in "${octets[@]}"; do
            if (( octet > 255 )); then
                return 1
            fi
        done
        return 0
    fi
    return 1
}

if valider_ip "192.168.1.1"; then
    echo "IP valide"
fi
```

## Conclusion

Les substitutions et expansions de Bash transforment le shell d'un simple interprète de commandes en un langage de programmation sophistiqué capable de traiter des données complexes. L'ordre précis des expansions permet de combiner différents mécanismes pour créer des scripts adaptatifs et puissants.

Comme un chef cuisinier qui maîtrise l'art de combiner les ingrédients dans le bon ordre, le programmeur Bash expérimenté utilise chaque type d'expansion à bon escient, créant des solutions élégantes qui s'adaptent aux données et aux contextes.

Dans le chapitre suivant, nous explorerons les conditions et boucles complexes, découvrant comment structurer la logique des programmes Bash pour prendre des décisions intelligentes et répéter des opérations efficacement.
