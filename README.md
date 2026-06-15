# T-NSA-810-CIA — Infrastructure Hybride Deux Sites

Infrastructure réseau hybride sécurisée composée de deux sites Proxmox interconnectés par VPN site-à-site, avec pare-feu pfSense, IPAM NetBox, monitoring OpenSearch et gestion des secrets HashiCorp Vault. Tout le déploiement est automatisé via Ansible.

## Prerequis

- Ansible >= 2.14
- Python 3.10+
- Accès SSH au pfSense (port 2222, clé `~/.ssh/ansible_key`)
- Fichier `infra-ansible/.env` configuré (voir `infra-ansible/.env.example`)

## Demarrage rapide

```bash
cd infra-ansible

# 1. Installer les dependances
make install

# 2. Configurer les variables locales
cp .env.example .env
# Editer .env avec les vraies valeurs

# 3. Deployer l'infrastructure
make site1-pfsense   # pfSense : VLANs, interfaces, regles firewall
make site1-vault     # HashiCorp Vault sur admin_vm
make site1-lan       # OpenSearch + NetBox sur lan_vm
make site1-vpn       # PKI + OpenVPN site-a-site
```

## Documentation

| Fichier                                                        | Contenu                                             |
|----------------------------------------------------------------|-----------------------------------------------------|
| [Architecture-Documentation.md](architecture-documentation.md) | Architecture reseau detaillee des deux sites        |
| [infa-ansible/DOCUMENTATION.md](infra-ansible/DOCUMENTATION.md) | Reference Ansible : structure, variables, commandes |
| [infa-ansible/SERVICES.md](infra-ansible/SERVICES.md)           | Catalogue des services deployes et acces            |
| [docs/onboarding.md](docs/onboarding.md)                       | Guide de prise de poste pas a pas                   |

## Structure du depot

```
T-NSA-810-CIA/
├── infra-ansible/          # Tout le code Ansible
│   ├── Makefile           # Commandes simplifiees
│   ├── .env.example       # Template de configuration locale
│   ├── inventories/       # Hotes et variables
│   ├── playbooks/         # Playbooks principaux
│   └── roles/             # Roles Ansible
├── docs/                  # Documentation
└── README.md
```
