# Chapitre 22 - Gestion des fichiers et dossiers

## Table des matières
- [Introduction](#introduction)
- [Création et suppression de fichiers](#création-et-suppression-de-fichiers)
- [Création et suppression de répertoires](#création-et-suppression-de-répertoires)
- [Copie de fichiers et répertoires](#copie-de-fichiers-et-répertoires)
- [Déplacement et renommage](#déplacement-et-renommage)
- [Liens symboliques et liens physiques](#liens-symboliques-et-liens-physiques)
- [Attributs étendus et métadonnées](#attributs-étendus-et-métadonnées)
- [Gestion de l'espace disque](#gestion-de-lespace-disque)
- [Opérations groupées et automation](#opérations-groupées-et-automation)
- [Gestion des fichiers temporaires](#gestion-des-fichiers-temporaires)
- [Archivage et compression](#archivage-et-compression)
- [Dépannage des opérations sur fichiers](#dépannage-des-opérations-sur-fichiers)
- [Conclusion](#conclusion)

## Introduction

La gestion des fichiers et dossiers constitue le cœur de l'interaction quotidienne avec un système Linux. Au-delà des simples opérations de création et suppression, cette gestion englobe la copie intelligente, le déplacement stratégique, les liens symboliques, et l'organisation logique des données.

Imaginez les fichiers et dossiers comme les pièces d'un immense puzzle tridimensionnel : chaque pièce doit être placée au bon endroit, connectée aux bonnes autres pièces, et maintenue dans un état optimal. Une mauvaise gestion initiale peut transformer une belle architecture en chaos inextricable.

## Création et suppression de fichiers

### touch : Création de fichiers

**Syntaxe et utilisation** :
```bash
touch [OPTIONS] fichier...

# Création simple
touch nouveau_fichier.txt

# Création multiple
touch fichier1.txt fichier2.txt fichier3.txt

# Création avec chemin
touch /tmp/mon_fichier.txt
```

**Options puissantes** :
```bash
# Ne pas créer si le fichier existe
touch -c fichier_existant.txt

# Définir une date spécifique
touch -t 202401151200 fichier.txt    # 15 jan 2024 12:00
touch -d "2024-01-15 12:00" fichier.txt

# Utiliser la date d'un autre fichier
touch -r reference.txt cible.txt

# Mode verbose
touch -v nouveau.txt
# nouveau.txt
```

**Applications pratiques** :
```bash
# Créer des fichiers de log vides
touch /var/log/app.log

# Marquer des points de contrôle
touch ~/.last_backup

# Créer des fichiers de configuration par défaut
[ ! -f config.ini ] && touch config.ini
```

### rm : Suppression de fichiers

**Syntaxe de base** :
```bash
rm [OPTIONS] fichier...

# Suppression simple
rm fichier.txt

# Suppression multiple
rm *.tmp *.bak

# Suppression récursive
rm -r repertoire/
```

**Options de sécurité** :
```bash
# Mode interactif (demande confirmation)
rm -i important.txt
# rm: remove regular file 'important.txt'? y

# Mode verbose
rm -v fichier.txt
# removed 'fichier.txt'

# Sécurité maximale
rm -iv fichier.txt

# Ne pas suivre les liens symboliques
rm -r --preserve-root /  # Interdit la suppression de /
```

**Suppression intelligente** :
```bash
# Supprimer seulement certains types
find . -name "*.tmp" -type f -delete

# Supprimer les fichiers plus vieux que 7 jours
find . -name "*.log" -mtime +7 -delete

# Supprimer les fichiers vides
find . -empty -type f -delete
```

## Création et suppression de répertoires

### mkdir : Création de répertoires

**Syntaxe et options** :
```bash
mkdir [OPTIONS] répertoire...

# Création simple
mkdir nouveau_dossier

# Création imbriquée
mkdir -p parent/enfant/petitenfant

# Mode verbose
mkdir -v mon_dossier
# mkdir: created directory 'mon_dossier'

# Définir les permissions
mkdir -m 755 mon_dossier
```

**Création en lot** :
```bash
# Structure de projet standard
mkdir -p projet/{src,tests,docs,scripts}

# Création basée sur une liste
cat répertoires.txt | xargs mkdir -p

# Création conditionnelle
[ ! -d "$dir" ] && mkdir -p "$dir"
```

### rmdir : Suppression de répertoires vides

**Utilisation de base** :
```bash
# Supprimer un répertoire vide
rmdir dossier_vide

# Supprimer plusieurs
rmdir dir1 dir2 dir3

# Supprimer récursivement (si vides)
rmdir -p parent/enfant  # Supprime enfant, puis parent si vide
```

**Limitations et alternatives** :
```bash
# rmdir échoue si le répertoire n'est pas vide
rmdir plein/
# rmdir: failed to remove 'plein/': Directory not empty

# Alternative : rm -r (dangereux)
rm -r plein/

# Alternative sécurisée : vérifier d'abord
[ -z "$(ls -A plein/)" ] && rmdir plein/ || echo "Répertoire non vide"
```

## Copie de fichiers et répertoires

### cp : Copie standard

**Syntaxe complète** :
```bash
cp [OPTIONS] SOURCE DEST
cp [OPTIONS] SOURCE... RÉPERTOIRE

# Copie simple
cp source.txt destination.txt

# Copie vers un répertoire
cp fichier.txt /tmp/

# Copie multiple
cp *.txt /tmp/
```

**Options essentielles** :
```bash
# Récursif (pour les répertoires)
cp -r source/ destination/

# Préserver les attributs
cp -p fichier.txt copie.txt  # permissions, propriétaire, timestamps
cp -a source/ destination/   # équivalent à -dR --preserve=all

# Mode interactif (confirmation)
cp -i source.txt dest.txt

# Mode verbose
cp -v *.txt /tmp/
```

**Copie intelligente** :
```bash
# Copie seulement si plus récent
cp -u source.txt destination.txt

# Copie avec backup automatique
cp -b source.txt destination.txt  # Crée destination.txt~

# Copie avec liens symboliques
cp -s source.txt lien_symbolique

# Copie sans suivre les liens
cp -P fichier_via_lien.txt destination.txt
```

### rsync : Copie avancée et synchronisation

**Installation et syntaxe** :
```bash
# Installation (si nécessaire)
sudo apt install rsync

# Syntaxe de base
rsync [OPTIONS] SOURCE DEST

# Synchronisation de répertoires
rsync -av source/ destination/
```

**Options puissantes** :
```bash
# Archive (préserver tout)
rsync -a source/ dest/

# Compression en transit
rsync -az source/ dest/

# Mode dry-run (simulation)
rsync -av --dry-run source/ dest/

# Supprimer les fichiers obsolètes
rsync -av --delete source/ dest/

# Progression détaillée
rsync -avh --progress source/ dest/

# Exclure des patterns
rsync -av --exclude "*.tmp" source/ dest/
```

**Applications avancées** :
```bash
# Sauvegarde incrémentielle
rsync -a --link-dest=/path/to/previous /source /backup/$(date +%Y%m%d)

# Synchronisation bidirectionnelle (avec unison)
# rsync ne fait que monodirectionnel

# Copie via SSH
rsync -av -e ssh source/ user@remote:/destination/
```

## Déplacement et renommage

### mv : Déplacement et renommage

**Syntaxe et utilisation** :
```bash
mv [OPTIONS] SOURCE DEST
mv [OPTIONS] SOURCE... RÉPERTOIRE

# Renommage simple
mv ancien_nom.txt nouveau_nom.txt

# Déplacement
mv fichier.txt /tmp/

# Déplacement et renommage
mv /tmp/fichier.txt ./nouveau_nom.txt
```

**Options importantes** :
```bash
# Mode interactif
mv -i source.txt dest.txt

# Mode verbose
mv -v fichier.txt /tmp/

# Ne pas suivre les liens symboliques
mv -P lien_symbolique destination

# Forcer l'écrasement
mv -f source.txt existing.txt
```

**Déplacement en lot** :
```bash
# Renommer en masse
for file in *.txt; do
    mv "$file" "${file%.txt}.md"
done

# Organiser par type
mkdir -p images textes autres
mv *.jpg *.png images/ 2>/dev/null || true
mv *.txt *.md textes/ 2>/dev/null || true
mv * autres/ 2>/dev/null || true
```

### rename : Renommage par expressions régulières

**Utilisation avancée** :
```bash
# Installation (Ubuntu/Debian)
sudo apt install rename

# Syntaxe
rename 's/ancien/nouveau/' fichiers...

# Exemples pratiques
rename 's/\.jpeg$/.jpg/' *.jpeg        # Changer l'extension
rename 's/^/prefix_/' *.txt            # Ajouter un préfixe
rename 's/(\w+)/\U$1/g' *.txt          # Majuscules
rename 's/ /_/g' *                     # Remplacer espaces par _
```

## Liens symboliques et liens physiques

### Liens physiques (hard links)

**Principe** : Plusieurs noms pour le même inode
```bash
# Création d'un lien physique
ln source.txt lien_physique.txt

# Vérification
ls -li fichier*
# 123456 -rw-r--r-- 2 user group ... source.txt
# 123456 -rw-r--r-- 2 user group ... lien_physique.txt

# Le compteur de liens (2) montre qu'il y a deux noms
```

**Propriétés des liens physiques** :
- Même inode, même données
- Impossible entre systèmes de fichiers différents
- Suppression d'un lien n'affecte pas les autres
- Dernier lien supprimé efface réellement le fichier

### Liens symboliques (soft links)

**Principe** : Pointeur vers un chemin
```bash
# Création d'un lien symbolique
ln -s cible.txt lien_symbolique.txt

# Vérification
ls -li lien_symbolique.txt
# 123457 lrwxrwxrwx 1 user group ... lien_symbolique.txt -> cible.txt

# Avec chemin absolu
ln -s /chemin/absolu/cible.txt lien_symbolique.txt
```

**Comparaison des liens** :
```bash
# Liens physiques
ln fichier1 fichier2        # Même inode
ls -li fichier1 fichier2    # Même numéro d'inode

# Liens symboliques
ln -s fichier1 fichier2     # Pointeur vers le nom
ls -li fichier1 fichier2    # Inodes différents
```

**Gestion des liens cassés** :
```bash
# Trouver les liens cassés
find . -type l ! -exec test -e {} \; -print

# Supprimer les liens cassés
find . -type l ! -exec test -e {} \; -delete

# Recréer un lien cassé
ln -s /nouveau/chemin cible.txt
```

## Attributs étendus et métadonnées

### Attributs étendus (extended attributes)

**Gestion avec attr** :
```bash
# Installation
sudo apt install attr

# Définir un attribut
setfattr -n user.comment -v "Fichier important" document.txt

# Lire un attribut
getfattr -n user.comment document.txt

# Lister tous les attributs
getfattr -d document.txt
```

**Attributs système courants** :
```bash
# Fichier immuable (root seulement)
sudo chattr +i fichier.txt

# Fichier append-only
sudo chattr +a logfile.txt

# Pas de dump automatique
sudo chattr +d gros_fichier.dat

# Compression automatique (btrfs)
sudo chattr +c fichier.txt
```

### Métadonnées étendues

**Informations détaillées** :
```bash
# Métadonnées complètes
stat fichier.txt

# Timestamps détaillés
stat -c '%y %n' *  # Dernière modification

# Droits étendus
getfacl fichier.txt  # Access Control Lists
```

## Gestion de l'espace disque

### df : Espace disponible

**Informations sur les systèmes de fichiers** :
```bash
# Vue d'ensemble
df -h

# Systèmes spécifiques
df -h /home

# Types de systèmes de fichiers
df -Th

# Inodes disponibles
df -ih
```

### du : Utilisation de l'espace

**Analyse de l'utilisation** :
```bash
# Taille d'un répertoire
du -sh /home/user

# Triage par taille
du -h | sort -hr | head -10

# Analyse détaillée
du -ah /home/user | sort -hr | head -20

# Exclure certains types
du -h --exclude="*.iso" /home/user
```

**ncdu : Analyse interactive** :
```bash
# Analyse visuelle
sudo apt install ncdu
ncdu /home/user

# Options
ncdu -x /        # Ne pas traverser les FS
ncdu -q          # Mode silencieux
```

## Opérations groupées et automation

### find avec exécution

**Actions automatisées** :
```bash
# Changer les permissions récursivement
find . -type f -name "*.sh" -exec chmod +x {} \;

# Compresser les anciens logs
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;

# Renommer en masse
find . -name "*old*" -exec rename 's/old/new/' {} \;

# Copie intelligente
find . -name "*.jpg" -exec cp {} /tmp/photos/ \;
```

### xargs : Traitement par lots

**Transformation find → commande** :
```bash
# Copie groupée
find . -name "*.txt" | xargs cp -t /tmp/

# Suppression sécurisée
find . -name "*.tmp" | xargs rm -i

# Traitement parallèle
find . -name "*.jpg" | xargs -P 4 -n 10 convert -resize 50% {}
```

**Gestion des noms avec espaces** :
```bash
# Problème avec les espaces
find . -name "*.txt" | xargs rm  # Échoue avec "fichier avec espaces.txt"

# Solution
find . -name "*.txt" -print0 | xargs -0 rm

# Alternative moderne
find . -name "*.txt" -exec rm {} +
```

### Scripts d'automatisation

**Script de nettoyage automatique** :
```bash
#!/bin/bash
# cleanup.sh - Nettoyage automatique du système

LOGFILE="/var/log/cleanup.log"
log() { echo "$(date): $*" >> "$LOGFILE"; }

log "Début du nettoyage"

# Supprimer les fichiers temporaires
find /tmp -type f -mtime +7 -delete 2>/dev/null
log "Fichiers temporaires nettoyés"

# Vider les caches utilisateur
find /home -name ".cache" -type d -exec rm -rf {}/* \; 2>/dev/null
log "Caches utilisateur vidés"

# Nettoyer les paquets
if command -v apt >/dev/null; then
    apt autoremove -y >/dev/null 2>&1
    apt autoclean -y >/dev/null 2>&1
    log "Paquets système nettoyés"
fi

log "Nettoyage terminé"
```

## Gestion des fichiers temporaires

### Répertoires temporaires

**Emplacements standard** :
```bash
# Variables d'environnement
echo "$TMPDIR"  # Peut être personnalisé
echo "$TMP"     # Alternative

# Répertoires par défaut
/tmp            # Temporaire système
/var/tmp        # Temporaire préservé
~/tmp           # Temporaire utilisateur
```

**Création sécurisée** :
```bash
# Créer un répertoire temporaire unique
TEMP_DIR=$(mktemp -d)
echo "Répertoire temporaire: $TEMP_DIR"

# Utilisation
echo "données temporaires" > "$TEMP_DIR/tempfile.txt"

# Nettoyage automatique
trap "rm -rf '$TEMP_DIR'" EXIT

# Le répertoire sera supprimé automatiquement à la sortie
```

### mktemp : Création sécurisée

**Fichiers temporaires** :
```bash
# Fichier temporaire
TEMP_FILE=$(mktemp)
echo "Fichier temporaire: $TEMP_FILE"

# Avec suffixe
TEMP_SCRIPT=$(mktemp --suffix=.sh)
echo "#!/bin/bash" > "$TEMP_SCRIPT"

# Avec préfixe
TEMP_DATA=$(mktemp /tmp/data_XXXXXX.txt)
```

**Nettoyage automatique** :
```bash
# Nettoyer à la sortie
cleanup_temp() {
    rm -f "$TEMP_FILE" "$TEMP_DIR"
}
trap cleanup_temp EXIT INT TERM
```

## Archivage et compression

### tar : Archivage standard

**Création d'archives** :
```bash
# Archive simple
tar -cf archive.tar fichiers/

# Avec compression gzip
tar -czf archive.tar.gz fichiers/

# Avec compression xz (meilleure)
tar -cJf archive.tar.xz fichiers/

# Archive avec chemin absolu préservé
tar -cJf backup.tar.xz -P /home/user
```

**Extraction d'archives** :
```bash
# Extraction simple
tar -xf archive.tar

# Avec compression
tar -xzf archive.tar.gz
tar -xJf archive.tar.xz

# Extraction dans un répertoire spécifique
tar -xzf archive.tar.gz -C /tmp/extraction/

# Lister le contenu
tar -tf archive.tar.gz
```

### Autres outils de compression

**gzip/bzip2/xz individuels** :
```bash
# Compression
gzip fichier.txt         # → fichier.txt.gz
bzip2 fichier.txt        # → fichier.txt.bz2
xz fichier.txt           # → fichier.txt.xz

# Décompression
gunzip fichier.txt.gz
bunzip2 fichier.txt.bz2
unxz fichier.txt.xz
```

**zip (compatible Windows)** :
```bash
# Création
zip archive.zip fichiers/

# Avec mot de passe
zip -e secure.zip fichiers/

# Extraction
unzip archive.zip
unzip -d /tmp/ secure.zip
```

## Dépannage des opérations sur fichiers

### Problèmes courants

**Permissions insuffisantes** :
```bash
# Diagnostic
ls -l fichier.txt
# -rw------- 1 autre_user group fichier.txt

# Solutions
sudo chown $USER:$USER fichier.txt
sudo chmod 644 fichier.txt
```

**Espace disque insuffisant** :
```bash
# Vérifications
df -h /path/to/destination
du -sh /path/to/source

# Solutions
# Libérer de l'espace ou changer de destination
find /tmp -type f -delete  # Nettoyer /tmp
```

**Fichiers ouverts par des processus** :
```bash
# Trouver le processus
lsof fichier.txt

# Forcer la fermeture (dangereux)
kill -9 $PID

# Attendre que le processus se termine
while lsof fichier.txt >/dev/null; do sleep 1; done
```

### Outils de diagnostic

**Vérification d'intégrité** :
```bash
# Somme de contrôle
md5sum fichier.txt > fichier.txt.md5
md5sum -c fichier.txt.md5

# Pour les archives
tar -tf archive.tar >/dev/null && echo "Archive valide"
```

**Récupération de fichiers supprimés** :
```bash
# TestDisk pour récupération avancée
sudo apt install testdisk

# Récupération simple (si partition ext4)
extundelete /dev/sda1 --restore-file deleted_file
```

### Journalisation des opérations

**Script avec logging complet** :
```bash
#!/bin/bash
# Script avec journalisation détaillée

LOGFILE="/var/log/file_operations.log"
log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') [$1] $2" | tee -a "$LOGFILE"
}

# Fonction de copie avec vérification
safe_copy() {
    local source="$1" destination="$2"
    
    if [ ! -f "$source" ]; then
        log "ERROR" "Source inexistante: $source"
        return 1
    fi
    
    if ! cp -v "$source" "$destination" 2>>"$LOGFILE"; then
        log "ERROR" "Échec de copie: $source → $destination"
        return 1
    fi
    
    log "INFO" "Copie réussie: $source → $destination"
}

# Utilisation
safe_copy important.txt /backup/
```

## Conclusion

La gestion des fichiers et dossiers sous Linux est un art qui combine précision technique et stratégie organisationnelle. Des opérations de base comme `touch` et `rm` aux techniques avancées comme les liens symboliques et l'archivage compressé, chaque outil contribue à une gestion efficace des données.

La véritable maîtrise vient de la compréhension des implications de chaque opération : un lien physique n'est pas la même chose qu'un lien symbolique, une copie récursive peut avoir des conséquences inattendues, et une suppression sans confirmation peut être désastreuse.

Dans le chapitre suivant, nous explorerons les permissions et la sécurité, apprenant comment contrôler précisément qui peut accéder à vos fichiers et dans quelles conditions.

