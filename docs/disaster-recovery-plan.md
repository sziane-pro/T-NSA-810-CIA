# Plan de Reprise d'Activité (DRP) & Runbooks

> Document opérationnel : **comment isoler, diagnostiquer et reconstruire** l'infrastructure
> hybride (Site 1 on-prem / Site 2 remote). À garder à jour à chaque évolution.

## 1. Objet & périmètre

Ce document couvre :

- la **coupure d'urgence** (kill switch) et le rétablissement — cf. [ADR-009](adr/ADR-009-kill-switch-restauration.md) ;
- la **reconstruction complète** d'un site à partir du code (IaC) ;
- des **runbooks ciblés** par scénario d'incident (VPN, DNS, lockout SSH, perte de VM, compromission).

Principe directeur : **tout est reconstructible depuis Git + Vault**. La coupure d'urgence
**ne doit jamais empêcher la reprise** (accès hors-bande conservé).

## 2. Sources de vérité & prérequis de reprise

| Élément | Où | Remarque |
|---|---|---|
| Code IaC (Ansible) | Dépôt Git | Source de vérité de la config |
| Secrets applicatifs / PKI | **Vault** (admin_vm S1, `192.168.30.101:8200`) | Nécessite les **clés d'unseal** |
| Secrets chiffrés inventaire | `ansible-vault` (`group_vars/*/vault.yml`) | Mot de passe dans `~/.vault_pass` (jamais commité) |
| Plan d'adressage / inventaire | **NetBox** (lan_vm S1) + `inventories/` | IPAM = référence |
| Clé SSH d'automatisation | `~/.ssh/ansible_key` | Déployée sur les VMs |

**Accès de secours (hors-bande) — indispensables avant tout incident :**

- **Console Proxmox** (S1 & S2) → accès VM même réseau coupé.
- **Console / KVM OVH** du pfSense → accès firewall même WAN coupé.
- **Bastion S2** (`192.168.120.10`, via NAT WAN `:2231`) → rebond d'admin vers le réseau interne.

>  Sans console Proxmox, la reprise après une coupure totale est impossible. **Vérifier
> régulièrement que ces accès fonctionnent.**

## 3. Niveaux d'incident & arbre de décision

| Niveau | Situation | Action |
|---|---|---|
| **N1** | Service dégradé (1 VM, 1 daemon) | Runbook ciblé §6 |
| **N2** | Compromission suspectée d'un site | **Kill switch** §4 → investigation → restauration |
| **N3** | Perte d'un site / VM | Reconstruction §5 |

## 4. Runbook A — Coupure d'urgence (kill switch)

Implémentation : règles **flottantes `quick`** pré-créées **désactivées** (`roles/killswitch`),
basculées via la variable `killswitch_enabled`. Isolation **totale** du WAN (entrant + sortant)
+ VPN. Recovery : **console** (S1 & S2) + **bastion** (S2, entrée SSH conservée).

### 4.1 Activation

**Méthode A — Ansible (si le pfSense est encore joignable depuis le contrôleur) :**
```bash
make site2-killswitch      # isole le Site 2
make site1-killswitch      # isole le Site 1
```
> Le run d'activation peut se terminer par une erreur de connexion **après** avoir appliqué la
> règle (normal : il vient de couper le WAN).

**Méthode B — GUI/console (recommandée en vrai incident, fiable même réseau dégradé) :**
1. Console pfSense → option **8) Shell**, ou GUI **Firewall > Rules > Floating**.
2. Activer la règle `KILL-SWITCH block all WAN` (décocher *disabled*).
3. Appliquer.

### 4.2 Flux coupés / maintenus

| Coupé | Maintenu (recovery) |
|---|---|
| Tout le WAN entrant/sortant | **Console Proxmox** (hors-bande) |
| Tunnel VPN inter-sites | **Bastion S2** : entrée SSH `:2231` (exception flottante) |
| Exposition des services / NAT | Réseaux **internes** (LAN/DMZ/Admin) inchangés |

### 4.3 Rétablissement

> Une fois le site isolé, **Ansible ne peut plus joindre le pfSense par le WAN**. Le retour
> arrière se fait **par la console**.

- **Console pfSense** : Firewall > Rules > Floating → **désactiver** `KILL-SWITCH block all WAN` → Appliquer.
- Puis, accès réseau rétabli, re-synchroniser l'état désactivé en IaC :
  ```bash
  make site2-killswitch-restore
  make site1-killswitch-restore
  ```

### 4.4 Test / simulation (à faire AVANT un vrai incident)

1. Avoir la **console Proxmox ouverte** (filet de sécurité).
2. `make site2-killswitch` → vérifier que :
   - les services externes ne répondent plus,
   - **le bastion répond toujours** (`ssh -p 2231 …`),
   - la console reste accessible.
3. Rétablir via console (§4.3) puis `make site2-killswitch-restore`.

## 5. Runbook B — Reconstruction complète d'un site (IaC)

### 5.1 Prérequis
- Dépôt Git cloné, `~/.vault_pass` en place, `~/.ssh/ansible_key` présent.
- Accès console Proxmox des hôtes.

### 5.2 Recréer les VMs (Proxmox)
>  La couche VM n'est pas (encore) en IaC. Recréer les VMs selon le plan d'adressage NetBox :

| Site | VMs (max 3) | Réseau |
|---|---|---|
| S1 | pfSense, lan_vm (`.10.101`), admin_vm (`.30.101`) | LAN 10 / ADMIN 30 |
| S2 | pfSense, bastion (`.120.10`), webserver (`.110.100`) | LAN 110 / DMZ 120 / ADMIN 130 |

