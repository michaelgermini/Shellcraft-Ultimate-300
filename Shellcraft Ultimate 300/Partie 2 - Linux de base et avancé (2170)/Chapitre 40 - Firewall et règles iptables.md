# Chapitre 40 - Firewall et règles iptables

## Table des matières
- [Introduction](#introduction)
- [Architecture de Netfilter/iptables](#architecture-de-netfilteriptables)
- [Tables, chaînes et politiques](#tables-chaînes-et-politiques)
- [Syntaxe des règles iptables](#syntaxe-des-règles-iptables)
- [Règles de filtrage de base](#règles-de-filtrage-de-base)
- [Filtrage avancé](#filtrage-avancé)
- [NAT et translation d'adresses](#nat-et-translation-dadresses)
- [Port forwarding et redirection](#port-forwarding-et-redirection)
- [Chaînes personnalisées](#chaînes-personnalisées)
- [Configuration persistante](#configuration-persistante)
- [Dépannage et audit](#dépannage-et-audit)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Iptables constitue le pare-feu historique de Linux, offrant un contrôle granulaire sur le trafic réseau entrant et sortant. Au-delà d'un simple bloqueur de ports, c'est un langage de règles sophistiqué permettant de construire des politiques de sécurité complexes et adaptatives.

Imaginez iptables comme un douanier expérimenté à la frontière : il inspecte chaque véhicule (paquet), vérifie les papiers (en-têtes), applique les réglementations (règles), et décide de laisser passer, refuser, ou rediriger le trafic selon des critères sophistiqués. Dans un système Linux moderne, iptables reste l'outil de référence pour la configuration avancée des pare-feu, même si des outils comme nftables le remplacent progressivement.

## Architecture de Netfilter/iptables

### Netfilter : framework du kernel

**Netfilter** :
- Framework du kernel Linux pour le filtrage de paquets
- Hooks dans le stack réseau du kernel
- Points d'interception : PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING

**Iptables** :
- Interface utilisateur pour Netfilter
- Gère les règles de filtrage
- Tables : filter, nat, mangle, raw, security

**Flux des paquets** :
```bash
#!/bin/bash
# Flux des paquets dans Netfilter

# Paquet entrant depuis le réseau
#   ↓
# PREROUTING (raw, mangle, nat)
#   ↓
# Routing decision
#   ↓
# ┌─────────────┬──────────────┐
# │   INPUT     │   FORWARD    │
# │  (filter)   │   (filter)   │
# └─────────────┴──────────────┘
#   ↓              ↓
# POSTROUTING (mangle, nat)
#   ↓
# Paquet sortant vers le réseau
```

## Tables, chaînes et politiques

### Tables iptables

**Tables principales** :
```bash
#!/bin/bash
# Tables iptables

# filter : Filtrage de base (défaut)
# - INPUT : Paquets destinés au système local
# - FORWARD : Paquets routés à travers le système
# - OUTPUT : Paquets générés par le système local

# nat : Translation d'adresses
# - PREROUTING : Avant le routage
# - OUTPUT : Paquets générés localement
# - POSTROUTING : Après le routage

# mangle : Modification des paquets
# - PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING

# raw : Traitement avant le suivi de connexion
# - PREROUTING, OUTPUT

# security : Mandatory Access Control (SELinux)
# - INPUT, OUTPUT, FORWARD
```

**Afficher les tables** :
```bash
#!/bin/bash
# Afficher les tables et chaînes

# Table filter (défaut)
sudo iptables -L
sudo iptables -t filter -L

# Table nat
sudo iptables -t nat -L

# Table mangle
sudo iptables -t mangle -L

# Table raw
sudo iptables -t raw -L

# Avec numéros de ligne
sudo iptables -L -n --line-numbers

# Format verbose
sudo iptables -L -v

# Avec compteurs
sudo iptables -L -v -n
```

### Chaînes et politiques

**Politiques par défaut** :
```bash
#!/bin/bash
# Politiques par défaut

# Politique ACCEPT (tout autoriser)
sudo iptables -P INPUT ACCEPT
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD ACCEPT

# Politique DROP (tout bloquer)
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT DROP
sudo iptables -P FORWARD DROP

# Politique REJECT (bloquer avec message)
sudo iptables -P INPUT REJECT
sudo iptables -P OUTPUT REJECT
sudo iptables -P FORWARD REJECT

# Vérifier les politiques
sudo iptables -L | grep policy
```

## Syntaxe des règles iptables

### Structure d'une règle

**Format général** :
```bash
iptables [-t table] command [chain] [match] [target]
```

**Commandes principales** :
```bash
#!/bin/bash
# Commandes iptables

# Ajouter une règle
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Insérer une règle (au début)
sudo iptables -I INPUT 1 -p tcp --dport 80 -j ACCEPT

# Supprimer une règle
sudo iptables -D INPUT 1  # Par numéro
sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT  # Par règle

# Remplacer une règle
sudo iptables -R INPUT 1 -p tcp --dport 443 -j ACCEPT

# Vider une chaîne
sudo iptables -F INPUT

# Vider toutes les chaînes
sudo iptables -F

# Vider une table
sudo iptables -t nat -F

# Supprimer une chaîne personnalisée
sudo iptables -X CUSTOM_CHAIN

# Supprimer toutes les chaînes personnalisées
sudo iptables -X
```

## Règles de filtrage de base

### Filtrage par protocole

**Filtrage TCP/UDP/ICMP** :
```bash
#!/bin/bash
# Filtrage par protocole

# Autoriser TCP
sudo iptables -A INPUT -p tcp -j ACCEPT

# Autoriser UDP
sudo iptables -A INPUT -p udp -j ACCEPT

# Autoriser ICMP (ping)
sudo iptables -A INPUT -p icmp -j ACCEPT

# Autoriser ICMP spécifique
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
sudo iptables -A INPUT -p icmp --icmp-type echo-reply -j ACCEPT

# Refuser un protocole
sudo iptables -A INPUT -p tcp -j DROP
```

### Filtrage par port

**Ports TCP/UDP** :
```bash
#!/bin/bash
# Filtrage par port

# Autoriser un port spécifique
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Port source
sudo iptables -A OUTPUT -p tcp --sport 22 -j ACCEPT

# Plage de ports
sudo iptables -A INPUT -p tcp --dport 8000:8099 -j ACCEPT

# Plusieurs ports
sudo iptables -A INPUT -p tcp -m multiport --dports 22,80,443 -j ACCEPT

# Ports UDP
sudo iptables -A INPUT -p udp --dport 53 -j ACCEPT
```

### Filtrage par adresse IP

**Filtrage IP** :
```bash
#!/bin/bash
# Filtrage par adresse IP

# Autoriser depuis une IP
sudo iptables -A INPUT -s 192.168.1.100 -j ACCEPT

# Autoriser depuis un réseau
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# Refuser depuis une IP
sudo iptables -A INPUT -s 10.0.0.100 -j DROP

# Destination IP
sudo iptables -A OUTPUT -d 192.168.1.100 -j ACCEPT

# Exclure une IP
sudo iptables -A INPUT ! -s 192.168.1.100 -j ACCEPT
```

### Filtrage par interface

**Filtrage par interface** :
```bash
#!/bin/bash
# Filtrage par interface

# Paquets entrants sur une interface
sudo iptables -A INPUT -i eth0 -j ACCEPT

# Paquets sortants sur une interface
sudo iptables -A OUTPUT -o eth0 -j ACCEPT

# Interface loopback
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# Exclure une interface
sudo iptables -A INPUT ! -i eth0 -j DROP
```

## Filtrage avancé

### États de connexion

**Filtrage par état** :
```bash
#!/bin/bash
# Filtrage par état de connexion

# Module conntrack nécessaire
sudo modprobe nf_conntrack

# États possibles :
# NEW : Nouvelle connexion
# ESTABLISHED : Connexion établie
# RELATED : Connexion liée (FTP data, ICMP error)
# INVALID : Paquet invalide
# UNTRACKED : Non suivi

# Autoriser les connexions établies
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Autoriser les nouvelles connexions SSH
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT

# Autoriser les nouvelles connexions HTTP
sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT

# Refuser les paquets invalides
sudo iptables -A INPUT -m state --state INVALID -j DROP
```

### Limitation de connexions

**Limiter les connexions** :
```bash
#!/bin/bash
# Limitation de connexions

# Module limit
sudo iptables -A INPUT -p tcp --dport 22 -m limit --limit 3/min --limit-burst 5 -j ACCEPT

# Module connlimit
sudo iptables -A INPUT -p tcp --dport 22 -m connlimit --connlimit-above 3 -j REJECT

# Limiter les connexions par IP
sudo iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 10 --connlimit-mask 32 -j REJECT
```

### Filtrage par chaîne de caractères

**Filtrage par contenu** :
```bash
#!/bin/bash
# Filtrage par contenu

# Module string
sudo iptables -A INPUT -p tcp --dport 80 -m string --string "GET /admin" --algo bm -j DROP

# Recherche hexadécimale
sudo iptables -A INPUT -p tcp --dport 80 -m string --hex-string "|00 00 00 00|" --algo bm -j DROP
```

## NAT et translation d'adresses

### NAT Source (MASQUERADE)

**MASQUERADE** :
```bash
#!/bin/bash
# NAT Source avec MASQUERADE

# Masquer le trafic sortant
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Masquer depuis une interface spécifique
sudo iptables -t nat -A POSTROUTING -o eth0 -s 192.168.1.0/24 -j MASQUERADE

# SNAT (Source NAT) - alternative à MASQUERADE
sudo iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 203.0.113.1
```

### NAT Destination (DNAT)

**DNAT** :
```bash
#!/bin/bash
# NAT Destination

# Rediriger vers une IP interne
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:80

# Rediriger vers un port différent
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 192.168.1.100:80

# Depuis une IP spécifique
sudo iptables -t nat -A PREROUTING -s 192.168.1.0/24 -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100
```

## Port forwarding et redirection

### Redirection de port local

**REDIRECT** :
```bash
#!/bin/bash
# Redirection de port local

# Rediriger le port 8080 vers 80
sudo iptables -t nat -A PREROUTING -p tcp --dport 8080 -j REDIRECT --to-port 80

# Rediriger depuis OUTPUT (paquets générés localement)
sudo iptables -t nat -A OUTPUT -p tcp --dport 8080 -j REDIRECT --to-port 80
```

### Port forwarding complet

**Configuration complète** :
```bash
#!/bin/bash
# Port forwarding complet

setup_port_forwarding() {
    local external_port="$1"
    local internal_ip="$2"
    local internal_port="${3:-$external_port}"
    
    # DNAT : Rediriger le trafic entrant
    sudo iptables -t nat -A PREROUTING -p tcp --dport "$external_port" \
        -j DNAT --to-destination "$internal_ip:$internal_port"
    
    # Autoriser le forwarding
    sudo iptables -A FORWARD -p tcp -d "$internal_ip" --dport "$internal_port" -j ACCEPT
    
    # Autoriser le retour
    sudo iptables -A FORWARD -p tcp -s "$internal_ip" --sport "$internal_port" -j ACCEPT
    
    echo "Port forwarding configuré : $external_port -> $internal_ip:$internal_port"
}

# Exemple
# setup_port_forwarding 8080 192.168.1.100 80
```

## Chaînes personnalisées

### Création de chaînes

**Chaînes personnalisées** :
```bash
#!/bin/bash
# Création de chaînes personnalisées

# Créer une chaîne
sudo iptables -N CUSTOM_CHAIN

# Ajouter des règles à la chaîne
sudo iptables -A CUSTOM_CHAIN -p tcp --dport 22 -j ACCEPT
sudo iptables -A CUSTOM_CHAIN -p tcp --dport 80 -j ACCEPT

# Utiliser la chaîne depuis INPUT
sudo iptables -A INPUT -j CUSTOM_CHAIN

# Renommer une chaîne
sudo iptables -E OLD_CHAIN NEW_CHAIN

# Supprimer une chaîne (doit être vide)
sudo iptables -X CUSTOM_CHAIN

# Vider une chaîne avant de la supprimer
sudo iptables -F CUSTOM_CHAIN
sudo iptables -X CUSTOM_CHAIN
```

**Exemple de chaîne personnalisée** :
```bash
#!/bin/bash
# Exemple de chaîne personnalisée

setup_custom_chains() {
    # Créer les chaînes
    sudo iptables -N SSH_INPUT
    sudo iptables -N HTTP_INPUT
    sudo iptables -N BLOCKED
    
    # Règles SSH
    sudo iptables -A SSH_INPUT -p tcp --dport 22 -m state --state NEW -m limit --limit 3/min -j ACCEPT
    sudo iptables -A SSH_INPUT -p tcp --dport 22 -j DROP
    
    # Règles HTTP
    sudo iptables -A HTTP_INPUT -p tcp --dport 80 -j ACCEPT
    sudo iptables -A HTTP_INPUT -p tcp --dport 443 -j ACCEPT
    
    # Règles de blocage
    sudo iptables -A BLOCKED -j LOG --log-prefix "BLOCKED: "
    sudo iptables -A BLOCKED -j DROP
    
    # Utiliser les chaînes
    sudo iptables -A INPUT -p tcp --dport 22 -j SSH_INPUT
    sudo iptables -A INPUT -p tcp --dport 80 -j HTTP_INPUT
    sudo iptables -A INPUT -p tcp --dport 443 -j HTTP_INPUT
    sudo iptables -A INPUT -j BLOCKED
}
```

## Configuration persistante

### Sauvegarde et restauration

**Sauvegarder les règles** :
```bash
#!/bin/bash
# Sauvegarder les règles iptables

# Sauvegarder toutes les tables
sudo iptables-save > /etc/iptables/rules.v4

# Sauvegarder une table spécifique
sudo iptables-save -t nat > /etc/iptables/nat.rules

# Restaurer les règles
sudo iptables-restore < /etc/iptables/rules.v4

# Restaurer sans vider les règles existantes
sudo iptables-restore -n < /etc/iptables/rules.v4
```

**Script de sauvegarde automatique** :
```bash
#!/bin/bash
# Script de sauvegarde automatique

backup_iptables() {
    local backup_dir="/var/backups/iptables"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    mkdir -p "$backup_dir"
    
    # Sauvegarder toutes les tables
    sudo iptables-save > "$backup_dir/rules_$timestamp.v4"
    sudo ip6tables-save > "$backup_dir/rules_$timestamp.v6"
    
    # Garder seulement les 10 dernières sauvegardes
    ls -t "$backup_dir"/rules_*.v4 | tail -n +11 | xargs rm -f
    ls -t "$backup_dir"/rules_*.v6 | tail -n +11 | xargs rm -f
    
    echo "Règles sauvegardées dans $backup_dir"
}
```

### Persistance au démarrage

**Configuration Debian/Ubuntu** :
```bash
#!/bin/bash
# Configuration persistante Debian/Ubuntu

# Installer iptables-persistent
sudo apt install iptables-persistent

# Sauvegarder les règles actuelles
sudo netfilter-persistent save

# Les règles seront restaurées au démarrage
# Fichiers : /etc/iptables/rules.v4 et /etc/iptables/rules.v6
```

**Configuration systemd** :
```bash
#!/bin/bash
# Service systemd pour iptables

# Créer le service
sudo tee /etc/systemd/system/iptables-restore.service << 'EOF'
[Unit]
Description=Restore iptables rules
Before=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
ExecStart=/sbin/iptables-restore /etc/iptables/rules.v4
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

# Activer
sudo systemctl enable iptables-restore.service
```

## Dépannage et audit

### Vérification des règles

**Commandes de vérification** :
```bash
#!/bin/bash
# Vérification des règles

# Lister toutes les règles avec compteurs
sudo iptables -L -v -n --line-numbers

# Statistiques par chaîne
sudo iptables -L -v -x

# Vérifier les règles NAT
sudo iptables -t nat -L -v -n

# Vérifier les règles mangle
sudo iptables -t mangle -L -v -n

# Tester une règle
sudo iptables -C INPUT -p tcp --dport 22 -j ACCEPT
echo $?  # 0 si la règle existe
```

### Logging

**Activer le logging** :
```bash
#!/bin/bash
# Logging avec iptables

# Logger les paquets bloqués
sudo iptables -A INPUT -j LOG --log-prefix "INPUT DROP: " --log-level 4

# Logger avec limite
sudo iptables -A INPUT -j LOG --log-prefix "INPUT: " -m limit --limit 5/min

# Logger vers un fichier spécifique
sudo iptables -A INPUT -j LOG --log-prefix "IPTABLES: " --log-ip-options

# Voir les logs
sudo tail -f /var/log/kern.log
sudo dmesg | grep iptables
```

### Dépannage

**Scripts de dépannage** :
```bash
#!/bin/bash
# Scripts de dépannage

# Vérifier si un port est bloqué
check_port() {
    local port="$1"
    sudo iptables -L -n | grep "$port"
}

# Tracer le chemin d'un paquet
trace_packet() {
    local src_ip="$1"
    local dst_ip="$2"
    local dst_port="$3"
    
    echo "Vérification des règles pour $src_ip -> $dst_ip:$dst_port"
    
    # Vérifier INPUT
    sudo iptables -L INPUT -n -v | grep -E "(dpt|spt).*$dst_port"
    
    # Vérifier FORWARD
    sudo iptables -L FORWARD -n -v | grep -E "(dpt|spt).*$dst_port"
    
    # Vérifier NAT
    sudo iptables -t nat -L -n -v | grep "$dst_port"
}

# Réinitialiser iptables
reset_iptables() {
    # Sauvegarder d'abord
    sudo iptables-save > /tmp/iptables_backup_$(date +%Y%m%d_%H%M%S).rules
    
    # Vider toutes les règles
    sudo iptables -F
    sudo iptables -X
    sudo iptables -t nat -F
    sudo iptables -t mangle -F
    sudo iptables -t raw -F
    
    # Politiques par défaut
    sudo iptables -P INPUT ACCEPT
    sudo iptables -P OUTPUT ACCEPT
    sudo iptables -P FORWARD ACCEPT
    
    echo "Iptables réinitialisé"
}
```

## Scripts d'automatisation

### Script de configuration sécurisée

**Configuration complète** :
```bash
#!/bin/bash
# Configuration iptables sécurisée

set -euo pipefail

configure_secure_iptables() {
    # Sauvegarder les règles existantes
    sudo iptables-save > /tmp/iptables_backup_$(date +%Y%m%d_%H%M%S).rules
    
    # Vider toutes les règles
    sudo iptables -F
    sudo iptables -X
    sudo iptables -t nat -F
    sudo iptables -t mangle -F
    
    # Politiques par défaut
    sudo iptables -P INPUT DROP
    sudo iptables -P OUTPUT ACCEPT
    sudo iptables -P FORWARD DROP
    
    # Autoriser loopback
    sudo iptables -A INPUT -i lo -j ACCEPT
    sudo iptables -A OUTPUT -o lo -j ACCEPT
    
    # Autoriser les connexions établies
    sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    
    # Autoriser ICMP (ping)
    sudo iptables -A INPUT -p icmp --icmp-type echo-request -m limit --limit 1/s -j ACCEPT
    
    # SSH (limité)
    sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m limit --limit 3/min --limit-burst 5 -j ACCEPT
    
    # HTTP/HTTPS
    sudo iptables -A INPUT -p tcp --dport 80 -m state --state NEW -j ACCEPT
    sudo iptables -A INPUT -p tcp --dport 443 -m state --state NEW -j ACCEPT
    
    # Bloquer les paquets invalides
    sudo iptables -A INPUT -m state --state INVALID -j DROP
    
    # Logging des paquets bloqués
    sudo iptables -A INPUT -j LOG --log-prefix "IPTABLES DROP: " --log-level 4 -m limit --limit 5/min
    
    # Sauvegarder
    sudo mkdir -p /etc/iptables
    sudo iptables-save > /etc/iptables/rules.v4
    
    echo "Iptables configuré de manière sécurisée"
}

# Utilisation
# configure_secure_iptables
```

### Gestionnaire iptables

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire iptables

set -euo pipefail

iptables_manager() {
    local action="$1"
    shift
    
    case "$action" in
        status)
            show_status "$@"
            ;;
        allow)
            allow_rule "$@"
            ;;
        deny)
            deny_rule "$@"
            ;;
        nat)
            configure_nat "$@"
            ;;
        save)
            save_rules
            ;;
        restore)
            restore_rules "$@"
            ;;
        reset)
            reset_rules
            ;;
        *)
            echo "Usage: iptables_manager {status|allow|deny|nat|save|restore|reset}"
            exit 1
            ;;
    esac
}

show_status() {
    local table="${1:-filter}"
    sudo iptables -t "$table" -L -v -n --line-numbers
}

allow_rule() {
    local chain="$1"
    local rule="$2"
    
    sudo iptables -A "$chain" $rule -j ACCEPT
    echo "Règle ajoutée : $chain $rule ACCEPT"
}

deny_rule() {
    local chain="$1"
    local rule="$2"
    
    sudo iptables -A "$chain" $rule -j DROP
    echo "Règle ajoutée : $chain $rule DROP"
}

configure_nat() {
    local subaction="$1"
    shift
    
    case "$subaction" in
        masquerade)
            local interface="$1"
            sudo iptables -t nat -A POSTROUTING -o "$interface" -j MASQUERADE
            ;;
        dnat)
            local port="$1"
            local dest_ip="$2"
            local dest_port="${3:-$port}"
            sudo iptables -t nat -A PREROUTING -p tcp --dport "$port" \
                -j DNAT --to-destination "$dest_ip:$dest_port"
            ;;
        *)
            echo "Usage: iptables_manager nat {masquerade|dnat}"
            ;;
    esac
}

