# Documentation Infrastructure NSA-810-CIA

## Structure du projet

```
infa-ansible/
├── ansible.cfg                          # Configuration Ansible
├── Makefile                             # Commandes simplifiées
├── inventories/
│   ├── inventory                        # Hôtes et variables de connexion
│   ├── group_vars/
│   │   └── firewall.yml                 # Variables pfSense
│   └── host_vars/
│       ├── lan_vm.yml                   # Variables OpenSearch + Netbox
│       └── admin_vm.yml                 # Variables Vault
├── playbooks/
│   ├── pfsense_network.yml              # Playbook pfSense
│   ├── vault.yml                        # Playbook Vault
│   └── lan.yml                          # Playbook OpenSearch + Netbox
└── roles/
    ├── pfsense_network/
    │   └── tasks/
    │       ├── main.yml                 # Orchestre vlans + interfaces + rules
    │       ├── vlans.yml                # Création des VLANs
    │       ├── interfaces.yml           # Configuration des interfaces
    │       └── rules.yml                # Règles firewall WAN
    ├── vault/
    │   ├── tasks/
    │   │   ├── main.yml                 # Installation Vault
    │   │   └── init.yml                 # Initialisation + unseal + KV engine
    │   ├── templates/
    │   │   └── vault.hcl.j2             # Configuration Vault
    │   └── handlers/
    │       └── main.yml                 # Restart Vault
    ├── opensearch/
    │   ├── tasks/main.yml               # Installation OpenSearch
    │   ├── templates/opensearch.yml.j2  # Configuration OpenSearch
    │   └── handlers/main.yml            # Restart OpenSearch
    └── netbox/
        ├── tasks/main.yml               # Installation Netbox
        ├── templates/
        │   ├── configuration.py.j2      # Configuration Netbox
        │   └── netbox.service.j2        # Service systemd Netbox
        └── handlers/main.yml            # Restart Netbox
```

---

## Architecture réseau

```
Internet (WAN) — IP publique: 5.196.51.228
│
pfSense (firewall/routeur)
│
├── VLAN 10 — LAN (192.168.10.0/24)
│   └── lan_vm (192.168.10.101)
│       ├── OpenSearch  :9200
│       └── Netbox      :8080
│
└── VLAN 30 — ADMIN (192.168.30.0/24)
    └── admin_vm (192.168.30.101)
        └── Vault       :8200
```

### Accès SSH depuis l'extérieur (NAT pfSense)
| Port WAN | Destination interne | Machine |
|----------|--------------------|---------||
| 2221     | 192.168.10.101:22  | lan_vm  |
| 2222     | 192.168.30.101:22  | admin_vm|
| 22       | pfSense lui-même   | firewall|

---

## Commandes Makefile

```bash
make pfsense         # Configure VLANs + interfaces + règles firewall pfSense
make pfsense-rules   # Rejoue uniquement les règles firewall (sans toucher aux interfaces)
make vault           # Installe et initialise Vault sur admin_vm
make lan             # Installe OpenSearch + Netbox sur lan_vm
make ping            # Teste la connectivité vers tous les hôtes
```

---

## Connexions SSH

```bash
# pfSense
ssh -p 22 -i ~/.ssh/ansible_key admin@5.196.51.228

# lan_vm
ssh -p 2221 -i ~/.ssh/ansible_key sysadmin@5.196.51.228

# admin_vm
ssh -p 2222 -i ~/.ssh/ansible_key sysadmin@5.196.51.228

# Tunnel SSH pour accéder à Vault depuis le navigateur
ssh -p 2222 -i ~/.ssh/ansible_key -L 8200:192.168.30.101:8200 sysadmin@5.196.51.228
# Puis ouvrir http://localhost:8200
```

---

## Tags Ansible

Les tags permettent de lancer uniquement une partie du playbook pfSense :

```bash
# Uniquement les règles firewall
ansible-playbook -i inventories/inventory playbooks/pfsense_network.yml --tags rules

# Uniquement les VLANs
ansible-playbook -i inventories/inventory playbooks/pfsense_network.yml --tags vlans

# Tout sauf les interfaces (pour ne pas casser les VMs)
ansible-playbook -i inventories/inventory playbooks/pfsense_network.yml --skip-tags interfaces
```

---

## pfSense — Configuration réseau

### VLANs (group_vars/firewall.yml)
| Variable          | Valeur    | Description              |
|-------------------|-----------|--------------------------|
| wan_interface     | vtnet0    | Interface WAN            |
| trunk_interface   | vtnet1    | Interface trunk (VLANs)  |
| Lan_vlan          | 10        | ID VLAN LAN              |
| admin_vlan        | 30        | ID VLAN ADMIN            |
| lan_ip            | 192.168.10.1 | Gateway VLAN LAN      |
| admin_ip          | 192.168.30.1 | Gateway VLAN ADMIN    |

