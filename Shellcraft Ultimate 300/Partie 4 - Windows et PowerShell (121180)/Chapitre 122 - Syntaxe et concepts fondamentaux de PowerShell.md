# Chapitre 122 - Syntaxe et concepts fondamentaux de PowerShell

> "La syntaxe PowerShell est comme une conversation avec votre système d'exploitation - claire, cohérente, et incroyablement puissante." - Expert PowerShell

## Introduction : La grammaire du shell objet

Maintenant que nous avons posé les bases philosophiques et historiques de PowerShell, plongeons dans sa syntaxe et ses concepts fondamentaux. PowerShell possède une grammaire riche et cohérente qui, une fois maîtrisée, permet d'exprimer des idées complexes avec une élégance surprenante.

Dans ce chapitre, nous explorerons la syntaxe de base, les conventions de nommage, les opérateurs, et les structures de contrôle qui forment le vocabulaire de PowerShell. Nous verrons comment ces éléments s'articulent pour créer des scripts puissants et maintenables.

## Section 1 : Syntaxe de base et conventions

### 1.1 Conventions de nommage des cmdlets

PowerShell suit des conventions strictes pour nommer ses cmdlets :

```powershell
# Structure : Verbe-Nom
# Verbes standardisés
Get-Command        # Obtenir des informations
Set-Variable       # Définir/modifier des valeurs
New-Object         # Créer de nouveaux objets
Remove-Item        # Supprimer des éléments
Add-Member         # Ajouter des propriétés/méthodes
Clear-Host         # Effacer/nettoyer

# Pluriel pour les collections
Get-Process        # Plusieurs processus
Get-Service        # Plusieurs services
Get-ChildItem      # Plusieurs éléments enfants
```

**Verbes approuvés (liste non exhaustive) :**
```powershell
Get-    # Obtenir, lire, récupérer
Set-    # Définir, modifier, configurer
New-    # Créer, instancier
Remove- # Supprimer, détruire
Add-    # Ajouter à une collection
Clear-  # Vider, nettoyer (sans supprimer)
Copy-   # Copier
Move-   # Déplacer
Rename- # Renommer
Import- # Importer
Export- # Exporter
```

### 1.2 Syntaxe des paramètres

PowerShell offre plusieurs façons de passer des paramètres :

```powershell
# Paramètres positionnels
Get-ChildItem C:\Windows\System32 *.dll

# Paramètres nommés (recommandé)
Get-ChildItem -Path C:\Windows\System32 -Filter *.dll -Recurse

# Paramètres avec abbréviations
Get-ChildItem -Pa C:\Windows -Fi *.exe -R

# Paramètres booléens
Get-Process -IncludeUserName
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Force

# Paramètres avec valeurs multiples
Get-Process -Name powershell, notepad, explorer
```

### 1.3 Guillemets et littéraux

```powershell
# Guillemets simples : littéraux (pas d'expansion)
$chemin = 'C:\Program Files\PowerShell'
$message = 'Bonjour $env:USERNAME'  # $env:USERNAME non expansé

# Guillemets doubles : expansion de variables
$message = "Bonjour $env:USERNAME"
$rapport = "Fichier: $chemin, Utilisateur: $env:USERNAME"

# Guillemets pour les propriétés d'objet
$user = Get-ADUser -Identity "john.doe"
$processus = Get-Process -Name "powershell*"

# Échappement
$message = "Il a dit `"Bonjour`""
$regex = "motif avec \[crochets\]"
```

## Section 2 : Variables et types de données

### 2.1 Déclaration et portée des variables

```powershell
# Variables simples
$maVariable = "Hello World"
$nombre = 42
$booleen = $true

# Variables typées (recommandé pour la robustesse)
[string]$nom = "Alice"
[int]$age = 30
[DateTime]$date = Get-Date

# Variables automatiques
$$         # Dernière ligne saisie
$?         # Succès de la dernière opération
$LASTEXITCODE  # Code de sortie du dernier programme externe
$PID       # ID du processus PowerShell
$PSVersionTable  # Informations de version

# Variables d'environnement
$env:PATH
$env:COMPUTERNAME
$env:USERNAME

# Portée des variables
$globale = "visible partout"          # Portée globale
$script:variable = "portée script"    # Portée script
$local:variable = "portée locale"     # Portée fonction/locale
$private:variable = "privée"          # Non visible depuis enfant
```

### 2.2 Types de données fondamentaux

```powershell
# Types numériques
[int]$entier = 42
[long]$grandEntier = 9223372036854775807
[double]$decimal = 3.14159
[decimal]$precision = 123.456789012345678901234567890

