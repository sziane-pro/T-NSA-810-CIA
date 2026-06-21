# Annuaire des commandes — mémo complet

Toutes les commandes utiles du projet, regroupées par thème : déploiement,
accès, tunnels, Vault, DNS, OpenSearch/logs, NetBox, kill switch, pfSense,
ansible-vault, git, et **dépannage** (avec les pannes réellement rencontrées).

> Installation pas à pas : [INSTALL.md](./INSTALL.md) — Reprise de poste : [onboarding.md](./onboarding.md).
> Toutes les commandes `make` se lancent depuis `infra-ansible/`.

---

## 1. Aide & dépendances

```bash
make help            # liste toutes les cibles
make install         # collections Ansible + libs Python
make ping            # teste la connectivité Ansible (les 2 sites)
make ping-s2         # Site 2 uniquement
```

---

## 2. Déploiement — Site 1

```bash
make site1-all              # TOUT le Site 1, étape par étape (avec pauses manuelles)

# ... ou étape par étape :
make site1-pfsense          # VLANs + interfaces + règles + NAT
make site1-pfsense-rules    # règles firewall uniquement
make site1-pfsense-nat      # NAT uniquement
make site1-pfsense-vpn      # config VPN du pfSense uniquement (token Vault)
make site1-vault            # installe Vault sur admin_vm
make site1-vpn              # PKI + OpenVPN (token Vault)
make site1-pki              # régénère la PKI uniquement
make site1-lan              # OpenSearch + NetBox sur lan_vm
make site1-dns              # DNS Resolver (Unbound)
make site1-netbox-token     # token API NetBox -> Vault (token Vault)
make netbox-sync            # synchro NetBox (IPAM, les 2 sites)
make site1-netbox-autosync  # timer systemd auto-sync sur lan_vm
make site1-fluentbit        # Fluent Bit sur les VMs Site 1
make site1-harden           # SSH key-only (lan_vm + admin_vm) — EN DERNIER
```

## 3. Déploiement — Site 2

```bash
make site2-vpn              # OpenVPN client -> monte le tunnel (token Vault)
make site2-all             # pfsense -> bastion -> webserver -> website -> fluentbit -> dns

# ... ou étape par étape :
make site2-pfsense          # règles + NAT
make site2-pfsense-rules    # règles uniquement
make site2-pfsense-nat      # NAT uniquement
make site2-vpn-client       # client VPN Site 2 uniquement (ne touche pas Site 1)
make site2-bastion          # bastion (SSH hardening + fail2ban + logging)
make site2-webserver        # OpenSearch Dashboards
make site2-website          # site web interne Nginx (INTERNE only)
make site2-fluentbit        # Fluent Bit -> OpenSearch Site 1 (via VPN)
make site2-dns              # DNS Resolver (Unbound)
make site2-harden           # SSH key-only (webserver)

# Dry-run (simulation, aucune modif) :
make site2-check
make site2-pfsense-check
make site2-bastion-check
make site2-webserver-check
```

## 4. Orchestration globale

```bash
make deploy-all      # Site 1 complet, puis VPN Site 2, puis Site 2 complet
```

---

## 5. Accès SSH

```bash
# Site 1
make ssh-pfsense     # pfSense Site 1 (admin@WAN:22)
make ssh-lan         # lan_vm   (sysadmin, port 2221 via NAT)
make ssh-admin       # admin_vm (sysadmin, port 2222 via NAT)

# Site 2
make ssh-pfsense-s2  # pfSense Site 2
make ssh-bastion     # bastion  (port 2231)
make ssh-webserver   # webserver(port 2232)

# Connexion manuelle équivalente
ssh -p 2222 -i ~/.ssh/ansible_key sysadmin@5.196.51.228
```

---

## 6. Tunnels SSH (accès aux services internes)

Les services (Vault, NetBox, OpenSearch) ne sont **pas exposés sur internet** :
on passe par un tunnel SSH via admin_vm (la passerelle, port `2222`).

```bash
make tunnel-vault        # https://localhost:8200 -> Vault
make tunnel-netbox       # https://localhost:8443 -> NetBox
make tunnel-opensearch   # https://localhost:9200 -> OpenSearch
make tunnel-all          # les trois d'un coup (Ctrl+C pour fermer)

# Tunnel manuel (en arrière-plan)
ssh -fN -p 2222 -i ~/.ssh/ansible_key \
    -L 8200:192.168.30.101:8200 \
    -o StrictHostKeyChecking=no sysadmin@5.196.51.228

# Lister / tuer les tunnels
pgrep -af "ssh.*-L"
pkill -f "ssh.*-L"            # ferme TOUS les tunnels -L
ss -tlnp | grep ':8200'      # voir qui occupe le port local
```

