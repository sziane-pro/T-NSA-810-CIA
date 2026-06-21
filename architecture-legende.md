# Légende de l'architecture réseau hybride Proxmox

> Synthèse de l'infrastructure réellement déployée. Détail complet :
> [architecture-documentation.md](architecture-documentation.md).

## Sites et infrastructure

- **SITE 1 (cœur, on-premise)** : services centralisés (Vault, NetBox, OpenSearch), hub VPN.
- **SITE 2 (distant, remote)** : bastion d'accès, OpenSearch Dashboards, site web interne.
- **Chaque site** : maximum 3 VMs sur Proxmox VE.

## Segmentation réseau

### Site 1
| Segment | VLAN | Plage | Usage |
|---|---|---|---|
| LAN | 10 | `192.168.10.0/24` | Services internes (lan_vm) |
| ADMIN | 30 | `192.168.30.0/24` | Management (admin_vm) |

### Site 2
| Segment | VLAN | Plage | Usage |
|---|---|---|---|
| LAN | 110 | `192.168.110.0/24` | Serveur web interne |
| DMZ | 120 | `192.168.120.0/24` | Bastion (accès externe) |
| ADMIN | 130 | `192.168.130.0/24` | Management |

## VPN site-à-site

- **Tunnel OpenVPN** : `10.0.0.0/30` (S1 `10.0.0.1` = serveur, S2 `10.0.0.2` = client)
- **Transport** : UDP `1194`
- **Chiffrement** : AES-256-GCM
- **Authentification** : PKI X.509 (CA + certificats), gérée dans Vault
- **Routage** : annonce automatique des réseaux internes entre les sites

## Sécurité

- Pare-feu **pfSense** sur chaque site, filtrage least privilege (versionné en IaC)
- NAT/accès restreints à des **IP sources whitelistées**
- **Bastion** SSH durci (key-only, fail2ban) pour l'accès externe
- **DNS** Unbound : resolver pur + DNSSEC + RPZ (anti-malware) + résolution croisée
- **Kill switch** : anti-fuite VPN (automatique) + isolation totale (à la demande)
- Aucun service exposé directement sur Internet (accès via tunnels SSH)

## Services centralisés

| Service | Outil | Localisation | Rôle |
|---|---|---|---|
| Secrets / PKI | HashiCorp Vault | Site 1 (admin_vm) | Source des secrets |
| IPAM | NetBox | Site 1 (lan_vm) | Source de vérité réseau |
| Observabilité | OpenSearch + Dashboards | Site 1 (collecte) / Site 2 (dashboards) | Logs multi-sites |
| DNS | Unbound (pfSense) | Chaque site | Résolution + croisée |
| VPN hub | OpenVPN | Site 1 | Point central |

## Automatisation

- Tout en **Infrastructure as Code** (Ansible) — reproductible depuis Git + Vault
- **NetBox** peuplé par le code + **synchro automatique** (timer systemd)
- Collecte de logs via **Fluent Bit** (les deux sites)

## Observabilité

- Logs centralisés dans **OpenSearch**, visualisés via **OpenSearch Dashboards**
- Sources : firewall, SSH/sudo, fail2ban (`auth.log`), accès **Nginx** (JSON)
- Champ `source_host` ajouté à chaque événement

## Procédures d'urgence

| Procédure | Description |
|---|---|
| Anti-fuite VPN | Automatique : bloque la fuite inter-sites si le tunnel tombe |
| Kill switch | À la demande : isolation totale du site (WAN + VPN) |
| Reprise | Hors-bande : console Proxmox (S1 & S2) + bastion (S2) |
| Runbooks | Procédures détaillées : [disaster-recovery-plan.md](docs/disaster-recovery-plan.md) |

## Évolutivité

- Convention d'adressage extensible (nouveau site = triplet VLAN + tunnel `10.0.0.x/30`)
- Rôles Ansible paramétrés par site, description IPAM data-driven
- Architecture hub-and-spoke scalable

## Flux principaux

1. **Admin externe** → Bastion (SSH, IP whitelistées) → réseau interne
2. **Site 1 ↔ Site 2** via VPN chiffré
3. **Tous les logs** → OpenSearch (Site 1)
4. **Gestion IP** → NetBox (Site 1)
5. **DNS** : résolution croisée `site1.lan ↔ site2.lan` via le VPN
