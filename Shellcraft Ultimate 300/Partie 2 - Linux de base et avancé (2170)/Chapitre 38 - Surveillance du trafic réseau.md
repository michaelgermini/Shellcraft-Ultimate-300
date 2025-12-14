# Chapitre 38 - Surveillance du trafic réseau

## Table des matières
- [Introduction](#introduction)
- [Architecture de la surveillance réseau](#architecture-de-la-surveillance-réseau)
- [Outils de surveillance de base](#outils-de-surveillance-de-base)
- [tcpdump : capture de paquets en ligne de commande](#tcpdump--capture-de-paquets-en-ligne-de-commande)
- [Wireshark et tshark : analyse avancée](#wireshark-et-tshark--analyse-avancée)
- [Analyse de paquets avancée](#analyse-de-paquets-avancée)
- [Monitoring en temps réel](#monitoring-en-temps-réel)
- [Monitoring par processus](#monitoring-par-processus)
- [Statistiques et métriques réseau](#statistiques-et-métriques-réseau)
- [Détection d'anomalies](#détection-danomalies)
- [Collecte de métriques](#collecte-de-métriques)
- [Visualisation et rapports](#visualisation-et-rapports)
- [Automation de la surveillance](#automation-de-la-surveillance)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La surveillance du trafic réseau constitue un pilier essentiel de la sécurité et des performances système. Comprendre ce qui transite sur le réseau permet de détecter les intrusions, d'optimiser les ressources, et d'assurer la qualité de service.

Imaginez la surveillance réseau comme un contrôleur aérien dans une tour de contrôle : il observe tous les mouvements, détecte les anomalies, coordonne les flux, et intervient lorsque nécessaire pour maintenir la sécurité et l'efficacité du trafic. Dans un système Linux moderne, cette surveillance s'étend de la capture basique de paquets à l'analyse approfondie des protocoles, en passant par le monitoring continu et l'automatisation complète.

## Architecture de la surveillance réseau

### Méthodes de capture

**Types de capture** :
```bash
#!/bin/bash
# Méthodes de capture réseau

# 1. Capture promiscuité
# L'interface écoute tous les paquets, pas seulement ceux qui lui sont destinés
# Nécessite les privilèges root

# 2. Capture sur interface spécifique
# Capture uniquement sur une interface réseau donnée

# 3. Capture en miroir (port mirroring)
# Les paquets sont copiés vers une interface de monitoring
# Utilisé dans les environnements réseau complexes

# 4. Capture depuis un fichier
# Analyse de captures précédemment sauvegardées
```

**Interfaces de capture** :
```bash
#!/bin/bash
# Interfaces de capture

# Interfaces physiques
# eth0, enp0s3 : Ethernet
# wlan0 : WiFi

# Interfaces virtuelles
# any : Toutes les interfaces
# lo : Loopback

# Lister les interfaces disponibles pour capture
tcpdump -D
# ou
ip link show
```

## Outils de surveillance de base

### Outils essentiels

**Outils de base** :
```bash
#!/bin/bash
# Outils de surveillance réseau

# tcpdump : Capture de paquets en ligne de commande
# wireshark/tshark : Analyse graphique et CLI
# iftop : Monitoring de bande passante en temps réel
# nethogs : Trafic par processus
# vnstat : Statistiques réseau historiques
# netstat/ss : Connexions actives
# iptraf : Monitoring interactif
# ntopng : Monitoring web avancé
```

## tcpdump : capture de paquets en ligne de commande

### Utilisation de base

**Commandes tcpdump essentielles** :
```bash
#!/bin/bash
# Utilisation de base de tcpdump

# Capturer sur une interface
sudo tcpdump -i eth0

# Capturer sur toutes les interfaces
sudo tcpdump -i any

# Capturer avec nombre limité de paquets
sudo tcpdump -i eth0 -c 100

# Capturer et sauvegarder dans un fichier
sudo tcpdump -i eth0 -w capture.pcap

# Lire depuis un fichier
tcpdump -r capture.pcap

# Format verbose
sudo tcpdump -i eth0 -v      # Verbose
sudo tcpdump -i eth0 -vv     # Plus verbose
sudo tcpdump -i eth0 -vvv    # Très verbose

# Sans résolution DNS (plus rapide)
sudo tcpdump -i eth0 -n

# Sans résolution de ports
sudo tcpdump -i eth0 -nn

# Afficher les données
sudo tcpdump -i eth0 -A      # ASCII
sudo tcpdump -i eth0 -X      # Hex + ASCII
sudo tcpdump -i eth0 -XX     # Hex + ASCII avec en-têtes
```

### Filtres tcpdump

**Filtres de base** :
```bash
#!/bin/bash
# Filtres tcpdump

# Par hôte
sudo tcpdump -i eth0 host 192.168.1.1
sudo tcpdump -i eth0 host google.com

# Par réseau
sudo tcpdump -i eth0 net 192.168.1.0/24
sudo tcpdump -i eth0 net 192.168.1.0 mask 255.255.255.0

# Par port
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 port 22
sudo tcpdump -i eth0 port 80 or port 443

# Par protocole
sudo tcpdump -i eth0 tcp
sudo tcpdump -i eth0 udp
sudo tcpdump -i eth0 icmp
sudo tcpdump -i eth0 arp

# Combinaisons
sudo tcpdump -i eth0 tcp port 80
sudo tcpdump -i eth0 host 192.168.1.1 and port 22
sudo tcpdump -i eth0 host 192.168.1.1 and not port 22

# Par direction
sudo tcpdump -i eth0 src host 192.168.1.1
sudo tcpdump -i eth0 dst host 192.168.1.1
sudo tcpdump -i eth0 src port 80
sudo tcpdump -i eth0 dst port 443
```

**Filtres avancés** :
```bash
#!/bin/bash
# Filtres avancés tcpdump

# Par taille de paquet
sudo tcpdump -i eth0 greater 1000    # Paquets > 1000 bytes
sudo tcpdump -i eth0 less 64          # Paquets < 64 bytes

# Par plage de ports
sudo tcpdump -i eth0 portrange 8000-8099

# Flags TCP
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-syn != 0'    # SYN
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-ack != 0'    # ACK
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-fin != 0'    # FIN
sudo tcpdump -i eth0 'tcp[tcpflags] & tcp-rst != 0'    # RST

# Connexions TCP établies
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn|tcp-ack) == tcp-syn|tcp-ack'

# HTTP
sudo tcpdump -i eth0 -A 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# DNS
sudo tcpdump -i eth0 port 53
sudo tcpdump -i eth0 -A port 53  # Avec contenu

# SSH
sudo tcpdump -i eth0 port 22

# FTP
sudo tcpdump -i eth0 port 21
```

### Options avancées tcpdump

**Options de capture** :
```bash
#!/bin/bash
# Options avancées tcpdump

# Taille de buffer
sudo tcpdump -i eth0 -B 4096

# Snap length (taille max de paquet capturé)
sudo tcpdump -i eth0 -s 0        # Tous les paquets
sudo tcpdump -i eth0 -s 1514     # Taille Ethernet standard
sudo tcpdump -i eth0 -s 96       # En-têtes seulement

# Timestamp
sudo tcpdump -i eth0 -t          # Pas de timestamp
sudo tcpdump -i eth0 -tt         # Timestamp Unix
sudo tcpdump -i eth0 -ttt        # Timestamp relatif
sudo tcpdump -i eth0 -tttt       # Timestamp formaté

# Rotation de fichiers
sudo tcpdump -i eth0 -w capture.pcap -C 100  # Nouveau fichier tous les 100 MB
sudo tcpdump -i eth0 -w capture.pcap -W 10    # Rotation de 10 fichiers
sudo tcpdump -i eth0 -w capture.pcap -G 3600 # Nouveau fichier toutes les heures

# Capture avec compression
sudo tcpdump -i eth0 -w capture.pcap -z gzip

# Affichage formaté
sudo tcpdump -i eth0 -l | tee capture.txt    # Ligne par ligne
```

**Scripts avec tcpdump** :
```bash
#!/bin/bash
# Scripts avec tcpdump

# Capturer le trafic HTTP
capture_http() {
    local iface="${1:-eth0}"
    local output="${2:-http_capture.pcap}"
    
    sudo tcpdump -i "$iface" -w "$output" \
        'tcp port 80 or tcp port 443'
    
    echo "Capture HTTP sauvegardée dans $output"
}

# Capturer les connexions SSH
capture_ssh() {
    local iface="${1:-eth0}"
    local output="${2:-ssh_capture.pcap}"
    
    sudo tcpdump -i "$iface" -w "$output" \
        'tcp port 22'
    
    echo "Capture SSH sauvegardée dans $output"
}

# Analyser les paquets DNS
analyze_dns() {
    local pcap_file="$1"
    
    tcpdump -r "$pcap_file" -n 'udp port 53' | \
    awk '{print $3, $5, $NF}' | \
    sort | uniq -c | sort -rn
}

# Compter les paquets par protocole
count_by_protocol() {
    local pcap_file="$1"
    
    tcpdump -r "$pcap_file" -n | \
    awk '{print $5}' | \
    cut -d'.' -f1 | \
    sort | uniq -c | sort -rn
}
```

## Wireshark et tshark : analyse avancée

### tshark : Wireshark en ligne de commande

**Utilisation de tshark** :
```bash
#!/bin/bash
# Utilisation de tshark

# Capturer sur une interface
sudo tshark -i eth0

# Capturer avec filtres
sudo tshark -i eth0 -f "tcp port 80"

# Capturer et sauvegarder
sudo tshark -i eth0 -w capture.pcapng

# Lire depuis un fichier
tshark -r capture.pcapng

# Statistiques
tshark -r capture.pcapng -q -z io,stat,0

# Statistiques par protocole
tshark -r capture.pcapng -q -z io,phs

# Top talkers
tshark -r capture.pcapng -q -z endpoints,ip

# Conversations
tshark -r capture.pcapng -q -z conv,ip

# Filtres d'affichage
tshark -r capture.pcapng -Y "http"
tshark -r capture.pcapng -Y "dns"
tshark -r capture.pcapng -Y "tcp.port == 80"

# Champs spécifiques
tshark -r capture.pcapng -T fields -e ip.src -e ip.dst -e tcp.port

# Format JSON
tshark -r capture.pcapng -T json

# Format CSV
tshark -r capture.pcapng -T fields -e frame.number -e ip.src -e ip.dst -E header=y -E separator=, > output.csv
```

**Scripts avec tshark** :
```bash
#!/bin/bash
# Scripts avec tshark

# Extraire les URLs HTTP
extract_http_urls() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "http.request" \
        -T fields -e http.host -e http.request.uri | \
    sort | uniq
}

# Analyser les requêtes DNS
analyze_dns_queries() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "dns" \
        -T fields -e dns.qry.name | \
    sort | uniq -c | sort -rn
}

# Statistiques de protocoles
protocol_statistics() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -q -z io,phs
}
```

## Analyse de paquets avancée

### Analyse de protocoles

**Analyse HTTP** :
```bash
#!/bin/bash
# Analyse HTTP

# Capturer le trafic HTTP
capture_http_traffic() {
    sudo tcpdump -i eth0 -A -s 0 'tcp port 80' | \
    grep -E "(GET|POST|PUT|DELETE|HEAD|OPTIONS)"
}

# Extraire les en-têtes HTTP
extract_http_headers() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "http" \
        -T fields -e http.request.method \
        -e http.request.uri \
        -e http.host \
        -e http.user_agent
}

# Analyser les codes de réponse
analyze_http_status() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "http.response" \
        -T fields -e http.response.code | \
    sort | uniq -c | sort -rn
}
```

**Analyse DNS** :
```bash
#!/bin/bash
# Analyse DNS

# Capturer les requêtes DNS
capture_dns() {
    sudo tcpdump -i eth0 -A 'udp port 53'
}

# Analyser les requêtes DNS
analyze_dns_queries() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "dns.flags.response == 0" \
        -T fields -e dns.qry.name | \
    sort | uniq -c | sort -rn
}

# Analyser les réponses DNS
analyze_dns_responses() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "dns.flags.response == 1" \
        -T fields -e dns.qry.name -e dns.a
}
```

**Analyse TCP** :
```bash
#!/bin/bash
# Analyse TCP

# Analyser les connexions TCP
analyze_tcp_connections() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -q -z conv,tcp
}

# Détecter les scans de ports
detect_port_scans() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "tcp.flags.syn == 1 and tcp.flags.ack == 0" \
        -T fields -e ip.src | \
    sort | uniq -c | sort -rn | head -10
}

# Analyser les retransmissions
analyze_retransmissions() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" -Y "tcp.analysis.retransmission" \
        -T fields -e ip.src -e ip.dst -e tcp.port
}
```

## Monitoring en temps réel

### iftop : monitoring de bande passante

**Utilisation de iftop** :
```bash
#!/bin/bash
# Utilisation de iftop

# Monitoring sur une interface
sudo iftop -i eth0

# Sans résolution DNS (plus rapide)
sudo iftop -i eth0 -n

# Afficher les ports
sudo iftop -i eth0 -P

# Filtrer par réseau
sudo iftop -i eth0 -f "net 192.168.1.0/24"

# Filtrer par port
sudo iftop -i eth0 -f "port 80"

# Mode texte (pour scripts)
sudo iftop -i eth0 -t -s 10  # 10 secondes

# Sauvegarder dans un fichier
sudo iftop -i eth0 -t -s 10 > iftop_output.txt
```

**Scripts avec iftop** :
```bash
#!/bin/bash
# Scripts avec iftop

# Monitoring continu avec alertes
monitor_bandwidth() {
    local iface="${1:-eth0}"
    local threshold="${2:-1000}"  # KB/s
    
    while true; do
        iftop -i "$iface" -t -s 5 | \
        awk '/Total send and receive rate/ {
            rate = $6
            gsub(/KB/, "", rate)
            if (rate > threshold) {
                print "ALERT: High bandwidth usage: " rate " KB/s"
            }
        }'
        sleep 5
    done
}
```

### iptraf : monitoring interactif

**Utilisation de iptraf** :
```bash
#!/bin/bash
# Utilisation de iptraf

# Interface interactive
sudo iptraf-ng

# Options :
# - General interface statistics
# - Detailed interface statistics
# - Statistical breakdowns
# - IP traffic monitor
# - LAN station monitor
```

## Monitoring par processus

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

# Mode tracé (pour scripts)
sudo nethogs -t
```

**Scripts avec nethogs** :
```bash
#!/bin/bash
# Scripts avec nethogs

# Identifier les processus consommateurs
find_bandwidth_hogs() {
    sudo nethogs -t -d 1 | \
    awk '/^[0-9]/ {
        process = $1
        sent = $2
        received = $3
        total = sent + received
        if (total > 1000) {  # Plus de 1 MB/s
            print process ": " total " KB/s"
        }
    }'
}
```

## Statistiques et métriques réseau

### vnstat : statistiques historiques

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

# Statistiques sur 5 minutes
vnstat -5

# Export JSON
vnstat --json

# Export XML
vnstat --xml
```

**Configuration vnstat** :
```bash
#!/bin/bash
# Configuration vnstat

# Créer une base de données pour une interface
sudo vnstat -i eth0 --create

# Mettre à jour les statistiques
sudo vnstat -u

# Configuration : /etc/vnstat.conf
```

### Statistiques système

**Statistiques depuis /proc** :
```bash
#!/bin/bash
# Statistiques réseau depuis /proc

# Statistiques par interface
cat /proc/net/dev

# Statistiques détaillées
cat /proc/net/snmp

# Connexions TCP
cat /proc/net/tcp

# Connexions UDP
cat /proc/net/udp

# Statistiques IP
cat /proc/net/snmp | grep Ip:

# Statistiques TCP
cat /proc/net/snmp | grep Tcp:

# Script d'analyse
analyze_network_stats() {
    echo "=== Statistiques interfaces ==="
    cat /proc/net/dev | awk 'NR>2 {
        iface = $1
        gsub(/:/, "", iface)
        rx = $2
        tx = $10
        printf "%s: RX=%d bytes, TX=%d bytes\n", iface, rx, tx
    }'
}
```

## Détection d'anomalies

### Détection de scans

**Détecter les scans de ports** :
```bash
#!/bin/bash
# Détection de scans de ports

detect_port_scans() {
    local pcap_file="$1"
    local threshold="${2:-10}"  # Nombre de ports par IP
    
    tshark -r "$pcap_file" \
        -Y "tcp.flags.syn == 1 and tcp.flags.ack == 0" \
        -T fields -e ip.src -e tcp.dstport | \
    awk '{print $1}' | \
    sort | uniq -c | \
    awk -v thresh="$threshold" '$1 > thresh {
        print "Potential port scan from " $2 ": " $1 " ports"
    }'
}

# En temps réel
monitor_port_scans() {
    sudo tcpdump -i eth0 -n 'tcp[tcpflags] & tcp-syn != 0' | \
    awk '{print $3}' | \
    cut -d'.' -f1-4 | \
    sort | uniq -c | \
    awk '$1 > 10 {print "Potential scan from " $2 ": " $1 " SYN packets"}'
}
```

### Détection de trafic suspect

**Détecter le trafic suspect** :
```bash
#!/bin/bash
# Détection de trafic suspect

# Trafic vers des ports non standards
detect_suspicious_ports() {
    local pcap_file="$1"
    
    tshark -r "$pcap_file" \
        -Y "tcp.dstport > 1024" \
        -T fields -e ip.dst -e tcp.dstport | \
    sort | uniq -c | sort -rn | head -20
}

# Trafic vers des IPs externes
detect_external_traffic() {
    local pcap_file="$1"
    local internal_net="${2:-192.168.1.0/24}"
    
    tshark -r "$pcap_file" \
        -Y "ip.dst and not ip.dst == $internal_net" \
        -T fields -e ip.dst | \
    sort | uniq -c | sort -rn
}

# Paquets de grande taille
detect_large_packets() {
    local pcap_file="$1"
    local size="${2:-1500}"
    
    tshark -r "$pcap_file" \
        -Y "frame.len > $size" \
        -T fields -e ip.src -e ip.dst -e frame.len | \
    sort -k3 -rn | head -20
}
```

## Collecte de métriques

### Scripts de collecte

**Collecteur de métriques réseau** :
```bash
#!/bin/bash
# Collecteur de métriques réseau

collect_network_metrics() {
    local output_dir="${1:-/var/log/network-metrics}"
    local interval="${2:-60}"  # secondes
    
    mkdir -p "$output_dir"
    
    while true; do
        timestamp=$(date +%Y%m%d_%H%M%S)
        metrics_file="$output_dir/metrics_$timestamp.json"
        
        # Collecter les métriques
        cat > "$metrics_file" << EOF
{
    "timestamp": "$(date -Iseconds)",
    "interfaces": {
$(ip -s link show | awk '
    /^[0-9]+:/ {
        iface = $2
        gsub(/:/, "", iface)
    }
    /RX:/ {
        rx_bytes = $2
        rx_packets = $3
        rx_errors = $4
        rx_dropped = $5
    }
    /TX:/ {
        tx_bytes = $2
        tx_packets = $3
        tx_errors = $4
        tx_dropped = $5
        printf "        \"%s\": {\n", iface
        printf "            \"rx_bytes\": %s,\n", rx_bytes
        printf "            \"rx_packets\": %s,\n", rx_packets
        printf "            \"rx_errors\": %s,\n", rx_errors
        printf "            \"rx_dropped\": %s,\n", rx_dropped
        printf "            \"tx_bytes\": %s,\n", tx_bytes
        printf "            \"tx_packets\": %s,\n", tx_packets
        printf "            \"tx_errors\": %s,\n", tx_errors
        printf "            \"tx_dropped\": %s\n", tx_dropped
        printf "        },\n"
    }
' | sed '$ s/,$//')
    },
    "connections": {
        "established": $(ss -tn state established | wc -l),
        "listening": $(ss -tln | grep LISTEN | wc -l)
    }
}
EOF
        
        sleep "$interval"
    done
}
```

## Visualisation et rapports

### Génération de rapports

**Générer des rapports** :
```bash
#!/bin/bash
# Génération de rapports

generate_traffic_report() {
    local pcap_file="$1"
    local output_file="${2:-report.txt}"
    
    {
        echo "=== Rapport d'analyse réseau ==="
        echo "Fichier: $pcap_file"
        echo "Date: $(date)"
        echo ""
        
        echo "=== Statistiques générales ==="
        tshark -r "$pcap_file" -q -z io,stat,0
        echo ""
        
        echo "=== Top 10 protocoles ==="
        tshark -r "$pcap_file" -q -z io,phs | head -15
        echo ""
        
        echo "=== Top 10 conversations ==="
        tshark -r "$pcap_file" -q -z conv,tcp | head -15
        echo ""
        
        echo "=== Top 10 endpoints ==="
        tshark -r "$pcap_file" -q -z endpoints,ip | head -15
        
    } > "$output_file"
    
    echo "Rapport généré: $output_file"
}
```

## Automation de la surveillance

### Surveillance continue

**Script de surveillance continue** :
```bash
#!/bin/bash
# Surveillance réseau continue

network_monitor() {
    local iface="${1:-eth0}"
    local log_dir="${2:-/var/log/network-monitor}"
    local interval="${3:-300}"  # 5 minutes
    
    mkdir -p "$log_dir"
    
    while true; do
        timestamp=$(date +%Y%m%d_%H%M%S)
        
        # Capture de 1 minute
        capture_file="$log_dir/capture_$timestamp.pcap"
        sudo tcpdump -i "$iface" -w "$capture_file" -G 60 -W 1
        
        # Analyse
        analyze_capture "$capture_file" "$log_dir/report_$timestamp.txt"
        
        sleep "$interval"
    done
}

analyze_capture() {
    local pcap_file="$1"
    local report_file="$2"
    
    {
        echo "=== Analyse de capture ==="
        echo "Fichier: $pcap_file"
        echo ""
        
        echo "=== Statistiques ==="
        tshark -r "$pcap_file" -q -z io,stat,0
        echo ""
        
        echo "=== Alertes ==="
        # Détecter les scans
        detect_port_scans "$pcap_file" >> "$report_file" 2>/dev/null || true
        
    } > "$report_file"
}
```

## Scripts d'automatisation

### Gestionnaire de surveillance complet

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de surveillance réseau

set -euo pipefail

network_surveillance_manager() {
    local action="$1"
    shift
    
    case "$action" in
        capture)
            start_capture "$@"
            ;;
        analyze)
            analyze_capture_file "$@"
            ;;
        monitor)
            start_monitoring "$@"
            ;;
        report)
            generate_report "$@"
            ;;
        *)
            echo "Usage: network_surveillance_manager {capture|analyze|monitor|report}"
            exit 1
            ;;
    esac
}

start_capture() {
    local iface="${1:-eth0}"
    local duration="${2:-60}"
    local output="${3:-capture_$(date +%Y%m%d_%H%M%S).pcap}"
    
    echo "Capture démarrée sur $iface pour $duration secondes..."
    sudo tcpdump -i "$iface" -w "$output" -G "$duration" -W 1
    
    echo "Capture sauvegardée: $output"
}

analyze_capture_file() {
    local pcap_file="$1"
    
    if [ ! -f "$pcap_file" ]; then
        echo "Fichier non trouvé: $pcap_file"
        exit 1
    fi
    
    echo "=== Analyse de $pcap_file ==="
    echo ""
    
    echo "=== Statistiques générales ==="
    tshark -r "$pcap_file" -q -z io,stat,0
    echo ""
    
    echo "=== Top protocoles ==="
    tshark -r "$pcap_file" -q -z io,phs | head -10
    echo ""
    
    echo "=== Top conversations ==="
    tshark -r "$pcap_file" -q -z conv,tcp | head -10
}

start_monitoring() {
    local iface="${1:-eth0}"
    
    echo "Monitoring démarré sur $iface"
    echo "Appuyez sur Ctrl+C pour arrêter"
    
    while true; do
        clear
        echo "=== Monitoring réseau - $(date) ==="
        echo ""
        
        echo "=== Interfaces ==="
        ip -s link show | grep -E "^[0-9]+:|RX:|TX:" | head -20
        echo ""
        
        echo "=== Connexions actives ==="
        ss -tn state established | wc -l
        echo "connexions établies"
        echo ""
        
        echo "=== Top processus réseau ==="
        sudo nethogs -t -d 1 | head -10
        
        sleep 5
    done
}

generate_report() {
    local pcap_file="$1"
    local output_file="${2:-network_report_$(date +%Y%m%d_%H%M%S).txt}"
    
    generate_traffic_report "$pcap_file" "$output_file"
    echo "Rapport généré: $output_file"
}

# Utilisation
# network_surveillance_manager capture eth0 60
# network_surveillance_manager analyze capture.pcap
# network_surveillance_manager monitor eth0
# network_surveillance_manager report capture.pcap
```

## Conclusion

La surveillance du trafic réseau est essentielle pour maintenir la sécurité, les performances et la disponibilité des systèmes. En maîtrisant les outils de capture comme tcpdump et tshark, les outils de monitoring en temps réel comme iftop et nethogs, et les techniques d'analyse avancée, vous pouvez détecter les anomalies, optimiser les ressources, et assurer la qualité de service.

Un système bien surveillé utilise une combinaison d'outils pour capturer, analyser et monitorer le trafic réseau de manière continue et automatisée. La compréhension approfondie de ces outils permet de créer des systèmes de surveillance robustes qui détectent les problèmes avant qu'ils n'affectent les utilisateurs.

Dans le chapitre suivant, nous explorerons la sécurité réseau de base, découvrant les principes fondamentaux de la sécurisation des communications réseau et la protection contre les menaces courantes.
