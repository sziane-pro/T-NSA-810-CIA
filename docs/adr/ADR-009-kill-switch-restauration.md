# ADR-009 - Choix d’une stratégie de kill switch et de restauration

## Statut
Accepté

## Date
11/06/2026

## Contexte

Le projet impose une capacité de coupure d’urgence, appelée kill switch.

Cette coupure doit permettre d’isoler rapidement une partie de l’infrastructure en cas :

- d’incident de sécurité ;
- de mauvaise configuration ;
- de comportement réseau anormal ;
- de compromission suspectée ;
- de propagation non souhaitée entre les sites.

Cependant, la coupure d’urgence ne doit pas empêcher la restauration de l’infrastructure.

Il faut donc prévoir une stratégie permettant d’interrompre certains flux sans perdre l’accès aux moyens de diagnostic et de reprise.

## Décision

Nous avons décidé de mettre en place une stratégie de kill switch basée sur des règles firewall contrôlées dans pfSense.

Le kill switch doit permettre de couper rapidement :

- les flux inter-sites via VPN ;
- certains flux sortants ;
- certains accès vers les services internes ;
- les communications non essentielles.

La restauration doit rester possible grâce à des accès d’administration contrôlés, notamment via le réseau Admin ou le bastion selon le scénario.

## Options envisagées

### Option 1 : Coupure complète de toutes les communications

#### Avantages

- Isolation immédiate.
- Réduction rapide du risque de propagation.

#### Inconvénients

- Peut bloquer l’administration.
- Peut empêcher le diagnostic.
- Peut ralentir la restauration.
- Risque opérationnel élevé.

### Option 2 : Kill switch ciblé par règles firewall

#### Avantages

- Isolation contrôlée.
- Possibilité de maintenir un accès d’administration.
- Meilleure maîtrise des impacts.
- Compatible avec pfSense.
- Plus adapté à un runbook de restauration.

#### Inconvénients

- Nécessite une bonne préparation.
- Besoin de tester les scénarios.
- Demande une documentation précise.

### Option 3 : Désactivation manuelle des services un par un

#### Avantages

- Très granulaire.
- Peut être adapté à chaque incident.

#### Inconvénients

- Trop lent en situation d’urgence.
- Risque d’erreur humaine.
- Moins fiable.
- Difficile à documenter simplement.

## Justification

Le kill switch ciblé par règles firewall est retenu car il offre un bon équilibre entre sécurité et capacité de restauration.

Il permet d’isoler rapidement des flux dangereux sans couper totalement les accès nécessaires à l’administration et au diagnostic.

Cette approche est cohérente avec l’utilisation de pfSense comme point de contrôle central par site.

## Conséquences positives

- Réaction rapide en cas d’incident.
- Limitation de la propagation.
- Maintien possible des accès de restauration.
- Procédure documentable dans un runbook.
- Meilleure maîtrise des flux critiques.

## Conséquences négatives

- Besoin de tester les règles avant un incident réel.
- Risque de mauvaise activation si la procédure est floue.
- Nécessite une documentation très claire.
- Demande de prévoir plusieurs scénarios.

## Impact sur le projet

La documentation finale devra inclure :

- les règles concernées par le kill switch ;
- les flux coupés ;
- les flux maintenus pour la restauration ;
- la procédure d’activation ;
- la procédure de retour arrière ;
- les tests réalisés ;
- les captures d’écran ou exports de configuration.

## Preuves de mise en œuvre

Cette décision est mise en œuvre par :

- un scénario de coupure est défini ;
- les flux coupés sont identifiés ;
- les flux nécessaires à la restauration restent disponibles ;
- une procédure de retour arrière existe ;
- le kill switch est testé ou simulé.