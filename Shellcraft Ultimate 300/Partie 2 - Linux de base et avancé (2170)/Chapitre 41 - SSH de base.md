# Chapitre 41 - SSH de base

## Table des matières
- [Introduction](#introduction)
- [Principes du protocole SSH](#principes-du-protocole-ssh)
- [Connexion SSH basique](#connexion-ssh-basique)
- [Authentification et clés](#authentification-et-clés)
- [Configuration du serveur SSH](#configuration-du-serveur-ssh)
- [Transfert de fichiers](#transfert-de-fichiers)
- [Tunnels SSH](#tunnels-ssh)
- [Sécurité et bonnes pratiques](#sécurité-et-bonnes-pratiques)
- [Conclusion](#conclusion)

## Introduction

SSH (Secure Shell) constitue l'épine dorsale de l'administration système moderne. Au-delà d'un simple remplacement du telnet, SSH offre un canal de communication sécurisé, chiffré, et polyvalent pour l'accès distant aux systèmes.

Imaginez SSH comme un tunnel sécurisé traversant l'autoroute de l'internet : il protège vos données de l'espionnage, garantit l'authenticité des communications, et permet d'accéder à distance comme si vous étiez physiquement présent devant la machine.

## Principes du protocole SSH

### Historique et évolution

**Création et développement** :
- **1995** : Création par Tatu Ylönen pour remplacer telnet, rlogin, rsh
- **SSH-1** : Première version (dépréciée pour raisons de sécurité)
- **SSH-2** : Version moderne standardisée (RFC 4251-4256)
- **OpenSSH** : Implémentation open source la plus répandue

**Pourquoi SSH ?** :
- **Chiffrement** : Toutes les communications sont cryptées
- **Authentification** : Vérification de l'identité du serveur et du client
- **Intégrité** : Détection des modifications de données
- **Confidentialité** : Protection contre l'écoute passive

### Architecture du protocole

**Couches du protocole SSH** :
1. **Transport** : Chiffrement, compression, intégrité
2. **Authentification** : Vérification de l'identité utilisateur
3. **Connexion** : Gestion des canaux multiples

**Composants principaux** :
- **sshd** : Serveur SSH (daemon)
- **ssh** : Client SSH
- **ssh-keygen** : Génération de clés
- **ssh-copy-id** : Copie de clés publiques
- **scp** : Copie sécurisée de fichiers
- **sftp** : Transfert de fichiers sécurisé

## Connexion SSH basique

### Installation et vérification

**Vérifier l'installation** :
```bash
# Vérifier si SSH est installé
which ssh
which sshd

# Vérifier la version
ssh -V

# Vérifier le statut du service
sudo systemctl status sshd    # systemd
sudo service ssh status       # SysV init
```

**Installation** :
```bash
# Ubuntu/Debian
sudo apt update
sudo apt install openssh-client openssh-server

# CentOS/RHEL/Fedora
sudo dnf install openssh-clients openssh-server

# Arch Linux
sudo pacman -S openssh

# Démarrer le service
sudo systemctl start sshd
sudo systemctl enable sshd   # Démarrage automatique
```

### Connexion simple

**Syntaxe de base** :
```bash
# Connexion avec nom d'utilisateur
ssh username@hostname

# Exemples
ssh alice@192.168.1.100
ssh alice@server.example.com
ssh alice@server.example.com -p 2222  # Port personnalisé
```

**Première connexion** :
```bash
$ ssh alice@server.example.com

The authenticity of host 'server.example.com (192.168.1.100)' can't be established.
ECDSA key fingerprint is SHA256:AbCdEf123456...
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'server.example.com' to the list of known hosts.
alice@server.example.com's password: 
Welcome to Ubuntu 22.04 LTS
Last login: Mon Jan 15 10:30:45 2024 from 192.168.1.50
alice@server:~$
```

**Fichier known_hosts** :
```bash
# Emplacement
~/.ssh/known_hosts

# Contenu après première connexion
server.example.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ...

# Vérifier les empreintes
ssh-keygen -l -f ~/.ssh/known_hosts
```

### Options de connexion courantes

**Options essentielles** :
```bash
# Spécifier le port
ssh -p 2222 user@host

# Spécifier l'identité (clé privée)
ssh -i ~/.ssh/id_rsa_custom user@host

# Mode verbeux (débogage)
ssh -v user@host      # Niveau 1
ssh -vv user@host    # Niveau 2
ssh -vvv user@host   # Niveau 3 (très verbeux)

# Exécuter une commande distante sans session interactive
ssh user@host "ls -la /tmp"

# Forcer IPv4 ou IPv6
ssh -4 user@host     # IPv4 seulement
ssh -6 user@host     # IPv6 seulement

# Désactiver la vérification de l'empreinte (non recommandé)
ssh -o StrictHostKeyChecking=no user@host
```

**Connexion avec options multiples** :
```bash
# Exemple complet
ssh -v -p 2222 -i ~/.ssh/id_rsa \
    -o UserKnownHostsFile=~/.ssh/custom_hosts \
    alice@server.example.com
```

## Authentification et clés

### Authentification par mot de passe

**Avantages** :
- Simple à configurer
- Pas de gestion de clés nécessaire
- Familiarité pour les utilisateurs

**Inconvénients** :
- Vulnérable aux attaques par force brute
- Nécessite de taper le mot de passe à chaque connexion
- Moins sécurisé que les clés

**Configuration pour désactiver l'authentification par mot de passe** :
```bash
# /etc/ssh/sshd_config
PasswordAuthentication no
```

### Authentification par clés SSH

**Avantages** :
- Plus sécurisé que les mots de passe
- Pas besoin de taper de mot de passe (ou avec passphrase)
- Permet l'automatisation
- Meilleure pratique recommandée

**Génération de clés** :
```bash
# Générer une nouvelle paire de clés
ssh-keygen -t rsa -b 4096

# Avec options avancées
ssh-keygen -t ed25519 -C "alice@example.com" -f ~/.ssh/id_ed25519

# Types de clés supportés
ssh-keygen -t rsa        # RSA (défaut, 2048 bits)
ssh-keygen -t dsa        # DSA (déprécié)
ssh-keygen -t ecdsa      # ECDSA
ssh-keygen -t ed25519    # Ed25519 (recommandé, plus rapide et sécurisé)
```

**Processus de génération** :
```bash
$ ssh-keygen -t ed25519 -C "alice@example.com"

Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/alice/.ssh/id_ed25519): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/alice/.ssh/id_ed25519
Your public key has been saved in /home/alice/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:AbCdEf123456... alice@example.com
```

**Structure des fichiers de clés** :
```bash
~/.ssh/
├── id_rsa              # Clé privée RSA (NE JAMAIS PARTAGER)
├── id_rsa.pub          # Clé publique RSA (peut être partagée)
├── id_ed25519          # Clé privée Ed25519
├── id_ed25519.pub      # Clé publique Ed25519
├── authorized_keys     # Clés publiques autorisées (sur le serveur)
└── known_hosts         # Empreintes des serveurs connus
```

### Copie de clés publiques

**Méthode manuelle** :
```bash
# Afficher la clé publique
cat ~/.ssh/id_rsa.pub

# Copier manuellement sur le serveur
# Sur le serveur :
mkdir -p ~/.ssh
chmod 700 ~/.ssh
echo "clé_publique" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

**Méthode automatique (ssh-copy-id)** :
```bash
# Copier la clé publique vers le serveur
ssh-copy-id user@host

# Avec port personnalisé
ssh-copy-id -p 2222 user@host

# Spécifier la clé
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host

# Exemple
$ ssh-copy-id alice@server.example.com

/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/home/alice/.ssh/id_rsa.pub"
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) should be installed (if you are prompted it is because they are already installed)

Number of key(s) added: 1
```

**Vérification de l'authentification** :
```bash
# Tester la connexion sans mot de passe
ssh user@host

# Si configuré correctement, la connexion devrait être automatique
# (ou demander seulement la passphrase de la clé)
```

### Gestion des clés multiples

**Fichier de configuration SSH** :
```bash
# ~/.ssh/config

# Serveur par défaut
Host server1
    HostName server1.example.com
    User alice
    Port 22
    IdentityFile ~/.ssh/id_rsa

# Serveur avec clé spécifique
Host server2
    HostName 192.168.1.100
    User bob
    Port 2222
    IdentityFile ~/.ssh/id_ed25519_custom
    IdentitiesOnly yes

# Serveur avec proxy jump
Host server3
    HostName server3.example.com
    User alice
    ProxyJump bastion.example.com
    IdentityFile ~/.ssh/id_rsa

# Utilisation
ssh server1    # Au lieu de ssh alice@server1.example.com
ssh server2    # Utilise automatiquement les bonnes options
```

**Agent SSH** :
```bash
# Démarrer l'agent SSH
eval "$(ssh-agent -s)"

# Ajouter une clé à l'agent
ssh-add ~/.ssh/id_rsa

# Lister les clés dans l'agent
ssh-add -l

# Supprimer une clé de l'agent
ssh-add -d ~/.ssh/id_rsa

# Supprimer toutes les clés
ssh-add -D

# Ajouter automatiquement au démarrage (dans ~/.bashrc)
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_rsa
fi
```

## Configuration du serveur SSH

### Fichier de configuration serveur

**Emplacement** :
```bash
# Configuration principale
/etc/ssh/sshd_config

# Configuration par utilisateur (si activée)
~/.ssh/config
```

**Paramètres essentiels** :
```bash
# /etc/ssh/sshd_config

# Port d'écoute
Port 22

# Adresses d'écoute
ListenAddress 0.0.0.0          # Toutes les interfaces
ListenAddress 192.168.1.100     # Interface spécifique

# Protocoles supportés
Protocol 2                     # SSH-2 seulement (recommandé)

# Authentification
PermitRootLogin no             # Interdire connexion root directe
PasswordAuthentication yes      # Autoriser mots de passe
PubkeyAuthentication yes        # Autoriser clés publiques
PermitEmptyPasswords no         # Interdire mots de passe vides

# Utilisateurs autorisés/interdits
AllowUsers alice bob           # Seulement ces utilisateurs
DenyUsers hacker               # Interdire cet utilisateur
AllowGroups admins             # Seulement ce groupe

# Timeout et connexions
ClientAliveInterval 60         # Envoyer keepalive toutes les 60s
ClientAliveCountMax 3          # Nombre de keepalives avant déconnexion
MaxStartups 10                 # Limite de connexions simultanées

# Logging
LogLevel INFO                  # Niveau de log
SyslogFacility AUTH            # Facility syslog
```

**Recharger la configuration** :
```bash
# Vérifier la configuration avant de recharger
sudo sshd -t

# Recharger la configuration
sudo systemctl reload sshd     # systemd
sudo service ssh reload         # SysV init

# Redémarrer le service
sudo systemctl restart sshd
```

### Sécurisation du serveur SSH

**Recommandations de sécurité** :
```bash
# /etc/ssh/sshd_config

# Changer le port par défaut (défense en profondeur)
Port 2222

# Désactiver l'authentification root
PermitRootLogin no

# Désactiver l'authentification par mot de passe
PasswordAuthentication no
PubkeyAuthentication yes

# Limiter les tentatives de connexion
MaxAuthTries 3

# Désactiver les fonctionnalités inutiles
X11Forwarding no               # Si pas besoin de X11
AllowTcpForwarding no          # Si pas besoin de port forwarding

# Utiliser seulement les algorithmes modernes
KexAlgorithms curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com
```

**Fail2ban pour protection** :
```bash
# Installation
sudo apt install fail2ban

# Configuration SSH
# /etc/fail2ban/jail.local
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

## Transfert de fichiers

### SCP (Secure Copy)

**Syntaxe de base** :
```bash
# Copier un fichier vers le serveur distant
scp file.txt user@host:/path/to/destination/

# Copier depuis le serveur distant
scp user@host:/path/to/file.txt /local/destination/

# Copier un répertoire récursivement
scp -r directory/ user@host:/path/to/destination/

# Avec port personnalisé
scp -P 2222 file.txt user@host:/destination/

# Afficher la progression
scp -v file.txt user@host:/destination/
```

**Exemples pratiques** :
```bash
# Copier un fichier local vers distant
scp backup.tar.gz alice@server.example.com:/home/alice/backups/

# Copier depuis distant vers local
scp alice@server.example.com:/var/log/app.log ./logs/

# Copier plusieurs fichiers
scp file1.txt file2.txt alice@server.example.com:/tmp/

# Copier avec préservation des permissions
scp -p file.txt user@host:/destination/
```

### SFTP (SSH File Transfer Protocol)

**Mode interactif** :
```bash
# Connexion SFTP
sftp user@host

# Commandes SFTP
sftp> ls                    # Lister fichiers distants
sftp> lls                   # Lister fichiers locaux
sftp> cd /remote/path       # Changer répertoire distant
sftp> lcd /local/path       # Changer répertoire local
sftp> put file.txt          # Envoyer fichier
sftp> get file.txt          # Télécharger fichier
sftp> mkdir newdir          # Créer répertoire distant
sftp> rm file.txt           # Supprimer fichier distant
sftp> help                  # Aide
sftp> quit                  # Quitter
```

**Mode non-interactif** :
```bash
# Télécharger un fichier
sftp user@host:/path/to/file.txt /local/destination/

# Envoyer un fichier
sftp user@host << EOF
put /local/file.txt /remote/destination/
quit
EOF
```

### Rsync via SSH

**Utilisation avec SSH** :
```bash
# Synchronisation via SSH
rsync -avz -e ssh /local/dir/ user@host:/remote/dir/

# Options utiles
rsync -avz --progress -e ssh file.txt user@host:/destination/
# -a : archive (préserve permissions, timestamps)
# -v : verbeux
# -z : compression
# --progress : afficher progression

# Exclure des fichiers
rsync -avz -e ssh --exclude '*.log' /local/ user@host:/remote/

# Synchronisation bidirectionnelle
rsync -avz -e ssh /local/ user@host:/remote/
rsync -avz -e ssh user@host:/remote/ /local/
```

## Tunnels SSH

### Port forwarding local

**Rediriger un port local vers un port distant** :
```bash
# Syntaxe
ssh -L [bind_address:]local_port:remote_host:remote_port user@ssh_server

# Exemple : accéder à un serveur web distant via SSH
ssh -L 8080:localhost:80 user@server.example.com

# Accès local
# http://localhost:8080 → http://server.example.com:80

# Avec bind address spécifique
ssh -L 127.0.0.1:8080:localhost:80 user@server.example.com
```

**Cas d'usage** :
```bash
# Accéder à une base de données distante
ssh -L 3306:localhost:3306 user@db-server.example.com
# mysql -h 127.0.0.1 -P 3306

# Accéder à un service web interne
ssh -L 8080:internal-server:80 user@bastion.example.com
```

### Port forwarding distant

**Rediriger un port distant vers un port local** :
```bash
# Syntaxe
ssh -R [bind_address:]remote_port:local_host:local_port user@ssh_server

# Exemple : exposer un service local sur le serveur distant
ssh -R 8080:localhost:3000 user@server.example.com

# Sur le serveur distant, http://localhost:8080 accède au service local
```

**Configuration serveur pour remote forwarding** :
```bash
# /etc/ssh/sshd_config
GatewayPorts yes              # Permettre binding sur toutes les interfaces
AllowTcpForwarding yes        # Autoriser le port forwarding
```

### Tunnels dynamiques (SOCKS)

**Proxy SOCKS via SSH** :
```bash
# Créer un proxy SOCKS
ssh -D 1080 user@server.example.com

# Utilisation dans le navigateur
# Configuration proxy : SOCKS5, localhost:1080

# Avec authentification
ssh -D 1080 -N -f user@server.example.com
# -N : ne pas exécuter de commande
# -f : passer en arrière-plan
```

**Utilisation avec applications** :
```bash
# curl via proxy SOCKS
curl --socks5-hostname localhost:1080 http://example.com

# wget via proxy SOCKS
export http_proxy=socks5://localhost:1080
wget http://example.com
```

## Sécurité et bonnes pratiques

### Bonnes pratiques générales

**Côté client** :
```bash
# Toujours vérifier les empreintes de clés
ssh-keygen -l -f ~/.ssh/known_hosts

# Utiliser des clés fortes
ssh-keygen -t ed25519 -a 100    # 100 rounds de dérivation

# Protéger les clés privées
chmod 600 ~/.ssh/id_*
chmod 700 ~/.ssh

# Ne jamais partager les clés privées
# Utiliser des passphrases fortes
```

**Côté serveur** :
```bash
# Désactiver l'authentification root
PermitRootLogin no

# Limiter les utilisateurs autorisés
AllowUsers alice bob

# Utiliser des ports non standards (défense en profondeur)
Port 2222

# Désactiver les protocoles obsolètes
Protocol 2

# Activer le logging
LogLevel VERBOSE
```

### Détection d'intrusions

**Surveillance des connexions** :
```bash
# Voir les connexions SSH actives
who
w

# Voir les dernières connexions
last
lastlog

# Logs d'authentification
sudo tail -f /var/log/auth.log        # Debian/Ubuntu
sudo tail -f /var/log/secure          # CentOS/RHEL

# Filtrer les tentatives échouées
sudo grep "Failed password" /var/log/auth.log
```

**Script de monitoring** :
```bash
#!/bin/bash
# monitor_ssh.sh

LOG_FILE="/var/log/auth.log"
ALERT_EMAIL="admin@example.com"

# Compter les tentatives échouées
FAILED_ATTEMPTS=$(grep "Failed password" "$LOG_FILE" | tail -20 | wc -l)

if [ "$FAILED_ATTEMPTS" -gt 10 ]; then
    echo "Alerte: $FAILED_ATTEMPTS tentatives SSH échouées récentes" | \
        mail -s "Alerte SSH" "$ALERT_EMAIL"
fi
```

### Hardening SSH

**Configuration sécurisée complète** :
```bash
# /etc/ssh/sshd_config

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

# Utilisateurs
AllowUsers alice bob
DenyUsers hacker

# Timeouts
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 60

# Désactiver fonctionnalités inutiles
X11Forwarding no
AllowTcpForwarding no
PermitTunnel no

# Algorithmes modernes seulement
KexAlgorithms curve25519-sha256@libssh.org
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com

# Logging
LogLevel VERBOSE
SyslogFacility AUTH
```

## Conclusion

SSH représente bien plus qu'un simple outil de connexion distante : c'est la fondation de l'administration système moderne, offrant sécurité, flexibilité et puissance. La maîtrise de SSH - de la connexion basique aux tunnels avancés - est essentielle pour tout administrateur système et développeur DevOps.

Les concepts appris ici - authentification par clés, port forwarding, transfert de fichiers sécurisé - constituent les bases sur lesquelles vous construirez des solutions d'automatisation et d'administration plus complexes. Dans les chapitres suivants, nous explorerons le transfert de fichiers avancé avec rsync, les tunnels SSH complexes, et le multiplexage pour optimiser les performances.

La sécurité SSH n'est pas optionnelle : elle est fondamentale. Appliquez toujours les bonnes pratiques de sécurité, surveillez les connexions, et gardez vos systèmes à jour pour protéger contre les vulnérabilités connues.

Dans le prochain chapitre, nous approfondirons le transfert de fichiers avec scp et rsync, explorant les techniques avancées pour synchroniser efficacement les données entre systèmes.