# Chaînes de caractères
[string]$chaine = "Hello World"
[string]$multiligne = @"
Ceci est une chaîne
sur plusieurs lignes
"@

# Booléens
[bool]$vrai = $true
[bool]$faux = $false

# Dates et heures
[DateTime]$maintenant = Get-Date
[TimeSpan]$duree = New-TimeSpan -Hours 2 -Minutes 30

# Tableaux
[array]$nombres = 1, 2, 3, 4, 5
[string[]]$couleurs = "Rouge", "Vert", "Bleu"

# Hashtables (dictionnaires)
[hashtable]$personne = @{
    Nom = "Dupont"
    Prenom = "Jean"
    Age = 35
    Ville = "Paris"
}

# Types personnalisés
Add-Type -TypeDefinition @"
public class Employe {
    public string Nom { get; set; }
    public int Age { get; set; }
    public decimal Salaire { get; set; }
}
"@

[Employe]$employe = New-Object Employe
$employe.Nom = "Alice"
$employe.Age = 28
$employe.Salaire = 45000.50
```

### 2.3 Tableaux et collections

```powershell
# Création de tableaux
$tableauVide = @()
$tableauNombres = @(1, 2, 3, 4, 5)
$tableauStrings = @("un", "deux", "trois")

# Opérations sur tableaux
$tableauNombres.Length        # Taille
$tableauNombres[0]           # Premier élément
$tableauNombres[-1]          # Dernier élément
$tableauNombres[1..3]        # Slice
$tableauNombres += 6         # Ajout d'élément

# Tableaux strongly typed
[int[]]$nombresEntiers = 1, 2, 3, 4, 5

# Arrays génériques
$liste = New-Object System.Collections.Generic.List[string]
$liste.Add("élément1")
$liste.Add("élément2")

# Tri et filtrage
$nombres | Sort-Object
$nombres | Where-Object { $_ -gt 3 }
```

### 2.4 Hashtables et dictionnaires

```powershell
# Création de hashtables
$config = @{
    Serveur = "localhost"
    Port = 8080
    SSL = $true
    Timeout = 30
}

# Accès aux valeurs
$config.Serveur
$config["Serveur"]

# Modification
$config.Port = 8443
$config["NouveauParam"] = "valeur"

# Itération
foreach ($cle in $config.Keys) {
    Write-Host "$cle = $($config[$cle])"
}

# Hashtables imbriquées
$configurationComplexe = @{
    BaseDeDonnees = @{
        Serveur = "db.example.com"
        Port = 5432
        Nom = "ma_base"
    }
    Cache = @{
        Type = "Redis"
        Endpoints = @("redis1:6379", "redis2:6379")
    }
}

# Accès aux valeurs imbriquées
$configurationComplexe.BaseDeDonnees.Serveur
$configurationComplexe.Cache.Endpoints[0]
```

## Section 3 : Opérateurs et expressions

### 3.1 Opérateurs arithmétiques

```powershell
# Arithmétique de base
$addition = 5 + 3          # 8
$soustraction = 10 - 4     # 6
$multiplication = 6 * 7    # 42
$division = 15 / 3         # 5
$modulo = 17 % 5           # 2

# Puissance
$puissance = [Math]::Pow(2, 8)  # 256

# Incrémentation/décrémentation
$compteur = 0
$compteur++    # Post-incrémentation
++$compteur    # Pré-incrémentation
$compteur--    # Post-décrémentation

# Opérateurs d'assignation composée
$valeur = 10
$valeur += 5    # $valeur = $valeur + 5
$valeur -= 3    # $valeur = $valeur - 3
$valeur *= 2    # $valeur = $valeur * 2
$valeur /= 4    # $valeur = $valeur / 4
$valeur %= 7    # $valeur = $valeur % 7
```

### 3.2 Opérateurs de comparaison

```powershell
# Égalité
5 -eq 5        # $true (égal)
"hello" -eq "HELLO"  # $false (case-sensitive par défaut)
"hello" -ieq "HELLO" # $true (case-insensitive)

# Inégalité
10 -ne 5       # $true (différent)
10 -ne 10      # $false

# Comparaisons numériques
5 -lt 10       # $true (inférieur)
5 -le 5        # $true (inférieur ou égal)
10 -gt 5       # $true (supérieur)
10 -ge 10      # $true (supérieur ou égal)

# Comparaisons de chaînes
"apple" -lt "banana"   # $true (ordre alphabétique)
"Apple" -ilt "banana"  # $true (case-insensitive)

# Opérateurs de correspondance
"PowerShell" -like "*Shell*"    # $true
"PowerShell" -notlike "*Bash*"  # $true
"test.txt" -match "\.txt$"      # $true (regex)
"test.txt" -notmatch "\.exe$"   # $true

