# ADR-001 - Choix d’une architecture hybride à deux sites interconnectés par VPN

## Statut
Accepté

## Date
11/06/2026

## Contexte

Le projet consiste à concevoir, déployer, sécuriser et documenter une infrastructure hybride composée de deux sites :

- un site on-premise ;
- un site distant / cloud.

Ces deux sites doivent être interconnectés de manière sécurisée afin de permettre la communication entre certains services internes, tout en évitant l’exposition directe des ressources sensibles sur Internet.

L’infrastructure doit intégrer :

- deux plateformes Proxmox VE ;
- un firewall par site avec pfSense ;
- un VPN site-to-site chiffré avec OpenVPN ;
- un bastion pour l’administration distante ;
- une gestion centralisée des adresses IP avec NetBox ;
- une centralisation des logs avec Elasticsearch ;
- un DNS forwarding entre les sites ;
- un mécanisme de coupure d’urgence ;
- une documentation permettant de reconstruire l’infrastructure.

Le projet impose aussi plusieurs contraintes :

- maximum 3 VM par site Proxmox ;
- utilisation de technologies maintenues ;
- architecture claire, sécurisée et documentée ;
- capacité à accueillir de nouveaux sites plus tard.

## Décision

Nous avons décidé de construire une architecture hybride composée de deux sites Proxmox VE interconnectés par un VPN site-to-site chiffré avec OpenVPN.

Chaque site disposera d’un firewall pfSense chargé de contrôler les flux réseau, de gérer le routage et de sécuriser les communications entre les différentes zones.

L’architecture sera construite autour des principes suivants :

- séparation des flux réseau ;
- contrôle des accès par firewall ;
- principe du moindre privilège ;
- accès distant via un bastion ;
- centralisation des informations réseau dans NetBox ;
- centralisation des logs dans Elasticsearch ;
- documentation des procédures de reconstruction ;
- préparation à l’ajout futur d’un troisième site.

Le tunnel VPN ne donnera pas un accès global à toute l’infrastructure. Les flux inter-sites devront être explicitement autorisés par les règles firewall.

## Options envisagées

### Option 1 : Infrastructure centralisée sur un seul site

#### Avantages

- Déploiement plus simple.
- Moins de configuration réseau.
- Moins de composants à maintenir.

#### Inconvénients

- Ne répond pas au besoin d’infrastructure hybride.
- Pas de séparation réelle entre site local et site distant.
- Pas de test réaliste d’un VPN site-to-site.
- Moins représentatif d’une infrastructure professionnelle.
- Évolutivité limitée.

### Option 2 : Deux sites Proxmox interconnectés par VPN site-to-site

#### Avantages

- Répond directement au besoin d’architecture hybride.
- Permet de tester une vraie interconnexion sécurisée.
- Permet de séparer les rôles et les flux réseau.
- Permet de contrôler les communications avec pfSense.
- Prépare l’ajout futur d’autres sites.
- Offre une architecture réaliste et professionnelle.

#### Inconvénients

- Configuration plus complexe.
- Nécessite une bonne gestion du routage.
- Nécessite une documentation précise des flux.
- Demande des tests sur le VPN, le DNS, les firewalls et la supervision.

### Option 3 : Architecture multi-sites avancée dès le départ

#### Avantages

- Très évolutif.
- Démontre immédiatement une capacité multi-sites.
- Permet de valider plus fortement la scalabilité de l’architecture.

#### Inconvénients

- Trop complexe pour le périmètre initial.
- Risque de dépasser les contraintes du projet.
- Plus difficile à sécuriser correctement.
- Plus difficile à documenter.

## Justification

L’option retenue est l’architecture hybride à deux sites Proxmox VE interconnectés par VPN site-to-site.

Ce choix correspond directement aux objectifs du projet. Il permet de mettre en place une infrastructure hybride réelle, tout en conservant un périmètre maîtrisable.

L’utilisation d’un VPN site-to-site permet de sécuriser les communications entre les deux sites sans exposer directement les services internes sur Internet. Les firewalls pfSense jouent le rôle de points de contrôle réseau et permettent d’appliquer des règles strictes entre les zones.

Cette architecture répond aussi aux objectifs de sécurité :

- les flux sont contrôlés ;
- les accès d’administration sont limités ;
- les services internes ne sont pas exposés inutilement ;
- les logs peuvent être centralisés ;
- les adresses IP sont documentées ;
- les procédures de restauration peuvent être formalisées.

## Conséquences positives

- Architecture conforme au besoin hybride du projet.
- Sécurisation des échanges entre les deux sites.
- Meilleur contrôle des flux réseau.
- Réduction de l’exposition des services internes.
- Architecture réaliste et professionnelle.
- Préparation à l’ajout futur de nouveaux sites.
- Meilleure traçabilité grâce aux logs centralisés.
- Meilleure documentation grâce à NetBox et aux ADR.

## Conséquences négatives

- Complexité réseau plus importante.
- Besoin de documenter précisément les routes, sous-réseaux et règles firewall.
- Risque d’erreur dans la configuration du VPN.
- Risque d’erreur dans les règles de filtrage inter-sites.
- Besoin de surveiller l’état du tunnel VPN.
- Contrainte forte liée au maximum de 3 VM par site.

## Impact sur le projet

Cette décision structure toute l’infrastructure.

Elle impacte directement :

- le diagramme d’architecture ;
- le plan d’adressage IP ;
- la configuration des firewalls pfSense ;
- la configuration du VPN OpenVPN ;
- la mise en place du bastion ;
- la gestion des flux DNS ;
- l’intégration de NetBox ;
- l’intégration d’Elasticsearch ;
- la stratégie de kill switch ;
- la stratégie de sauvegarde et restauration ;
- la documentation technique finale.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- les deux sites Proxmox sont opérationnels ;
- le tunnel VPN site-to-site fonctionne ;
- les flux inter-sites sont filtrés par pfSense ;
- les services internes ne sont pas exposés directement sur Internet ;
- les accès d’administration passent par les chemins prévus ;
- le plan d’adressage est documenté dans NetBox ;
- les logs critiques sont envoyés vers Elasticsearch ;
- le diagramme d’architecture représente correctement les flux.