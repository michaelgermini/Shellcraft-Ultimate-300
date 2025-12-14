# Chapitre 278 - Checklist et bonnes pratiques

## Table des matières
- [Introduction](#introduction)
- [Checklist de développement](#checklist-de-développement)
- [Checklist de sécurité](#checklist-de-sécurité)
- [Checklist de performance](#checklist-de-performance)
- [Checklist de déploiement](#checklist-de-déploiement)
- [Bonnes pratiques générales](#bonnes-pratiques-générales)
- [Conclusion](#conclusion)

## Introduction

Les checklists et bonnes pratiques sont les garde-fous qui transforment un script fonctionnel en un script professionnel, maintenable et fiable. Elles servent de référence rapide pour s'assurer que tous les aspects importants sont couverts, de la sécurité à la performance, en passant par la maintenabilité.

Imaginez les checklists comme une liste de contrôle d'un pilote d'avion : chaque élément vérifié augmente la sécurité et la fiabilité du vol. De même, chaque point de la checklist garantit la qualité de votre automatisation.

## Checklist de développement

### Checklist avant commit

**Script de vérification pré-commit** :
```bash
#!/bin/bash
# Checklist de développement automatisée

pre_commit_checklist() {
    local errors=0
    
    echo "=== Checklist de développement ==="
    echo
    
    # 1. Syntaxe Bash
    echo "1. Vérification de la syntaxe..."
    if check_bash_syntax; then
        echo "✓ Syntaxe Bash valide"
    else
        echo "✗ Erreurs de syntaxe détectées"
        ((errors++))
    fi
    
    # 2. ShellCheck
    echo "2. Analyse ShellCheck..."
    if check_shellcheck; then
        echo "✓ ShellCheck passé"
    else
        echo "✗ Problèmes ShellCheck détectés"
        ((errors++))
    fi
    
    # 3. Variables non initialisées
    echo "3. Vérification des variables..."
    if check_uninitialized_vars; then
        echo "✓ Toutes les variables sont initialisées"
    else
        echo "✗ Variables non initialisées détectées"
        ((errors++))
    fi
    
    # 4. Gestion d'erreurs
    echo "4. Vérification de la gestion d'erreurs..."
    if check_error_handling; then
        echo "✓ Gestion d'erreurs appropriée"
    else
        echo "✗ Gestion d'erreurs manquante"
        ((errors++))
    fi
    
    # 5. Documentation
    echo "5. Vérification de la documentation..."
    if check_documentation; then
        echo "✓ Documentation présente"
    else
        echo "✗ Documentation manquante ou incomplète"
        ((errors++))
    fi
    
    # 6. Tests
    echo "6. Vérification des tests..."
    if check_tests; then
        echo "✓ Tests présents et passent"
    else
        echo "✗ Tests manquants ou échouent"
        ((errors++))
    fi
    
    # Résultat final
    echo
    if [ $errors -eq 0 ]; then
        echo "✓ Toutes les vérifications passées"
        return 0
    else
        echo "✗ $errors erreur(s) détectée(s)"
        return 1
    fi
}

check_bash_syntax() {
    local files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.sh$')
    
    for file in $files; do
        if ! bash -n "$file"; then
            return 1
        fi
    done
    
    return 0
}

check_shellcheck() {
    if ! command -v shellcheck &> /dev/null; then
        echo "⚠ ShellCheck non installé, skip"
        return 0
    fi
    
    local files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.sh$')
    
    for file in $files; do
        if ! shellcheck "$file"; then
            return 1
        fi
    done
    
    return 0
}

check_uninitialized_vars() {
    local files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.sh$')
    
    for file in $files; do
        # Vérifier la présence de set -u ou set -euo pipefail
        if ! grep -qE 'set\s+(-u|-euo)' "$file"; then
            echo "⚠ $file: Considérez 'set -euo pipefail'"
            return 1
        fi
    done
    
    return 0
}

check_error_handling() {
    local files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.sh$')
    
    for file in $files; do
        # Vérifier la présence de gestion d'erreurs
        if ! grep -qE '(set\s+-e|trap|ERR|EXIT)' "$file"; then
            echo "⚠ $file: Ajoutez une gestion d'erreurs"
            return 1
        fi
    done
    
    return 0
}

check_documentation() {
    local files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.sh$')
    
    for file in $files; do
        # Vérifier la présence d'un en-tête avec description
        if ! head -20 "$file" | grep -qE '(Description|Usage|Author|#!)'; then
            echo "⚠ $file: Ajoutez une documentation en en-tête"
            return 1
        fi
    done
    
    return 0
}

check_tests() {
    local files=$(git diff --cached --name-only --diff-filter=ACM | grep '\.sh$')
    
    for file in $files; do
        local test_file="tests/$(basename "$file" .sh)_test.sh"
        if [ ! -f "$test_file" ]; then
            echo "⚠ $file: Aucun fichier de test trouvé"
            return 1
        fi
        
        # Exécuter les tests
        if ! bash "$test_file"; then
            return 1
        fi
    done
    
    return 0
}
```

### Checklist de code review

**Checklist de review** :
```bash
#!/bin/bash
# Checklist de code review

code_review_checklist() {
    local file="$1"
    
    echo "=== Checklist de Code Review ==="
    echo "Fichier: $file"
    echo
    
    local score=0
    local max_score=0
    
    # 1. Lisibilité (20 points)
    echo "1. Lisibilité (20 points)"
    max_score=$((max_score + 20))
    if check_readability "$file"; then
        echo "✓ Code lisible et bien formaté"
        score=$((score + 20))
    else
        echo "✗ Améliorer la lisibilité"
    fi
    
    # 2. Sécurité (30 points)
    echo "2. Sécurité (30 points)"
    max_score=$((max_score + 30))
    if check_security "$file"; then
        echo "✓ Pas de problèmes de sécurité évidents"
        score=$((score + 30))
    else
        echo "✗ Problèmes de sécurité détectés"
    fi
    
    # 3. Performance (20 points)
    echo "3. Performance (20 points)"
    max_score=$((max_score + 20))
    if check_performance "$file"; then
        echo "✓ Optimisations appropriées"
        score=$((score + 20))
    else
        echo "✗ Opportunités d'optimisation"
    fi
    
    # 4. Maintenabilité (30 points)
    echo "4. Maintenabilité (30 points)"
    max_score=$((max_score + 30))
    if check_maintainability "$file"; then
        echo "✓ Code maintenable"
        score=$((score + 30))
    else
        echo "✗ Améliorer la maintenabilité"
    fi
    
    # Score final
    local percentage=$((score * 100 / max_score))
    echo
    echo "Score: $score/$max_score ($percentage%)"
    
    if [ $percentage -ge 80 ]; then
        echo "✓ Code review approuvé"
        return 0
    else
        echo "✗ Code review nécessite des améliorations"
        return 1
    fi
}
```

## Checklist de sécurité

### Checklist de sécurité

**Vérifications de sécurité** :
```bash
#!/bin/bash
# Checklist de sécurité

security_checklist() {
    local script="$1"
    
    echo "=== Checklist de Sécurité ==="
    echo "Script: $script"
    echo
    
    local issues=0
    
    # 1. Injection de commandes
    echo "1. Vérification des injections de commandes..."
    if check_command_injection "$script"; then
        echo "✓ Pas d'injection de commandes détectée"
    else
        echo "✗ Risques d'injection de commandes"
        ((issues++))
    fi
    
    # 2. Variables non échappées
    echo "2. Vérification des variables échappées..."
    if check_quoted_vars "$script"; then
        echo "✓ Variables correctement échappées"
    else
        echo "✗ Variables non échappées détectées"
        ((issues++))
    fi
    
    # 3. Permissions sensibles
    echo "3. Vérification des permissions..."
    if check_permissions "$script"; then
        echo "✓ Permissions appropriées"
    else
        echo "✗ Permissions trop permissives"
        ((issues++))
    fi
    
    # 4. Secrets hardcodés
    echo "4. Vérification des secrets..."
    if check_hardcoded_secrets "$script"; then
        echo "✓ Pas de secrets hardcodés"
    else
        echo "✗ Secrets hardcodés détectés"
        ((issues++))
    fi
    
    # 5. Accès réseau
    echo "5. Vérification des accès réseau..."
    if check_network_access "$script"; then
        echo "✓ Accès réseau sécurisés"
    else
        echo "✗ Accès réseau non sécurisés"
        ((issues++))
    fi
    
    # Résultat
    echo
    if [ $issues -eq 0 ]; then
        echo "✓ Aucun problème de sécurité détecté"
        return 0
    else
        echo "✗ $issues problème(s) de sécurité détecté(s)"
        return 1
    fi
}

check_command_injection() {
    local script="$1"
    
    # Chercher les utilisations dangereuses d'évaluation
    if grep -E '\$\(.*\$' "$script" | grep -vE '^\s*#' | grep -vE 'set\s+-'; then
        return 1
    fi
    
    # Chercher les eval non sécurisés
    if grep -E 'eval\s+\$' "$script" | grep -vE '^\s*#'; then
        return 1
    fi
    
    return 0
}

check_quoted_vars() {
    local script="$1"
    
    # Chercher les variables non échappées dans les commandes
    if grep -E '\$\{[^}]*\}' "$script" | grep -vE '["\047]' | grep -vE '^\s*#'; then
        return 1
    fi
    
    return 0
}

check_permissions() {
    local script="$1"
    
    # Vérifier que le script n'a pas de permissions trop permissives
    local perms=$(stat -c "%a" "$script")
    if [ "$perms" -gt 755 ]; then
        return 1
    fi
    
    return 0
}

check_hardcoded_secrets() {
    local script="$1"
    
    # Chercher les patterns de secrets
    if grep -iE '(password|secret|key|token|api_key)\s*=\s*["\047][^"\047]+["\047]' "$script" | \
       grep -vE '^\s*#' | grep -vE 'example|placeholder|dummy'; then
        return 1
    fi
    
    return 0
}
```

## Checklist de performance

### Checklist de performance

**Vérifications de performance** :
```bash
#!/bin/bash
# Checklist de performance

performance_checklist() {
    local script="$1"
    
    echo "=== Checklist de Performance ==="
    echo "Script: $script"
    echo
    
    local optimizations=0
    
    # 1. Boucles optimisées
    echo "1. Vérification des boucles..."
    if check_loop_optimization "$script"; then
        echo "✓ Boucles optimisées"
    else
        echo "⚠ Opportunités d'optimisation des boucles"
        ((optimizations++))
    fi
    
    # 2. Appels système
    echo "2. Vérification des appels système..."
    if check_system_calls "$script"; then
        echo "✓ Appels système optimisés"
    else
        echo "⚠ Réduire les appels système"
        ((optimizations++))
    fi
    
    # 3. Parallélisation
    echo "3. Vérification de la parallélisation..."
    if check_parallelization "$script"; then
        echo "✓ Parallélisation appropriée"
    else
        echo "⚠ Opportunités de parallélisation"
        ((optimizations++))
    fi
    
    # 4. Cache
    echo "4. Vérification du cache..."
    if check_caching "$script"; then
        echo "✓ Cache utilisé efficacement"
    else
        echo "⚠ Opportunités de mise en cache"
        ((optimizations++))
    fi
    
    echo
    echo "Optimisations suggérées: $optimizations"
}
```

## Checklist de déploiement

### Checklist pré-déploiement

**Vérifications avant déploiement** :
```bash
#!/bin/bash
# Checklist de déploiement

pre_deployment_checklist() {
    echo "=== Checklist Pré-Déploiement ==="
    echo
    
    local errors=0
    
    # 1. Tests passent
    echo "1. Tests..."
    if run_all_tests; then
        echo "✓ Tous les tests passent"
    else
        echo "✗ Des tests échouent"
        ((errors++))
    fi
    
    # 2. Documentation à jour
    echo "2. Documentation..."
    if check_documentation_up_to_date; then
        echo "✓ Documentation à jour"
    else
        echo "✗ Documentation obsolète"
        ((errors++))
    fi
    
    # 3. Variables d'environnement
    echo "3. Variables d'environnement..."
    if check_env_vars; then
        echo "✓ Variables d'environnement configurées"
    else
        echo "✗ Variables d'environnement manquantes"
        ((errors++))
    fi
    
    # 4. Backups
    echo "4. Backups..."
    if check_backups; then
        echo "✓ Backups à jour"
    else
        echo "✗ Backups manquants ou obsolètes"
        ((errors++))
    fi
    
    # 5. Rollback plan
    echo "5. Plan de rollback..."
    if check_rollback_plan; then
        echo "✓ Plan de rollback prêt"
    else
        echo "✗ Plan de rollback manquant"
        ((errors++))
    fi
    
    # Résultat
    echo
    if [ $errors -eq 0 ]; then
        echo "✓ Prêt pour le déploiement"
        return 0
    else
        echo "✗ $errors problème(s) à résoudre avant le déploiement"
        return 1
    fi
}
```

## Bonnes pratiques générales

### Guide de bonnes pratiques

**Bonnes pratiques essentielles** :

1. **Utiliser `set -euo pipefail`**
   ```bash
   # Toujours au début du script
   set -euo pipefail
   ```

2. **Quoter toutes les variables**
   ```bash
   # Bon
   echo "$variable"
   cp "$source" "$dest"
   
   # Mauvais
   echo $variable
   cp $source $dest
   ```

3. **Utiliser des noms de variables descriptifs**
   ```bash
   # Bon
   local backup_directory="/backups"
   local max_retry_count=3
   
   # Mauvais
   local dir="/backups"
   local max=3
   ```

4. **Gérer les erreurs explicitement**
   ```bash
   # Bon
   if ! command; then
       echo "Erreur: command a échoué" >&2
       exit 1
   fi
   
   # Mauvais
   command  # Pas de gestion d'erreur
   ```

5. **Documenter les fonctions**
   ```bash
   # Bon
   # Fonction: Crée une sauvegarde
   # Arguments: $1 - source, $2 - destination
   # Retour: 0 si succès, 1 si erreur
   create_backup() {
       local source="$1"
       local dest="$2"
       # ...
   }
   ```

6. **Utiliser des chemins absolus pour les fichiers critiques**
   ```bash
   # Bon
   LOG_FILE="/var/log/app.log"
   
   # Mauvais
   LOG_FILE="./app.log"
   ```

7. **Valider les entrées utilisateur**
   ```bash
   # Bon
   if [[ ! "$input" =~ ^[a-zA-Z0-9]+$ ]]; then
       echo "Entrée invalide" >&2
       exit 1
   fi
   ```

8. **Utiliser des fichiers temporaires sécurisés**
   ```bash
   # Bon
   TEMP_FILE=$(mktemp)
   trap "rm -f $TEMP_FILE" EXIT
   
   # Mauvais
   TEMP_FILE="/tmp/temp.txt"
   ```

## Conclusion

Les checklists et bonnes pratiques sont les fondations d'un code professionnel. En suivant systématiquement ces vérifications et en appliquant ces pratiques, vous créez des scripts qui sont non seulement fonctionnels, mais aussi sécurisés, performants, et maintenables.

Ces pratiques deviennent une seconde nature avec le temps, transformant votre façon de coder et garantissant la qualité de vos automatisations.

Dans le chapitre suivant, nous explorerons la FAQ et les erreurs courantes, consolidant les connaissances acquises et préparant les solutions aux problèmes fréquents.

