# ADR-010 - Choix d’une stratégie de gestion sécurisée des secrets

## Statut
Accepté

## Date
11/06/2026

## Contexte

L’infrastructure utilise plusieurs éléments sensibles :

- identifiants administrateurs ;
- clés VPN ;
- certificats ;
- mots de passe pfSense ;
- accès Proxmox ;
- accès NetBox ;
- accès Elasticsearch ;
- secrets liés aux scripts ou à l’automatisation.

Ces informations ne doivent pas être stockées en clair dans les dépôts Git, dans la documentation publique du projet ou dans des fichiers non protégés.

Le projet demande explicitement un stockage sécurisé des secrets, avec une solution de type vault ou chiffrement.

## Décision

Nous avons décidé de séparer strictement les secrets du code et de la documentation.

Les secrets seront stockés dans un espace sécurisé dédié ou dans des fichiers chiffrés.

Les dépôts Git ne devront contenir que :

- des fichiers d’exemple ;
- des templates sans secrets ;
- des variables fictives ;
- des instructions de configuration.

Les fichiers contenant des secrets devront être exclus du dépôt avec un `.gitignore`.

## Options envisagées

### Option 1 : Stocker les secrets dans les fichiers de configuration

#### Avantages

- Simple à utiliser.
- Déploiement rapide.

#### Inconvénients

- Risque majeur de fuite.
- Non conforme aux bonnes pratiques.
- Dangereux avec Git.
- Difficile à auditer.

### Option 2 : Utiliser des fichiers `.env` non versionnés

#### Avantages

- Simple à mettre en place.
- Séparation minimale entre code et secrets.
- Compatible avec des scripts.

#### Inconvénients

- Protection limitée.
- Risque de mauvaise manipulation.
- Pas idéal pour des secrets critiques.

### Option 3 : Utiliser une solution de coffre-fort ou de chiffrement

#### Avantages

- Meilleure sécurité.
- Secrets séparés du code.
- Compatible avec une démarche professionnelle.
- Réduction du risque de fuite.
- Meilleure préparation à l’automatisation.

#### Inconvénients

- Mise en place plus complexe.
- Nécessite une procédure d’accès.
- Besoin de documenter la restauration des secrets.

## Justification

La séparation des secrets et du code est retenue car elle est indispensable pour sécuriser l’infrastructure.

Le projet valorise la qualité de la documentation, la sécurité et la reproductibilité.

Une bonne gestion des secrets permet de versionner les configurations sans exposer d’informations sensibles.

## Conséquences positives

- Réduction du risque de fuite de secrets.
- Dépôt Git plus sûr.
- Meilleure conformité aux bonnes pratiques.
- Documentation plus professionnelle.
- Meilleure maîtrise des accès sensibles.

## Conséquences négatives

- Nécessite une procédure claire.
- Peut complexifier le déploiement initial.
- Besoin de sauvegarder les secrets de manière sécurisée.
- Risque de perte d’accès si la gestion des clés est mal organisée.

## Impact sur le projet

La documentation devra préciser :

- quels secrets existent ;
- où ils sont stockés ;
- qui peut y accéder ;
- comment les restaurer ;
- quels fichiers sont exclus de Git ;
- quels templates sont fournis pour reconstruire l’infrastructure.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- aucun secret n’est stocké en clair dans Git ;
- les fichiers sensibles sont ignorés ;
- des templates sans secrets sont fournis ;
- une procédure de stockage et restauration des secrets existe ;
- les accès aux secrets sont limités.