# Tests de contenance
@(1,2,3) -contains 2           # $true
@(1,2,3) -notcontains 4        # $true
"PowerShell" -in @("Bash", "PowerShell", "Zsh")  # $true

# Tests de type
$variable -is [string]          # Test de type
$variable -isnot [int]          # Test de non-type
```

### 3.3 Opérateurs logiques

```powershell
# ET logique
($age -ge 18) -and ($pays -eq "FR")   # $true si majeur ET français

# OU logique
($role -eq "admin") -or ($role -eq "superuser")  # $true si admin OU superuser

# NON logique
-not ($estConnecte)   # $true si non connecté
!$estConnecte         # Forme abrégée

# Opérateur ternaire (PowerShell 7+)
$resultat = ($condition) ? "vrai" : "faux"

# Court-circuit
# -and s'arrête dès qu'une condition est fausse
# -or s'arrête dès qu'une condition est vraie
```

### 3.4 Opérateurs de redirection et de pipeline

```powershell
# Redirections de base
Write-Host "Message normal"
Write-Host "Message d'erreur" 2>&1                    # Erreur vers sortie standard
Get-Process > processus.txt                         # Sortie vers fichier
Get-Process >> processus.txt                        # Ajout à fichier
Get-Process 2> erreurs.txt                          # Erreurs vers fichier

# Redirections avancées
Get-Process 1> stdout.txt 2> stderr.txt             # Séparation stdout/stderr
Get-Process > $null                                 # Suppression de la sortie
Get-Process 2> $null                                # Suppression des erreurs

# Pipeline avec objets
Get-Process | Where-Object { $_.CPU -gt 10 } | Select-Object Name, CPU

# Pipeline avec foreach
1..10 | ForEach-Object { $_ * 2 }

# Pipeline avec groupage
Get-Service | Group-Object Status

# Opérateurs de pipeline avancés (PowerShell 7+)
Get-Process | Where-Object CPU -gt 10 | Select-Object Name, CPU  # Syntaxe simplifiée
```

## Section 4 : Structures de contrôle

### 4.1 Conditions (if/elseif/else)

```powershell
# Structure if simple
if ($age -ge 18) {
    Write-Host "Vous êtes majeur"
}

# Structure if-else
if ($temperature -gt 30) {
    Write-Host "Il fait chaud"
} else {
    Write-Host "Il fait frais"
}

# Structure if-elseif-else
if ($note -ge 90) {
    $mention = "Excellent"
} elseif ($note -ge 80) {
    $mention = "Très bien"
} elseif ($note -ge 70) {
    $mention = "Bien"
} elseif ($note -ge 60) {
    $mention = "Passable"
} else {
    $mention = "Insuffisant"
}

# Conditions complexes
if (($age -ge 18 -and $pays -eq "FR") -or $estCitoyenUE) {
    Write-Host "Éligible au vote européen"
}
```

### 4.2 Boucles

```powershell
# Boucle for
for ($i = 1; $i -le 10; $i++) {
    Write-Host "Itération $i"
}

# Boucle foreach
$fruits = @("pomme", "banane", "orange")
foreach ($fruit in $fruits) {
    Write-Host "J'aime les $fruit"
}

# Boucle foreach avec pipeline
Get-ChildItem | ForEach-Object {
    Write-Host "Fichier: $($_.Name), Taille: $($_.Length) octets"
}

# Boucle while
$compteur = 1
while ($compteur -le 5) {
    Write-Host "Compteur: $compteur"
    $compteur++
}

# Boucle do-while
$reponse = ""
do {
    $reponse = Read-Host "Entrez une valeur (ou 'quit' pour sortir)"
    Write-Host "Vous avez entré: $reponse"
} while ($reponse -ne "quit")

# Boucle do-until
$tentatives = 0
do {
    $tentatives++
    $resultat = Test-Connection -ComputerName "google.com" -Count 1 -Quiet
} until ($resultat -or $tentatives -ge 3)
```

### 4.3 Instructions de contrôle de flux

```powershell
# break : sortie de boucle
for ($i = 1; $i -le 10; $i++) {
    if ($i -eq 5) {
        Write-Host "Arrêt à 5"
        break
    }
    Write-Host "Nombre: $i"
}

# continue : passage à l'itération suivante
for ($i = 1; $i -le 10; $i++) {
    if ($i % 2 -eq 0) {
        continue  # Saute les nombres pairs
    }
    Write-Host "Nombre impair: $i"
}

# return : sortie de fonction
function Test-Valeur {
    param($valeur)
    
    if ($valeur -lt 0) {
        return "Valeur négative"
    }
    
    if ($valeur -eq 0) {
        return "Valeur nulle"
    }
    
    return "Valeur positive"
}

