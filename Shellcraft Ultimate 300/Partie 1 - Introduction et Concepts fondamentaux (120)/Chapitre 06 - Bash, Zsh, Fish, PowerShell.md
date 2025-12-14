# Chapitre 06 - Bash, Zsh, Fish, PowerShell : comparaison

## Table des matières
- [Introduction](#introduction)
- [Bash : le standard universel](#bash--le-standard-universel)
- [Zsh : l'évolution moderne](#zsh--lévolution-moderne)
- [Fish : l'expérience utilisateur optimale](#fish--lexpérience-utilisateur-optimale)
- [PowerShell : la puissance orientée objets](#powershell--la-puissance-orientée-objets)
- [Comparaison détaillée](#comparaison-détaillée)
- [Critères de choix](#critères-de-choix)
- [Migration entre shells](#migration-entre-shells)
- [Configuration et personnalisation](#configuration-et-personnalisation)
- [Conclusion](#conclusion)

## Introduction

Le choix d'un shell détermine fondamentalement votre expérience de travail avec le terminal. Chaque shell offre un équilibre différent entre compatibilité, fonctionnalités et facilité d'utilisation. Comprendre leurs forces et faiblesses permet de faire un choix éclairé adapté à vos besoins spécifiques.

Imaginez les shells comme différents types de véhicules : Bash est la voiture universelle fiable et omniprésente ; Zsh est la berline moderne avec toutes les options ; Fish est la voiture de sport intuitive et réactive ; PowerShell est le camion utilitaire puissant pour les charges lourdes. Chacun excelle dans son domaine, et le meilleur choix dépend de votre destination.

## Bash : le standard universel

### Caractéristiques principales

**Points forts** :
- Disponible par défaut sur presque tous les systèmes UNIX-like
- Compatibilité maximale avec les scripts existants
- Documentation exhaustive et communauté vaste
- Standard POSIX, garantissant la portabilité

**Points faibles** :
- Syntaxe parfois verbeuse
- Autocomplétion basique sans configuration
- Pas de correction automatique des erreurs
- Interface utilisateur spartiate par défaut

### Syntaxe et fonctionnalités

**Variables et expansions** :
```bash
#!/bin/bash
# Variables simples
NAME="Alice"
echo "Hello, $NAME"

# Tableaux
FRUITS=("apple" "banana" "orange")
echo "${FRUITS[0]}"  # apple

# Expansion arithmétique
RESULT=$((5 + 3))
echo "$RESULT"  # 8
```

**Fonctions** :
```bash
# Définition de fonction
greet() {
    local name="$1"
    echo "Hello, $name!"
}

greet "World"  # Hello, World!
```

**Gestion d'erreurs** :
```bash
#!/bin/bash
set -euo pipefail  # Mode strict

# -e : arrêter sur erreur
# -u : erreur si variable non définie
# -o pipefail : erreur dans les pipelines
```

### Configuration de base

**Fichiers de configuration** :
```bash
# ~/.bashrc : Configuration interactive
# ~/.bash_profile : Configuration de login
# ~/.bash_aliases : Alias personnalisés

# Exemple ~/.bashrc
export EDITOR="vim"
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
```

**Prompt personnalisé** :
```bash
# ~/.bashrc
PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
# Résultat: user@hostname:/current/directory$ 
```

### Cas d'usage

**Quand utiliser Bash** :
- Scripts système et administration
- Compatibilité maximale requise
- Environnements de production stricts
- Apprentissage des bases UNIX

## Zsh : l'évolution moderne

### Caractéristiques principales

**Points forts** :
- Autocomplétion avancée et intelligente
- Correction automatique des fautes de frappe
- Thèmes et plugins riches (Oh My Zsh)
- Compatibilité avec Bash
- Historique partagé entre sessions

**Points faibles** :
- Légèrement plus lent que Bash
- Configuration initiale plus complexe
- Certaines fonctionnalités nécessitent des plugins

### Fonctionnalités avancées

**Globbing étendu** :
```zsh
# Globbing récursif
ls **/*.txt              # Tous les .txt récursivement
ls **/*.{txt,md}         # Plusieurs extensions
ls **/*(.)               # Seulement les fichiers
ls **/*(/)               # Seulement les répertoires
```

**Correction automatique** :
```zsh
# ~/.zshrc
setopt correct           # Correction de commandes
setopt correct_all       # Correction de tous les arguments

# Exemple
$ sl
zsh: correct 'sl' to 'ls' [nyae]? y
```

**Historique amélioré** :
```zsh
# ~/.zshrc
setopt share_history          # Historique partagé
setopt inc_append_history     # Ajout immédiat
setopt hist_ignore_dups       # Ignorer doublons
setopt hist_find_no_dups      # Pas de doublons dans recherche
```

### Oh My Zsh

**Installation** :
```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

**Thèmes populaires** :
```zsh
# ~/.zshrc
ZSH_THEME="robbyrussell"      # Thème classique
ZSH_THEME="agnoster"          # Thème avec Git
ZSH_THEME="powerlevel10k/powerlevel10k"  # Thème moderne
```

**Plugins utiles** :
```zsh
# ~/.zshrc
plugins=(
    git           # Intégration Git
    docker        # Complétion Docker
    kubectl       # Complétion Kubernetes
    zsh-autosuggestions  # Suggestions automatiques
    zsh-syntax-highlighting  # Coloration syntaxique
)
```

### Cas d'usage

**Quand utiliser Zsh** :
- Développement quotidien
- Interface utilisateur riche souhaitée
- Productivité maximale recherchée
- macOS (shell par défaut depuis Catalina)

## Fish : l'expérience utilisateur optimale

### Caractéristiques principales

**Points forts** :
- Configuration minimale requise
- Autocomplétion intelligente par défaut
- Coloration syntaxique intégrée
- Documentation intégrée excellente
- Interface utilisateur moderne

**Points faibles** :
- Incompatibilité avec Bash (syntaxe différente)
- Moins répandu que Bash/Zsh
- Certaines fonctionnalités avancées limitées

### Fonctionnalités uniques

**Autocomplétion intelligente** :
```fish
# Fish suggère automatiquement les commandes
$ git sta[TAB]
# Suggestions: status, stash, stage

# Complétion contextuelle
$ git commit -m "mes[TAB]
# Suggestions basées sur l'historique Git
```

**Coloration syntaxique** :
```fish
# Fish colore automatiquement la syntaxe
# Vert = commande valide
# Rouge = commande invalide
# Bleu = option valide
```

**Documentation intégrée** :
```fish
# Aide contextuelle
$ help git
# Affiche l'aide pour Git directement dans le terminal
```

### Configuration

**Fichier de configuration** :
```fish
# ~/.config/fish/config.fish

# Variables d'environnement
set -gx EDITOR "vim"
set -gx PATH $PATH ~/bin

# Alias
alias ll "ls -lah"
alias gs "git status"

# Fonctions
function greet
    echo "Hello, $argv[1]!"
end
```

**Thèmes** :
```fish
# Installer un thème
fisher install oh-my-fish/theme-bobthefish

# Configuration du thème
set -g theme_color_scheme dark
set -g theme_display_git yes
```

### Cas d'usage

**Quand utiliser Fish** :
- Débutants cherchant une expérience fluide
- Développement interactif quotidien
- Environnements où la compatibilité Bash n'est pas critique
- Productivité immédiate sans configuration

## PowerShell : la puissance orientée objets

### Caractéristiques principales

**Points forts** :
- Modèle orienté objets puissant
- Intégration native avec Windows
- Gestion avancée des erreurs
- Accès aux APIs .NET
- Multi-plateforme (PowerShell Core)

**Points faibles** :
- Syntaxe différente de Bash/Zsh
- Courbe d'apprentissage pour les utilisateurs UNIX
- Plus verbeux pour les tâches simples

### Modèle orienté objets

**Objets au lieu de texte** :
```powershell
# PowerShell retourne des objets, pas du texte
Get-Process | Where-Object {$_.CPU -gt 100}

# Accès aux propriétés
$process = Get-Process notepad
$process.Name
$process.Id
$process.CPU

# Méthodes disponibles
$process.Kill()
```

**Pipelines d'objets** :
```powershell
# Pipeline d'objets
Get-ChildItem | 
    Where-Object {$_.Length -gt 1MB} | 
    Sort-Object Length -Descending | 
    Select-Object Name, Length, LastWriteTime
```

### Fonctionnalités avancées

**Gestion d'erreurs robuste** :
```powershell
# Try-Catch-Finally
try {
    Get-Content "nonexistent.txt"
} catch {
    Write-Error "Erreur: $_"
} finally {
    Write-Host "Nettoyage effectué"
}

# Erreurs non bloquantes
Get-Content "file1.txt", "file2.txt" -ErrorAction SilentlyContinue
```

**Modules et fonctions** :
```powershell
# Créer une fonction
function Get-Greeting {
    param(
        [string]$Name = "World"
    )
    return "Hello, $Name!"
}

Get-Greeting -Name "Alice"

# Modules
Import-Module ActiveDirectory
Get-ADUser -Filter *
```

### PowerShell Core (multi-plateforme)

**Installation sur Linux/macOS** :
```bash
# Ubuntu/Debian
wget https://github.com/PowerShell/PowerShell/releases/download/v7.3.0/powershell_7.3.0-1.deb_amd64.deb
sudo dpkg -i powershell_7.3.0-1.deb_amd64.deb

# macOS
brew install --cask powershell

# Utilisation
pwsh
```

**Compatibilité** :
```powershell
# PowerShell Core est compatible avec PowerShell Windows
# Mais certaines commandes Windows spécifiques ne sont pas disponibles
```

### Cas d'usage

**Quand utiliser PowerShell** :
- Administration Windows
- Scripts nécessitant l'accès aux APIs .NET
- Gestion de systèmes Microsoft (Active Directory, Exchange)
- Automatisation d'infrastructure Windows
- Scripts nécessitant une gestion d'erreurs avancée

## Comparaison détaillée

### Tableau comparatif

| Caractéristique | Bash | Zsh | Fish | PowerShell |
|----------------|------|-----|------|------------|
| **Compatibilité POSIX** | ✅ Complète | ✅ Presque complète | ❌ Non | ❌ Non |
| **Autocomplétion** | ⚠️ Basique | ✅ Avancée | ✅ Excellente | ✅ Avancée |
| **Coloration syntaxique** | ⚠️ Via plugins | ✅ Via plugins | ✅ Intégrée | ✅ Intégrée |
| **Correction automatique** | ❌ Non | ✅ Oui | ✅ Oui | ⚠️ Limitée |
| **Orienté objets** | ❌ Non | ❌ Non | ❌ Non | ✅ Oui |
| **Configuration** | ⚠️ Manuelle | ⚠️ Manuelle | ✅ Minimale | ⚠️ Manuelle |
| **Performance** | ✅ Rapide | ✅ Rapide | ✅ Rapide | ⚠️ Plus lent |
| **Documentation** | ✅ Excellente | ✅ Excellente | ✅ Excellente | ✅ Excellente |
| **Communauté** | ✅ Très large | ✅ Large | ⚠️ Moyenne | ✅ Large |
| **Multi-plateforme** | ✅ Oui | ✅ Oui | ✅ Oui | ✅ Oui |

### Exemples comparatifs

**Liste de fichiers** :
```bash
# Bash/Zsh
ls -lah | grep "\.txt$" | head -10

# Fish
ls -lah | grep "\.txt$" | head -10
# (Même syntaxe mais avec meilleure autocomplétion)

# PowerShell
Get-ChildItem -Force | Where-Object {$_.Name -like "*.txt"} | Select-Object -First 10
```

**Gestion des processus** :
```bash
# Bash/Zsh
ps aux | grep "nginx" | awk '{print $2}' | xargs kill

# Fish
ps aux | grep "nginx" | awk '{print $2}' | xargs kill

# PowerShell
Get-Process nginx | Stop-Process
```

**Manipulation de fichiers** :
```bash
# Bash/Zsh
find . -name "*.log" -type f -mtime +30 -delete

# Fish
find . -name "*.log" -type f -mtime +30 -delete

# PowerShell
Get-ChildItem -Recurse -Filter "*.log" | 
    Where-Object {$_.LastWriteTime -lt (Get-Date).AddDays(-30)} | 
    Remove-Item
```

## Critères de choix

### Pour les débutants

**Recommandation : Fish**
- Configuration minimale
- Autocomplétion excellente par défaut
- Documentation intégrée
- Interface intuitive

### Pour la compatibilité maximale

**Recommandation : Bash**
- Standard universel
- Disponible partout
- Scripts existants compatibles
- Documentation exhaustive

### Pour la productivité

**Recommandation : Zsh**
- Autocomplétion avancée
- Plugins riches (Oh My Zsh)
- Correction automatique
- Compatible avec Bash

### Pour Windows/Administration Microsoft

**Recommandation : PowerShell**
- Intégration native Windows
- Accès aux APIs .NET
- Gestion d'erreurs robuste
- Orienté objets

### Pour le développement moderne

**Recommandation : Zsh ou Fish**
- Expérience utilisateur optimale
- Outils modernes intégrés
- Productivité maximale
- Communauté active

## Migration entre shells

### De Bash vers Zsh

**Migration simple** :
```bash
# Copier la configuration Bash
cp ~/.bashrc ~/.zshrc

# Adapter les spécificités
# Bash: HISTSIZE=1000
# Zsh: HISTSIZE=1000 (identique)

# Ajouter les fonctionnalités Zsh
echo 'setopt share_history' >> ~/.zshrc
echo 'setopt correct' >> ~/.zshrc
```

**Compatibilité** :
```zsh
# Zsh peut exécuter la plupart des scripts Bash
# Activer la compatibilité si nécessaire
emulate bash -c 'source ~/.bashrc'
```

### De Bash/Zsh vers Fish

**Migration plus complexe** :
```fish
# Fish a une syntaxe différente
# Nécessite de réécrire les scripts

# Bash/Zsh
if [ -f "$HOME/.bashrc" ]; then
    source "$HOME/.bashrc"
fi

# Fish équivalent
if test -f "$HOME/.bashrc"
    source "$HOME/.bashrc"
end
```

**Outils de conversion** :
```bash
# Certains outils peuvent aider à la conversion
# Mais la réécriture manuelle est souvent nécessaire
```

### Vers PowerShell

**Migration complète** :
```powershell
# PowerShell nécessite une réécriture complète
# Mais offre des fonctionnalités puissantes

# Bash équivalent
for file in *.txt; do
    echo "$file"
done

# PowerShell équivalent
Get-ChildItem *.txt | ForEach-Object {
    Write-Host $_.Name
}
```

## Configuration et personnalisation

### Configuration Bash

**Fichier de configuration complet** :
```bash
# ~/.bashrc
# Options du shell
set -o vi                    # Mode vi pour l'édition
shopt -s histappend         # Historique append
shopt -s checkwinsize       # Vérifier taille fenêtre
shopt -s globstar           # ** pour récursif

# Variables d'environnement
export EDITOR="vim"
export VISUAL="vim"
export PAGER="less"

# Alias
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
alias grep='grep --color=auto'
alias fgrep='fgrep --color=auto'
alias egrep='egrep --color=auto'

# Fonctions
mkcd() {
    mkdir -p "$1" && cd "$1"
}
```

### Configuration Zsh

**Configuration avec Oh My Zsh** :
```zsh
# ~/.zshrc
export ZSH="$HOME/.oh-my-zsh"
ZSH_THEME="robbyrussell"
plugins=(git docker kubectl)

source $ZSH/oh-my-zsh.sh

# Configuration personnalisée
export EDITOR="vim"
alias ll='ls -alF'
```

### Configuration Fish

**Configuration minimale** :
```fish
# ~/.config/fish/config.fish
set -gx EDITOR vim
set -gx PATH $PATH ~/bin

alias ll "ls -lah"
alias gs "git status"

# Thème
set -g theme_color_scheme dark
```

### Configuration PowerShell

**Profil PowerShell** :
```powershell
# Windows: $PROFILE
# Linux/macOS: ~/.config/powershell/Microsoft.PowerShell_profile.ps1

# Alias
Set-Alias -Name ll -Value Get-ChildItem
Set-Alias -Name grep -Value Select-String

# Fonctions
function Get-Greeting {
    param([string]$Name = "World")
    "Hello, $Name!"
}

# Thème (via Oh My Posh)
oh-my-posh init pwsh | Invoke-Expression
```

## Conclusion

Le choix d'un shell est une décision personnelle qui dépend de vos besoins spécifiques, de votre environnement de travail, et de vos préférences. Bash reste le standard universel pour la compatibilité maximale, Zsh offre la meilleure expérience moderne avec compatibilité Bash, Fish fournit la meilleure expérience utilisateur sans configuration, et PowerShell excelle dans l'administration Windows et les environnements orientés objets.

La meilleure approche est souvent d'apprendre plusieurs shells : utiliser Bash pour la compatibilité et les scripts système, Zsh ou Fish pour le développement quotidien, et PowerShell pour l'administration Windows. Cette polyvalence vous rendra plus efficace et adaptable dans différents environnements.

Dans le chapitre suivant, nous explorerons les principes de scripting, qui s'appliquent à tous ces shells mais avec des nuances spécifiques à chacun.

