# Chapitre 31 - Compression et archivage

## Table des matières
- [Introduction](#introduction)
- [Concepts de compression](#concepts-de-compression)
- [Outils d'archivage : tar](#outils-darchivage-tar)
- [Algorithmes de compression](#algorithmes-de-compression)
- [Compression parallèle et optimisation](#compression-parallèle-et-optimisation)
- [Archivage intelligent et automation](#archivage-intelligent-et-automation)
- [Vérification d'intégrité](#vérification-dintégrité)
- [Gestion des archives volumineuses](#gestion-des-archives-volumineuses)
- [Conclusion](#conclusion)

## Introduction

La compression et l'archivage constituent des compétences essentielles dans l'administration système moderne. Que ce soit pour sauvegarder des données, transférer des fichiers volumineux, ou optimiser l'espace de stockage, maîtriser ces techniques permet d'économiser du temps et des ressources.

Imaginez la compression comme un art de l'emballage : il s'agit de réduire le volume d'informations sans perdre leur essence, puis de les organiser dans des conteneurs faciles à transporter et à stocker.

## Concepts de compression

### Principe fondamental

**Définition** : La compression réduit la taille d'un fichier en éliminant la redondance et en utilisant des algorithmes efficaces pour représenter les données.

**Types de compression** :
- **Sans perte (lossless)** : Les données originales peuvent être reconstruites exactement
  - Exemples : gzip, bzip2, xz, zip
  - Utilisation : fichiers texte, code source, données critiques
- **Avec perte (lossy)** : Certaines données sont perdues pour une meilleure compression
  - Exemples : JPEG, MP3, MPEG
  - Utilisation : images, audio, vidéo

**Ratio de compression** :
```bash
# Calculer le ratio
taille_originale=1000000  # 1 MB
taille_compressée=200000  # 200 KB
ratio=$(echo "scale=2; $taille_originale / $taille_compressée" | bc)
echo "Ratio: ${ratio}:1"  # 5:1
```

### Facteurs affectant la compression

**Type de données** :
- **Texte** : Compression excellente (50-90% de réduction)
- **Images non compressées** : Compression bonne (30-70%)
- **Fichiers déjà compressés** : Compression limitée (0-10%)
- **Données aléatoires** : Compression impossible

**Taille des fichiers** :
```bash
# Petits fichiers : overhead de métadonnées important
# Grands fichiers : meilleur ratio de compression

# Exemple
echo "test" > small.txt
echo "test" > small2.txt
tar -czf small.tar.gz small.txt small2.txt
# Taille peut être plus grande que la somme des fichiers !
```

## Outils d'archivage : tar

### Tar : Tape Archive

**Historique** :
- Créé en 1979 pour les sauvegardes sur bandes magnétiques
- Standard de facto pour l'archivage sous UNIX/Linux
- Combine plusieurs fichiers en une seule archive

**Syntaxe de base** :
```bash
# Créer une archive
tar -cf archive.tar fichiers/

# Extraire une archive
tar -xf archive.tar

# Lister le contenu
tar -tf archive.tar

# Options courantes
# -c : créer
# -x : extraire
# -t : lister
# -f : fichier
# -v : verbeux
# -z : gzip
# -j : bzip2
# -J : xz
```

### Création d'archives

**Archives simples** :
```bash
# Créer une archive
tar -cf backup.tar /home/user/documents

# Avec affichage verbeux
tar -cvf backup.tar /home/user/documents

# Exclure des fichiers
tar -cvf backup.tar --exclude='*.log' /home/user/

# Exclure plusieurs patterns
tar -cvf backup.tar \
    --exclude='*.log' \
    --exclude='*.tmp' \
    --exclude='.git' \
    /home/user/
```

**Archives compressées** :
```bash
# Avec gzip (.tar.gz ou .tgz)
tar -czf backup.tar.gz /home/user/documents

# Avec bzip2 (.tar.bz2)
tar -cjf backup.tar.bz2 /home/user/documents

# Avec xz (.tar.xz)
tar -cJf backup.tar.xz /home/user/documents

# Comparaison des tailles
ls -lh backup.tar*
```

### Extraction d'archives

**Extraction de base** :
```bash
# Extraire dans le répertoire courant
tar -xf archive.tar

# Extraire dans un répertoire spécifique
tar -xf archive.tar -C /destination/

# Extraire avec affichage
tar -xvf archive.tar

# Extraire des fichiers spécifiques
tar -xvf archive.tar fichier1.txt fichier2.txt

# Extraire avec wildcards
tar -xvf archive.tar "*.txt"
```

**Préservation des permissions** :
```bash
# Par défaut, tar préserve les permissions
tar -czf backup.tar.gz /home/user/

# Vérifier les permissions après extraction
tar -tzf backup.tar.gz | head -5
```

### Options avancées de tar

**Fichiers de liste** :
```bash
# Créer une liste de fichiers à archiver
cat > files_to_backup.txt << EOF
/home/user/documents
/home/user/images
/home/user/configs
EOF

# Utiliser la liste
tar -czf backup.tar.gz -T files_to_backup.txt
```

**Archives incrémentielles** :
```bash
# Créer une archive complète
tar -czf full_backup_$(date +%Y%m%d).tar.gz /data/

# Créer une archive incrémentielle (fichiers modifiés depuis)
tar -czf incremental_backup_$(date +%Y%m%d).tar.gz \
    --newer-mtime="1 day ago" \
    /data/
```

**Vérification d'intégrité** :
```bash
# Créer avec vérification
tar -czf backup.tar.gz --checkpoint=.1000 /data/

# Vérifier une archive
tar -tzf backup.tar.gz > /dev/null && echo "Archive valide"
```

## Algorithmes de compression

### Gzip (GNU Zip)

**Caractéristiques** :
- Algorithme : DEFLATE (LZ77 + Huffman)
- Vitesse : Rapide
- Compression : Moyenne
- Utilisation : Standard général

**Utilisation** :
```bash
# Compresser
gzip file.txt              # Crée file.txt.gz
gzip -9 file.txt           # Compression maximale (plus lent)

# Décompresser
gunzip file.txt.gz
gzip -d file.txt.gz

# Conserver le fichier original
gzip -c file.txt > file.txt.gz
# ou
gzip -k file.txt           # Keep original

# Niveaux de compression (1-9)
gzip -1 file.txt           # Rapide, compression faible
gzip -9 file.txt           # Lent, compression maximale
```

**Comparaison des niveaux** :
```bash
# Test de compression
time gzip -1 large_file.txt    # Rapide
time gzip -9 large_file.txt    # Plus lent mais meilleure compression

# Voir les statistiques
gzip -l file.txt.gz
```

### Bzip2

**Caractéristiques** :
- Algorithme : Burrows-Wheeler Transform
- Vitesse : Plus lent que gzip
- Compression : Meilleure que gzip (surtout pour texte)
- Utilisation : Fichiers texte volumineux

**Utilisation** :
```bash
# Compresser
bzip2 file.txt             # Crée file.txt.bz2
bzip2 -9 file.txt          # Compression maximale

# Décompresser
bunzip2 file.txt.bz2
bzip2 -d file.txt.bz2

# Conserver l'original
bzip2 -k file.txt

# Niveaux (1-9)
bzip2 -1 file.txt          # Rapide
bzip2 -9 file.txt          # Compression maximale
```

**Comparaison avec gzip** :
```bash
# Test comparatif
time gzip -9 large_file.txt
time bzip2 -9 large_file.txt

# Comparer les tailles
ls -lh large_file.txt.gz large_file.txt.bz2
```

### XZ (LZMA)

**Caractéristiques** :
- Algorithme : LZMA2
- Vitesse : Très lent
- Compression : Excellente (meilleure que bzip2)
- Utilisation : Archives à long terme, distributions Linux

**Utilisation** :
```bash
# Compresser
xz file.txt                # Crée file.txt.xz
xz -9 file.txt             # Compression maximale

# Décompresser
unxz file.txt.xz
xz -d file.txt.xz

# Conserver l'original
xz -k file.txt

# Niveaux (0-9)
xz -0 file.txt             # Rapide, compression faible
xz -9 file.txt             # Très lent, compression maximale
```

**Comparaison globale** :
```bash
#!/bin/bash
# compare_compression.sh

FILE="large_file.txt"
echo "Fichier original:"
ls -lh "$FILE"

echo -e "\nGzip:"
time gzip -9 -c "$FILE" > "${FILE}.gz"
ls -lh "${FILE}.gz"

echo -e "\nBzip2:"
time bzip2 -9 -c "$FILE" > "${FILE}.bz2"
ls -lh "${FILE}.bz2"

echo -e "\nXZ:"
time xz -9 -c "$FILE" > "${FILE}.xz"
ls -lh "${FILE}.xz"
```

### Zip et Unzip

**Caractéristiques** :
- Format : ZIP (compatible Windows)
- Algorithme : DEFLATE
- Avantage : Compatibilité multi-plateforme

**Utilisation** :
```bash
# Créer une archive ZIP
zip -r archive.zip directory/

# Ajouter des fichiers
zip archive.zip newfile.txt

# Extraire
unzip archive.zip

# Extraire dans un répertoire
unzip archive.zip -d /destination/

# Lister le contenu
unzip -l archive.zip

# Compression maximale
zip -9 archive.zip files/
```

## Compression parallèle et optimisation

### Pigz (Parallel gzip)

**Installation et utilisation** :
```bash
# Installation
sudo apt install pigz        # Ubuntu/Debian
sudo dnf install pigz        # Fedora

# Utilisation (compatible avec gzip)
pigz file.txt                # Compresse en parallèle
pigz -d file.txt.gz          # Décompresse

# Avec tar
tar -cf - /data/ | pigz > backup.tar.gz

# Nombre de threads
pigz -p 8 file.txt           # Utiliser 8 threads
```

### Pbzip2 (Parallel bzip2)

**Utilisation** :
```bash
# Installation
sudo apt install pbzip2

# Compression parallèle
pbzip2 file.txt

# Avec tar
tar -cf - /data/ | pbzip2 > backup.tar.bz2

# Nombre de threads
pbzip2 -p4 file.txt          # 4 threads
```

### XZ avec threads

**Compression multi-threadée** :
```bash
# XZ avec support threads (xz-utils récent)
xz -T 0 file.txt             # Utiliser tous les CPU disponibles
xz -T 4 file.txt             # Utiliser 4 threads

# Avec tar
tar -cJf backup.tar.xz -T 0 /data/
```

### Optimisation selon le cas d'usage

**Choix de l'algorithme** :
```bash
# Besoin de vitesse → gzip
tar -czf backup.tar.gz /data/

# Besoin de compression maximale → xz
tar -cJf backup.tar.xz /data/

# Compromis → bzip2
tar -cjf backup.tar.bz2 /data/

# Compression parallèle → pigz/pbzip2
tar -cf - /data/ | pigz > backup.tar.gz
```

## Archivage intelligent et automation

### Scripts d'archivage automatique

**Script de sauvegarde quotidienne** :
```bash
#!/bin/bash
# daily_backup.sh

BACKUP_DIR="/backups"
SOURCE_DIR="/home/user"
DATE=$(date +%Y%m%d)
ARCHIVE="${BACKUP_DIR}/backup_${DATE}.tar.gz"

# Créer l'archive
tar -czf "$ARCHIVE" "$SOURCE_DIR"

# Vérifier la création
if [ $? -eq 0 ]; then
    echo "Sauvegarde créée: $ARCHIVE"
    ls -lh "$ARCHIVE"
else
    echo "Erreur lors de la création de la sauvegarde"
    exit 1
fi
```

**Archivage avec rotation** :
```bash
#!/bin/bash
# backup_with_rotation.sh

BACKUP_DIR="/backups"
KEEP_DAYS=7
SOURCE_DIR="/home/user"

# Créer la sauvegarde
DATE=$(date +%Y%m%d)
ARCHIVE="${BACKUP_DIR}/backup_${DATE}.tar.gz"
tar -czf "$ARCHIVE" "$SOURCE_DIR"

# Supprimer les anciennes sauvegardes
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +$KEEP_DAYS -delete
```

### Archivage sélectif

**Exclure des patterns** :
```bash
# Exclure des types de fichiers
tar -czf backup.tar.gz \
    --exclude='*.log' \
    --exclude='*.tmp' \
    --exclude='.git' \
    --exclude='node_modules' \
    /project/

# Exclure via fichier
cat > exclude_patterns.txt << EOF
*.log
*.tmp
.git/
node_modules/
*.cache
EOF

tar -czf backup.tar.gz \
    --exclude-from=exclude_patterns.txt \
    /project/
```

**Archivage incrémentiel** :
```bash
# Fichiers modifiés dans les dernières 24h
tar -czf incremental.tar.gz \
    --newer-mtime="1 day ago" \
    /data/

# Fichiers modifiés depuis une date
tar -czf incremental.tar.gz \
    --newer-mtime="2024-01-01" \
    /data/

# Basé sur un fichier de référence
tar -czf incremental.tar.gz \
    --newer-than=/var/log/last_backup \
    /data/
touch /var/log/last_backup
```

## Vérification d'intégrité

### Checksums et vérification

**MD5** :
```bash
# Créer un checksum
md5sum file.txt > file.txt.md5

# Vérifier
md5sum -c file.txt.md5
```

**SHA256** :
```bash
# Créer un checksum
sha256sum file.txt > file.txt.sha256

# Vérifier
sha256sum -c file.txt.sha256
```

**Avec tar** :
```bash
# Créer archive avec checksum
tar -czf backup.tar.gz /data/
sha256sum backup.tar.gz > backup.tar.gz.sha256

# Vérifier avant extraction
sha256sum -c backup.tar.gz.sha256 && tar -xzf backup.tar.gz
```

### Vérification de l'intégrité des archives

**Test d'extraction** :
```bash
# Tester sans extraire
tar -tzf backup.tar.gz > /dev/null && echo "Archive valide"

# Vérifier avec verbose
tar -tzvf backup.tar.gz | head -10

# Test complet
tar -xzf backup.tar.gz -C /tmp/test/ && echo "Extraction réussie"
```

**Script de vérification** :
```bash
#!/bin/bash
# verify_archive.sh

ARCHIVE="$1"

if [ -z "$ARCHIVE" ]; then
    echo "Usage: $0 <archive.tar.gz>"
    exit 1
fi

# Vérifier l'existence
if [ ! -f "$ARCHIVE" ]; then
    echo "Erreur: Archive non trouvée"
    exit 1
fi

# Vérifier l'intégrité
if tar -tzf "$ARCHIVE" > /dev/null 2>&1; then
    echo "✓ Archive valide"
    
    # Vérifier le checksum si disponible
    if [ -f "${ARCHIVE}.sha256" ]; then
        if sha256sum -c "${ARCHIVE}.sha256" > /dev/null 2>&1; then
            echo "✓ Checksum valide"
        else
            echo "✗ Checksum invalide"
            exit 1
        fi
    fi
else
    echo "✗ Archive corrompue"
    exit 1
fi
```

## Gestion des archives volumineuses

### Archivage de gros volumes

**Archives multi-volumes** :
```bash
# Créer une archive multi-volumes
tar -czf - /large/data/ | split -b 1G - backup_part_

# Reconstituer
cat backup_part_* | tar -xzf -
```

**Archivage avec compression adaptative** :
```bash
# Script adaptatif
#!/bin/bash
# adaptive_backup.sh

SOURCE="$1"
DEST="$2"

# Tester la taille
SIZE=$(du -s "$SOURCE" | cut -f1)

if [ "$SIZE" -gt 10000000 ]; then  # > 10GB
    echo "Grand volume détecté, utilisation de xz"
    tar -cJf "${DEST}.tar.xz" "$SOURCE"
else
    echo "Volume moyen, utilisation de gzip"
    tar -czf "${DEST}.tar.gz" "$SOURCE"
fi
```

### Optimisation de l'espace

**Déduplication avant archivage** :
```bash
# Trouver les fichiers dupliqués
fdupes -r /data/ > duplicates.txt

# Créer des liens symboliques
# (nécessite un script personnalisé)

# Puis archiver
tar -czf backup.tar.gz /data/
```

**Compression différentielle** :
```bash
# Créer une archive de référence
tar -czf full_backup.tar.gz /data/

# Créer une archive différentielle
tar -czf diff_backup.tar.gz \
    --newer-than=full_backup.tar.gz \
    /data/
```

## Conclusion

La maîtrise de la compression et de l'archivage est essentielle pour l'administration système efficace. Chaque algorithme a ses forces : gzip pour la vitesse, bzip2 pour le compromis, xz pour la compression maximale. Le choix dépend de vos contraintes de temps, d'espace et de ressources CPU.

Les techniques avancées - compression parallèle, archivage incrémentiel, vérification d'intégrité - permettent d'optimiser les sauvegardes et les transferts de données à grande échelle. L'automatisation de ces processus libère du temps pour d'autres tâches critiques.

Dans le prochain chapitre, nous explorerons la gestion des utilisateurs et groupes, fondamental pour la sécurité et l'organisation des systèmes multi-utilisateurs.