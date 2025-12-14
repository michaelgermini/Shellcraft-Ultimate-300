# Chapitre 12 - Notions de base en permissions

## Table des matières
- [Introduction](#introduction)
- [Qu'est-ce que les permissions ?](#quest-ce-que-les-permissions-)
- [Le modèle propriétaire-groupe-autres](#le-modèle-propriétaire-groupe-autres)
- [Représentation numérique : les octaux](#représentation-numérique--les-octaux)
- [Représentation symbolique : rwx](#représentation-symbolique--rwx)
- [Commandes de gestion des permissions](#commandes-de-gestion-des-permissions)
- [Permissions spéciales : SUID, SGID, sticky bit](#permissions-spéciales--suid-sgid-sticky-bit)
- [Permissions par défaut et umask](#permissions-par-défaut-et-umask)
- [Permissions sur les répertoires](#permissions-sur-les-répertoires)
- [Dépannage des problèmes de permissions](#dépannage-des-problèmes-de-permissions)
- [Bonnes pratiques de sécurité](#bonnes-pratiques-de-sécurité)
- [Conclusion](#conclusion)

## Introduction

Les permissions constituent le système de contrôle d'accès fondamental des systèmes UNIX-like. Elles déterminent qui peut lire, modifier ou exécuter chaque fichier et répertoire, créant ainsi les frontières de sécurité et de partage dans votre environnement de travail.

Imaginez les permissions comme le système de clés d'un immeuble : chaque porte (fichier) a des serrures spécifiques, et chaque résident (utilisateur) possède un trousseau de clés adapté à ses besoins. Un système de permissions bien conçu permet la collaboration tout en préservant la sécurité et la confidentialité.

## Qu'est-ce que les permissions ?

### Définition fonctionnelle

**Les permissions** sont des attributs associés à chaque fichier et répertoire qui définissent :
- **Qui** peut accéder à l'objet
- **Comment** il peut être utilisé (lecture, écriture, exécution)
- **Dans quelles conditions** l'accès est autorisé

### Trois niveaux d'autorisation

**Lecture (Read)** : Permission d'examiner le contenu
- Fichiers : voir le contenu
- Répertoires : lister les fichiers contenus

**Écriture (Write)** : Permission de modifier
- Fichiers : changer le contenu
- Répertoires : créer/supprimer des fichiers

**Exécution (Execute)** : Permission d'exécuter ou traverser
- Fichiers : exécuter comme programme
- Répertoires : accéder aux fichiers contenus

## Le modèle propriétaire-groupe-autres

### Les trois catégories d'utilisateurs

**Propriétaire (User - u)** :
```bash
# Le propriétaire est généralement le créateur
ls -l fichier.txt
# -rw-r--r-- 1 alice users 1024 Jan 15 10:30 fichier.txt
#           ↑ Propriétaire: alice
```

**Groupe (Group - g)** :
```bash
# Le groupe permet le partage d'équipe
ls -l projet/
# drwxrwxr-x 2 alice developers 4096 Jan 15 10:30 projet/
#              ↑ Groupe: developers
```

**Autres (Others - o)** :
```bash
# Tous les autres utilisateurs système
ls -l public.txt
# -rw-r--r-- 1 alice users 1024 Jan 15 10:30 public.txt
#                 ↑ Autres: r--
```

### Gestion des propriétaires et groupes

**Changement de propriétaire** :
```bash
# Changer le propriétaire
sudo chown bob fichier.txt

# Changer propriétaire et groupe
sudo chown bob:admins fichier.txt

# Récursif pour un répertoire
sudo chown -R alice:developers projet/
```

**Changement de groupe** :
```bash
# Changer seulement le groupe
sudo chgrp www-data site_web/

# Changer groupe récursivement
sudo chgrp -R staff documents/
```

## Représentation numérique : les octaux

### Système octal (base 8)

**Chaque permission vaut une puissance de 2** :
- Lecture (r) = 4
- Écriture (w) = 2
- Exécution (x) = 1

**Calcul des valeurs** :
```bash
# rw- = 4 + 2 + 0 = 6
# r-x = 4 + 0 + 1 = 5
# --x = 0 + 0 + 1 = 1
# rwx = 4 + 2 + 1 = 7
```

### Permissions complètes en octal

**Format : propriétaire-groupe-autres**
```bash
# rw-r--r-- = 644
# 6 (rw-) + 4 (r--) + 4 (r--) = 644

# rwxr-xr-x = 755
# 7 (rwx) + 5 (r-x) + 5 (r-x) = 755

# rw------- = 600
# 6 (rw-) + 0 (---) + 0 (---) = 600
```

### Commandes avec notation octale

**chmod octal** :
```bash
# Rendre un script exécutable
chmod 755 script.sh

# Fichier privé (lecture/écriture propriétaire seulement)
chmod 600 secret.txt

# Répertoire partagé en lecture
chmod 755 public/

# Récursif pour tout un projet
chmod -R 750 projet/
```

## Représentation symbolique : rwx

### Notation symbolique complète

**Format : [ugoa][+-=][rwx]**
```bash
# Ajouter l'exécution pour tous
chmod a+x script.sh

# Retirer l'écriture du groupe
chmod g-w fichier.txt

# Donner lecture/écriture au propriétaire seulement
chmod u=rw,go= fichier.txt
```

### Opérateurs détaillés

**Opérateur + (ajouter)** :
```bash
# Ajouter l'exécution au propriétaire
chmod u+x script.sh

# Ajouter lecture au groupe et autres
chmod go+r document.txt
```

**Opérateur - (retirer)** :
```bash
# Retirer l'écriture de tous
chmod a-w fichier.txt

# Retirer toutes les permissions du groupe
chmod g-rwx dossier/
```

**Opérateur = (définir exactement)** :
```bash
# Propriétaire: lecture/écriture, groupe: lecture, autres: rien
chmod u=rw,g=r,o= fichier.txt

# Permissions classiques pour script
chmod u=rwx,go=rx script.sh
```

## Commandes de gestion des permissions

### chmod : changer les permissions

**Utilisation de base** :
```bash
# Notation octale
chmod 644 fichier.txt
chmod 755 script.sh

# Notation symbolique
chmod u+x script.sh
chmod g+rw document.txt
chmod o-rwx secret.txt
```

**Options avancées** :
```bash
# Récursif
chmod -R 755 projet/

# Verbeux (afficher les changements)
chmod -v 644 *.txt

# Simuler (ne pas appliquer)
chmod --dry-run -R 755 projet/

# Conserver les liens symboliques
chmod -h 755 lien_symbolique
```

### chown : changer le propriétaire

**Syntaxe complète** :
```bash
# Changer propriétaire seulement
chown nouvel_user fichier.txt

# Changer propriétaire et groupe
chown nouvel_user:nouveau_groupe fichier.txt

# Utiliser UID/GID numériques
chown 1000:1000 fichier.txt

# Récursif
chown -R alice:developers projet/
```

### chgrp : changer le groupe

**Utilisation spécialisée** :
```bash
# Changer groupe seulement
chgrp www-data site_web/

# Utiliser GID numérique
chgrp 1000 fichier.txt

# Avec sudo pour les groupes système
sudo chgrp adm /var/log/app.log
```

## Permissions spéciales : SUID, SGID, sticky bit

### SUID (Set User ID)

**Principe** : Le programme s'exécute avec les droits du propriétaire
```bash
# Exemple classique : passwd
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 59976 Jan 15 10:30 /usr/bin/passwd
#    ↑ Le 's' indique SUID

# Définir SUID
chmod u+s programme

# Retirer SUID
chmod u-s programme
```

**Cas d'usage** :
- Programmes système nécessitant des droits élevés
- Utilitaires de changement de mot de passe
- Programmes d'administration

### SGID (Set Group ID)

**Principe** : Le programme s'exécute avec les droits du groupe
```bash
# Sur les fichiers
chmod g+s programme

# Sur les répertoires (héritage de groupe)
chmod g+s dossier/
# Tous les fichiers créés auront le groupe du dossier
```

**Applications pratiques** :
- Répertoires de projet d'équipe
- Services web avec groupe spécifique
- Environnements de développement partagés

### Sticky bit

**Principe** : Seuls le propriétaire et root peuvent supprimer des fichiers
```bash
# Classique sur /tmp
ls -ld /tmp
# drwxrwxrwt 1 root root 4096 Jan 15 10:30 /tmp
#         ↑ Le 't' indique sticky bit

# Définir sticky bit
chmod +t dossier/

# Retirer sticky bit
chmod -t dossier/
```

**Utilisations** :
- Répertoires temporaires (/tmp)
- Répertoires de spool d'impression
- Zones de partage contrôlé

## Permissions par défaut et umask

### Qu'est-ce que umask ?

**umask** définit les permissions à retirer par défaut :
```bash
# Voir umask actuel
umask
# Output: 0022 (ou 022)

# Signification: retirer les permissions d'écriture du groupe et autres
```

### Calcul des permissions par défaut

**Pour les fichiers** :
```bash
# umask 022
# Permissions de base: 666 (rw-rw-rw-)
# Après umask: 666 - 022 = 644 (rw-r--r--)
```

**Pour les répertoires** :
```bash
# umask 022
# Permissions de base: 777 (rwxrwxrwx)
# Après umask: 777 - 022 = 755 (rwxr-xr-x)
```

### Configuration de umask

**Temporaire** :
```bash
# Plus restrictif
umask 077  # rw------- pour fichiers, rwx------ pour répertoires

# Plus permissif
umask 002  # rw-rw-r-- pour fichiers, rwxrwxr-x pour répertoires
```

**Permanent** :
```bash
# Dans ~/.bashrc ou ~/.profile
umask 022

# Pour root (plus restrictif)
umask 077
```

## Permissions sur les répertoires

### Permissions spéciales pour répertoires

**Lecture (r)** : Pouvoir lister le contenu
```bash
ls repertoire/  # Nécessite r sur le répertoire
```

**Écriture (w)** : Pouvoir créer/supprimer des fichiers
```bash
touch repertoire/nouveau.txt  # Nécessite w sur le répertoire
rm repertoire/ancien.txt      # Nécessite w sur le répertoire
```

**Exécution (x)** : Pouvoir traverser le répertoire
```bash
cd repertoire/  # Nécessite x sur le répertoire
ls repertoire/  # Nécessite x sur le répertoire
```

### Combinaisons courantes

**Répertoire privé** :
```bash
chmod 700 repertoire_prive/
# rwx------ : propriétaire seulement
```

**Répertoire partagé en lecture** :
```bash
chmod 755 repertoire_public/
# rwxr-xr-x : lecture pour tous, écriture propriétaire
```

**Répertoire de projet d'équipe** :
```bash
chmod 2770 projet_equipe/
# drwxrws--- : SGID pour héritage de groupe
```

## Dépannage des problèmes de permissions

### Diagnostic des problèmes

**Vérification des permissions** :
```bash
# Permissions détaillées
ls -l fichier.txt

# Permissions récursives
ls -laR repertoire/

# Comparaison avec les droits effectifs
id  # Qui suis-je ?
groups  # Mes groupes
```

**Tests d'accès** :
```bash
# Tester la lecture
[ -r fichier.txt ] && echo "Peut lire" || echo "Ne peut pas lire"

# Tester l'écriture
[ -w fichier.txt ] && echo "Peut écrire" || echo "Ne peut pas écrire"

# Tester l'exécution
[ -x script.sh ] && echo "Peut exécuter" || echo "Ne peut pas exécuter"
```

### Résolution courante

**"Permission denied"** :
```bash
# Vérifier les permissions
ls -l fichier.txt

# Vérifier le propriétaire
stat fichier.txt

# Changer si nécessaire
sudo chown $USER:$USER fichier.txt
chmod 644 fichier.txt
```

**Scripts non exécutables** :
```bash
# Ajouter l'exécution
chmod +x script.sh

# Vérifier le shebang
head -1 script.sh  # Doit être #!/bin/bash ou similaire
```

## Bonnes pratiques de sécurité

### Principe du moindre privilège

**Permissions minimales** :
```bash
# Fichiers de configuration : lecture propriétaire seulement
chmod 600 config.ini

# Scripts système : exécution contrôlée
chmod 755 script.sh

# Données sensibles : accès restreint
chmod 400 secret.key
```

### Gestion des groupes

**Groupes fonctionnels** :
```bash
# Créer un groupe de projet
sudo groupadd projet_web

# Ajouter des utilisateurs
sudo usermod -aG projet_web alice
sudo usermod -aG projet_web bob

# Permissions de groupe
chmod 770 projet/
sudo chgrp projet_web projet/
```

### Audit et monitoring

**Vérification régulière** :
```bash
# Fichiers sans propriétaire
find /home -nouser -o -nogroup

# Fichiers avec permissions trop permissives
find /home -type f -perm 777

# Scripts setuid/setgid
find /usr -type f \( -perm -4000 -o -perm -2000 \)
```

### Sauvegarde des permissions

**Archivage avec préservation** :
```bash
# Sauvegarder avec permissions
tar -czpf backup.tar.gz --same-owner repertoire/

# Restaurer avec permissions
tar -xzpf backup.tar.gz --same-owner
```

## Conclusion

Les permissions constituent le premier niveau de sécurité et de contrôle d'accès dans les systèmes UNIX-like. Elles permettent de définir précisément qui peut faire quoi, créant ainsi un environnement sécurisé et collaboratif.

Maîtriser les permissions - leur notation octale et symbolique, leurs commandes de gestion, leurs aspects spéciaux - est essentiel pour l'administration système, la sécurité, et le développement d'applications robustes.

Dans le chapitre suivant, nous explorerons les concepts de processus et de multitâche, qui expliquent comment le système gère l'exécution simultanée de multiples programmes dans cet environnement de permissions contrôlé.
