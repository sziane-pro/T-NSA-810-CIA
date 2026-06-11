# ADR-005 - Choix d’un bastion host pour l’administration distante

## Statut
Accepté

## Date
11/06/2026

## Contexte

Le site distant doit pouvoir être administré depuis l’extérieur sans exposer directement les services critiques comme Proxmox, pfSense, NetBox ou les machines internes.

Un accès direct depuis Internet aux interfaces d’administration augmenterait fortement la surface d’attaque.

Il est donc nécessaire de mettre en place un point d’entrée sécurisé, contrôlé et journalisé.

## Décision

Nous avons décidé de mettre en place un bastion host comme point d’entrée unique pour l’administration distante.

Le bastion permettra d’accéder aux ressources internes autorisées du site distant, selon des règles strictes.

Les accès seront limités aux administrateurs autorisés. Les connexions devront être journalisées et les flux depuis le bastion vers les autres machines seront explicitement définis dans pfSense.

## Options envisagées

### Option 1 : Accès direct aux interfaces d’administration

#### Avantages

- Simple à mettre en place.
- Accès rapide aux services.

#### Inconvénients

- Très mauvaise pratique de sécurité.
- Surface d’attaque importante.
- Difficulté à tracer les accès.
- Risque élevé en cas de compromission.

### Option 2 : Accès uniquement via VPN administrateur

#### Avantages

- Accès chiffré.
- Moins exposé qu’un accès direct.
- Bonne sécurité si bien configuré.

#### Inconvénients

- Moins de contrôle centralisé sur les sessions.
- Logs d’administration potentiellement dispersés.
- Moins clair dans le diagramme des flux.

### Option 3 : Bastion host dédié

#### Avantages

- Point d’entrée unique.
- Meilleure traçabilité.
- Réduction de la surface d’exposition.
- Application du moindre privilège.
- Possibilité de journaliser les accès.
- Plus simple à présenter et à auditer.

#### Inconvénients

- Composant supplémentaire à maintenir.
- Devient critique pour l’administration distante.
- Nécessite un durcissement spécifique.

## Justification

Le bastion host est retenu car il permet de contrôler, limiter et journaliser les accès d’administration distante.

Il évite d’exposer directement les interfaces sensibles sur Internet. Il permet aussi d’appliquer le principe du moindre privilège en limitant les flux autorisés depuis le bastion vers les ressources internes.

Ce choix améliore la sécurité globale de l’infrastructure et facilite l’audit des accès.

## Conséquences positives

- Réduction de la surface d’attaque.
- Point d’entrée unique pour l’administration.
- Meilleure traçabilité des accès.
- Contrôle précis des flux.
- Plus grande conformité avec les bonnes pratiques de sécurité.

## Conséquences négatives

- Besoin de sécuriser fortement le bastion.
- Dépendance à sa disponibilité pour l’administration distante.
- Nécessité de surveiller les logs.
- Ajout d’un composant à maintenir dans la limite des 3 VM par site.

## Impact sur le projet

Le diagramme devra montrer :

- l’emplacement du bastion ;
- les flux entrants autorisés ;
- les flux sortants vers les machines internes ;
- les règles firewall associées ;
- les méthodes d’authentification ;
- les logs générés ;
- les procédures d’accès et de restauration.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- l’administration distante passe par le bastion ;
- les accès directs aux interfaces sensibles sont bloqués ;
- les flux depuis le bastion sont limités ;
- les connexions sont journalisées ;
- le bastion est intégré à la documentation et au schéma.