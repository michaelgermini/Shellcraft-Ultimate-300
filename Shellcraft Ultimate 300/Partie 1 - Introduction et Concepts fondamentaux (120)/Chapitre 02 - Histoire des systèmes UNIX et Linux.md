# Chapitre 02 - Histoire des systèmes UNIX et Linux

## Table des matières
- [Introduction](#introduction)
- [Les origines : MULTICS et UNIX](#les-origines--multics-et-unix)
- [La révolution BSD](#la-révolution-bsd)
- [La naissance de Linux](#la-naissance-de-linux)
- [L'écosystème GNU et la Free Software Foundation](#lécosystème-gnu-et-la-free-software-foundation)
- [L'explosion des distributions Linux](#lexplosion-des-distributions-linux)
- [L'impact sur l'informatique moderne](#limpact-sur-linformatique-moderne)
- [Le terminal à travers les époques](#le-terminal-à-travers-les-époques)
- [Conclusion](#conclusion)

## Introduction

Comprendre l'histoire des systèmes UNIX et Linux, c'est comprendre les racines philosophiques et techniques du terminal moderne. Cette histoire n'est pas seulement une chronologie d'événements, mais une évolution des idées qui ont façonné l'informatique telle que nous la connaissons aujourd'hui.

Imaginez l'informatique comme une immense bibliothèque. Les systèmes graphiques seraient les magazines populaires en couverture, tandis que UNIX et Linux représenteraient les ouvrages de référence fondamentaux, consultés par tous les auteurs ultérieurs. Sans cette base historique, le terminal resterait un mystère.

## Les origines : MULTICS et UNIX

### Le projet MULTICS (1964-1969)

Au milieu des années 1960, l'informatique était dominée par des mainframes coûteux et des systèmes propriétaires. Bell Labs, MIT et General Electric lancent le projet MULTICS (MULTiplexed Information and Computing Service), une vision ambitieuse d'un système d'exploitation temps partagé capable de servir des centaines d'utilisateurs simultanément.

MULTICS introduit des concepts révolutionnaires :
- La séparation des espaces mémoire entre processus
- Un système de fichiers hiérarchique
- La notion de "tout est fichier"
- Les pipes pour la communication inter-processus

Cependant, MULTICS devient trop complexe et coûteux. En 1969, Bell Labs se retire du projet, laissant une équipe réduite continuer le développement.

### La naissance d'UNIX (1969-1970)

Ken Thompson, Dennis Ritchie et leurs collègues chez Bell Labs décident de créer un système plus simple. En quelques semaines, Thompson écrit la première version d'UNIX sur un PDP-7 inutilisé.

Le nom "UNIX" est un jeu de mots sur MULTICS : "UNI" signifiant "un" en latin, par opposition à "MULTI".

Les principes fondateurs d'UNIX :
- **Simplicité** : Faire une chose et la faire bien
- **Composition** : Combiner de petits outils via des pipes
- **Texte comme interface universelle** : Tout programme peut lire et écrire du texte
- **Portabilité** : Écrit en C, indépendant du matériel

## La révolution BSD

### Berkeley Software Distribution (1977-1990s)

L'Université de Berkeley obtient une licence d'UNIX et commence à le modifier. Les versions BSD introduisent des innovations majeures :

- Le TCP/IP stack (1980s) - base d'Internet
- Le système de fichiers Fast File System (FFS)
- Les sockets BSD pour la programmation réseau
- Des utilitaires avancés comme vi, csh, et sendmail

BSD devient la référence pour les universités et la recherche, formant des générations de programmeurs.

### Le litige AT&T vs BSD (1990s)

AT&T découvre le potentiel commercial d'UNIX et commence à faire valoir ses droits. Un long procès s'ensuit, forçant la communauté BSD à réécrire complètement le code propriétaire d'AT&T. Cette "NetBSD cleanup" donne naissance aux projets modernes BSD : NetBSD, FreeBSD, OpenBSD.

## La naissance de Linux

### Richard Stallman et le mouvement du logiciel libre

En 1983, Richard Stallman lance le projet GNU (GNU's Not Unix) avec pour objectif de créer un système d'exploitation entièrement libre. Il fonde la Free Software Foundation (FSF) et définit les quatre libertés du logiciel libre :

1. Liberté d'exécuter le programme
2. Liberté d'étudier et modifier le code
3. Liberté de redistribuer des copies
4. Liberté de distribuer des versions modifiées

### Linus Torvalds et le kernel Linux (1991)

Linus Torvalds, un étudiant finlandais de 21 ans, annonce sur Usenet la création d'un nouveau système d'exploitation. Initialement conçu comme un hobby, Linux évolue rapidement grâce à la communauté open source.

Le 25 août 1991, Torvalds poste : "Je fais un système d'exploitation (libre) (juste un hobby, ne sera pas grand et professionnel comme gnu)"

Linux apporte des innovations clés :
- Architecture modulaire et monolithique hybride
- Support SMP (Symmetric Multi-Processing)
- Gestion mémoire avancée
- Drivers pour une multitude de périphériques

### La combinaison GNU/Linux

Linux seul n'est qu'un kernel. La combinaison avec les outils GNU (GCC, Bash, coreutils) crée le premier système GNU/Linux complet. Cette collaboration entre le projet GNU et Linux donne naissance aux distributions modernes.

## L'écosystème GNU et la Free Software Foundation

Le projet GNU fournit l'environnement utilisateur complet :
- Bash : le shell
- GCC : le compilateur C
- Emacs : l'éditeur
- GDB : le débogueur
- Make : l'outil de build
- Et des centaines d'autres outils

La philosophie GNU influence profondément la culture du logiciel libre et l'approche "UNIX way" du développement.

## L'explosion des distributions Linux

### Slackware (1993) - La première distribution

Patrick Volkerding crée Slackware, la première distribution Linux complète, inspirée de la simplicité BSD.

### Red Hat (1995) - L'entreprise du Linux

Red Hat popularise Linux dans les entreprises avec des outils de gestion et du support commercial.

### Debian (1993) - La démocratie du logiciel

Ian Murdock fonde Debian sur les principes de la démocratie et de la qualité. Debian devient la base de nombreuses distributions (Ubuntu, Linux Mint, etc.).

### Ubuntu (2004) - Linux pour le grand public

Canonical lance Ubuntu, rendant Linux accessible aux utilisateurs non-techniques avec une installation simplifiée et des mises à jour régulières.

Aujourd'hui, il existe plus de 600 distributions Linux, chacune adaptée à des cas d'utilisation spécifiques.

## L'impact sur l'informatique moderne

### L'infrastructure mondiale

- Plus de 90% des serveurs web tournent sur Linux
- Les supercalculateurs les plus puissants utilisent Linux
- Android (basé sur Linux) équipe 85% des smartphones
- L'Internet des objets repose largement sur Linux embarqué

### La culture du développement

UNIX/Linux influence tous les systèmes modernes :
- macOS est basé sur un kernel BSD dérivé
- Windows inclut WSL (Windows Subsystem for Linux)
- Les conteneurs Docker utilisent des primitives Linux
- Git, le système de contrôle de version dominant, est né sur Linux

### La philosophie technique

Les principes UNIX continuent d'influencer :
- L'architecture microservices
- Les API REST
- Le développement d'outils composables
- La culture DevOps

## Le terminal à travers les époques

### Les premiers terminaux (1960s-1970s)

Les terminaux étaient des machines dédiées (Teletype, DEC VT100) connectées aux mainframes. L'interface était exclusivement textuelle, avec des commandes limitées.

### L'ère graphique (1980s-1990s)

L'avènement des interfaces graphiques (X Window System, Windows) relègue le terminal au second plan. Cependant, les administrateurs système et les développeurs continuent de l'utiliser.

### La renaissance (2000s-2020s)

Le cloud computing, DevOps et les conteneurs ramènent le terminal au premier plan. Les outils modernes comme tmux, zsh, et les terminaux graphiques avancés enrichissent l'expérience.

Aujourd'hui, le terminal n'est plus une limitation mais un choix conscient pour la puissance et l'efficacité.

## Conclusion

L'histoire d'UNIX et Linux n'est pas seulement celle de systèmes d'exploitation ; c'est l'histoire d'une philosophie technique qui privilégie la simplicité, la composition et la liberté.

Comprendre cette histoire, c'est comprendre pourquoi le terminal fonctionne comme il le fait. Les choix architecturaux des années 1970 continuent d'influencer notre façon de développer, déployer et maintenir des systèmes informatiques.

Dans notre ère de l'abstraction et des interfaces graphiques sophistiquées, le terminal reste le rappel que la véritable puissance vient souvent de la simplicité et de la compréhension profonde des mécanismes sous-jacents.

Les développeurs qui maîtrisent ces outils historiques ne sont pas seulement techniciens ; ils sont les gardiens d'une tradition d'excellence technique qui a façonné l'informatique moderne.