save_rules() {
    sudo mkdir -p /etc/iptables
    sudo iptables-save > /etc/iptables/rules.v4
    echo "Règles sauvegardées dans /etc/iptables/rules.v4"
}

restore_rules() {
    local file="${1:-/etc/iptables/rules.v4}"
    
    if [ -f "$file" ]; then
        sudo iptables-restore < "$file"
        echo "Règles restaurées depuis $file"
    else
        echo "Fichier non trouvé: $file"
        exit 1
    fi
}

reset_rules() {
    sudo iptables -F
    sudo iptables -X
    sudo iptables -t nat -F
    sudo iptables -t mangle -F
    sudo iptables -P INPUT ACCEPT
    sudo iptables -P OUTPUT ACCEPT
    sudo iptables -P FORWARD ACCEPT
    echo "Règles réinitialisées"
}

# Utilisation
# iptables_manager status
# iptables_manager allow INPUT "-p tcp --dport 80"
# iptables_manager nat masquerade eth0
# iptables_manager save
```

## Conclusion

Iptables reste l'outil de référence pour la configuration avancée des pare-feu Linux. En maîtrisant les tables, les chaînes, les règles de filtrage, et le NAT, vous pouvez créer des configurations de sécurité sophistiquées qui protègent efficacement vos systèmes tout en permettant les communications nécessaires.

Un pare-feu bien configuré utilise plusieurs couches de protection : filtrage par protocole et port, limitation des connexions, NAT pour masquer les réseaux internes, et logging pour l'audit. La compréhension approfondie d'iptables permet de créer des règles complexes qui s'adaptent aux besoins spécifiques de chaque environnement.

Dans le chapitre suivant, nous explorerons SSH en détail, découvrant comment configurer et sécuriser les connexions SSH pour l'administration distante sécurisée des systèmes Linux.