Bootstrap de chaque VM Linux : créer le compte d'admin (`sysadmin` S1 / `t-nsa-810-cia` S2),
déposer `ansible_key.pub` dans `~/.ssh/authorized_keys` (perms `700`/`600`).

### 5.3 Ordre de déploiement — Site 1
```bash
make site1-pfsense        # interfaces, VLANs, NAT, règles
make site1-vault          # Vault sur admin_vm  -> UNSEAL (cf. §7)
make site1-pki            # PKI dans Vault
make site1-vpn            # serveur OpenVPN
make site1-lan            # OpenSearch + NetBox
make site1-netbox-token   # token API NetBox -> Vault
make netbox-sync          # peuple/MAJ NetBox (IPAM, les 2 sites)
make site1-dns            # resolver Unbound + RPZ
make site1-fluentbit      # logs -> OpenSearch
make site1-harden         # SSH key-only
make site1-killswitch-restore   # pré-crée le kill switch (désactivé)
```

### 5.4 Ordre de déploiement — Site 2
```bash
make site2-pfsense
make site2-vpn            # client OpenVPN (relance la PKI S1)
make site2-bastion        # durcissement + fail2ban
make site2-webserver      # OpenSearch Dashboards
make site2-website        # site web interne (Nginx)
make site2-dns
make site2-fluentbit
make site2-harden
make site2-killswitch-restore
# ou tout-en-un (hors dns/harden/killswitch) : make site2-all
```

### 5.5 Dépendances clés
- `site2-vpn` dépend de la **PKI S1** (Vault unsealed).
- `*-fluentbit` / `*-netbox-*` dépendent du **tunnel Vault** (géré par le Makefile) + `VAULT_TOKEN`.
- DNS croisé dépend du **VPN up**.

## 6. Runbooks ciblés (scénarios)

### 6.1 VPN inter-sites down
- pfSense S2 : **Status > OpenVPN** ; vérifier `Status > System Logs`.
- Vérifier règle WAN UDP 1194 + accessibilité `site1_wan_ip`.
- Re-pousser : `make site2-vpn-client`.
- Effet attendu si down : résolution croisée `site1.lan ↔ site2.lan` KO, **local OK** (resolver pur).

### 6.2 GUI / DNS pfSense injoignable
- Timeout = paquet droppé (firewall) ; *refused* = service down.
- Console : `unbound-checkconf /var/unbound/unbound.conf` ; `sockstat -4 -l | grep ':53'`.
- GUI : option **11) Restart webConfigurator**. **Ne jamais** `pfctl -d` (purge les états, casse le NAT).

### 6.3 Lockout SSH (après durcissement)
- **Console Proxmox** de la VM → éditer `/etc/ssh/sshd_config` ou corriger `~/.ssh/authorized_keys` (perms `700`/`600`, owner correct) → `systemctl restart ssh`.
- Cause fréquente : mauvaise clé dans `authorized_keys` ou mauvaises permissions.

### 6.4 Perte d'une VM
- Recréer la VM (§5.2) puis rejouer **uniquement** son playbook (`make site2-website`, `make site1-lan`, …).
- Les secrets sont dans **Vault** (pas sur la VM) → pas de perte.

### 6.5 Compromission suspectée
1. **Isoler** : kill switch §4 (méthode console).
2. **Investiguer** : OpenSearch Dashboards (logs centralisés), `auth.log`, fail2ban.
3. **Reconstruire** la/les VM(s) compromise(s) depuis l'IaC (§5).
4. **Rotation des secrets** dans Vault + redéploiement.

## 7. Restauration des secrets (Vault)

- Vault démarre **scellé** → fournir les **clés d'unseal** (stockées hors dépôt, en lieu sûr).
- `make site1-vault` configure le service ; l'unseal/initialisation est dans `roles/vault`.
- Le `VAULT_TOKEN` (root/opérateur) est requis pour les playbooks qui lisent des secrets
  (`*-vpn`, `*-fluentbit`, `*-netbox-*`) ; le tunnel `:8200` est ouvert par le Makefile.
- **Sans clés d'unseal** : PKI/secrets irrécupérables → les régénérer (nouvelle PKI, nouveau VPN).

## 8. Checklist de vérification post-reprise

- [ ] VPN up (`Status > OpenVPN`, ping inter-LAN `10.x` / `192.168.x`)
- [ ] DNS local + croisé (`drill webserver.site2.lan`, `drill netbox.site1.lan`)
- [ ] RPZ actif (`drill <domaine_bloqué>` → NXDOMAIN)
- [ ] Site web interne (`curl -I http://192.168.110.100` depuis le bastion → 200)
- [ ] Site web **non** exposé sur Internet
- [ ] Bastion joignable (`ssh -p 2231 …`)
- [ ] NetBox + OpenSearch accessibles (tunnels) ; logs qui remontent
- [ ] SSH key-only (password refusé) ; kill switch pré-créé (désactivé)

## 9. RTO/RPO indicatifs

| | Cible |
|---|---|
| **RTO** (reconstruction 1 site, VMs prêtes) | ~1–2 h via les `make` |
| **RPO** | Config = dernier commit Git ; secrets = état Vault ; données OpenSearch selon snapshots |

> Améliorations prévues : provisioning des VMs en IaC (Terraform/cloud-init), snapshots
> OpenSearch planifiés, sauvegarde chiffrée des clés d'unseal Vault.