> « **bind [127.0.0.1]:8200: Address already in use** » = un tunnel occupe déjà
> le port → `pkill -f "ssh.*-L"` avant d'en ouvrir un neuf.

---

## 7. Vault

```bash
# Sur admin_vm
export VAULT_ADDR='https://192.168.30.101:8200'
export VAULT_SKIP_VERIFY=true

vault status                 # Sealed: false attendu
vault operator init          # 1re fois seulement (note les unseal keys + root token)
vault operator unseal        # à refaire après chaque reboot de la VM

vault kv get secret/netbox   # lire les secrets NetBox/OpenSearch
vault kv put secret/netbox db_password="..." admin_password="..." \
      secret_key="..." opensearch_admin_password="..."

# Via tunnel depuis le poste local (après make tunnel-vault)
curl -k https://127.0.0.1:8200/v1/sys/health    # {"sealed":false,...} attendu

# Sauvegarde / restauration : voir onboarding.md §11 et §8
```

---

## 8. DNS (Unbound / pfSense)

```bash
# Déploiement
make site1-dns
make site2-dns

# Tests de résolution (depuis une VM interne)
dig netbox.site1.lan @192.168.10.1 +short      # zone locale Site 1
dig bastion.site2.lan @192.168.10.1 +short     # CROISÉ Site1 -> Site2 (via VPN)
dig vault.site1.lan @192.168.110.1 +short      # CROISÉ Site2 -> Site1
dig -x 192.168.10.101 @192.168.10.1 +short     # PTR (reverse)

# DNSSEC (doit renvoyer AD flag pour un domaine signé)
dig dnssec-failed.org @192.168.10.1
dig cloudflare.com @192.168.10.1 +dnssec

# RPZ (un domaine de la blocklist doit renvoyer NXDOMAIN)
dig <domaine-malveillant> @192.168.10.1

# Sur le pfSense : recharger Unbound / inspecter
pfSsh.php playback svc restart unbound
unbound-checkconf
cat /var/unbound/rpz/hagezi-tif.rpz | head
```

---

## 9. OpenSearch & logs (Fluent Bit)

```bash
# Déploiement collecteurs
make site1-fluentbit
make site2-fluentbit

# Sur une VM : état du collecteur
sudo systemctl status fluent-bit
sudo journalctl -u fluent-bit -n 50 --no-pager
sudo tail -f /var/log/td-agent-bit/td-agent-bit.log   # selon packaging

# OpenSearch via tunnel (make tunnel-opensearch)
curl -k -u admin:<password> https://localhost:9200
curl -k -u admin:<password> https://localhost:9200/_cat/indices?v
curl -k -u admin:<password> "https://localhost:9200/fluent-bit/_count"
curl -k -u admin:<password> "https://localhost:9200/fluent-bit/_search?size=5&pretty"

# Voir les logs d'un hôte précis (champ ajouté par Fluent Bit)
curl -k -u admin:<pwd> "https://localhost:9200/fluent-bit/_search?q=source_host:webserver_vm&pretty"
```

Requêtes de sécurité (DQL dans OpenSearch Dashboards) : voir
[analyze-monitoring.md](./analyze-monitoring.md) §8.1 (brute-force SSH, sudo,
fail2ban, statuts HTTP nginx).

---

## 10. NetBox (IPAM)

```bash
make site1-netbox-token      # crée le token API et le met dans Vault
make netbox-sync             # pousse la description des 2 sites dans NetBox
make site1-netbox-autosync   # timer quotidien

# Accès UI (après make tunnel-netbox) : https://localhost:8443
#   user admin / mot de passe = Vault secret/netbox:admin_password

# Sur lan_vm : état du timer auto-sync
systemctl list-timers | grep netbox
systemctl status netbox-sync.service
journalctl -u netbox-sync.service -n 50 --no-pager
```

---

## 11. Kill switch (coupure d'urgence)

> Toujours avoir un **accès console** (Proxmox/OVH) prêt AVANT de couper le Site 1
> (pas de bastion côté Site 1). Le Site 2 garde le bastion joignable pour la reprise.

```bash
make site1-killswitch          # isole totalement le Site 1
make site1-killswitch-restore  # rétablit (et pré-crée les règles)
make site2-killswitch          # isole le Site 2
make site2-killswitch-restore  # rétablit
```

Détails de conception : [ADR-009](./adr/ADR-009-kill-switch-restauration.md) et
[disaster-recovery-plan.md](./disaster-recovery-plan.md).

---

## 12. pfSense (shell)

