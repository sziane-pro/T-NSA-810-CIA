# ADR-004 - Choix d’OpenVPN pour le VPN site-to-site

## Statut
Accepté

## Date
11/06/2026

## Contexte

Les deux sites de l’infrastructure doivent communiquer de manière sécurisée à travers un réseau non maîtrisé.

Les échanges entre les sites peuvent concerner :

- l’administration ;
- les services internes ;
- la résolution DNS ;
- les logs ;
- l’accès à des ressources internes.

Le projet impose la mise en place d’un VPN site-to-site chiffré avec OpenVPN.

## Décision

Nous avons décidé d’utiliser OpenVPN pour établir un tunnel VPN site-to-site chiffré entre le site on-premise et le site distant.

Le VPN permettra de router uniquement les sous-réseaux nécessaires entre les deux sites.

Les flux traversant le tunnel seront ensuite filtrés par les firewalls pfSense.

Le VPN ne doit pas donner un accès complet et implicite à tous les réseaux. Chaque flux inter-site devra être autorisé explicitement par les règles firewall.

## Options envisagées

### Option 1 : Exposer directement les services sur Internet

#### Avantages

- Simplicité apparente.
- Pas de tunnel VPN à configurer.

#### Inconvénients

- Risque de sécurité élevé.
- Surface d’exposition importante.
- Non conforme aux objectifs du projet.
- Mauvaise pratique pour des services internes.

### Option 2 : VPN site-to-site avec OpenVPN

#### Avantages

- Chiffrement des communications.
- Compatible avec pfSense.
- Adapté à l’interconnexion de sites.
- Permet de router uniquement certains sous-réseaux.
- Solution largement documentée.

#### Inconvénients

- Configuration plus complexe.
- Besoin de gérer les certificats et clés.
- Nécessite des tests de routage et de filtrage.
- Peut devenir un point critique si le tunnel tombe.

### Option 3 : VPN basé sur WireGuard

#### Avantages

- Performant.
- Configuration souvent plus légère.
- Moderne.

#### Inconvénients

- Non conforme à la technologie imposée.
- Moins aligné avec les contraintes du projet.
- Nécessiterait une justification supplémentaire.

## Justification

OpenVPN est retenu car il est imposé par le projet et adapté à la création d’un VPN site-to-site sécurisé.

Il permet de chiffrer les communications entre les deux sites et d’éviter l’exposition directe des services internes sur Internet.

Son intégration avec pfSense permet de centraliser la gestion du VPN et du filtrage réseau. Les règles firewall restent indispensables afin de limiter les communications aux seuls flux nécessaires.

## Conséquences positives

- Communications inter-sites chiffrées.
- Réduction de l’exposition des services internes.
- Contrôle des routes entre sites.
- Intégration cohérente avec pfSense.
- Architecture compatible avec l’ajout futur d’autres sites.

## Conséquences négatives

- Gestion des certificats et secrets à sécuriser.
- Besoin de documenter les routes VPN.
- Dépendance au bon fonctionnement du tunnel.
- Besoin de surveillance et de logs sur l’état du VPN.

## Impact sur le projet

La documentation devra préciser :

- les endpoints VPN ;
- les sous-réseaux routés ;
- le mode de chiffrement ;
- les règles firewall associées ;
- les tests de connectivité ;
- la procédure de coupure d’urgence ;
- la procédure de restauration du tunnel.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- le tunnel VPN est opérationnel ;
- seuls les sous-réseaux prévus sont routés ;
- les flux inter-sites sont filtrés ;
- les logs VPN sont disponibles ;
- une procédure de restauration du tunnel existe.