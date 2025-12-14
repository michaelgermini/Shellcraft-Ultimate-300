# Chapitre 54 - Gestion des packages Snap et Flatpak

## Table des matières
- [Introduction](#introduction)
- [Philosophie des packages universels](#philosophie-des-packages-universels)
- [Snap : packages de Canonical](#snap-packages-de-canonical)
- [Flatpak : alternative indépendante](#flatpak-alternative-indépendante)
- [Installation et configuration](#installation-et-configuration)
- [Création de packages](#création-de-packages)
- [Sécurité et sandboxing](#sécurité-et-sandboxing)
- [Comparaison et choix](#comparaison-et-choix)
- [Conclusion](#conclusion)

## Introduction

Snap et Flatpak représentent l'évolution moderne de la gestion des packages sous Linux, offrant des solutions de déploiement universelles qui transcendent les barrières entre distributions. Ces technologies permettent d'installer des applications de manière isolée, sécurisée, et indépendante du système hôte.

Imaginez Snap et Flatpak comme des containers pour applications : chaque application voyage dans son propre environnement isolé, avec toutes ses dépendances, et peut être déployée sur n'importe quelle distribution Linux sans conflits avec le système hôte.

## Philosophie des packages universels

### Problèmes des packages traditionnels

**Limitations des gestionnaires de paquets classiques** :
- **Dépendances spécifiques** : Chaque distribution a ses propres versions
- **Conflits de bibliothèques** : Versions incompatibles entre applications
- **Fragmentation** : Packages différents pour chaque distribution
- **Maintenance complexe** : Nécessité de maintenir plusieurs versions

**Exemple de problème** :
```bash
# Application nécessite Python 3.9
# Distribution fournit Python 3.8
# Conflit avec autres applications nécessitant Python 3.8
```

### Avantages des packages universels

**Isolation** :
- Chaque application dans son propre environnement
- Pas de conflits de dépendances
- Mises à jour indépendantes

**Universalité** :
- Un seul package pour toutes les distributions
- Déploiement simplifié
- Maintenance centralisée

**Sécurité** :
- Sandboxing par défaut
- Permissions granulaires
- Mises à jour automatiques

## Snap : packages de Canonical

### Architecture Snap

**Composants principaux** :
- **snapd** : Daemon gérant les snaps
- **snap** : Outil en ligne de commande
- **Snap Store** : Dépôt centralisé de packages
- **snapcraft** : Outil de création de snaps

**Structure d'un snap** :
```
snap/
├── meta/           # Métadonnées (snap.yaml)
├── snap/           # Scripts et hooks
└── [application]/  # Fichiers de l'application
```

### Installation de Snap

**Installation sur différentes distributions** :
```bash
# Ubuntu (pré-installé depuis 16.04)
sudo apt update
sudo apt install snapd

# Debian
sudo apt update
sudo apt install snapd
sudo systemctl enable --now snapd.socket

# Fedora
sudo dnf install snapd
sudo systemctl enable --now snapd.socket

# Arch Linux (via AUR)
yay -S snapd
sudo systemctl enable --now snapd.socket

# CentOS/RHEL
sudo yum install epel-release
sudo yum install snapd
sudo systemctl enable --now snapd.socket
```

**Vérification de l'installation** :
```bash
# Vérifier la version
snap --version

# Vérifier le statut
sudo systemctl status snapd
```

### Utilisation de base

**Recherche de packages** :
```bash
# Rechercher un package
snap find application

# Rechercher avec filtre
snap find "text editor"

# Informations sur un package
snap info application
```

**Installation** :
```bash
# Installation simple
sudo snap install application

# Installation d'une version spécifique
sudo snap install application --channel=stable
sudo snap install application --channel=edge
sudo snap install application --channel=beta

# Installation classique (sans confinement strict)
sudo snap install application --classic

# Exemples
sudo snap install code --classic          # VS Code
sudo snap install docker
sudo snap install node --classic
```

**Gestion des snaps installés** :
```bash
# Lister les snaps installés
snap list

# Informations détaillées
snap list --all

# Mettre à jour un snap
sudo snap refresh application

# Mettre à jour tous les snaps
sudo snap refresh

# Désinstaller
sudo snap remove application
```

**Gestion des versions** :
```bash
# Voir les révisions disponibles
snap info application

# Revenir à une version précédente
sudo snap revert application

# Voir l'historique des révisions
snap list --all application
```

### Configuration avancée

**Channels et tracks** :
```bash
# Changer de channel
sudo snap refresh application --channel=edge

# Voir les channels disponibles
snap info application

# Tracks (versions majeures)
sudo snap install application --channel=2.0/stable
```

**Services et daemons** :
```bash
# Lister les services
snap services

# Contrôler un service
sudo snap start application.service
sudo snap stop application.service
sudo snap restart application.service

# Voir les logs
snap logs application
snap logs application.service
```

**Connexions et interfaces** :
```bash
# Voir les connexions
snap connections application

# Connexions disponibles
snap interfaces

# Connecter une interface
sudo snap connect application:interface system:interface

# Déconnecter
sudo snap disconnect application:interface
```

### Snap Store et dépôts

**Store officiel** :
```bash
# Rechercher dans le store
snap find keyword

# Informations du store
snap info application
```

**Snaps locaux** :
```bash
# Installer depuis un fichier local
sudo snap install application.snap --dangerous

# Installer depuis un répertoire
sudo snap install /path/to/snap --dangerous
```

## Flatpak : alternative indépendante

### Architecture Flatpak

**Composants principaux** :
- **flatpak** : Outil en ligne de commande
- **OSTree** : Système de fichiers versionné
- **Bubblewrap** : Sandboxing
- **Flatpak Builder** : Outil de création

**Structure** :
```
~/.local/share/flatpak/    # Applications utilisateur
/var/lib/flatpak/          # Applications système
```

### Installation de Flatpak

**Installation** :
```bash
# Ubuntu/Debian
sudo apt install flatpak

# Fedora (pré-installé)
sudo dnf install flatpak

# Arch Linux
sudo pacman -S flatpak

# openSUSE
sudo zypper install flatpak
```

**Configuration des dépôts** :
```bash
# Ajouter le dépôt Flathub (principal)
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo

# Lister les dépôts
flatpak remote-list

# Voir les détails d'un dépôt
flatpak remote-info flathub
```

### Utilisation de base

**Recherche** :
```bash
# Rechercher une application
flatpak search application

# Recherche détaillée
flatpak search --columns=name,description application
```

**Installation** :
```bash
# Installation depuis Flathub
flatpak install flathub application

# Installation système (pour tous les utilisateurs)
sudo flatpak install flathub application

# Installation d'une version spécifique
flatpak install flathub application//stable
flatpak install flathub application//beta

# Exemples
flatpak install flathub org.gnome.Gedit
flatpak install flathub com.visualstudio.code
flatpak install flathub org.mozilla.firefox
```

**Gestion des applications** :
```bash
# Lister les applications installées
flatpak list

# Applications système seulement
flatpak list --system

# Applications utilisateur seulement
flatpak list --user

# Informations sur une application
flatpak info application

# Mettre à jour
flatpak update application

# Mettre à jour toutes les applications
flatpak update

# Désinstaller
flatpak uninstall application
```

**Exécution** :
```bash
# Exécuter une application
flatpak run application

# Avec options
flatpak run --env=VAR=value application

# Voir les permissions
flatpak info --show-permissions application
```

### Gestion des runtimes

**Runtimes Flatpak** :
```bash
# Lister les runtimes installés
flatpak list --runtime

# Installer un runtime
flatpak install flathub org.freedesktop.Platform//22.08

# Mettre à jour les runtimes
flatpak update --runtime
```

**Runtimes disponibles** :
- **org.freedesktop.Platform** : Runtime de base
- **org.gnome.Platform** : Runtime GNOME
- **org.kde.Platform** : Runtime KDE

## Installation et configuration

### Configuration Snap

**Fichiers de configuration** :
```bash
# Configuration système
/etc/environment
# Ajouter: PATH="/snap/bin:$PATH"

# Configuration utilisateur
~/.bashrc
# Ajouter: export PATH="/snap/bin:$PATH"
```

**Paramètres système** :
```bash
# Voir la configuration
snap get system

# Configurer le proxy
sudo snap set system proxy.http="http://proxy:port"
sudo snap set system proxy.https="https://proxy:port"

# Désactiver les mises à jour automatiques
sudo snap set system refresh.timer=00:00~24:00/never
```

**Limites et quotas** :
```bash
# Voir les limites
snap get system

# Configurer les limites de refresh
sudo snap set system refresh.rate-limit=10
```

### Configuration Flatpak

**Fichiers de configuration** :
```bash
# Configuration système
/etc/flatpak/installations.conf

# Configuration utilisateur
~/.config/flatpak/installations.conf
```

**Permissions et sandboxing** :
```bash
# Voir les permissions d'une application
flatpak info --show-permissions application

# Modifier les permissions
flatpak override application --socket=wayland
flatpak override application --filesystem=home
flatpak override application --filesystem=/path/to/dir

# Réinitialiser les permissions
flatpak override --reset application
```

**Configuration des dépôts** :
```bash
# Ajouter un dépôt
flatpak remote-add repo-name https://example.com/repo.flatpakrepo

# Modifier un dépôt
flatpak remote-modify repo-name --url=https://new-url.com/repo

# Supprimer un dépôt
flatpak remote-delete repo-name

# Voir les applications d'un dépôt
flatpak remote-ls repo-name
```

## Création de packages

### Création d'un Snap

**Installation de Snapcraft** :
```bash
# Via snap
sudo snap install snapcraft --classic

# Via apt (Ubuntu)
sudo apt install snapcraft
```

**Structure de base** :
```yaml
# snap/snapcraft.yaml
name: my-application
version: '1.0'
summary: Description courte
description: Description détaillée
base: core22
grade: stable
confinement: strict

apps:
  my-app:
    command: bin/my-app
    plugs:
      - network
      - home

parts:
  my-app:
    plugin: nil
    source: .
    build-packages:
      - build-essential
```

**Build et publication** :
```bash
# Construire le snap
snapcraft

# Construire avec options
snapcraft --target-arch=amd64

# Tester localement
sudo snap install my-application.snap --dangerous

# Publier sur le store
snapcraft login
snapcraft register my-application
snapcraft push my-application.snap --release=stable
```

### Création d'un Flatpak

**Installation de Flatpak Builder** :
```bash
# Ubuntu/Debian
sudo apt install flatpak-builder

# Fedora
sudo dnf install flatpak-builder
```

**Structure de base** :
```json
{
  "app-id": "com.example.MyApp",
  "runtime": "org.freedesktop.Platform",
  "runtime-version": "22.08",
  "sdk": "org.freedesktop.Sdk",
  "command": "my-app",
  "finish-args": [
    "--socket=wayland",
    "--socket=x11",
    "--filesystem=home"
  ],
  "modules": [
    {
      "name": "my-app",
      "buildsystem": "simple",
      "build-commands": [
        "install -D my-app /app/bin/my-app"
      ],
      "sources": [
        {
          "type": "file",
          "url": "https://example.com/my-app.tar.gz",
          "sha256": "..."
        }
      ]
    }
  ]
}
```

**Build et installation** :
```bash
# Construire
flatpak-builder build-dir com.example.MyApp.json

# Installer localement
flatpak-builder --user --install build-dir com.example.MyApp.json

# Exporter pour distribution
flatpak-builder --repo=repo --force-clean build-dir com.example.MyApp.json
```

## Sécurité et sandboxing

### Sécurité Snap

**Confinement** :
- **Strict** : Isolation maximale (défaut)
- **Classic** : Accès complet au système
- **Devmode** : Mode développement avec warnings

**Interfaces et permissions** :
```bash
# Voir les interfaces disponibles
snap interfaces

# Voir les connexions d'une application
snap connections application

# Permissions communes
# - network : Accès réseau
# - home : Accès au répertoire home
# - x11 : Accès à X11
# - wayland : Accès à Wayland
# - pulseaudio : Accès audio
```

**Audit de sécurité** :
```bash
# Voir les permissions d'un snap
snap info application

# Vérifier les connexions
snap connections application
```

### Sécurité Flatpak

**Sandboxing** :
- Isolation par défaut via Bubblewrap
- Permissions explicites nécessaires
- Pas d'accès système par défaut

**Permissions communes** :
```bash
# Réseau
--socket=network

# Système de fichiers
--filesystem=home
--filesystem=/path/to/dir
--filesystem=host  # Accès complet (dangereux)

# Audio
--socket=pulseaudio

# Affichage
--socket=wayland
--socket=x11

# USB
--device=all
```

**Override de sécurité** :
```bash
# Voir les overrides
flatpak override --show application

# Appliquer des overrides
flatpak override application \
    --socket=wayland \
    --filesystem=home \
    --env=VAR=value
```

## Comparaison et choix

### Tableau comparatif

| Caractéristique | Snap | Flatpak |
|----------------|------|---------|
| **Créateur** | Canonical | Red Hat / GNOME |
| **Store centralisé** | Oui (Snap Store) | Oui (Flathub) |
| **Sandboxing** | Oui | Oui (plus strict) |
| **Taille des packages** | Plus grands | Plus compacts |
| **Démarrage** | Plus lent | Plus rapide |
| **Intégration desktop** | Moyenne | Excellente |
| **Support multi-utilisateur** | Oui | Oui |
| **Mises à jour** | Automatiques | Configurables |
| **Dépendances** | Incluses | Runtimes partagés |

### Quand utiliser Snap

**Avantages** :
- Support officiel Ubuntu
- Store centralisé bien géré
- Bon pour les outils système et serveurs
- Mises à jour automatiques fiables

**Cas d'usage** :
- Applications serveur (Docker, Kubernetes)
- Outils de développement (VS Code, Node.js)
- Applications nécessitant un store centralisé

### Quand utiliser Flatpak

**Avantages** :
- Meilleure intégration desktop
- Sandboxing plus strict
- Runtimes partagés (moins d'espace)
- Indépendant des distributions

**Cas d'usage** :
- Applications desktop graphiques
- Applications nécessitant une isolation stricte
- Environnements multi-distributions

### Utilisation combinée

**Stratégie hybride** :
```bash
# Snap pour outils système
sudo snap install docker
sudo snap install kubectl

# Flatpak pour applications desktop
flatpak install flathub org.gnome.Gedit
flatpak install flathub com.visualstudio.code
```

## Conclusion

Snap et Flatpak représentent l'avenir de la distribution d'applications Linux, offrant des solutions modernes aux problèmes traditionnels de gestion de packages. Chaque technologie a ses forces : Snap excelle pour les outils système et serveurs, tandis que Flatpak brille pour les applications desktop avec une meilleure intégration.

La maîtrise de ces deux systèmes permet de choisir la meilleure solution pour chaque cas d'usage, créant un environnement Linux moderne, sécurisé et facile à maintenir. L'adoption croissante de ces technologies par les développeurs et les distributions confirme leur importance dans l'écosystème Linux moderne.

Dans le prochain chapitre, nous explorerons la gestion des dépôts Git sous Linux, essentielle pour le contrôle de version et la collaboration dans les projets de développement et d'administration système.