```bash
# Console pfSense — utilise tcsh : pour du bash, encapsuler
sh -c 'commande bash ...'

# API de config pfSense 2.8 (le $config global est déprécié)
pfSsh.php playback ...
#   config_set_path('system/webgui/protocol','https')   # ex. forcer HTTPS GUI
#   config_get_path('...')

# Recharger un service
pfSsh.php playback svc restart unbound
pfSsh.php playback svc restart openvpn

# États / filtre (LECTURE SEULE — ne PAS désactiver pf)
pfctl -s rules        # règles chargées
pfctl -s nat          # NAT/port-forward
pfctl -s states       # états
```

> **`pfctl -d` est INTERDIT** : il désactive le pare-feu, vide les états et
> **casse le NAT** → perte de l'accès SSH. Si bloqué, recharger les règles
> proprement, ne jamais désactiver pf.

---

## 13. ansible-vault (secrets de l'inventaire)

```bash
# Le mot de passe vault est dans ~/.vault_pass (jamais commité)
ansible-vault view  inventories/site1/group_vars/all/vault.yml --vault-password-file ~/.vault_pass
ansible-vault edit  inventories/site1/group_vars/all/vault.yml --vault-password-file ~/.vault_pass
ansible-vault encrypt <fichier> --vault-password-file ~/.vault_pass

# Dans l'éditeur (vi) : i pour insérer, Échap puis :wq pour sauver+quitter
#                       (ou EDITOR=nano ansible-vault edit ... ; Ctrl+O, Entrée, Ctrl+X)
```

---

## 14. Ansible — exécution ciblée & debug

```bash
# Limiter à un hôte
ansible-playbook -i inventories/site1 playbooks/ssh_hardening.yml --limit admin_vm

# Tags
ansible-playbook -i inventories/site2 playbooks/pfsense_network_s2.yml --tags rules
ansible-playbook -i inventories/site1 playbooks/pfsense_network.yml   --tags nat

# Simulation
ansible-playbook -i inventories/site2 playbooks/bastion_s2.yml --check --diff

# Verbosité
ansible-playbook ... -vvv

# Module ad-hoc
ansible all -i inventories/site1 -m ping
ansible admin_vm -i inventories/site1 -m command -a "systemctl status vault"
```

---

## 15. Git

```bash
git status
git add <fichiers ciblés>          # commits séparés et propres
git commit -m "feat(dns): ..."     # PAS de trailer Co-Authored-By (préférence projet)
git log --oneline -10
```

---

## 16. Dépannage — pannes réellement rencontrées

| Symptôme | Cause | Correctif |
|---|---|---|
| SSH/pfSense « tourne dans le vide » jusqu'au timeout | `pfctl -d` lancé / états vidés / NAT cassé | Ne jamais faire `pfctl -d` ; recharger les règles proprement |
| GUI pfSense injoignable | webConfigurator en HTTP/80 mais règle en HTTPS/443 | `config_set_path('system/webgui/protocol','https')` |
| `bind [127.0.0.1]:8200: Address already in use` | tunnel résiduel sur le port | `pkill -f "ssh.*-L"` puis rouvrir |
| Vault lookup « SSL EOF » lors d'un `make ...fluentbit` | `AllowTcpForwarding no` sur admin_vm (passerelle des tunnels) | `ssh_allow_tcp_forwarding: "yes"` dans host_vars admin_vm, puis `make site1-harden` |
| SSH demande un mot de passe au lieu de la clé | mauvaise clé publique dans `authorized_keys` | corriger la pubkey ; le vault stocke le mdp **sudo**, pas le mdp **login** |
| « Incorrect sudo password » | typo dans le secret vault | `ansible-vault edit ... --vault-password-file ~/.vault_pass` |
| NetBox autosync : « Failed to import pynetbox » | mauvais interpréteur Python | pinner `ansible_python_interpreter` sur le venv |
| Unbound : « RPZ requires respip module » | module respip non activé | `module-config: "respip validator iterator"` |
| Unbound : erreur sur `access-control` | guillemets parasites | `access-control: 10.0.0.0/30 allow` (sans guillemets) |
| RPZ feed HTTP 405 / banni | abuse.ch exige une Auth-Key | feed HaGeZi `tif.mini` |
| DNS croisé ne résout pas | flux `:53` non autorisé depuis le tunnel | règle « Allow DNS over VPN » + `access-control` sur `10.0.0.0/30` |

**Réflexes de diagnostic tunnel/Vault :**
```bash
pkill -f "ssh.*-L"; sleep 1                       # repartir propre
make site1-harden                                  # rétablit le forwarding admin_vm
ssh -fN -p 2222 -i ~/.ssh/ansible_key \
    -L 8200:192.168.30.101:8200 \
    -o StrictHostKeyChecking=no sysadmin@5.196.51.228
curl -k https://127.0.0.1:8200/v1/sys/health; echo "  exit:$?"   # {"sealed":false}
```
