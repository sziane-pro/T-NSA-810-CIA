# ADR-003 - Choix de pfSense comme firewall et routeur central par site

## Statut
Accepté

## Date
11/06/2026

## Contexte

Chaque site Proxmox doit disposer d’un point de contrôle réseau permettant de filtrer les flux, gérer le routage, sécuriser les échanges et appliquer les règles de sécurité.

Le projet impose l’utilisation de pfSense pour les fonctions de firewall et de routing.

Les deux sites doivent communiquer via un VPN site-to-site, tout en conservant un filtrage strict entre les réseaux internes et les flux inter-sites.

## Décision

Nous avons décidé d’utiliser une instance pfSense sur chaque site Proxmox.

Chaque pfSense aura les responsabilités suivantes :

- routage entre les différents réseaux du site ;
- filtrage des flux entrants et sortants ;
- terminaison ou gestion du VPN site-to-site ;
- application des règles de sécurité ;
- support d’une coupure d’urgence ;
- contrôle des flux DNS, VPN, bastion, IPAM et logs.

pfSense sera placé comme point de passage obligatoire entre les zones réseau sensibles.

## Options envisagées

### Option 1 : Utiliser le firewall natif de Proxmox uniquement

#### Avantages

- Déploiement plus simple.
- Moins de machines virtuelles.
- Intégration directe avec Proxmox.

#### Inconvénients

- Moins adapté au routage inter-zones complet.
- Moins lisible dans une architecture réseau multi-sites.
- Moins pertinent pour représenter un firewall dédié.
- Fonctionnalités VPN et filtrage avancé moins centralisées.

### Option 2 : Utiliser pfSense par site

#### Avantages

- Solution spécialisée firewall/routing.
- Interface d’administration claire.
- Bonne gestion des règles réseau.
- Compatible avec une architecture VPN site-to-site.
- Adapté à une architecture pédagogique et professionnelle.
- Permet une documentation claire des flux.

#### Inconvénients

- Nécessite une VM dédiée par site.
- Demande une configuration rigoureuse.
- Peut devenir un point critique si mal configuré.

### Option 3 : Utiliser un routeur Linux personnalisé

#### Avantages

- Très flexible.
- Léger.
- Automatisable finement.

#### Inconvénients

- Plus complexe à maintenir.
- Moins lisible pour l’évaluation.
- Plus de risques d’erreurs manuelles.
- Demande plus de documentation.

## Justification

pfSense est retenu car il répond directement aux besoins du projet : filtrage, routage, VPN, contrôle des flux et documentation des règles.

Son utilisation permet d’avoir un point de contrôle clair par site. Cela facilite la lecture du diagramme d’architecture, la justification des règles firewall et la mise en place d’une stratégie de sécurité cohérente.

pfSense permet aussi d’appliquer une politique de sécurité basée sur le refus par défaut : seuls les flux nécessaires sont explicitement autorisés.

## Conséquences positives

- Point de contrôle clair par site.
- Centralisation des règles firewall.
- Meilleure lisibilité de l’architecture.
- Gestion plus propre des flux VPN.
- Possibilité de mettre en place un kill switch.
- Facilité de diagnostic réseau.

## Conséquences négatives

- Une VM pfSense consomme une partie de la limite des 3 VM par site.
- Mauvaise configuration possible si les règles ne sont pas testées.
- Le firewall devient un composant critique de l’infrastructure.
- Nécessite une sauvegarde régulière de la configuration.

## Impact sur le projet

Chaque site devra intégrer pfSense dans le schéma d’architecture.

La documentation devra inclure :

- les interfaces réseau de pfSense ;
- les sous-réseaux associés ;
- les règles firewall principales ;
- les règles VPN ;
- les règles DNS ;
- les règles liées au bastion ;
- la procédure de sauvegarde et de restauration de la configuration pfSense.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- chaque site dispose d’un pfSense fonctionnel ;
- les règles firewall appliquent le moindre privilège ;
- les flux inter-zones sont contrôlés ;
- les flux VPN sont filtrés ;
- les configurations pfSense sont sauvegardées et documentées.