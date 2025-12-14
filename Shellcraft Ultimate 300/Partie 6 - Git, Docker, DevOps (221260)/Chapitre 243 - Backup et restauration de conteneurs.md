# Chapitre 243 - Backup et restauration de conteneurs

## Table des matières
- [Introduction](#introduction)
- [Backup de volumes](#backup-de-volumes)
- [Backup de configurations](#backup-de-configurations)
- [Restauration](#restauration)
- [Conclusion](#conclusion)

## Introduction

Les backups sont essentiels pour protéger vos données et configurations. Ce chapitre couvre les stratégies de backup et restauration pour les conteneurs Docker et Kubernetes.

## Backup de volumes

**Script de backup** :
```bash
#!/bin/bash
# Backup de volumes Docker

backup_volume() {
    local volume="$1"
    local backup_file="${2:-backup_$(date +%Y%m%d_%H%M%S).tar.gz}"
    
    docker run --rm \
        -v "$volume":/data \
        -v "$(pwd)":/backup \
        alpine tar czf "/backup/$backup_file" /data
    
    echo "✓ Volume sauvegardé: $backup_file"
}

# Backup de tous les volumes
backup_all_volumes() {
    docker volume ls -q | while read -r volume; do
        backup_volume "$volume" "backup_${volume}_$(date +%Y%m%d_%H%M%S).tar.gz"
    done
}
```

## Backup de configurations

**Backup Kubernetes** :
```bash
#!/bin/bash
# Backup de configurations Kubernetes

backup_k8s_resources() {
    local namespace="${1:-all}"
    local backup_dir="backup_$(date +%Y%m%d_%H%M%S)"
    
    mkdir -p "$backup_dir"
    
    if [ "$namespace" == "all" ]; then
        kubectl get all --all-namespaces -o yaml > "$backup_dir/all_resources.yaml"
    else
        kubectl get all -n "$namespace" -o yaml > "$backup_dir/${namespace}_resources.yaml"
    fi
    
    echo "✓ Backup Kubernetes créé: $backup_dir"
}
```

## Restauration

**Script de restauration** :
```bash
#!/bin/bash
# Restauration de volumes

restore_volume() {
    local volume="$1"
    local backup_file="$2"
    
    docker run --rm \
        -v "$volume":/data \
        -v "$(pwd)":/backup \
        alpine tar xzf "/backup/$backup_file" -C /data
    
    echo "✓ Volume restauré depuis $backup_file"
}
```

## Conclusion

Les backups réguliers et les tests de restauration sont essentiels pour la continuité de service.

