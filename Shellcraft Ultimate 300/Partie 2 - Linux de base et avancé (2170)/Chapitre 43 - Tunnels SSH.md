# Chapitre 43 - Tunnels SSH

## Table des matières
- [Introduction](#introduction)
- [Principes du tunneling SSH](#principes-du-tunneling-ssh)
- [Architecture des tunnels](#architecture-des-tunnels)
- [Tunnels locaux (Local Port Forwarding)](#tunnels-locaux-local-port-forwarding)
- [Tunnels distants (Remote Port Forwarding)](#tunnels-distants-remote-port-forwarding)
- [Tunnels dynamiques (SOCKS proxy)](#tunnels-dynamiques-socks-proxy)
- [Tunnels inversés et reverse SSH](#tunnels-inversés-et-reverse-ssh)
- [Configuration avancée](#configuration-avancée)
- [Applications pratiques](#applications-pratiques)
- [Sécurité et risques](#sécurité-et-risques)
- [Dépannage des tunnels](#dépannage-des-tunnels)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Les tunnels SSH constituent l'une des fonctionnalités les plus puissantes et méconnues de SSH. Au-delà du simple accès distant, ils permettent de sécuriser des connexions non chiffrées, d'accéder à des services locaux à distance, et de contourner des restrictions réseau.

Imaginez les tunnels SSH comme des passages secrets dans un château fort : ils permettent d'accéder à des zones normalement inaccessibles, de transporter des marchandises (données) en sécurité, et d'échapper aux gardes (pare-feu) qui surveillent les portes principales. Dans un environnement réseau moderne, les tunnels SSH sont essentiels pour sécuriser les communications, accéder aux ressources internes, et créer des VPN légers.

## Principes du tunneling SSH

### Concept de tunneling

**Qu'est-ce qu'un tunnel SSH ?** :
- Un tunnel SSH crée une connexion chiffrée entre deux points
- Les données transitent de manière sécurisée à travers le tunnel
- Le tunnel peut rediriger des ports locaux vers des ports distants (ou vice versa)
- Le tunnel peut agir comme un proxy SOCKS dynamique

**Avantages** :
- Chiffrement de bout en bout
- Contournement de pare-feu
- Accès sécurisé aux services internes
- Pas besoin de configuration VPN complexe

## Architecture des tunnels

### Types de tunnels

**Trois types principaux** :
```bash
#!/bin/bash
# Types de tunnels SSH

# 1. Local Port Forwarding (-L)
# Redirige un port local vers un port distant
# Utilisé pour accéder à des services distants via SSH

# 2. Remote Port Forwarding (-R)
# Redirige un port distant vers un port local
# Utilisé pour exposer des services locaux à distance

# 3. Dynamic Port Forwarding (-D)
# Crée un proxy SOCKS dynamique
# Utilisé pour router tout le trafic via SSH
```

**Flux des données** :
```bash
#!/bin/bash
# Flux des données dans un tunnel

# Tunnel local (-L)
# Client local → Port local → Tunnel SSH → Serveur SSH → Port distant

# Tunnel distant (-R)
# Serveur SSH → Port distant → Tunnel SSH → Client local → Port local

# Tunnel dynamique (-D)
# Application → Proxy SOCKS local → Tunnel SSH → Serveur SSH → Internet
```

## Tunnels locaux (Local Port Forwarding)

### Syntaxe de base

**Syntaxe** :
```bash
ssh -L [bind_address:]local_port:destination_host:destination_port user@ssh_server
```

**Exemples de base** :
```bash
#!/bin/bash
# Tunnels locaux de base

# Accéder à un serveur web distant via SSH
ssh -L 8080:localhost:80 user@server.example.com
# http://localhost:8080 → http://server.example.com:80

# Accéder à une base de données distante
ssh -L 3306:localhost:3306 user@db-server.example.com
# mysql -h 127.0.0.1 -P 3306

# Accéder à un service sur un autre serveur via le serveur SSH
ssh -L 8080:internal-server:80 user@bastion.example.com
# http://localhost:8080 → http://internal-server:80 (via bastion)

# Avec bind address spécifique
ssh -L 127.0.0.1:8080:localhost:80 user@server.example.com
# Seulement accessible depuis localhost

# Accessible depuis toutes les interfaces
ssh -L 0.0.0.0:8080:localhost:80 user@server.example.com
```

### Options avancées

**Options utiles** :
```bash
#!/bin/bash
# Options pour tunnels locaux

# -N : Ne pas exécuter de commande distante (tunnel seulement)
ssh -N -L 8080:localhost:80 user@server.example.com

# -f : Passer en arrière-plan
ssh -f -N -L 8080:localhost:80 user@server.example.com

# -g : Permettre l'accès depuis d'autres machines (GatewayPorts)
ssh -g -L 8080:localhost:80 user@server.example.com

# Combinaison
ssh -f -N -L 8080:localhost:80 user@server.example.com

# Avec clé spécifique
ssh -i ~/.ssh/id_rsa -L 8080:localhost:80 user@server.example.com

# Avec port SSH non standard
ssh -p 2222 -L 8080:localhost:80 user@server.example.com
```

### Cas d'usage

**Accès à des services internes** :
```bash
#!/bin/bash
# Cas d'usage : Accès à des services internes

# Accéder à un serveur web interne
ssh -L 8080:web-internal:80 user@bastion.example.com

# Accéder à une base de données MySQL
ssh -L 3306:db-internal:3306 user@bastion.example.com
mysql -h 127.0.0.1 -P 3306 -u user -p

# Accéder à PostgreSQL
ssh -L 5432:db-internal:5432 user@bastion.example.com
psql -h 127.0.0.1 -p 5432 -U user -d database

# Accéder à Redis
ssh -L 6379:redis-internal:6379 user@bastion.example.com
redis-cli -h 127.0.0.1 -p 6379

# Accéder à un service RDP (Windows)
ssh -L 3389:windows-server:3389 user@bastion.example.com
# Puis se connecter avec mstsc à localhost:3389
```

## Tunnels distants (Remote Port Forwarding)

### Syntaxe de base

**Syntaxe** :
```bash
ssh -R [bind_address:]remote_port:local_host:local_port user@ssh_server
```

**Exemples de base** :
```bash
#!/bin/bash
# Tunnels distants de base

# Exposer un service web local sur le serveur distant
ssh -R 8080:localhost:3000 user@server.example.com
# Sur server.example.com, http://localhost:8080 accède au service local sur port 3000

# Exposer une base de données locale
ssh -R 3306:localhost:3306 user@server.example.com

# Exposer un service accessible depuis d'autres machines
ssh -R 0.0.0.0:8080:localhost:3000 user@server.example.com
# Accessible depuis n'importe quelle machine sur le réseau du serveur

# Avec options
ssh -f -N -R 8080:localhost:3000 user@server.example.com
```

### Configuration serveur

**Configuration sshd pour remote forwarding** :
```bash
#!/bin/bash
# Configuration serveur pour remote forwarding

# /etc/ssh/sshd_config

# Autoriser le port forwarding
AllowTcpForwarding yes

# Permettre binding sur toutes les interfaces (pour -R 0.0.0.0:port)
GatewayPorts yes

# Ou seulement pour certains utilisateurs
GatewayPorts no
Match User tunneluser
    GatewayPorts yes

# Recharger la configuration
sudo systemctl reload sshd
```

### Cas d'usage

**Exposer des services locaux** :
```bash
#!/bin/bash
# Cas d'usage : Exposer des services locaux

# Développement local accessible à distance
ssh -R 8080:localhost:3000 user@server.example.com
# Les collègues peuvent accéder à votre app de développement via server.example.com:8080

# Service de monitoring local
ssh -R 9090:localhost:9090 user@server.example.com
# Prometheus/Grafana local accessible à distance

# Service de test
ssh -R 8080:localhost:8080 user@server.example.com
# Service de test accessible depuis l'extérieur
```

## Tunnels dynamiques (SOCKS proxy)

### Proxy SOCKS

**Syntaxe** :
```bash
ssh -D [bind_address:]local_port user@ssh_server
```

**Exemples** :
```bash
#!/bin/bash
# Tunnels dynamiques SOCKS

# Créer un proxy SOCKS sur le port 1080
ssh -D 1080 user@server.example.com

# Proxy SOCKS accessible seulement depuis localhost
ssh -D 127.0.0.1:1080 user@server.example.com

# Proxy en arrière-plan
ssh -f -N -D 1080 user@server.example.com

# Proxy accessible depuis d'autres machines
ssh -g -D 1080 user@server.example.com
```

### Utilisation avec applications

**Configuration navigateur** :
```bash
#!/bin/bash
# Configuration navigateur pour proxy SOCKS

# Firefox :
# Préférences → Réseau → Paramètres
# Proxy SOCKS v5 : localhost, Port : 1080

# Chrome/Chromium :
# --proxy-server="socks5://localhost:1080"
google-chrome --proxy-server="socks5://localhost:1080"

# Configuration système (Linux)
export http_proxy=socks5://localhost:1080
export https_proxy=socks5://localhost:1080
export all_proxy=socks5://localhost:1080
```

**Utilisation avec outils en ligne de commande** :
```bash
#!/bin/bash
# Utilisation avec outils CLI

# curl via proxy SOCKS
curl --socks5-hostname localhost:1080 http://example.com
curl --socks5 localhost:1080 http://example.com

# wget via proxy SOCKS
export http_proxy=socks5://localhost:1080
wget http://example.com

# git via proxy SOCKS
git config --global http.proxy socks5://localhost:1080
git config --global https.proxy socks5://localhost:1080

# ssh via proxy SOCKS (ProxyCommand)
ssh -o ProxyCommand="nc -X 5 -x localhost:1080 %h %p" user@destination
```

## Tunnels inversés et reverse SSH

### Reverse SSH

**Connexion inversée** :
```bash
#!/bin/bash
# Reverse SSH : Machine derrière NAT se connecte au serveur

# Sur la machine derrière NAT (client)
ssh -R 2222:localhost:22 user@public-server.example.com

# Sur le serveur public, on peut maintenant se connecter à la machine derrière NAT
ssh -p 2222 user@localhost

# Avec keepalive pour maintenir la connexion
ssh -R 2222:localhost:22 -o ServerAliveInterval=60 user@public-server.example.com
```

**Script de connexion automatique** :
```bash
#!/bin/bash
# Script de reverse SSH automatique

reverse_ssh_connection() {
    local public_server="$1"
    local user="$2"
    local local_port="${3:-22}"
    local remote_port="${4:-2222}"
    
    while true; do
        ssh -R ${remote_port}:localhost:${local_port} \
            -o ServerAliveInterval=60 \
            -o ServerAliveCountMax=3 \
            -o ExitOnForwardFailure=yes \
            -N -f "$user@$public_server"
        
        sleep 60
    done
}

# Utilisation
# reverse_ssh_connection public-server.example.com user 22 2222
```

## Configuration avancée

### Configuration SSH pour tunnels

**Fichier ~/.ssh/config** :
```bash
#!/bin/bash
# Configuration SSH pour tunnels

# Tunnel local dans config
Host web-tunnel
    HostName server.example.com
    User user
    LocalForward 8080 localhost:80
    LocalForward 3306 localhost:3306

# Utilisation : ssh web-tunnel

# Tunnel distant dans config
Host expose-local
    HostName server.example.com
    User user
    RemoteForward 8080 localhost:3000

# Tunnel dynamique dans config
Host socks-proxy
    HostName server.example.com
    User user
    DynamicForward 1080

# Tunnel avec plusieurs forwards
Host multi-tunnel
    HostName server.example.com
    User user
    LocalForward 8080 localhost:80
    LocalForward 3306 localhost:3306
    LocalForward 5432 localhost:5432
    DynamicForward 1080
```

### Tunnels persistants

**Maintenir les tunnels actifs** :
```bash
#!/bin/bash
# Maintenir les tunnels actifs

# Avec autossh (installation requise)
# sudo apt install autossh

autossh -M 20000 -f -N -L 8080:localhost:80 user@server.example.com

# Avec systemd service
create_tunnel_service() {
    cat > /tmp/ssh-tunnel.service << 'EOF'
[Unit]
Description=SSH Tunnel
After=network.target

[Service]
Type=simple
User=user
ExecStart=/usr/bin/ssh -N -L 8080:localhost:80 user@server.example.com
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
    
    sudo cp /tmp/ssh-tunnel.service /etc/systemd/system/
    sudo systemctl enable ssh-tunnel.service
    sudo systemctl start ssh-tunnel.service
}
```

## Applications pratiques

### Accès sécurisé à des bases de données

**Tunnels pour bases de données** :
```bash
#!/bin/bash
# Accès sécurisé aux bases de données

# MySQL
ssh -L 3306:db-server:3306 user@bastion.example.com &
mysql -h 127.0.0.1 -P 3306 -u user -p

# PostgreSQL
ssh -L 5432:db-server:5432 user@bastion.example.com &
psql -h 127.0.0.1 -p 5432 -U user -d database

# MongoDB
ssh -L 27017:db-server:27017 user@bastion.example.com &
mongo --host 127.0.0.1 --port 27017

# Redis
ssh -L 6379:redis-server:6379 user@bastion.example.com &
redis-cli -h 127.0.0.1 -p 6379
```

### Contournement de pare-feu

**Accès web via tunnel** :
```bash
#!/bin/bash
# Contourner les restrictions web

# Proxy SOCKS pour navigation
ssh -D 1080 user@server.example.com

# Configuration navigateur : SOCKS5 localhost:1080
# Toutes les requêtes web passent par le tunnel SSH
```

### VPN léger

**Créer un VPN simple** :
```bash
#!/bin/bash
# VPN simple avec SSH

# Sur le client
ssh -w 0:0 user@server.example.com

# Sur le serveur (après connexion)
# sudo ip link set tun0 up
# sudo ip addr add 10.0.0.1/24 dev tun0

# Sur le client (après connexion)
# sudo ip link set tun0 up
# sudo ip addr add 10.0.0.2/24 dev tun0
# sudo ip route add 10.0.0.0/24 dev tun0
```

## Sécurité et risques

### Risques de sécurité

**Risques potentiels** :
```bash
#!/bin/bash
# Risques de sécurité avec les tunnels SSH

# 1. Exposition de services non sécurisés
# Ne pas exposer des services non chiffrés sans protection supplémentaire

# 2. Bypass de pare-feu
# Les tunnels peuvent contourner les politiques de sécurité

# 3. Accès non autorisé
# Limiter qui peut créer des tunnels

# 4. Ressources système
# Les tunnels consomment des ressources
```

### Bonnes pratiques

**Recommandations de sécurité** :
```bash
#!/bin/bash
# Bonnes pratiques de sécurité

# 1. Limiter les utilisateurs autorisés
# /etc/ssh/sshd_config
AllowTcpForwarding yes
Match User tunneluser
    AllowTcpForwarding yes
Match all
    AllowTcpForwarding no

# 2. Limiter les ports autorisés
# Utiliser des ports spécifiques au lieu de permettre tous les ports

# 3. Monitoring
# Surveiller les tunnels actifs
sudo netstat -tlnp | grep ssh
sudo ss -tlnp | grep ssh

# 4. Authentification forte
# Utiliser des clés SSH au lieu de mots de passe

# 5. Chiffrement fort
# Utiliser des algorithmes modernes
```

### Configuration sécurisée

**Configuration serveur sécurisée** :
```bash
#!/bin/bash
# Configuration serveur sécurisée

# /etc/ssh/sshd_config

# Autoriser le port forwarding seulement pour certains utilisateurs
AllowTcpForwarding no
Match User tunneluser
    AllowTcpForwarding yes
    GatewayPorts no  # Pas de binding sur toutes les interfaces

# Limiter les tunnels distants
PermitOpen localhost:8080  # Seulement ce port autorisé

# Désactiver X11 forwarding si non nécessaire
X11Forwarding no

# Désactiver Agent forwarding si non nécessaire
AllowAgentForwarding no
```

## Dépannage des tunnels

### Vérification des tunnels

**Commandes de vérification** :
```bash
#!/bin/bash
# Vérification des tunnels

# Lister les tunnels actifs
sudo netstat -tlnp | grep ssh
sudo ss -tlnp | grep ssh

# Vérifier les processus SSH
ps aux | grep ssh

# Vérifier les connexions SSH
sudo lsof -i -P -n | grep ssh

# Tester la connectivité du tunnel
telnet localhost 8080
nc -zv localhost 8080
```

### Dépannage courant

**Problèmes courants** :
```bash
#!/bin/bash
# Dépannage des problèmes courants

# Problème : Port déjà utilisé
# Solution : Utiliser un autre port ou libérer le port
sudo lsof -i :8080
sudo kill -9 <PID>

# Problème : Permission refusée
# Solution : Vérifier les permissions SSH et la configuration serveur
ssh -v -L 8080:localhost:80 user@server.example.com

# Problème : Connexion fermée
# Solution : Vérifier que le service distant est accessible depuis le serveur SSH
ssh user@server.example.com "nc -zv localhost 80"

# Problème : Tunnel ne fonctionne pas
# Solution : Vérifier la configuration et les logs
sudo tail -f /var/log/auth.log
ssh -vvv -L 8080:localhost:80 user@server.example.com
```

### Scripts de diagnostic

**Scripts de diagnostic** :
```bash
#!/bin/bash
# Scripts de diagnostic

check_tunnels() {
    echo "=== Tunnels SSH actifs ==="
    sudo ss -tlnp | grep ssh | awk '{print $4, $5}'
    
    echo ""
    echo "=== Processus SSH ==="
    ps aux | grep "ssh.*-L\|ssh.*-R\|ssh.*-D" | grep -v grep
    
    echo ""
    echo "=== Connexions SSH ==="
    sudo netstat -tn | grep ESTABLISHED | grep :22
}

test_tunnel() {
    local port="$1"
    local destination="${2:-localhost}"
    
    echo "Test du tunnel sur le port $port..."
    if nc -zv "$destination" "$port" 2>&1; then
        echo "✓ Tunnel fonctionnel"
    else
        echo "✗ Tunnel non fonctionnel"
    fi
}
```

## Scripts d'automatisation

### Gestionnaire de tunnels

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de tunnels SSH

set -euo pipefail

tunnel_manager() {
    local action="$1"
    shift
    
    case "$action" in
        local)
            create_local_tunnel "$@"
            ;;
        remote)
            create_remote_tunnel "$@"
            ;;
        socks)
            create_socks_tunnel "$@"
            ;;
        list)
            list_tunnels
            ;;
        kill)
            kill_tunnel "$@"
            ;;
        *)
            echo "Usage: tunnel_manager {local|remote|socks|list|kill}"
            exit 1
            ;;
    esac
}

create_local_tunnel() {
    local local_port="$1"
    local remote_host="$2"
    local remote_port="$3"
    local ssh_server="$4"
    local ssh_user="${5:-$USER}"
    
    ssh -f -N -L ${local_port}:${remote_host}:${remote_port} \
        "${ssh_user}@${ssh_server}"
    
    echo "Tunnel local créé : localhost:${local_port} -> ${remote_host}:${remote_port}"
}

create_remote_tunnel() {
    local remote_port="$1"
    local local_host="$2"
    local local_port="$3"
    local ssh_server="$4"
    local ssh_user="${5:-$USER}"
    
    ssh -f -N -R ${remote_port}:${local_host}:${local_port} \
        "${ssh_user}@${ssh_server}"
    
    echo "Tunnel distant créé : ${ssh_server}:${remote_port} -> ${local_host}:${local_port}"
}

create_socks_tunnel() {
    local local_port="${1:-1080}"
    local ssh_server="$2"
    local ssh_user="${3:-$USER}"
    
    ssh -f -N -D ${local_port} "${ssh_user}@${ssh_server}"
    
    echo "Proxy SOCKS créé sur localhost:${local_port}"
}

list_tunnels() {
    echo "=== Tunnels SSH actifs ==="
    ps aux | grep "ssh.*-L\|ssh.*-R\|ssh.*-D" | grep -v grep | \
    awk '{print $2, $11, $12, $13, $14, $15, $16, $17}'
}

kill_tunnel() {
    local pid="$1"
    kill "$pid"
    echo "Tunnel $pid arrêté"
}

# Utilisation
# tunnel_manager local 8080 localhost 80 server.example.com
# tunnel_manager remote 8080 localhost 3000 server.example.com
# tunnel_manager socks 1080 server.example.com
# tunnel_manager list
# tunnel_manager kill <PID>
```

## Conclusion

Les tunnels SSH sont des outils puissants qui permettent de sécuriser les communications, d'accéder aux ressources internes, et de créer des solutions de réseau flexibles. En maîtrisant les tunnels locaux, distants, et dynamiques, vous pouvez créer des architectures réseau sécurisées sans avoir besoin de configurations VPN complexes.

Un système bien administré utilise les tunnels SSH de manière appropriée : tunnels locaux pour accéder aux services distants, tunnels distants pour exposer les services locaux, et proxies SOCKS pour router le trafic de manière sécurisée. La compréhension approfondie de ces mécanismes permet de créer des solutions réseau robustes et sécurisées.

Dans le chapitre suivant, nous explorerons le multiplexage SSH, découvrant comment optimiser les connexions SSH multiples et réduire la latence grâce au contrôle de connexion maître.
