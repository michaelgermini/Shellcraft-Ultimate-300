# Chapitre 04 - Multi-plateforme : Linux, macOS, Windows

## Table des matières
- [Introduction](#introduction)
- [L'universalité du terminal](#luniversalité-du-terminal)
- [Linux : la diversité des distributions](#linux--la-diversité-des-distributions)
- [macOS : UNIX avec une interface moderne](#macos--unix-avec-une-interface-moderne)
- [Windows : l'évolution vers le terminal moderne](#windows--lévolution-vers-le-terminal-moderne)
- [Similitudes et différences](#similitudes-et-différences)
- [Outils communs multi-plateforme](#outils-communs-multi-plateforme)
- [Migration et portabilité](#migration-et-portabilité)
- [WSL : Windows Subsystem for Linux](#wsl--windows-subsystem-for-linux)
- [Scripts multi-plateforme](#scripts-multi-plateforme)
- [Bonnes pratiques multi-plateforme](#bonnes-pratiques-multi-plateforme)
- [Conclusion](#conclusion)

## Introduction

Dans un monde où les développeurs et administrateurs système travaillent sur des environnements variés, la capacité à naviguer efficacement entre Linux, macOS et Windows devient essentielle. Chaque système offre ses propres forces et particularités, mais tous partagent des concepts fondamentaux hérités de l'écosystème UNIX.

Imaginez ces trois systèmes comme trois dialectes d'une même langue : ils partagent une grammaire commune (les concepts UNIX), mais chacun a ses expressions idiomatiques (commandes spécifiques). Un expert polyglotte peut communiquer efficacement dans tous ces dialectes, adaptant son discours tout en conservant la même pensée fondamentale.

## L'universalité du terminal

### Philosophie commune

**Héritage UNIX** :
- Tous les systèmes modernes descendent ou s'inspirent d'UNIX
- Concepts fondamentaux partagés : fichiers, processus, permissions
- Philosophie "tout est fichier" présente partout
- Redirections et pipelines fonctionnent de manière similaire

**Langage commun** :
```bash
# Ces concepts fonctionnent partout
ls -la                    # Lister les fichiers
grep "pattern" file.txt   # Rechercher
ps aux                    # Processus (syntaxe peut varier)
```

### Différences superficielles

**Noms de commandes** :
- Linux/macOS : `ls`, `grep`, `find`
- Windows CMD : `dir`, `findstr`, `dir /s`
- Windows PowerShell : `Get-ChildItem`, `Select-String`, `Get-ChildItem -Recurse`

**Chemins** :
- Linux/macOS : `/home/user/documents`
- Windows : `C:\Users\user\Documents`

## Linux : la diversité des distributions

### Caractéristiques générales

**Points forts** :
- Open source et libre
- Grande diversité de distributions
- Contrôle total sur le système
- Idéal pour les serveurs et le développement

**Distributions populaires** :

**Debian/Ubuntu** :
```bash
# Gestionnaire de paquets : apt
sudo apt update
sudo apt install package_name
sudo apt upgrade
```

**Red Hat/CentOS/Fedora** :
```bash
# Gestionnaire de paquets : dnf/yum
sudo dnf install package_name
sudo yum install package_name  # Anciennes versions
```

**Arch Linux** :
```bash
# Gestionnaire de paquets : pacman
sudo pacman -Syu              # Mise à jour complète
sudo pacman -S package_name    # Installation
```

**openSUSE** :
```bash
# Gestionnaire de paquets : zypper
sudo zypper refresh
sudo zypper install package_name
```

### Environnements de bureau

**Terminaux populaires** :
- **GNOME Terminal** : Interface moderne et intuitive
- **Konsole** : Terminal KDE avec fonctionnalités avancées
- **Terminator** : Multi-panneaux et personnalisable
- **Alacritty** : Terminal GPU-accéléré ultra-rapide

**Configuration** :
```bash
# Fichier de configuration Bash
~/.bashrc
~/.bash_profile

# Fichier de configuration Zsh
~/.zshrc
```

## macOS : UNIX avec une interface moderne

### Fondations UNIX

**Certification POSIX** :
- macOS est certifié UNIX (Single UNIX Specification)
- Shell Bash par défaut (jusqu'à macOS Catalina)
- Zsh par défaut depuis macOS Catalina (10.15+)
- Outils UNIX complets disponibles

**Terminal natif** :
```bash
# Terminal.app : Terminal intégré macOS
# iTerm2 : Terminal amélioré avec fonctionnalités avancées
```

### Particularités macOS

**Chemins système** :
```bash
# Répertoires système
/System/Library/          # Bibliothèques système
/Library/                 # Bibliothèques globales
~/Library/                # Bibliothèques utilisateur
/Applications/            # Applications installées
/usr/local/               # Logiciels tiers
```

**Gestionnaire de paquets** :
```bash
# Homebrew : gestionnaire de paquets populaire
brew install package_name
brew update
brew upgrade

# MacPorts : alternative à Homebrew
sudo port install package_name
```

**Commandes spécifiques** :
```bash
# Afficher les informations système
system_profiler SPSoftwareDataType

# Gérer les services système
launchctl list            # Lister les services
launchctl start service   # Démarrer un service
launchctl stop service    # Arrêter un service

# Gérer les applications
open -a "Application Name"
open .                    # Ouvrir le Finder dans le répertoire courant
```

### Intégration avec l'écosystème Apple

**Scripts AppleScript** :
```bash
# Exécuter AppleScript depuis le terminal
osascript -e 'tell application "Finder" to display dialog "Hello"'

# Automatisation avec Automator
# Intégration avec les services macOS
```

## Windows : l'évolution vers le terminal moderne

### Évolution historique

**CMD (Command Prompt)** :
```cmd
REM Ancien shell Windows
dir
cd C:\Users
copy file1.txt file2.txt
```

**PowerShell** :
```powershell
# Shell moderne et puissant
Get-ChildItem
Set-Location C:\Users
Copy-Item file1.txt file2.txt

# Objets au lieu de texte
Get-Process | Where-Object {$_.CPU -gt 100}
```

**Windows Terminal** :
- Terminal moderne avec onglets
- Support de plusieurs shells (CMD, PowerShell, WSL)
- Personnalisation avancée
- Intégration avec Windows 11

### Windows Subsystem for Linux (WSL)

**WSL 1 vs WSL 2** :
```bash
# WSL 1 : Couche de compatibilité
# WSL 2 : Machine virtuelle Linux complète

# Vérifier la version
wsl --list --verbose

# Installer une distribution
wsl --install -d Ubuntu
wsl --install -d Debian
```

**Utilisation de WSL** :
```bash
# Accéder à WSL depuis Windows
wsl

# Exécuter une commande Linux depuis Windows
wsl ls -la

# Accéder aux fichiers Windows depuis WSL
cd /mnt/c/Users/username/Documents
```

**Intégration avec Windows** :
```powershell
# Accéder aux fichiers Linux depuis PowerShell
cd \\wsl$\Ubuntu\home\username

# Exécuter des commandes Linux depuis PowerShell
wsl bash -c "ls -la"
```

## Similitudes et différences

### Tableau comparatif des commandes

| Action | Linux/macOS | Windows CMD | Windows PowerShell |
|--------|-------------|-------------|-------------------|
| Lister fichiers | `ls -la` | `dir` | `Get-ChildItem` |
| Changer répertoire | `cd /path` | `cd C:\path` | `Set-Location C:\path` |
| Copier fichier | `cp file1 file2` | `copy file1 file2` | `Copy-Item file1 file2` |
| Déplacer fichier | `mv file1 file2` | `move file1 file2` | `Move-Item file1 file2` |
| Supprimer fichier | `rm file` | `del file` | `Remove-Item file` |
| Rechercher texte | `grep pattern file` | `findstr pattern file` | `Select-String pattern file` |
| Processus | `ps aux` | `tasklist` | `Get-Process` |
| Variables env | `echo $VAR` | `echo %VAR%` | `$env:VAR` |

### Chemins et séparateurs

**Séparateurs de répertoires** :
```bash
# Linux/macOS
/path/to/file

# Windows
C:\path\to\file

# Dans les scripts multi-plateforme
# Utiliser des variables ou fonctions de normalisation
```

**Chemins relatifs vs absolus** :
```bash
# Linux/macOS
./script.sh              # Répertoire courant
../parent/script.sh      # Répertoire parent
~/documents/file.txt     # Répertoire home

# Windows
.\script.bat             # Répertoire courant
..\parent\script.bat     # Répertoire parent
%USERPROFILE%\Documents\file.txt  # Répertoire home
```

## Outils communs multi-plateforme

### Outils universels

**Git** :
```bash
# Fonctionne identiquement partout
git init
git add .
git commit -m "Message"
git push origin main
```

**Docker** :
```bash
# Même syntaxe sur toutes les plateformes
docker build -t myimage .
docker run -d -p 80:80 myimage
docker ps
```

**Node.js / npm** :
```bash
# Identique partout
npm install package_name
npm run script
node app.js
```

**Python** :
```bash
# Même syntaxe
python script.py
pip install package_name
```

### Outils de développement

**Éditeurs de texte** :
- **Vim/Neovim** : Fonctionne partout
- **Emacs** : Disponible sur toutes les plateformes
- **VS Code** : Éditeur moderne multi-plateforme

**Gestionnaires de versions** :
- **Git** : Standard universel
- **Mercurial** : Alternative multi-plateforme
- **SVN** : Encore utilisé dans certains environnements

## Migration et portabilité

### Détection de la plateforme

**Script Bash multi-plateforme** :
```bash
#!/bin/bash
# Détecter le système d'exploitation

detect_os() {
    case "$(uname -s)" in
        Linux*)     echo "Linux" ;;
        Darwin*)    echo "macOS" ;;
        CYGWIN*)   echo "Windows (Cygwin)" ;;
        MINGW*)    echo "Windows (MinGW)" ;;
        MSYS*)     echo "Windows (MSYS)" ;;
        *)         echo "Unknown" ;;
    esac
}

OS=$(detect_os)
echo "Système détecté: $OS"

# Commandes conditionnelles
if [ "$OS" = "Linux" ]; then
    PACKAGE_MANAGER="apt"  # ou dnf, pacman selon la distro
elif [ "$OS" = "macOS" ]; then
    PACKAGE_MANAGER="brew"
fi
```

**Script PowerShell multi-plateforme** :
```powershell
# Détecter le système
$IsWindows = $PSVersionTable.Platform -eq "Win32NT"
$IsLinux = $PSVersionTable.Platform -eq "Unix"
$IsMacOS = $IsLinux -and (uname -s) -eq "Darwin"

if ($IsWindows) {
    Write-Host "Windows détecté"
} elseif ($IsMacOS) {
    Write-Host "macOS détecté"
} elseif ($IsLinux) {
    Write-Host "Linux détecté"
}
```

### Normalisation des chemins

**Fonction de normalisation** :
```bash
#!/bin/bash
# Normaliser les chemins pour la portabilité

normalize_path() {
    local path="$1"
    
    # Remplacer les backslashes par des slashes
    path="${path//\\//}"
    
    # Supprimer les doubles slashes
    path="${path//\/\///}"
    
    echo "$path"
}

# Utilisation
PATH1=$(normalize_path "C:\\Users\\Name\\Documents")
PATH2=$(normalize_path "/home/name/documents")
```

## WSL : Windows Subsystem for Linux

### Installation et configuration

**Installation WSL 2** :
```powershell
# Activer WSL
wsl --install

# Installer une distribution spécifique
wsl --install -d Ubuntu-22.04
wsl --install -d Debian
wsl --install -d openSUSE-Leap-15-4

# Lister les distributions disponibles
wsl --list --online
```

**Configuration WSL** :
```bash
# Fichier de configuration WSL
# C:\Users\<username>\.wslconfig

[wsl2]
memory=4GB
processors=2
swap=2GB
localhostForwarding=true
```

### Intégration avec Windows

**Accès aux fichiers Windows** :
```bash
# Depuis WSL, accéder aux fichiers Windows
cd /mnt/c/Users/username/Documents

# Monter un lecteur réseau
sudo mkdir /mnt/z
sudo mount -t drvfs Z: /mnt/z
```

**Exécution de commandes Windows** :
```bash
# Depuis WSL, exécuter des commandes Windows
cmd.exe /c "dir C:\Users"
powershell.exe -Command "Get-ChildItem C:\Users"
```

**Exécution de commandes Linux depuis Windows** :
```powershell
# Depuis PowerShell, exécuter des commandes Linux
wsl ls -la
wsl bash -c "cd /home/user && ls -la"
```

## Scripts multi-plateforme

### Script Bash portable

**Exemple complet** :
```bash
#!/bin/bash
# Script multi-plateforme

set -euo pipefail

# Détection de la plateforme
detect_platform() {
    case "$(uname -s)" in
        Linux*)     echo "linux" ;;
        Darwin*)    echo "macos" ;;
        CYGWIN*|MINGW*|MSYS*) echo "windows" ;;
        *)         echo "unknown" ;;
    esac
}

PLATFORM=$(detect_platform)

# Installation selon la plateforme
install_package() {
    local package="$1"
    
    case "$PLATFORM" in
        linux)
            if command -v apt >/dev/null 2>&1; then
                sudo apt install "$package"
            elif command -v dnf >/dev/null 2>&1; then
                sudo dnf install "$package"
            elif command -v pacman >/dev/null 2>&1; then
                sudo pacman -S "$package"
            fi
            ;;
        macos)
            if command -v brew >/dev/null 2>&1; then
                brew install "$package"
            fi
            ;;
        windows)
            if command -v wsl >/dev/null 2>&1; then
                wsl sudo apt install "$package"
            fi
            ;;
    esac
}

# Utilisation
install_package "git"
```

### Script PowerShell portable

**Exemple complet** :
```powershell
# Script PowerShell multi-plateforme

function Get-Platform {
    if ($PSVersionTable.Platform -eq "Win32NT") {
        return "Windows"
    } elseif ($PSVersionTable.Platform -eq "Unix") {
        $uname = (uname -s)
        if ($uname -eq "Darwin") {
            return "macOS"
        } else {
            return "Linux"
        }
    }
    return "Unknown"
}

$Platform = Get-Platform
Write-Host "Plateforme détectée: $Platform"

# Actions selon la plateforme
switch ($Platform) {
    "Windows" {
        Write-Host "Utilisation des commandes Windows"
        Get-ChildItem
    }
    "macOS" {
        Write-Host "Utilisation des commandes macOS"
        Get-ChildItem
    }
    "Linux" {
        Write-Host "Utilisation des commandes Linux"
        Get-ChildItem
    }
}
```

## Bonnes pratiques multi-plateforme

### Écriture de scripts portables

**Utiliser des chemins relatifs** :
```bash
#!/bin/bash
# Préférer les chemins relatifs
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
cd "$SCRIPT_DIR"
```

**Éviter les chemins absolus spécifiques** :
```bash
# ❌ Mauvais
cd /home/user/project

# ✅ Bon
cd "$HOME/project"
# ou
cd ~/project
```

**Gérer les différences de fin de ligne** :
```bash
# Convertir les fins de ligne
dos2unix script.sh    # Windows vers Unix
unix2dos script.sh    # Unix vers Windows

# Git peut gérer automatiquement
git config core.autocrlf true   # Windows
git config core.autocrlf input  # Linux/macOS
```

### Tests multi-plateforme

**Matrice de tests** :
```yaml
# Exemple GitHub Actions
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
runs-on: ${{ matrix.os }}
```

**Tests locaux** :
```bash
#!/bin/bash
# Tester sur différentes plateformes

test_on_platforms() {
    local script="$1"
    
    # Linux
    docker run --rm -v "$PWD:/workspace" ubuntu:latest bash -c "cd /workspace && bash $script"
    
    # macOS (nécessite un Mac)
    # bash "$script"
    
    # Windows (via WSL)
    wsl bash "$script"
}
```

### Documentation multi-plateforme

**README avec instructions par plateforme** :
```markdown
# Installation

## Linux
```bash
sudo apt install package_name
```

## macOS
```bash
brew install package_name
```

## Windows
```powershell
# Via WSL
wsl sudo apt install package_name

# Ou via Chocolatey
choco install package_name
```
```

## Conclusion

La maîtrise multi-plateforme représente une compétence essentielle dans l'environnement technique moderne. Comprendre les similitudes et différences entre Linux, macOS et Windows permet non seulement de travailler efficacement sur n'importe quel système, mais aussi de créer des solutions portables et robustes.

Les outils modernes comme WSL, Docker, et les langages interprétés réduisent les barrières entre les plateformes, mais la compréhension profonde des différences fondamentales reste cruciale pour créer des solutions vraiment portables.

Dans le chapitre suivant, nous explorerons les différences entre terminal et console, clarifiant ces concepts souvent confus mais fondamentaux pour comprendre l'environnement de ligne de commande.

