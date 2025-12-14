# Chapitre 121 - Introduction à Windows PowerShell

> "PowerShell n'est pas seulement un shell - c'est une révolution dans la façon dont nous pensons l'administration système sous Windows." - Jeffrey Snover, créateur de PowerShell

## Introduction : De CMD.EXE à PowerShell

Si les chapitres précédents nous ont fait voyager dans l'univers UNIX/Linux avec Bash et ses dérivés, nous abordons maintenant un écosystème différent mais tout aussi riche : Windows et son shell moderne, PowerShell. Nécessairement différent de ses cousins UNIX par sa philosophie et son architecture, PowerShell représente pourtant une convergence fascinante vers les mêmes objectifs d'automatisation et de productivité.

Dans ce chapitre inaugural de notre exploration Windows/PowerShell, nous poserons les bases de cet environnement unique, en comprenant son histoire, sa philosophie, et ses principes fondamentaux qui en font un outil indispensable pour tout administrateur système moderne.

## Section 1 : Genèse et évolution de PowerShell

### 1.1 Avant PowerShell : l'ère des batch files et CMD

Avant 2006, l'automatisation sous Windows reposait sur des technologies primitives comparées aux shells UNIX :

```batch
:: Script batch traditionnel - verbeux et limité
@echo off
setlocal enabledelayedexpansion

REM Vérification des paramètres
if "%1"=="" (
    echo Usage: %0 ^<nom_fichier^>
    exit /b 1
)

REM Traitement du fichier
if exist "%1" (
    for /f "tokens=*" %%i in (%1) do (
        echo Traitement de: %%i
        REM Logique complexe ici...
    )
) else (
    echo Erreur: Fichier %1 introuvable
    exit /b 2
)

echo Terminé
exit /b 0
```

Ces scripts souffraient de limitations majeures :
- Syntaxe verbeuse et peu intuitive
- Gestion d'erreur primitive
- Manipulation de données limitée
- Pas d'accès programmatique aux API système
- Difficilement maintenable à grande échelle

### 1.2 La vision de Jeffrey Snover et l'équipe PowerShell

En 2002, Jeffrey Snover, architecte chez Microsoft, identifie un problème majeur : l'administration Windows manque cruellement d'outils d'automatisation puissants. Sa vision : créer un shell qui serait :

- **Orienté objet** : Tout est objet, pas seulement du texte
- **Extensible** : Architecture modulaire basée sur des cmdlets
- **Cohérent** : Conventions de nommage et comportements uniformes
- **Sécurisé** : Modèle d'exécution restrictif par défaut
- **Interopérable** : Avec .NET, COM, WMI, et les API Windows

### 1.3 Les versions majeures et leur évolution

| Version | Année | Nom | Innovations majeures |
|---------|-------|-----|---------------------|
| 1.0 | 2006 | Monad | Concepts de base, pipeline objet |
| 2.0 | 2009 | - | Remote management, modules, background jobs |
| 3.0 | 2012 | - | Workflow, scheduled jobs, module auto-loading |
| 4.0 | 2013 | - | Desired State Configuration (DSC) |
| 5.0 | 2016 | - | Classes personnalisées, OneGet package management |
| 5.1 | 2017 | - | Corrections et améliorations |
| 6.0 | 2018 | Core | Cross-platform (.NET Core), open-source |
| 7.0 | 2020 | - | Pipeline parallélisation, null coalescing |
| 7.2+ | 2021+ | - | Continual improvements |

### 1.4 PowerShell Core vs Windows PowerShell

Depuis PowerShell 6.0, nous distinguons deux implémentations :

**Windows PowerShell (5.1 et antérieur)** :
- Basé sur .NET Framework (Windows uniquement)
- Intégration profonde avec Windows
- Legacy support pour anciens systèmes

**PowerShell Core (6.0+)** :
- Basé sur .NET Core/Cross-platform
- Open-source (GitHub)
- Cross-platform (Windows, Linux, macOS)
- Focus sur la modernité et la performance

## Section 2 : Philosophie et architecture de PowerShell

### 2.1 Tout est objet : la révolution du pipeline

Contrairement aux shells traditionnels qui manipulent du texte, PowerShell traite des objets .NET :

