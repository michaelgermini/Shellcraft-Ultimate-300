# Chapitre 13 - Concepts de processus et multitâche

## Table des matières
- [Introduction](#introduction)
- [Qu'est-ce qu'un processus ?](#quest-ce-quun-processus-)
- [Le cycle de vie d'un processus](#le-cycle-de-vie-dun-processus)
- [Hiérarchie et relations parent-enfant](#hiérarchie-et-relations-parent-enfant)
- [États des processus](#états-des-processus)
- [Signaux et communication inter-processus](#signaux-et-communication-inter-processus)
- [Multitâche et ordonnancement](#multitâche-et-ordonnancement)
- [Commandes de gestion des processus](#commandes-de-gestion-des-processus)
- [Processus en arrière-plan et foreground](#processus-en-arrière-plan-et-foreground)
- [Limites et ressources des processus](#limites-et-ressources-des-processus)
- [Débogage et monitoring](#débogage-et-monitoring)
- [Conclusion](#conclusion)

## Introduction

Les processus représentent l'unité fondamentale d'exécution dans les systèmes informatiques modernes. Comprendre leur fonctionnement, leur cycle de vie, et leurs interactions permet non seulement de maîtriser l'administration système, mais aussi d'écrire des programmes plus efficaces et robustes.

Imaginez les processus comme les acteurs d'une pièce de théâtre complexe : chacun joue son rôle spécifique, communique avec les autres, et contribue à l'ensemble de la performance. Un bon metteur en scène (le système d'exploitation) coordonne parfaitement tous ces acteurs pour créer une représentation harmonieuse.

## Qu'est-ce qu'un processus ?

### Définition technique

**Un processus** est une instance d'un programme en cours d'exécution, avec :
- **Code exécutable** : les instructions du programme
- **Données** : variables et structures en mémoire
- **Contexte d'exécution** : registres CPU, pile d'appel, pointeur d'instruction
- **Ressources système** : fichiers ouverts, connexions réseau, mémoire allouée

### Analogie avec la cuisine

Considérez un processus comme un chef cuisinier dans une cuisine professionnelle :
- **Recette** = Code du programme
- **Ingrédients** = Données d'entrée
- **Ustensiles** = Ressources système
- **Actions en cours** = État d'exécution
- **Plat final** = Sortie du programme

### Caractéristiques essentielles

**Identité unique** :
```bash
# Chaque processus a un PID unique
echo "Mon PID: $$"
ps aux | grep "$$"
```

**Isolation mémoire** :
- Chaque processus a son propre espace d'adressage
- Protection contre les interférences d'autres processus
- Communication contrôlée via des mécanismes spécifiques

**Ressources indépendantes** :
- CPU alloué dynamiquement
- Mémoire gérée par le système
- Fichiers et périphériques accessibles via des descripteurs

## Le cycle de vie d'un processus

### Création d'un processus

**Fork : duplication du parent** :
```bash
# Exemple simple de fork en C
#include <unistd.h>
#include <stdio.h>

int main() {
    pid_t pid = fork();

    if (pid == 0) {
        // Processus enfant
        printf("Je suis l'enfant, PID: %d\n", getpid());
    } else {
        // Processus parent
        printf("Je suis le parent, PID: %d, enfant: %d\n", getpid(), pid);
    }
    return 0;
}
```

**Exec : remplacement du code** :
```bash
# En shell, chaque commande externe crée un processus
ls -l
# 1. Fork du shell
# 2. Exec de /bin/ls dans le processus enfant
# 3. Attente de fin par le parent
```

### Terminaison d'un processus

**Sortie normale** :
```bash
# Code de retour
exit 0  # Succès
exit 1  # Erreur
echo "$?"  # Afficher le code de retour du dernier processus
```

**Terminaison par signal** :
```bash
# Interruption par Ctrl+C (SIGINT)
sleep 100
# Ctrl+C envoie SIGINT

# Terminaison forcée
kill -9 1234  # SIGKILL (ne peut pas être ignoré)
```

## Hiérarchie et relations parent-enfant

### Arbre des processus

**Visualisation de la hiérarchie** :
```bash
# Arbre complet des processus
pstree

# Arbre avec PIDs
pstree -p

# Focus sur un utilisateur
pstree alice
```

**Processus init (PID 1)** :
```bash
# Le père de tous les processus
ps -p 1
# Sur systemd: /usr/lib/systemd/systemd
# Sur anciens systèmes: /sbin/init
```

### Relations parent-enfant

**PPID (Parent Process ID)** :
```bash
# Afficher le PPID
echo "Mon PPID: $PPID"

# Lister avec PPID
ps -eo pid,ppid,cmd | head -10
```

**Orphelins et zombies** :
```bash
# Processus orphelin (parent terminé)
# Adopté automatiquement par init

# Processus zombie (terminé mais pas attendu par le parent)
ps aux | grep "Z+"  # État Zombie
```

## États des processus

### Les cinq états principaux

**Running (R)** : En cours d'exécution sur CPU
```bash
ps aux | grep " R "
# Processus actif utilisant le CPU
```

**Sleeping (S)** : En attente d'événement
```bash
ps aux | grep " S "
# Attente d'I/O, signal, etc.
```

**Stopped (T)** : Arrêté (par signal STOP)
```bash
# Arrêt temporaire
kill -STOP 1234
ps -p 1234  # État T

# Reprise
kill -CONT 1234
```

**Zombie (Z)** : Terminé mais pas nettoyé
```bash
ps aux | grep " Z "
# Attend que le parent fasse wait()
```

**Dead (X)** : En cours de terminaison (rarement visible)

### Transitions d'états

**Cycle typique** :
```
Création → Running → Sleeping → Running → Terminated
    ↓         ↓         ↓         ↓         ↓
  fork()   I/O      I/O       CPU      exit()
                attente  complété
```

## Signaux et communication inter-processus

### Signaux POSIX standards

**Signaux courants** :
```bash
# SIGTERM (15) : Terminaison propre
kill 1234

# SIGKILL (9) : Terminaison forcée
kill -9 1234

# SIGSTOP (19) : Arrêt
kill -STOP 1234

# SIGCONT (18) : Reprise
kill -CONT 1234

# SIGHUP (1) : Rechargement configuration
kill -HUP 1234  # Souvent pour daemons
```

### Gestion des signaux en shell

**Trap : interception de signaux** :
```bash
#!/bin/bash
# Gestion propre de l'arrêt
cleanup() {
    echo "Nettoyage avant arrêt..."
    rm -f /tmp/tempfile
    exit 0
}

trap cleanup SIGINT SIGTERM

echo "Script en cours... Ctrl+C pour arrêter"
sleep 100
```

**Signaux ignorés** :
```bash
# Ignorer SIGINT (Ctrl+C)
trap '' SIGINT
sleep 10  # Ctrl+C sera ignoré

# Restaurer le comportement normal
trap - SIGINT
```

## Multitâche et ordonnancement

### Ordonnanceur du système

**Principe de l'ordonnancement** :
- **Préemptif** : Le système interrompt les processus pour partager le CPU
- **Priorité** : Certains processus ont la priorité sur d'autres
- **Quantum** : Temps alloué à chaque processus avant commutation

**Visualisation de l'activité** :
```bash
# Charge système
uptime
# load average: 0.52, 0.58, 0.59

# Utilisation CPU par processus
top
htop  # Version améliorée
```

### Nice et priorité

**Modification de la priorité** :
```bash
# Lancer avec faible priorité
nice -n 10 long_task.sh

# Changer la priorité d'un processus existant
renice +5 1234

# Priorité en temps réel (root seulement)
chrt --rr 50 task
```

**Niveaux de nice** :
- **-20** : Très haute priorité
- **0** : Priorité normale (défaut)
- **19** : Très basse priorité

## Commandes de gestion des processus

### ps : instantané des processus

**Utilisation de base** :
```bash
# Processus de l'utilisateur
ps

# Tous les processus
ps aux

# Arbre des processus
ps -ejH
ps f  # Alternative simple
```

**Filtres et recherche** :
```bash
# Processus par nom
ps aux | grep nginx

# Processus par utilisateur
ps -u alice

# Processus par PID
ps -p 1234,5678
```

### top et htop : monitoring en temps réel

**top : monitoring standard** :
```bash
top
# Appuyer sur:
# h : aide
# k : tuer un processus
# r : renice
# q : quitter
```

**htop : version améliorée** :
```bash
htop
# Interface graphique dans le terminal
# Navigation au clavier
# Arbre des processus
```

### kill et signaux

**Syntaxe complète** :
```bash
# Signal par numéro
kill -15 1234  # SIGTERM
kill -9 1234   # SIGKILL

# Signal par nom
kill -TERM 1234
kill -KILL 1234

# Plusieurs processus
kill -HUP 1234 5678 9012
```

**killall : par nom** :
```bash
# Tuer tous les processus d'un programme
killall firefox
killall -9 python3  # Forcé
```

## Processus en arrière-plan et foreground

### Gestion des jobs

**Lancer en arrière-plan** :
```bash
# Lancement immédiat en background
long_task.sh &

# Passage en background d'une tâche foreground
# Ctrl+Z pour suspendre, puis:
bg

# Lister les jobs
jobs
# [1]+  Running                 long_task.sh &
```

**Gestion des jobs** :
```bash
# Passage au foreground
fg %1  # Job numéro 1

# Arrêt d'un job
kill %2  # Job numéro 2

# Liste détaillée
jobs -l
```

### Disown : détachement complet

**Détachement d'un processus** :
```bash
# Lancement détaché
long_task.sh &
disown

# Ou détacher un job existant
disown %1
```

**Différences** :
- **Background (&)** : Attaché à la session shell
- **Disown** : Complètement détaché, survit à la fermeture du shell

## Limites et ressources des processus

### Limites système

**Visualisation des limites** :
```bash
# Limites actuelles
ulimit -a

# Limite de fichiers ouverts
ulimit -n

# Limite de taille de fichier
ulimit -f
```

**Types de limites** :
- **Soft limit** : Limite actuelle (peut être augmentée)
- **Hard limit** : Limite maximale (root seulement)

**Modification des limites** :
```bash
# Augmenter le nombre de fichiers ouverts
ulimit -n 4096

# Supprimer la limite de taille de fichier
ulimit -f unlimited

# Configuration permanente dans /etc/security/limits.conf
```

### Gestion des ressources

**cgroups : contrôle des groupes** :
```bash
# Créer un groupe de contrôle
cgcreate -g cpu,memory:mygroup

# Lancer un processus dans le groupe
cgexec -g cpu,memory:mygroup long_task.sh

# Limiter l'utilisation CPU
cgset -r cpu.shares=512 mygroup
```

**Monitoring des ressources** :
```bash
# Utilisation mémoire par processus
ps aux --sort=-%mem | head

# Utilisation CPU
ps aux --sort=-%cpu | head
```

## Débogage et monitoring

### Outils de débogage

**strace : appels système** :
```bash
# Tracer les appels système
strace ls -l

# Tracer un processus existant
strace -p 1234

# Sauvegarder la trace
strace -o trace.log programme
```

**gdb : débogueur** :
```bash
# Attacher à un processus
gdb -p 1234

# Dans gdb:
# bt : backtrace
# info threads : threads
# continue : reprendre
```

### Monitoring avancé

**System Activity Reporter** :
```bash
# Rapport quotidien
sar -u 1 5  # CPU chaque seconde, 5 fois

# Historique
sar -f /var/log/sa/sa$(date +%d)

# I/O disques
sar -d 1 5
```

**Performance monitoring** :
```bash
# perf : profiling avancé
perf record -p 1234
perf report

# Flame graphs pour l'analyse de performance
```

## Conclusion

Les processus constituent l'essence même de l'informatique moderne. Leur compréhension approfondie - du cycle de vie aux mécanismes d'ordonnancement, en passant par la communication inter-processus - permet non seulement d'administrer efficacement les systèmes, mais aussi d'écrire des programmes plus robustes et performants.

Dans le chapitre suivant, nous explorerons l'introduction au réseau, qui explique comment les processus communiquent non seulement entre eux sur une même machine, mais aussi à travers le monde entier via les réseaux informatiques.
