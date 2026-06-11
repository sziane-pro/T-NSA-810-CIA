# ADR-008 - Choix d’une approche GitOps pour versionner les configurations

## Statut
Accepté

## Date
11/06/2026

## Contexte

Le projet demande une infrastructure documentée, automatisée, reproductible et capable d’être reconstruite.

Il ne suffit donc pas de configurer les composants manuellement. Il faut aussi pouvoir expliquer comment reconstruire l’infrastructure.

Les configurations réseau, les règles firewall, les scripts, les fichiers d’automatisation, les procédures et les éléments de documentation doivent être versionnés.

## Décision

Nous avons décidé d’adopter une approche GitOps pour centraliser et versionner les éléments techniques du projet.

Les dépôts Git contiendront notamment :

- documentation d’architecture ;
- diagrammes ;
- ADR ;
- configurations réseau exportées ;
- scripts d’automatisation ;
- templates de VM si disponibles ;
- procédures de déploiement ;
- procédures de restauration ;
- backlog technique ;
- suivi des décisions d’architecture.

Les secrets ne seront pas stockés en clair dans Git.

## Options envisagées

### Option 1 : Documentation locale non versionnée

#### Avantages

- Simple au départ.
- Pas de structure à mettre en place.

#### Inconvénients

- Risque de perte.
- Pas d’historique.
- Difficile à relire ou auditer.
- Mauvaise reproductibilité.

### Option 2 : Git pour la documentation uniquement

#### Avantages

- Historique des changements.
- Documentation centralisée.
- Simple à mettre en place.

#### Inconvénients

- Les configurations techniques restent dispersées.
- Reproductibilité partielle.
- Moins aligné avec l’objectif GitOps.

### Option 3 : GitOps pour documentation, scripts et configurations

#### Avantages

- Historique clair.
- Meilleure reproductibilité.
- Meilleure collaboration.
- Facilite les revues.
- Permet de reconstruire l’infrastructure plus facilement.
- Prépare une future CI/CD sur l’IaC.

#### Inconvénients

- Demande une organisation stricte.
- Risque de fuite si les secrets sont mal gérés.
- Nécessite des conventions de nommage.

## Justification

L’approche GitOps est retenue car elle répond directement à l’objectif de reproductibilité et de documentation.

Elle permet de montrer l’évolution du projet, de documenter les choix techniques et de conserver un historique des configurations.

Elle facilite aussi la préparation d’un runbook et d’un plan de reprise d’activité.

## Conséquences positives

- Meilleure traçabilité.
- Documentation versionnée.
- Reproductibilité améliorée.
- Meilleure qualité du code et des configurations.
- Préparation à une future CI/CD.
- Meilleure organisation du projet.

## Conséquences négatives

- Nécessité de maintenir le dépôt propre.
- Besoin de conventions claires.
- Risque de stocker des informations sensibles si les règles ne sont pas respectées.
- Demande une discipline de commit régulière.

## Impact sur le projet

Le projet devra inclure :

- une structure de dépôt claire ;
- un fichier README ;
- des dossiers pour l’architecture, les ADR, les scripts, les configurations et les procédures ;
- une règle stricte d’exclusion des secrets ;
- un suivi des changements importants par commits.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- les documents techniques sont versionnés ;
- les ADR sont présents dans le dépôt ;
- les configurations exportables sont stockées sans secret ;
- le dépôt contient une structure claire ;
- le projet peut être compris et reconstruit à partir de la documentation.