# Monitoring avancé avec Fluent Bit, OpenSearch et OpenSearch Dashboards

## 1. Objectif

Ce document présente la mise en place d’un système de monitoring avancé basé sur la collecte, le parsing, l’indexation, la visualisation et l’analyse des logs.

La solution mise en place permet de couvrir les éléments suivants :

* collecte centralisée des logs systèmes et applicatifs ;
* extraction et parsing des payloads JSON encapsulés ;
* enrichissement des logs avec des métadonnées géographiques et systèmes ;
* indexation des logs dans OpenSearch ;
* création de dashboards dans OpenSearch Dashboards ;
* configuration d’alertes sur les anomalies critiques ;
* analyse et investigation des incidents.

---

## 2. Stack technique utilisée

La stack de monitoring repose sur les composants suivants :

| Composant               | Rôle                                                        |
| ----------------------- | ----------------------------------------------------------- |
| Fluent Bit              | Collecte, parsing, enrichissement et transfert des logs     |
| OpenSearch              | Stockage, indexation et recherche dans les logs             |
| OpenSearch Dashboards   | Visualisation, dashboards, exploration des logs et alerting |
| Serveur Web / Backend   | Génération des logs                                         |

---

## 3. Architecture globale

L’architecture de monitoring est organisée autour d’une chaîne complète de traitement des logs.

```txt
Serveur (auth.log, syslog, etc.)
   ↓
Logs bruts (Texte + payload JSON)
   ↓
Fluent Bit (Filtre Regex + Decode JSON)
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

1. Le serveur ou l’application génère des logs.
2. Fluent Bit collecte les logs.
3. Fluent Bit parse les logs pour extraire les champs importants.
4. Fluent Bit enrichit les logs avec des métadonnées.
5. Les logs structurés sont envoyés vers OpenSearch.
6. OpenSearch indexe les logs.
7. OpenSearch Dashboards permet de visualiser les logs, créer des dashboards et configurer des alertes.

---

## 4. Collecte des logs avec Fluent Bit

Fluent Bit est utilisé comme agent de collecte de logs.

Il permet de :

* récupérer les logs générés par l’application ;
* lire les logs depuis des fichiers ;
* parser les logs structurés ;
* ajouter des informations complémentaires ;
* transférer les logs vers OpenSearch.

---

## 5. Parsing des logs

Les logs collectés contiennent souvent un en-tête système suivi d'un payload JSON. L’objectif du parsing est d'isoler ce JSON pour transformer le log brut en un document structuré avec des champs facilement filtrables.

### Exemple de parser Fluent Bit

```ini
[PARSER]
   Name         extract_syslog_json
   Format       regex
   Regex        ^(?<syslog_time>[^ ]+)\s+(?<syslog_host>[^ ]+)\s+(?<process>[^:]+):\s+(?<json_payload>\{.*\})$
   Decode_Field json json_payload
