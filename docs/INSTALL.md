# Guide d'installation — étape par étape

Ce document décrit, **dans l'ordre**, comment déployer l'infrastructure complète
(2 sites Proxmox, VPN site-à-site, pfSense, bastion, Vault, NetBox, OpenSearch,
DNS, site web interne, kill switch) entièrement en IaC (Ansible).

> Pour la reprise rapide après réinstallation, voir aussi [onboarding.md](./onboarding.md).
> Pour l'annuaire complet des commandes (debug, tunnels, tests), voir [COMMANDS.md](./COMMANDS.md).

Chaque étape précise : **ce qu'elle fait**, **ce qu'elle installe**, **comment la vérifier**.

---

## Vue d'ensemble de l'ordre de déploiement

```
SITE 1 (cœur : Vault, NetBox, OpenSearch)
  1. pfSense (VLANs + interfaces + règles + NAT)
  2. Vault            ──►  [MANUEL] init + unseal
  3. PKI + OpenVPN    (génère les certificats du tunnel site-à-site)
  4. OpenSearch + NetBox
  5. DNS Resolver (Unbound)
  6. Token API NetBox -> Vault
  7. Synchronisation NetBox (IPAM)
  8. Timer auto-sync NetBox
  9. Fluent Bit (collecte de logs)
 10. Durcissement SSH (key-only) — EN DERNIER

SITE 2 (services exposés : bastion, web interne, dashboards)
 11. pfSense (règles + NAT)
 12. OpenVPN client (monte le tunnel vers le Site 1)
 13. Bastion
 14. WebServer (OpenSearch Dashboards)
 15. Site web interne (Nginx)
 16. Fluent Bit (logs -> OpenSearch Site 1, via VPN)
 17. DNS Resolver (Unbound)
 18. Durcissement SSH (key-only)
```

**Tout en une commande** (avec pauses manuelles aux bons endroits) :

```bash
cd infra-ansible
make deploy-all
```

