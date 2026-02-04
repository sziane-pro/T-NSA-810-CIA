# Architecture Réseau - Infrastructure Hybride Proxmox
## Contexte et Objectifs
Conception d'une infrastructure hybride sécurisée composée de deux sites Proxmox (on-premise et remote) avec interconnexion VPN, pare-feu, IPAM automatisé, monitoring centralisé et capacité d'extension future.
**Contraintes techniques:**
* Maximum 3 VMs par site Proxmox
* Technologies maintenues et supportées par la communauté
* Architecture évolutive pour intégration de sites supplémentaires
## Architecture Réseau Proposée
### Site 1 - On-Premise (Datacenter Principal)
**Réseaux segmentés:**
* **LAN:** 192.168.10.0/24 (réseau utilisateurs)
* **DMZ:** 192.168.20.0/24 (services exposés)
* **ADMIN:** 192.168.30.0/24 (management)
* **WAN:** Interface publique pour VPN
**VMs déployées (3 max):**
1. **pfSense-S1** (192.168.30.1) - Pare-feu/Router principal
    * Interface WAN: IP publique
    * Interface LAN: 192.168.10.1
    * Interface DMZ: 192.168.20.1
    * Interface ADMIN: 192.168.30.1
    * Serveur OpenVPN site-à-site
2. **NetBox-Server** (192.168.20.10) - IPAM centralisé
    * Base de données IPAM/DCIM
    * API pour automatisation
    * Interface web sécurisée
3. **Elasticsearch-Server** (192.168.20.20) - Monitoring centralisé
    * Collecte de logs centralisée
    * Stockage et analyse des métriques
    * Dashboards Kibana
### Site 2 - Remote (Site Distant)
**Réseaux segmentés:**
* **LAN:** 192.168.110.0/24 (réseau utilisateurs)
* **DMZ:** 192.168.120.0/24 (services + bastion)
* **ADMIN:** 192.168.130.0/24 (management)
* **WAN:** Interface publique pour VPN
**VMs déployées (3 max):**
1. **pfSense-S2** (192.168.130.1) - Pare-feu/Router secondaire
    * Interface WAN: IP publique
    * Interface LAN: 192.168.110.1
    * Interface DMZ: 192.168.120.1
    * Interface ADMIN: 192.168.130.1
    * Client OpenVPN vers Site 1
2. **Bastion-Host** (192.168.120.10) - Accès sécurisé externe
    * SSH jump server
    * Authentification renforcée
    * Logging des connexions
    * Accès restreint depuis Internet
3. **WebServer-Internal** (192.168.110.10) - Site web interne
    * Application web accessible uniquement depuis le réseau interne
    * Logs envoyés vers Elasticsearch
### Interconnexion VPN Site-à-Site
**Configuration OpenVPN:**
* **Tunnel Principal:** 10.0.0.0/30
    * pfSense-S1: 10.0.0.1
    * pfSense-S2: 10.0.0.2
* **Chiffrement:** AES-256-GCM
* **Authentification:** RSA-4096 + certificats
* **Routes statiques:** Propagation automatique des réseaux
**Réseaux routés via VPN:**
* Site 1 → Site 2: 192.168.110.0/24, 192.168.120.0/24, 192.168.130.0/24
* Site 2 → Site 1: 192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24
### Sécurité et Pare-feu
**Règles pfSense principales:**
**Site 1 (pfSense-S1):**
* WAN → DMZ: HTTP/HTTPS vers NetBox et Elasticsearch (ports spécifiques)
* WAN → WAN: OpenVPN (port 1194/UDP)
* LAN → DMZ: Accès contrôlé aux services
* LAN → VPN: Accès Site 2 via tunnel
* ADMIN → ALL: Management complet
* Règle d'urgence: Kill switch pour couper VPN
**Site 2 (pfSense-S2):**
* WAN → DMZ: SSH vers Bastion uniquement (port 22/TCP)
* WAN → WAN: OpenVPN client vers Site 1
* LAN → VPN: Accès Site 1 via tunnel
* DMZ → LAN: Bastion vers LAN restreint
* ADMIN → ALL: Management local
* Règle d'urgence: Kill switch pour isoler le site
### DNS et Résolution de Noms
**Configuration DNS:**
* **Site 1:** pfSense-S1 comme DNS principal (192.168.10.1)
    * Zone interne: site1.local
    * Forwarder vers 8.8.8.8 pour Internet
    * Résolution croisée vers Site 2
* **Site 2:** pfSense-S2 comme DNS secondaire (192.168.110.1)
    * Zone interne: site2.local
    * Forwarder vers pfSense-S1 pour résolution complète
    * Cache local pour performance
**Forwarding entre sites:**
* Résolution site1.local ↔ site2.local via VPN
* Synchronisation des zones DNS
### IPAM et Automatisation
**NetBox (192.168.20.10):**
* **Préfixes gérés:**
    * Site 1: 192.168.10.0/24, 192.168.20.0/24, 192.168.30.0/24
    * Site 2: 192.168.110.0/24, 192.168.120.0/24, 192.168.130.0/24
    * VPN: 10.0.0.0/30
* **Automatisation:**
    * API REST pour allocation d'IPs
    * Synchronisation avec pfSense via scripts
    * Documentation automatique de la topologie
    * Webhooks pour mises à jour en temps réel
### Monitoring et Observabilité
**Elasticsearch (192.168.20.20):**
* **Sources de logs:**
    * pfSense-S1 et pfSense-S2: logs firewall, VPN
    * Bastion-Host: logs d'authentification SSH
    * WebServer-Internal: logs applicatifs
    * NetBox: logs d'API et modifications
* **Configuration Beats:**
    * Filebeat sur chaque VM
    * Metricbeat pour métriques système
    * Packetbeat pour analyse réseau
* **Dashboards Kibana:**
    * Vue d'ensemble infrastructure
    * Monitoring VPN et connectivité
    * Analyse des connexions bastion
    * Alerting sur anomalies
### Accès Externe et Bastion
**Bastion Host (192.168.120.10):**
* **Accès autorisés:**
    * SSH depuis Internet (port 22, IP whitelistées)
    * Jump vers LAN Site 2 (authentification double)
    * Accès Site 1 via VPN si nécessaire
* **Sécurisation:**
    * Authentification par clés SSH uniquement
    * 2FA avec Google Authenticator
    * Session recording et logging
    * Fail2ban contre brute force
    * Accès restreint par plages horaires
### Capacité d'Urgence (Kill Switch)
**Mécanismes de coupure:**
* **Site 1:** Règle pfSense pour bloquer tout trafic VPN
* **Site 2:** Isolation complète avec maintien accès bastion
* **Procédure de récupération:** Scripts automatisés de reconnexion
* **Monitoring:** Alertes automatiques en cas de déconnexion
### Scalabilité et Extension Future
**Convention d'adressage:**
* Site 1: 192.168.1X.0/24
* Site 2: 192.168.11X.0/24
* Site 3: 192.168.21X.0/24 (futur)
* VPN: 10.0.X.0/30 (X = numéro de tunnel)
**Templates réutilisables:**
* Configuration pfSense standardisée
* Déploiement VM automatisé via Terraform
* Règles firewall modulaires
* Intégration NetBox automatique
## Services Centralisés
* **IPAM:** NetBox sur Site 1 (source de vérité unique)
* **Monitoring:** Elasticsearch sur Site 1 (collecte multi-sites)
* **DNS:** Hiérarchie pfSense-S1 → pfSense-S2
* **VPN:** Hub-and-spoke avec Site 1 comme hub
