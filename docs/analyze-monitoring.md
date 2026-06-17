# Monitoring avancé avec Fluent Bit, OpenSearch et OpenSearch Dashboards

## 1. Objectif

Ce document présente la mise en place d’un système de monitoring avancé basé sur la collecte, le parsing, l’indexation, la visualisation et l’analyse des logs applicatifs.

La solution mise en place permet de couvrir les éléments suivants :

* collecte centralisée des logs applicatifs ;
* parsing et structuration des logs ;
* enrichissement des logs avec des métadonnées ;
* indexation des logs dans OpenSearch ;
* création de dashboards dans OpenSearch Dashboards ;
* configuration d’alertes sur les anomalies critiques ;
* analyse et investigation des incidents applicatifs.

---

## 2. Stack technique utilisée

La stack de monitoring repose sur les composants suivants :

| Composant               | Rôle                                                        |
| ----------------------- | ----------------------------------------------------------- |
| Fluent Bit              | Collecte, parsing, enrichissement et transfert des logs     |
| OpenSearch              | Stockage, indexation et recherche dans les logs             |
| OpenSearch Dashboards   | Visualisation, dashboards, exploration des logs et alerting |
| Application backend     | Génération des logs applicatifs                             |

---

## 3. Architecture globale

L’architecture de monitoring est organisée autour d’une chaîne complète de traitement des logs.

```txt
Application backend
   ↓
Logs applicatifs
   ↓
Fluent Bit
   ↓
Parsing + enrichissement
   ↓
OpenSearch
   ↓
OpenSearch Dashboards
   ↓
Dashboards / Alertes / Investigation
```

Le rôle de chaque étape est le suivant :

1. L’application génère des logs.
2. Fluent Bit collecte les logs.
3. Fluent Bit parse les logs pour extraire les champs importants.
4. Fluent Bit enrichit les logs avec des métadonnées.
5. Les logs sont envoyés vers OpenSearch.
6. OpenSearch indexe les logs.
7. OpenSearch Dashboards permet de visualiser les logs, créer des dashboards et configurer des alertes.

---

## 4. Collecte des logs avec Fluent Bit

Fluent Bit est utilisé comme agent de collecte de logs.

Il permet de :

* récupérer les logs générés par l’application ;
* lire les logs depuis des fichiers ou depuis la sortie standard des conteneurs ;
* parser les logs structurés ;
* ajouter des informations complémentaires ;
* transférer les logs vers OpenSearch.

---

## 5. Parsing des logs (A SUPPRIMER SI PAS DE PARSER DE LOGS)

Les logs applicatifs sont structurés afin d’être exploitables dans OpenSearch Dashboards.

L’objectif du parsing est de transformer un log brut en document structuré avec des champs facilement filtrables.

### Exemple de log applicatif

```json
{
  "timestamp": "2026-06-17T10:15:30",
  "level": "ERROR",
  "endpoint": "/api/users",
  "method": "GET",
  "status": 500,
  "message": "Database connection timeout",
  "correlationId": "req-123456"
}
```

### Exemple de parser Fluent Bit

```ini
[PARSER]
    Name        app_json
    Format      json
    Time_Key    timestamp
    Time_Format %Y-%m-%dT%H:%M:%S
```

Ce parser permet à Fluent Bit d’interpréter les logs au format JSON et d’utiliser le champ `timestamp` comme date de référence.

---

## 6. Champs exploités dans OpenSearch (A ADAPTER)

Les logs envoyés dans OpenSearch contiennent plusieurs champs permettant l’analyse et la recherche.

| Champ           | Description                                       |
| --------------- | ------------------------------------------------- |
| `timestamp`     | Date et heure du log                              |
| `level`         | Niveau du log : INFO, WARN, ERROR                 |
| `service`       | Service applicatif concerné                       |
| `environment`   | Environnement concerné : dev, staging, production |
| `endpoint`      | Route API appelée                                 |
| `method`        | Méthode HTTP utilisée                             |
| `status`        | Code HTTP retourné                                |
| `message`       | Message du log                                    |
| `correlationId` | Identifiant permettant de suivre une requête      |

Ces champs permettent de filtrer rapidement les logs dans OpenSearch Dashboards.

---

## 7. Indexation dans OpenSearch

Les logs collectés par Fluent Bit sont envoyés vers OpenSearch dans un index dédié.

L’indexation permet ensuite de rechercher, filtrer, agréger et visualiser les logs dans OpenSearch Dashboards.

---

## 8. Recherches utiles dans OpenSearch Dashboards

OpenSearch Dashboards permet d’explorer les logs à l’aide de requêtes.

### Rechercher toutes les erreurs applicatives

```txt
level: "ERROR"
```

### Rechercher les erreurs HTTP 500

```txt
status: 500
```

### Rechercher les erreurs sur un endpoint précis

```txt
endpoint: "/api/users" AND level: "ERROR"
```

### Suivre une requête avec un correlationId

```txt
correlationId: "req-123456"
```

### Rechercher les logs d’un service précis

```txt
service: "backend-api"
```

### Rechercher les logs d’un environnement précis

```txt
environment: "dev"
```

Ces requêtes facilitent l’analyse des incidents et permettent de réduire le temps de diagnostic.

---

## 9. Dashboards

Des dashboards sont créés dans OpenSearch Dashboards afin d’avoir une vision claire de l’état de l’application.

### Dashboard applicatif

Le dashboard applicatif permet de suivre les indicateurs suivants :

* volume total de logs ;
* nombre de logs par niveau ;
* nombre d’erreurs dans le temps ;
* répartition des codes HTTP ;
* erreurs par endpoint ;
* top des endpoints générant le plus d’erreurs ;
* top des messages d’erreur ;
* logs filtrés par service ;
* logs filtrés par environnement.

