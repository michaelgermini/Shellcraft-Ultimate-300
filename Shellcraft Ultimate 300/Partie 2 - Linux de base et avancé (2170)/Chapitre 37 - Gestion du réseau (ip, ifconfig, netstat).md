# Chapitre 37 - Gestion du réseau (ip, ifconfig, netstat)

## Table des matières
- [Introduction](#introduction)
- [Architecture réseau Linux](#architecture-réseau-linux)
- [Commandes réseau modernes : ip](#commandes-réseau-modernes-ip)
- [ifconfig : commande traditionnelle](#ifconfig--commande-traditionnelle)
- [Configuration des interfaces](#configuration-des-interfaces)
- [Routage avancé](#routage-avancé)
- [Résolution de noms et DNS](#résolution-de-noms-et-dns)
- [netstat et ss : connexions réseau](#netstat-et-ss--connexions-réseau)
- [Outils de diagnostic réseau](#outils-de-diagnostic-réseau)
- [Monitoring du trafic](#monitoring-du-trafic)
- [Configuration réseau persistante](#configuration-réseau-persistante)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La gestion du réseau constitue l'un des aspects les plus complexes de l'administration système moderne. Des commandes traditionnelles comme ifconfig aux outils modernes comme ip, comprendre la configuration réseau permet de diagnostiquer les problèmes, optimiser les performances, et assurer la connectivité fiable.

Imaginez le réseau comme un système circulatoire sophistiqué : les interfaces sont les artères principales, les routes déterminent le flux du trafic, et les outils de diagnostic permettent de détecter les obstructions ou les anomalies dans ce système vital. Dans un système Linux moderne, cette gestion s'étend de la configuration basique des interfaces aux routages complexes multi-homed, en passant par le monitoring avancé et l'automatisation complète.

## Architecture réseau Linux

### Interfaces réseau

**Types d'interfaces** :
```bash
#!/bin/bash
# Types d'interfaces réseau

# Interfaces physiques
# eth0, eth1 : Ethernet (ancienne nomenclature)
# enp0s3, eno1 : Ethernet (nouvelle nomenclature basée sur emplacement)
# wlan0, wlp2s0 : WiFi

# Interfaces virtuelles
# lo : Loopback (127.0.0.1)
# tun0, tap0 : Tunnels VPN
# br0 : Bridge
# veth0 : Virtual Ethernet pair
# docker0 : Interface Docker

# Lister toutes les interfaces
ip link show
# ou
ifconfig -a
```

**Nomenclature moderne** :
```bash
#!/bin/bash
# Nomenclature systemd (predictable network interface names)

# Format : <type><bus_id><port>
# en : Ethernet
# wl : WiFi
# ww : WWAN

# Exemples :
# enp0s3 : Ethernet, bus PCI 0, slot 3
# wlp2s0 : WiFi, bus PCI 2, slot 0
# eno1 : Ethernet onboard 1
# ens33 : Ethernet network slot 33

# Voir la correspondance
ip link show
ls -l /sys/class/net/
```

## Commandes réseau modernes : ip

### ip : commande unifiée

**ip est la commande moderne** qui remplace ifconfig, route, arp, etc.

**Structure générale** :
```bash
ip [OPTIONS] OBJECT {COMMAND | help}
```

**Objets principaux** :
- `link` : Interfaces réseau
- `addr` : Adresses IP
- `route` : Table de routage
- `neigh` : Table ARP/NDISC
- `rule` : Règles de routage
- `maddr` : Adresses multicast
- `tunnel` : Tunnels IP

### ip link : gestion des interfaces

**Commandes ip link** :
```bash
#!/bin/bash
# Gestion des interfaces avec ip link

# Lister toutes les interfaces
ip link show
ip link
ip -s link  # Avec statistiques

# Informations sur une interface spécifique
ip link show eth0
ip -s link show eth0  # Statistiques détaillées

# Activer une interface
sudo ip link set eth0 up

# Désactiver une interface
sudo ip link set eth0 down

# Changer l'état
sudo ip link set eth0 up
sudo ip link set eth0 down

# Renommer une interface
sudo ip link set eth0 name myeth0

# Changer l'adresse MAC
sudo ip link set eth0 address 00:11:22:33:44:55

# Changer le MTU
sudo ip link set eth0 mtu 1500
sudo ip link set eth0 mtu 9000  # Jumbo frames

# Statistiques détaillées
ip -s -s link show eth0  # Double -s pour plus de détails
```

**Scripts avec ip link** :
```bash
#!/bin/bash
# Scripts utilitaires avec ip link

# Lister les interfaces actives
list_active_interfaces() {
    ip link show | grep -E "^[0-9]+:" | grep -v "lo:" | \
    awk -F': ' '{print $2}' | while read iface; do
        if ip link show "$iface" | grep -q "state UP"; then
            echo "✓ $iface : ACTIVE"
        else
            echo "✗ $iface : INACTIVE"
        fi
    done
}

# Statistiques complètes
interface_stats() {
    local iface="${1:-}"
    
    if [ -z "$iface" ]; then
        ip -s link show
    else
        ip -s -s link show "$iface"
    fi
}
```

### ip addr : gestion des adresses IP

**Commandes ip addr** :
```bash
#!/bin/bash
# Gestion des adresses IP avec ip addr

# Lister toutes les adresses
ip addr show
ip addr
ip a  # Version courte

# Adresses d'une interface spécifique
ip addr show eth0
ip a show eth0

# Ajouter une adresse IP
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip addr add 192.168.1.100/24 broadcast 192.168.1.255 dev eth0

# Ajouter plusieurs adresses sur une interface
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip addr add 192.168.1.101/24 dev eth0

# Supprimer une adresse IP
sudo ip addr del 192.168.1.100/24 dev eth0

# Vider toutes les adresses d'une interface
sudo ip addr flush dev eth0

# Vider seulement les adresses IPv4
sudo ip -4 addr flush dev eth0

# Vider seulement les adresses IPv6
sudo ip -6 addr flush dev eth0

# Adresse avec label (alias)
sudo ip addr add 192.168.1.100/24 dev eth0 label eth0:0
```

**Scripts avec ip addr** :
```bash
#!/bin/bash
# Scripts avec ip addr

# Lister toutes les adresses IP
list_all_ips() {
    ip -4 addr show | grep -E "inet " | awk '{print $2}' | cut -d'/' -f1
}

# Adresse IP d'une interface
get_interface_ip() {
    local iface="$1"
    ip -4 addr show "$iface" | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -1
}

# Vérifier si une interface a une adresse IP
has_ip() {
    local iface="$1"
    ip -4 addr show "$iface" | grep -q "inet "
}

# Ajouter une adresse IP si elle n'existe pas
add_ip_if_not_exists() {
    local ip="$1"
    local iface="$2"
    
    if ! ip addr show "$iface" | grep -q "$ip"; then
        sudo ip addr add "$ip" dev "$iface"
        echo "Adresse $ip ajoutée à $iface"
    else
        echo "Adresse $ip existe déjà sur $iface"
    fi
}
```

### ip route : gestion du routage

**Commandes ip route** :
```bash
#!/bin/bash
# Gestion du routage avec ip route

# Afficher la table de routage
ip route show
ip route
ip r  # Version courte

# Route par défaut
ip route show default
ip r show default

# Routes vers un réseau spécifique
ip route show 192.168.1.0/24

# Ajouter une route
sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth0

# Route par défaut
sudo ip route add default via 192.168.1.1 dev eth0

# Route avec métrique
sudo ip route add 192.168.2.0/24 via 192.168.1.1 dev eth0 metric 100

# Route avec table de routage spécifique
sudo ip route add 10.0.0.0/8 via 192.168.1.1 table 100

# Supprimer une route
sudo ip route del 192.168.2.0/24

# Supprimer la route par défaut
sudo ip route del default

# Vider toutes les routes
sudo ip route flush table main

# Routes pour une interface
ip route show dev eth0

# Route vers une destination spécifique
ip route get 8.8.8.8
```

**Routage avancé** :
```bash
#!/bin/bash
# Routage avancé

# Tables de routage multiples
# Table main (par défaut)
ip route show table main

# Table local (routes locales)
ip route show table local

# Créer une table de routage personnalisée
echo "100 mytable" | sudo tee -a /etc/iproute2/rt_tables

# Utiliser une table personnalisée
sudo ip route add 10.0.0.0/8 via 192.168.1.1 table mytable

# Routes avec source spécifique
sudo ip route add 192.168.2.0/24 via 192.168.1.1 src 192.168.1.100

# Routes avec scope
sudo ip route add 192.168.1.0/24 dev eth0 scope link
```

### ip neigh : table ARP

**Gestion de la table ARP** :
```bash
#!/bin/bash
# Gestion ARP avec ip neigh

# Afficher la table ARP
ip neigh show
ip neigh
ip n  # Version courte

# Table ARP pour une interface
ip neigh show dev eth0

# Ajouter une entrée ARP statique
sudo ip neigh add 192.168.1.100 lladdr 00:11:22:33:44:55 dev eth0

# Supprimer une entrée ARP
sudo ip neigh del 192.168.1.100 dev eth0

# Vider la table ARP
sudo ip neigh flush dev eth0

# Vérifier un voisin
ip neigh get 192.168.1.1 dev eth0
```

## ifconfig : commande traditionnelle

### Utilisation de base

**ifconfig est déprécié mais encore utilisé** :
```bash
#!/bin/bash
# Utilisation de ifconfig

# Lister toutes les interfaces
ifconfig -a

# Informations sur une interface
ifconfig eth0

# Activer une interface
sudo ifconfig eth0 up

# Désactiver une interface
sudo ifconfig eth0 down

# Configurer une adresse IP
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0

# Avec broadcast
sudo ifconfig eth0 192.168.1.100 netmask 255.255.255.0 broadcast 192.168.1.255

# Changer le MTU
sudo ifconfig eth0 mtu 1500

# Interface avec alias
sudo ifconfig eth0:0 192.168.1.101 netmask 255.255.255.0

# Supprimer un alias
sudo ifconfig eth0:0 down
```

**Comparaison ip vs ifconfig** :
```bash
#!/bin/bash
# Comparaison des commandes

# Lister les interfaces
ip link show          # Moderne
ifconfig -a           # Traditionnel

# Adresses IP
ip addr show          # Moderne
ifconfig              # Traditionnel

# Activer/désactiver
ip link set eth0 up   # Moderne
ifconfig eth0 up      # Traditionnel

# Adresse IP
ip addr add 192.168.1.100/24 dev eth0  # Moderne
ifconfig eth0 192.168.1.100 netmask 255.255.255.0  # Traditionnel
```

## Configuration des interfaces

### Configuration statique

**Configuration manuelle complète** :
```bash
#!/bin/bash
# Configuration statique complète

configure_static_interface() {
    local iface="$1"
    local ip="$2"
    local netmask="$3"
    local gateway="$4"
    
    # Calculer le préfixe CIDR
    local prefix=$(ipcalc -p "$ip" "$netmask" 2>/dev/null || echo "24")
    
    # Désactiver l'interface
    sudo ip link set "$iface" down
    
    # Ajouter l'adresse IP
    sudo ip addr add "$ip/$prefix" dev "$iface"
    
    # Activer l'interface
    sudo ip link set "$iface" up
    
    # Ajouter la route par défaut
    if [ -n "$gateway" ]; then
        sudo ip route add default via "$gateway" dev "$iface"
    fi
    
    echo "Interface $iface configurée : $ip/$prefix"
    if [ -n "$gateway" ]; then
        echo "Passerelle : $gateway"
    fi
}

# Utilisation
# configure_static_interface eth0 192.168.1.100 255.255.255.0 192.168.1.1
```

### Configuration DHCP

**Configuration DHCP** :
```bash
#!/bin/bash
# Configuration DHCP

# Avec dhclient
configure_dhcp() {
    local iface="$1"
    
    # Libérer l'adresse actuelle
    sudo dhclient -r "$iface" 2>/dev/null
    
    # Demander une nouvelle adresse
    sudo dhclient "$iface"
    
    echo "Configuration DHCP demandée pour $iface"
}

# Avec systemd-networkd
# Créer /etc/systemd/network/20-wired.network
# [Match]
# Name=eth0
# 
# [Network]
# DHCP=yes

# Avec NetworkManager
# nmcli connection modify "Wired connection 1" ipv4.method auto
```

### Configuration IPv6

**Configuration IPv6** :
```bash
#!/bin/bash
# Configuration IPv6

# Adresse IPv6 statique
sudo ip -6 addr add 2001:db8::1/64 dev eth0

# Activer IPv6 sur une interface
sudo sysctl -w net.ipv6.conf.eth0.disable_ipv6=0

# Route IPv6 par défaut
sudo ip -6 route add default via 2001:db8::1 dev eth0

# Désactiver IPv6 temporairement
sudo sysctl -w net.ipv6.conf.all.disable_ipv6=1

# Vérifier les adresses IPv6
ip -6 addr show
```

## Routage avancé

### Routage multi-homed

**Configuration multi-homed** :
```bash
#!/bin/bash
# Configuration multi-homed

# Système avec plusieurs interfaces réseau
# eth0 : 192.168.1.0/24 (réseau interne)
# eth1 : 10.0.0.0/24 (réseau DMZ)

configure_multihomed() {
    # Interface 1
    sudo ip addr add 192.168.1.100/24 dev eth0
    sudo ip link set eth0 up
    sudo ip route add 192.168.1.0/24 dev eth0
    
    # Interface 2
    sudo ip addr add 10.0.0.100/24 dev eth1
    sudo ip link set eth1 up
    sudo ip route add 10.0.0.0/24 dev eth1
    
    # Route par défaut via eth0
    sudo ip route add default via 192.168.1.1 dev eth0
    
    # Route spécifique via eth1
    sudo ip route add 10.0.0.0/8 via 10.0.0.1 dev eth1
}
```

### Policy-based routing

**Routage basé sur des politiques** :
```bash
#!/bin/bash
# Policy-based routing

# Créer une table de routage
echo "100 custom" | sudo tee -a /etc/iproute2/rt_tables

# Ajouter des routes dans la table personnalisée
sudo ip route add 192.168.2.0/24 via 192.168.1.1 table custom

# Créer une règle de routage
sudo ip rule add from 192.168.1.100 table custom
sudo ip rule add fwmark 1 table custom

# Lister les règles
ip rule show

# Supprimer une règle
sudo ip rule del from 192.168.1.100 table custom
```

## Résolution de noms et DNS

### Configuration DNS

**Fichier /etc/resolv.conf** :
```bash
#!/bin/bash
# Configuration DNS

# Fichier /etc/resolv.conf (peut être géré par systemd-resolved)
cat /etc/resolv.conf

# Configuration manuelle
configure_dns() {
    local dns1="$1"
    local dns2="${2:-}"
    
    sudo tee /etc/resolv.conf << EOF
nameserver $dns1
${dns2:+nameserver $dns2}
options timeout:2
options attempts:3
EOF
    
    echo "DNS configuré : $dns1 ${dns2:+$dns2}"
}

# Avec systemd-resolved
# /etc/systemd/resolved.conf
# [Resolve]
# DNS=8.8.8.8 1.1.1.1
```

**Commandes DNS** :
```bash
#!/bin/bash
# Commandes DNS

# Résolution avec host
host google.com
host 8.8.8.8

# Résolution avec nslookup
nslookup google.com
nslookup 8.8.8.8

# Résolution avec dig (plus détaillé)
dig google.com
dig google.com +short
dig @8.8.8.8 google.com
dig -x 8.8.8.8  # Reverse DNS

# Résolution avec getent
getent hosts google.com

# Tester la résolution DNS
test_dns() {
    local domain="$1"
    
    if host "$domain" >/dev/null 2>&1; then
        echo "✓ DNS fonctionne pour $domain"
        host "$domain" | head -1
    else
        echo "✗ DNS échoue pour $domain"
    fi
}
```

## netstat et ss : connexions réseau

### netstat : commande traditionnelle

**Utilisation de netstat** :
```bash
#!/bin/bash
# Utilisation de netstat

# Toutes les connexions
netstat -a

# Connexions TCP
netstat -t

# Connexions UDP
netstat -u

# Ports en écoute
netstat -l

# Avec numéros de ports
netstat -n

# Avec processus
netstat -p

# Combinaisons
netstat -tuln  # TCP/UDP, écoute, numérique
netstat -tulnp # Avec processus

# Statistiques par protocole
netstat -s

# Statistiques TCP
netstat -st

# Statistiques UDP
netstat -su

# Connexions établies
netstat -tn | grep ESTABLISHED

# Connexions en TIME_WAIT
netstat -tn | grep TIME_WAIT
```

### ss : remplacement moderne

**ss est plus rapide que netstat** :
```bash
#!/bin/bash
# Utilisation de ss

# Toutes les connexions
ss -a

# Connexions TCP
ss -t

# Connexions UDP
ss -u

# Ports en écoute
ss -l

# Avec numéros
ss -n

# Avec processus
ss -p

# Combinaisons
ss -tuln
ss -tulnp

# Connexions établies
ss -tn state established

# États TCP
ss -tn state listening
ss -tn state established
ss -tn state time-wait
ss -tn state close-wait

# Connexions d'un processus
ss -p | grep apache

# Connexions sur un port spécifique
ss -tlnp sport = :80
ss -tlnp dport = :443

# Statistiques
ss -s

# Connexions depuis/vers une IP
ss -tn dst 192.168.1.1
ss -tn src 192.168.1.100

# Filtrer par interface
ss -tn dev eth0
```

**Scripts avec ss** :
```bash
#!/bin/bash
# Scripts avec ss

# Lister les ports en écoute
list_listening_ports() {
    ss -tuln | grep LISTEN | awk '{print $4}' | cut -d':' -f2 | sort -n | uniq
}

# Processus utilisant un port
port_usage() {
    local port="$1"
    ss -tlnp | grep ":$port " | awk '{print $6}' | cut -d',' -f2 | cut -d'=' -f2
}

# Connexions actives par processus
connections_by_process() {
    ss -tnp | grep ESTAB | awk '{print $6}' | \
    sed 's/users:((\(.*\)))/\1/' | sort | uniq -c | sort -rn
}

# Statistiques de connexions
connection_stats() {
    echo "=== Statistiques de connexions ==="
    ss -s
    echo ""
    echo "=== Connexions établies ==="
    ss -tn state established | wc -l
    echo ""
    echo "=== Ports en écoute ==="
    ss -tln | grep LISTEN | wc -l
}
```

## Outils de diagnostic réseau

### ping : test de connectivité

**Utilisation de ping** :
```bash
#!/bin/bash
# Utilisation de ping

# Test simple
ping google.com

# Nombre limité de paquets
ping -c 4 google.com

# Intervalle entre paquets
ping -i 2 google.com

# Taille de paquet
ping -s 1000 google.com

# Timeout
ping -W 5 google.com

# Flood (root seulement)
sudo ping -f localhost

# Avec timestamp
ping -D google.com

# IPv6
ping6 ipv6.google.com
```

### traceroute : chemin réseau

**Utilisation de traceroute** :
```bash
#!/bin/bash
# Utilisation de traceroute

# Chemin vers une destination
traceroute google.com

# Avec nombre max de sauts
traceroute -m 30 google.com

# Version Linux (tracepath)
tracepath google.com

# Avec UDP (défaut)
traceroute -U google.com

# Avec ICMP
traceroute -I google.com

# Avec TCP
traceroute -T google.com

# IPv6
traceroute6 ipv6.google.com
```

### tcpdump : capture de paquets

**Capture de paquets** :
```bash
#!/bin/bash
# Utilisation de tcpdump

# Capturer sur une interface
sudo tcpdump -i eth0

# Capturer avec filtres
sudo tcpdump -i eth0 host 192.168.1.1
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 tcp port 22

# Sauvegarder dans un fichier
sudo tcpdump -i eth0 -w capture.pcap

# Lire depuis un fichier
tcpdump -r capture.pcap

# Nombre limité de paquets
sudo tcpdump -i eth0 -c 100

# Format verbose
sudo tcpdump -i eth0 -v
sudo tcpdump -i eth0 -vv
sudo tcpdump -i eth0 -vvv

# Sans résolution DNS
sudo tcpdump -i eth0 -n

# Afficher les données
sudo tcpdump -i eth0 -A  # ASCII
sudo tcpdump -i eth0 -X  # Hex + ASCII
```

## Monitoring du trafic

### iftop : monitoring en temps réel

**Utilisation de iftop** :
```bash
#!/bin/bash
# Utilisation de iftop

# Monitoring sur une interface
sudo iftop -i eth0

# Sans résolution DNS
sudo iftop -i eth0 -n

# Afficher les ports
sudo iftop -i eth0 -P

# Filtrer par réseau
sudo iftop -i eth0 -f "net 192.168.1.0/24"
```

### nethogs : trafic par processus

**Utilisation de nethogs** :
```bash
#!/bin/bash
# Utilisation de nethogs

# Trafic par processus
sudo nethogs

# Sur une interface spécifique
sudo nethogs eth0

# Rafraîchissement personnalisé
sudo nethogs -d 5  # Toutes les 5 secondes
```

### vnstat : statistiques réseau

**Utilisation de vnstat** :
```bash
#!/bin/bash
# Utilisation de vnstat

# Statistiques actuelles
vnstat -i eth0

# Statistiques quotidiennes
vnstat -d

# Statistiques mensuelles
vnstat -m

# Statistiques horaires
vnstat -h

# Top jours
vnstat -t
```

## Configuration réseau persistante

### Debian/Ubuntu : /etc/network/interfaces

**Configuration traditionnelle** :
```bash
#!/bin/bash
# Configuration /etc/network/interfaces

# Exemple de configuration statique
cat > /tmp/interfaces.example << 'EOF'
# Interface loopback
auto lo
iface lo inet loopback

# Interface Ethernet statique
auto eth0
iface eth0 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8 1.1.1.1
    dns-search example.com

# Interface avec DHCP
auto eth1
iface eth1 inet dhcp

# Interface avec plusieurs adresses
auto eth0:0
iface eth0:0 inet static
    address 192.168.1.101
    netmask 255.255.255.0
EOF

# Appliquer la configuration
# sudo cp /tmp/interfaces.example /etc/network/interfaces
# sudo systemctl restart networking
```

### Systemd-networkd

**Configuration systemd-networkd** :
```bash
#!/bin/bash
# Configuration systemd-networkd

# Fichier : /etc/systemd/network/20-wired.network
cat > /tmp/20-wired.network << 'EOF'
[Match]
Name=eth0

[Network]
DHCP=no
Address=192.168.1.100/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=1.1.1.1
Domains=example.com
EOF

# Activer systemd-networkd
# sudo systemctl enable systemd-networkd
# sudo systemctl start systemd-networkd
```

### NetworkManager

**Configuration avec nmcli** :
```bash
#!/bin/bash
# Configuration NetworkManager

# Lister les connexions
nmcli connection show

# Créer une connexion statique
nmcli connection add type ethernet con-name "Static-eth0" \
    ifname eth0 ipv4.addresses 192.168.1.100/24 \
    ipv4.gateway 192.168.1.1 ipv4.dns "8.8.8.8 1.1.1.1" \
    ipv4.method manual

# Modifier une connexion
nmcli connection modify "Static-eth0" ipv4.addresses 192.168.1.101/24

# Activer une connexion
nmcli connection up "Static-eth0"

# Désactiver une connexion
nmcli connection down "Static-eth0"

# Activer DHCP
nmcli connection modify "Static-eth0" ipv4.method auto
```

## Scripts d'automatisation

### Script complet de gestion réseau

**Gestionnaire réseau complet** :
```bash
#!/bin/bash
# Gestionnaire réseau complet

set -euo pipefail

network_manager() {
    local action="$1"
    shift
    
    case "$action" in
        status)
            show_network_status
            ;;
        config)
            configure_interface "$@"
            ;;
        route)
            manage_route "$@"
            ;;
        dns)
            configure_dns "$@"
            ;;
        test)
            test_connectivity "$@"
            ;;
        *)
            echo "Usage: network_manager {status|config|route|dns|test}"
            exit 1
            ;;
    esac
}

show_network_status() {
    echo "=== Interfaces réseau ==="
    ip -4 addr show | grep -E "^[0-9]+:|inet " | \
    awk 'NR%2==1 {iface=$2; gsub(/:/,"",iface)} NR%2==0 {print iface ": " $2}'
    
    echo ""
    echo "=== Routes ==="
    ip route show
    
    echo ""
    echo "=== DNS ==="
    cat /etc/resolv.conf | grep nameserver
    
    echo ""
    echo "=== Connexions actives ==="
    ss -tn state established | wc -l
    echo "connexions établies"
}

configure_interface() {
    local iface="$1"
    local ip="$2"
    local gateway="${3:-}"
    
    sudo ip addr flush dev "$iface"
    sudo ip addr add "$ip" dev "$iface"
    sudo ip link set "$iface" up
    
    if [ -n "$gateway" ]; then
        sudo ip route del default 2>/dev/null || true
        sudo ip route add default via "$gateway" dev "$iface"
    fi
    
    echo "Interface $iface configurée : $ip"
}

manage_route() {
    local action="$1"
    shift
    
    case "$action" in
        add)
            sudo ip route add "$@"
            ;;
        del)
            sudo ip route del "$@"
            ;;
        show)
            ip route show "$@"
            ;;
        *)
            echo "Usage: network_manager route {add|del|show}"
            ;;
    esac
}

configure_dns() {
    local dns1="$1"
    local dns2="${2:-}"
    
    sudo tee /etc/resolv.conf << EOF
nameserver $dns1
${dns2:+nameserver $dns2}
EOF
    
    echo "DNS configuré"
}

test_connectivity() {
    local target="${1:-8.8.8.8}"
    
    if ping -c 1 -W 2 "$target" >/dev/null 2>&1; then
        echo "✓ Connectivité OK vers $target"
    else
        echo "✗ Pas de connectivité vers $target"
    fi
}

# Utilisation
# network_manager status
# network_manager config eth0 192.168.1.100/24 192.168.1.1
# network_manager route show
# network_manager dns 8.8.8.8 1.1.1.1
# network_manager test google.com
```

## Conclusion

La gestion du réseau Linux avec les commandes modernes `ip` et les outils traditionnels comme `ifconfig` et `netstat` est essentielle pour tout administrateur système. En maîtrisant ces outils, vous pouvez configurer des interfaces complexes, gérer le routage avancé, diagnostiquer les problèmes réseau, et automatiser la configuration réseau de manière professionnelle.

Un système bien configuré utilise les outils appropriés pour chaque tâche : `ip` pour la configuration moderne, `ss` pour le monitoring des connexions, et les outils de diagnostic pour résoudre rapidement les problèmes. La compréhension approfondie de ces outils permet de créer des configurations réseau robustes, performantes et maintenables.

Dans le chapitre suivant, nous explorerons la surveillance du trafic réseau avec des outils comme tcpdump, wireshark, et les techniques avancées d'analyse de paquets pour comprendre et optimiser le trafic réseau.
