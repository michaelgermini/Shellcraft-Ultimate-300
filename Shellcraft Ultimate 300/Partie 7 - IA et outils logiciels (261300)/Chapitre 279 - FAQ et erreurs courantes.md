# Chapitre 279 - FAQ et erreurs courantes

## Table des matières
- [Introduction](#introduction)
- [FAQ Générale](#faq-générale)
- [Erreurs courantes et solutions](#erreurs-courantes-et-solutions)
- [Problèmes de performance](#problèmes-de-performance)
- [Problèmes de sécurité](#problèmes-de-sécurité)
- [Dépannage avancé](#dépannage-avancé)
- [Conclusion](#conclusion)

## Introduction

Même avec une solide compréhension des concepts, les erreurs sont inévitables. Ce chapitre compile les questions fréquemment posées et les erreurs courantes, avec leurs solutions. C'est votre référence rapide pour résoudre les problèmes les plus communs et comprendre les pièges subtils du shell scripting.

Imaginez ce chapitre comme un manuel de dépannage : chaque problème a sa solution, chaque erreur a son explication, et chaque question a sa réponse claire.

## FAQ Générale

### Questions fréquentes

**Q: Pourquoi mon script échoue avec "command not found" même si la commande existe ?**

**R:** Problème de PATH ou de permissions. Solutions :
```bash
# Vérifier le PATH
echo "$PATH"

# Utiliser le chemin complet
/usr/bin/command

# Vérifier les permissions
ls -l /usr/bin/command

# Ajouter au PATH si nécessaire
export PATH="/custom/path:$PATH"
```

**Q: Comment gérer les espaces dans les noms de fichiers ?**

**R:** Toujours quoter les variables :
```bash
# Bon
for file in *.txt; do
    process_file "$file"
done

# Mauvais
for file in *.txt; do
    process_file $file  # Échoue avec des espaces
done
```

**Q: Pourquoi `set -e` ne fonctionne pas comme attendu ?**

**R:** `set -e` ne s'applique pas dans certaines conditions :
```bash
# set -e est ignoré dans :
# - Les commandes dans un if
if command; then
    # set -e ne s'applique pas ici
fi

# - Les pipelines (sauf le dernier)
command1 | command2  # set -e ne s'applique qu'à command2

# Solution: Utiliser set -euo pipefail et vérifier explicitement
set -euo pipefail
if ! command; then
    echo "Erreur" >&2
    exit 1
fi
```

**Q: Comment déboguer un script qui ne fonctionne pas ?**

**R:** Utiliser les options de débogage :
```bash
#!/bin/bash
set -x  # Affiche chaque commande avant exécution
set -v  # Affiche les lignes telles qu'elles sont lues
set -e  # Arrête à la première erreur

# Ou combiner
set -xve

# Pour une section spécifique
set -x
# Code à déboguer
set +x
```

## Erreurs courantes et solutions

### Erreur: "unbound variable"

**Problème** :
```bash
#!/bin/bash
set -u
echo "$undefined_variable"  # Erreur: unbound variable
```

**Solution** :
```bash
#!/bin/bash
set -u

# Vérifier avant utilisation
if [ -z "${undefined_variable:-}" ]; then
    undefined_variable="default_value"
fi

# Ou utiliser la substitution de paramètre
echo "${undefined_variable:-default_value}"
```

### Erreur: "syntax error near unexpected token"

**Problème** :
```bash
# Erreur de syntaxe courante
if [ $var = "value" ]  # Manque des espaces ou quotes
```

**Solution** :
```bash
# Toujours quoter les variables dans les tests
if [ "$var" = "value" ]; then
    # ...
fi

# Ou utiliser [[ ]] qui gère mieux les variables
if [[ "$var" == "value" ]]; then
    # ...
fi
```

### Erreur: "too many arguments"

**Problème** :
```bash
# Si $var contient plusieurs mots
if [ $var = "value" ]  # Erreur si $var = "a b"
```

**Solution** :
```bash
# Toujours quoter
if [ "$var" = "value" ]; then
    # ...
fi
```

### Erreur: "binary operator expected"

**Problème** :
```bash
# Variable vide dans un test
if [ $var -gt 10 ]; then  # Erreur si $var est vide
```

**Solution** :
```bash
# Vérifier que la variable n'est pas vide
if [ -n "$var" ] && [ "$var" -gt 10 ]; then
    # ...
fi

# Ou utiliser la substitution
if [ "${var:-0}" -gt 10 ]; then
    # ...
fi
```

## Problèmes de performance

### Script lent: boucles inefficaces

**Problème** :
```bash
# Boucle lente avec appels système répétés
for file in $(ls *.txt); do
    process_file "$file"
done
```

**Solution** :
```bash
# Utiliser glob directement
for file in *.txt; do
    [ -f "$file" ] || continue
    process_file "$file"
done

# Ou utiliser find pour plus de contrôle
find . -maxdepth 1 -name "*.txt" -type f -print0 | \
while IFS= read -r -d '' file; do
    process_file "$file"
done
```

### Script lent: appels système répétés

**Problème** :
```bash
# Appel répété de commandes lentes
for user in $(cat users.txt); do
    id "$user"  # Appel système à chaque itération
    groups "$user"  # Autre appel système
done
```

**Solution** :
```bash
# Cacher les résultats
declare -A user_ids
declare -A user_groups

# Charger une fois
while IFS= read -r user; do
    user_ids["$user"]=$(id -u "$user")
    user_groups["$user"]=$(groups "$user")
done < users.txt

# Utiliser le cache
for user in "${!user_ids[@]}"; do
    echo "User: $user, ID: ${user_ids[$user]}, Groups: ${user_groups[$user]}"
done
```

### Script lent: pas de parallélisation

**Problème** :
```bash
# Traitement séquentiel
for file in *.txt; do
    process_file "$file"  # Lent si beaucoup de fichiers
done
```

**Solution** :
```bash
# Paralléliser avec xargs
find . -name "*.txt" -print0 | \
xargs -0 -P 4 -I {} bash -c 'process_file "$@"' _ {}

# Ou avec background jobs
for file in *.txt; do
    process_file "$file" &
done
wait  # Attendre que tous les jobs se terminent
```

## Problèmes de sécurité

### Injection de commandes

**Problème** :
```bash
# Dangereux: injection possible
filename="$1"
rm "$filename"  # Si $1 = "file; rm -rf /"
```

**Solution** :
```bash
# Valider l'input
filename="$1"

# Vérifier que c'est un nom de fichier valide
if [[ ! "$filename" =~ ^[a-zA-Z0-9._-]+$ ]]; then
    echo "Nom de fichier invalide" >&2
    exit 1
fi

# Utiliser le chemin complet et vérifier l'existence
if [ ! -f "$filename" ]; then
    echo "Fichier non trouvé" >&2
    exit 1
fi

rm "$filename"
```

### Variables non échappées

**Problème** :
```bash
# Dangereux
user_input="$1"
grep "$user_input" file.txt  # Injection possible
```

**Solution** :
```bash
# Valider et échapper
user_input="$1"

# Valider le format attendu
if [[ ! "$user_input" =~ ^[a-zA-Z0-9\s]+$ ]]; then
    echo "Input invalide" >&2
    exit 1
fi

# Utiliser -- pour séparer les options
grep -- "$user_input" file.txt
```

## Dépannage avancé

### Script de diagnostic

**Outil de diagnostic** :
```bash
#!/bin/bash
# Script de diagnostic automatique

diagnose_script() {
    local script="$1"
    
    echo "=== Diagnostic du script ==="
    echo "Script: $script"
    echo
    
    # 1. Vérifier la syntaxe
    echo "1. Syntaxe..."
    if bash -n "$script"; then
        echo "✓ Syntaxe valide"
    else
        echo "✗ Erreurs de syntaxe"
        bash -n "$script" 2>&1
    fi
    
    # 2. Analyser avec ShellCheck
    echo "2. ShellCheck..."
    if command -v shellcheck &> /dev/null; then
        shellcheck "$script" || true
    else
        echo "⚠ ShellCheck non installé"
    fi
    
    # 3. Vérifier les variables
    echo "3. Variables..."
    local uninitialized=$(grep -oE '\$\{[^}]+\}' "$script" | \
        sed 's/\${//;s/}//' | sort -u)
    
    for var in $uninitialized; do
        if ! grep -qE "^\s*${var}=" "$script"; then
            echo "⚠ Variable potentiellement non initialisée: $var"
        fi
    done
    
    # 4. Vérifier les erreurs courantes
    echo "4. Erreurs courantes..."
    
    # Variables non quotées
    if grep -E '\$[a-zA-Z_][a-zA-Z0-9_]*[^"]' "$script" | \
       grep -vE '^\s*#' | grep -vE 'set\s+-'; then
        echo "⚠ Variables potentiellement non quotées"
    fi
    
    # Tests avec = au lieu de ==
    if grep -E '\[.*=.*\]' "$script" | grep -vE '^\s*#'; then
        echo "⚠ Utilisez == dans [[ ]] ou = dans [ ]"
    fi
    
    # 5. Suggestions d'amélioration
    echo "5. Suggestions..."
    suggest_improvements "$script"
}

suggest_improvements() {
    local script="$1"
    
    # Vérifier set -euo pipefail
    if ! head -5 "$script" | grep -qE 'set\s+(-e|-u|-o)'; then
        echo "💡 Ajoutez 'set -euo pipefail' au début du script"
    fi
    
    # Vérifier la gestion d'erreurs
    if ! grep -qE '(trap|ERR|EXIT)' "$script"; then
        echo "💡 Ajoutez une gestion d'erreurs avec trap"
    fi
    
    # Vérifier la documentation
    if ! head -20 "$script" | grep -qE '(Description|Usage|Author)'; then
        echo "💡 Ajoutez une documentation en en-tête"
    fi
}
```

### Debug interactif

**Mode debug interactif** :
```bash
#!/bin/bash
# Mode debug interactif

debug_script() {
    local script="$1"
    
    # Activer le mode debug
    set -xve
    
    # Exécuter le script avec des pauses
    bash -x "$script" 2>&1 | \
    while IFS= read -r line; do
        echo "$line"
        
        # Pause sur les erreurs
        if echo "$line" | grep -qE 'error|Error|ERROR|failed|Failed'; then
            read -p "Pause (appuyez sur Entrée pour continuer): "
        fi
    done
}
```

## Conclusion

Les erreurs sont une partie naturelle du développement. En comprenant les erreurs courantes et leurs solutions, vous développez non seulement vos compétences de dépannage, mais aussi votre compréhension profonde du shell scripting.

Ce chapitre sert de référence rapide pour résoudre les problèmes les plus fréquents et éviter les pièges communs. Gardez-le à portée de main lors du développement.

Dans le chapitre suivant, nous explorerons le projet final : un workflow 100% automatisé multi-plateforme qui intègre toutes les techniques apprises dans un projet complet et professionnel.

