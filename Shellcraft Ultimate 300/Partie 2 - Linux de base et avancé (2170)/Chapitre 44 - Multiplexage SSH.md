# Chapitre 44 - Multiplexage SSH

## Table des matières
- [Introduction](#introduction)
- [Principe du multiplexage](#principe-du-multiplexage)
- [Architecture du multiplexage](#architecture-du-multiplexage)
- [Configuration du multiplexage](#configuration-du-multiplexage)
- [Utilisation pratique](#utilisation-pratique)
- [Avantages et optimisation](#avantages-et-optimisation)
- [Multiplexage et scripts](#multiplexage-et-scripts)
- [Gestion des connexions maîtres](#gestion-des-connexions-maîtres)
- [Dépannage et diagnostic](#dépannage-et-diagnostic)
- [Sécurité et bonnes pratiques](#sécurité-et-bonnes-pratiques)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Le multiplexage SSH constitue une optimisation puissante qui transforme l'établissement des connexions SSH. Au lieu d'ouvrir une nouvelle connexion pour chaque session, le multiplexage maintient un tunnel maître et crée des canaux secondaires, offrant des connexions instantanées et réduisant considérablement la latence.

Imaginez le multiplexage SSH comme un train à grande vitesse avec plusieurs wagons : le premier wagon (connexion maître) ouvre la voie, et les wagons suivants peuvent embarquer instantanément sans attendre que les rails soient re-posés à chaque fois. Dans un environnement où vous vous connectez fréquemment au même serveur, le multiplexage SSH peut réduire le temps de connexion de plusieurs secondes à quelques millisecondes.

## Principe du multiplexage

### Concept de base

**Comment ça fonctionne** :
- Une connexion SSH "maître" est établie et maintenue ouverte
- Les connexions suivantes réutilisent cette connexion maître
- Chaque nouvelle session SSH crée un canal dans la connexion existante
- La connexion maître reste active jusqu'à ce qu'elle soit fermée explicitement

**Avantages** :
- Connexions instantanées (pas de handshake SSH)
- Réduction de la charge réseau
- Partage de l'authentification
- Meilleure performance globale

## Architecture du multiplexage

### Composants

**Éléments clés** :
```bash
#!/bin/bash
# Composants du multiplexage SSH

# ControlMaster : Active le multiplexage
# ControlPath : Chemin du socket de contrôle
# ControlPersist : Durée de vie de la connexion maître
# ControlPersist : yes (reste ouvert indéfiniment)
# ControlPersist : 10 (reste ouvert 10 secondes après dernière utilisation)
```

**Flux de connexion** :
```bash
#!/bin/bash
# Flux avec multiplexage

# Première connexion
ssh user@host
# → Établit la connexion maître
# → Crée le socket de contrôle
# → Authentifie l'utilisateur

# Connexions suivantes
ssh user@host
scp file.txt user@host:/dest/
rsync -av source/ user@host:/dest/
# → Réutilisent la connexion maître
# → Connexion instantanée
```

## Configuration du multiplexage

### Configuration de base

**Fichier ~/.ssh/config** :
```bash
#!/bin/bash
# Configuration de base du multiplexage

# Configuration globale
Host *
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 10m

# Configuration pour un serveur spécifique
Host myserver
    HostName server.example.com
    User alice
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 1h
```

**Options de configuration** :
```bash
#!/bin/bash
# Options de configuration

# ControlMaster
# auto : Crée automatiquement une connexion maître si nécessaire
# yes : Force toujours une connexion maître
# no : Désactive le multiplexage
# ask : Demande confirmation avant création

# ControlPath
# Chemin du socket de contrôle
# Variables disponibles :
# %r : Nom d'utilisateur
# %h : Nom d'hôte
# %p : Port
# %l : Nom d'hôte local

# ControlPersist
# Durée de vie de la connexion maître après dernière utilisation
# yes : Reste ouvert indéfiniment
# no : Ferme immédiatement après dernière session
# 10m : Reste ouvert 10 minutes
# 1h : Reste ouvert 1 heure
```

### Configuration avancée

**Configuration optimisée** :
```bash
#!/bin/bash
# Configuration optimisée

Host *
    # Multiplexage
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 10m
    
    # Compression pour connexions lentes
    Compression yes
    
    # Keepalive
    ServerAliveInterval 60
    ServerAliveCountMax 3
    
    # Optimisations
    TCPKeepAlive yes
    ForwardAgent no
```

**Configuration par serveur** :
```bash
#!/bin/bash
# Configuration par serveur

# Serveur de production (connexion longue)
Host production
    HostName prod.example.com
    User deploy
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 2h
    ServerAliveInterval 30

# Serveur de développement (connexion courte)
Host dev
    HostName dev.example.com
    User developer
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 5m

# Serveur avec multiplexage désactivé
Host legacy
    HostName old.example.com
    User admin
    ControlMaster no
```

## Utilisation pratique

### Première connexion

**Établir la connexion maître** :
```bash
#!/bin/bash
# Établir la connexion maître

# Connexion normale (crée automatiquement la connexion maître)
ssh user@host

# Vérifier que la connexion maître est créée
ls -la ~/.ssh/control-*

# Connexion en arrière-plan (pour maintenir la connexion maître)
ssh -f -N user@host
# -f : Arrière-plan
# -N : Ne pas exécuter de commande
```

### Connexions suivantes

**Réutiliser la connexion maître** :
```bash
#!/bin/bash
# Connexions suivantes

# Connexion SSH instantanée
ssh user@host

# SCP instantané
scp file.txt user@host:/dest/

# Rsync instantané
rsync -av source/ user@host:/dest/

# Git via SSH instantané
git push origin main

# Toutes ces commandes réutilisent la connexion maître
```

## Avantages et optimisation

### Performance

**Gains de performance** :
```bash
#!/bin/bash
# Mesure des performances

# Sans multiplexage
time ssh user@host "echo test"
# Temps : ~2-3 secondes (handshake SSH complet)

# Avec multiplexage
time ssh user@host "echo test"
# Temps : ~0.1 seconde (réutilisation de la connexion)

# Gain : 20-30x plus rapide
```

**Réduction de la charge** :
```bash
#!/bin/bash
# Réduction de la charge réseau

# Sans multiplexage : Chaque connexion nécessite
# - Handshake TCP
# - Négociation SSH
# - Authentification
# - Établissement de session

# Avec multiplexage : Seulement la première connexion nécessite tout cela
# Les connexions suivantes réutilisent la connexion existante
```

### Économie de ressources

**Ressources économisées** :
- CPU : Pas de chiffrement/déchiffrement répété
- Réseau : Pas de handshake répété
- Temps : Connexions instantanées
- Authentification : Partage de l'authentification

## Multiplexage et scripts

### Scripts automatisés

**Scripts bénéficiant du multiplexage** :
```bash
#!/bin/bash
# Scripts automatisés avec multiplexage

# Déploiement avec plusieurs connexions SSH
deploy_to_servers() {
    local servers=("server1" "server2" "server3")
    
    for server in "${servers[@]}"; do
        # Première connexion : établit la connexion maître
        ssh "$server" "echo 'Connected to $server'"
        
        # Connexions suivantes : instantanées
        scp app.tar.gz "$server:/tmp/"
        ssh "$server" "tar -xzf /tmp/app.tar.gz -C /opt/app"
        ssh "$server" "systemctl restart app"
    done
}

# Synchronisation avec rsync
sync_multiple_servers() {
    local source="/local/data"
    local servers=("server1" "server2" "server3")
    
    for server in "${servers[@]}"; do
        # Connexion maître établie automatiquement
        rsync -avz "$source/" "$server:/backup/data/"
    done
}
```

### Scripts de monitoring

**Monitoring avec multiplexage** :
```bash
#!/bin/bash
# Monitoring avec multiplexage

monitor_servers() {
    local servers=("server1" "server2" "server3")
    
    while true; do
        for server in "${servers[@]}"; do
            # Connexions instantanées grâce au multiplexage
            local uptime=$(ssh "$server" "uptime")
            local load=$(ssh "$server" "cat /proc/loadavg")
            local disk=$(ssh "$server" "df -h / | tail -1")
            
            echo "[$server]"
            echo "  Uptime: $uptime"
            echo "  Load: $load"
            echo "  Disk: $disk"
            echo ""
        done
        
        sleep 60
    done
}
```

## Gestion des connexions maîtres

### Vérification des connexions

**Commandes de vérification** :
```bash
#!/bin/bash
# Vérifier les connexions maîtres

# Lister les sockets de contrôle
ls -la ~/.ssh/control-*

# Vérifier les connexions actives
ssh -O check user@host

# Statut de toutes les connexions
for socket in ~/.ssh/control-*; do
    if [ -S "$socket" ]; then
        echo "Connexion active : $socket"
        ssh -O check -S "$socket" user@host 2>/dev/null || echo "  Inactive"
    fi
done
```

### Fermeture des connexions

**Fermer les connexions maîtres** :
```bash
#!/bin/bash
# Fermer les connexions maîtres

# Fermer une connexion spécifique
ssh -O exit user@host

# Fermer toutes les connexions maîtres
for socket in ~/.ssh/control-*; do
    if [ -S "$socket" ]; then
        ssh -O exit -S "$socket" user@host 2>/dev/null
    fi
done

# Supprimer les sockets orphelins
find ~/.ssh -name "control-*" -type s -delete
```

### Scripts de gestion

**Gestionnaire de connexions** :
```bash
#!/bin/bash
# Gestionnaire de connexions maîtres

ssh_connection_manager() {
    local action="$1"
    shift
    
    case "$action" in
        list)
            list_master_connections
            ;;
        check)
            check_connection "$@"
            ;;
        exit)
            close_connection "$@"
            ;;
        cleanup)
            cleanup_orphaned_sockets
            ;;
        *)
            echo "Usage: ssh_connection_manager {list|check|exit|cleanup}"
            exit 1
            ;;
    esac
}

list_master_connections() {
    echo "=== Connexions maîtres actives ==="
    for socket in ~/.ssh/control-*; do
        if [ -S "$socket" ]; then
            echo "$socket"
            ssh -O check -S "$socket" user@host 2>/dev/null || echo "  (inactive)"
        fi
    done
}

check_connection() {
    local host="$1"
    ssh -O check "$host"
}

close_connection() {
    local host="$1"
    ssh -O exit "$host"
}

cleanup_orphaned_sockets() {
    echo "Nettoyage des sockets orphelins..."
    find ~/.ssh -name "control-*" -type s -mtime +1 -delete
    echo "Nettoyage terminé"
}
```

## Dépannage et diagnostic

### Problèmes courants

**Dépannage** :
```bash
#!/bin/bash
# Dépannage du multiplexage

# Problème : Socket verrouillé
# Solution : Supprimer le socket et reconnecter
rm ~/.ssh/control-user@host:22
ssh user@host

# Problème : Permission refusée sur le socket
# Solution : Vérifier les permissions du répertoire
chmod 700 ~/.ssh
chmod 600 ~/.ssh/control-*

# Problème : Connexion maître ne se crée pas
# Solution : Vérifier la configuration
ssh -v user@host
# Vérifier les messages sur ControlMaster

# Problème : Connexion maître se ferme trop tôt
# Solution : Augmenter ControlPersist
# ControlPersist 1h au lieu de 10m
```

### Mode debug

**Debug du multiplexage** :
```bash
#!/bin/bash
# Debug du multiplexage

# Mode verbose
ssh -v user@host

# Vérifier la création du socket
ssh -v user@host 2>&1 | grep -i control

# Vérifier la réutilisation
ssh -v user@host 2>&1 | grep -i "reusing\|master"

# Test de connexion
ssh -O check user@host
```

## Sécurité et bonnes pratiques

### Considérations de sécurité

**Sécurité** :
```bash
#!/bin/bash
# Considérations de sécurité

# 1. Permissions du socket
# Le socket doit être accessible seulement par le propriétaire
chmod 600 ~/.ssh/control-*

# 2. Répertoire de contrôle
# Utiliser un répertoire sécurisé
ControlPath ~/.ssh/control-%r@%h:%p

# 3. Durée de vie
# Ne pas laisser les connexions maîtres ouvertes indéfiniment
ControlPersist 10m  # Au lieu de yes

# 4. Authentification
# Le multiplexage partage l'authentification
# S'assurer que l'authentification est sécurisée
```

### Bonnes pratiques

**Recommandations** :
```bash
#!/bin/bash
# Bonnes pratiques

# 1. Utiliser ControlMaster auto
# Crée automatiquement la connexion maître si nécessaire

# 2. Configurer ControlPersist raisonnablement
# 10m à 1h selon l'usage

# 3. Nettoyer régulièrement les sockets orphelins
# Script de nettoyage périodique

# 4. Monitorer les connexions maîtres
# Vérifier régulièrement les connexions actives

# 5. Désactiver pour certains serveurs
# Serveurs sensibles ou temporaires
```

## Scripts d'automatisation

### Configuration automatique

**Script de configuration** :
```bash
#!/bin/bash
# Configuration automatique du multiplexage

setup_ssh_multiplexing() {
    local ssh_config="$HOME/.ssh/config"
    
    # Créer le répertoire de contrôle
    mkdir -p ~/.ssh
    
    # Ajouter la configuration si elle n'existe pas
    if ! grep -q "ControlMaster" "$ssh_config" 2>/dev/null; then
        cat >> "$ssh_config" << 'EOF'

# Multiplexage SSH
Host *
    ControlMaster auto
    ControlPath ~/.ssh/control-%r@%h:%p
    ControlPersist 10m
    ServerAliveInterval 60
    ServerAliveCountMax 3
EOF
        echo "Configuration du multiplexage ajoutée à $ssh_config"
    else
        echo "Le multiplexage est déjà configuré"
    fi
}

# Utilisation
# setup_ssh_multiplexing
```

### Script de monitoring

**Monitoring des connexions** :
```bash
#!/bin/bash
# Monitoring des connexions maîtres

monitor_master_connections() {
    while true; do
        clear
        echo "=== Connexions maîtres SSH ==="
        echo ""
        
        local count=0
        for socket in ~/.ssh/control-*; do
            if [ -S "$socket" ]; then
                ((count++))
                echo "[$count] $socket"
                ssh -O check -S "$socket" user@host 2>/dev/null && echo "  Status: Active" || echo "  Status: Inactive"
            fi
        done
        
        if [ $count -eq 0 ]; then
            echo "Aucune connexion maître active"
        fi
        
        sleep 5
    done
}
```

## Conclusion

Le multiplexage SSH est une optimisation essentielle qui transforme l'expérience d'utilisation de SSH. En réutilisant les connexions existantes, il offre des connexions instantanées, réduit la charge réseau, et améliore significativement les performances des scripts automatisés.

Un système bien configuré utilise le multiplexage SSH de manière appropriée : configuration automatique pour la plupart des serveurs, durée de vie raisonnable des connexions maîtres, et nettoyage régulier des sockets orphelins. La compréhension approfondie de ce mécanisme permet d'optimiser les workflows d'administration et de développement qui dépendent fortement de SSH.

Dans le chapitre suivant, nous explorerons la sécurisation avancée des connexions SSH, découvrant les techniques pour renforcer l'authentification, le chiffrement, et la protection contre les attaques.
