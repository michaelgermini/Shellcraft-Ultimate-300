# Chapitre 07 - Principes de scripting

## Table des matières
- [Introduction](#introduction)
- [Qu'est-ce que le scripting ?](#quest-ce-que-le-scripting-)
- [Philosophie UNIX : petits outils, composition](#philosophie-unix--petits-outils-composition)
- [Structure d'un script](#structure-dun-script)
- [Variables et types de données](#variables-et-types-de-données)
- [Contrôle de flux](#contrôle-de-flux)
- [Fonctions et modularité](#fonctions-et-modularité)
- [Gestion d'erreurs](#gestion-derreurs)
- [Bonnes pratiques](#bonnes-pratiques)
- [Débogage et maintenance](#débogage-et-maintenance)
- [Conclusion](#conclusion)

## Introduction

Le scripting transforme des tâches répétitives en automatisations fiables et maintenables. Au-delà de la simple exécution de commandes, le scripting permet de créer des programmes complexes qui interagissent avec le système, traitent des données, et prennent des décisions intelligentes.

Imaginez le scripting comme la programmation d'un robot personnel : au lieu d'exécuter manuellement chaque étape, vous programmez une séquence logique qui s'adapte aux conditions et produit des résultats cohérents.

## Qu'est-ce que le scripting ?

### Définition et portée

**Scripting** : Écriture de programmes interprétés qui automatisent des tâches système.

**Caractéristiques clés** :
- Interprétation à la volée (pas de compilation)
- Intégration étroite avec le shell
- Accès direct aux commandes système
- Portabilité entre systèmes similaires

### Niveaux de scripting

**Scripts simples** :
- Automatisation de tâches répétitives
- Séquences de commandes
- Exemple : sauvegarde quotidienne

**Scripts complexes** :
- Logique conditionnelle avancée
- Traitement de données
- Interfaces utilisateur
- Intégration avec APIs

## Philosophie UNIX : petits outils, composition

### Principe de composition

**"Do one thing and do it well"** - Doug McIlroy

Chaque outil UNIX est conçu pour une tâche spécifique :
- `grep` : recherche de texte
- `sed` : édition de flux
- `awk` : traitement de données
- `sort` : tri de données

**Composition via pipes** :
```bash
# Recherche, tri, comptage
grep "ERROR" logfile.txt | sort | uniq -c | sort -nr
```

### Avantages de cette approche

**Modularité** :
- Outils réutilisables indépendamment
- Combinaisons flexibles
- Maintenance simplifiée

**Efficacité** :
- Pas de réinvention de la roue
- Optimisation de chaque outil
- Pipeline processing (pas de fichiers temporaires)

## Structure d'un script

### Shebang et en-tête

**Shebang** : Spécifie l'interpréteur
```bash
#!/bin/bash
# Script de sauvegarde automatique
# Auteur: Votre nom
# Date: 2024
# Description: Sauvegarde des fichiers importants
```

### Sections logiques

**Initialisation** :
```bash
# Configuration
set -euo pipefail  # Mode strict

# Variables
SOURCE_DIR="/home/user/documents"
BACKUP_DIR="/mnt/backup"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
```

**Traitement principal** :
```bash
# Fonction principale
main() {
    check_dependencies
    create_backup
    verify_backup
    cleanup_old
}

# Exécution
main "$@"
```

## Variables et types de données

### Types de base

**Chaînes** :
```bash
nom="Alice"
message="Bonjour $nom"
chemin='/home/user/file.txt'  # Pas d'expansion
```

**Nombres** :
```bash
compteur=0
((compteur++))  # Incrémentation arithmétique
let "total = 10 + 5"
```

**Tableaux** :
```bash
# Déclaration
fichiers=("file1.txt" "file2.txt" "file3.txt")

# Accès
echo "${fichiers[0]}"     # Premier élément
echo "${fichiers[@]}"     # Tous les éléments
echo "${#fichiers[@]}"    # Nombre d'éléments
```

### Variables spéciales

**Paramètres de position** :
```bash
echo "Script: $0"
echo "Premier argument: $1"
echo "Nombre d'arguments: $#"
echo "Tous les arguments: $@"
```

**Variables système** :
```bash
echo "Utilisateur: $USER"
echo "Répertoire: $PWD"
echo "Process ID: $$"
echo "Code de retour précédent: $?"
```

## Contrôle de flux

### Conditions

**Test simple** :
```bash
if [ -f "$fichier" ]; then
    echo "Le fichier existe"
fi
```

**Test avec alternatives** :
```bash
if [ "$age" -lt 18 ]; then
    echo "Mineur"
elif [ "$age" -lt 65 ]; then
    echo "Adulte"
else
    echo "Senior"
fi
```

**Opérateurs de test** :
```bash
[ -f "$file" ]  # Fichier existe
[ -d "$dir" ]   # Répertoire existe
[ -z "$var" ]   # Variable vide
[ "$a" = "$b" ] # Égalité de chaînes
[ "$n" -eq 5 ]  # Égalité numérique
```

### Boucles

**Boucle for** :
```bash
# Sur éléments
for fichier in *.txt; do
    echo "Traitement de $fichier"
    traitement "$fichier"
done

# Sur séquence
for i in {1..10}; do
    echo "Itération $i"
done
```

**Boucle while** :
```bash
compteur=0
while [ $compteur -lt 5 ]; do
    echo "Compteur: $compteur"
    ((compteur++))
done
```

## Fonctions et modularité

### Définition de fonctions

**Syntaxe de base** :
```bash
# Fonction simple
saluer() {
    echo "Bonjour $1!"
}

# Appel
saluer "Alice"
```

**Fonction avec retour** :
```bash
calculer_taille() {
    local fichier="$1"
    if [ -f "$fichier" ]; then
        wc -c < "$fichier"
    else
        echo "0"
    fi
}

taille=$(calculer_taille "monfichier.txt")
```

### Bonnes pratiques

**Variables locales** :
```bash
ma_fonction() {
    local variable_locale="valeur"
    # ...
}
```

**Validation des paramètres** :
```bash
valider_fichier() {
    local fichier="$1"
    if [ ! -f "$fichier" ]; then
        echo "Erreur: fichier '$fichier' introuvable" >&2
        return 1
    fi
}
```

## Gestion d'erreurs

### Mode strict

**Options recommandées** :
```bash
set -e          # Sortie sur erreur
set -u          # Erreur sur variable non définie
set -o pipefail # Échec du pipe si une commande échoue
```

### Gestion explicite

**Try-catch like** :
```bash
commande_risquee() {
    if ! commande; then
        echo "Erreur lors de l'exécution" >&2
        return 1
    fi
}
```

**Nettoyage automatique** :
```bash
cleanup() {
    echo "Nettoyage..."
    rm -f /tmp/tempfile
}

trap cleanup EXIT  # Exécuté à la sortie du script
```

## Bonnes pratiques

### Lisibilité

**Indentation cohérente** :
```bash
if [ condition ]; then
    commande1
    if [ sous_condition ]; then
        commande2
    fi
fi
```

**Commentaires et documentation** :
```bash
# Vérifie si le service est actif
# Arguments: nom_du_service
verifier_service() {
    local service="$1"
    # ...
}
```

### Sécurité

**Validation des entrées** :
```bash
# Échapper les caractères spéciaux
fichier=$(printf '%q' "$input")

# Utiliser des chemins absolus
cp "$SOURCE" "/destination/safe/$(basename "$SOURCE")"
```

**Permissions appropriées** :
```bash
# Scripts exécutables uniquement par propriétaire
chmod 700 script.sh
```

### Performance

**Éviter les sous-shells inutiles** :
```bash
# Mauvais
for file in $(ls *.txt); do

# Bon
for file in *.txt; do
```

**Utiliser les builtins** :
```bash
# Préférer echo à /bin/echo
# Préférer [ à test ou /bin/test
```

## Débogage et maintenance

### Outils de débogage

**Mode verbose** :
```bash
bash -x script.sh  # Affiche chaque commande exécutée
bash -v script.sh  # Affiche le script avant exécution
```

**Points d'arrêt** :
```bash
# Debug interactif
set -x  # Activer
set +x  # Désactiver

# Vérifications intermédiaires
echo "DEBUG: variable = $variable" >&2
```

### Tests et validation

**Tests unitaires** :
```bash
# Fonction de test
test_ma_fonction() {
    assert_equals "$(ma_fonction input)" "expected_output"
}

# Framework de test simple
run_tests() {
    test_ma_fonction && echo "✓ Test passé" || echo "✗ Test échoué"
}
```

### Maintenance

**Versionnage** :
```bash
VERSION="1.2.3"
# Inclure dans l'en-tête
```

**Logs structurés** :
```bash
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$1] $2" >&2
}

log "INFO" "Début du traitement"
```

## Conclusion

Le scripting shell transforme l'interaction ponctuelle avec le système en automatisations puissantes et maintenables. Les principes fondamentaux - composition, modularité, gestion d'erreurs - s'appliquent quel que soit le langage de scripting utilisé.

Un bon script n'est pas seulement fonctionnel ; il est lisible, maintenable, et robuste. Il traite les erreurs gracieusement, valide ses entrées, et fournit un feedback utile à l'utilisateur.

Dans les chapitres suivants, nous appliquerons ces principes à des cas concrets, créant des scripts qui résolvent des problèmes réels et s'intègrent harmonieusement dans les workflows DevOps modernes.
