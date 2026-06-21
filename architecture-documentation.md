# Architecture réseau — Infrastructure hybride Proxmox

> Document de référence décrivant l'infrastructure **réellement déployée** (IaC Ansible).
> Pour les décisions de conception, voir les [ADR](docs/adr/). Pour l'installation,
> voir [docs/INSTALL.md](docs/INSTALL.md).

## 1. Contexte et objectifs

Infrastructure **hybride sécurisée** composée de deux sites Proxmox interconnectés
par un VPN site-à-site, avec pare-feu, IPAM automatisé, observabilité centralisée,
gestion des secrets et capacité d'extension à de nouveaux sites.

**Contraintes :**

- maximum **3 VMs par site** ;
- uniquement des technologies **open source, maintenues par la communauté** ;
- **tout en Infrastructure as Code** (Ansible), reproductible depuis Git + Vault ;
- architecture **évolutive** (ajout de sites sans refonte).

## 2. Vue d'ensemble

```
                              Internet
                                 │
              ┌──────────────────┴───────────────────┐
              │                                       │
     ┌────────┴─────────┐   VPN site-à-site   ┌───────┴──────────┐
     │   SITE 1 (cœur)  │◄═══ 10.0.0.0/30 ═══►│  SITE 2 (distant)│
     │  pfSense (FW/VPN)│   OpenVPN AES-256   │  pfSense (FW/VPN)│
     └────────┬─────────┘                     └────────┬─────────┘
        ┌─────┴─────┐                            ┌──────┴───────┐
   VLAN 10 LAN   VLAN 30 ADMIN          VLAN 110 LAN  120 DMZ  130 ADMIN
        │             │                       │          │
   ┌────┴───┐   ┌─────┴────┐            ┌─────┴────┐ ┌───┴─────┐
   │ lan_vm │   │ admin_vm │            │webserver │ │ bastion │
   │OpenSearch│ │  Vault   │            │Dashboards│ │SSH jump │
   │ NetBox │   │   PKI    │            │web interne│ └─────────┘
   └────────┘   └──────────┘            └──────────┘
```

- **Site 1** héberge les **services centraux** : Vault (secrets/PKI), NetBox (IPAM),
  OpenSearch (observabilité). C'est le **hub** du VPN (serveur OpenVPN).
- **Site 2** héberge les **services exposés/d'accès** : bastion d'administration,
  OpenSearch Dashboards et le site web interne. C'est le **client** du VPN.

## 3. Plan d'adressage

### Site 1 — cœur (on-premise)

| Segment | VLAN | Plage | Passerelle | Usage |
|---|---|---|---|---|
| LAN | 10 | `192.168.10.0/24` | `192.168.10.1` | Services internes (lan_vm) |
| ADMIN | 30 | `192.168.30.0/24` | `192.168.30.1` | Management (admin_vm) |

### Site 2 — distant (remote)

| Segment | VLAN | Plage | Passerelle | Usage |
|---|---|---|---|---|
| LAN | 110 | `192.168.110.0/24` | `192.168.110.1` | Serveur web interne |
| DMZ | 120 | `192.168.120.0/24` | `192.168.120.1` | Bastion (accès externe) |
| ADMIN | 130 | `192.168.130.0/24` | `192.168.130.1` | Management |

### Tunnel VPN

| | Adresse |
|---|---|
| Réseau de transit | `10.0.0.0/30` |
| pfSense Site 1 | `10.0.0.1` |
| pfSense Site 2 | `10.0.0.2` |

## 4. Inventaire des VMs (3 max / site)

| Site | VM | IP | Rôle |
|---|---|---|---|
| 1 | **pfSense-S1** | LAN `.10.1` / ADMIN `.30.1` / WAN public | Pare-feu, routeur, **serveur OpenVPN**, DNS |
| 1 | **lan_vm** | `192.168.10.101` | **OpenSearch** (+ Dashboards backend) et **NetBox** (IPAM) |
| 1 | **admin_vm** | `192.168.30.101` | **HashiCorp Vault** (secrets + PKI), passerelle des tunnels d'admin |
| 2 | **pfSense-S2** | LAN `.110.1` / DMZ `.120.1` / ADMIN `.130.1` / WAN public | Pare-feu, routeur, **client OpenVPN**, DNS |
| 2 | **bastion** | `192.168.120.10` | Jump host SSH durci (accès admin distant) |
| 2 | **webserver** | `192.168.110.100` | **OpenSearch Dashboards** + **site web interne** (Nginx) |

## 5. Interconnexion VPN site-à-site

- **Technologie :** OpenVPN en mode `p2p_tls` (Site 1 = serveur, Site 2 = client).
- **Transport :** UDP, port `1194` sur le WAN.
- **Chiffrement :** `AES-256-GCM` (data ciphers + fallback alignés des deux côtés).
- **Authentification :** PKI X.509 (CA + certificats serveur/client) **générée et
  stockée dans Vault**, importée dans pfSense au déploiement.
- **Routage :** chaque pfSense annonce ses réseaux internes via le tunnel
  (`remote_network`), permettant la communication inter-LAN bidirectionnelle.

| Sens | Réseaux routés |
|---|---|
| Site 1 → Site 2 | `192.168.110.0/24`, `192.168.120.0/24`, `192.168.130.0/24` |
| Site 2 → Site 1 | `192.168.10.0/24`, `192.168.30.0/24` |

