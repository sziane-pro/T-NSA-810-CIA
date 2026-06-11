# ADR-002 - Choix d’une segmentation réseau Admin / Services / Utilisateurs / DMZ

## Statut
Accepté

## Date
11/06/2026

## Contexte

Le projet nécessite la mise en place d’une infrastructure hybride composée de deux sites interconnectés par VPN.

Plusieurs types de flux doivent coexister :

- administration ;
- utilisateurs ;
- services internes ;
- supervision ;
- DNS ;
- IPAM ;
- accès distant ;
- VPN inter-sites.

Sans séparation réseau, une compromission d’un service ou d’une machine pourrait faciliter les mouvements latéraux vers des composants critiques comme Proxmox, pfSense, NetBox, Elasticsearch ou le bastion.

Le projet impose aussi de réfléchir au principe du moindre privilège et à la séparation des flux.

## Décision

Nous avons décidé de segmenter les réseaux par usage fonctionnel et par niveau de sensibilité.

La segmentation prévue repose sur plusieurs zones logiques :

- réseau Admin ;
- réseau Services ;
- réseau Utilisateurs / LAN interne ;
- réseau DMZ si nécessaire ;
- réseau VPN inter-sites.

Les flux entre ces zones seront contrôlés par pfSense avec des règles firewall explicites.

Par défaut, les communications inter-zones seront bloquées. Seuls les flux nécessaires seront autorisés et documentés.

## Options envisagées

### Option 1 : Réseau unique par site

#### Avantages

- Simplicité de mise en place.
- Moins de règles firewall à gérer.
- Déploiement initial plus rapide.

#### Inconvénients

- Faible sécurité.
- Pas de séparation entre administration, utilisateurs et services.
- Risque important de mouvement latéral en cas de compromission.
- Moins conforme aux bonnes pratiques attendues.

### Option 2 : Segmentation réseau par usage

#### Avantages

- Meilleure sécurité.
- Application du principe du moindre privilège.
- Contrôle précis des flux.
- Meilleure lisibilité de l’architecture.
- Réduction de l’impact d’une compromission.

#### Inconvénients

- Configuration plus complexe.
- Besoin d’un plan d’adressage précis.
- Nécessite une documentation claire des règles firewall.

### Option 3 : Micro-segmentation très fine

#### Avantages

- Sécurité renforcée.
- Contrôle très précis des communications.
- Réduction forte des mouvements latéraux.

#### Inconvénients

- Complexité élevée.
- Surdimensionné pour le projet.
- Risque d’erreurs de configuration.
- Maintenance plus difficile.

## Justification

La segmentation réseau par usage est retenue car elle représente le meilleur compromis entre sécurité, lisibilité et faisabilité.

Elle permet de respecter les bonnes pratiques de sécurité sans rendre l’infrastructure trop complexe. Elle facilite aussi l’analyse des flux dans le diagramme d’architecture et dans la documentation technique.

Cette approche permet de montrer clairement quels flux sont autorisés entre les zones :

- accès administrateur vers Proxmox ;
- accès du bastion vers les machines administrées ;
- accès des services vers NetBox ;
- envoi des logs vers Elasticsearch ;
- résolution DNS entre les sites ;
- communication inter-sites via VPN.

## Conséquences positives

- Réduction des risques de mouvement latéral.
- Meilleure maîtrise des flux réseau.
- Architecture plus lisible.
- Règles firewall plus faciles à justifier.
- Meilleure conformité avec le principe du moindre privilège.
- Préparation facilitée à l’ajout d’un troisième site.

## Conséquences négatives

- Besoin de maintenir une documentation précise.
- Configuration firewall plus longue à mettre en place.
- Risque de bloquer certains services si les règles sont incomplètes.
- Besoin de tester chaque flux critique.

## Impact sur le projet

Cette décision impose de produire :

- un plan d’adressage par zone ;
- un schéma réseau détaillé ;
- une matrice de flux ;
- des règles firewall documentées ;
- une procédure de test des flux autorisés et interdits.

Elle impacte directement la configuration de pfSense, du VPN, du bastion, de NetBox, d’Elasticsearch et du DNS forwarding.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- les réseaux sont clairement séparés ;
- les flux non nécessaires sont bloqués ;
- les règles firewall sont documentées ;
- la matrice de flux correspond au schéma d’architecture ;
- les tests de connectivité confirment les autorisations et les blocages.