# throw : lever une exception
function Diviser {
    param([double]$dividende, [double]$diviseur)
    
    if ($diviseur -eq 0) {
        throw "Division par zéro impossible"
    }
    
    return $dividende / $diviseur
}

# Gestion d'erreurs avec try/catch
try {
    $resultat = Diviser 10 0
} catch {
    Write-Host "Erreur capturée: $($_.Exception.Message)"
}
```

## Section 5 : Fonctions et portée

### 5.1 Définition de fonctions

```powershell
# Fonction simple
function Get-Salutation {
    param($nom)
    return "Bonjour $nom"
}

# Fonction avec paramètres typés
function Set-Utilisateur {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Nom,
        
        [Parameter(Mandatory=$false)]
        [int]$Age = 18,
        
        [ValidateSet("Admin", "User", "Guest")]
        [string]$Role = "User"
    )
    
    Write-Host "Création utilisateur: $Nom, Age: $Age, Role: $Role"
}

# Fonction avancée avec pipeline
function Select-GrandsFichiers {
    param(
        [Parameter(ValueFromPipeline=$true)]
        [System.IO.FileInfo]$Fichier,
        
        [int]$TailleMin = 1MB
    )
    
    process {
        if ($Fichier.Length -ge $TailleMin) {
            Write-Output $Fichier
        }
    }
}

# Utilisation de la fonction pipeline
Get-ChildItem C:\Windows -Recurse -File | Select-GrandsFichiers -TailleMin 100MB
```

### 5.2 Portée des variables dans les fonctions

```powershell
# Variable globale
$globale = "visible partout"

function Test-Portee {
    # Variable locale (par défaut)
    $locale = "visible dans la fonction"
    
    # Accès à la variable globale
    Write-Host "Globale: $globale"
    Write-Host "Locale: $locale"
    
    # Modification de la globale
    $global:globale = "modifiée dans la fonction"
    
    # Variable script (visible dans le script appelant)
    $script:scriptVar = "variable de script"
}

Test-Portee
Write-Host "Après fonction - Globale: $globale"
Write-Host "Après fonction - Script: $script:scriptVar"
```

## Section 6 : Gestion d'erreurs et debugging

### 6.1 Gestion d'erreurs de base

```powershell
# Vérification du succès
$succes = Test-Path "C:\fichier_inexistant.txt"
if (-not $succes) {
    Write-Host "Le fichier n'existe pas"
}

# Gestion d'erreurs avec $?
Get-ChildItem "C:\dossier_inexistant" 2>$null
if (-not $?) {
    Write-Host "Erreur lors de l'accès au dossier"
}

# Try/Catch/Finally
try {
    # Code pouvant échouer
    $contenu = Get-Content "fichier_inexistant.txt"
}
catch [System.IO.FileNotFoundException] {
    Write-Host "Fichier non trouvé: $($_.Exception.Message)"
}
catch {
    Write-Host "Erreur inattendue: $($_.Exception.Message)"
}
finally {
    Write-Host "Nettoyage effectué"
}
```

### 6.2 Debugging de base

```powershell
# Mode debug
Set-PSDebug -Trace 1  # Trace l'exécution
Set-PSDebug -Trace 2  # Trace détaillée avec variables
Set-PSDebug -Off     # Désactiver le debug

# Points d'arrêt (breakpoints)
Set-PSBreakpoint -Script "monscript.ps1" -Line 15
Set-PSBreakpoint -Command "Get-Process"

# Inspection des variables
Get-Variable
Get-Variable | Where-Object Name -like "*process*"

# Historique des commandes
Get-History
Invoke-History 5  # Répète la commande numéro 5
```

## Conclusion : Les fondations d'un langage cohérent

La syntaxe de PowerShell, bien que différente de celle des shells traditionnels, offre une cohérence et une puissance remarquables. En traitant tout comme des objets et en appliquant des conventions strictes, PowerShell permet d'écrire du code maintenable et robuste.

Dans le prochain chapitre, nous explorerons les cmdlets intégrés et le pipeline objet, qui forment le cœur de la puissance de PowerShell.

---

**Exercice pratique :** Écrivez un script PowerShell qui :
1. Liste tous les processus dont la consommation CPU dépasse 5%
2. Trie ces processus par consommation mémoire décroissante
3. Affiche les 5 premiers avec nom, PID, CPU et mémoire
4. Gère les erreurs si aucun processus ne correspond

**Challenge :** Comparez la verbosité et la lisibilité de ce script avec l'équivalent en batch/cmd traditionnel.

