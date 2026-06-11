# ADR-006 - Choix de NetBox comme source de vérité IPAM

## Statut
Accepté

## Date
11/06/2026

## Contexte

L’infrastructure comporte plusieurs sites, plusieurs sous-réseaux, des firewalls, des services internes, un VPN, du DNS forwarding et une possible extension future vers d’autres sites.

Une gestion manuelle des adresses IP dans des fichiers dispersés pourrait créer des incohérences, des doublons ou des erreurs de documentation.

Le projet impose l’utilisation de NetBox pour la gestion IPAM.

## Décision

Nous avons décidé d’utiliser NetBox comme source de vérité pour la gestion :

- des adresses IP ;
- des sous-réseaux ;
- des sites ;
- des services réseau ;
- des équipements logiques ;
- des zones réseau.

NetBox permettra de documenter :

- les sites ;
- les préfixes IP ;
- les adresses IP utilisées ;
- les VLAN ou zones réseau si utilisés ;
- les équipements et services ;
- les liens logiques entre composants.

## Options envisagées

### Option 1 : Documentation IP dans un tableur

#### Avantages

- Simple à créer.
- Rapide pour un petit projet.
- Facile à lire.

#### Inconvénients

- Risque d’erreurs manuelles.
- Pas adapté à l’automatisation.
- Peu scalable.
- Pas de validation centralisée.

### Option 2 : Documentation IP dans des fichiers Markdown

#### Avantages

- Versionnable dans Git.
- Simple à maintenir.
- Adapté à une documentation textuelle.

#### Inconvénients

- Pas de gestion IPAM native.
- Risque de doublons.
- Pas de vue structurée des objets réseau.

### Option 3 : NetBox comme IPAM central

#### Avantages

- Source de vérité dédiée.
- Meilleure structuration des données réseau.
- Adapté à une infrastructure multi-sites.
- Peut être exploité dans une logique d’automatisation.
- Améliore la documentation et la reproductibilité.

#### Inconvénients

- Nécessite un service supplémentaire.
- Besoin de maintenir les données à jour.
- Demande une phase de configuration initiale.

## Justification

NetBox est retenu car il répond au besoin de documentation structurée et de gestion centralisée de l’adressage IP.

Il permet de limiter les erreurs dans une architecture multi-sites et prépare l’ajout futur de nouveaux sites.

Il peut aussi servir de référence pour l’automatisation, les configurations réseau et les procédures de reconstruction.

## Conséquences positives

- Plan d’adressage clair.
- Réduction du risque de conflit IP.
- Meilleure documentation.
- Source de vérité pour les réseaux et services.
- Préparation à l’automatisation.
- Architecture plus évolutive.

## Conséquences négatives

- Nécessite une mise à jour régulière.
- Peut devenir obsolète si les changements ne sont pas documentés.
- Ajoute un service critique à sauvegarder.
- Nécessite de contrôler les accès.

## Impact sur le projet

NetBox devra être intégré à la documentation technique.

Il faudra documenter :

- où NetBox est hébergé ;
- qui peut y accéder ;
- quelles données y sont stockées ;
- les sous-réseaux de chaque site ;
- les IP des composants critiques ;
- la procédure de sauvegarde et restauration de NetBox.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- les sites sont déclarés dans NetBox ;
- les plages IP sont documentées ;
- les IP critiques sont renseignées ;
- les informations correspondent au schéma réseau ;
- NetBox est utilisé comme référence dans la documentation.