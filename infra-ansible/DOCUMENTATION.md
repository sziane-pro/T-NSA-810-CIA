# Documentation Ansible — Infrastructure NSA-810-CIA

## Structure du projet

```
infra-ansible/
├── ansible.cfg                          # Configuration Ansible
├── Makefile                             # Commandes simplifiees
├── requirements.yml                     # Collections Ansible
├── requirements.txt                     # Dependances Python
├── .env.example                         # Template de configuration locale
├── inventories/
│   ├── inventory                        # Hotes et variables de connexion
│   ├── group_vars/
│   │   ├── firewall.yml                 # Variables pfSense (IPs, VLANs, VPN)
│   │   └── admin_vm.yml                 # Variables admin_vm
│   └── host_vars/
│       ├── lan_vm.yml                   # Variables OpenSearch + NetBox (Vault lookups)
│       └── admin_vm.yml                 # Variables Vault
├── playbooks/
│   ├── pfsense_network.yml              # Playbook pfSense
│   ├── vault.yml                        # Playbook Vault
│   ├── lan.yml                          # Playbook OpenSearch + NetBox
│   └── vpn.yml                          # Playbook PKI + OpenVPN
└── roles/
    ├── pfsense_network/
    │   └── tasks/
    │       ├── main.yml                 # Orchestre vlans + interfaces + rules + vpn
    │       ├── vlans.yml                # Creation des VLANs
    │       ├── interfaces.yml           # Configuration des interfaces
    │       ├── rules.yml                # Regles firewall WAN + inter-VLAN
    │       └── vpn.yml                  # Import PKI + configuration OpenVPN + kill switch
    ├── pki/
    │   └── tasks/main.yml               # Generation PKI EasyRSA (CA, certs, DH)
    ├── vault/
    │   ├── tasks/
    │   │   ├── main.yml                 # Installation + TLS + configuration Vault
    │   │   └── init.yml                 # Initialisation + unseal + moteur KV
    │   ├── templates/
    │   │   └── vault.hcl.j2             # Configuration Vault (HTTPS)
    │   └── handlers/main.yml            # Restart Vault
    ├── opensearch/
    │   ├── tasks/main.yml               # Installation OpenSearch
    │   ├── templates/opensearch.yml.j2  # Configuration OpenSearch
    │   └── handlers/main.yml            # Restart OpenSearch
    └── netbox/
        ├── tasks/main.yml               # Installation NetBox + Nginx + Gunicorn
        ├── templates/
        │   ├── configuration.py.j2      # Configuration Django NetBox
        │   ├── nginx.conf.j2            # Reverse proxy HTTPS Nginx
        │   └── netbox.service.j2        # Service systemd Gunicorn
        └── handlers/main.yml            # Restart NetBox + Nginx
```

---

## Architecture reseau

```
Internet (WAN) — IP publique : 5.196.51.228
│
pfSense (firewall/routeur/gateway)
├── VLAN 10 — LAN (192.168.10.0/24)
│   └── lan_vm (192.168.10.101)
│       ├── OpenSearch   :9200  (HTTPS)
│       └── NetBox       :443   (HTTPS via Nginx)
│
└── VLAN 30 — ADMIN (192.168.30.0/24)
    └── admin_vm (192.168.30.101)
        └── Vault        :8200  (HTTPS)
```

### Acces SSH depuis l'exterieur (NAT pfSense)

| Port WAN | Destination interne    | Machine   |
|----------|------------------------|-----------|
| 2221     | 192.168.10.101:22      | lan_vm    |
| 2222     | 192.168.30.101:22      | admin_vm  |
| 22       | pfSense lui-meme       | firewall  |

---

## Configuration locale requise

Copier `.env.example` en `.env` et renseigner les valeurs :

```bash
cp .env.example .env
```

| Variable          | Description                                    |
|-------------------|------------------------------------------------|
| VAULT_TOKEN       | Root token Vault (dans /etc/vault.d/init.json) |
| VAULT_ADDR        | https://127.0.0.1:8200 (avec tunnel SSH actif) |
| VAULT_SKIP_VERIFY | true (certificat auto-signe)                   |
| GATEWAY_USER      | Utilisateur SSH du pfSense                     |
| GATEWAY_HOST      | IP publique du pfSense                         |
| GATEWAY_PORT      | Port SSH du pfSense                            |
| SSH_KEY           | Chemin vers la cle privee SSH                  |

