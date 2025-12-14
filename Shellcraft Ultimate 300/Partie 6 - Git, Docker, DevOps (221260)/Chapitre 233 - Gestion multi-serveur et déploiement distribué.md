# Chapitre 233 - Gestion multi-serveur et déploiement distribué

## Table des matières
- [Introduction](#introduction)
- [Orchestration multi-serveur](#orchestration-multi-serveur)
- [Déploiement distribué](#déploiement-distribué)
- [Gestion de configuration](#gestion-de-configuration)
- [Monitoring distribué](#monitoring-distribué)
- [Conclusion](#conclusion)

## Introduction

La gestion multi-serveur et le déploiement distribué sont essentiels pour les applications modernes à grande échelle. Ces techniques permettent de gérer des infrastructures complexes réparties sur plusieurs serveurs.

Imaginez la gestion multi-serveur comme un chef d'orchestre dirigeant plusieurs orchestres simultanément : chaque serveur est un orchestre, et vous devez les coordonner pour créer une performance harmonieuse.

## Orchestration multi-serveur

### Gestion avec Ansible

**Playbook Ansible** :
```yaml
# deploy-multi-server.yml
---
- name: Déploiement multi-serveur
  hosts: all
  become: yes
  tasks:
    - name: Installer Docker
      apt:
        name: docker.io
        state: present
      when: ansible_os_family == "Debian"
    
    - name: Démarrer Docker
      systemd:
        name: docker
        state: started
        enabled: yes
    
    - name: Déployer l'application
      docker_container:
        name: app
        image: myapp:latest
        state: started
        ports:
          - "80:8080"
```

**Script de déploiement** :
```bash
#!/bin/bash
# Déploiement multi-serveur avec Ansible

deploy_to_servers() {
    local inventory_file="${1:-inventory.ini}"
    local playbook="${2:-deploy.yml}"
    
    ansible-playbook -i "$inventory_file" "$playbook"
}

# Inventaire des serveurs
create_inventory() {
    cat > inventory.ini << EOF
[web_servers]
web1.example.com ansible_user=deploy
web2.example.com ansible_user=deploy
web3.example.com ansible_user=deploy

[db_servers]
db1.example.com ansible_user=deploy
db2.example.com ansible_user=deploy

[all:vars]
ansible_ssh_private_key_file=~/.ssh/deploy_key
EOF
}
```

## Déploiement distribué

### Script de déploiement distribué

**Déploiement avec rsync** :
```bash
#!/bin/bash
# Déploiement distribué avec rsync

set -euo pipefail

DEPLOY_DIR="/var/www/app"
SERVERS=("server1.example.com" "server2.example.com" "server3.example.com")

deploy_to_all_servers() {
    local source_dir="${1:-./dist}"
    
    for server in "${SERVERS[@]}"; do
        echo "Déploiement sur $server..."
        
        # Synchroniser les fichiers
        rsync -avz --delete \
            --exclude='.git' \
            --exclude='node_modules' \
            "$source_dir/" "$server:$DEPLOY_DIR/"
        
        # Exécuter les commandes post-déploiement
        ssh "$server" "cd $DEPLOY_DIR && ./deploy.sh"
        
        echo "✓ $server déployé"
    done
}

# Déploiement en parallèle
parallel_deploy() {
    local source_dir="${1:-./dist}"
    
    for server in "${SERVERS[@]}"; do
        (
            echo "Déploiement sur $server..."
            rsync -avz --delete "$source_dir/" "$server:$DEPLOY_DIR/"
            ssh "$server" "cd $DEPLOY_DIR && ./deploy.sh"
            echo "✓ $server déployé"
        ) &
    done
    
    wait
    echo "Tous les déploiements terminés"
}
```

## Gestion de configuration

### Configuration distribuée

**Gestionnaire de configuration** :
```bash
#!/bin/bash
# Gestion de configuration distribuée

update_config_all_servers() {
    local config_file="$1"
    
    for server in "${SERVERS[@]}"; do
        echo "Mise à jour de la configuration sur $server..."
        
        scp "$config_file" "$server:$DEPLOY_DIR/config/"
        
        # Redémarrer le service si nécessaire
        ssh "$server" "systemctl restart app"
        
        echo "✓ Configuration mise à jour sur $server"
    done
}

# Synchronisation de configuration
sync_config() {
    local config_dir="${1:-./config}"
    
    for server in "${SERVERS[@]}"; do
        rsync -avz "$config_dir/" "$server:$DEPLOY_DIR/config/"
    done
}
```

## Monitoring distribué

### Monitoring multi-serveur

**Script de monitoring** :
```bash
#!/bin/bash
# Monitoring distribué

monitor_all_servers() {
    for server in "${SERVERS[@]}"; do
        echo "=== État de $server ==="
        
        # Vérifier la disponibilité
        if ssh "$server" "systemctl is-active app" &>/dev/null; then
            echo "✓ Service actif"
        else
            echo "✗ Service inactif"
        fi
        
        # Vérifier l'utilisation CPU
        cpu=$(ssh "$server" "top -bn1 | grep 'Cpu(s)' | awk '{print \$2}'")
        echo "CPU: $cpu"
        
        # Vérifier l'utilisation mémoire
        memory=$(ssh "$server" "free | grep Mem | awk '{printf \"%.2f\", \$3/\$2 * 100}'")
        echo "Mémoire: $memory%"
        
        echo
    done
}
```

## Conclusion

La gestion multi-serveur et le déploiement distribué sont essentiels pour les applications à grande échelle. En utilisant des outils comme Ansible, rsync, et des scripts personnalisés, vous pouvez gérer efficacement des infrastructures complexes.

Dans le chapitre suivant, nous explorerons le monitoring de conteneurs et l'observabilité.

