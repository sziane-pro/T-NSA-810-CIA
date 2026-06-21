ANSIBLE_CONFIG := $(shell pwd)/ansible.cfg
export ANSIBLE_CONFIG

PLAYBOOKS_DIR := playbooks

-include .env
export

SITE1_INVENTORY := inventories/site1
SITE2_INVENTORY := inventories/site2

LAN_VM_PORT      ?= 2221
ADMIN_VM_PORT    ?= 2222

BASTION_PORT     ?= 2231
WEBSERVER_PORT   ?= 2232
S2_GATEWAY_HOST  ?= 5.39.122.40
S2_GATEWAY_USER  ?= t-nsa-810-cia

.PHONY: help install \
        site1-pfsense site1-pfsense-rules site1-pfsense-vpn site1-vault site1-lan site1-vpn site1-pki site1-dns \
        site1-killswitch site1-killswitch-restore site2-killswitch site2-killswitch-restore \
        site1-harden site2-harden \
        site1-netbox-token netbox-sync site1-netbox-autosync site1-fluentbit \
        site1-all deploy-all \
        tunnel-netbox tunnel-vault tunnel-opensearch tunnel-all \
        ssh-pfsense ssh-lan ssh-admin \
        ssh-pfsense-s2 ssh-bastion ssh-webserver \
        site2-pfsense site2-pfsense-rules site2-pfsense-nat site2-vpn site2-vpn-client site2-bastion site2-webserver site2-website site2-fluentbit site2-dns site2-all \
        site2-check site2-pfsense-check site2-bastion-check site2-webserver-check \
        ping ping-s2