```

Ce parser permet à Fluent Bit de lire une ligne de log classique, d'isoler la partie json_payload, et de la décoder pour injecter les variables directement à la racine du document dans OpenSearch.

---

## 6. Champs exploités dans OpenSearch (A ADAPTER)

Les logs envoyés dans OpenSearch contiennent plusieurs champs permettant l’analyse et la recherche.

| Champ             | Description                                       |
| ----------------- | ------------------------------------------------- |
| `@timestamp`      | Date et heure du log                              |
| `response`        | Code de statut HTTP retourné (ex: 200, 404, 500)  |
| `url`             | Route API ou page web appelée                     |
| `request`         | Méthode HTTP ou requête brute utilisée            |
| `clientip`        | Adresse IP de l'utilisateur ou du client          |
| `geo.coordinates` | Coordonnées géographiques déduites de l'IP        |
| `machine.os`      | Système d'exploitation du client                  |
| `bytes`           | Taille de la réponse renvoyée en octets           |
| `_source_host`    | VM source ayant généré le log                     |

Ces champs permettent de filtrer rapidement les logs dans OpenSearch Dashboards.

---

## 7. Indexation dans OpenSearch

Les logs collectés par Fluent Bit sont envoyés vers OpenSearch dans un index dédié (ex: fluent-bit-*).

L’indexation permet ensuite de rechercher, filtrer, agréger et visualiser les données géographiques et systèmes dans OpenSearch Dashboards.

---

## 8. Recherches utiles dans OpenSearch Dashboards

OpenSearch Dashboards permet d’explorer les logs à l’aide de requêtes.

### Rechercher les erreurs HTTP 500

```txt
response: 500
```

### Rechercher toute les erreurs côté client et serveur

```txt
response >= 400
```

### Rechercher les erreurs sur un endpoint précis

```txt
url: "/api/users" AND response >= 400
```

### Rechercher les requêtes provenant d'une IP spécifique

```txt
clientip: "192.168.1.1"
```

### Rechercher les logs provenant d'une machine spécifique

```txt
_source_host: "webserver_vm"
```

Ces requêtes facilitent l’analyse des incidents et permettent de réduire le temps de diagnostic.

---

## 9. Dashboards

Des dashboards sont créés dans OpenSearch Dashboards afin d’avoir une vision claire de l’état du trafic et de la santé des services.

### Dashboard analytique

Le dashboard applicatif permet de suivre les indicateurs suivants :

* volume total des requêtes ;
* répartition des codes HTTP (response) ;
* cartographie du trafic entrant (geo.coordinates) ;
* répartition des systèmes d'exploitation et navigateurs (machine.os, agent) ;
* top des endpoints générant le plus d’erreurs (url) ;
* top des adresses IP clientes (clientip).

### Exemples de visualisations

| Visualisation          | Objectif                                             |
| ---------------------- | ---------------------------------------------------- |
| Metric Card (Erreurs)  | Identifier rapidement une hausse d’erreurs 500/400   |
| Histogramme HTTP       | Visualiser la santé globale et le volume du trafic   |
| Coordinate Map         | Afficher la provenance géographique des requêtes     |
| Data Table (Endpoints) | Identifier les routes les plus instables ou visitées |
| Pie Chart (OS/Agent)   | Analyser les habitudes des utilisateurs              |

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

### Alerte : erreurs HTTP 500

Condition surveillée :

```txt
response: 500
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

### Alerte : endpoint instable ou scan de vulnérabilité

Condition surveillée :

```txt
response >= 400
```

Seuil d’alerte :

```txt
Nombre d’erreurs > 50 sur 10 minutes (provenant de la même IP)
```

Niveau :

```txt
Warning
```

Objectif :

Identifier un endpoint en échec répété ou un comportement suspect d'une adresse IP.

---

## 11. Procédure d’investigation

En cas d’alerte, la procédure suivante peut être appliquée :

1. Ouvrir OpenSearch Dashboards.
2. Sélectionner la période correspondant à l’alerte.
3. Filtrer les logs avec le code HTTP concerné (response).
4. Identifier la route (url) en erreur.
5. Isoler l'adresse IP cliente (clientip) pour retracer le parcours de l'utilisateur.
6. Identifier la cause probable de l’incident (ex: tentative d'accès non autorisé, erreur backend).
7. Corriger le problème côté infrastructure ou réseau.
8. Vérifier le retour à la normale dans le dashboard.
9. Documenter l’incident si nécessaire.

---

## 12. Exemple d’analyse d’incident

Exemple de scénario :

Une alerte est déclenchée car le nombre d’erreurs HTTP 500 dépasse le seuil configuré.

### Étapes d’analyse

Filtrage des erreurs HTTP 500 :

```txt
response: 500
```

Identification des endpoints les plus touchés :

```txt
Agrégation sur le champ "url"
```

Filtrage sur l'IP cliente pour retracer l'action :

```txt
clientip: "203.0.113.42"
```

### Conclusion possible

L’analyse croisée de la route (url) et des tentatives de l'utilisateur (clientip) permet de confirmer si l'erreur 500 est due à un cas d'usage spécifique ou à une indisponibilité générale d'un service.

---

## 13. Organisation recommandée du repo

Une organisation possible des fichiers liés au monitoring est la suivante :

```txt
roles/fluentbit/
├── tasks/
│   └── main.yml
├── templates/
│   ├── fluent-bit.conf.j2
│   └── parsers.conf.j2
└── handlers/
    └── main.yml

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
* l’extraction de métriques web clés (clientip, response, url) ;
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
* définition d’un parser Regex personnalisé pour structurer les logs ;
* envoi des logs vers OpenSearch ;
* exploitation des logs dans OpenSearch Dashboards ;
* création de visualisations ;
* création de dashboards ;
* mise en place de règles d’alerte ;
* documentation d’une procédure d’investigation.