## 6. Pare-feu et flux

Chaque pfSense applique un filtrage **least privilege**, versionné en Ansible.

**NAT (port-forward WAN → VMs internes), restreint à des IP sources whitelistées :**

| Site | Port WAN | Destination |
|---|---|---|
| 1 | `2221` | `lan_vm:22` |
| 1 | `2222` | `admin_vm:22` |
| 2 | `2231` | `bastion:22` |
| 2 | `2232` | `webserver:22` |

**Flux principaux autorisés :**

- WAN → pfSense : OpenVPN (`UDP/1194`), SSH d'admin depuis IP whitelistées.
- VLANs internes → pfSense : DNS (`53`).
- Tunnel VPN : trafic inter-sites bidirectionnel + DNS croisé.
- Le **site web interne n'est jamais exposé sur le WAN** (accès via VPN / bastion).

## 7. DNS

DNS assuré par **Unbound** (intégré à pfSense), configuré par le rôle Ansible `dns` :

- **Resolver pur** (récursif depuis la racine) + **DNSSEC**.
- **Zones internes** : `site1.lan` et `site2.lan` (enregistrements A + **PTR**).
- **RPZ** (Response Policy Zone) : blocage des domaines malveillants (feed HaGeZi).
- **Résolution croisée inter-sites** via le VPN : `site1.lan ↔ site2.lan`
  (domain overrides + flux DNS autorisé sur le tunnel).

| Site | Zone | Exemples d'enregistrements |
|---|---|---|
| 1 | `site1.lan` | `netbox`, `opensearch`, `vault`, `fw` |
| 2 | `site2.lan` | `bastion`, `webserver`, `dashboards`, `fw` |

## 8. IPAM — NetBox

- **NetBox** (sur `lan_vm`, Site 1) est la **source de vérité** réseau (sites,
  préfixes, VLANs, VMs, interfaces, IP, services, tunnels).
- Peuplé **par le code** (rôle `netbox_sync`, description déclarative des **deux sites**).
- **Maintenu à jour automatiquement** par un **timer systemd** (synchro quotidienne).

## 9. Observabilité — OpenSearch

- **OpenSearch** + **OpenSearch Dashboards** pour la centralisation et l'analyse
  des logs (choix justifié face à Elastic dans [ADR-007](docs/adr/ADR-007-opensearch-observabilite.md)).
- Collecte via **Fluent Bit** sur les VMs des **deux sites** ; les logs du Site 2
  remontent vers OpenSearch (Site 1) **à travers le VPN**.
- Sources : `auth.log`/`syslog` (SSH, sudo, fail2ban — structurés par parsers) et
  les logs d'accès **Nginx** (JSON). Champ `source_host` ajouté à chaque événement.

## 10. Bastion et accès administrateur

- **Bastion** (Site 2, DMZ) : jump host SSH **durci** (key-only, `PermitRootLogin no`),
  **fail2ban** anti-brute-force, accès restreint aux **IP whitelistées**, logging.
- Accès aux services internes (Vault, NetBox, OpenSearch) **uniquement via tunnels
  SSH** — aucun de ces services n'est exposé directement sur Internet.

## 11. Gestion des secrets

- **HashiCorp Vault** (sur `admin_vm`) : PKI VPN, mots de passe applicatifs
  (NetBox, OpenSearch). Démarre **scellé** → unseal requis (clés hors dépôt).
- **ansible-vault** : chiffrement des secrets d'inventaire (`group_vars/*/vault.yml`),
  mot de passe dans `~/.vault_pass` (jamais commité).
- Aucun secret en clair dans Git (`.env`, `pki/`, sauvegardes Vault sont ignorés).
  Détails : [ADR-010](docs/adr/ADR-010-gestion-secrets.md).

## 12. Coupure d'urgence (kill switch)

Deux mécanismes complémentaires (cf. [ADR-009](docs/adr/ADR-009-kill-switch-restauration.md)
et le [plan de reprise](docs/disaster-recovery-plan.md)) :

| Mécanisme | Déclenchement | Effet |
|---|---|---|
| **Anti-fuite VPN** | Automatique, permanent | Règle flottante `out` + `quick` : si le tunnel tombe, le trafic inter-sites est **droppé** au lieu de fuiter en clair par le WAN. Présent sur **les deux sites**. |
| **Isolation totale** | À la demande (`make siteX-killswitch`) | Règles flottantes `quick` (pré-créées, désactivées) qui **coupent tout le WAN** (entrant/sortant) + VPN. Reprise hors-bande garantie : **console Proxmox** (S1 & S2) + **bastion** (S2). |

## 13. Évolutivité multi-sites

- **Convention d'adressage** normalisée et extensible (un nouveau site = un nouveau
  triplet de VLANs + un tunnel `10.0.0.x/30`).
- **Rôles Ansible paramétrés par site** (group_vars), description **IPAM data-driven**.
- Procédure d'onboarding d'un 3ᵉ site : [docs/multi-site_readiness/](docs/multi-site_readiness/).

## 14. Références

- Décisions d'architecture : [docs/adr/](docs/adr/)
- Installation pas à pas : [docs/INSTALL.md](docs/INSTALL.md)
- Annuaire des commandes : [docs/COMMANDS.md](docs/COMMANDS.md)
- Plan de reprise (DRP) : [docs/disaster-recovery-plan.md](docs/disaster-recovery-plan.md)
