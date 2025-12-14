# Chapitre 14 - Introduction au réseau

## Table des matières
- [Introduction](#introduction)
- [Qu'est-ce que le réseau informatique ?](#quest-ce-que-le-réseau-informatique-)
- [Le modèle OSI et TCP/IP](#le-modèle-osi-et-tcpip)
- [Adresses IP et masquage](#adresses-ip-et-masquage)
- [Ports et services](#ports-et-services)
- [DNS : résolution de noms](#dns--résolution-de-noms)
- [Interfaces réseau](#interfaces-réseau)
- [Commandes réseau essentielles](#commandes-réseau-essentielles)
- [Configuration réseau](#configuration-réseau)
- [Dépannage réseau](#dépannage-réseau)
- [Sécurité réseau de base](#sécurité-réseau-de-base)
- [Conclusion](#conclusion)

## Introduction

Le réseau constitue l'infrastructure invisible qui connecte les ordinateurs modernes, permettant la communication, le partage de ressources et l'accès aux services distants. Comprendre les concepts réseau fondamentaux est essentiel pour tout administrateur système et développeur moderne.

Imaginez le réseau comme le système circulatoire d'un organisme vivant : les ordinateurs sont les organes, les câbles et ondes radio sont les vaisseaux sanguins, et les protocoles sont les règles qui régulent le flux d'information. Un réseau bien conçu permet à chaque partie du système de communiquer efficacement avec les autres.

## Qu'est-ce que le réseau informatique ?

### Définition et portée

**Un réseau informatique** est un ensemble d'équipements interconnectés qui peuvent échanger des informations selon des règles prédéfinies (protocoles).

### Types de réseaux

**Par étendue géographique** :
- **PAN** (Personal Area Network) : Bluetooth, USB
- **LAN** (Local Area Network) : réseau local (maison, entreprise)
- **MAN** (Metropolitan Area Network) : réseau métropolitain
- **WAN** (Wide Area Network) : Internet

**Par topologie** :
- **Étoile** : tous les équipements connectés à un point central
- **Bus** : tous les équipements sur une ligne commune
- **Anneau** : équipements connectés en cercle
- **Maillé** : connexions multiples entre équipements

### Rôles dans un réseau

**Clients** : Demandent des services
- Ordinateurs personnels
- Téléphones mobiles
- Tablettes

**Serveurs** : Fournissent des services
- Serveurs web (HTTP)
- Serveurs de fichiers
- Serveurs de messagerie

**Équipements d'infrastructure** :
- Routeurs : aiguillage du trafic
- Commutateurs : connexions locales
- Points d'accès WiFi : réseau sans fil

## Le modèle OSI et TCP/IP

### Le modèle OSI (Open Systems Interconnection)

**7 couches du modèle OSI** :

**Couche 7 - Application** : Interface utilisateur
- HTTP, FTP, SMTP, SSH
- Protocoles de haut niveau

**Couche 6 - Présentation** : Format des données
- Encodage, compression, chiffrement
- Traduction des formats

**Couche 5 - Session** : Gestion des sessions
- Établissement, maintenance, terminaison
- Synchronisation des dialogues

**Couche 4 - Transport** : Fiabilité de la transmission
- TCP (fiable), UDP (non fiable)
- Contrôle de flux, correction d'erreurs

**Couche 3 - Réseau** : Routage et adressage
- IP, ICMP, ARP
- Routage entre réseaux

**Couche 2 - Liaison** : Communication locale
- Ethernet, WiFi, PPP
- Adresses MAC, trames

**Couche 1 - Physique** : Support matériel
- Câbles, ondes radio, fibre optique
- Signaux électriques/optiques

### Le modèle TCP/IP (plus pratique)

**4 couches simplifiées** :

**Application** : Couches 5-7 OSI
- HTTP, FTP, DNS, etc.

**Transport** : Couche 4 OSI
- TCP, UDP

**Internet** : Couche 3 OSI
- IP, ICMP, ARP

**Accès réseau** : Couches 1-2 OSI
- Ethernet, WiFi

## Adresses IP et masquage

### Adresses IPv4

**Structure d'une adresse IPv4** :
```
192.168.1.100
├──┼───┼──┘
Classe A   │
192.0.0.0 - 223.255.255.255
```

**Classes d'adresses** :
- **Classe A** : 1.0.0.0 - 126.255.255.255 (réseaux très grands)
- **Classe B** : 128.0.0.0 - 191.255.255.255 (réseaux moyens)
- **Classe C** : 192.0.0.0 - 223.255.255.255 (réseaux petits)
- **Classe D** : 224.0.0.0 - 239.255.255.255 (multicast)
- **Classe E** : 240.0.0.0 - 255.255.255.255 (réservé)

### Masque de sous-réseau

**Notation CIDR** :
```bash
# Adresse avec masque
192.168.1.0/24

# Équivalent à
192.168.1.0
255.255.255.0
```

**Calcul des sous-réseaux** :
```bash
# Masque /24 = 255.255.255.0
# 256 adresses disponibles (0-255)
# Adresses utilisables : 192.168.1.1 - 192.168.1.254
# Adresse de réseau : 192.168.1.0
# Adresse de broadcast : 192.168.1.255
```

### Adresses spéciales

**Adresses privées (RFC 1918)** :
- **10.0.0.0/8** : Classe A privée
- **172.16.0.0/12** : Classe B privée
- **192.168.0.0/16** : Classe C privée

**Adresses de loopback** :
- **127.0.0.1** : localhost, toujours point vers la machine locale

**Adresses réservées** :
- **0.0.0.0** : Adresse par défaut, "toute adresse"
- **255.255.255.255** : Broadcast général

## Ports et services

### Concept de port

**Numéros de port** : 0-65535
- **0-1023** : Ports bien connus (root seulement)
- **1024-49151** : Ports enregistrés
- **49152-65535** : Ports dynamiques/privés

### Services courants et leurs ports

**Ports TCP courants** :
```bash
# Services essentiels
22   : SSH (Secure Shell)
23   : Telnet (non sécurisé)
25   : SMTP (envoi d'emails)
53   : DNS (résolution de noms)
80   : HTTP (web)
443  : HTTPS (web sécurisé)
3306 : MySQL
5432 : PostgreSQL
```

**Ports UDP courants** :
```bash
# Protocoles sans connexion
53   : DNS
67   : DHCP (serveur)
68   : DHCP (client)
123  : NTP (horloge réseau)
```

### Sockets : IP + Port

**Définition complète d'une connexion** :
```
192.168.1.100:80
├──┼─────────┘
IP   Port
```

**Exemples** :
- **192.168.1.100:80** → Serveur web sur cette machine
- **127.0.0.1:3306** → Base de données MySQL locale
- **0.0.0.0:22** → SSH accessible depuis n'importe quelle interface

## DNS : résolution de noms

### Principe du DNS

**Domain Name System** : Traduction nom ↔ adresse IP
```bash
# Résolution
www.example.com → 93.184.216.34

# Reverse DNS
93.184.216.34 → www.example.com
```

### Hiérarchie DNS

**Structure arborescente** :
```
. (racine)
├── com
│   ├── example
│   │   └── www
│   └── google
│       └── www
└── org
    └── wikipedia
        └── www
```

### Serveurs DNS

**Types de serveurs** :
- **Root servers** : 13 serveurs mondiaux (a.root-servers.net, etc.)
- **TLD servers** : Serveurs de domaines de premier niveau (.com, .org)
- **Authoritative servers** : Serveurs de domaine spécifiques
- **Recursive resolvers** : Serveurs cache (souvent fournis par le FAI)

### Configuration DNS

**Fichier /etc/resolv.conf** :
```bash
# Serveurs DNS à utiliser
nameserver 8.8.8.8        # Google DNS
nameserver 1.1.1.1        # Cloudflare DNS
nameserver 192.168.1.1    # DNS local

# Options
options timeout:2
options attempts:3
```

**Test de résolution** :
```bash
# Résoudre un nom
nslookup www.google.com

# Résolution détaillée
dig www.google.com

# Reverse DNS
dig -x 8.8.8.8
```

## Interfaces réseau

### Types d'interfaces

**Interfaces physiques** :
```bash
# Ethernet câblé
eth0, enp0s3, eno1

# WiFi
wlan0, wlp2s0

# Interfaces spéciales
lo : loopback (127.0.0.1)
```

**Interfaces virtuelles** :
```bash
# VPN
tun0, tap0

# Bridges
br0

# VLAN
eth0.10 (VLAN 10)
```

### Configuration d'interfaces

**Visualisation** :
```bash
# Toutes les interfaces
ip addr show
ip a  # Version courte

# Interfaces actives seulement
ip link show up
```

**Informations détaillées** :
```bash
# Statistiques d'une interface
ip -s link show eth0

# Table de routage
ip route show
ip r  # Version courte
```

## Commandes réseau essentielles

### ping : test de connectivité

**Utilisation de base** :
```bash
# Test simple
ping google.com

# Nombre limité de paquets
ping -c 4 google.com

# Intervalle personnalisé
ping -i 0.5 google.com

# Test de connectivité locale
ping 127.0.0.1
```

### traceroute : chemin réseau

**Tracer la route** :
```bash
# Chemin vers une destination
traceroute google.com

# Avec ICMP (Linux)
tracepath google.com

# Version Windows
tracert google.com
```

### netstat/ss : connexions réseau

**Connexions actives** :
```bash
# Toutes les connexions (ancienne commande)
netstat -tuln

# Version moderne (recommandée)
ss -tuln

# Connexions avec processus
ss -tulnp
```

**Ports en écoute** :
```bash
# Services en écoute
ss -ln | grep LISTEN

# Par programme
ss -lntp | grep ssh
```

## Configuration réseau

### Configuration statique

**Configuration manuelle** :
```bash
# Interface Ethernet
sudo ip addr add 192.168.1.100/24 dev eth0

# Passerelle par défaut
sudo ip route add default via 192.168.1.1

# Activer l'interface
sudo ip link set eth0 up
```

### Configuration automatique (DHCP)

**Client DHCP** :
```bash
# Demander une adresse
sudo dhclient eth0

# Libérer l'adresse
sudo dhclient -r eth0
```

### Configuration persistante

**Debian/Ubuntu (/etc/network/interfaces)** :
```bash
# Configuration statique
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
```

**Systemd (iproute2)** :
```bash
# Configuration temporaire (persiste jusqu'au reboot)
# Les configurations sont dans /etc/systemd/network/
```

## Dépannage réseau

### Diagnostic étape par étape

**1. Vérifier la configuration locale** :
```bash
# Adresse IP
ip addr show

# Route par défaut
ip route show

# Serveurs DNS
cat /etc/resolv.conf
```

**2. Tester la connectivité locale** :
```bash
# Loopback
ping 127.0.0.1

# Passerelle
ping 192.168.1.1

# Autre machine locale
ping 192.168.1.101
```

**3. Tester la connectivité distante** :
```bash
# DNS
nslookup google.com

# Connectivité Internet
ping 8.8.8.8

# Route complète
traceroute google.com
```

### Outils de diagnostic avancés

**tcpdump : analyseur de paquets** :
```bash
# Capturer le trafic sur une interface
sudo tcpdump -i eth0

# Filtrer par port
sudo tcpdump -i eth0 port 80

# Sauvegarder la capture
sudo tcpdump -i eth0 -w capture.pcap
```

**Wireshark** : Interface graphique pour tcpdump

**nmap : scanner de ports** :
```bash
# Scanner un hôte
nmap 192.168.1.100

# Scanner un réseau
nmap 192.168.1.0/24

# Détection de service
nmap -sV 192.168.1.100
```

## Sécurité réseau de base

### Règles de base

**Principe du moindre privilège** :
- Ouvrir seulement les ports nécessaires
- Utiliser le chiffrement (SSH au lieu de Telnet)
- Mettre à jour régulièrement

**Firewall de base** :
```bash
# UFW (Ubuntu/Debian)
sudo ufw enable
sudo ufw allow ssh
sudo ufw allow 80/tcp

# iptables (avancé)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -j DROP
```

### Bonnes pratiques

**SSH sécurisé** :
```bash
# Désactiver root login
# /etc/ssh/sshd_config
PermitRootLogin no

# Utiliser des clés au lieu des mots de passe
ssh-keygen -t ed25519
ssh-copy-id user@remote-server
```

**Monitoring** :
```bash
# Connexions SSH actives
who
w

# Tentatives de connexion
sudo journalctl -u ssh | grep "Failed"
```

## Conclusion

Le réseau constitue le fondement de l'informatique moderne, permettant la communication et le partage de ressources à toutes les échelles. Comprendre ses concepts fondamentaux - des adresses IP aux protocoles de transport, en passant par le DNS et les interfaces - est essentiel pour tout professionnel de l'informatique.

Dans les chapitres suivants, nous explorerons les redirections et pipelines, qui permettent de combiner les commandes réseau avec les outils locaux pour créer des workflows d'automatisation puissants.
