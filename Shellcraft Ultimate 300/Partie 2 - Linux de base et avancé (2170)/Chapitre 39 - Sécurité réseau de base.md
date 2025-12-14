# Chapitre 39 - Sécurité réseau de base

## Table des matières
- [Introduction](#introduction)
- [Principes de sécurité réseau](#principes-de-sécurité-réseau)
- [Architecture de sécurité en couches](#architecture-de-sécurité-en-couches)
- [Configuration des pare-feu](#configuration-des-pare-feu)
- [SSH et sécurisation des connexions](#ssh-et-sécurisation-des-connexions)
- [Fail2ban : protection contre les attaques](#fail2ban--protection-contre-les-attaques)
- [VPN et tunnels sécurisés](#vpn-et-tunnels-sécurisés)
- [Détection d'intrusion](#détection-dintrusion)
- [Chiffrement des communications](#chiffrement-des-communications)
- [Sécurisation des services réseau](#sécurisation-des-services-réseau)
- [Audit et conformité](#audit-et-conformité)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La sécurité réseau constitue la première ligne de défense contre les menaces extérieures. Au-delà des simples mots de passe et des pare-feu, il s'agit de concevoir une architecture réseau qui protège les actifs tout en permettant les communications nécessaires.

Imaginez la sécurité réseau comme les fortifications d'un château médiéval : des douves, des remparts, des portes gardées, et des tours de guet, le tout conçu pour repousser les assaillants tout en permettant aux alliés d'entrer et de sortir librement. Dans un système Linux moderne, cette sécurité s'étend de la configuration des pare-feu aux tunnels chiffrés, en passant par la détection d'intrusion et l'audit continu.

## Principes de sécurité réseau

### Principes fondamentaux

**Principes essentiels** :
```bash
#!/bin/bash
# Principes de sécurité réseau

# 1. Principe du moindre privilège
# - Ouvrir seulement les ports nécessaires
# - Accorder seulement les permissions nécessaires
# - Utiliser des comptes avec privilèges minimaux

# 2. Défense en profondeur
# - Plusieurs couches de sécurité
# - Pare-feu + IDS + Chiffrement + Monitoring

# 3. Séparation des réseaux
# - Réseaux internes séparés
# - DMZ pour les services publics
# - Isolation des segments critiques

# 4. Chiffrement par défaut
# - Utiliser TLS/SSL pour toutes les communications
# - Chiffrer les données sensibles au repos

# 5. Monitoring et audit
# - Surveiller les connexions
# - Logger toutes les activités
# - Détecter les anomalies
```

**Modèle de sécurité** :
```bash
#!/bin/bash
# Modèle de sécurité en couches

# Couche 1 : Pare-feu réseau
# - Filtrage des paquets
# - Blocage des ports non utilisés

# Couche 2 : Pare-feu applicatif
# - Filtrage au niveau application
# - Protection contre les attaques spécifiques

# Couche 3 : Authentification forte
# - Clés SSH
# - Authentification multi-facteurs

# Couche 4 : Chiffrement
# - TLS/SSL
# - VPN

# Couche 5 : Monitoring
# - Détection d'intrusion
# - Logging et audit
```

## Architecture de sécurité en couches

### Segmentation réseau

**Architecture sécurisée** :
```bash
#!/bin/bash
# Architecture réseau sécurisée

# Réseau externe (Internet)
#   ↓
# Pare-feu périmétrique
#   ↓
# DMZ (Zone démilitarisée)
#   - Serveurs web publics
#   - Serveurs DNS publics
#   ↓
# Pare-feu interne
#   ↓
# Réseau interne
#   - Serveurs applicatifs
#   - Bases de données
#   - Réseau de gestion
```

**Configuration de segmentation** :
```bash
#!/bin/bash
# Configuration de segmentation réseau

# Interfaces réseau
# eth0 : Réseau externe
# eth1 : DMZ
# eth2 : Réseau interne

# Configuration des routes
configure_segmented_network() {
    # Interface externe
    sudo ip addr add 203.0.113.100/24 dev eth0
    sudo ip link set eth0 up
    
    # Interface DMZ
    sudo ip addr add 192.168.1.100/24 dev eth1
    sudo ip link set eth1 up
    
    # Interface interne
    sudo ip addr add 10.0.0.100/24 dev eth2
    sudo ip link set eth2 up
    
    # Routes
    sudo ip route add default via 203.0.113.1 dev eth0
    sudo ip route add 192.168.1.0/24 dev eth1
    sudo ip route add 10.0.0.0/24 dev eth2
}
```

## Configuration des pare-feu

### UFW : Uncomplicated Firewall

**Utilisation de UFW** :
```bash
#!/bin/bash
# Utilisation de UFW

# Activer UFW
sudo ufw enable

# Statut
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered

# Autoriser des ports
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 22

# Autoriser par service
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Autoriser depuis une IP spécifique
sudo ufw allow from 192.168.1.100
sudo ufw allow from 192.168.1.0/24

# Autoriser un port depuis une IP
sudo ufw allow from 192.168.1.100 to any port 22

# Refuser des connexions
sudo ufw deny 23/tcp
sudo ufw deny telnet

# Supprimer une règle
sudo ufw delete allow 22/tcp
sudo ufw delete 1  # Par numéro

# Réinitialiser
sudo ufw reset

# Désactiver
sudo ufw disable
```

**Configuration UFW avancée** :
```bash
#!/bin/bash
# Configuration UFW avancée

# Fichier de configuration : /etc/ufw/before.rules
# Fichier personnalisé : /etc/ufw/before6.rules

# Règles personnalisées
configure_ufw_advanced() {
    # Limiter les connexions SSH
    sudo ufw limit ssh/tcp
    
    # Autoriser avec logging
    sudo ufw allow 80/tcp comment 'HTTP'
    sudo ufw allow 443/tcp comment 'HTTPS'
    
    # Règles par défaut
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    sudo ufw default deny forward
}

# Script de configuration sécurisée
secure_ufw_setup() {
    # Réinitialiser
    sudo ufw --force reset
    
    # Règles par défaut
    sudo ufw default deny incoming
    sudo ufw default allow outgoing
    
    # SSH (limité)
    sudo ufw limit ssh/tcp
    
    # Services web
    sudo ufw allow 80/tcp
    sudo ufw allow 443/tcp
    
    # Activer
    sudo ufw --force enable
    
    echo "UFW configuré et activé"
}
```

### firewalld : Firewall dynamique

**Utilisation de firewalld** :
```bash
#!/bin/bash
# Utilisation de firewalld

# Statut
sudo firewall-cmd --state
sudo firewall-cmd --list-all

# Zones
sudo firewall-cmd --get-zones
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones

# Changer de zone
sudo firewall-cmd --set-default-zone=public

# Ajouter des ports
sudo firewall-cmd --add-port=80/tcp
sudo firewall-cmd --add-port=443/tcp
sudo firewall-cmd --add-port=8080-8090/tcp

# Ajouter des services
sudo firewall-cmd --add-service=http
sudo firewall-cmd --add-service=https
sudo firewall-cmd --add-service=ssh

# Rendre permanent
sudo firewall-cmd --add-port=80/tcp --permanent
sudo firewall-cmd --reload

# Supprimer
sudo firewall-cmd --remove-port=80/tcp
sudo firewall-cmd --remove-service=http

# Règles riches (Rich Rules)
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept'
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="10.0.0.0/8" port port="3306" protocol="tcp" accept'
```

## SSH et sécurisation des connexions

### Configuration SSH sécurisée

**Configuration de base** :
```bash
#!/bin/bash
# Configuration SSH sécurisée

# Fichier : /etc/ssh/sshd_config

secure_ssh_config() {
    sudo tee -a /etc/ssh/sshd_config << 'EOF'
# Port non standard (défense en profondeur)
Port 2222

# Désactiver root login
PermitRootLogin no

# Désactiver authentification par mot de passe
PasswordAuthentication no
PubkeyAuthentication yes

# Limiter les tentatives
MaxAuthTries 3
LoginGraceTime 60

# Désactiver fonctionnalités inutiles
X11Forwarding no
AllowTcpForwarding no
PermitTunnel no

# Utilisateurs autorisés
AllowUsers alice bob
# AllowGroups admins

# Timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# Logging
LogLevel VERBOSE
SyslogFacility AUTH

# Algorithmes modernes
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group16-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com
EOF
    
    # Vérifier la configuration
    sudo sshd -t
    
    # Recharger
    sudo systemctl reload sshd
    
    echo "SSH sécurisé configuré"
}
```

**Génération et gestion des clés SSH** :
```bash
#!/bin/bash
# Gestion des clés SSH

# Générer une clé ED25519 (recommandé)
ssh-keygen -t ed25519 -a 100 -C "user@hostname"

# Générer une clé RSA (alternative)
ssh-keygen -t rsa -b 4096 -a 100

# Copier la clé publique
ssh-copy-id user@remote-server

# Vérifier les clés
ssh-keygen -l -f ~/.ssh/id_ed25519.pub

# Permissions correctes
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub
chmod 600 ~/.ssh/authorized_keys
```

## Fail2ban : protection contre les attaques

### Installation et configuration

**Installation Fail2ban** :
```bash
#!/bin/bash
# Installation Fail2ban

# Debian/Ubuntu
sudo apt update
sudo apt install fail2ban

# CentOS/RHEL
sudo yum install epel-release
sudo yum install fail2ban

# Activer le service
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

**Configuration Fail2ban** :
```bash
#!/bin/bash
# Configuration Fail2ban

# Fichier principal : /etc/fail2ban/jail.conf
# Fichier personnalisé : /etc/fail2ban/jail.local

configure_fail2ban() {
    sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
# Adresse IP à ignorer (whitelist)
ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24

# Temps de bannissement
bantime = 3600

# Fenêtre de temps pour les tentatives
findtime = 600

# Nombre de tentatives avant bannissement
maxretry = 3

# Actions
action = %(action_)s
EOF
    
    # Configuration SSH
    sudo tee -a /etc/fail2ban/jail.local << 'EOF'
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
EOF
    
    # Recharger
    sudo systemctl restart fail2ban
    
    echo "Fail2ban configuré"
}
```

**Gestion Fail2ban** :
```bash
#!/bin/bash
# Gestion Fail2ban

# Statut
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Bannir une IP manuellement
sudo fail2ban-client set sshd banip 192.168.1.100

# Débannir une IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# Lister les IPs bannies
sudo fail2ban-client get sshd banned

# Recharger la configuration
sudo fail2ban-client reload

# Redémarrer
sudo systemctl restart fail2ban
```

## VPN et tunnels sécurisés

### OpenVPN

**Installation OpenVPN** :
```bash
#!/bin/bash
# Installation OpenVPN

# Debian/Ubuntu
sudo apt install openvpn easy-rsa

# CentOS/RHEL
sudo yum install openvpn easy-rsa
```

**Configuration OpenVPN** :
```bash
#!/bin/bash
# Configuration OpenVPN

# Créer l'infrastructure PKI
setup_openvpn_ca() {
    cd /etc/openvpn
    sudo make-cadir ~/openvpn-ca
    cd ~/openvpn-ca
    
    # Générer le CA
    ./easyrsa init-pki
    ./easyrsa build-ca
    
    # Générer le certificat serveur
    ./easyrsa gen-req server nopass
    ./easyrsa sign-req server server
    
    # Générer la clé Diffie-Hellman
    ./easyrsa gen-dh
    
    # Générer la clé TLS
    openvpn --genkey --secret ta.key
}
```

### WireGuard

**Installation WireGuard** :
```bash
#!/bin/bash
# Installation WireGuard

# Debian/Ubuntu
sudo apt install wireguard

# CentOS/RHEL
sudo yum install epel-release
sudo yum install wireguard-tools

# Charger le module kernel
sudo modprobe wireguard
```

**Configuration WireGuard** :
```bash
#!/bin/bash
# Configuration WireGuard

# Générer les clés
generate_wireguard_keys() {
    # Clé privée
    wg genkey | tee privatekey | wg pubkey > publickey
    
    # Clé pré-partagée (optionnel)
    wg genpsk > presharedkey
}

# Configuration serveur
configure_wireguard_server() {
    sudo tee /etc/wireguard/wg0.conf << 'EOF'
[Interface]
PrivateKey = <SERVER_PRIVATE_KEY>
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
EOF
    
    # Activer
    sudo wg-quick up wg0
    sudo systemctl enable wg-quick@wg0
}
```

## Détection d'intrusion

### AIDE : Advanced Intrusion Detection Environment

**Installation AIDE** :
```bash
#!/bin/bash
# Installation AIDE

# Debian/Ubuntu
sudo apt install aide

# CentOS/RHEL
sudo yum install aide
```

**Configuration AIDE** :
```bash
#!/bin/bash
# Configuration AIDE

# Initialiser la base de données
sudo aideinit

# Vérifier l'intégrité
sudo aide --check

# Mettre à jour la base de données
sudo aide --update
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

### rkhunter : Rootkit Hunter

**Installation rkhunter** :
```bash
#!/bin/bash
# Installation rkhunter

# Debian/Ubuntu
sudo apt install rkhunter

# CentOS/RHEL
sudo yum install rkhunter
```

**Utilisation rkhunter** :
```bash
#!/bin/bash
# Utilisation rkhunter

# Mettre à jour la base de données
sudo rkhunter --update

# Vérification complète
sudo rkhunter --check

# Vérification rapide
sudo rkhunter --check --skip-keyfiles --skip-group --skip-passwd --skip-logfiles

# Propager les propriétés
sudo rkhunter --propupd
```

### Scripts de détection personnalisés

**Script de détection d'intrusion** :
```bash
#!/bin/bash
# Script de détection d'intrusion

intrusion_detection() {
    local log_file="/var/log/intrusion.log"
    
    # Détecter les connexions SSH suspectes
    detect_suspicious_ssh() {
        sudo journalctl -u ssh | grep "Failed password" | \
        awk '{print $11}' | sort | uniq -c | sort -rn | \
        awk '$1 > 5 {print "ALERT: Multiple SSH failures from " $2}'
    }
    
    # Détecter les ports ouverts suspects
    detect_suspicious_ports() {
        ss -tlnp | awk 'NR>1 {print $4}' | cut -d':' -f2 | \
        sort -n | uniq | \
        awk '$1 > 1024 && $1 < 10000 {
            print "ALERT: Non-standard port open: " $1
        }'
    }
    
    # Détecter les processus suspects
    detect_suspicious_processes() {
        ps aux | awk 'NR>1 {
            if ($3 > 50 || $4 > 50) {  # CPU ou RAM élevé
                print "ALERT: High resource usage: " $11 " (PID: " $2 ")"
            }
        }'
    }
    
    # Exécuter les détections
    {
        echo "=== Détection d'intrusion - $(date) ==="
        detect_suspicious_ssh
        detect_suspicious_ports
        detect_suspicious_processes
    } >> "$log_file"
}
```

## Chiffrement des communications

### TLS/SSL

**Configuration TLS** :
```bash
#!/bin/bash
# Configuration TLS/SSL

# Générer un certificat auto-signé
generate_self_signed_cert() {
    openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem \
        -days 365 -nodes -subj "/CN=example.com"
}

# Vérifier un certificat
verify_certificate() {
    local cert_file="$1"
    openssl x509 -in "$cert_file" -text -noout
}

# Vérifier la connexion SSL
test_ssl_connection() {
    local host="$1"
    local port="${2:-443}"
    
    echo | openssl s_client -connect "$host:$port" -showcerts 2>/dev/null | \
    openssl x509 -inform pem -noout -text
}
```

### Chiffrement des données au repos

**Chiffrement de disque** :
```bash
#!/bin/bash
# Chiffrement de disque avec LUKS

# Créer un volume chiffré
create_luks_volume() {
    local device="$1"
    local mount_point="$2"
    
    # Créer le volume
    sudo cryptsetup luksFormat "$device"
    
    # Ouvrir le volume
    sudo cryptsetup luksOpen "$device" encrypted_volume
    
    # Formater
    sudo mkfs.ext4 /dev/mapper/encrypted_volume
    
    # Monter
    sudo mount /dev/mapper/encrypted_volume "$mount_point"
}

# Monter un volume chiffré
mount_luks_volume() {
    local device="$1"
    local mount_point="$2"
    
    sudo cryptsetup luksOpen "$device" encrypted_volume
    sudo mount /dev/mapper/encrypted_volume "$mount_point"
}
```

## Sécurisation des services réseau

### Sécurisation Apache/Nginx

**Configuration Apache sécurisée** :
```bash
#!/bin/bash
# Configuration Apache sécurisée

secure_apache() {
    sudo tee -a /etc/apache2/apache2.conf << 'EOF'
# Désactiver les informations serveur
ServerTokens Prod
ServerSignature Off

# Désactiver les répertoires
Options -Indexes

# Désactiver les méthodes HTTP dangereuses
<LimitExcept GET POST HEAD>
    deny from all
</LimitExcept>
EOF
    
    # Activer les modules de sécurité
    sudo a2enmod headers
    sudo a2enmod ssl
    sudo a2enmod rewrite
    
    # Recharger
    sudo systemctl reload apache2
}
```

**Configuration Nginx sécurisée** :
```bash
#!/bin/bash
# Configuration Nginx sécurisée

secure_nginx() {
    sudo tee /etc/nginx/conf.d/security.conf << 'EOF'
# Cacher la version
server_tokens off;

# Headers de sécurité
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Désactiver les méthodes HTTP dangereuses
if ($request_method !~ ^(GET|HEAD|POST|PUT|DELETE|OPTIONS)$ ) {
    return 405;
}
EOF
    
    # Recharger
    sudo nginx -t
    sudo systemctl reload nginx
}
```

## Audit et conformité

### Audit système

**Utilisation de auditd** :
```bash
#!/bin/bash
# Utilisation de auditd

# Installation
sudo apt install auditd  # Debian/Ubuntu
sudo yum install audit   # CentOS/RHEL

# Activer le service
sudo systemctl enable auditd
sudo systemctl start auditd

# Vérifier le statut
sudo auditctl -s

# Ajouter des règles
sudo auditctl -w /etc/passwd -p rwxa -k passwd_changes
sudo auditctl -w /etc/shadow -p rwxa -k shadow_changes
sudo auditctl -w /var/log/auth.log -p rwxa -k auth_log

# Lister les règles
sudo auditctl -l

# Rechercher dans les logs
sudo ausearch -k passwd_changes
sudo ausearch -m USER_LOGIN
```

**Configuration auditd** :
```bash
#!/bin/bash
# Configuration auditd

# Fichier : /etc/audit/auditd.conf

configure_auditd() {
    sudo tee -a /etc/audit/auditd.conf << 'EOF'
# Taille max des fichiers de log
max_log_file = 50
num_logs = 5

# Action quand l'espace disque est plein
space_left_action = email
action_mail_acct = root
admin_space_left_action = suspend
disk_full_action = suspend
disk_error_action = suspend
EOF
    
    # Recharger
    sudo systemctl restart auditd
}
```

### Conformité et durcissement

**Script de durcissement** :
```bash
#!/bin/bash
# Script de durcissement système

harden_system() {
    # Désactiver les services inutiles
    disable_unnecessary_services() {
        local services=("telnet" "rsh" "rlogin" "rexec" "ftp")
        
        for service in "${services[@]}"; do
            sudo systemctl stop "$service" 2>/dev/null || true
            sudo systemctl disable "$service" 2>/dev/null || true
        done
    }
    
    # Configurer sysctl pour la sécurité
    configure_sysctl_security() {
        sudo tee /etc/sysctl.d/99-security.conf << 'EOF'
# Protection contre IP spoofing
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Désactiver ICMP redirects
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Désactiver send redirects
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Protection SYN flood
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 5

# Log les paquets suspects
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Ignorer les ICMP pings
net.ipv4.icmp_echo_ignore_all = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Désactiver source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
EOF
        
        sudo sysctl -p /etc/sysctl.d/99-security.conf
    }
    
    # Exécuter
    disable_unnecessary_services
    configure_sysctl_security
    
    echo "Système durci"
}
```

## Scripts d'automatisation

### Gestionnaire de sécurité réseau

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de sécurité réseau

set -euo pipefail

security_manager() {
    local action="$1"
    shift
    
    case "$action" in
        firewall)
            manage_firewall "$@"
            ;;
        ssh)
            secure_ssh "$@"
            ;;
        fail2ban)
            manage_fail2ban "$@"
            ;;
        audit)
            run_audit "$@"
            ;;
        harden)
            harden_system
            ;;
        *)
            echo "Usage: security_manager {firewall|ssh|fail2ban|audit|harden}"
            exit 1
            ;;
    esac
}

manage_firewall() {
    local subaction="$1"
    shift
    
    case "$subaction" in
        status)
            sudo ufw status verbose
            ;;
        allow)
            sudo ufw allow "$@"
            ;;
        deny)
            sudo ufw deny "$@"
            ;;
        enable)
            sudo ufw enable
            ;;
        disable)
            sudo ufw disable
            ;;
        *)
            echo "Usage: security_manager firewall {status|allow|deny|enable|disable}"
            ;;
    esac
}

