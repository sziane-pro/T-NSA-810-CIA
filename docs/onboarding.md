# Guide de prise de poste

Ce document decrit les etapes necessaires pour reprendre le projet sur une nouvelle machine ou apres une reinstallation.

---

## 1. Prerequis a installer

```bash
# Ansible
pip3 install ansible

# Ou via apt
sudo apt install ansible python3-pip -y
```

---

## 2. Cloner le depot

```bash
git clone <url-du-depot>
cd T-NSA-810-CIA/infra-ansible
```

---

## 3. Installer les dependances

```bash
make install
```

Cette commande installe :
- Les collections Ansible (`community.postgresql`, `community.hashi_vault`, `pfsensible.core`)
- Les librairies Python (`hvac`, `psycopg2-binary`)

---

## 4. Configurer les variables locales

```bash
cp .env.example .env
```

Editer `.env` avec les valeurs reelles :

```bash
VAULT_TOKEN=<root_token>         # sudo cat /etc/vault.d/init.json sur admin_vm
VAULT_ADDR=https://127.0.0.1:8200
VAULT_SKIP_VERIFY=true
GATEWAY_USER=sysadmin
GATEWAY_HOST=<ip_publique_pfsense>
GATEWAY_PORT=2222
SSH_KEY=~/.ssh/ansible_key
```

---

## 5. Configurer la cle SSH

La cle privee SSH doit etre presente sur la machine locale pour acceder aux VMs via pfSense.

```bash
# Generer une nouvelle cle si necessaire
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -N ""

# Copier la cle publique sur pfSense (manuellement via l'interface web pfSense)
# System > User Manager > Users > admin > Authorized SSH Keys
cat ~/.ssh/ansible_key.pub
```

Tester la connexion :
```bash
make ping
```

---

## 6. Recuperer le token Vault

Le token Vault est necessaire pour que les playbooks recuperent les secrets.

```bash
# Se connecter sur admin_vm
ssh -p 2222 -i ~/.ssh/ansible_key sysadmin@5.196.51.228

# Afficher les cles
sudo cat /etc/vault.d/init.json
```

Copier la valeur de `root_token` dans le fichier `.env` local (`VAULT_TOKEN=`).

---

## 7. Deverrouiller Vault

Vault se verrouille a chaque redemarrage de la VM. Verifier son etat :

```bash
# Sur admin_vm
export VAULT_ADDR='https://192.168.30.101:8200'
export VAULT_SKIP_VERIFY=true
vault status
```

Si `Sealed: true`, deverrouiller avec la cle dans `init.json` :

```bash
vault operator unseal   # coller la valeur de unseal_keys_b64[0]
```

---

## 8. Verifier les secrets dans Vault

```bash
# Sur admin_vm (ou via tunnel SSH ouvert)
vault kv get secret/netbox
```

Si les secrets sont absents (VM reinitialise), les re-saisir :

```bash
vault kv put secret/netbox \
  db_password="..." \
  admin_password="..." \
  secret_key="..." \
  opensearch_admin_password="..."
```

Les valeurs de reference se trouvent dans la sauvegarde Vault (`vault-backup.tar.gz`).

### Restaurer Vault depuis une sauvegarde

```bash
# Copier la sauvegarde sur admin_vm
scp -P 2222 -i ~/.ssh/ansible_key vault-backup.tar.gz sysadmin@5.196.51.228:/home/sysadmin/

# Sur admin_vm
sudo systemctl stop vault
sudo rm -rf /opt/vault/data
sudo tar xzf /home/sysadmin/vault-backup.tar.gz -C /
sudo systemctl start vault
vault operator unseal
```

---

## 9. Deployer ou redeployer l'infrastructure

```bash
# Si tout est a reconstruire
make site1-pfsense   # pfSense : VLANs, interfaces, regles firewall
make site1-vault     # Vault sur admin_vm
make site1-lan       # OpenSearch + NetBox sur lan_vm
make site1-vpn       # PKI + OpenVPN

# Si seulement une partie est a redeployer
make site1-pfsense-rules   # Regles firewall uniquement
make site1-pfsense-vpn     # VPN pfSense uniquement
make site1-pki             # Regenerer la PKI uniquement
```

---

## 10. Acceder aux services

Tous les services sont sur des reseaux internes non accessibles directement depuis internet. L'acces se fait via tunnel SSH a travers pfSense.

### NetBox
```bash
ssh -p 2221 -i ~/.ssh/ansible_key -L 8443:192.168.10.101:443 sysadmin@5.196.51.228 -N &
# Ouvrir https://localhost:8443
# Utilisateur : admin / Mot de passe : voir Vault secret/netbox:admin_password
```

### OpenSearch
```bash
ssh -p 2221 -i ~/.ssh/ansible_key -L 9200:192.168.10.101:9200 sysadmin@5.196.51.228 -N &
# curl -k -u admin:<password> https://localhost:9200
```

### Vault
```bash
ssh -p 2222 -i ~/.ssh/ansible_key -L 8200:192.168.30.101:8200 sysadmin@5.196.51.228 -N &
# Ouvrir https://localhost:8200
```

---

## 11. Sauvegarder Vault

Apres toute modification des secrets, creer une nouvelle sauvegarde :

```bash
# Sur admin_vm
sudo tar czf /home/sysadmin/vault-backup.tar.gz /opt/vault/data
sudo chown sysadmin:sysadmin /home/sysadmin/vault-backup.tar.gz

# Telecharger en local
scp -P 2222 -i ~/.ssh/ansible_key sysadmin@5.196.51.228:/home/sysadmin/vault-backup.tar.gz ./vault-backup.tar.gz
```

Le fichier `vault-backup.tar.gz` est dans `.gitignore` et ne doit jamais etre commite.

---

## Rappel des fichiers sensibles

Ces fichiers ne sont jamais dans git :

| Fichier | Contenu |
|---------|---------|
| `infra-ansible/.env` | Tokens, adresses, cles de configuration locale |
| `infra-ansible/pki/` | Certificats et cles privees VPN |
| `vault-backup.tar.gz` | Sauvegarde des secrets Vault |
| `*.key` | Toute cle privee |
