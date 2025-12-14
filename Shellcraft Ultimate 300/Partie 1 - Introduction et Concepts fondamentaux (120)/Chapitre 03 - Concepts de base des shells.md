# Chapitre 03 - Concepts de base des shells

## Table des matières
- [Introduction](#introduction)
- [Qu'est-ce qu'un shell ?](#quest-ce-quun-shell-)
- [Le modèle client-serveur du terminal](#le-modèle-client-serveur-du-terminal)
- [Types de shells](#types-de-shells)
- [L'interpréteur de commandes](#linterpréteur-de-commandes)
- [Le prompt et l'invite de commande](#le-prompt-et-linvite-de-commande)
- [Exécution de commandes](#exécution-de-commandes)
- [Variables d'environnement](#variables-denvironnement)
- [Configuration et personnalisation](#configuration-et-personnalisation)
- [Conclusion](#conclusion)

## Introduction

Avant de plonger dans les commandes spécifiques, il est essentiel de comprendre ce qu'est réellement un shell. Cette compréhension conceptuelle transforme l'apprentissage du terminal d'une liste de commandes mémorisées en une compréhension intuitive des interactions système.

Imaginez le shell comme le cerveau d'un système nerveux : il reçoit les stimuli (vos commandes), les interprète, et déclenche les réponses appropriées dans le système. Sans cette compréhension fondamentale, chaque commande reste un mystère isolé.

## Qu'est-ce qu'un shell ?

### Définition technique

Un shell est un programme qui fournit une interface textuelle entre l'utilisateur et le système d'exploitation. Il agit comme un interpréteur, traduisant vos commandes en actions système.

### Analogie fonctionnelle

Considérez le shell comme un traducteur dans une conversation internationale :
- Vous parlez en langage humain naturel
- Le shell traduit vers le langage machine binaire
- Le système d'exploitation exécute les instructions
- Le shell présente les résultats dans un format lisible

### Rôles du shell

1. **Interpréteur de commandes** : Analyse et exécute les commandes saisies
2. **Gestionnaire d'environnement** : Maintient les variables et l'état de session
3. **Orchestrateur de processus** : Lance et gère les programmes en arrière-plan
4. **Interface de scripting** : Permet l'automatisation via des scripts

## Le modèle client-serveur du terminal

### L'émulateur de terminal

L'émulateur de terminal (Terminal, iTerm, Windows Terminal, etc.) est l'application graphique qui affiche le shell. C'est le "client" dans ce modèle.

### Le shell comme serveur

Le shell lui-même fonctionne comme un serveur de commandes, attendant vos instructions et y répondant. Cette séparation permet :
- Plusieurs shells dans différentes fenêtres
- Sessions shell distantes via SSH
- Intégration dans d'autres applications

### Le système d'exploitation comme backend

Le shell communique avec le kernel via des appels système, traduisant vos intentions en opérations de bas niveau.

## Types de shells

### Shells interactifs vs non-interactifs

**Shells interactifs** :
- Fournissent un prompt et attendent l'entrée utilisateur
- Supportent l'édition de ligne, l'historique, l'autocomplétion
- Utilisés pour les sessions utilisateur directes

**Shells non-interactifs** :
- Exécutent des scripts sans interaction utilisateur
- N'affichent pas de prompt
- Optimisés pour l'automatisation

### Shells de connexion vs non-connexion

**Shells de connexion** :
- Lancés lors de la connexion utilisateur
- Exécutent les fichiers de profil globaux et personnels
- Établissent l'environnement complet

**Shells non-connexion** :
- Lancés depuis un shell existant (ex: `bash` dans un terminal)
- Héritent de l'environnement parent
- N'exécutent que les fichiers de configuration locaux

## L'interpréteur de commandes

### Processus d'interprétation

1. **Lecture** : Le shell lit la ligne de commande saisie
2. **Analyse lexicale** : Divise la ligne en tokens (mots, opérateurs)
3. **Analyse syntaxique** : Construit une structure de commande
4. **Expansion** : Remplace les variables, jokers, etc.
5. **Exécution** : Lance le programme ou la fonction intégrée
6. **Affichage** : Présente la sortie à l'utilisateur

### Commandes intégrées vs externes

**Commandes intégrées** (builtins) :
- Implémentées directement dans le shell
- Plus rapides (pas de fork/exec)
- Exemples : `cd`, `echo`, `export`, `alias`

**Commandes externes** :
- Programmes séparés dans `/bin`, `/usr/bin`, etc.
- Lancés via fork/exec
- Exemples : `ls`, `grep`, `vim`

## Le prompt et l'invite de commande

### Structure du prompt

Le prompt par défaut (`PS1`) contient généralement :
- Nom d'utilisateur
- Nom d'hôte
- Répertoire courant (souvent abrégé avec `~`)
- Caractère final (`$` pour utilisateur normal, `#` pour root)

Exemple : `user@hostname:~/directory$`

### Personnalisation du prompt

Le prompt peut être entièrement personnalisé pour afficher :
- L'heure
- Le statut Git
- Les couleurs
- Des informations système
- Des emojis ou symboles

### Variables de prompt

- `PS1` : Prompt principal
- `PS2` : Prompt de continuation (lorsque la commande n'est pas complète)
- `PS3` : Prompt de sélection dans `select`
- `PS4` : Prompt de debugging

## Exécution de commandes

### Modes d'exécution

**Synchrones** :
- Le shell attend la fin de la commande
- Mode par défaut pour la plupart des commandes

**Asynchrones** :
- La commande s'exécute en arrière-plan
- Utilise `&` à la fin de la commande
- Le shell retourne immédiatement le prompt

### Gestion des processus

Chaque commande lance un ou plusieurs processus :
- **PID** : Process ID unique
- **PPID** : Parent Process ID (le shell)
- **Code de retour** : 0 pour succès, autre valeur pour erreur

### Redirections et pipes

Le shell gère automatiquement :
- Redirection d'entrée/sortie (`<`, `>`, `>>`)
- Pipes pour chaîner les commandes (`|`)
- Groupement de commandes (`()`, `{}`)

## Variables d'environnement

### Variables locales vs globales

**Variables locales** :
- Visibles uniquement dans le shell courant
- Créées avec `variable=value`

**Variables d'environnement** :
- Héritées par les processus enfants
- Créées avec `export variable=value`
- Accessibles via `env` ou `printenv`

### Variables importantes

- `PATH` : Liste des répertoires contenant les exécutables
- `HOME` : Répertoire personnel de l'utilisateur
- `USER` : Nom d'utilisateur
- `PWD` : Répertoire de travail courant
- `SHELL` : Shell par défaut
- `TERM` : Type de terminal

## Configuration et personnalisation

### Fichiers de configuration

**Fichiers système** (globaux) :
- `/etc/profile`
- `/etc/bash.bashrc` (Bash)
- `/etc/zsh/zshrc` (Zsh)

**Fichiers utilisateur** :
- `~/.profile` ou `~/.bash_profile`
- `~/.bashrc` ou `~/.zshrc`
- `~/.bash_logout`

### Alias et fonctions

**Alias** :
```bash
alias ll='ls -alF'
alias ..='cd ..'
```

**Fonctions** :
```bash
mkcd() {
    mkdir -p "$1" && cd "$1"
}
```

### Completion et historique

Le shell maintient :
- Un historique des commandes (`~/.bash_history`)
- Des capacités d'autocomplétion
- Des raccourcis clavier (`Ctrl+R` pour recherche historique)

## Conclusion

Comprendre les concepts de base des shells transforme votre approche du terminal. Au lieu de voir une simple ligne de commande, vous percevez un environnement de programmation complet avec :

- Un modèle d'interaction sophistiqué
- Des mécanismes d'expansion et d'exécution puissants
- Un système de configuration flexible
- Des capacités d'automatisation intégrées

Cette compréhension conceptuelle est la clé pour maîtriser non seulement les commandes individuelles, mais aussi leur orchestration en workflows complexes.

Dans les chapitres suivants, nous explorerons comment ces concepts se manifestent dans différents shells (Bash, Zsh, etc.) et comment les utiliser pour créer des environnements de travail personnalisés et efficaces.

Le shell n'est pas seulement un outil ; c'est un partenaire de programmation qui apprend et s'adapte à votre style de travail.