### Exemples de visualisations

| Visualisation          | Objectif                                       |
| ---------------------- | ---------------------------------------------- |
| Logs par niveau        | Identifier une hausse d’erreurs ou de warnings |
| Erreurs dans le temps  | Suivre l’évolution des incidents               |
| Codes HTTP             | Visualiser la santé globale de l’API           |
| Erreurs par endpoint   | Identifier les routes les plus instables       |
| Top messages d’erreur  | Repérer les erreurs les plus fréquentes        |
| Logs par service       | Comparer le comportement des services          |
| Logs par environnement | Distinguer dev, staging et production          |

### Captures d’écran à ajouter

Les captures d’écran suivantes peuvent être ajoutées dans le dossier `docs/images/` :

```txt
docs/images/dashboard-overview.png
docs/images/dashboard-errors.png
docs/images/dashboard-http-status.png
docs/images/alert-rule.png
```

Exemple d’intégration dans la documentation :

```md
![Dashboard overview](./images/dashboard-overview.png)

![Dashboard erreurs](./images/dashboard-errors.png)

![Règle d’alerte](./images/alert-rule.png)
```

---

## 10. Alerting

Des alertes sont configurées dans OpenSearch Dashboards afin de détecter automatiquement les anomalies.

L’objectif est d’être notifié lorsqu’un comportement anormal est détecté, par exemple une hausse importante des erreurs applicatives ou des erreurs HTTP 500.

### Alerte : taux d’erreurs élevé

Condition surveillée :

```txt
level: "ERROR"
```

Seuil d’alerte :

```txt
Nombre d’erreurs > 10 sur 5 minutes
```

Niveau :

```txt
Critical
```

Objectif :

Détecter rapidement une augmentation anormale des erreurs applicatives.

---

### Alerte : erreurs HTTP 500

Condition surveillée :

```txt
status: 500
```

Seuil d’alerte :

```txt
Nombre d’erreurs HTTP 500 > 5 sur 5 minutes
```

Niveau :

```txt
Critical
```

Objectif :

Identifier les erreurs serveur susceptibles d’impacter les utilisateurs.

---

### Alerte : endpoint instable

Condition surveillée :

```txt
endpoint: "/api/users" AND status >= 500
```

Seuil d’alerte :

```txt
Nombre d’erreurs > 3 sur 10 minutes
```

Niveau :

```txt
Warning
```

Objectif :

Identifier un endpoint instable ou en échec répété.

---

## 11. Procédure d’investigation

En cas d’alerte, la procédure suivante peut être appliquée :

1. Ouvrir OpenSearch Dashboards.
2. Sélectionner la période correspondant à l’alerte.
3. Filtrer les logs avec les champs concernés.
4. Identifier le service ou l’endpoint en erreur.
5. Analyser les messages d’erreur associés.
6. Utiliser le `correlationId` pour suivre le parcours d’une requête.
7. Identifier la cause probable de l’incident.
8. Corriger le problème ou créer un ticket d’anomalie.
9. Vérifier le retour à la normale dans le dashboard.
10. Documenter l’incident si nécessaire.

---

## 12. Exemple d’analyse d’incident

Exemple de scénario :

Une alerte est déclenchée car le nombre d’erreurs HTTP 500 dépasse le seuil configuré.

### Étapes d’analyse

Filtrage des erreurs HTTP 500 :

```txt
status: 500
```

Filtrage sur l’endpoint concerné :

```txt
endpoint: "/api/users" AND status: 500
```

Analyse du message d’erreur :

```txt
message: "Database connection timeout"
```

Suivi d’une requête précise :

```txt
correlationId: "req-123456"
```

### Conclusion possible

L’analyse des logs permet d’identifier une erreur liée à une indisponibilité temporaire de la base de données ou à un timeout de connexion.

Cette information permet d’orienter rapidement l’investigation vers la couche base de données ou la configuration réseau.

---

## 13. Organisation recommandée du repo

Une organisation possible des fichiers liés au monitoring est la suivante :

```txt
monitoring/
├── fluent-bit.conf
├── parsers.conf
├── docker-compose.monitoring.yml
└── README.md

docs/
├── monitoring-opensearch.md
└── images/
    ├── dashboard-overview.png
    ├── dashboard-errors.png
    ├── dashboard-http-status.png
    └── alert-rule.png
```

---

## 14. Résultat obtenu

La mise en place de cette chaîne de monitoring permet de disposer d’une solution complète pour la supervision applicative.

Elle couvre :

* la collecte automatisée des logs avec Fluent Bit ;
* le parsing des logs applicatifs ;
* l’enrichissement des logs avec des métadonnées ;
* l’indexation des logs dans OpenSearch ;
* la visualisation des données dans OpenSearch Dashboards ;
* la création de dashboards de suivi ;
* la configuration d’alertes ;
* l’analyse rapide des incidents.

Cette solution améliore la capacité à détecter, comprendre et résoudre les anomalies applicatives.

---

## 15. Compétence validée

Cette mise en place permet de valider la compétence suivante :

> Advanced monitoring: dashboards, alerting, log parsing.

Les éléments concrets permettant de justifier cette compétence sont :

* configuration de Fluent Bit pour la collecte des logs ;
* définition d’un parser pour structurer les logs ;
* envoi des logs vers OpenSearch ;
* exploitation des logs dans OpenSearch Dashboards ;
* création de visualisations ;
* création de dashboards ;
* mise en place de règles d’alerte ;
* documentation d’une procédure d’investigation.
