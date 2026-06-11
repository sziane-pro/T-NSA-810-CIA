# ADR-011 - Préparation de l’architecture à l’ajout d’un troisième site

## Statut
Proposé

## Date
11/06/2026

## Contexte

L’objectif global du projet est de construire une architecture sécurisée, automatisée, reproductible et évolutive, capable d’accueillir de nouveaux sites plus tard.

Même si le périmètre initial concerne deux sites, l’architecture ne doit pas être conçue comme une solution figée.

Elle doit pouvoir intégrer un troisième site sans remettre en cause toute la structure réseau, les règles firewall, l’IPAM ou la supervision.

## Décision

Nous avons décidé de préparer l’architecture à l’ajout futur d’un troisième site.

Cette préparation repose sur :

- un plan d’adressage extensible ;
- une nomenclature claire des sites ;
- des règles firewall réutilisables ;
- une documentation standardisée ;
- une structure Git adaptée ;
- une déclaration des réseaux dans NetBox ;
- une supervision capable d’intégrer de nouveaux composants ;
- des templates de configuration lorsque c’est possible.

Le troisième site ne sera pas forcément déployé dans le périmètre initial, mais l’architecture devra permettre son intégration future.

## Options envisagées

### Option 1 : Architecture limitée aux deux sites actuels

#### Avantages

- Plus simple.
- Plus rapide.
- Moins de réflexion initiale.

#### Inconvénients

- Peu évolutif.
- Risque de devoir refaire l’adressage.
- Documentation moins réutilisable.
- Moins aligné avec l’objectif du projet.

### Option 2 : Préparer l’ajout d’un troisième site sans le déployer

#### Avantages

- Évolutif.
- Réaliste.
- Compatible avec les contraintes de temps.
- Montre une réflexion d’architecture.
- Facilite les évolutions futures.

#### Inconvénients

- Demande plus de conception.
- Besoin d’anticiper les plages IP.
- Demande une documentation plus structurée.

### Option 3 : Déployer directement trois sites

#### Avantages

- Démonstration complète de scalabilité.
- Architecture multi-sites réelle.

#### Inconvénients

- Plus complexe.
- Risque de dépasser les contraintes du projet.
- Plus difficile à sécuriser correctement.
- Charge de travail plus importante.

## Justification

La préparation à un troisième site est retenue, sans déploiement immédiat.

Ce choix permet de répondre à l’objectif d’évolutivité tout en gardant un périmètre réaliste.

Il montre que l’architecture est pensée pour grandir, sans complexifier inutilement la première version.

## Conséquences positives

- Architecture plus évolutive.
- Plan d’adressage mieux structuré.
- Documentation plus professionnelle.
- Réutilisation possible des configurations.
- Ajout futur d’un site facilité.
- Meilleure démonstration de la réflexion architecturale.

## Conséquences négatives

- Demande une conception plus rigoureuse.
- Nécessite des conventions stables.
- Peut être plus long à documenter.
- Certaines hypothèses devront être validées lors d’un vrai ajout de site.

## Impact sur le projet

La documentation devra inclure :

- une plage IP réservée pour un futur site ;
- une convention de nommage ;
- un modèle de règles firewall ;
- un modèle de configuration VPN ;
- une structure NetBox compatible multi-sites ;
- une procédure théorique d’ajout d’un nouveau site.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- un espace d’adressage est réservé pour un futur site ;
- les conventions de nommage sont compatibles multi-sites ;
- NetBox peut intégrer un nouveau site ;
- les règles firewall peuvent être reproduites ;
- une procédure d’ajout d’un site est documentée.