```powershell
# PowerShell : objets natifs
Get-Process | Where-Object { $_.CPU -gt 10 } | Select-Object Name, CPU, Memory

# Equivalent Bash : manipulation de texte verbeuse
ps aux | awk '$3 > 10 {print $11, $3, $4}' | head -10
```

**Avantages du modèle objet :**
- **Type safety** : Pas de conversion implicite d'erreurs
- **IntelliSense** : Auto-completion intelligente
- **Méthodes riches** : Accès direct aux propriétés et méthodes
- **Performance** : Pas de parsing/convertion répété

### 2.2 Les cmdlets : le cœur de PowerShell

Les cmdlets sont des commandes natives PowerShell suivant une convention stricte :

```powershell
# Structure: Verbe-Nom
Get-Process      # Récupère des objets processus
Set-Location     # Définit l'emplacement actuel
New-Item        # Crée un nouvel élément
Remove-Item     # Supprime un élément
```

**Verbes standardisés :**
- `Get-` : Récupère des données
- `Set-` : Modifie des données
- `New-` : Crée de nouveaux objets
- `Remove-` : Supprime des objets
- `Add-` : Ajoute à un conteneur
- `Clear-` : Supprime le contenu sans supprimer le conteneur

### 2.3 Le pipeline objet : composition puissante

Le pipeline PowerShell passe des objets entiers, pas du texte :

```powershell
# Pipeline complexe avec objets
Get-Service |
    Where-Object { $_.Status -eq 'Running' } |
    Select-Object Name, DisplayName, StartType |
    Sort-Object Name |
    Format-Table -AutoSize
```

### 2.4 Le provider model : système de fichiers unifié

PowerShell abstrait l'accès aux données via des providers :

```powershell
# Providers intégrés
Get-PSProvider

# Accès unifié à différents "systèmes de fichiers"
Set-Location C:\                    # Système de fichiers
Set-Location HKLM:\                 # Registre Windows
Set-Location Cert:\CurrentUser\     # Magasin de certificats
Set-Location WSMan:\localhost\      # Configuration WS-Management
```

## Section 3 : Installation et premiers pas

### 3.1 Installation de PowerShell Core

**Sur Windows :**
```powershell
# Via Winget (recommandé)
winget install Microsoft.PowerShell

# Via Chocolatey
choco install powershell-core

# Via MSI direct depuis GitHub releases
```

**Sur Linux :**
```bash
# Ubuntu/Debian
wget -q https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update
sudo apt-get install -y powershell

# CentOS/RHEL
curl https://packages.microsoft.com/config/rhel/7/prod.repo | sudo tee /etc/yum.repos.d/microsoft.repo
sudo yum install -y powershell
```

**Sur macOS :**
```bash
# Via Homebrew
brew install --cask powershell

# Via direct download
```

### 3.2 Lancement et configuration de base

```powershell
# Lancement
pwsh  # PowerShell Core
powershell  # Windows PowerShell (legacy)

# Version et informations système
$PSVersionTable
Get-Host

# Configuration de l'exécution
Get-ExecutionPolicy
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# Profil utilisateur
$PROFILE
New-Item -Path $PROFILE -ItemType File -Force
```

### 3.3 L'environnement de développement

**ISE (Integrated Scripting Environment) - Legacy :**
```powershell
# Lancement de l'ISE
powershell_ise
```

**Visual Studio Code avec extension PowerShell :**
```json
// settings.json pour VS Code
{
    "powershell.powerShellDefaultVersion": "PowerShell Core",
    "powershell.enableProfileLoading": true,
    "powershell.codeFormatting.useCorrectCasing": true
}
```

## Section 4 : Concepts fondamentaux

### 4.1 Variables et types

```powershell
# Variables typées et non-typées
$maVariable = "Hello World"
[string]$chaineTypée = "Hello"
[int]$nombre = 42

# Tableaux et hashtables
$monTableau = 1, 2, 3, 4, 5
$maHashtable = @{
    Nom = "Alice"
    Age = 30
    Ville = "Paris"
}

# Accès aux propriétés
$maHashtable.Nom
$maHashtable["Nom"]
```

### 4.2 L'opérateur pipeline et les cmdlets de base

