# Services déployés

## LAN VM — 192.168.10.101

| Service      | Port | Protocole | URL d'accès                          |
|-------------|------|-----------|---------------------------------------|
| OpenSearch   | 9200 | HTTPS     | https://192.168.10.101:9200           |
| NetBox       | 8080 | HTTP      | http://192.168.10.101:8080            |

### OpenSearch
- Version : 2.19.5
- Mode : single-node
- Sécurité : **activée** (TLS + authentification)
- TLS : certificats auto-signés (`/etc/opensearch/certs/`)
- Heap JVM : 512m
- Authentification : `admin` / voir `host_vars/lan_vm.yml`
- Commande de test : `curl -k -u admin:<password> https://192.168.10.101:9200`
- Plugins supprimés (gain d'espace) : opensearch-ml, opensearch-knn, opensearch-performance-analyzer, opensearch-skills, opensearch-flow-framework, opensearch-security-analytics, opensearch-neural-search, opensearch-ltr, opensearch-cross-cluster-replication

### NetBox
- Version : v4.1.0
- Base de données : PostgreSQL 16
- Cache : Redis
- Identifiants admin : `admin` / voir `host_vars/lan_vm.yml`

---

## Admin VM — 192.168.30.101

| Service | Port | Protocole | URL d'accès                        |
|---------|------|-----------|-------------------------------------|
| Vault   | 8200 | HTTP      | http://192.168.30.101:8200          |

### Vault
- Stockage : fichier (`/opt/vault/data`)
- TLS : désactivé
- UI web : activée

---

## Accès depuis l'extérieur

Ces services ne sont **pas exposés sur Internet**. Pour y accéder depuis ta machine locale :

```bash
# Tunnel SSH vers OpenSearch
ssh -L 9200:192.168.10.101:9200 <user>@<jump_host>

# Tunnel SSH vers NetBox
ssh -L 8080:192.168.10.101:8080 <user>@<jump_host>

# Tunnel SSH vers Vault
ssh -L 8200:192.168.30.101:8200 <user>@<jump_host>
```
