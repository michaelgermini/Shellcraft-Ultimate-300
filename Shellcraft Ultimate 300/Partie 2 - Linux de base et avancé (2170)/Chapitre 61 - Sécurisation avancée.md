# Chapitre 61 - Sécurisation avancée

## Table des matières
- [Introduction](#introduction)
- [SELinux](#selinux)
- [AppArmor](#apparmor)
- [Chiffrement avancé](#chiffrement-avancé)
- [Audit système](#audit-système)
- [Hardening avancé](#hardening-avancé)
- [Compliance](#compliance)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La sécurisation avancée transforme un système Linux fonctionnel en une forteresse numérique. Elle implique l'utilisation de mécanismes de sécurité multi-couches, de politiques de sécurité strictes, et de systèmes d'audit complets pour protéger contre les menaces modernes.

Imaginez la sécurisation avancée comme un système de sécurité multicouche d'une banque : chaque couche (SELinux, chiffrement, audit) protège contre différents types de menaces, créant une défense en profondeur qui rend la compromission extrêmement difficile.

## SELinux

### Configuration SELinux

**Gestion SELinux** :
```bash
#!/bin/bash
# Configuration SELinux

# Vérifier le statut
getenforce

# Mode permissif (test)
setenforce 0

# Mode enforcing
setenforce 1

# Politiques
semanage port -a -t http_port_t -p tcp 8080
```

## AppArmor

### Profils AppArmor

**Gestion AppArmor** :
```bash
#!/bin/bash
# AppArmor

# Statut
aa-status

# Charger un profil
apparmor_parser -r /etc/apparmor.d/myprofile

# Mode complain
aa-complain /usr/bin/myapp
```

## Chiffrement avancé

### LUKS et dm-crypt

**Chiffrement de disque** :
```bash
#!/bin/bash
# Chiffrement LUKS

# Créer un volume chiffré
cryptsetup luksFormat /dev/sdb1

# Ouvrir
cryptsetup luksOpen /dev/sdb1 encrypted_volume

# Formater
mkfs.ext4 /dev/mapper/encrypted_volume

# Monter
mount /dev/mapper/encrypted_volume /mnt
```

## Audit système

### auditd

**Configuration auditd** :
```bash
#!/bin/bash
# Configuration auditd

# Règles d'audit
auditctl -w /etc/passwd -p wa -k passwd_changes
auditctl -w /etc/shadow -p wa -k shadow_changes

# Rechercher dans les logs
ausearch -k passwd_changes
```

## Hardening avancé

### Sécurisation système

**Hardening** :
```bash
#!/bin/bash
# Hardening système

# Désactiver services inutiles
systemctl disable service_name

# Configurer firewall
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw enable

# Sécuriser SSH
sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
```

## Compliance

### Conformité réglementaire

**Checklists de compliance** :
```bash
#!/bin/bash
# Vérification de compliance

check_compliance() {
    # Vérifier les permissions
    find /etc -type f -perm /o+w
    
    # Vérifier les mots de passe
    awk -F: '$2 == "" {print $1}' /etc/shadow
    
    # Vérifier les services
    systemctl list-unit-files | grep enabled
}
```

## Scripts d'automatisation

**Gestionnaire de sécurité** :
```bash
#!/bin/bash
# Gestionnaire de sécurité

security_manager() {
    local action="$1"
    
    case "$action" in
        harden)
            apply_hardening
            ;;
        audit)
            run_audit
            ;;
        check)
            check_compliance
            ;;
        *)
            echo "Usage: $0 {harden|audit|check}"
            ;;
    esac
}
```

## Conclusion

La sécurisation avancée crée des systèmes qui résistent aux attaques les plus sophistiquées, en combinant des mécanismes de sécurité multiples et des politiques strictes pour protéger les données et les services critiques.