```powershell
# Navigation et fichiers
Get-ChildItem -Path C:\Windows\System32 -Filter *.dll | Select-Object Name, Length | Sort-Object Length -Descending

# Processus
Get-Process | Where-Object { $_.WorkingSet -gt 100MB } | Stop-Process -Confirm:$false

# Services
Get-Service | Where-Object { $_.Status -eq 'Stopped' } | Start-Service
```

### 4.3 L'aide intégrée

```powershell
# Aide détaillée
Get-Help Get-Process -Detailed

# Exemples d'utilisation
Get-Help Get-Process -Examples

# Recherche dans l'aide
Get-Help *process*

# Mise à jour de l'aide
Update-Help
```

## Section 5 : Différences culturelles avec UNIX/Linux

### 5.1 Philosophie différente

| Aspect | UNIX/Linux | Windows/PowerShell |
|--------|------------|-------------------|
| **Interface** | Fichiers et processus | API et objets COM/.NET |
| **Permissions** | POSIX (user/group/other) | NTFS ACLs complexes |
| **Configuration** | Fichiers texte | Registre + fichiers |
| **Extensibilité** | Scripts shell + C | Cmdlets + .NET assemblies |
| **Parallélisation** | fork/exec | Threads et async |

### 5.2 Approches équivalentes

```powershell
# UNIX : ps aux | grep nginx | wc -l
# PowerShell : (Get-Process nginx).Count

# UNIX : ls -la /etc/ | head -5
# PowerShell : Get-ChildItem C:\Windows | Select-Object -First 5

# UNIX : find /home -name "*.log" -mtime +7 -delete
# PowerShell : Get-ChildItem C:\Users\*\*.log -Recurse | Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-7) } | Remove-Item
```

### 5.3 Avantages comparatifs

**PowerShell excelle dans :**
- Intégration profonde avec Windows
- Gestion d'objets complexes (.NET, COM, WMI)
- Administration à distance (WinRM)
- Traitement de données structurées
- Intégration avec Azure/AWS

**Bash reste supérieur pour :**
- Traitement de texte pur
- Scripts légers et rapides
- Compatibilité POSIX
- Écosystème Linux natif

## Section 6 : Écosystème et communauté

### 6.1 PowerShell Gallery : le dépôt officiel

```powershell
# Recherche de modules
Find-Module *Azure*

# Installation
Install-Module -Name Az -AllowClobber

# Liste des modules installés
Get-InstalledModule

# Mise à jour
Update-Module
```

### 6.2 Communauté et ressources

**Ressources officielles :**
- [PowerShell Documentation](https://docs.microsoft.com/powershell/)
- [PowerShell Gallery](https://www.powershellgallery.com/)
- [PowerShell GitHub](https://github.com/PowerShell/PowerShell)

**Communautés :**
- [PowerShell.org](https://powershell.org/)
- Reddit r/PowerShell
- Stack Overflow
- Conferences : PowerShell + DevOps Global Summit

### 6.3 Outils complémentaires

```powershell
# Outils de développement
Install-Module -Name PSScriptAnalyzer    # Analyse statique
Install-Module -Name Pester             # Framework de test
Install-Module -Name PSReadLine         # Amélioration de la ligne de commande

# Outils d'administration
Install-Module -Name ActiveDirectory   # AD management
Install-Module -Name ExchangeOnlineManagement  # Exchange Online
Install-Module -Name Az                 # Azure management
```

## Conclusion : Un nouveau paradigme d'administration

PowerShell représente une rupture philosophique majeure avec les shells traditionnels. En traitant tout comme des objets plutôt que du texte, il offre une puissance d'expression et une sécurité sans précédent pour l'administration système Windows.

Dans les chapitres suivants, nous explorerons en profondeur :
- La programmation avancée avec PowerShell
- L'administration système et réseau
- L'automatisation avec DSC
- L'intégration avec le cloud
- Les bonnes pratiques et patterns avancés

---

**Exercice préparatoire :** Installez PowerShell Core sur votre système et explorez les cmdlets de base. Comparez l'expérience avec votre shell habituel (Bash, Zsh, etc.).

**Réflexion :** Comment PowerShell change-t-il votre perception de ce qu'un shell moderne devrait être ?