### Règles firewall WAN
| Source          | Destination         | Port | Description             |
|-----------------|---------------------|------|-------------------------|
| 163.5.3.84      | This Firewall       | 443  | HTTPS firewall          |
| 163.5.3.84      | This Firewall       | 22   | SSH firewall            |
| 163.5.3.84      | 192.168.10.100      | 22   | Pc Jojo => Vm1          |
| *               | 192.168.10.101      | 22   | NAT LAN Vm1             |
| 163.5.3.84      | 192.168.30.101      | 22   | Reseau Epitech => Vm1   |
| *               | 192.168.30.100      | 22   | NAT ADMIN Vm2           |
| 94.238.234.44   | This Firewall       | 443  | Chez Jaures HTTPS       |
| 94.238.234.44   | This Firewall       | 22   | Chez Jaures SSH         |
| 94.238.234.44   | 192.168.10.101      | 22   | Réseau Jaures => Vm1    |
| 94.238.234.44   | 192.168.30.100      | 22   | Réseau Jaures => Vm2    |

---

## Vault

### Initialisation automatique (via Ansible)
- Clés unseal et root token sauvegardés dans `/etc/vault.d/init.json` sur admin_vm
- Moteur KV activé sur `secret/`

### Récupérer le root token
```bash
ssh -p 2222 -i ~/.ssh/ansible_key sysadmin@5.196.51.228
sudo cat /etc/vault.d/init.json
```

### Stocker des secrets (une seule fois après installation)
```bash
export VAULT_ADDR="http://192.168.30.101:8200"
export VAULT_TOKEN="hvs.XXXXXXXXXXXXXXXX"

vault kv put secret/vms/sysadmin become_password="tonMotDePasse"
vault kv put secret/pfsense password="tonMotDePassePfsense"
vault kv put secret/ansible ssh_key="$(cat ~/.ssh/ansible_key)"
```

### Récupérer un secret dans un playbook Ansible
```yaml
- name: Get password from Vault
  community.hashi_vault.vault_kv2_get:
    url: "http://192.168.30.101:8200"
    path: vms/sysadmin
    token: "{{ vault_token }}"
  register: vm_secret
```

---

## Netplan — Configuration IP statique des VMs

### lan_vm (/etc/netplan/50-cloud-init.yaml)
```yaml
network:
  version: 2
  ethernets:
    enp6s18: {}
  vlans:
    enp6s18.10:
      id: 10
      link: "enp6s18"
      addresses:
        - 192.168.10.101/24
      routes:
        - to: default
          via: 192.168.10.1
      nameservers:
        addresses:
          - 8.8.8.8
```

### admin_vm (/etc/netplan/50-cloud-init.yaml)
```yaml
network:
  version: 2
  ethernets:
    enp6s18: {}
  vlans:
    enp6s18.30:
      id: 30
      link: "enp6s18"
      addresses:
        - 192.168.30.101/24
      routes:
        - to: default
          via: 192.168.30.1
      nameservers:
        addresses:
          - 8.8.8.8
```

```bash
sudo netplan apply   # Appliquer la configuration
sudo netplan try     # Tester (revient en arrière automatiquement si pas confirmé)
```

---

## Concepts réseau

### DMZ
Zone réseau entre internet et le LAN interne. Accueille les services accessibles depuis internet.
- Internet → DMZ : autorisé
- DMZ → LAN : bloqué
- LAN → DMZ : autorisé

Dans ce projet, pas de DMZ — tous les services sont internes et accessibles uniquement via tunnel SSH.

### NAT Outbound
Permet aux VMs du réseau interne d'accéder à internet. Configuré dans pfSense :
**Firewall → NAT → Outbound → Hybrid**
- Source 192.168.10.0/24 → WAN address
- Source 192.168.30.0/24 → WAN address

### Idempotence Ansible
Les modules Ansible sont idempotents — tu peux relancer un playbook plusieurs fois sans risque :
- Config identique → `ok` (rien ne change)
- Config différente → `changed` (mise à jour)
- Config absente → `changed` (création)

---

## Problèmes rencontrés et solutions

| Problème | Cause | Solution |
|----------|-------|----------|
| Role not found | ansible.cfg non lu (WSL world-writable) | Utiliser `ANSIBLE_CONFIG` via Makefile |
| roles_path incorrect | Chemin `../roles` au lieu de `roles` | Corriger dans ansible.cfg |
| Variable undefined | group_vars au lieu de host_vars pour un host | Déplacer dans host_vars/ |
| Python not found sur pfSense | pfSense = FreeBSD, pas de Python | Retirer ansible_python_interpreter |
| Interfaces pfSense non actives | Paramètre `enable: true` manquant | Ajouter dans pfsense_interface |
| DHCP perdu après Ansible | Interfaces reconfigurées par Ansible | Passer en IP statique via netplan |
| Incorrect sudo password | Caractères spéciaux mal interprétés | Utiliser NOPASSWD ou mot de passe simple |