---

## Commandes Makefile

```bash
make install        # Installe collections Ansible + dependances Python
make pfsense        # Configure VLANs + interfaces + regles firewall pfSense
make pfsense-rules  # Rejoue uniquement les regles firewall
make pfsense-vpn    # Rejoue uniquement la configuration VPN sur pfSense
make vault          # Installe et configure Vault sur admin_vm
make lan            # Installe OpenSearch + NetBox sur lan_vm (ouvre tunnel Vault auto)
make vpn            # Genere la PKI + configure OpenVPN sur pfSense
make pki            # Genere uniquement la PKI (sur admin_vm)
make ping           # Teste la connectivite vers tous les hotes
```

Le tunnel SSH vers Vault est ouvert et ferme automatiquement par `make lan`.

---

## Connexions SSH

```bash
# pfSense
ssh -p 22 -i ~/.ssh/ansible_key admin@5.196.51.228

# lan_vm
ssh -p 2221 -i ~/.ssh/ansible_key sysadmin@5.196.51.228

# admin_vm
ssh -p 2222 -i ~/.ssh/ansible_key sysadmin@5.196.51.228
```

### Tunnels SSH vers les services internes

```bash
# Vault
ssh -p 2222 -i ~/.ssh/ansible_key -L 8200:192.168.30.101:8200 sysadmin@5.196.51.228 -N
# Puis ouvrir https://localhost:8200

# NetBox
ssh -p 2221 -i ~/.ssh/ansible_key -L 8443:192.168.10.101:443 sysadmin@5.196.51.228 -N
# Puis ouvrir https://localhost:8443

# OpenSearch
ssh -p 2221 -i ~/.ssh/ansible_key -L 9200:192.168.10.101:9200 sysadmin@5.196.51.228 -N
# Puis ouvrir https://localhost:9200
```

---

## Gestion des secrets (HashiCorp Vault)

Les credentials des services sont stockes dans Vault, jamais en clair dans le code.

### Secrets stockes

```
secret/netbox
  db_password              Mot de passe PostgreSQL NetBox
  admin_password           Mot de passe admin interface NetBox
  secret_key               Cle secrete Django NetBox
  opensearch_admin_password Mot de passe admin OpenSearch
```

### Ajouter ou mettre a jour un secret

```bash
# Sur admin_vm
export VAULT_ADDR='https://192.168.30.101:8200'
export VAULT_SKIP_VERIFY=true
export VAULT_TOKEN=$(sudo cat /etc/vault.d/init.json | python3 -c "import sys,json; print(json.load(sys.stdin)['root_token'])")

# Creer ou ecraser tous les champs
vault kv put secret/netbox \
  db_password="..." \
  admin_password="..." \
  secret_key="..." \
  opensearch_admin_password="..."

# Ajouter ou modifier un seul champ sans toucher aux autres
vault kv patch secret/netbox db_password="nouveau_mdp"

# Lister les secrets stockes
vault kv get secret/netbox
```

### Vault sealed apres redemarrage

Vault se verrouille a chaque redemarrage de la VM. Pour le deverrouiller :

```bash
export VAULT_ADDR='https://192.168.30.101:8200'
export VAULT_SKIP_VERIFY=true
sudo cat /etc/vault.d/init.json   # recuperer unseal_keys_b64[0]
vault operator unseal             # coller la cle
```

### Sauvegarde et restauration Vault

```bash
# Creer une archive des donnees Vault (sur admin_vm)
sudo tar czf /home/sysadmin/vault-backup.tar.gz /opt/vault/data
sudo chown sysadmin:sysadmin /home/sysadmin/vault-backup.tar.gz

# Telecharger en local
scp -P 2222 -i ~/.ssh/ansible_key sysadmin@5.196.51.228:/home/sysadmin/vault-backup.tar.gz ~/vault-backup.tar.gz

# Restaurer (sur admin_vm)
sudo systemctl stop vault
sudo rm -rf /opt/vault/data
sudo tar xzf /home/sysadmin/vault-backup.tar.gz -C /
sudo systemctl start vault
vault operator unseal
```