help:
	@echo "Usage:"
	@echo ""
	@echo "  make install             - Installe collections Ansible + dependances Python"
	@echo ""
	@echo "  --- Site 1 ---"
	@echo "  make site1-pfsense       - Configure pfSense Site 1 (VLANs + interfaces + regles)"
	@echo "  make site1-pfsense-rules - Regles firewall Site 1 uniquement"
	@echo "  make site1-pfsense-vpn   - Configuration VPN pfSense Site 1 uniquement"
	@echo "  make site1-vault         - Installe Vault sur admin_vm Site 1"
	@echo "  make site1-lan           - Installe OpenSearch + NetBox sur lan_vm Site 1"
	@echo "  make site1-vpn           - Genere PKI + configure OpenVPN Site 1"
	@echo "  make site1-pki            - Genere la PKI Site 1 uniquement"
	@echo "  make site1-netbox-token   - Genere le token API NetBox et le stocke dans Vault"
	@echo "  make netbox-sync           - Synchronise NetBox (IPAM, les 2 sites)"
	@echo "  make site1-netbox-autosync - Installe le timer systemd de synchro NetBox sur lan_vm"
	@echo "  make site1-fluentbit       - Installe Fluent Bit sur toutes les VMs Site 1"
	@echo "  make site1-dns             - Configure le DNS Resolver (Unbound) Site 1"
	@echo "  make site1-killswitch      - COUPURE D'URGENCE : isole totalement le Site 1"
	@echo "  make site1-killswitch-restore - Retablit le Site 1 (et pre-cree les regles)"
	@echo "  make site1-harden          - Durcit SSH key-only sur lan_vm/admin_vm Site 1"
	@echo "  make site1-all             - Deploie TOUT le Site 1, etape par etape (voir docs/INSTALL.md)"
	@echo ""
	@echo "  --- Orchestration globale ---"
	@echo "  make deploy-all          - Deploie Site 1 puis Site 2 (infra complete)"
	@echo ""
	@echo "  --- Tunnels SSH ---"
	@echo "  make tunnel-netbox       - Tunnel vers NetBox     (localhost:8443)"
	@echo "  make tunnel-vault        - Tunnel vers Vault      (localhost:8200)"
	@echo "  make tunnel-opensearch   - Tunnel vers OpenSearch (localhost:9200)"
	@echo "  make tunnel-all          - Tous les tunnels en une commande"
	@echo "  make site1-pki           - Genere la PKI Site 1 uniquement"
	@echo ""
	@echo "  --- Site 2 ---"
	@echo "  make site2-pfsense       - Configure pfSense Site 2 (regles + NAT)"
	@echo "  make site2-pfsense-rules - Regles firewall Site 2 uniquement"
	@echo "  make site2-pfsense-nat   - NAT Site 2 uniquement"
	@echo "  make site2-vpn           - Configure OpenVPN client Site 2 (relance aussi la PKI Site 1)"
	@echo "  make site2-vpn-client    - Configure le client VPN Site 2 uniquement (ne touche pas au Site 1)"
	@echo "  make site2-bastion        - Configure le Bastion Host Site 2 (SSH hardening + fail2ban + logging)"
	@echo "  make site2-webserver      - Deploie OpenSearch Dashboards sur le WebServer Site 2"
	@echo "  make site2-website        - Deploie le site web interne (Nginx) sur le WebServer Site 2"
	@echo "  make site2-fluentbit      - Installe Fluent Bit sur les VMs Site 2 (logs vers OpenSearch Site 1)"
	@echo "  make site2-dns            - Configure le DNS Resolver (Unbound) Site 2"
	@echo "  make site2-killswitch     - COUPURE D'URGENCE : isole totalement le Site 2"
	@echo "  make site2-killswitch-restore - Retablit le Site 2 (et pre-cree les regles)"
	@echo "  make site2-harden         - Durcit SSH key-only sur le WebServer Site 2"
	@echo "  make site2-all            - Deploie tout le Site 2 (pfsense + bastion + webserver + fluentbit + dns)"
	@echo ""
	@echo "  --- Dry-run Site 2 (simulation) ---"
	@echo "  make site2-check          - Simule tout le deploiement Site 2"
	@echo "  make site2-pfsense-check  - Simule la config pfSense Site 2"
	@echo "  make site2-bastion-check  - Simule la config Bastion Site 2"
	@echo "  make site2-webserver-check - Simule la config WebServer Site 2"
	@echo ""
	@echo "  --- Connexions SSH Site 2 ---"
	@echo "  make ssh-pfsense-s2      - SSH vers pfSense Site 2"
	@echo "  make ssh-bastion         - SSH vers Bastion     (port $(BASTION_PORT))"
	@echo "  make ssh-webserver       - SSH vers WebServer   (port $(WEBSERVER_PORT))"
	@echo ""
	@echo "  make ping                - Teste la connectivite vers tous les hotes"
	@echo "  make ping-s2             - Teste la connectivite Site 2 uniquement"
	@echo ""
	@echo "  --- Connexions SSH Site 1 ---"
	@echo "  make ssh-pfsense         - SSH vers pfSense Site 1"
	@echo "  make ssh-lan             - SSH vers lan_vm   (port $(LAN_VM_PORT))"
	@echo "  make ssh-admin           - SSH vers admin_vm (port $(ADMIN_VM_PORT))"

install:
	ansible-galaxy collection install -r requirements.yml --upgrade
	python3 -m pip install -r requirements.txt --break-system-packages

# --- Site 1 ---

site1-pfsense:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --skip-tags vpn

site1-pfsense-rules:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags rules

site1-pfsense-nat:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags nat