`deploy-all` enchaîne `site1-all` puis le Site 2. Il **s'arrête** là où une action
humaine est obligatoire (initialisation de Vault, saisie d'un token). On peut
aussi tout faire étape par étape avec les commandes ci-dessous.

---

## 0. Prérequis (poste local)

```bash
cd infra-ansible
make install                 # collections Ansible + libs Python
cp .env.example .env         # puis éditer .env (voir onboarding.md §4)
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -N ""   # si pas de clé
make ping                    # doit répondre "pong" sur tous les hôtes
```

`.env` contient les variables locales (jamais commitées) : `SSH_KEY`,
`GATEWAY_HOST`, `GATEWAY_USER`, `GATEWAY_PORT`, `VAULT_INTERNAL_HOST`,
`NETBOX_INTERNAL_HOST`, etc. Le fichier d'inventaire chiffré (`ansible-vault`)
utilise `~/.vault_pass`.

---

# SITE 1

## 1. pfSense — réseau + règles + NAT

```bash
make site1-pfsense
```

**Ce que ça fait :** crée les VLANs (LAN 10, ADMIN 30), configure les interfaces,
les règles firewall et le NAT (port-forward WAN → VMs internes : `2221`→lan_vm,
`2222`→admin_vm). Le NAT crée automatiquement les règles de filtrage associées.

**Vérifier :**
```bash
make ssh-lan      # connexion SSH à lan_vm via le NAT
make ssh-admin    # connexion SSH à admin_vm via le NAT
```

> Ne **jamais** lancer `pfctl -d` sur le pfSense : ça désactive le pare-feu,
> vide les états et **casse le NAT** → on perd l'accès SSH. Voir [COMMANDS.md](./COMMANDS.md).

---

## 2. Vault (admin_vm)

```bash
make site1-vault
```

**Ce que ça fait :** installe HashiCorp Vault sur `admin_vm` (192.168.30.101:8200),
service systemd, écoute sur l'IP ADMIN (pas seulement loopback).

### Initialiser Vault (MANUEL, une seule fois)

À la **première** installation, Vault est *non initialisé*. Sur `admin_vm` :

```bash
make ssh-admin
export VAULT_ADDR='https://192.168.30.101:8200'
export VAULT_SKIP_VERIFY=true
vault operator init        # NOTER les 5 unseal keys + le root token (init.json)
vault operator unseal      # coller une unseal key (x3)
vault status               # doit afficher Sealed: false
```

> Vault se **rescelle à chaque reboot** de la VM → re-`unseal` après redémarrage.
> Les clés sont dans `/etc/vault.d/init.json` (sur admin_vm). Sauvegarde Vault :
> voir [onboarding.md](./onboarding.md) §11.

---

## 3. PKI + OpenVPN site-à-site

```bash
make site1-vpn        # demande le token Vault (root)
```

**Ce que ça fait :** génère la PKI (CA, certificats serveur/client) **côté Site 1**,
stocke les clés dans Vault, configure le serveur OpenVPN sur le pfSense Site 1.
C'est le Site 1 qui produit les certificats ; le Site 2 les consomme (étape 12).

**Vérifier :** sur le pfSense, l'instance OpenVPN serveur est *up* (Status > OpenVPN).

---

## 4. OpenSearch + NetBox (lan_vm)

```bash
make site1-lan        # ouvre seul le tunnel Vault, puis déploie
```

**Ce que ça fait :** installe **OpenSearch** + **OpenSearch Dashboards backend**
et **NetBox** (IPAM) sur `lan_vm` (192.168.10.101). Récupère les mots de passe
depuis Vault (`secret/netbox`).

**Vérifier :**
```bash
make tunnel-netbox     # https://localhost:8443  (admin / cf Vault secret/netbox)
make tunnel-opensearch # curl -k -u admin:<pwd> https://localhost:9200
```

---

## 5. DNS Resolver (Unbound) — Site 1

```bash
make site1-dns
```

**Ce que ça fait :** configure Unbound sur le pfSense Site 1 en **resolver pur +
DNSSEC**, déclare la zone interne `site1.lan` (host overrides A + PTR), le
**RPZ** (blocage domaines malveillants, feed HaGeZi) et la **résolution croisée**
vers `site2.lan` via le VPN. Détails : section [DNS](#le-dns-en-détail) plus bas.

**Vérifier :** depuis une VM interne du Site 1 :
```bash
dig netbox.site1.lan @192.168.10.1 +short      # -> 192.168.10.101
dig bastion.site2.lan @192.168.10.1 +short     # -> 192.168.120.10 (croisé via VPN)
```

---

## 6. Token API NetBox -> Vault

```bash
make site1-netbox-token    # demande le token Vault (root)
```

**Ce que ça fait :** crée un token API NetBox et le **stocke dans Vault**
(`secret/netbox:api_token`). Ce token sert à la synchro IPAM automatisée.

---

## 7. Synchronisation NetBox (IPAM)

```bash
make netbox-sync
```

**Ce que ça fait :** pousse **dans NetBox** la description déclarative des **deux
sites** (sites, préfixes, VLANs, clusters, VMs, interfaces, IPs, devices,
services, tunnels). Source de vérité = `roles/netbox_sync/defaults/main.yml`.

**Vérifier :** dans NetBox (https://localhost:8443), les 2 sites, leurs préfixes
et VMs apparaissent.

---

## 8. Timer auto-sync NetBox (lan_vm)

```bash
make site1-netbox-autosync
```

**Ce que ça fait :** déploie sur `lan_vm` un venv Python + la collection NetBox +
un **timer systemd** (`netbox-sync.timer`, quotidien 03:00) qui rejoue la synchro
→ NetBox reste à jour automatiquement.

**Vérifier :** sur lan_vm : `systemctl list-timers | grep netbox`.

---

## 9. Fluent Bit (collecte de logs)

```bash
make site1-fluentbit
```

**Ce que ça fait :** installe **Fluent Bit** sur les VMs du Site 1. Collecte
`auth.log`/`syslog` (structurés via parsers `extract_syslog_json` + `syslog_plain`)
et `nginx/access.log` (parser `nginx_json`), ajoute `source_host`, envoie vers
**OpenSearch** (index `fluent-bit`).

**Vérifier :** sur chaque VM : `sudo systemctl status fluent-bit`. Dans
OpenSearch Dashboards, l'index `fluent-bit` reçoit des documents.

---

## 10. Durcissement SSH — EN DERNIER

```bash
make site1-harden     # lan_vm + admin_vm
```

**Ce que ça fait :** passe SSH en **key-only** (`PasswordAuthentication no`,
`PermitRootLogin no`, `AllowUsers` restreint). `admin_vm` garde
`AllowTcpForwarding yes` car c'est la **passerelle des tunnels** de management
(Vault 8200, NetBox 8443, OpenSearch 9200).

> À faire **en dernier** : une fois la clé SSH validée partout, sinon risque de
> se verrouiller dehors. Valider d'abord `make ssh-lan` / `make ssh-admin` sans
> mot de passe.

---

# SITE 2

`make site2-all` enchaîne 11→17. `make site2-vpn` (étape 12) est à lancer
explicitement car il demande le token Vault.

## 11–17. Déploiement Site 2

```bash
make site2-pfsense     # 11. règles + NAT (2231->bastion, 2232->webserver)
make site2-vpn         # 12. OpenVPN client -> monte le tunnel vers Site 1 (token Vault)
make site2-bastion     # 13. bastion (SSH hardening + fail2ban + logging)
make site2-webserver   # 14. OpenSearch Dashboards
make site2-website     # 15. site web interne (Nginx, INTERNE uniquement)
make site2-fluentbit   # 16. Fluent Bit -> OpenSearch Site 1 (via VPN)
make site2-dns         # 17. DNS Resolver (Unbound) Site 2
make site2-harden      # 18. durcissement SSH key-only (webserver)
```

Ou tout d'un coup (sauf le VPN qui demande le token) :
```bash
make site2-vpn
make site2-all         # pfsense -> bastion -> webserver -> website -> fluentbit -> dns
make site2-harden
```

**Vérifier :**
```bash
make ssh-bastion
make ssh-webserver
# site web interne (depuis le bastion, jamais depuis internet) :
curl -I http://192.168.110.100
```

> Le site web est **interne uniquement** (exigence du sujet) : pas de
> port-forward WAN vers Nginx. On y accède via le bastion / le VPN.

---

## 10bis / Vérifications finales (les deux sites)

```bash
# Connectivité
make ping

# VPN site-à-site : depuis lan_vm Site 1, joindre le webserver Site 2
ping 192.168.110.100

# DNS croisé
dig bastion.site2.lan @192.168.10.1 +short    # depuis Site 1
dig netbox.site1.lan  @192.168.110.1 +short   # depuis Site 2

# Logs : OpenSearch reçoit bien Site 1 ET Site 2 (champ source_host)
# (Dashboards Site 2 : index fluent-bit, filtrer par source_host)
```

Kill switch (coupure d'urgence) — **à tester en dernier, avec un accès console**
prêt :
```bash
make site1-killswitch          # isole le Site 1
make site1-killswitch-restore  # rétablit
make site2-killswitch          # isole le Site 2 (recovery via bastion)
make site2-killswitch-restore
```

---

## Le DNS en détail

> Réponse aux questions : **oui le DNS est bon, et oui les deux sites se parlent.**

### Ce qui est installé / configuré

Le DNS, c'est **Unbound**, le résolveur intégré de **pfSense** (« DNS Resolver »).
Aucune VM dédiée : c'est le pfSense de **chaque** site qui rend le service DNS à
son réseau. Configuré par le rôle Ansible [`roles/dns`](../infra-ansible/roles/dns)
(playbook [`dns.yml`](../infra-ansible/playbooks/dns.yml)), appliqué par
`make site1-dns` / `make site2-dns`.

| Élément | Détail | Fichier |
|---|---|---|
| **Resolver pur** | Unbound résout lui-même depuis les serveurs racine (pas de forward vers un DNS tiers) | `dns_forwarding: false` |
| **DNSSEC** | Validation des réponses publiques | `dns_dnssec: true` |
| **Zone interne** | `site1.lan` / `site2.lan` (host overrides A) | group_vars `firewall*.yml` |
| **PTR (reverse)** | `local-data-ptr` généré pour chaque hôte interne | `templates/custom_options.j2` |
| **RPZ** | Blocage malware/phishing, feed HaGeZi TIF mini (module `respip`) | `tasks/rpz.yml` |
| **Résolution croisée** | chaque site forwarde la zone de l'autre via le VPN | `dns_domainoverrides` |
| **Règles firewall DNS** | autorise le `:53` depuis les VLANs internes + depuis le tunnel VPN | `tasks/main.yml` |

### Hôtes résolus

**Site 1 (`site1.lan`)** → `netbox`, `opensearch` (192.168.10.101), `vault`
(192.168.30.101), `fw` (192.168.10.1).
**Site 2 (`site2.lan`)** → `bastion` (192.168.120.10), `webserver`, `dashboards`
(192.168.110.100), `fw` (192.168.110.1).

### Comment les deux sites se parlent (résolution croisée)

```
Client LAN Site 1  --(site2.lan ?)-->  Unbound pfSense Site 1
        Unbound Site 1 a un "domain override" : site2.lan -> 192.168.110.1
        --(requête DNS via le tunnel VPN, source 10.0.0.x)-->  Unbound pfSense Site 2
        Site 2 répond depuis sa zone site2.lan
```

C'est rendu possible par :
- `dns_domainoverrides` : `site2.lan -> 192.168.110.1` (côté Site 1) et
  `site1.lan -> 192.168.10.1` (côté Site 2) ;
- une règle firewall « **Allow DNS over VPN** » : autorise `udp/tcp 53` depuis le
  réseau tunnel `10.0.0.0/30` vers `(self)` sur l'interface `openvpn` ;
- `access-control: 10.0.0.0/30 allow` dans les custom options Unbound ;
- `domain-insecure` sur les zones internes (sinon DNSSEC casserait la résolution
  des zones `.lan` non signées).

**Test de bout en bout :**
```bash
# Depuis une VM Site 1
dig bastion.site2.lan @192.168.10.1 +short     # -> 192.168.120.10
# Depuis une VM Site 2
dig vault.site1.lan @192.168.110.1 +short      # -> 192.168.30.101
```

---