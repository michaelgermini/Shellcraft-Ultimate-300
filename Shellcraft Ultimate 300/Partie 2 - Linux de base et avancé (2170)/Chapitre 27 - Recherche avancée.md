# Chapitre 27 - Recherche avancée

## Table des matières
- [Introduction](#introduction)
- [Commande find : recherche de fichiers avancée](#commande-find--recherche-de-fichiers-avancée)
- [locate et updatedb : recherche rapide](#locate-et-updatedb--recherche-rapide)
- [Recherche de contenu avec grep](#recherche-de-contenu-avec-grep)
- [ripgrep (rg) : grep moderne et rapide](#ripgrep-rg--grep-moderne-et-rapide)
- [ag (The Silver Searcher) : alternative rapide](#ag-the-silver-searcher--alternative-rapide)
- [fd : find moderne](#fd--find-moderne)
- [Outils de recherche spécialisés](#outils-de-recherche-spécialisés)
- [Recherche interactive avec fzf](#recherche-interactive-avec-fzf)
- [Recherche dans les archives et fichiers compressés](#recherche-dans-les-archives-et-fichiers-compressés)
- [Optimisation des performances de recherche](#optimisation-des-performances-de-recherche)
- [Recherche dans les bases de données et logs](#recherche-dans-les-bases-de-données-et-logs)
- [Automatisation des recherches complexes](#automatisation-des-recherches-complexes)
- [Conclusion](#conclusion)

## Introduction

La recherche avancée constitue l'un des piliers de l'efficacité au terminal. Que vous cherchiez des fichiers spécifiques, du contenu dans des documents, ou des patterns complexes dans des logs système, maîtriser les outils de recherche vous permet de naviguer dans des volumes de données importants avec précision et rapidité.

Imaginez la recherche comme un détective numérique : vous disposez d'indices variés (noms de fichiers, contenu, dates, permissions) et devez combiner ces éléments pour trouver exactement ce que vous cherchez, parfois dans des collections gigantesques. Les outils modernes comme ripgrep, fd, et fzf ont révolutionné la recherche, offrant des performances exceptionnelles et des interfaces intuitives.

## Commande find : recherche de fichiers avancée

### Syntaxe et options de base

**find : l'outil universel** :
```bash
#!/bin/bash
# Utilisation de base de find

# Recherche simple par nom
find . -name "*.txt"

# Recherche insensible à la casse
find . -iname "*.TXT"

# Recherche par type
find . -type f  # Fichiers seulement
find . -type d  # Répertoires seulement
find . -type l  # Liens symboliques

# Recherche par taille
find . -size +100M      # Plus de 100MB
find . -size -1k        # Moins de 1KB
find . -size 500c       # Exactement 500 bytes

# Recherche par date
find . -mtime -7        # Modifiés dans les 7 derniers jours
find . -mtime +30       # Modifiés il y a plus de 30 jours
find . -mmin -60        # Modifiés dans la dernière heure
find . -newer reference.txt  # Plus récents que reference.txt
```

### Critères de recherche avancés

**Combinaisons complexes** :
```bash
#!/bin/bash
# Recherches complexes avec find

# Fichiers Python modifiés récemment et plus grands que 1KB
find . -name "*.py" -mtime -7 -size +1k

# Fichiers appartenant à un utilisateur spécifique
find /home -user alice -type f

# Fichiers avec permissions spécifiques
find . -type f -perm 644
find . -type f -perm -u+x  # Exécutable par propriétaire

# Fichiers sans propriétaire ou groupe
find / -nouser -o -nogroup 2>/dev/null

# Fichiers vides
find . -type f -empty

# Fichiers avec plusieurs liens (hard links)
find . -type f -links +1
```

### Actions avec find

**Exécuter des commandes sur les résultats** :
```bash
#!/bin/bash
# Actions avec find

# Exécuter une commande sur chaque fichier trouvé
find . -name "*.log" -exec ls -lh {} \;

# Supprimer les fichiers trouvés (avec confirmation)
find . -name "*.tmp" -ok rm {} \;

# Copier les fichiers trouvés
find . -name "*.conf" -exec cp {} /backup/ \;

# Compter les fichiers
find . -type f | wc -l

# Lister avec format personnalisé
find . -type f -printf "%p %s bytes\n"

# Trouver et archiver
find . -name "*.log" -mtime +30 -exec tar -czf logs_old.tar.gz {} +
```

### Optimisations find

**Performance et efficacité** :
```bash
#!/bin/bash
# Optimisations de find

# Éviter les répertoires spécifiques
find . -path ./node_modules -prune -o -name "*.js" -print

# Éviter plusieurs répertoires
find . \( -path ./node_modules -o -path ./.git \) -prune -o -type f -print

# Limiter la profondeur de recherche
find . -maxdepth 3 -name "*.txt"

# Recherche parallèle (GNU find)
find . -name "*.txt" -print0 | xargs -0 -P 4 grep "pattern"

# Utiliser locate pour recherche rapide (si disponible)
locate "*.txt" | grep "^$(pwd)"
```

### Scripts utilitaires avec find

**Fonctions pratiques** :
```bash
#!/bin/bash
# Fonctions utilitaires basées sur find

# Trouver les fichiers les plus récents
find_recent() {
    local days="${1:-7}"
    find . -type f -mtime -"$days" -printf "%T@ %p\n" | \
    sort -rn | head -10 | cut -d' ' -f2-
}

# Trouver les fichiers les plus volumineux
find_largest() {
    local count="${1:-10}"
    find . -type f -printf "%s %p\n" | \
    sort -rn | head -"$count" | \
    awk '{printf "%.2f MB\t%s\n", $1/1024/1024, $2}'
}

# Trouver les fichiers dupliqués (par taille)
find_duplicates() {
    find . -type f -printf "%s %p\n" | \
    sort -n | uniq -d -w 8 | \
    awk '{print $2}'
}

# Nettoyer les fichiers temporaires
cleanup_temp() {
    find . \( -name "*.tmp" -o -name "*.bak" -o -name "*~" \) \
        -type f -mtime +7 -delete
    echo "Fichiers temporaires nettoyés"
}

# Trouver les fichiers binaires
find_binaries() {
    find . -type f -exec file {} \; | \
    grep -E "(executable|binary)" | cut -d: -f1
}
```

## locate et updatedb : recherche rapide

### Utilisation de locate

**locate : recherche instantanée** :
```bash
#!/bin/bash
# Utilisation de locate

# Recherche simple
locate filename.txt

# Recherche avec pattern
locate "*.conf"

# Recherche insensible à la casse
locate -i "readme"

# Limiter le nombre de résultats
locate "*.log" | head -20

# Compter les résultats
locate "*.py" | wc -l

# Recherche avec regex
locate -r "\.log$"

# Afficher seulement les fichiers existants
locate -e "*.txt"
```

### Mise à jour de la base de données

**updatedb : maintenir la base** :
```bash
#!/bin/bash
# Gestion de updatedb

# Mise à jour manuelle (nécessite sudo)
sudo updatedb

# Mise à jour avec exclusions
sudo updatedb --prunepaths="/tmp /var/tmp /media"

# Vérifier la base de données
locate -S

# Créer une base personnalisée
updatedb -U /home/user -o ~/mydb.db
locate -d ~/mydb.db "pattern"
```

### Configuration de locate

**Personnalisation** :
```bash
#!/bin/bash
# Configuration locate

# Vérifier la configuration
cat /etc/updatedb.conf

# Exemples de configuration
# PRUNEPATHS="/tmp /var/spool /media"
# PRUNEFS="NFS afs proc"
# PRUNE_BIND_MOUNTS="yes"

# Créer un alias pour locate avec exclusions
alias locate-clean='locate -e'
```

## Recherche de contenu avec grep

### grep de base

**Utilisation fondamentale** :
```bash
#!/bin/bash
# Utilisation de base de grep

# Recherche simple
grep "pattern" file.txt

# Recherche insensible à la casse
grep -i "pattern" file.txt

# Recherche avec numéros de ligne
grep -n "pattern" file.txt

# Recherche inverse (lignes sans le pattern)
grep -v "pattern" file.txt

# Recherche récursive
grep -r "pattern" directory/

# Recherche avec contexte
grep -C 3 "pattern" file.txt    # 3 lignes avant/après
grep -B 2 "pattern" file.txt    # 2 lignes avant
grep -A 2 "pattern" file.txt    # 2 lignes après

# Recherche de mots complets
grep -w "word" file.txt

# Coloration des résultats
grep --color=always "pattern" file.txt
```

### grep avancé

**Options avancées** :
```bash
#!/bin/bash
# Utilisation avancée de grep

# Recherche avec regex étendue
grep -E "pattern1|pattern2" file.txt

# Recherche avec regex Perl
grep -P "\d{3}-\d{2}-\d{4}" file.txt  # Numéro de sécurité sociale

# Recherche dans fichiers binaires
grep -a "pattern" binary_file

# Afficher seulement les noms de fichiers
grep -l "pattern" *.txt

# Afficher seulement les fichiers sans correspondance
grep -L "pattern" *.txt

# Recherche avec exclusion de fichiers
grep -r --exclude="*.log" "pattern" .

# Recherche avec inclusion de types de fichiers
grep -r --include="*.py" "pattern" .

# Limiter le nombre de correspondances par fichier
grep -m 5 "pattern" file.txt

# Ignorer les erreurs (fichiers non lisibles)
grep -s "pattern" file.txt
```

### Scripts avec grep

**Utilitaires grep** :
```bash
#!/bin/bash
# Scripts utilitaires avec grep

# Rechercher dans plusieurs fichiers
multi_grep() {
    local pattern="$1"
    shift
    grep -Hn "$pattern" "$@"
}

# Compter les occurrences
count_occurrences() {
    local pattern="$1"
    local file="$2"
    grep -o "$pattern" "$file" | wc -l
}

# Extraire des lignes entre deux patterns
extract_between() {
    local start_pattern="$1"
    local end_pattern="$2"
    local file="$3"
    
    awk "/$start_pattern/,/$end_pattern/" "$file"
}

# Rechercher avec highlight
highlight_search() {
    local pattern="$1"
    local file="$2"
    
    grep --color=always "$pattern" "$file" | \
    less -R
}
```

## ripgrep (rg) : grep moderne et rapide

### Installation et utilisation de base

**Installation ripgrep** :
```bash
#!/bin/bash
# Installation de ripgrep

# Linux (Debian/Ubuntu)
sudo apt install ripgrep

# Linux (Fedora/RHEL)
sudo dnf install ripgrep

# macOS
brew install ripgrep

# Via cargo
cargo install ripgrep

# Vérifier l'installation
rg --version
```

**Utilisation de base** :
```bash
#!/bin/bash
# Utilisation de base de ripgrep

# Recherche simple (recherche récursive par défaut)
rg "pattern"

# Recherche dans un répertoire spécifique
rg "pattern" /path/to/directory

# Recherche avec numéros de ligne
rg -n "pattern"

# Recherche insensible à la casse
rg -i "pattern"

# Recherche de mots complets
rg -w "word"

# Recherche avec contexte
rg -C 3 "pattern"

# Afficher seulement les fichiers correspondants
rg -l "pattern"

# Compter les correspondances par fichier
rg -c "pattern"
```

### Options avancées de ripgrep

**Fonctionnalités avancées** :
```bash
#!/bin/bash
# Options avancées de ripgrep

# Recherche avec types de fichiers
rg "pattern" --type py        # Python seulement
rg "pattern" --type js        # JavaScript seulement
rg "pattern" --type-not py    # Tout sauf Python

# Recherche avec exclusions
rg "pattern" --glob "*.log"   # Inclure seulement .log
rg "pattern" --glob "!*.tmp"  # Exclure .tmp

# Recherche avec taille de fichier
rg "pattern" --max-filesize 10M

# Recherche avec profondeur
rg "pattern" --max-depth 3

# Recherche avec threads
rg "pattern" --threads 8

# Recherche avec encoding
rg "pattern" --encoding utf-8

# Recherche avec smart-case (auto-insensible à la casse)
rg -S "Pattern"  # Insensible si pattern en minuscules

# Recherche avec multiline
rg -U "pattern1.*\n.*pattern2"

# Recherche avec lookaround
rg -P "(?<=prefix)pattern(?=suffix)"
```

### Configuration ripgrep

**Fichier de configuration** :
```bash
#!/bin/bash
# Configuration ripgrep

# Créer ~/.ripgreprc
cat > ~/.ripgreprc << 'EOF'
# Configuration ripgrep
--smart-case
--color=always
--heading
--line-number
--max-columns=150
EOF

# Utiliser avec alias
alias rg='rg --smart-case --color=always'
```

## ag (The Silver Searcher) : alternative rapide

### Installation et utilisation

**Installation ag** :
```bash
#!/bin/bash
# Installation ag

# Linux (Debian/Ubuntu)
sudo apt install silversearcher-ag

# Linux (Fedora/RHEL)
sudo dnf install the_silver_searcher

# macOS
brew install the_silver_searcher

# Vérifier
ag --version
```

**Utilisation de base** :
```bash
#!/bin/bash
# Utilisation de base d'ag

# Recherche simple
ag "pattern"

# Recherche avec contexte
ag -C 3 "pattern"

# Recherche insensible à la casse
ag -i "pattern"

# Recherche avec types de fichiers
ag -G "\.py$" "pattern"

# Recherche avec exclusions
ag --ignore "*.log" "pattern"

# Recherche avec numéros de ligne
ag -n "pattern"

# Recherche avec stats
ag --stats "pattern"

# Recherche avec groupe de fichiers
ag --group "pattern"
```

## fd : find moderne

### Installation et utilisation

**Installation fd** :
```bash
#!/bin/bash
# Installation fd

# Linux (Debian/Ubuntu)
sudo apt install fd-find

# Linux (Fedora/RHEL)
sudo dnf install fd-find

# macOS
brew install fd

# Via cargo
cargo install fd-find

# Vérifier
fd --version
```

**Utilisation de base** :
```bash
#!/bin/bash
# Utilisation de base de fd

# Recherche simple (pattern optionnel)
fd "pattern"

# Recherche par extension
fd -e txt
fd -e py -e js

# Recherche par type
fd -t f  # Fichiers seulement
fd -t d  # Répertoires seulement

# Recherche avec exclusion
fd --exclude node_modules
fd --ignore-file .gitignore

# Recherche avec profondeur
fd --max-depth 3

# Recherche avec taille
fd --size +100M

# Recherche avec date
fd --changed-within 7d
fd --changed-before "2024-01-01"

# Recherche avec exécution
fd -e sh -x chmod +x

# Recherche avec preview
fd -e md -X bat
```

## Outils de recherche spécialisés

### Recherche dans les logs

**Outils spécialisés logs** :
```bash
#!/bin/bash
# Recherche dans les logs

# journalctl (systemd)
journalctl -u service_name | grep "pattern"
journalctl --since "1 hour ago" | grep "error"
journalctl -f | grep "pattern"  # Suivi en temps réel

# Recherche dans logs Apache/Nginx
grep "pattern" /var/log/apache2/access.log
grep "pattern" /var/log/nginx/error.log

# Recherche avec tail et grep
tail -f /var/log/syslog | grep "pattern"

# Recherche dans logs rotatifs
zcat /var/log/syslog.1.gz | grep "pattern"
zgrep "pattern" /var/log/*.gz
```

### Recherche dans le code source

**Outils pour développeurs** :
```bash
#!/bin/bash
# Recherche dans le code source

# Rechercher des fonctions
rg "function\s+\w+" --type py

# Rechercher des imports
rg "^import|^from" --type py

# Rechercher des TODOs
rg "TODO|FIXME|XXX" --type-not md

# Rechercher des définitions de classe
rg "class\s+\w+" --type py

# Rechercher des appels de fonction
rg "function_name\(" --type js

# Rechercher avec contexte de code
rg -A 5 -B 5 "pattern" --type py
```

## Recherche interactive avec fzf

### Installation et configuration

**Installation fzf** :
```bash
#!/bin/bash
# Installation fzf

# Linux/macOS
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

# Intégration shell
# Ajouter à ~/.bashrc ou ~/.zshrc
[ -f ~/.fzf.bash ] && source ~/.fzf.bash
```

**Utilisation de base** :
```bash
#!/bin/bash
# Utilisation de base de fzf

# Recherche de fichiers
fzf

# Recherche avec preview
fzf --preview 'cat {}'

# Recherche avec commande personnalisée
find . -type f | fzf

# Recherche dans l'historique
history | fzf

# Recherche avec exécution
fzf -m | xargs rm  # Sélection multiple
```

### Intégration avancée fzf

**Fonctions personnalisées** :
```bash
#!/bin/bash
# Fonctions fzf avancées

# Recherche et édition de fichiers
fe() {
    local file
    file=$(fzf --query="$1" --select-1 --exit-0 \
        --preview 'bat --color=always {}')
    [ -n "$file" ] && ${EDITOR:-vim} "$file"
}

# Recherche et changement de répertoire
fd() {
    local dir
    dir=$(find ${1:-.} -path '*/\.*' -prune \
        -o -type d -print 2> /dev/null | fzf +m) &&
    cd "$dir"
}

# Recherche dans l'historique avec exécution
fh() {
    print -z $(history | fzf +s +m | sed 's/^[ ]*[0-9]*[ ]*//')
}

# Recherche de processus avec kill
fkill() {
    local pid
    pid=$(ps -ef | sed 1d | fzf -m | awk '{print $2}')
    [ -n "$pid" ] && kill -${1:-9} "$pid"
}

# Recherche Git
fg() {
    local file
    file=$(git ls-files | fzf -m --preview 'git diff {}')
    [ -n "$file" ] && git add "$file"
}
```

## Recherche dans les archives et fichiers compressés

### Recherche dans archives

**Outils pour archives** :
```bash
#!/bin/bash
# Recherche dans les archives

# Recherche dans tar.gz
tar -tzf archive.tar.gz | grep "pattern"

# Recherche dans zip
unzip -l archive.zip | grep "pattern"

# Recherche avec zgrep (fichiers compressés)
zgrep "pattern" file.gz
zgrep "pattern" *.gz

# Recherche dans plusieurs archives
for archive in *.tar.gz; do
    echo "=== $archive ==="
    tar -tzf "$archive" | grep "pattern"
done

# Extraire et rechercher
tar -xzf archive.tar.gz -O | grep "pattern"
```

## Optimisation des performances de recherche

### Techniques d'optimisation

**Optimisations pratiques** :
```bash
#!/bin/bash
# Optimisations de recherche

# Utiliser locate pour recherche rapide de fichiers
locate "pattern"  # Beaucoup plus rapide que find

# Limiter la profondeur de recherche
find . -maxdepth 3 -name "*.txt"

# Éviter les répertoires volumineux
find . -path ./node_modules -prune -o -name "*.js" -print

# Utiliser ripgrep au lieu de grep (plus rapide)
rg "pattern"  # Généralement plus rapide que grep -r

# Paralléliser les recherches
find . -name "*.txt" -print0 | xargs -0 -P 4 grep "pattern"

# Utiliser des index pour recherche fréquente
# Créer un index avec locate ou ctags
```

## Recherche dans les bases de données et logs

### Recherche structurée

**Outils pour données structurées** :
```bash
#!/bin/bash
# Recherche dans données structurées

# Recherche dans JSON
rg '"key":\s*"value"' file.json
jq '.[] | select(.key == "value")' file.json

# Recherche dans CSV
rg "pattern" file.csv
awk -F',' '$2 ~ /pattern/ {print}' file.csv

# Recherche dans logs structurés
rg '"level":\s*"error"' app.log
rg '"timestamp":\s*"2024-.*"' app.log

# Recherche avec contexte temporel
rg --after-context 10 '"error"' app.log
```

## Automatisation des recherches complexes

### Scripts de recherche automatisés

**Automatisation complète** :
```bash
#!/bin/bash
# Script de recherche automatisée

# Recherche multi-critères
multi_search() {
    local pattern="$1"
    local file_type="${2:-}"
    local max_age="${3:-}"
    
    local find_cmd="find ."
    
    [ -n "$file_type" ] && find_cmd+=" -name \"*.$file_type\""
    [ -n "$max_age" ] && find_cmd+=" -mtime -$max_age"
    
    eval "$find_cmd" | while read -r file; do
        if rg -q "$pattern" "$file"; then
            echo "$file"
        fi
    done
}

# Recherche avec rapport
search_with_report() {
    local pattern="$1"
    local output_file="${2:-search_report.txt}"
    
    {
        echo "=== Rapport de recherche ==="
        echo "Pattern: $pattern"
        echo "Date: $(date)"
        echo ""
        echo "=== Fichiers correspondants ==="
        rg -l "$pattern"
        echo ""
        echo "=== Statistiques ==="
        rg -c "$pattern" | awk -F: '{sum+=$2} END {print "Total correspondances:", sum}'
    } > "$output_file"
    
    echo "Rapport généré: $output_file"
}

# Recherche récurrente avec monitoring
monitor_search() {
    local pattern="$1"
    local interval="${2:-60}"  # secondes
    
    while true; do
        echo "[$(date)] Recherche de '$pattern'..."
        rg -l "$pattern" | while read -r file; do
            echo "  Trouvé dans: $file"
        done
        sleep "$interval"
    done
}
```

## Conclusion

La maîtrise des outils de recherche avancés transforme votre efficacité au terminal. Que vous utilisiez find pour des recherches complexes de fichiers, ripgrep pour des recherches rapides de contenu, ou fzf pour des interfaces interactives, chaque outil a ses forces et ses cas d'usage optimaux.

Les outils modernes comme ripgrep, fd, et fzf ont considérablement amélioré les performances et l'expérience utilisateur par rapport aux outils traditionnels. En combinant ces outils avec des scripts d'automatisation, vous pouvez créer des workflows de recherche puissants et efficaces.

Dans le chapitre suivant, nous explorerons les expressions régulières en profondeur, découvrant comment créer des patterns de recherche complexes et puissants pour tous ces outils.