site1-pfsense-vpn:
	@rm -f ~/.ansible/cp/* 2>/dev/null || true
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@read -p "Token Vault (root) : " vault_token; \
	VAULT_TOKEN=$$vault_token ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags vpn; \
	pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site1-vault:
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vault.yml

site1-lan:
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/lan.yml
	@pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site1-dns:
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/dns.yml

site1-killswitch:
	@echo ">>> COUPURE D'URGENCE Site 1 : isolation totale (recovery: console Proxmox/OVH)"
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/killswitch.yml -e killswitch_enabled=true

site1-killswitch-restore:
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/killswitch.yml -e killswitch_enabled=false

site1-harden:
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/ssh_hardening.yml --limit lan_vm:admin_vm

site1-vpn:
	@rm -f ~/.ansible/cp/* 2>/dev/null || true
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@echo ">>> Token Vault (root) :"
	@read -p "Token Vault (root) : " vault_token; \
	VAULT_TOKEN=$$vault_token ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml; \
	pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site1-pki:
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml --tags pki

site1-netbox-token:
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@read -p "Token Vault admin (root) : " vault_token; \
	VAULT_TOKEN=$$vault_token ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/netbox_token.yml; \
	pkill -f "8443:$(NETBOX_INTERNAL_HOST):443" 2>/dev/null || true

site1-fluentbit:
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/fluent-bit.yml --limit lan_vm
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/fluent-bit.yml --limit admin_vm
	@pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

netbox-sync:
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/netbox_sync.yml; \
	pkill -f "8443:$(NETBOX_INTERNAL_HOST):443" 2>/dev/null || true

site1-netbox-autosync:
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/netbox_autosync.yml; \
	pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

# --- Orchestration Site 1 (deploiement complet, etape par etape) ---
# Reproduit l'ordre documente dans docs/INSTALL.md. Les etapes Vault (init +
# unseal) et les prompts de token restent MANUELS : on s'arrete avant si besoin.

site1-all:
	@echo "============================================================"
	@echo ">>> DEPLOIEMENT COMPLET SITE 1"
	@echo ">>> Pre-requis : .env rempli, cle SSH en place, 'make ping' OK."
	@echo ">>> Detail de chaque etape : docs/INSTALL.md"
	@echo "============================================================"
	@echo ""
	@echo ">>> [1/9] pfSense : VLANs, interfaces, regles firewall + NAT"
	@$(MAKE) site1-pfsense
	@echo ""
	@echo ">>> [2/9] Vault (admin_vm)"
	@$(MAKE) site1-vault
	@echo ""
	@echo "    !!! ACTION MANUELLE REQUISE !!!"
	@echo "    Si Vault vient d'etre installe : l'initialiser et le desceller."
	@echo "    Voir docs/INSTALL.md section 'Initialiser Vault'."
	@read -p "    Vault est initialise ET descelle (Sealed: false) ? [Entree pour continuer] " _ack
	@echo ""
	@echo ">>> [3/9] PKI + OpenVPN site-a-site (token Vault demande)"
	@$(MAKE) site1-vpn
	@echo ""
	@echo ">>> [4/9] OpenSearch + NetBox (lan_vm)"
	@$(MAKE) site1-lan
	@echo ""
	@echo ">>> [5/9] DNS Resolver (Unbound) Site 1"
	@$(MAKE) site1-dns
	@echo ""
	@echo ">>> [6/9] Token API NetBox -> Vault (token Vault demande)"
	@$(MAKE) site1-netbox-token
	@echo ""
	@echo ">>> [7/9] Synchronisation NetBox (IPAM, les 2 sites)"
	@$(MAKE) netbox-sync
	@echo ""
	@echo ">>> [8/9] Timer systemd d'auto-sync NetBox (lan_vm)"
	@$(MAKE) site1-netbox-autosync
	@echo ""
	@echo ">>> [9/9] Fluent Bit (collecte de logs -> OpenSearch)"
	@$(MAKE) site1-fluentbit
	@echo ""
	@echo ">>> Durcissement SSH key-only (lan_vm + admin_vm) — EN DERNIER"
	@$(MAKE) site1-harden
	@echo ""
	@echo ">>> SITE 1 DEPLOYE. Lancez 'make deploy-all' ou 'make site2-all' pour le Site 2."

# --- Orchestration globale (les deux sites) ---
# Le VPN site-a-site doit etre UP (etape site1-vpn + site2-vpn) pour que le
# Site 2 atteigne OpenSearch sur le Site 1.

deploy-all:
	@echo "############################################################"
	@echo "## DEPLOIEMENT INFRASTRUCTURE COMPLETE (Site 1 + Site 2)  ##"
	@echo "############################################################"
	@$(MAKE) site1-all
	@echo ""
	@echo ">>> Bascule sur le SITE 2..."
	@echo "    (le tunnel VPN cote Site 2 est etabli par 'make site2-vpn')"
	@$(MAKE) site2-vpn
	@$(MAKE) site2-all
	@$(MAKE) site2-harden
	@echo ""
	@echo "############################################################"
	@echo "## INFRASTRUCTURE DEPLOYEE. Verifs : docs/INSTALL.md §10   ##"
	@echo "############################################################"

# --- Site 2 ---

site2-pfsense:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --skip-tags vpn

site2-pfsense-rules:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --tags rules

site2-pfsense-nat:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --tags nat

site2-vpn:
	@rm -f ~/.ansible/cp/* 2>/dev/null || true
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@echo ">>> Token Vault (root) :"
	@read -p "Token Vault (root) : " vault_token; \
	VAULT_TOKEN=$$vault_token ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml; \
	VAULT_TOKEN=$$vault_token ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --tags vpn; \
	pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site2-vpn-client:
	@rm -f ~/.ansible/cp/* 2>/dev/null || true
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@read -p "Token Vault (root) : " vault_token; \
	VAULT_TOKEN=$$vault_token ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --tags vpn; \
	pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site2-bastion:
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/bastion_s2.yml

site2-webserver:
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/webserver_s2.yml

site2-website:
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/website_s2.yml

site2-fluentbit:
	@ssh -f -N -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/fluent-bit_s2.yml
	@pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site2-dns:
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/dns.yml

site2-killswitch:
	@echo ">>> COUPURE D'URGENCE Site 2 : isolation totale (recovery: console + bastion)"
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/killswitch.yml -e killswitch_enabled=true

site2-killswitch-restore:
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/killswitch.yml -e killswitch_enabled=false

site2-harden:
	@ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/ssh_hardening.yml --limit webserver_vm

site2-all:
	@echo ">>> Deploiement complet Site 2 (pfsense → bastion → webserver → website → fluentbit → dns)"
	@$(MAKE) site2-pfsense
	@$(MAKE) site2-bastion
	@$(MAKE) site2-webserver
	@$(MAKE) site2-website
	@$(MAKE) site2-fluentbit
	@$(MAKE) site2-dns

site2-check:
	@$(MAKE) site2-pfsense-check
	@$(MAKE) site2-bastion-check
	@$(MAKE) site2-webserver-check

site2-pfsense-check:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --check --diff --skip-tags vpn

site2-bastion-check:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/bastion_s2.yml --check --diff

site2-webserver-check:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/webserver_s2.yml --check --diff

# --- Global ---

tunnel-netbox:
	@echo "Tunnel NetBox : https://localhost:8443 -> $(NETBOX_INTERNAL_HOST):443  (Ctrl+C pour fermer)"
	ssh -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

tunnel-vault:
	@echo "Tunnel Vault : https://localhost:8200 -> $(VAULT_INTERNAL_HOST):8200  (Ctrl+C pour fermer)"
	ssh -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

tunnel-opensearch:
	@echo "Tunnel OpenSearch : https://localhost:9200 -> $(NETBOX_INTERNAL_HOST):9200  (Ctrl+C pour fermer)"
	ssh -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 9200:$(NETBOX_INTERNAL_HOST):9200 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

tunnel-all:
	@echo "Tunnels ouverts :"
	@echo "  https://localhost:8443  -> NetBox     ($(NETBOX_INTERNAL_HOST):443)"
	@echo "  https://localhost:8200  -> Vault      ($(VAULT_INTERNAL_HOST):8200)"
	@echo "  https://localhost:9200  -> OpenSearch ($(NETBOX_INTERNAL_HOST):9200)"
	@echo "Ctrl+C pour tout fermer."
	ssh -p $(ADMIN_VM_PORT) -i $(SSH_KEY) \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-L 9200:$(NETBOX_INTERNAL_HOST):9200 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

ping:
	ansible all -i $(SITE1_INVENTORY) -m ping
	ansible all -i $(SITE2_INVENTORY) -m ping

ping-s2:
	ansible all -i $(SITE2_INVENTORY) -m ping

# --- Connexions SSH Site 1 ---

ssh-pfsense:
	ssh -p $(GATEWAY_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no admin@$(GATEWAY_HOST)

ssh-lan:
	ssh -p $(LAN_VM_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no sysadmin@$(GATEWAY_HOST)

ssh-admin:
	ssh -p $(ADMIN_VM_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no sysadmin@$(GATEWAY_HOST)

# --- Connexions SSH Site 2 ---

ssh-pfsense-s2:
	ssh -p 22 -i $(SSH_KEY) -o StrictHostKeyChecking=no admin@$(S2_GATEWAY_HOST)

ssh-bastion:
	ssh -p $(BASTION_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no $(S2_GATEWAY_USER)@$(S2_GATEWAY_HOST)

ssh-webserver:
	ssh -p $(WEBSERVER_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no $(S2_GATEWAY_USER)@$(S2_GATEWAY_HOST)