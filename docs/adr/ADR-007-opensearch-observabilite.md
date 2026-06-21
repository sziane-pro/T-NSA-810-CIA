# ADR-007 - Choix d’OpenSearch pour la centralisation des logs et l’observabilité

## Statut
Accepté

## Date
11/06/2026

## Contexte

Le projet demande la mise en place d’une solution d’observabilité permettant de centraliser, indexer et analyser les logs de l’infrastructure.

Le besoin initial mentionne Elasticsearch comme technologie d’observabilité. Cependant, dans le cadre de la mise en œuvre du projet, nous avons utilisé OpenSearch avec OpenSearch Dashboards.

L’objectif reste identique : centraliser les logs des composants critiques afin de faciliter le diagnostic, l’analyse réseau et la supervision de sécurité.

Les composants concernés sont notamment :

- pfSense / OPNSense selon le firewall réellement utilisé ;
- VPN ;
- bastion ;
- machines virtuelles ;
- services internes ;
- DNS si disponible.

## Décision

Nous avons décidé d’utiliser OpenSearch pour la centralisation, l’indexation et la recherche dans les logs.

OpenSearch Dashboards est utilisé pour visualiser les données, créer des recherches, consulter les événements et préparer des tableaux de bord.

Les logs prioritaires à collecter sont :

- logs firewall ;
- logs VPN ;
- logs du bastion ;
- logs système ;
- logs des services internes ;
- événements liés aux accès d’administration.

## Options envisagées

### Option 1 : Logs locaux uniquement

#### Avantages

- Simple à mettre en place.
- Pas de service central supplémentaire.

#### Inconvénients

- Logs dispersés.
- Analyse difficile.
- Risque de perte d’information en cas d’incident.
- Peu adapté à une infrastructure multi-sites.

### Option 2 : Elasticsearch avec Kibana

#### Avantages

- Solution connue pour la centralisation et l’analyse de logs.
- Large écosystème.
- Bonne capacité d’indexation et de recherche.
- Répond directement à la technologie mentionnée dans le sujet.

#### Inconvénients

- Certaines fonctionnalités peuvent dépendre des licences et versions.
- Mise en place potentiellement plus lourde selon l’environnement.
- Nécessite une attention particulière à la consommation de ressources.

### Option 3 : OpenSearch avec OpenSearch Dashboards

#### Avantages

- Solution open source.
- Adaptée à la centralisation et à l’analyse de logs.
- OpenSearch Dashboards permet la visualisation des données.
- Bon choix pour un environnement pédagogique et reproductible.
- Permet de répondre au besoin fonctionnel d’observabilité.

#### Inconvénients

- Diffère de la technologie explicitement mentionnée dans le sujet.
- Doit être justifié clairement dans la documentation.
- Certaines documentations ou intégrations peuvent différer d’Elasticsearch/Kibana.

## Justification

OpenSearch a été retenu car il permet de répondre au besoin fonctionnel d’observabilité : centraliser, indexer, rechercher et analyser les logs de l’infrastructure.

Même si le sujet mentionne Elasticsearch, OpenSearch répond au même objectif d’usage dans le cadre du projet : fournir une plateforme de recherche et d’analyse des événements.

OpenSearch Dashboards permet également de visualiser les logs et de construire des vues exploitables pour le diagnostic et la supervision.

### Raison déterminante : la licence (contrainte du sujet)

Le sujet impose des stacks **« actively supported, maintained and updated by the community »**. Or, en 2021, Elastic a changé la licence d’Elasticsearch et Kibana (passage d’Apache 2.0 à **SSPL/Elastic License**, non-OSI, source-available). **OpenSearch** est le **fork communautaire sous licence Apache 2.0** né de ce changement, soutenu par une fondation et une large communauté.

OpenSearch est donc retenu car il est :

- **réellement open source** (Apache 2.0), sans restriction d’usage liée à la licence ;
- **maintenu activement par la communauté** → coche directement la contrainte du sujet ;
- **fonctionnellement équivalent** à Elasticsearch/Kibana pour notre besoin (collecte, indexation, recherche, dashboards, alerting), car issu du même code base.

Le besoin du sujet (« centraliser et analyser les logs via Elasticsearch ») est donc **satisfait sur le fond** : OpenSearch est le successeur open source direct d’Elasticsearch, plus conforme à la contrainte de licence communautaire que l’Elasticsearch actuel.

Ce choix est documenté ici car il s’agit d’un écart de **nom** par rapport à la stack mentionnée — mais pas de **finalité**.

## Conséquences positives

- Centralisation des logs.
- Recherche rapide dans les événements.
- Visualisation possible via OpenSearch Dashboards.
- Meilleure capacité de diagnostic.
- Solution open source adaptée au projet.
- Amélioration de la traçabilité des accès et flux réseau.

## Conséquences négatives

- Écart avec la technologie Elasticsearch indiquée dans le sujet.
- Nécessite une justification dans la documentation.
- Nécessite de documenter précisément la stack réellement utilisée.
- Les captures et procédures doivent utiliser les bons noms : OpenSearch et OpenSearch Dashboards.

## Impact sur le projet

La documentation devra préciser :

- pourquoi OpenSearch a été utilisé ;
- où OpenSearch est hébergé ;
- où OpenSearch Dashboards est hébergé ;
- quels composants envoient leurs logs ;
- quels flux réseau sont nécessaires ;
- qui peut accéder aux dashboards ;
- quelles recherches ou visualisations sont disponibles ;
- comment sauvegarder ou restaurer la configuration.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- OpenSearch reçoit les logs des composants critiques ;
- OpenSearch Dashboards permet de consulter les événements ;
- les accès aux dashboards sont limités ;
- les flux de logs sont documentés ;
- la différence avec Elasticsearch est justifiée ;
- le diagramme d’architecture reflète la stack réellement utilisée.