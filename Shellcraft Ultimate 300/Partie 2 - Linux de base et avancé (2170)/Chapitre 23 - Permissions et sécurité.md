# Chapitre 23 - Permissions et sécurité

## Table des matières
- [Introduction](#introduction)
- [Modèle de permissions UNIX avancé](#modèle-de-permissions-unix-avancé)
- [Commandes avancées chmod, chown, chgrp](#commandes-avancées-chmod-chown-chgrp)
- [Permissions spéciales (SUID, SGID, sticky bit)](#permissions-spéciales-suid-sgid-sticky-bit)
- [ACL (Access Control Lists)](#acl-access-control-lists)
- [Sécurité des fichiers et répertoires](#sécurité-des-fichiers-et-répertoires)
- [Audit et monitoring des accès](#audit-et-monitoring-des-accès)
- [Sécurité avancée : SELinux et AppArmor](#sécurité-avancée--selinux-et-apparmor)
- [Attributs étendus et capabilities](#attributs-étendus-et-capabilities)
- [Bonnes pratiques de sécurité](#bonnes-pratiques-de-sécurité)
- [Scripts de sécurité automatisés](#scripts-de-sécurité-automatisés)
- [Conclusion](#conclusion)

## Introduction

Les permissions constituent le fondement de la sécurité dans les systèmes UNIX et Linux. Elles déterminent qui peut accéder à quoi, comment et dans quelles conditions. Au-delà d'un simple système de contrôle d'accès, les permissions représentent une philosophie de partage contrôlé et de responsabilité individuelle.

Imaginez les permissions comme les serrures d'une maison moderne : chaque porte a ses propres clés, certaines pièces sont accessibles à tous, d'autres seulement à certains membres de la famille, et le coffre-fort reste l'apanage du propriétaire uniquement. Mais dans un environnement professionnel, vous avez besoin de systèmes plus sophistiqués : badges d'accès, caméras de surveillance, journaux d'audit, et même des gardes de sécurité (SELinux/AppArmor) qui vérifient chaque action.

## Modèle de permissions UNIX avancé

### Compréhension approfondie

**Structure complète des permissions** :
```bash
#!/bin/bash
# Analyse détaillée des permissions

analyze_permissions() {
    local file="$1"
    
    # Obtenir les informations complètes
    local stat_info=$(stat -c "%a %U %G %n" "$file")
    local ls_info=$(ls -ld "$file")
    
    echo "=== Analyse des permissions ==="
    echo "Fichier: $file"
    echo "Permissions numériques: $(stat -c %a "$file")"
    echo "Permissions symboliques: $(stat -c %A "$file")"
    echo "Propriétaire: $(stat -c %U "$file") (UID: $(stat -c %u "$file"))"
    echo "Groupe: $(stat -c %G "$file") (GID: $(stat -c %g "$file"))"
    echo "Inode: $(stat -c %i "$file")"
    echo "Liens: $(stat -c %h "$file")"
    echo ""
    
    # Analyser chaque bit
    local perms=$(stat -c %a "$file")
    local owner_perm=$((perms / 100))
    local group_perm=$(((perms % 100) / 10))
    local other_perm=$((perms % 10))
    
    echo "Détail des permissions:"
    echo "  Propriétaire: $owner_perm ($(octal_to_symbolic $owner_perm))"
    echo "  Groupe: $group_perm ($(octal_to_symbolic $group_perm))"
    echo "  Autres: $other_perm ($(octal_to_symbolic $other_perm))"
}

octal_to_symbolic() {
    local octal="$1"
    local result=""
    
    [ $((octal & 4)) -eq 4 ] && result+="r" || result+="-"
    [ $((octal & 2)) -eq 2 ] && result+="w" || result+="-"
    [ $((octal & 1)) -eq 1 ] && result+="x" || result+="-"
    
    echo "$result"
}
```

### Permissions et sécurité contextuelle

**Vérification contextuelle** :
```bash
#!/bin/bash
# Vérification de sécurité contextuelle

check_security_context() {
    local file="$1"
    
    # Vérifier les permissions
    local perms=$(stat -c %a "$file")
    
    # Vérifier les permissions dangereuses
    if [ $((perms % 10)) -ge 7 ]; then
        echo "⚠️  ATTENTION: Autres utilisateurs ont accès complet (rwx)"
        return 1
    fi
    
    # Vérifier SUID/SGID
    if [ -u "$file" ] || [ -g "$file" ]; then
        echo "⚠️  Fichier avec SUID/SGID détecté"
        if [ ! -f "$file" ]; then
            echo "   Vérifier la nécessité de cette permission"
        fi
    fi
    
    # Vérifier les fichiers world-writable
    if [ -w "$file" ] && [ ! -O "$file" ]; then
        echo "⚠️  Fichier modifiable par d'autres utilisateurs"
    fi
    
    return 0
}
```

## Commandes avancées chmod, chown, chgrp

### chmod avancé

**Techniques avancées** :
```bash
#!/bin/bash
# Utilisation avancée de chmod

# 1. Permissions conditionnelles
set_conditional_permissions() {
    local dir="$1"
    
    # Scripts exécutables seulement pour propriétaire et groupe
    find "$dir" -name "*.sh" -type f -exec chmod 750 {} \;
    
    # Fichiers de configuration lisibles par tous, modifiables par propriétaire
    find "$dir" -name "*.conf" -type f -exec chmod 644 {} \;
    
    # Répertoires avec accès traversable
    find "$dir" -type d -exec chmod 755 {} \;
}

# 2. Permissions basées sur le type de fichier
smart_permissions() {
    local file="$1"
    local ext="${file##*.}"
    
    case "$ext" in
        sh|bash|py|pl)
            chmod 755 "$file"  # Exécutable
            ;;
        txt|md|log)
            chmod 644 "$file"  # Lecture seule
            ;;
        key|pem|p12)
            chmod 600 "$file"  # Privé
            ;;
        *)
            chmod 644 "$file"  # Par défaut
            ;;
    esac
}

# 3. Masque de permissions personnalisé
apply_custom_mask() {
    local dir="$1"
    local owner_mask="${2:-7}"    # rwx par défaut
    local group_mask="${3:-5}"    # r-x par défaut
    local other_mask="${4:-0}"    # --- par défaut
    
    local mask="${owner_mask}${group_mask}${other_mask}"
    
    find "$dir" -type f -exec chmod "$mask" {} \;
    find "$dir" -type d -exec chmod "${mask}5" {} \;  # +x pour répertoires
}

# 4. Permissions récursives avec exclusions
chmod_recursive_smart() {
    local dir="$1"
    local file_perms="${2:-644}"
    local dir_perms="${3:-755}"
    
    find "$dir" -type f ! -perm "$file_perms" -exec chmod "$file_perms" {} \;
    find "$dir" -type d ! -perm "$dir_perms" -exec chmod "$dir_perms" {} \;
}
```

### chown avancé

**Gestion avancée de propriété** :
```bash
#!/bin/bash
# Utilisation avancée de chown

# 1. Changement de propriétaire avec préservation
change_owner_preserve() {
    local target="$1"
    local new_owner="$2"
    
    # Sauvegarder les permissions actuelles
    local old_perms=$(stat -c %a "$target")
    
    # Changer le propriétaire
    chown "$new_owner" "$target"
    
    # Restaurer les permissions si nécessaire
    chmod "$old_perms" "$target"
}

# 2. Migration de propriété en masse
migrate_ownership() {
    local old_owner="$1"
    local new_owner="$2"
    local base_dir="${3:-.}"
    
    # Trouver tous les fichiers de l'ancien propriétaire
    find "$base_dir" -user "$old_owner" -exec chown "$new_owner" {} \;
    
    echo "Migration terminée: $old_owner -> $new_owner"
}

# 3. Correction de propriété selon structure
fix_ownership_structure() {
    local base_dir="$1"
    local owner="$2"
    local group="${3:-$owner}"
    
    # Répertoires: propriétaire et groupe
    find "$base_dir" -type d -exec chown "$owner:$group" {} \;
    
    # Fichiers: même propriétaire
    find "$base_dir" -type f -exec chown "$owner:$group" {} \;
    
    # Liens symboliques: préserver
    find "$base_dir" -type l -exec chown -h "$owner:$group" {} \;
}
```

## Permissions spéciales (SUID, SGID, sticky bit)

### SUID (Set User ID) avancé

**Gestion sécurisée de SUID** :
```bash
#!/bin/bash
# Gestion sécurisée des fichiers SUID

# 1. Audit des fichiers SUID
audit_suid_files() {
    local search_dir="${1:-/}"
    
    echo "=== Audit des fichiers SUID ==="
    find "$search_dir" -type f -perm -4000 -ls | \
    while read -r line; do
        local file=$(echo "$line" | awk '{print $NF}')
        local owner=$(stat -c %U "$file")
        
        echo "Fichier: $file"
        echo "  Propriétaire: $owner"
        echo "  Permissions: $(stat -c %a "$file")"
        
        # Vérifier si le SUID est justifié
        if [ "$owner" = "root" ]; then
            echo "  ⚠️  SUID root - vérifier la nécessité"
        fi
        echo ""
    done
}

# 2. Retirer SUID non nécessaires
remove_unnecessary_suid() {
    local file="$1"
    
    if [ ! -f "$file" ]; then
        echo "Erreur: $file n'existe pas"
        return 1
    fi
    
    # Vérifier si SUID est présent
    if [ -u "$file" ]; then
        echo "Retrait du SUID de $file"
        chmod u-s "$file"
        
        # Tester si le programme fonctionne toujours
        if "$file" --version >/dev/null 2>&1; then
            echo "✓ Programme fonctionne sans SUID"
        else
            echo "⚠️  Le programme peut nécessiter SUID"
            chmod u+s "$file"  # Restaurer
        fi
    fi
}

# 3. Créer un wrapper SUID sécurisé
create_suid_wrapper() {
    local target_script="$1"
    local wrapper_name="${2:-$(basename "$target_script")_suid}"
    
    cat > "$wrapper_name" << 'EOF'
#!/bin/bash
# Wrapper SUID sécurisé

# Vérifier l'utilisateur appelant
CALLER=$(whoami)

# Liste des utilisateurs autorisés
AUTHORIZED_USERS=("user1" "user2")

# Vérifier autorisation
if [[ ! " ${AUTHORIZED_USERS[@]} " =~ " ${CALLER} " ]]; then
    echo "Accès refusé"
    exit 1
fi

# Exécuter le script cible avec les privilèges SUID
exec /path/to/target_script "$@"
EOF
    
    chmod 4750 "$wrapper_name"
    chown root:authorized_group "$wrapper_name"
}
```

### SGID (Set Group ID) avancé

**Utilisation avancée de SGID** :
```bash
#!/bin/bash
# Gestion avancée de SGID

# 1. Configuration SGID pour projets d'équipe
setup_team_project() {
    local project_dir="$1"
    local team_group="$2"
    
    # Créer le répertoire avec SGID
    mkdir -p "$project_dir"
    chmod 2775 "$project_dir"  # rwxrwxr-x + SGID
    chgrp "$team_group" "$project_dir"
    
    # Tous les fichiers créés auront le groupe de l'équipe
    echo "Répertoire de projet configuré avec SGID"
    echo "Groupe: $team_group"
    echo "Permissions: $(stat -c %a "$project_dir")"
}

# 2. Audit SGID
audit_sgid_files() {
    local search_dir="${1:-/}"
    
    echo "=== Audit des fichiers/répertoires SGID ==="
    find "$search_dir" -type f -perm -2000 -o -type d -perm -2000 | \
    while read -r item; do
        echo "Item: $item"
        echo "  Type: $(file "$item" | cut -d: -f2)"
        echo "  Groupe: $(stat -c %G "$item")"
        echo "  Permissions: $(stat -c %a "$item")"
        echo ""
    done
}

# 3. SGID pour services web
setup_web_service_sgid() {
    local web_dir="$1"
    local web_group="${2:-www-data}"
    
    # Répertoires avec SGID
    find "$web_dir" -type d -exec chmod 2775 {} \;
    find "$web_dir" -type d -exec chgrp "$web_group" {} \;
    
    # Fichiers avec groupe web
    find "$web_dir" -type f -exec chgrp "$web_group" {} \;
    find "$web_dir" -type f -exec chmod 664 {} \;
    
    # Scripts exécutables
    find "$web_dir" -name "*.sh" -o -name "*.php" -o -name "*.py" | \
    xargs chmod 775
}
```

### Sticky bit avancé

**Utilisation du sticky bit** :
```bash
#!/bin/bash
# Gestion avancée du sticky bit

# 1. Configuration de répertoires partagés
setup_shared_directory() {
    local shared_dir="$1"
    
    # Créer avec sticky bit
    mkdir -p "$shared_dir"
    chmod 1777 "$shared_dir"  # rwxrwxrwt
    
    echo "Répertoire partagé créé: $shared_dir"
    echo "Les utilisateurs peuvent créer des fichiers mais ne peuvent supprimer que les leurs"
}

# 2. Audit sticky bit
audit_sticky_bit() {
    local search_dir="${1:-/tmp}"
    
    echo "=== Audit des répertoires avec sticky bit ==="
    find "$search_dir" -type d -perm -1000 | \
    while read -r dir; do
        echo "Répertoire: $dir"
        echo "  Permissions: $(stat -c %a "$dir")"
        echo "  Propriétaire: $(stat -c %U "$dir")"
        echo ""
    done
}

# 3. Sticky bit pour répertoires de travail
setup_work_directories() {
    local base_dir="$1"
    
    # Créer des répertoires de travail avec sticky bit
    for dir in uploads temp cache; do
        local full_path="$base_dir/$dir"
        mkdir -p "$full_path"
        chmod 1777 "$full_path"
        echo "Configuré: $full_path avec sticky bit"
    done
}
```

## ACL (Access Control Lists)

### Installation et configuration

**Installation des outils ACL** :
```bash
#!/bin/bash
# Installation et configuration ACL

install_acl_tools() {
    # Linux (Debian/Ubuntu)
    if command -v apt >/dev/null 2>&1; then
        sudo apt install acl
    fi
    
    # Linux (Fedora/RHEL)
    if command -v dnf >/dev/null 2>&1; then
        sudo dnf install acl
    fi
    
    # Vérifier l'installation
    if command -v getfacl >/dev/null 2>&1; then
        echo "✓ ACL installé avec succès"
    else
        echo "✗ Échec de l'installation ACL"
        return 1
    fi
}

# Activer ACL sur un système de fichiers
enable_acl_on_filesystem() {
    local mount_point="$1"
    
    # Vérifier si ACL est activé
    if mount | grep "$mount_point" | grep -q acl; then
        echo "ACL déjà activé sur $mount_point"
        return 0
    fi
    
    # Activer ACL dans /etc/fstab
    echo "Ajouter 'acl' aux options de montage pour $mount_point dans /etc/fstab"
}
```

### Utilisation des ACL

**Commandes ACL de base** :
```bash
#!/bin/bash
# Utilisation des ACL

# 1. Afficher les ACL
show_acl() {
    local file="$1"
    
    echo "=== ACL pour $file ==="
    getfacl "$file"
    echo ""
}

# 2. Définir des ACL
set_acl() {
    local file="$1"
    local user="$2"
    local perms="$3"  # rwx, r-x, etc.
    
    # ACL pour un utilisateur spécifique
    setfacl -m "u:$user:$perms" "$file"
    
    echo "ACL défini: $user -> $perms sur $file"
}

# 3. ACL pour un groupe
set_group_acl() {
    local file="$1"
    local group="$2"
    local perms="$3"
    
    setfacl -m "g:$group:$perms" "$file"
}

# 4. ACL par défaut (héritage)
set_default_acl() {
    local dir="$1"
    local user="$2"
    local perms="$3"
    
    # ACL par défaut pour nouveaux fichiers
    setfacl -d -m "u:$user:$perms" "$dir"
    
    echo "ACL par défaut défini pour $dir"
}

# 5. ACL masque (masque maximum)
set_acl_mask() {
    local file="$1"
    local mask_perms="$2"  # rwx, r-x, etc.
    
    setfacl -m "m:$mask_perms" "$file"
}

# 6. Exemples pratiques
acl_examples() {
    local project_dir="/var/www/project"
    
    # Propriétaire: accès complet
    setfacl -m "u:www-data:rwx" "$project_dir"
    
    # Groupe développeurs: lecture/écriture
    setfacl -m "g:developers:rw-" "$project_dir"
    
    # Utilisateur spécifique: lecture seule
    setfacl -m "u:auditor:r--" "$project_dir"
    
    # ACL par défaut pour nouveaux fichiers
    setfacl -d -m "u:www-data:rwx" "$project_dir"
    setfacl -d -m "g:developers:rw-" "$project_dir"
    
    # Masque pour limiter les permissions maximales
    setfacl -m "m:rwx" "$project_dir"
}
```

### Gestion avancée des ACL

**Scripts de gestion ACL** :
```bash
#!/bin/bash
# Gestion avancée des ACL

# 1. Backup et restauration des ACL
backup_acl() {
    local target="$1"
    local backup_file="${2:-${target}.acl.backup}"
    
    getfacl -R "$target" > "$backup_file"
    echo "ACL sauvegardées dans $backup_file"
}

restore_acl() {
    local backup_file="$1"
    local target="${2:-.}"
    
    setfacl --restore="$backup_file"
    echo "ACL restaurées depuis $backup_file"
}

# 2. Audit des ACL
audit_acl() {
    local search_dir="${1:-.}"
    
    echo "=== Audit des ACL ==="
    find "$search_dir" -type f -exec getfacl {} \; 2>/dev/null | \
    grep -E "^[^#]" | sort | uniq -c | sort -rn
    
    echo ""
    echo "Fichiers avec ACL personnalisées:"
    find "$search_dir" -type f -exec sh -c 'getfacl "$1" | grep -q "^[^#].*:" && echo "$1"' _ {} \;
}

# 3. Nettoyage des ACL
cleanup_acl() {
    local target="$1"
    
    # Retirer toutes les ACL personnalisées
    find "$target" -type f -exec setfacl -b {} \;
    find "$target" -type d -exec setfacl -b {} \;
    
    echo "Toutes les ACL ont été retirées de $target"
}

# 4. Migration de permissions vers ACL
migrate_to_acl() {
    local file="$1"
    local target_user="$2"
    local target_group="${3:-}"
    
    # Obtenir les permissions actuelles
    local owner=$(stat -c %U "$file")
    local group=$(stat -c %G "$file")
    
    # Créer des ACL équivalentes
    setfacl -m "u:$owner:rwx" "$file"
    if [ -n "$target_group" ]; then
        setfacl -m "g:$target_group:rwx" "$file"
    fi
    
    echo "Permissions migrées vers ACL pour $file"
}
```

## Sécurité des fichiers et répertoires

### Attributs de sécurité

**Attributs étendus de sécurité** :
```bash
#!/bin/bash
# Gestion des attributs de sécurité

# 1. Attributs immuables (chattr)
make_immutable() {
    local file="$1"
    
    # Rendre le fichier immuable (même root ne peut le modifier)
    sudo chattr +i "$file"
    echo "$file est maintenant immuable"
}

remove_immutable() {
    local file="$1"
    
    sudo chattr -i "$file"
    echo "$file n'est plus immuable"
}

# 2. Attribut append-only (logs)
make_append_only() {
    local file="$1"
    
    # Seule l'ajout est autorisée, pas la modification
    sudo chattr +a "$file"
    echo "$file est maintenant append-only"
}

# 3. Attribut no-dump (sauvegarde)
prevent_backup() {
    local file="$1"
    
    # Exclure du dump système
    sudo chattr +d "$file"
    echo "$file sera exclu des sauvegardes système"
}

# 4. Liste des attributs
list_attributes() {
    local file="$1"
    
    lsattr "$file"
}

# 5. Script de sécurisation complète
secure_file() {
    local file="$1"
    
    # Permissions restrictives
    chmod 600 "$file"
    
    # Propriétaire root
    sudo chown root:root "$file"
    
    # Immutable si nécessaire
    if [ "$2" = "immutable" ]; then
        sudo chattr +i "$file"
    fi
    
    echo "Fichier sécurisé: $file"
}
```

### Sécurité des répertoires

**Sécurisation des répertoires** :
```bash
#!/bin/bash
# Sécurisation avancée des répertoires

# 1. Répertoire sécurisé pour fichiers sensibles
create_secure_directory() {
    local dir="$1"
    local owner="${2:-root}"
    
    mkdir -p "$dir"
    chmod 700 "$dir"  # Accès propriétaire uniquement
    chown "$owner:$owner" "$dir"
    
    # Sticky bit pour sécurité supplémentaire
    chmod +t "$dir"
    
    echo "Répertoire sécurisé créé: $dir"
}

# 2. Répertoire partagé sécurisé
create_shared_secure_directory() {
    local dir="$1"
    local group="$2"
    
    mkdir -p "$dir"
    chmod 2770 "$dir"  # SGID + groupe rwx
    chgrp "$group" "$dir"
    
    echo "Répertoire partagé sécurisé créé: $dir"
}

# 3. Vérification de sécurité des répertoires
check_directory_security() {
    local dir="$1"
    
    echo "=== Vérification de sécurité: $dir ==="
    
    # Vérifier les permissions
    local perms=$(stat -c %a "$dir")
    if [ "$perms" != "700" ] && [ "$perms" != "750" ] && [ "$perms" != "755" ]; then
        echo "⚠️  Permissions non standard: $perms"
    fi
    
    # Vérifier world-writable
    if [ -w "$dir" ] && [ ! -O "$dir" ]; then
        echo "⚠️  Répertoire modifiable par d'autres"
    fi
    
    # Vérifier les fichiers world-writable à l'intérieur
    find "$dir" -type f -perm -002 -ls
}
```

## Audit et monitoring des accès

### Audit avec auditd

**Configuration de l'audit système** :
```bash
#!/bin/bash
# Configuration de l'audit système

# 1. Installation auditd
install_auditd() {
    # Linux (Debian/Ubuntu)
    if command -v apt >/dev/null 2>&1; then
        sudo apt install auditd audispd-plugins
    fi
    
    # Linux (Fedora/RHEL)
    if command -v dnf >/dev/null 2>&1; then
        sudo dnf install audit
    fi
    
    # Démarrer le service
    sudo systemctl enable auditd
    sudo systemctl start auditd
}

# 2. Règles d'audit pour fichiers sensibles
audit_sensitive_files() {
    local file="$1"
    
    # Surveiller les accès au fichier
    sudo auditctl -w "$file" -p rwxa -k sensitive_file
    
    echo "Audit activé pour $file"
}

# 3. Audit des changements de permissions
audit_permission_changes() {
    # Surveiller chmod, chown, chgrp
    sudo auditctl -w /usr/bin/chmod -p x -k permission_change
    sudo auditctl -w /usr/bin/chown -p x -k permission_change
    sudo auditctl -w /usr/bin/chgrp -p x -k permission_change
    
    echo "Audit des changements de permissions activé"
}

# 4. Consultation des logs d'audit
view_audit_logs() {
    local search_key="${1:-}"
    
    if [ -n "$search_key" ]; then
        sudo ausearch -k "$search_key"
    else
        sudo ausearch -ts today
    fi
}

# 5. Génération de rapports d'audit
generate_audit_report() {
    local output_file="${1:-audit_report.txt}"
    
    sudo aureport --summary > "$output_file"
    sudo aureport -f >> "$output_file"
    
    echo "Rapport d'audit généré: $output_file"
}
```

### Monitoring des accès

**Scripts de monitoring** :
```bash
#!/bin/bash
# Monitoring des accès aux fichiers

# 1. Surveiller les accès en temps réel
monitor_file_access() {
    local file="$1"
    
    # Utiliser inotify pour surveiller
    if command -v inotifywait >/dev/null 2>&1; then
        inotifywait -m -e access,modify,open,close "$file" | \
        while read -r event; do
            echo "[$(date)] $event"
        done
    else
        echo "inotify-tools non installé"
        echo "Installer avec: sudo apt install inotify-tools"
    fi
}

# 2. Log des accès aux fichiers sensibles
log_sensitive_access() {
    local file="$1"
    local log_file="${2:-/var/log/file_access.log}"
    
    # Créer un wrapper qui log les accès
    local wrapper="/usr/local/bin/secure_$(basename "$file")"
    
    cat > "$wrapper" << EOF
#!/bin/bash
# Wrapper de sécurité pour $file

echo "\$(date): Accès à $file par \$(whoami)" >> "$log_file"
exec "$file" "\$@"
EOF
    
    chmod 755 "$wrapper"
    echo "Wrapper créé: $wrapper"
}

# 3. Statistiques d'accès
file_access_stats() {
    local file="$1"
    local log_file="${2:-/var/log/file_access.log}"
    
    if [ -f "$log_file" ]; then
        echo "=== Statistiques d'accès pour $file ==="
        echo "Total d'accès: $(grep -c "$file" "$log_file")"
        echo "Par utilisateur:"
        grep "$file" "$log_file" | awk '{print $NF}' | sort | uniq -c | sort -rn
    else
        echo "Fichier de log non trouvé: $log_file"
    fi
}
```

## Sécurité avancée : SELinux et AppArmor

### SELinux

**Gestion SELinux** :
```bash
#!/bin/bash
# Gestion SELinux

# 1. Vérifier le statut SELinux
check_selinux_status() {
    getenforce
    sestatus
}

# 2. Changer le contexte de sécurité
set_selinux_context() {
    local file="$1"
    local context="$2"  # Ex: httpd_sys_content_t
    
    sudo chcon -t "$context" "$file"
    echo "Contexte SELinux défini pour $file"
}

# 3. Restaurer le contexte par défaut
restore_default_context() {
    local path="$1"
    
    sudo restorecon -R "$path"
    echo "Contexte par défaut restauré pour $path"
}

# 4. Politique SELinux personnalisée
create_selinux_policy() {
    local module_name="$1"
    local te_file="${2:-${module_name}.te}"
    
    cat > "$te_file" << EOF
module ${module_name} 1.0;

require {
    type httpd_t;
    type httpd_sys_content_t;
    class file { read write };
}

# Allow httpd to read/write custom files
allow httpd_t httpd_sys_content_t:file { read write };
EOF
    
    echo "Fichier de politique créé: $te_file"
    echo "Compiler avec: checkmodule -M -m -o ${module_name}.mod $te_file"
}
```

### AppArmor

**Gestion AppArmor** :
```bash
#!/bin/bash
# Gestion AppArmor

# 1. Statut AppArmor
check_apparmor_status() {
    sudo aa-status
}

# 2. Charger un profil
load_apparmor_profile() {
    local profile="$1"
    
    sudo apparmor_parser -r "$profile"
    echo "Profil chargé: $profile"
}

# 3. Créer un profil avec aa-genprof
generate_apparmor_profile() {
    local program="$1"
    
    echo "Génération du profil pour $program"
    echo "Exécuter: sudo aa-genprof $program"
    echo "Puis suivre les instructions interactives"
}

# 4. Mode d'application
set_apparmor_mode() {
    local profile="$1"
    local mode="${2:-enforce}"  # enforce ou complain
    
    sudo aa-enforce "$profile" 2>/dev/null || sudo aa-complain "$profile"
    echo "Profil $profile en mode $mode"
}
```

## Attributs étendus et capabilities

### Linux Capabilities

**Gestion des capabilities** :
```bash
#!/bin/bash
# Gestion des Linux Capabilities

# 1. Lister les capabilities d'un processus
list_process_capabilities() {
    local pid="$1"
    
    cat "/proc/$pid/status" | grep Cap
}

# 2. Définir des capabilities sur un fichier
set_file_capabilities() {
    local file="$1"
    local caps="$2"  # Ex: cap_net_bind_service+ep
    
    sudo setcap "$caps" "$file"
    echo "Capabilities définies pour $file"
}

# 3. Exemples de capabilities utiles
capability_examples() {
    # Permettre à un programme non-root d'écouter sur port < 1024
    sudo setcap 'cap_net_bind_service=+ep' /usr/bin/my_web_server
    
    # Permettre l'accès réseau raw (ping sans setuid)
    sudo setcap 'cap_net_raw=+ep' /usr/bin/ping
    
    # Permettre la modification de la date système
    sudo setcap 'cap_sys_time=+ep' /usr/bin/set_date
}

# 4. Audit des capabilities
audit_capabilities() {
    echo "=== Fichiers avec capabilities ==="
    getcap -r / 2>/dev/null
}
```

## Bonnes pratiques de sécurité

### Checklist de sécurité

**Script de vérification de sécurité** :
```bash
#!/bin/bash
# Checklist de sécurité complète

security_checklist() {
    local target="${1:-.}"
    
    echo "=== Checklist de sécurité pour $target ==="
    echo ""
    
    # 1. Fichiers world-writable
    echo "1. Fichiers world-writable:"
    find "$target" -type f -perm -002 -ls
    echo ""
    
    # 2. Répertoires world-writable
    echo "2. Répertoires world-writable:"
    find "$target" -type d -perm -002 -ls
    echo ""
    
    # 3. Fichiers SUID/SGID
    echo "3. Fichiers SUID/SGID:"
    find "$target" -type f \( -perm -4000 -o -perm -2000 \) -ls
    echo ""
    
    # 4. Fichiers sans propriétaire
    echo "4. Fichiers sans propriétaire:"
    find "$target" -nouser -ls
    echo ""
    
    # 5. Fichiers sans groupe
    echo "5. Fichiers sans groupe:"
    find "$target" -nogroup -ls
    echo ""
    
    # 6. Fichiers avec permissions inhabituelles
    echo "6. Fichiers avec permissions 777:"
    find "$target" -type f -perm 777 -ls
    echo ""
    
    # 7. Fichiers de configuration sensibles
    echo "7. Fichiers de configuration sensibles:"
    find "$target" -name "*.key" -o -name "*.pem" -o -name "*password*" | \
    while read -r file; do
        local perms=$(stat -c %a "$file")
        if [ "$perms" != "600" ]; then
            echo "  ⚠️  $file: permissions $perms (devrait être 600)"
        fi
    done
}

# Correction automatique
auto_fix_security() {
    local target="${1:-.}"
    
    echo "=== Correction automatique de sécurité ==="
    
    # Corriger les fichiers de clés
    find "$target" \( -name "*.key" -o -name "*.pem" \) -exec chmod 600 {} \;
    
    # Retirer world-writable des fichiers
    find "$target" -type f -perm -002 -exec chmod o-w {} \;
    
    echo "Corrections appliquées"
}
```

## Scripts de sécurité automatisés

### Script complet de sécurisation

**Script de sécurisation automatique** :
```bash
#!/bin/bash
# Script complet de sécurisation

set -euo pipefail

secure_directory() {
    local dir="$1"
    local owner="${2:-$USER}"
    local group="${3:-$USER}"
    
    echo "=== Sécurisation de $dir ==="
    
    # 1. Permissions de base
    find "$dir" -type d -exec chmod 755 {} \;
    find "$dir" -type f -exec chmod 644 {} \;
    
    # 2. Propriété
    chown -R "$owner:$group" "$dir"
    
    # 3. Fichiers exécutables
    find "$dir" -name "*.sh" -exec chmod 755 {} \;
    
    # 4. Fichiers sensibles
    find "$dir" \( -name "*.key" -o -name "*.pem" \) -exec chmod 600 {} \;
    
    # 5. Répertoires sensibles
    find "$dir" -type d -name "*secret*" -exec chmod 700 {} \;
    
    echo "✓ Sécurisation terminée"
}

# Utilisation
# secure_directory "/var/www/myapp" "www-data" "www-data"
```

## Conclusion

La maîtrise avancée des permissions et de la sécurité Linux nécessite une compréhension approfondie des mécanismes de contrôle d'accès, des ACL, des attributs étendus, et des systèmes de sécurité avancés comme SELinux et AppArmor. En combinant ces outils avec des pratiques de sécurité rigoureuses et des scripts d'automatisation, vous créez un environnement sécurisé et maintenable.

Les permissions ne sont pas seulement un mécanisme technique, mais une philosophie de sécurité qui doit être appliquée de manière cohérente à travers tout le système. Un système bien sécurisé est un système où chaque accès est justifié, tracé, et contrôlé.

Dans le chapitre suivant, nous explorerons la gestion avancée des processus, découvrant comment surveiller, contrôler et optimiser l'exécution des programmes dans un environnement Linux.
