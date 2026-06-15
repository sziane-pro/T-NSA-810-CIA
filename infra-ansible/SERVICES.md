# Catalogue des services deployes

## lan_vm — 192.168.10.101

| Service     | Port | Protocole | Acces depuis l'exterieur                          |
|-------------|------|-----------|---------------------------------------------------|
| NetBox      | 443  | HTTPS     | Tunnel SSH puis https://localhost:8443            |
| OpenSearch  | 9200 | HTTPS     | Tunnel SSH puis https://localhost:9200            |

### NetBox

- Version : v4.1.0
- Stack : Django + Gunicorn + Nginx + PostgreSQL 16 + Redis
- TLS : certificat auto-signe (`/etc/ssl/netbox/`)
- Identifiants : stockes dans Vault (`secret/netbox:admin_password`)
- Utilisateur admin : `admin`

Tunnel d'acces :
```bash
ssh -p 2221 -i ~/.ssh/ansible_key -L 8443:192.168.10.101:443 sysadmin@5.196.51.228 -N
```
Puis ouvrir : `https://localhost:8443`

Test de l'API :
```bash
curl -k -H "Authorization: Token <api_token>" https://localhost:8443/api/dcim/sites/
```

### OpenSearch

- Version : 2.19.5
- Mode : single-node
- TLS : certificats auto-signes (`/etc/opensearch/certs/`)
- Heap JVM : 512m
- Authentification : `admin` / voir Vault (`secret/netbox:opensearch_admin_password`)
- Plugins desactives (gain d'espace) : opensearch-ml, opensearch-knn, opensearch-performance-analyzer, opensearch-skills, opensearch-flow-framework, opensearch-security-analytics, opensearch-neural-search, opensearch-ltr, opensearch-cross-cluster-replication

Tunnel d'acces :
```bash
ssh -p 2221 -i ~/.ssh/ansible_key -L 9200:192.168.10.101:9200 sysadmin@5.196.51.228 -N
```

Test de connexion :
```bash
curl -k -u admin:<password> https://localhost:9200
```

---

## admin_vm — 192.168.30.101

| Service | Port | Protocole | Acces depuis l'exterieur                          |
|---------|------|-----------|---------------------------------------------------|
| Vault   | 8200 | HTTPS     | Tunnel SSH puis https://localhost:8200            |

### HashiCorp Vault

- Version : derniere stable via depot HashiCorp
- Stockage : fichier (`/opt/vault/data`)
- TLS : certificat auto-signe (`/opt/vault/tls/`)
- UI web : activee
- Etat au demarrage : **sealed** (deverrouillage manuel requis apres chaque restart)

Tunnel d'acces :
```bash
ssh -p 2222 -i ~/.ssh/ansible_key -L 8200:192.168.30.101:8200 sysadmin@5.196.51.228 -N
```
Puis ouvrir : `https://localhost:8200`

Deverrouillage apres redemarrage :
```bash
export VAULT_ADDR='https://192.168.30.101:8200'
export VAULT_SKIP_VERIFY=true
sudo cat /etc/vault.d/init.json   # recuperer unseal_keys_b64[0]
vault operator unseal
```

---

## pfSense — 5.196.51.228

| Service        | Port | Description                           |
|----------------|------|---------------------------------------|
| SSH            | 22   | Acces administration pfSense          |
| SSH NAT lan_vm | 2221 | Acces SSH lan_vm via pfSense          |
| SSH NAT admin  | 2222 | Acces SSH admin_vm via pfSense        |
| HTTPS          | 443  | Interface web pfSense                 |
| OpenVPN        | 1194 | Tunnel VPN site-a-site (UDP)          |

Connexion :
```bash
ssh -p 22 -i ~/.ssh/ansible_key admin@5.196.51.228
```
