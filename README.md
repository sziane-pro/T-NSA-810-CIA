<div align="center">

# T-NSA-810-CIA — Infrastructure hybride multi-sites

**Infrastructure réseau hybride, sécurisée et entièrement pilotée par le code (IaC).**

Deux sites Proxmox interconnectés par un VPN site-à-site, pare-feu pfSense,
IPAM NetBox, observabilité OpenSearch, gestion des secrets HashiCorp Vault,
DNS interne, bastion d'administration et coupure d'urgence — le tout déployé
et reproductible via Ansible.

</div>

---

## Sommaire

- [Aperçu](#aperçu)
- [Points clés](#points-clés)
- [Architecture](#architecture)
- [Stack technique](#stack-technique)
- [Structure du dépôt](#structure-du-dépôt)
- [Prérequis](#prérequis)
- [Démarrage rapide](#démarrage-rapide)
- [Commandes principales](#commandes-principales)
- [Modèle de sécurité](#modèle-de-sécurité)
- [Documentation](#documentation)
- [Évolutivité multi-sites](#évolutivité-multi-sites)

---

## Aperçu

Le projet met en œuvre une infrastructure **hybride** composée de deux sites
Proxmox (un cœur de réseau et un site distant) reliés par un tunnel VPN chiffré.
Chaque site est protégé par un pare-feu **pfSense** assurant le routage, la
segmentation réseau (VLANs), le NAT, le DNS et la coupure d'urgence.

L'ensemble respecte des contraintes fortes :

- **maximum 3 VMs par site** ;
- uniquement des briques **open source, maintenues par la communauté** ;
- **tout en Infrastructure as Code** (Ansible) — reproductible depuis Git + Vault ;
- une architecture **évolutive** pour accueillir des sites supplémentaires.

## Points clés

| Domaine | Mise en œuvre |
|---|---|
| **Interconnexion** | VPN site-à-site OpenVPN (AES-256-GCM), PKI gérée dans Vault |
| **Pare-feu / routage** | pfSense par site, segmentation par VLANs, NAT, règles versionnées |
| **DNS** | Unbound (resolver pur + DNSSEC), zones internes `.lan`, RPZ anti-malware, **résolution croisée inter-sites** |
| **IPAM** | NetBox comme source de vérité, peuplé **par le code** + synchro automatique (timer systemd) |
| **Observabilité** | OpenSearch + OpenSearch Dashboards, collecte via Fluent Bit (les 2 sites) |
| **Secrets** | HashiCorp Vault (PKI, mots de passe applicatifs) + `ansible-vault` pour l'inventaire |
| **Accès admin** | Bastion (jump host) durci, SSH **key-only**, fail2ban, accès par IP whitelistées |
| **Site web interne** | Nginx, accessible **uniquement depuis le réseau interne** |
| **Coupure d'urgence** | Kill switch on-demand (isolation totale) + anti-fuite VPN automatique |

## Architecture

```
                            Internet
                               │
            ┌──────────────────┴───────────────────┐
            │                                       │
   ┌────────┴─────────┐   VPN site-à-site   ┌───────┴──────────┐
   │   SITE 1 (cœur)  │◄═══ 10.0.0.0/30 ═══►│  SITE 2 (distant)│
   │     pfSense      │   OpenVPN AES-256   │      pfSense      │
   ├──────────────────┤                     ├──────────────────┤
   │ LAN   192.168.10 │                     │ LAN   192.168.110│
   │ ADMIN 192.168.30 │                     │ DMZ   192.168.120│
   │                  │                     │ ADMIN 192.168.130│
   ├──────────────────┤                     ├──────────────────┤
   │ lan_vm  .10.101  │                     │ bastion   .120.10│
   │  • OpenSearch    │                     │  • SSH jump host │
   │  • NetBox (IPAM) │                     │ webserver .110.100│
   │ admin_vm .30.101 │                     │  • OS Dashboards │
   │  • Vault / PKI   │                     │  • site web interne│
   └──────────────────┘                     └──────────────────┘
```

| | Site 1 (cœur) | Site 2 (distant) |
|---|---|---|
| **Rôle** | Services centraux (Vault, NetBox, OpenSearch) | Services exposés (bastion, web, dashboards) |
| **VLANs** | LAN `10`, ADMIN `30` | LAN `110`, DMZ `120`, ADMIN `130` |
| **VMs** | pfSense, `lan_vm`, `admin_vm` | pfSense, `bastion`, `webserver` |
| **Accès SSH (NAT)** | `2221`→lan_vm, `2222`→admin_vm | `2231`→bastion, `2232`→webserver |

> Le détail des flux, des règles et des choix de conception est consigné dans les
> [ADR](docs/adr/) et la [documentation d'architecture](architecture-documentation.md).

## Stack technique

- **Virtualisation** : Proxmox VE
- **Pare-feu / routeur** : pfSense (FreeBSD)
- **VPN** : OpenVPN (site-à-site, p2p TLS)
- **IaC** : Ansible (+ collection `pfsensible.core`, `community.hashi_vault`)
- **Secrets** : HashiCorp Vault + `ansible-vault`
- **IPAM** : NetBox
- **Observabilité** : OpenSearch, OpenSearch Dashboards, Fluent Bit
- **DNS** : Unbound (intégré à pfSense) — DNSSEC + RPZ
- **Sécurité** : bastion SSH, fail2ban, durcissement SSH key-only

## Structure du dépôt

```
T-NSA-810-CIA/
├── infra-ansible/              # Tout le code Infrastructure as Code
│   ├── Makefile                # Point d'entrée : toutes les commandes de déploiement
│   ├── ansible.cfg
│   ├── requirements.yml/.txt   # Dépendances Ansible / Python
│   ├── inventories/            # Hôtes + variables par site (site1/, site2/)
│   ├── playbooks/              # Playbooks (pfsense, vpn, dns, vault, lan, bastion…)
│   └── roles/                  # Rôles réutilisables (dns, killswitch, fluentbit…)
├── docs/                       # Documentation projet
│   ├── INSTALL.md              # Guide d'installation pas à pas
│   ├── COMMANDS.md             # Annuaire complet des commandes
│   ├── onboarding.md           # Reprise de poste
│   ├── disaster-recovery-plan.md  # DRP & runbooks (kill switch, reconstruction)
│   ├── analyze-monitoring.md   # Analyse des logs / observabilité
│   ├── adr/                    # Architecture Decision Records
│   └── …
└── README.md
```

## Prérequis

- **Ansible** ≥ 2.14 et **Python** 3.10+
- Clé SSH d'automatisation `~/.ssh/ansible_key`
- Mot de passe `ansible-vault` dans `~/.vault_pass` (jamais commité)
- Fichier local `infra-ansible/.env` (voir `infra-ansible/.env.example`)

## Démarrage rapide

```bash
cd infra-ansible

# 1. Installer les dépendances (collections Ansible + libs Python)
make install

# 2. Configurer les variables locales
cp .env.example .env        # puis éditer .env

# 3. Vérifier la connectivité
make ping

# 4. Déployer
make deploy-all             # infrastructure complète (Site 1 puis Site 2)
#   ou étape par étape :
make site1-all              # tout le Site 1, dans l'ordre
make site2-all              # tout le Site 2
```

> Le déploiement détaillé, étape par étape (avec ce que fait chaque commande et
> comment la vérifier), est documenté dans **[docs/INSTALL.md](docs/INSTALL.md)**.

## Commandes principales

```bash
make help                   # liste toutes les cibles disponibles

# Déploiement
make deploy-all             # Site 1 + Site 2
make site1-all / site2-all  # un site complet

# Accès aux services internes (tunnels SSH via le bastion / admin_vm)
make tunnel-vault           # https://localhost:8200
make tunnel-netbox          # https://localhost:8443
make tunnel-opensearch      # https://localhost:9200

# Coupure d'urgence
make site1-killswitch       # isole totalement le Site 1
make site1-killswitch-restore
```

> Référence exhaustive (DNS, Vault, logs, dépannage…) : **[docs/COMMANDS.md](docs/COMMANDS.md)**.

## Modèle de sécurité

- **Aucun service exposé directement sur Internet** : Vault, NetBox, OpenSearch et
  le site web interne ne sont joignables que via le réseau interne ou un **tunnel SSH**.
- **Secrets centralisés** : la PKI et les mots de passe applicatifs vivent dans
  **Vault** ; les secrets d'inventaire sont chiffrés avec **ansible-vault**. Aucun
  secret en clair dans Git (`.env`, `pki/`, sauvegardes Vault sont ignorés).
- **Accès administrateur contrôlé** : bastion durci, SSH **par clé uniquement**,
  `fail2ban`, et NAT/règles restreints à des **IP sources whitelistées**.
- **Coupure d'urgence (kill switch)** :
  - **anti-fuite VPN** *(automatique)* — si le tunnel tombe, le trafic inter-sites
    est bloqué au lieu de fuiter en clair par le WAN ;
  - **isolation totale** *(à la demande)* — `make siteX-killswitch`, avec reprise
    garantie hors-bande (console Proxmox + bastion). Voir le
    [plan de reprise](docs/disaster-recovery-plan.md).

## Documentation

| Document | Contenu |
|---|---|
| [docs/INSTALL.md](docs/INSTALL.md) | Installation pas à pas (chaque étape expliquée) |
| [docs/COMMANDS.md](docs/COMMANDS.md) | Annuaire complet des commandes + dépannage |
| [docs/onboarding.md](docs/onboarding.md) | Reprise de poste sur une nouvelle machine |
| [docs/disaster-recovery-plan.md](docs/disaster-recovery-plan.md) | DRP, kill switch, runbooks d'incident |
| [docs/analyze-monitoring.md](docs/analyze-monitoring.md) | Observabilité & analyse des logs |
| [infra-ansible/DOCUMENTATION.md](infra-ansible/DOCUMENTATION.md) | Référence Ansible (structure, variables) |
| [infra-ansible/SERVICES.md](infra-ansible/SERVICES.md) | Catalogue des services et accès |
| [docs/adr/](docs/adr/) | Décisions d'architecture (ADR) |

## Évolutivité multi-sites

L'architecture est conçue pour accueillir de nouveaux sites sans refonte :
convention d'adressage normalisée, rôles Ansible paramétrés par site et
description IPAM data-driven. La procédure d'ajout d'un site est décrite dans
[docs/multi-site_readiness/](docs/multi-site_readiness/).

---

<div align="center">
<sub>Projet académique Epitech — T-NSA-810-CIA · Infrastructure as Code (Ansible)</sub>
</div>