---

## PKI et VPN site-a-site

### Architecture

- Site 1 = serveur OpenVPN (hub)
- Site 2 = client OpenVPN
- Tunnel : 10.0.0.0/30
- Chiffrement : AES-256-GCM

### Fichiers PKI generes (stockes localement, jamais dans git)

```
infra-ansible/pki/
├── ca.crt                 Certificat de l'autorite de certification
├── vpn-server.crt         Certificat serveur Site 1
├── vpn-server.key         Cle privee serveur (supprimee de la VM apres fetch)
├── vpn-client-site2.crt   Certificat client Site 2
├── vpn-client-site2.key   Cle privee client Site 2
└── dh.pem                 Parametres Diffie-Hellman
```

### Kill switch

Des regles de blocage sur le WAN empechent le trafic vers Site 2 de fuiter en clair si le tunnel tombe.

---

## Tags Ansible

```bash
# Uniquement les regles firewall
ansible-playbook -i inventories/inventory playbooks/pfsense_network.yml --tags rules

# Uniquement les VLANs
ansible-playbook -i inventories/inventory playbooks/pfsense_network.yml --tags vlans

# Uniquement le VPN
ansible-playbook -i inventories/inventory playbooks/pfsense_network.yml --tags vpn

# Uniquement la PKI
ansible-playbook -i inventories/inventory playbooks/vpn.yml --tags pki
```

---

## Variables pfSense (group_vars/firewall.yml)

| Variable             | Valeur              | Description                   |
|----------------------|---------------------|-------------------------------|
| wan_interface        | vtnet0              | Interface WAN                 |
| trunk_interface      | vtnet1              | Interface trunk VLANs         |
| Lan_vlan             | 10                  | ID VLAN LAN                   |
| admin_vlan           | 30                  | ID VLAN ADMIN                 |
| lan_ip               | 192.168.10.1        | Gateway VLAN LAN              |
| admin_ip             | 192.168.30.1        | Gateway VLAN ADMIN            |
| lan_vm_ip            | 192.168.10.101      | IP lan_vm                     |
| admin_vm_ip          | 192.168.30.101      | IP admin_vm                   |
| epitech_network_ip   | 163.5.3.84          | IP reseau Epitech (autorisee) |
| jaures_network_ip    | 94.238.238.44       | IP reseau personnel           |
| vpn_tunnel_network   | 10.0.0.0/30         | Reseau tunnel VPN             |
| vpn_site2_lan        | 192.168.110.0/24    | LAN Site 2                    |
| vpn_site2_admin      | 192.168.130.0/24    | ADMIN Site 2                  |

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
      link: enp6s18
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
      link: enp6s18
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
sudo netplan try     # Tester avec retour automatique si non confirme
```

---

## Problemes connus et solutions

| Probleme | Cause | Solution |
|----------|-------|----------|
| Role not found | ansible.cfg non lu (WSL world-writable) | ANSIBLE_CONFIG force via Makefile |
| Python not found sur pfSense | pfSense = FreeBSD | ansible_python_interpreter: /usr/local/bin/python3.11 |
| Vault connection refused | VAULT_ADDR pointe sur localhost sans tunnel | Ouvrir tunnel SSH ou utiliser make lan |
| Vault sealed apres restart | Comportement normal de Vault | vault operator unseal avec la cle de init.json |
| Certificate format error pfSense | EasyRSA ajoute du texte lisible avant le PEM | Extraire avec openssl x509 -in ... > ...-clean.crt |
| Key values mismatch OpenVPN | Cert et cle regeneres separement | Supprimer /opt/easy-rsa/pki et infra-ansible/pki/ puis relancer make vpn |
| CSRF 403 NetBox | Django rejette les origines non declarees | Ajouter CSRF_TRUSTED_ORIGINS dans configuration.py.j2 |
| hvac module manquant | Dependance Python non installee | make install |
