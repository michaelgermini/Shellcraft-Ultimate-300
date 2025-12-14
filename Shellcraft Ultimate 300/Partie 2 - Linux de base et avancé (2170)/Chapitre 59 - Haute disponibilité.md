# Chapitre 59 - Haute disponibilité

## Table des matières
- [Introduction](#introduction)
- [Concepts de haute disponibilité](#concepts-de-haute-disponibilité)
- [Clustering avec Pacemaker](#clustering-avec-pacemaker)
- [Failover automatique](#failover-automatique)
- [Load balancing](#load-balancing)
- [Réplication de données](#réplication-de-données)
- [Monitoring et alertes](#monitoring-et-alertes)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La haute disponibilité (HA) garantit qu'un service reste accessible même en cas de défaillance d'un composant. Elle repose sur la redondance, la détection automatique des pannes, et le basculement transparent vers des ressources de secours.

Imaginez la haute disponibilité comme un système de secours dans un avion : si un moteur tombe en panne, les autres prennent le relais automatiquement, permettant à l'avion de continuer à voler en toute sécurité. Dans un système Linux, cela signifie configurer des clusters, des mécanismes de failover, et des systèmes de réplication pour garantir la continuité de service.

## Concepts de haute disponibilité

### Principes fondamentaux

**Composants essentiels** :
```bash
#!/bin/bash
# Composants d'un système HA

# 1. Redondance
#    - Plusieurs serveurs pour le même service
#    - Pas de point de défaillance unique

# 2. Détection de pannes
#    - Monitoring continu
#    - Heartbeat entre nœuds

# 3. Failover automatique
#    - Basculement transparent
#    - Récupération automatique

# 4. Partage de données
#    - Réplication synchrone/asynchrone
#    - Stockage partagé (SAN/NAS)
```

## Clustering avec Pacemaker

### Configuration Pacemaker

**Installation et configuration** :
```bash
#!/bin/bash
# Configuration Pacemaker

# Installer
sudo apt install pacemaker corosync pcs

# Configurer Corosync
sudo pcs cluster setup mycluster node1 node2

# Démarrer le cluster
sudo pcs cluster start --all
sudo pcs cluster enable --all

# Vérifier le statut
sudo pcs status
```

## Failover automatique

### Configuration de ressources

**Ressources HA** :
```bash
#!/bin/bash
# Configuration de ressources

# IP virtuelle
sudo pcs resource create VirtualIP ocf:heartbeat:IPaddr2 \
    ip=192.168.1.100 cidr_netmask=24 op monitor interval=30s

# Service
sudo pcs resource create WebServer systemd:apache2 \
    op monitor interval=20s

# Contraintes
sudo pcs constraint colocation add WebServer VirtualIP INFINITY
sudo pcs constraint order VirtualIP then WebServer
```

## Load balancing

### HAProxy

**Configuration HAProxy** :
```bash
#!/bin/bash
# Configuration HAProxy

# /etc/haproxy/haproxy.cfg
cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend web
    bind *:80
    default_backend servers

backend servers
    balance roundrobin
    server web1 192.168.1.10:80 check
    server web2 192.168.1.11:80 check
EOF

systemctl restart haproxy
```

## Réplication de données

### DRBD

**Configuration DRBD** :
```bash
#!/bin/bash
# Configuration DRBD

# Installer
sudo apt install drbd-utils

# Configuration /etc/drbd.d/drbd0.res
resource drbd0 {
    device /dev/drbd0;
    disk /dev/sdb1;
    meta-disk internal;
    on node1 {
        address 192.168.1.10:7788;
    }
    on node2 {
        address 192.168.1.11:7788;
    }
}

# Initialiser
sudo drbdadm create-md drbd0
sudo drbdadm up drbd0
sudo drbdadm primary drbd0 --force
```

## Monitoring et alertes

### Surveillance du cluster

**Scripts de monitoring** :
```bash
#!/bin/bash
# Monitoring du cluster

monitor_cluster() {
    while true; do
        # Vérifier le statut
        if ! pcs status | grep -q "Online"; then
            send_alert "Cluster node offline"
        fi
        
        # Vérifier les ressources
        if pcs status | grep -q "Stopped"; then
            send_alert "Resource stopped"
        fi
        
        sleep 60
    done
}
```

## Scripts d'automatisation

**Gestionnaire HA** :
```bash
#!/bin/bash
# Gestionnaire de haute disponibilité

ha_manager() {
    local action="$1"
    
    case "$action" in
        status)
            pcs status
            ;;
        failover)
            pcs cluster standby node1
            ;;
        online)
            pcs cluster unstandby node1
            ;;
        *)
            echo "Usage: $0 {status|failover|online}"
            ;;
    esac
}
```

## Conclusion

La haute disponibilité transforme des systèmes individuels en infrastructures résilientes capables de résister aux défaillances. En combinant clustering, failover automatique, et réplication de données, vous créez des systèmes qui garantissent la continuité de service même face aux pannes les plus critiques.

