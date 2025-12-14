# Chapitre 50 - Cron avancé : expressions complexes

## Table des matières
- [Introduction](#introduction)
- [Expressions cron avancées](#expressions-cron-avancées)
- [Syntaxe étendue](#syntaxe-étendue)
- [Expressions conditionnelles](#expressions-conditionnelles)
- [Gestion des fuseaux horaires](#gestion-des-fuseaux-horaires)
- [Planification relative](#planification-relative)
- [Expressions non-standard](#expressions-non-standard)
- [Cas d'usage complexes](#cas-dusage-complexes)
- [Débogage des expressions](#débogage-des-expressions)
- [Validation et test](#validation-et-test)
- [Optimisation des performances](#optimisation-des-performances)
- [Alternatives avancées](#alternatives-avancées)
- [Scripts d'automatisation](#scripts-dautomatisation)
- [Conclusion](#conclusion)

## Introduction

Les expressions cron avancées permettent de créer des planifications sophistiquées qui vont bien au-delà des cas d'usage simples. De la gestion des jours ouvrables aux calculs de dates complexes, ces expressions permettent d'automatiser des scénarios qui semblaient impossibles avec la syntaxe de base.

Imaginez les expressions cron avancées comme un langage de programmation dédié à la planification : elles permettent d'exprimer des intentions complexes comme "le dernier jour ouvrable du mois", "toutes les 3 heures sauf entre 22h et 6h", ou "le premier lundi après le 15 de chaque mois". Cette puissance ouvre des possibilités d'automatisation qui s'adaptent aux besoins métier les plus spécifiques.

## Expressions cron avancées

### Syntaxe complète

**Format étendu** :
```bash
# Format standard (5 champs)
* * * * * command

# Format étendu (6 champs - seconde optionnelle)
# * * * * * * command  (certaines implémentations)

# Format avec année (7 champs - certaines implémentations)
# * * * * * * * command
```

**Champs détaillés** :
```bash
# Minute (0-59)
# Heure (0-23)
# Jour du mois (1-31)
# Mois (1-12 ou JAN-DEC)
# Jour de la semaine (0-7, 0/7=dimanche, ou SUN-SAT)
# Année (optionnel, 1970-2099)
```

### Opérateurs avancés

**Opérateurs disponibles** :
```bash
# * : Toutes les valeurs
* * * * * command  # Toutes les minutes

# , : Liste de valeurs
0 9,12,18 * * * command  # À 9h, 12h, 18h

# - : Plage de valeurs
0 9-17 * * 1-5 command  # 9h-17h, lun-ven

# / : Pas (step)
*/15 * * * * command  # Toutes les 15 minutes
0 */2 * * * command    # Toutes les 2 heures
0 0 */3 * * command    # Tous les 3 jours

# ? : Valeur non spécifiée (jour du mois OU jour de la semaine)
0 0 ? * 1 command  # Tous les lundis à minuit
0 0 1 * ? command  # Le 1er de chaque mois

# L : Dernier (dernier jour du mois, dernier jour de la semaine)
0 0 L * * command      # Dernier jour du mois
0 0 ? * L command       # Dernier jour de la semaine (samedi)
0 0 L-3 * * command     # 3 jours avant la fin du mois

# W : Jour ouvrable le plus proche
0 0 15W * * command     # Jour ouvrable le plus proche du 15

# # : Nième occurrence d'un jour de la semaine
0 0 ? * 1#2 command      # 2ème lundi du mois
0 0 ? * MON#1 command    # 1er lundi du mois
```

## Syntaxe étendue

### Expressions avec noms

**Noms de mois et jours** :
```bash
# Mois
0 0 1 JAN * command      # 1er janvier
0 0 1 JAN-MAR * command  # Janvier à mars
0 0 1 JAN,APR,JUL,OCT * command  # Trimestres

# Jours de la semaine
0 0 * * MON command       # Tous les lundis
0 0 * * MON-FRI command   # Lundi à vendredi
0 0 * * MON,WED,FRI command  # Lundi, mercredi, vendredi
```

### Combinaisons complexes

**Expressions combinées** :
```bash
# Heures de bureau, jours ouvrables
0 9-17 * * 1-5 command   # 9h-17h, lun-ven

# Plusieurs créneaux horaires
0 8,12,18 * * * command  # 8h, 12h, 18h tous les jours

# Jours spécifiques du mois
0 0 1,15 * * command     # 1er et 15 de chaque mois

# Combinaison jour/mois
0 0 1 1,4,7,10 * command # Premier jour de chaque trimestre
```

## Expressions conditionnelles

### Conditions métier

**Jours ouvrables** :
```bash
# Dernier jour ouvrable du mois
# Nécessite un script wrapper car cron ne le supporte pas directement
0 0 28-31 * * [ $(date -d tomorrow +\%d) -eq 1 ] && [ $(date +\%u) -le 5 ] && command

# Script wrapper pour dernier jour ouvrable
#!/bin/bash
# /usr/local/bin/last_weekday.sh
TODAY=$(date +%d)
TOMORROW_MONTH=$(date -d tomorrow +%m)
TODAY_MONTH=$(date +%m)

if [ "$TOMORROW_MONTH" != "$TODAY_MONTH" ]; then
    # Demain est un nouveau mois, donc aujourd'hui est le dernier jour
    DAY_OF_WEEK=$(date +%u)
    if [ "$DAY_OF_WEEK" -le 5 ]; then
        # C'est un jour ouvrable
        /usr/local/bin/monthly_task.sh
    fi
fi
```

**Planification conditionnelle** :
```bash
# Exécuter seulement si certaines conditions sont remplies
0 */2 * * * [ -f /var/run/maintenance.flag ] && /usr/local/bin/maintenance.sh

# Exécuter selon la charge système
*/5 * * * * [ $(uptime | awk '{print $(NF-2)}' | cut -d, -f1) -lt 1.0 ] && /usr/local/bin/heavy_task.sh
```

## Gestion des fuseaux horaires

### Fuseaux horaires dans cron

**Configuration du fuseau horaire** :
```bash
# Définir le fuseau horaire dans le crontab
TZ=Europe/Paris
0 9 * * * command  # 9h heure de Paris

# Fuseau horaire système
# /etc/timezone
echo "Europe/Paris" | sudo tee /etc/timezone
sudo dpkg-reconfigure -f noninteractive tzdata

# Variables d'environnement
export TZ=America/New_York
crontab -e  # Les expressions utilisent le fuseau configuré
```

**Conversion de fuseaux** :
```bash
#!/bin/bash
# Script avec conversion de fuseau horaire

# Exécuter à 9h heure de New York, peu importe le fuseau du serveur
TZ=America/New_York date +%H  # Affiche l'heure à New York

# Dans le crontab
TZ=America/New_York
0 9 * * * /usr/local/bin/daily_task.sh
```

## Planification relative

### Intervalles relatifs

**Depuis le démarrage** :
```bash
# Exécuter X minutes après le démarrage
@reboot sleep 300 && /usr/local/bin/post_boot.sh

# Exécuter toutes les X minutes depuis le démarrage
# Nécessite systemd timer plutôt que cron
```

**Intervalles depuis la dernière exécution** :
```bash
#!/bin/bash
# Exécution relative à la dernière exécution

LAST_RUN_FILE="/var/run/last_backup"
INTERVAL_HOURS=6

if [ -f "$LAST_RUN_FILE" ]; then
    LAST_RUN=$(cat "$LAST_RUN_FILE")
    NOW=$(date +%s)
    ELAPSED=$(( (NOW - LAST_RUN) / 3600 ))
    
    if [ $ELAPSED -ge $INTERVAL_HOURS ]; then
        /usr/local/bin/backup.sh
        echo "$NOW" > "$LAST_RUN_FILE"
    fi
else
    /usr/local/bin/backup.sh
    echo "$(date +%s)" > "$LAST_RUN_FILE"
fi
```

## Expressions non-standard

### Extensions Vixie cron

**Extensions courantes** :
```bash
# @yearly ou @annually
@yearly /usr/local/bin/yearly_task.sh  # 0 0 1 1 *

# @monthly
@monthly /usr/local/bin/monthly_task.sh  # 0 0 1 * *

# @weekly
@weekly /usr/local/bin/weekly_task.sh  # 0 0 * * 0

# @daily ou @midnight
@daily /usr/local/bin/daily_task.sh  # 0 0 * * *

# @hourly
@hourly /usr/local/bin/hourly_task.sh  # 0 * * * *

# @reboot
@reboot /usr/local/bin/startup_task.sh
```

### Expressions avec secondes

**Cron avec secondes (certaines implémentations)** :
```bash
# Format: seconde minute heure jour mois jour-semaine
# * * * * * * command

# Toutes les 30 secondes
*/30 * * * * * command

# À la seconde 0 de chaque minute
0 * * * * * command
```

## Cas d'usage complexes

### Dernier jour du mois

**Dernier jour du mois** :
```bash
# Méthode 1: Vérifier si demain est le 1er
0 0 28-31 * * [ $(date -d tomorrow +\%d) -eq 1 ] && command

# Méthode 2: Script wrapper
#!/bin/bash
# /usr/local/bin/last_day_of_month.sh
TODAY=$(date +%d)
TOMORROW=$(date -d tomorrow +%d)

if [ "$TOMORROW" -eq 1 ]; then
    /usr/local/bin/monthly_task.sh
fi

# Dans crontab
0 0 28-31 * * /usr/local/bin/last_day_of_month.sh
```

### Premier lundi du mois

**Premier lundi** :
```bash
# Script pour trouver le premier lundi
#!/bin/bash
# /usr/local/bin/first_monday.sh
FIRST_DAY=$(date -d "$(date +%Y-%m-01)" +%u)
MONDAY_OFFSET=$(( (8 - FIRST_DAY) % 7 ))
FIRST_MONDAY=$((1 + MONDAY_OFFSET))

if [ "$(date +%d)" -eq "$FIRST_MONDAY" ]; then
    /usr/local/bin/monthly_monday_task.sh
fi

# Dans crontab
0 0 1-7 * * [ $(date +\%u) -eq 1 ] && /usr/local/bin/first_monday.sh
```

### Jours ouvrables uniquement

**Filtrage jours ouvrables** :
```bash
# Exécuter seulement les jours ouvrables
0 9 * * 1-5 /usr/local/bin/business_task.sh

# Avec vérification supplémentaire
0 9 * * * [ $(date +\%u) -le 5 ] && /usr/local/bin/business_task.sh
```

### Heures de bureau

**Heures de bureau avec pauses** :
```bash
# 9h-12h et 14h-18h, lun-ven
0 9-11 * * 1-5 /usr/local/bin/morning_tasks.sh
0 14-17 * * 1-5 /usr/local/bin/afternoon_tasks.sh

# Avec exclusion de l'heure de déjeuner
0 9-11,14-17 * * 1-5 /usr/local/bin/business_hours_task.sh
```

## Débogage des expressions

### Outils de validation

**Validation d'expression** :
```bash
#!/bin/bash
# Validateur d'expressions cron

validate_cron_expression() {
    local expr="$1"
    
    # Vérifier le nombre de champs
    local fields=$(echo "$expr" | awk '{print NF}')
    if [ "$fields" -lt 5 ] || [ "$fields" -gt 7 ]; then
        echo "ERREUR: Nombre de champs incorrect ($fields)"
        return 1
    fi
    
    # Vérifier chaque champ
    local minute=$(echo "$expr" | awk '{print $1}')
    local hour=$(echo "$expr" | awk '{print $2}')
    local day=$(echo "$expr" | awk '{print $3}')
    local month=$(echo "$expr" | awk '{print $4}')
    local weekday=$(echo "$expr" | awk '{print $5}')
    
    # Validation basique
    echo "$minute" | grep -qE '^(\*|[0-5]?[0-9]|([0-5]?[0-9](-[0-5]?[0-9])?)(,([0-5]?[0-9](-[0-5]?[0-9])?))*|(\*|[0-5]?[0-9](-[0-5]?[0-9])?)(/[0-9]+)?)$' || {
        echo "ERREUR: Format de minute invalide"
        return 1
    }
    
    echo "Expression valide"
    return 0
}
```

### Test d'expressions

**Tester les prochaines exécutions** :
```bash
#!/bin/bash
# Testeur d'expressions cron

test_cron_expression() {
    local expr="$1"
    local count="${2:-10}"  # Nombre d'exécutions à afficher
    
    # Utiliser crontab pour tester (nécessite installation)
    if command -v crontab >/dev/null 2>&1; then
        echo "Expression: $expr"
        echo "Prochaines $count exécutions:"
        
        # Créer un crontab temporaire
        echo "$expr /bin/echo test" | crontab -
        
        # Utiliser un outil externe pour calculer les prochaines exécutions
        # (nécessite une bibliothèque comme python-crontab)
    else
        echo "crontab non disponible"
    fi
}
```

## Validation et test

### Outils en ligne de commande

**Validation avec Python** :
```bash
#!/bin/bash
# Validation avec python-crontab

validate_with_python() {
    local expr="$1"
    
    python3 << EOF
from crontab import CronTab

try:
    cron = CronTab("$expr")
    print("Expression valide")
    print(f"Prochaines exécutions:")
    for i, job in enumerate(cron.schedule().get_next(10)):
        print(f"  {i+1}. {job}")
except Exception as e:
    print(f"ERREUR: {e}")
    exit(1)
EOF
}
```

### Simulation d'exécution

**Simulateur d'exécution** :
```bash
#!/bin/bash
# Simulateur d'exécution cron

simulate_cron() {
    local expr="$1"
    local start_date="${2:-$(date +%Y-%m-%d)}"
    local days="${3:-30}"
    
    echo "Simulation de l'expression: $expr"
    echo "Période: $start_date à $(date -d "$start_date +$days days" +%Y-%m-%d)"
    echo ""
    
    # Parser l'expression et simuler
    # (Nécessite une implémentation complète)
}
```

## Optimisation des performances

### Réduction de la charge

**Optimisation des expressions** :
```bash
# Éviter les expressions trop fréquentes
# ❌ Mauvais: Toutes les secondes
* * * * * * command

# ✅ Bon: Toutes les minutes
* * * * * command

# Éviter les expressions complexes dans cron
# ❌ Mauvais: Logique complexe dans l'expression
0 0 * * * [ condition1 ] && [ condition2 ] && command

# ✅ Bon: Script wrapper
0 0 * * * /usr/local/bin/wrapper.sh
```

### Regroupement de tâches

**Regrouper les tâches similaires** :
```bash
# ❌ Mauvais: Plusieurs tâches séparées
0 9 * * * task1.sh
0 9 * * * task2.sh
0 9 * * * task3.sh

# ✅ Bon: Une seule tâche qui appelle plusieurs scripts
0 9 * * * /usr/local/bin/daily_tasks.sh

# daily_tasks.sh
#!/bin/bash
/usr/local/bin/task1.sh
/usr/local/bin/task2.sh
/usr/local/bin/task3.sh
```

## Alternatives avancées

### Systemd timers

**Timers avancés** :
```bash
# /etc/systemd/system/advanced.timer
[Unit]
Description=Advanced timer example

[Timer]
# Calendrier complexe
OnCalendar=Mon..Fri 09:00
OnCalendar=*-*-01 00:00:00
OnCalendar=*-12-25 00:00:00

# Intervalles relatifs
OnBootSec=5min
OnUnitActiveSec=1h

# Aléatoire pour éviter les pics
RandomizedDelaySec=300

[Install]
WantedBy=timers.target
```

### Anacron pour les machines intermittentes

**Configuration anacron** :
```bash
# /etc/anacrontab
# period delay job-id command
1       5       cron.daily      nice run-parts /etc/cron.daily
7       10      cron.weekly     nice run-parts /etc/cron.weekly
30      15      cron.monthly    nice run-parts /etc/cron.monthly
```

## Scripts d'automatisation

### Gestionnaire de crontabs

**Gestionnaire complet** :
```bash
#!/bin/bash
# Gestionnaire de crontabs avancé

set -euo pipefail

CRON_MANAGER_HOME="/opt/cron-manager"
CRON_CONFIG="${CRON_MANAGER_HOME}/config.conf"

# Charger la configuration
load_config() {
    if [ -f "$CRON_CONFIG" ]; then
        source "$CRON_CONFIG"
    fi
}

# Ajouter une tâche cron
add_cron_job() {
    local expr="$1"
    local command="$2"
    local user="${3:-$USER}"
    
    # Valider l'expression
    if ! validate_cron_expression "$expr"; then
        echo "ERREUR: Expression invalide"
        return 1
    fi
    
    # Ajouter au crontab
    (crontab -u "$user" -l 2>/dev/null; echo "$expr $command") | crontab -u "$user" -
    
    echo "Tâche ajoutée pour $user"
}

# Lister les tâches
list_cron_jobs() {
    local user="${1:-$USER}"
    
    echo "Tâches cron pour $user:"
    crontab -u "$user" -l 2>/dev/null || echo "Aucune tâche"
}

# Supprimer une tâche
remove_cron_job() {
    local pattern="$1"
    local user="${2:-$USER}"
    
    crontab -u "$user" -l 2>/dev/null | grep -v "$pattern" | crontab -u "$user" -
    
    echo "Tâches correspondant à '$pattern' supprimées"
}

# Backup des crontabs
backup_crontabs() {
    local backup_dir="${1:-/var/backups/crontabs}"
    local timestamp=$(date +%Y%m%d_%H%M%S)
    
    mkdir -p "$backup_dir"
    
    # Backup crontab système
    if [ -f /etc/crontab ]; then
        cp /etc/crontab "$backup_dir/crontab_system_$timestamp"
    fi
    
    # Backup crontabs utilisateurs
    for user in $(cut -d: -f1 /etc/passwd); do
        local user_cron=$(crontab -u "$user" -l 2>/dev/null)
        if [ -n "$user_cron" ]; then
            echo "$user_cron" > "$backup_dir/crontab_${user}_$timestamp"
        fi
    done
    
    echo "Backup créé dans $backup_dir"
}

# Utilisation
# add_cron_job "0 9 * * 1-5" "/usr/local/bin/daily.sh" "root"
# list_cron_jobs "root"
# remove_cron_job "daily.sh" "root"
# backup_crontabs
```

## Conclusion

Les expressions cron avancées permettent de créer des planifications sophistiquées qui s'adaptent aux besoins métier les plus complexes. En maîtrisant les opérateurs avancés, les expressions conditionnelles, et les techniques de débogage, vous pouvez créer des systèmes d'automatisation qui exécutent les tâches exactement quand elles sont nécessaires.

Un système bien conçu utilise des expressions optimisées, des scripts wrappers pour la logique complexe, et des alternatives comme systemd timers pour les cas d'usage modernes. La compréhension approfondie de ces mécanismes permet de créer des planifications qui s'adaptent aux besoins spécifiques de chaque environnement.

Dans le chapitre suivant, nous explorerons le scripting modulaire et les bibliothèques, découvrant comment créer des scripts réutilisables et maintenables.

