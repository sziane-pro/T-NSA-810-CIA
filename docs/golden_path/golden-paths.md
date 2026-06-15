# Golden Paths — Templates réutilisables

## Objectif du document

L’objectif est de démontrer que le projet est basée sur des **modèles réutilisables**, documentés et versionnés dans Git.

Ces modèles permettent de standardiser :

* la création ou la description des machines virtuelles ;
* les règles firewall ;
* la déclaration IPAM / NetBox ;
* la collecte et la structuration des logs ;
* le déploiement de services critiques comme Vault, Bastion, NetBox, OpenSearch ou le serveur web.

Un **Golden Path** représente donc une méthode recommandée et reproductible pour déployer ou configurer un composant de l’infrastructure.

---

# 1. Comment utiliser ces Golden Paths

Pour utiliser un Golden Path, il faut :

1. choisir le template adapté ;
2. modifier les variables associées ;
3. exécuter le playbook Ansible correspondant ;
4. vérifier que la configuration générée est correcte ;
5. vérifier que les logs remontent dans OpenSearch ;
6. documenter les éventuelles adaptations.

Exemple avec Vault :

```text
# Template utilisé :
infra-ansible/roles/vault/templates/vault.hcl.j2

# Playbook utilisé :
infra-ansible/playbooks/vault.yml
```

Exemple avec les logs :

```text
# Templates utilisés :
infra-ansible/roles/fluentbit/templates/fluent-bit.conf.j2
infra-ansible/roles/webserver/templates/filebeat.yml.j2
infra-ansible/roles/bastion/templates/rsyslog-bastion.conf.j2

# Playbooks utilisés :
infra-ansible/playbooks/fluent-bit.yml
infra-ansible/playbooks/fluent-bit_s2.yml
infra-ansible/playbooks/webserver_s2.yml
```

---

# 2. Synthèse des Golden Paths du projet

| Catégorie attendue       | Fichier ou template associé                                                                      | Statut    |
|--------------------------|--------------------------------------------------------------------------------------------------|-----------|
| VMs                      | [`vm-template.yml`](./vm-template.yml)                                                           |           |
| IPAM / NetBox            | [`netbox_populate_template.yml`](./netbox_populate_template.yml)                                 | Présent   |
| Logs / Fluent Bit        | [`fluent-bit.conf.j2`](../../infra-ansible/roles/fluentbit/templates/fluent-bit.conf.j2)         | Présent   |
| Logs / Filebeat          | [`filebeat.yml.j2`](../../infra-ansible/roles/webserver/templates/filebeat.yml.j2)               | Présent   |
| Logs / Rsyslog Bastion   | [`rsyslog-bastion.conf.j2`](../../infra-ansible/roles/bastion/templates/rsyslog-bastion.conf.j2) | Présent   |
| Vault                    | [`vault.hcl.j2`](../../infra-ansible/roles/bastion/templates/rsyslog-bastion.conf.j2)            | Présent   |
| Bastion SSH              | [`sshd_config.j2`](../../infra-ansible/roles/bastion/templates/sshd_config.j2)                   | Présent   |
| Bastion Fail2ban         | [`jail.local.j2`](../../infra-ansible/roles/bastion/templates/jail.local.j2)                     | Présent   |
| NetBox configuration     | [`configuration.py.j2`](../../infra-ansible/roles/netbox/templates/configuration.py.j2)          | Présent   |
| NetBox service systemd   | [`netbox.service.j2`](../../infra-ansible/roles/netbox/templates/netbox.service.j2)              | Présent   |
| NetBox reverse proxy     | [`nginx.conf.j2`](../../infra-ansible/roles/netbox/templates/nginx.conf.j2)                      | Présent   |
| OpenSearch configuration | [`opensearch.yml.j2`](../../infra-ansible/roles/opensearch/templates/opensearch.yml.j2)          | Présent   |
| OpenSearch users         | [`internal_users.yml.j2`](../../infra-ansible/roles/opensearch/templates/internal_users.yml.j2)  | Présent   |
| Webserver Nginx          | [`nginx.conf.j2`](../../infra-ansible/roles/webserver/templates/nginx.conf.j2)                   | Présent   |
| Webserver virtual host   | [`default.conf.j2`](../../infra-ansible/roles/webserver/templates/default.conf.j2)               | Présent   |
| Webserver page interne   | [`index.html.j2`](../../infra-ansible/roles/webserver/templates/index.html.j2)                   | Présent   |

---

# 3. Golden Path — VMs

## Objectif

Le Golden Path VM permet de définir une structure standard pour décrire une machine virtuelle du projet.

Même si les VMs sont créées manuellement dans Proxmox, ce template permet de documenter et reproduire leur configuration de manière homogène.

Il sert de référence pour :

* le nommage des VMs ;
* le site d’appartenance ;
* le rôle de la VM ;
* les ressources CPU / RAM / disque ;
* le réseau associé ;
* les paramètres de sécurité ;
* l’activation de la supervision et des logs.

---

# 4. Golden Path — Rules / Firewall

## Objectif

Le Golden Path Firewall permet de standardiser la création des règles de sécurité.

L’objectif est d’éviter les règles créées sans convention, sans justification ou sans journalisation.

Chaque règle firewall doit préciser :

* un nom ;
* une description ;
* une source ;
* une destination ;
* un protocole ;
* un port ;
* une action ;
* l’activation ou non des logs ;
* une justification.

---

# 5. Golden Path — IPAM / NetBox

## Objectif

Le Golden Path IPAM permet de standardiser la déclaration des sites, réseaux, VLANs, machines et adresses IP dans NetBox.

NetBox étant utilisé comme source de vérité IPAM, il est important que les données réseau suivent une structure commune.

Ce modèle permet de décrire :

* un site ;
* ses préfixes réseau ;
* ses VLANs ;
* ses équipements ;
* ses adresses IP ;
* ses noms DNS.

---

# 6. Golden Path — Logs

## Objectif

Le Golden Path Logs permet de standardiser la collecte, l’envoi et la structure des logs vers la plateforme d’observabilité.

Dans le projet, plusieurs services utilisent déjà des templates Ansible pour configurer les agents ou services de logs.

---

# 7. Golden Path — Vault

## Objectif

Même si Vault n’est pas directement cité dans le bonus Golden Paths, il renforce la logique de template réutilisable.

Le projet utilise Vault comme solution de gestion sécurisée des secrets.

Son déploiement est standardisé via un rôle Ansible et un template Jinja2.

---

# 8. Golden Path — Bastion

## Objectif

Le bastion est un composant central de l’architecture, car il permet l’accès administratif sécurisé au site distant.

Le projet contient plusieurs templates permettant de standardiser sa configuration.

---

# 9. Golden Path — NetBox

## Objectif

NetBox est utilisé comme IPAM et source de vérité réseau.

Le projet contient déjà des templates pour déployer et configurer NetBox.

---

# 10. Golden Path — OpenSearch

## Objectif

OpenSearch est utilisé pour l’observabilité et l’analyse des logs.

Le projet contient des templates permettant de standardiser sa configuration.

---

# 11. Golden Path — Webserver

## Objectif

Le serveur web interne est demandé dans l’architecture du projet.

Le projet contient plusieurs templates permettant de standardiser son déploiement.

