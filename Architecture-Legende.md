# Légende de l'Architecture Réseau Hybride Proxmox

## Sites et Infrastructure
- **SITE 1 (On-Premise)** : Datacenter principal avec services centralisés
- **SITE 2 (Remote)** : Site distant avec bastion pour accès externe
- **Chaque site** : Maximum 3 VMs sur Proxmox VE

## Segmentation Réseau

### Site 1
| Segment | Plage | Usage |
|---------|-------|-------|
| LAN | `192.168.10.0/24` | Utilisateurs |
| DMZ | `192.168.20.0/24` | Services exposés |
| ADMIN | `192.168.30.0/24` | Management |

### Site 2
| Segment | Plage | Usage |
|---------|-------|-------|
| LAN | `192.168.110.0/24` | Utilisateurs |
| DMZ | `192.168.120.0/24` | Bastion + Services |
| ADMIN | `192.168.130.0/24` | Management |

## VPN Site-à-Site
- **Tunnel OpenVPN** : `10.0.0.0/30`
- **Chiffrement** : AES-256-GCM
- **Authentification** : RSA-4096 + Certificats
- **Routage** : Automatique entre tous les segments

## Sécurité
- Pare-feu **pfSense** sur chaque site
- Règles de filtrage strictes (Least Privilege)
- **Kill Switch** d'urgence sur les deux sites
- **Bastion Host** avec 2FA pour accès externe
- Logging centralisé de toutes les connexions

## Services Centralisés
| Service | Outil | Localisation | Rôle |
|---------|-------|--------------|------|
| IPAM | NetBox | Site 1 | Source de vérité unique |
| Monitoring | Elasticsearch | Site 1 | Logs multi-sites |
| DNS | pfSense | Hiérarchie | Résolution croisée |
| VPN Hub | OpenVPN | Site 1 | Point central |

## Automatisation
- **API NetBox** pour gestion IP automatisée
- Synchronisation temps réel des configurations
- Templates **IaC** pour déploiement reproductible
- **Webhooks** pour mises à jour automatiques

## Monitoring
- Collecte logs centralisée (**Elasticsearch**)
- Dashboards **Kibana** pour visualisation
- Métriques système et réseau (**Beats**)
- Alerting automatique sur anomalies

## Procédures d'Urgence
| Procédure | Description |
|-----------|-------------|
| Kill Switch | Coupure VPN immédiate |
| Isolation sites | Maintien accès bastion |
| Recovery | Scripts automatisés de reconnexion |
| Runbooks | Procédures détaillées de récupération |

## Évolutivité
- Convention d'adressage extensible
- Templates réutilisables pour nouveaux sites
- Architecture **Hub-and-spoke** scalable
- Documentation automatisée de la topologie

## Flux Principaux
1. **Utilisateur externe** → Bastion Host (SSH)
2. **Site 1** ↔ **Site 2** (VPN chiffré)
3. **Tous les logs** → Elasticsearch (Site 1)
4. **Gestion IP** → NetBox API (Site 1)
5. **DNS** : Résolution croisée entre sites
