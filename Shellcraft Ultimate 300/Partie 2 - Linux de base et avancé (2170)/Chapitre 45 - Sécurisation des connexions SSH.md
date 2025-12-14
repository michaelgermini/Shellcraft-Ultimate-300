# Chapitre 45 - Sécurisation des connexions SSH

## Table des matières
- [Introduction](#introduction)
- [Configuration de sécurité de base](#configuration-de-sécurité-de-base)
- [Authentification renforcée](#authentification-renforcée)
- [Gestion avancée des clés SSH](#gestion-avancée-des-clés-ssh)
- [Chiffrement et algorithmes](#chiffrement-et-algorithmes)
- [Contrôle d'accès avancé](#contrôle-daccès-avancé)
- [Authentification multi-facteurs (MFA)](#authentification-multi-facteurs-mfa)
- [Monitoring et audit](#monitoring-et-audit)
- [Détection d'intrusion](#détection-dintrusion)
- [Fail2ban et protection avancée](#fail2ban-et-protection-avancée)
- [Hardening complet](#hardening-complet)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La sécurisation des connexions SSH constitue un impératif absolu dans tout environnement de production. Au-delà de la configuration par défaut, il existe de nombreuses couches de sécurité qui peuvent être activées pour protéger contre les attaques par force brute, les écoutes, et les compromissions.

Imaginez la sécurisation SSH comme les fortifications d'un château moderne : des douves (pare-feu), des portes blindées (chiffrement fort), des gardes (authentification multi-facteurs), et des caméras de surveillance (logging et monitoring) pour repousser les assaillants les plus déterminés. Dans un environnement où SSH est souvent la seule porte d'entrée vers vos serveurs, chaque couche de sécurité compte.

## Configuration de sécurité de base

### Configuration minimale sécurisée

**Configuration de base** :
```bash
#!/bin/bash
# Configuration SSH sécurisée de base

secure_ssh_basic() {
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

# Timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# Logging
LogLevel VERBOSE
SyslogFacility AUTH
EOF
    
    sudo sshd -t
    sudo systemctl reload sshd
    
    echo "SSH sécurisé configuré"
}
```

## Authentification renforcée

### Clés SSH fortes

**Génération de clés sécurisées** :
```bash
#!/bin/bash
# Génération de clés SSH sécurisées

# Clé ED25519 (recommandée)
ssh-keygen -t ed25519 -a 100 -C "user@hostname"
# -a 100 : 100 rounds de dérivation (plus sécurisé)

# Clé RSA 4096 bits (alternative)
ssh-keygen -t rsa -b 4096 -a 100 -C "user@hostname"

# Clé avec passphrase forte
ssh-keygen -t ed25519 -a 100 -C "user@hostname"
# Entrer une passphrase forte lors de la génération

# Vérifier la force de la clé
ssh-keygen -l -f ~/.ssh/id_ed25519.pub
```

**Gestion des clés** :
```bash
#!/bin/bash
# Gestion des clés SSH

# Permissions correctes
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_*
chmod 644 ~/.ssh/*.pub
chmod 600 ~/.ssh/authorized_keys
chmod 600 ~/.ssh/known_hosts

# Vérifier les empreintes
ssh-keygen -l -f ~/.ssh/id_ed25519.pub

# Lister les clés
ssh-add -l

# Supprimer une clé compromise
ssh-keygen -R hostname
```

## Gestion avancée des clés SSH

### Clés avec restrictions

**Clés avec restrictions** :
```bash
#!/bin/bash
# Clés SSH avec restrictions

# Clé avec restriction de commande
# Dans ~/.ssh/authorized_keys sur le serveur
restrict-key-command() {
    cat >> ~/.ssh/authorized_keys << 'EOF'
command="/usr/local/bin/restricted-command.sh",no-port-forwarding,no-X11-forwarding ssh-ed25519 AAAAC3...
EOF
}

# Clé avec restriction d'IP source
restrict-key-source() {
    cat >> ~/.ssh/authorized_keys << 'EOF'
from="192.168.1.100",no-port-forwarding ssh-ed25519 AAAAC3...
EOF
}

# Clé avec restriction de temps
restrict-key-time() {
    cat >> ~/.ssh/authorized_keys << 'EOF'
no-port-forwarding,no-X11-forwarding,no-pty ssh-ed25519 AAAAC3...
EOF
}
```

### Rotation des clés

**Rotation des clés SSH** :
```bash
#!/bin/bash
# Rotation des clés SSH

rotate_ssh_keys() {
    local user="$1"
    local old_key="$2"
    local new_key="$3"
    
    # Générer nouvelle clé
    ssh-keygen -t ed25519 -a 100 -f "$new_key"
    
    # Ajouter nouvelle clé au serveur
    ssh-copy-id -i "${new_key}.pub" "$user@server"
    
    # Vérifier que la nouvelle clé fonctionne
    ssh -i "$new_key" "$user@server" "echo 'New key works'"
    
    # Supprimer l'ancienne clé
    ssh "$user@server" "sed -i '/$old_key/d' ~/.ssh/authorized_keys"
    
    echo "Rotation des clés terminée"
}
```

## Chiffrement et algorithmes

### Algorithmes modernes

**Configuration des algorithmes** :
```bash
#!/bin/bash
# Configuration des algorithmes modernes

configure_modern_algorithms() {
    sudo tee -a /etc/ssh/sshd_config << 'EOF'
# Algorithmes d'échange de clés (KEX)
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512

# Algorithmes de chiffrement
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com,aes256-ctr,aes192-ctr,aes128-ctr

# Algorithmes MAC (Message Authentication Code)
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,hmac-sha2-256,hmac-sha2-512

# Désactiver les algorithmes obsolètes
# (déjà fait en spécifiant seulement les modernes)
EOF
    
    sudo sshd -t
    sudo systemctl reload sshd
}
```

**Vérification des algorithmes** :
```bash
#!/bin/bash
# Vérifier les algorithmes supportés

# Scanner les algorithmes du serveur
ssh -Q kex        # Algorithmes KEX
ssh -Q cipher     # Algorithmes de chiffrement
ssh -Q mac        # Algorithmes MAC

# Test de connexion avec algorithmes spécifiques
ssh -o KexAlgorithms=curve25519-sha256@libssh.org user@host
```

## Contrôle d'accès avancé

### Restrictions par utilisateur

**Contrôle d'accès granulaire** :
```bash
#!/bin/bash
# Contrôle d'accès avancé

configure_access_control() {
    sudo tee -a /etc/ssh/sshd_config << 'EOF'
# Utilisateurs autorisés
AllowUsers alice bob charlie
DenyUsers hacker attacker

# Groupes autorisés
AllowGroups ssh-users admins
DenyGroups restricted

# Restrictions par Match
Match User alice
    AllowTcpForwarding yes
    X11Forwarding yes

Match User deploy
    AllowTcpForwarding no
    X11Forwarding no
    ForceCommand /usr/local/bin/deploy-script.sh

Match Address 192.168.1.0/24
    PasswordAuthentication yes

Match Group admins
    PermitRootLogin yes
EOF
    
    sudo sshd -t
    sudo systemctl reload sshd
}
```

### Restrictions par IP

**Contrôle d'accès par IP** :
```bash
#!/bin/bash
# Contrôle d'accès par IP

# Avec TCP Wrappers
# /etc/hosts.allow
sshd: 192.168.1.0/24
sshd: 10.0.0.0/8

# /etc/hosts.deny
sshd: ALL

# Avec iptables
sudo iptables -A INPUT -p tcp --dport 22 -s 192.168.1.0/24 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j DROP
```

## Authentification multi-facteurs (MFA)

### Google Authenticator

**Configuration Google Authenticator** :
```bash
#!/bin/bash
# Configuration Google Authenticator

# Installation
sudo apt install libpam-google-authenticator

# Configuration pour un utilisateur
google-authenticator

# Questions :
# - Time-based tokens : yes
# - Update .google_authenticator : yes
# - Disallow multiple uses : yes
# - Increase window : no
# - Rate limiting : yes

# Configuration PAM
sudo tee -a /etc/pam.d/sshd << 'EOF'
auth required pam_google_authenticator.so
EOF

# Configuration SSH
sudo tee -a /etc/ssh/sshd_config << 'EOF'
ChallengeResponseAuthentication yes
AuthenticationMethods publickey,keyboard-interactive
EOF

sudo systemctl reload sshd
```

### Authentification par certificat

**Certificats SSH** :
```bash
#!/bin/bash
# Authentification par certificat SSH

# Générer une CA
ssh-keygen -t ed25519 -f ca_key -C "SSH CA"

# Signer une clé utilisateur
ssh-keygen -s ca_key -I user_id -n user1,user2 id_ed25519.pub

# Configuration serveur
sudo tee -a /etc/ssh/sshd_config << 'EOF'
TrustedUserCAKeys /etc/ssh/ca_key.pub
EOF

sudo systemctl reload sshd
```

## Monitoring et audit

### Logging avancé

**Configuration du logging** :
```bash
#!/bin/bash
# Configuration du logging avancé

configure_ssh_logging() {
    sudo tee -a /etc/ssh/sshd_config << 'EOF'
# Niveau de log détaillé
LogLevel VERBOSE

# Facility syslog
SyslogFacility AUTH

# Logs séparés
# (nécessite configuration syslog)
EOF
    
    # Configuration rsyslog pour logs SSH séparés
    sudo tee /etc/rsyslog.d/30-sshd.conf << 'EOF'
if $programname == 'sshd' then /var/log/sshd.log
& stop
EOF
    
    sudo systemctl restart rsyslog
    sudo systemctl reload sshd
}
```

**Analyse des logs** :
```bash
#!/bin/bash
# Analyse des logs SSH

# Tentatives échouées
analyze_failed_attempts() {
    sudo grep "Failed password" /var/log/auth.log | \
    awk '{print $11}' | sort | uniq -c | sort -rn
}

# Connexions réussies
analyze_successful_logins() {
    sudo grep "Accepted" /var/log/auth.log | \
    awk '{print $11, $9}' | sort | uniq -c
}

# Tentatives depuis IPs suspectes
analyze_suspicious_ips() {
    sudo grep "Failed" /var/log/auth.log | \
    awk '{print $11}' | sort | uniq -c | \
    awk '$1 > 10 {print $2, $1}'
}
```

## Détection d'intrusion

### Scripts de détection

**Détection automatique** :
```bash
#!/bin/bash
# Détection d'intrusion SSH

detect_ssh_intrusion() {
    local log_file="/var/log/auth.log"
    local threshold="${1:-5}"
    
    # Détecter les tentatives multiples depuis une IP
    local suspicious_ips=$(sudo grep "Failed password" "$log_file" | \
        tail -100 | awk '{print $11}' | sort | uniq -c | \
        awk -v thresh="$threshold" '$1 > thresh {print $2}')
    
    if [ -n "$suspicious_ips" ]; then
        echo "⚠️  IPs suspectes détectées :"
        echo "$suspicious_ips"
        
        # Optionnel : Bannir automatiquement
        # for ip in $suspicious_ips; do
        #     sudo iptables -A INPUT -s "$ip" -j DROP
        # done
    fi
}

# Monitoring continu
monitor_ssh_continuously() {
    while true; do
        detect_ssh_intrusion
        sleep 60
    done
}
```

## Fail2ban et protection avancée

### Configuration Fail2ban avancée

**Configuration complète** :
```bash
#!/bin/bash
# Configuration Fail2ban avancée

configure_fail2ban_advanced() {
    sudo tee /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 192.168.1.0/24
bantime = 3600
findtime = 600
maxretry = 3
destemail = admin@example.com
sendername = Fail2Ban
action = %(action_mwl)s

[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
action = %(action_mwl)s

[sshd-ddos]
enabled = true
port = 22
filter = sshd-ddos
logpath = /var/log/auth.log
maxretry = 10
findtime = 600
bantime = 3600
EOF
    
    sudo systemctl restart fail2ban
    echo "Fail2ban configuré"
}
```

**Gestion Fail2ban** :
```bash
#!/bin/bash
# Gestion Fail2ban

# Statut
fail2ban_status() {
    sudo fail2ban-client status
    sudo fail2ban-client status sshd
}

# Bannir une IP
ban_ip() {
    local ip="$1"
    sudo fail2ban-client set sshd banip "$ip"
}

# Débannir une IP
unban_ip() {
    local ip="$1"
    sudo fail2ban-client set sshd unbanip "$ip"
}

# Lister les IPs bannies
list_banned() {
    sudo fail2ban-client get sshd banned
}
```

## Hardening complet

### Script de durcissement complet

**Durcissement SSH complet** :
```bash
#!/bin/bash
# Durcissement SSH complet

harden_ssh_complete() {
    # Sauvegarder la configuration existante
    sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup.$(date +%Y%m%d)
    
    # Configuration complète
    sudo tee /etc/ssh/sshd_config << 'EOF'
# Port et interfaces
Port 2222
ListenAddress 192.168.1.100

# Protocole
Protocol 2

# Authentification
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30

# Utilisateurs
AllowUsers alice bob
DenyUsers hacker

# Timeouts
ClientAliveInterval 300
ClientAliveCountMax 2
MaxStartups 10:30:100

# Désactiver fonctionnalités
X11Forwarding no
AllowTcpForwarding no
PermitTunnel no
AllowAgentForwarding no
PermitUserEnvironment no

# Algorithmes modernes
KexAlgorithms curve25519-sha256@libssh.org,diffie-hellman-group16-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com

# Logging
LogLevel VERBOSE
SyslogFacility AUTH

# Sécurité supplémentaire
UsePrivilegeSeparation yes
StrictModes yes
Compression no
TCPKeepAlive yes
EOF
    
    # Vérifier la configuration
    sudo sshd -t
    
    if [ $? -eq 0 ]; then
        sudo systemctl reload sshd
        echo "SSH durci avec succès"
    else
        echo "Erreur dans la configuration SSH"
        sudo cp /etc/ssh/sshd_config.backup.* /etc/ssh/sshd_config
        exit 1
    fi
}
```

## Scripts d'automatisation

### Gestionnaire de sécurité SSH

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de sécurité SSH

set -euo pipefail

ssh_security_manager() {
    local action="$1"
    shift
    
    case "$action" in
        harden)
            harden_ssh_complete
            ;;
        audit)
            audit_ssh_config
            ;;
        monitor)
            monitor_ssh_connections
            ;;
        rotate-keys)
            rotate_ssh_keys "$@"
            ;;
        *)
            echo "Usage: ssh_security_manager {harden|audit|monitor|rotate-keys}"
            exit 1
            ;;
    esac
}

audit_ssh_config() {
    echo "=== Audit de configuration SSH ==="
    
    # Vérifier les permissions
    echo "Permissions :"
    ls -la /etc/ssh/sshd_config
    
    # Vérifier la configuration
    echo ""
    echo "Vérification de la configuration :"
    sudo sshd -t
    
    # Analyser les paramètres de sécurité
    echo ""
    echo "Paramètres de sécurité :"
    sudo grep -E "^(PermitRootLogin|PasswordAuthentication|Port|Protocol)" /etc/ssh/sshd_config
}

monitor_ssh_connections() {
    while true; do
        clear
        echo "=== Monitoring SSH - $(date) ==="
        echo ""
        
        echo "Connexions actives :"
        who
        echo ""
        
        echo "Tentatives récentes échouées :"
        sudo tail -20 /var/log/auth.log | grep "Failed" | tail -5
        
        sleep 10
    done
}

# Utilisation
# ssh_security_manager harden
# ssh_security_manager audit
# ssh_security_manager monitor
```

## Conclusion

La sécurisation des connexions SSH est un processus continu qui nécessite plusieurs couches de protection. En combinant une configuration sécurisée, une authentification forte, un monitoring actif, et des outils comme Fail2ban, vous pouvez créer un environnement SSH robuste qui résiste aux attaques courantes.

Un système bien sécurisé utilise toutes les couches disponibles : chiffrement fort, authentification par clés avec restrictions, monitoring et détection d'intrusion, et durcissement complet de la configuration. La compréhension approfondie de ces mécanismes permet de créer des systèmes qui résistent même aux attaques sophistiquées.

Dans le chapitre suivant, nous explorerons les sauvegardes et snapshots, découvrant comment protéger vos données avec des stratégies de sauvegarde robustes et des systèmes de snapshots pour la récupération rapide.
