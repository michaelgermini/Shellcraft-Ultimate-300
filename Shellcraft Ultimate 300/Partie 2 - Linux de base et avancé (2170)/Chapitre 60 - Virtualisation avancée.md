# Chapitre 60 - Virtualisation avancée

## Table des matières
- [Introduction](#introduction)
- [KVM avancé](#kvm-avancé)
- [Gestion de réseaux virtuels](#gestion-de-réseaux-virtuels)
- [Optimisation des performances VM](#optimisation-des-performances-vm)
- [Containers avancés](#containers-avancés)
- [Migration et sauvegarde](#migration-et-sauvegarde)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La virtualisation avancée permet de créer et gérer des environnements virtuels complexes, optimisés pour les performances et la sécurité. Au-delà de la création simple de VMs, il s'agit de concevoir des architectures virtuelles qui répondent aux besoins spécifiques de chaque charge de travail.

Imaginez la virtualisation avancée comme un architecte qui conçoit des bâtiments complexes : chaque VM est un appartement, mais l'architecture globale (réseaux virtuels, stockage partagé, migration en direct) détermine l'efficacité et la résilience de l'ensemble.

## KVM avancé

### Configuration avancée

**Création de VM optimisée** :
```bash
#!/bin/bash
# Création de VM KVM optimisée

virt-install \
    --name vm1 \
    --ram 2048 \
    --vcpus 2 \
    --disk path=/var/lib/libvirt/images/vm1.qcow2,size=20 \
    --network bridge=br0 \
    --graphics vnc \
    --os-type linux \
    --os-variant ubuntu20.04 \
    --cpu host-passthrough
```

## Gestion de réseaux virtuels

### Réseaux virtuels complexes

**Configuration de réseaux** :
```bash
#!/bin/bash
# Configuration réseau virtuel

# Créer un bridge
brctl addbr br0
ip addr add 192.168.1.1/24 dev br0
ip link set br0 up

# Ajouter interface au bridge
brctl addif br0 eth0
```

## Optimisation des performances VM

### Tuning des VMs

**Optimisation** :
```bash
#!/bin/bash
# Optimisation VM

# CPU pinning
virsh vcpupin vm1 0 0
virsh vcpupin vm1 1 1

# Mémoire
virsh setmem vm1 4096 --live
virsh setmaxmem vm1 8192 --config
```

## Containers avancés

### LXC et systemd-nspawn

**Containers système** :
```bash
#!/bin/bash
# Containers LXC

# Créer un container
lxc-create -t download -n mycontainer -- \
    -d ubuntu -r focal -a amd64

# Démarrer
lxc-start -n mycontainer
lxc-attach -n mycontainer
```

## Migration et sauvegarde

### Migration en direct

**Migration de VMs** :
```bash
#!/bin/bash
# Migration en direct

virsh migrate --live vm1 qemu+ssh://target/system
```

## Scripts d'automatisation

**Gestionnaire de virtualisation** :
```bash
#!/bin/bash
# Gestionnaire de VMs

vm_manager() {
    local action="$1"
    local vm="$2"
    
    case "$action" in
        create)
            create_vm "$vm"
            ;;
        migrate)
            migrate_vm "$vm" "$3"
            ;;
        backup)
            backup_vm "$vm"
            ;;
        *)
            echo "Usage: $0 {create|migrate|backup}"
            ;;
    esac
}
```

## Conclusion

La virtualisation avancée permet de créer des infrastructures flexibles et performantes qui s'adaptent aux besoins changeants des applications modernes.

