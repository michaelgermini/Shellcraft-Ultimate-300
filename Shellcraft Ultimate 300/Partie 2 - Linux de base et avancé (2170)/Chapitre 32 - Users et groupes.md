# Chapitre 32 - Users et groupes

## Table des matières
- [Introduction](#introduction)
- [Système d'utilisateurs UNIX](#système-dutilisateurs-unix)
- [Fichiers de configuration système](#fichiers-de-configuration-système)
- [Gestion des comptes utilisateurs](#gestion-des-comptes-utilisateurs)
- [Gestion des groupes](#gestion-des-groupes)
- [Mots de passe et sécurité](#mots-de-passe-et-sécurité)
- [Authentification et PAM](#authentification-et-pam)
- [sudo et élévation de privilèges](#sudo-et-élévation-de-privilèges)
- [Profils utilisateurs et environnements](#profils-utilisateurs-et-environnements)
- [Administration multi-utilisateurs](#administration-multi-utilisateurs)
- [Audit et conformité](#audit-et-conformité)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

La gestion des utilisateurs et groupes constitue le fondement de la sécurité et de l'organisation dans les systèmes multi-utilisateurs. Chaque utilisateur représente une identité distincte avec ses propres permissions, son environnement personnalisé, et ses responsabilités spécifiques.

Imaginez un grand bâtiment d'appartements : chaque résident a sa propre clé pour son appartement, mais partage certains espaces communs avec d'autres. Les groupes permettent de définir ces espaces partagés et de gérer collectivement les accès aux ressources communes. Dans un système Linux, cette métaphore s'étend : chaque utilisateur a son propre répertoire home, ses propres fichiers de configuration, et ses propres permissions, tandis que les groupes permettent de partager des ressources de manière contrôlée et sécurisée.

## Système d'utilisateurs UNIX

### Concepts fondamentaux

**Utilisateur** :
- Identité unique dans le système
- Identifié par un nom d'utilisateur (username) et un UID (User ID)
- Possède un répertoire home, un shell par défaut, et des groupes d'appartenance

**Groupe** :
- Collection d'utilisateurs partageant des permissions communes
- Identifié par un nom de groupe et un GID (Group ID)
- Permet de gérer les accès collectifs aux ressources

**UID et GID** :
```bash
#!/bin/bash
# Comprendre UID et GID

# Afficher les informations de l'utilisateur courant
id
# uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo),1001(developers)

# UID de l'utilisateur courant
echo "UID: $(id -u)"
echo "GID: $(id -g)"
echo "Groupes: $(id -Gn)"

# Plages d'UID/GID typiques
# 0 : root
# 1-99 : Système (statiques)
# 100-999 : Système (dynamiques)
# 1000+ : Utilisateurs normaux
```

### Types d'utilisateurs

**Utilisateur root** :
```bash
# Root : UID 0, privilèges complets
id root
# uid=0(root) gid=0(root) groups=0(root)

# Vérifier si on est root
[ "$(id -u)" -eq 0 ] && echo "Root" || echo "Utilisateur normal"
```

**Utilisateurs système** :
```bash
# Utilisateurs système (UID < 1000)
# Exemples : www-data, mysql, postgres, daemon
getent passwd | awk -F: '$3 < 1000 {print $1, $3}'
```

**Utilisateurs normaux** :
```bash
# Utilisateurs normaux (UID >= 1000)
getent passwd | awk -F: '$3 >= 1000 {print $1, $3}'
```

## Fichiers de configuration système

### /etc/passwd

**Structure du fichier passwd** :
```bash
#!/bin/bash
# Analyser /etc/passwd

# Format : username:password:x:UID:GID:comment:home:shell
# Exemple : alice:x:1000:1000:Alice User:/home/alice:/bin/bash

# Afficher tous les utilisateurs
cat /etc/passwd

# Afficher un utilisateur spécifique
getent passwd alice

# Extraire seulement les noms d'utilisateurs
cut -d: -f1 /etc/passwd

# Lister avec UID et shell
awk -F: '{print $1, $3, $7}' /etc/passwd

# Compter les utilisateurs
wc -l /etc/passwd
```

**Champs du fichier passwd** :
```bash
#!/bin/bash
# Analyser les champs de passwd

analyze_passwd_entry() {
    local username="$1"
    local entry=$(getent passwd "$username")
    
    IFS=':' read -r username password uid gid comment home shell <<< "$entry"
    
    echo "=== Utilisateur: $username ==="
    echo "UID: $uid"
    echo "GID: $gid"
    echo "Commentaire: $comment"
    echo "Home: $home"
    echo "Shell: $shell"
    echo "Mot de passe: $( [ "$password" = "x" ] && echo "Dans /etc/shadow" || echo "Ancien format")"
}

# Utilisation
analyze_passwd_entry "$USER"
```

### /etc/shadow

**Structure du fichier shadow** :
```bash
#!/bin/bash
# Analyser /etc/shadow (nécessite sudo)

# Format : username:password:last_change:min:max:warn:inactive:expire:reserved
# Le mot de passe est hashé (MD5, SHA-256, SHA-512, etc.)

# Afficher les informations d'un utilisateur (nécessite sudo)
sudo getent shadow alice

# Vérifier si un utilisateur a un mot de passe défini
check_password_set() {
    local username="$1"
    local entry=$(sudo getent shadow "$username" 2>/dev/null)
    
    if [ -z "$entry" ]; then
        echo "Utilisateur non trouvé"
        return 1
    fi
    
    local password=$(echo "$entry" | cut -d: -f2)
    
    if [ "$password" = "*" ] || [ "$password" = "!" ]; then
        echo "Pas de mot de passe défini"
        return 1
    else
        echo "Mot de passe défini"
        return 0
    fi
}
```

### /etc/group

**Structure du fichier group** :
```bash
#!/bin/bash
# Analyser /etc/group

# Format : groupname:password:GID:members
# Exemple : developers:x:1001:alice,bob,charlie

# Afficher tous les groupes
cat /etc/group

# Afficher un groupe spécifique
getent group developers

# Lister les membres d'un groupe
getent group developers | cut -d: -f4

# Trouver les groupes d'un utilisateur
groups alice
id -Gn alice

# Compter les groupes
wc -l /etc/group
```

## Gestion des comptes utilisateurs

### useradd : créer un utilisateur

**Création de base** :
```bash
#!/bin/bash
# Création d'utilisateur avec useradd

# Création simple
sudo useradd alice

# Création avec options complètes
sudo useradd -m -d /home/alice -s /bin/bash -c "Alice User" -G developers,sudo alice

# Options principales :
# -m : Créer le répertoire home
# -d : Spécifier le répertoire home
# -s : Spécifier le shell
# -c : Commentaire (nom complet)
# -G : Groupes secondaires
# -g : Groupe primaire
# -u : UID spécifique
# -e : Date d'expiration
# -f : Jours avant désactivation après expiration du mot de passe
```

**Création avancée** :
```bash
#!/bin/bash
# Création avancée d'utilisateur

create_user_advanced() {
    local username="$1"
    local full_name="$2"
    local groups="${3:-}"
    local shell="${4:-/bin/bash}"
    local uid="${5:-}"  # Auto si vide
    
    local useradd_cmd="sudo useradd"
    
    [ -n "$full_name" ] && useradd_cmd+=" -c \"$full_name\""
    [ -n "$shell" ] && useradd_cmd+=" -s $shell"
    [ -n "$uid" ] && useradd_cmd+=" -u $uid"
    [ -n "$groups" ] && useradd_cmd+=" -G $groups"
    
    useradd_cmd+=" -m $username"
    
    echo "Création de l'utilisateur: $username"
    eval "$useradd_cmd"
    
    # Définir un mot de passe initial
    echo "Définition du mot de passe..."
    sudo passwd "$username"
    
    echo "Utilisateur créé avec succès"
}

# Utilisation
# create_user_advanced "bob" "Bob Smith" "developers,sudo" "/bin/bash"
```

### usermod : modifier un utilisateur

**Modifications courantes** :
```bash
#!/bin/bash
# Modification d'utilisateur avec usermod

# Changer le shell
sudo usermod -s /bin/zsh alice

# Changer le répertoire home (et déplacer les fichiers)
sudo usermod -m -d /new/home/alice alice

# Ajouter à des groupes (sans retirer les autres)
sudo usermod -aG sudo,developers alice

# Changer le groupe primaire
sudo usermod -g developers alice

# Changer le commentaire (nom complet)
sudo usermod -c "Alice Johnson" alice

# Verrouiller un compte
sudo usermod -L alice  # Lock

# Déverrouiller un compte
sudo usermod -U alice  # Unlock

# Définir une date d'expiration
sudo usermod -e 2024-12-31 alice

# Désactiver après expiration du mot de passe
sudo usermod -f 30 alice  # 30 jours après expiration
```

**Script de modification complète** :
```bash
#!/bin/bash
# Script de modification complète

modify_user() {
    local username="$1"
    shift
    local options="$@"
    
    # Vérifier que l'utilisateur existe
    if ! id "$username" &>/dev/null; then
        echo "Erreur: Utilisateur $username n'existe pas"
        return 1
    fi
    
    echo "Modification de l'utilisateur: $username"
    sudo usermod $options "$username"
    
    if [ $? -eq 0 ]; then
        echo "Utilisateur modifié avec succès"
        # Afficher les nouvelles informations
        id "$username"
    else
        echo "Erreur lors de la modification"
        return 1
    fi
}
```

### userdel : supprimer un utilisateur

**Suppression d'utilisateur** :
```bash
#!/bin/bash
# Suppression d'utilisateur

# Suppression simple (garde le home)
sudo userdel alice

# Suppression avec home et mail
sudo userdel -r alice

# Suppression forcée (même si utilisateur connecté)
sudo userdel -f alice

# Script de suppression sécurisée
safe_delete_user() {
    local username="$1"
    
    # Vérifier que l'utilisateur existe
    if ! id "$username" &>/dev/null; then
        echo "Utilisateur $username n'existe pas"
        return 1
    fi
    
    # Vérifier les processus actifs
    local processes=$(pgrep -u "$username" | wc -l)
    if [ "$processes" -gt 0 ]; then
        echo "⚠️  $processes processus actifs pour $username"
        echo "Arrêt des processus..."
        sudo pkill -u "$username"
        sleep 2
    fi
    
    # Sauvegarder le home avant suppression
    if [ -d "/home/$username" ]; then
        echo "Sauvegarde du répertoire home..."
        sudo tar -czf "/backup/${username}_home_$(date +%Y%m%d).tar.gz" \
            -C /home "$username"
    fi
    
    # Supprimer l'utilisateur
    echo "Suppression de l'utilisateur..."
    sudo userdel -r "$username"
    
    echo "Utilisateur supprimé avec succès"
}
```

## Gestion des groupes

### groupadd : créer un groupe

**Création de groupes** :
```bash
#!/bin/bash
# Création de groupes

# Création simple
sudo groupadd developers

# Création avec GID spécifique
sudo groupadd -g 1001 developers

# Création système (GID < 1000)
sudo groupadd -r system_group

# Script de création avec vérification
create_group() {
    local groupname="$1"
    local gid="${2:-}"
    
    # Vérifier si le groupe existe
    if getent group "$groupname" >/dev/null 2>&1; then
        echo "Le groupe $groupname existe déjà"
        return 1
    fi
    
    local cmd="sudo groupadd"
    [ -n "$gid" ] && cmd+=" -g $gid"
    cmd+=" $groupname"
    
    eval "$cmd"
    
    if [ $? -eq 0 ]; then
        echo "Groupe créé: $groupname"
        getent group "$groupname"
    else
        echo "Erreur lors de la création du groupe"
        return 1
    fi
}
```

### groupmod : modifier un groupe

**Modification de groupes** :
```bash
#!/bin/bash
# Modification de groupes

# Changer le nom du groupe
sudo groupmod -n new_name old_name

# Changer le GID
sudo groupmod -g 2000 developers

# Script de modification
modify_group() {
    local old_name="$1"
    local new_name="${2:-}"
    local new_gid="${3:-}"
    
    if ! getent group "$old_name" >/dev/null 2>&1; then
        echo "Groupe $old_name n'existe pas"
        return 1
    fi
    
    local cmd="sudo groupmod"
    [ -n "$new_name" ] && cmd+=" -n $new_name"
    [ -n "$new_gid" ] && cmd+=" -g $new_gid"
    cmd+=" $old_name"
    
    eval "$cmd"
}
```

### groupdel : supprimer un groupe

**Suppression de groupes** :
```bash
#!/bin/bash
# Suppression de groupes

# Suppression simple
sudo groupdel developers

# Vérifier avant suppression
safe_delete_group() {
    local groupname="$1"
    
    # Vérifier que le groupe existe
    if ! getent group "$groupname" >/dev/null 2>&1; then
        echo "Groupe $groupname n'existe pas"
        return 1
    fi
    
    # Vérifier les membres
    local members=$(getent group "$groupname" | cut -d: -f4)
    if [ -n "$members" ]; then
        echo "⚠️  Le groupe a des membres: $members"
        echo "Voulez-vous continuer ? (y/N)"
        read -r response
        if [ "$response" != "y" ]; then
            return 1
        fi
    fi
    
    # Vérifier le groupe primaire
    local gid=$(getent group "$groupname" | cut -d: -f3)
    if getent passwd | cut -d: -f4 | grep -q "^${gid}$"; then
        echo "⚠️  Ce groupe est le groupe primaire d'utilisateurs"
        echo "Suppression annulée"
        return 1
    fi
    
    sudo groupdel "$groupname"
    echo "Groupe supprimé"
}
```

### Gestion des membres de groupe

**Ajouter/retirer des membres** :
```bash
#!/bin/bash
# Gestion des membres de groupe

# Ajouter un utilisateur à un groupe
sudo usermod -aG developers alice

# Retirer un utilisateur d'un groupe (nécessite modification manuelle)
remove_from_group() {
    local username="$1"
    local groupname="$2"
    
    # Obtenir les groupes actuels
    local current_groups=$(id -Gn "$username" | tr ' ' ',')
    
    # Retirer le groupe spécifié
    local new_groups=$(echo "$current_groups" | sed "s/,$groupname,/,/g" | sed "s/^$groupname,//" | sed "s/,$groupname$//")
    
    # Appliquer les changements
    sudo usermod -G "$new_groups" "$username"
}

# Lister les membres d'un groupe
list_group_members() {
    local groupname="$1"
    
    echo "=== Membres du groupe $groupname ==="
    getent group "$groupname" | cut -d: -f4 | tr ',' '\n'
    
    # Aussi les utilisateurs avec ce groupe comme primaire
    local gid=$(getent group "$groupname" | cut -d: -f3)
    getent passwd | awk -F: -v gid="$gid" '$4 == gid {print $1}'
}

# Ajouter plusieurs utilisateurs à un groupe
add_users_to_group() {
    local groupname="$1"
    shift
    local users="$@"
    
    for user in $users; do
        sudo usermod -aG "$groupname" "$user"
        echo "Ajouté $user à $groupname"
    done
}
```

## Mots de passe et sécurité

### passwd : gestion des mots de passe

**Changement de mot de passe** :
```bash
#!/bin/bash
# Gestion des mots de passe

# Changer son propre mot de passe
passwd

# Changer le mot de passe d'un autre utilisateur (root)
sudo passwd alice

# Forcer le changement au prochain login
sudo passwd -e alice  # Expire immédiatement

# Verrouiller un compte
sudo passwd -l alice  # Lock

# Déverrouiller un compte
sudo passwd -u alice  # Unlock

# Supprimer le mot de passe (dangereux)
sudo passwd -d alice  # Delete password

# Définir un mot de passe non expirable
sudo passwd -x -1 alice  # -1 = jamais
```

### chage : gestion de l'expiration

**Configuration de l'expiration** :
```bash
#!/bin/bash
# Gestion de l'expiration avec chage

# Afficher les informations d'expiration
sudo chage -l alice

# Définir l'expiration du mot de passe
sudo chage -M 90 alice  # Expire dans 90 jours

# Définir le minimum entre changements
sudo chage -m 7 alice  # Minimum 7 jours entre changements

# Définir l'avertissement
sudo chage -W 7 alice  # Avertir 7 jours avant expiration

# Définir l'inactivité
sudo chage -I 30 alice  # Désactiver après 30 jours d'inactivité

# Définir la date d'expiration du compte
sudo chage -E 2024-12-31 alice

# Forcer le changement au prochain login
sudo chage -d 0 alice

# Script de configuration complète
configure_password_policy() {
    local username="$1"
    local max_days="${2:-90}"
    local min_days="${3:-7}"
    local warn_days="${4:-7}"
    
    sudo chage -M "$max_days" "$username"
    sudo chage -m "$min_days" "$username"
    sudo chage -W "$warn_days" "$username"
    
    echo "Politique de mot de passe configurée pour $username"
    sudo chage -l "$username"
}
```

### Politiques de mots de passe

**Configuration système** :
```bash
#!/bin/bash
# Configuration des politiques de mots de passe

# Fichier de configuration : /etc/login.defs
# PASS_MAX_DAYS : Durée maximale
# PASS_MIN_DAYS : Durée minimale
# PASS_WARN_AGE : Avertissement

# Vérifier la configuration actuelle
grep -E "^PASS_" /etc/login.defs

# Fichier PAM : /etc/pam.d/common-password
# Contrôle la complexité des mots de passe

# Installer libpam-pwquality pour politiques avancées
# sudo apt install libpam-pwquality

# Configuration dans /etc/security/pwquality.conf
# minlen : Longueur minimale
# dcredit : Chiffres requis
# ucredit : Majuscules requises
# lcredit : Minuscules requises
# ocredit : Caractères spéciaux requis
```

## Authentification et PAM

### Pluggable Authentication Modules

**Comprendre PAM** :
```bash
#!/bin/bash
# Introduction à PAM

# Fichiers de configuration PAM
ls /etc/pam.d/

# Analyser la configuration pour login
cat /etc/pam.d/login

# Types de modules PAM :
# auth : Authentification
# account : Gestion de compte
# password : Changement de mot de passe
# session : Gestion de session

# Contrôles PAM :
# required : Doit réussir, continue même en cas d'échec
# requisite : Doit réussir, arrête immédiatement en cas d'échec
# sufficient : Suffisant si réussit
# optional : Optionnel
```

### Configuration PAM avancée

**Exemples de configuration** :
```bash
#!/bin/bash
# Configuration PAM

# Exemple : /etc/pam.d/common-auth
# auth required pam_unix.so nullok_secure

# Exemple : /etc/pam.d/common-account
# account required pam_unix.so

# Exemple : /etc/pam.d/common-password
# password requisite pam_pwquality.so retry=3
# password required pam_unix.so sha512 shadow nullok use_authtok

# Vérifier la configuration PAM
pam-auth-update
```

## sudo et élévation de privilèges

### Configuration sudo

**Fichier /etc/sudoers** :
```bash
#!/bin/bash
# Configuration sudo

# Toujours utiliser visudo pour éditer sudoers
sudo visudo

# Format de base :
# username ALL=(ALL:ALL) ALL
# %group ALL=(ALL:ALL) ALL

# Exemples de règles :
# alice ALL=(ALL) ALL  # Tous les privilèges
# bob ALL=(ALL) NOPASSWD: ALL  # Sans mot de passe
# charlie ALL=(www-data) /usr/bin/systemctl restart apache2  # Commande spécifique
# developers ALL=(ALL) /usr/bin/apt, /usr/bin/dpkg  # Commandes limitées

# Vérifier les privilèges sudo d'un utilisateur
sudo -l -U alice
```

### Gestion avancée sudo

**Scripts de gestion sudo** :
```bash
#!/bin/bash
# Gestion avancée de sudo

# Ajouter un utilisateur au groupe sudo
add_sudo_access() {
    local username="$1"
    sudo usermod -aG sudo "$username"
    echo "$username ajouté au groupe sudo"
}

# Créer une règle sudo personnalisée
create_sudo_rule() {
    local username="$1"
    local commands="$2"
    local nopasswd="${3:-false}"
    
    local rule="$username ALL=(ALL)"
    [ "$nopasswd" = "true" ] && rule+=" NOPASSWD:"
    rule+=" $commands"
    
    echo "$rule" | sudo tee -a /etc/sudoers.d/"${username}_custom"
    sudo chmod 0440 /etc/sudoers.d/"${username}_custom"
    
    echo "Règle sudo créée pour $username"
}

# Vérifier la syntaxe sudoers
check_sudoers_syntax() {
    sudo visudo -c
}

# Lister tous les utilisateurs avec sudo
list_sudo_users() {
    echo "=== Utilisateurs avec accès sudo ==="
    getent group sudo | cut -d: -f4 | tr ',' '\n'
    
    # Aussi ceux avec règles personnalisées
    grep -h "^[^#]" /etc/sudoers.d/* 2>/dev/null | \
    awk '{print $1}' | grep -v "^%" | sort -u
}
```

## Profils utilisateurs et environnements

### Fichiers de profil

**Fichiers de configuration** :
```bash
#!/bin/bash
# Fichiers de profil utilisateur

# ~/.bashrc : Configuration Bash (non-login)
# ~/.bash_profile : Configuration Bash (login)
# ~/.profile : Configuration shell générique
# ~/.zshrc : Configuration Zsh
# ~/.vimrc : Configuration Vim
# ~/.gitconfig : Configuration Git

# Ordre de chargement pour Bash :
# Login shell : /etc/profile → ~/.bash_profile → ~/.profile → ~/.bashrc
# Non-login shell : ~/.bashrc

# Créer un profil personnalisé
create_user_profile() {
    local username="$1"
    local home_dir="/home/$username"
    
    # Créer .bashrc de base
    cat > "$home_dir/.bashrc" << 'EOF'
# ~/.bashrc
export EDITOR=vim
export PATH="$HOME/bin:$PATH"
alias ll='ls -alF'
alias la='ls -A'
alias l='ls -CF'
EOF
    
    chown "$username:$username" "$home_dir/.bashrc"
    echo "Profil créé pour $username"
}
```

### Environnements personnalisés

**Configuration d'environnement** :
```bash
#!/bin/bash
# Configuration d'environnement utilisateur

# Variables d'environnement personnalisées
setup_user_environment() {
    local username="$1"
    local home_dir="/home/$username"
    
    cat >> "$home_dir/.bashrc" << 'EOF'

# Variables personnalisées
export PROJECTS_DIR="$HOME/projects"
export SCRIPTS_DIR="$HOME/scripts"
export EDITOR=vim

# Fonctions personnalisées
mkcd() {
    mkdir -p "$1" && cd "$1"
}

# Alias personnalisés
alias projects='cd $PROJECTS_DIR'
alias scripts='cd $SCRIPTS_DIR'
EOF
    
    chown "$username:$username" "$home_dir/.bashrc"
}
```

## Administration multi-utilisateurs

### Scripts de gestion en masse

**Gestion de plusieurs utilisateurs** :
```bash
#!/bin/bash
# Gestion en masse d'utilisateurs

# Créer plusieurs utilisateurs depuis un fichier
create_users_from_file() {
    local user_file="$1"
    
    while IFS=: read -r username full_name groups shell; do
        echo "Création de $username..."
        
        sudo useradd -m -c "$full_name" -s "${shell:-/bin/bash}" "$username"
        
        if [ -n "$groups" ]; then
            sudo usermod -aG "$groups" "$username"
        fi
        
        # Définir un mot de passe temporaire
        echo "$username:TempPass123!" | sudo chpasswd
        
        # Forcer le changement au prochain login
        sudo chage -d 0 "$username"
        
        echo "✓ $username créé"
    done < "$user_file"
}

# Format du fichier : username:full_name:groups:shell
# Exemple :
# alice:Alice Smith:developers,sudo:/bin/bash
# bob:Bob Jones:developers:/bin/zsh

# Supprimer plusieurs utilisateurs
delete_users_from_list() {
    local user_list="$1"
    
    while read -r username; do
        echo "Suppression de $username..."
        sudo userdel -r "$username" 2>/dev/null && \
            echo "✓ $username supprimé" || \
            echo "✗ Erreur pour $username"
    done < "$user_list"
}
```

### Synchronisation avec LDAP/Active Directory

**Intégration LDAP** :
```bash
#!/bin/bash
# Intégration LDAP (basique)

# Installer les outils LDAP
# sudo apt install libnss-ldap libpam-ldap ldap-utils

# Configuration dans /etc/nsswitch.conf
# passwd: files ldap
# group: files ldap
# shadow: files ldap

# Configuration dans /etc/ldap/ldap.conf
# BASE dc=example,dc=com
# URI ldap://ldap.example.com

# Rechercher un utilisateur LDAP
ldapsearch -x -H ldap://ldap.example.com -b "dc=example,dc=com" "(uid=alice)"
```

## Audit et conformité

### Scripts d'audit

**Audit des utilisateurs** :
```bash
#!/bin/bash
# Scripts d'audit

# Audit complet des utilisateurs
audit_users() {
    local report_file="${1:-/tmp/user_audit_$(date +%Y%m%d).txt}"
    
    {
        echo "=== Audit des utilisateurs ==="
        echo "Date: $(date)"
        echo ""
        
        echo "=== Utilisateurs sans mot de passe ==="
        sudo awk -F: '($2 == "" || $2 == "*" || $2 == "!") {print $1}' /etc/shadow
        
        echo ""
        echo "=== Utilisateurs avec accès sudo ==="
        getent group sudo | cut -d: -f4 | tr ',' '\n'
        
        echo ""
        echo "=== Comptes expirés ==="
        sudo chage -l $(cut -d: -f1 /etc/passwd) 2>/dev/null | \
        grep -B1 "Account expires" | grep -E "Account expires|^[^:]*:" | \
        awk '/Account expires/ {if ($3 == "never") next; print prev} {prev=$0}'
        
        echo ""
        echo "=== Utilisateurs inactifs ==="
        # Utilisateurs sans login récent (nécessite lastlog)
        lastlog | awk 'NR>1 && $2 == "**Never" {print $1}'
        
        echo ""
        echo "=== Utilisateurs système vs normaux ==="
        echo "Système (UID < 1000):"
        getent passwd | awk -F: '$3 < 1000 {print $1, $3}' | wc -l
        echo "Normaux (UID >= 1000):"
        getent passwd | awk -F: '$3 >= 1000 {print $1, $3}' | wc -l
        
    } > "$report_file"
    
    echo "Rapport généré: $report_file"
}

# Audit des groupes
audit_groups() {
    echo "=== Audit des groupes ==="
    echo ""
    echo "Groupes sans membres:"
    getent group | awk -F: '$4 == "" {print $1}'
    echo ""
    echo "Groupes avec un seul membre:"
    getent group | awk -F: 'split($4, a, ",") == 1 {print $1, $4}'
}
```

## Scripts d'automatisation

### Script complet de gestion

**Script de gestion utilisateur complet** :
```bash
#!/bin/bash
# Script complet de gestion utilisateur

set -euo pipefail

USER_MANAGEMENT_DIR="${USER_MANAGEMENT_DIR:-/etc/user-management}"

# Créer la structure de répertoires
setup_user_management() {
    sudo mkdir -p "$USER_MANAGEMENT_DIR"/{scripts,backups,logs}
    echo "Structure créée dans $USER_MANAGEMENT_DIR"
}

# Fonction principale de création
create_user_complete() {
    local username="$1"
    local full_name="${2:-}"
    local groups="${3:-}"
    local shell="${4:-/bin/bash}"
    local password="${5:-}"
    
    # Vérifier que l'utilisateur n'existe pas
    if id "$username" &>/dev/null; then
        echo "Erreur: Utilisateur $username existe déjà"
        return 1
    fi
    
    # Créer l'utilisateur
    local cmd="sudo useradd -m"
    [ -n "$full_name" ] && cmd+=" -c \"$full_name\""
    [ -n "$shell" ] && cmd+=" -s $shell"
    cmd+=" $username"
    
    eval "$cmd"
    
    # Ajouter aux groupes
    if [ -n "$groups" ]; then
        sudo usermod -aG "$groups" "$username"
    fi
    
    # Définir le mot de passe
    if [ -n "$password" ]; then
        echo "$username:$password" | sudo chpasswd
    else
        # Forcer le changement au prochain login
        sudo passwd -e "$username"
    fi
    
    # Configurer l'environnement
    setup_user_environment "$username"
    
    # Logger
    echo "$(date): Utilisateur $username créé" | \
        sudo tee -a "$USER_MANAGEMENT_DIR/logs/user_creation.log"
    
    echo "Utilisateur $username créé avec succès"
}

# Utilisation
# create_user_complete "alice" "Alice Smith" "developers,sudo" "/bin/bash"
```

## Conclusion

La gestion des utilisateurs et groupes est fondamentale pour la sécurité et l'organisation d'un système Linux. En maîtrisant les commandes useradd, usermod, groupadd, passwd, chage, et sudo, vous pouvez créer un environnement multi-utilisateur sécurisé et bien organisé.

Les scripts d'automatisation permettent de gérer efficacement de nombreux utilisateurs, tandis que les outils d'audit garantissent la conformité et la sécurité. Un système bien administré est un système où chaque utilisateur a les permissions appropriées, où les mots de passe sont gérés correctement, et où l'accès aux ressources est contrôlé et tracé.

Dans le chapitre suivant, nous explorerons les variables d'environnement avancées, découvrant comment configurer et gérer l'environnement système et utilisateur de manière professionnelle.