secure_ssh() {
    local subaction="$1"
    shift
    
    case "$subaction" in
        config)
            secure_ssh_config
            ;;
        keys)
            generate_ssh_keys "$@"
            ;;
        test)
            sudo sshd -t
            ;;
        reload)
            sudo systemctl reload sshd
            ;;
        *)
            echo "Usage: security_manager ssh {config|keys|test|reload}"
            ;;
    esac
}

manage_fail2ban() {
    local subaction="$1"
    shift
    
    case "$subaction" in
        status)
            sudo fail2ban-client status
            ;;
        ban)
            sudo fail2ban-client set sshd banip "$@"
            ;;
        unban)
            sudo fail2ban-client set sshd unbanip "$@"
            ;;
        reload)
            sudo fail2ban-client reload
            ;;
        *)
            echo "Usage: security_manager fail2ban {status|ban|unban|reload}"
            ;;
    esac
}

run_audit() {
    local audit_type="${1:-full}"
    
    case "$audit_type" in
        full)
            sudo rkhunter --check
            sudo aide --check
            ;;
        quick)
            sudo rkhunter --check --skip-keyfiles --skip-group
            ;;
        *)
            echo "Usage: security_manager audit {full|quick}"
            ;;
    esac
}

# Utilisation
# security_manager firewall status
# security_manager ssh config
# security_manager fail2ban status
# security_manager audit full
# security_manager harden
```

## Conclusion

La sécurité réseau est essentielle pour protéger les systèmes contre les menaces extérieures. En appliquant les principes de défense en profondeur, en configurant correctement les pare-feu, en sécurisant les services comme SSH, et en mettant en place des systèmes de détection d'intrusion, vous pouvez créer un environnement réseau robuste et sécurisé.

Un système bien sécurisé utilise plusieurs couches de protection : pare-feu pour filtrer le trafic, chiffrement pour protéger les communications, authentification forte pour contrôler l'accès, et monitoring pour détecter les anomalies. La compréhension approfondie de ces mécanismes permet de créer des architectures réseau qui résistent aux attaques tout en permettant les communications nécessaires.

Dans le chapitre suivant, nous explorerons en détail les règles iptables et la configuration avancée des pare-feu, découvrant comment créer des règles de filtrage complexes et professionnelles pour protéger efficacement les systèmes Linux.
