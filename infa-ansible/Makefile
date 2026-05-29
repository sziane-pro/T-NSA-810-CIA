ANSIBLE_CONFIG := $(shell pwd)/ansible.cfg
export ANSIBLE_CONFIG

PLAYBOOKS_DIR := playbooks

-include .env
export

SITE1_INVENTORY := inventories/site1
SITE2_INVENTORY := inventories/site2

LAN_VM_PORT   ?= 2221
ADMIN_VM_PORT ?= 2222

.PHONY: help install \
        site1-pfsense site1-pfsense-rules site1-pfsense-vpn site1-vault site1-lan site1-vpn site1-pki \
        site1-netbox-token site1-netbox-populate site1-filebeat \
        tunnel-netbox tunnel-vault tunnel-opensearch tunnel-all \
        ssh-pfsense ssh-lan ssh-admin \
        ping

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
	@echo "  make site1-netbox-populate - Peuple NetBox avec l'infrastructure Site 1"
	@echo "  make site1-filebeat        - Installe Filebeat sur toutes les VMs Site 1"
	@echo ""
	@echo "  --- Tunnels SSH ---"
	@echo "  make tunnel-netbox       - Tunnel vers NetBox     (localhost:8443)"
	@echo "  make tunnel-vault        - Tunnel vers Vault      (localhost:8200)"
	@echo "  make tunnel-opensearch   - Tunnel vers OpenSearch (localhost:9200)"
	@echo "  make tunnel-all          - Tous les tunnels en une commande"
	@echo ""
	@echo "  make ping                - Teste la connectivite vers tous les hotes"
	@echo ""
	@echo "  --- Connexions SSH ---"
	@echo "  make ssh-pfsense         - SSH vers pfSense"
	@echo "  make ssh-lan             - SSH vers lan_vm  (port $(LAN_VM_PORT))"
	@echo "  make ssh-admin           - SSH vers admin_vm (port $(ADMIN_VM_PORT))"

install:
	ansible-galaxy collection install -r requirements.yml --upgrade
	python3 -m pip install -r requirements.txt --break-system-packages

# --- Site 1 ---

site1-pfsense:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml

site1-pfsense-rules:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags rules

site1-pfsense-vpn:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags vpn

site1-vault:
	@echo ">>> Mot de passe sudo de admin_vm :"
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vault.yml --ask-become-pass

site1-lan:
	@ssh -f -N -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@echo ">>> Mot de passe sudo de lan_vm :"
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/lan.yml --ask-become-pass
	@pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site1-vpn:
	@echo ">>> Mot de passe sudo de admin_vm :"
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml --ask-become-pass

site1-pki:
	@echo ">>> Mot de passe sudo de admin_vm :"
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml --ask-become-pass --tags pki

site1-netbox-token:
	@ssh -f -N -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@read -p "Token Vault admin (root) : " vault_token; \
	VAULT_TOKEN=$$vault_token ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/netbox_token.yml; \
	pkill -f "8443:$(NETBOX_INTERNAL_HOST):443" 2>/dev/null || true

site1-filebeat:
	@ssh -f -N -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	@echo ">>> Mot de passe sudo de lan_vm :"
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/fluent-bit.yml --ask-become-pass --limit lan_vm
	@echo ">>> Mot de passe sudo de admin_vm :"
	@ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/fluent-bit.yml --ask-become-pass --limit admin_vm
	@pkill -f "8200:$(VAULT_INTERNAL_HOST):8200" 2>/dev/null || true

site1-netbox-populate:
	@ssh -f -N -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/netbox_populate.yml; \
	pkill -f "8443:$(NETBOX_INTERNAL_HOST):443" 2>/dev/null || true

# --- Site 2 ---

site2-pfsense:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml

site2-pfsense-rules:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --tags rules

site2-pfsense-nat:
	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network_s2.yml --tags nat

# --- Global ---

tunnel-netbox:
	@echo "Tunnel NetBox : https://localhost:8443 -> $(NETBOX_INTERNAL_HOST):443  (Ctrl+C pour fermer)"
	ssh -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

tunnel-vault:
	@echo "Tunnel Vault : https://localhost:8200 -> $(VAULT_INTERNAL_HOST):8200  (Ctrl+C pour fermer)"
	ssh -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

tunnel-opensearch:
	@echo "Tunnel OpenSearch : https://localhost:9200 -> $(NETBOX_INTERNAL_HOST):9200  (Ctrl+C pour fermer)"
	ssh -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 9200:$(NETBOX_INTERNAL_HOST):9200 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

tunnel-all:
	@echo "Tunnels ouverts :"
	@echo "  https://localhost:8443  -> NetBox     ($(NETBOX_INTERNAL_HOST):443)"
	@echo "  https://localhost:8200  -> Vault      ($(VAULT_INTERNAL_HOST):8200)"
	@echo "  https://localhost:9200  -> OpenSearch ($(NETBOX_INTERNAL_HOST):9200)"
	@echo "Ctrl+C pour tout fermer."
	ssh -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8443:$(NETBOX_INTERNAL_HOST):443 \
		-L 8200:$(VAULT_INTERNAL_HOST):8200 \
		-L 9200:$(NETBOX_INTERNAL_HOST):9200 \
		-o StrictHostKeyChecking=no \
		-N $(GATEWAY_USER)@$(GATEWAY_HOST)

ping:
	ansible all -i $(SITE1_INVENTORY) -m ping
	ansible all -i $(SITE2_INVENTORY) -m ping

# --- Connexions SSH ---

ssh-pfsense:
	ssh -p $(GATEWAY_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no $(GATEWAY_USER)@$(GATEWAY_HOST)

ssh-lan:
	ssh -p $(LAN_VM_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no sysadmin@$(GATEWAY_HOST)

ssh-admin:
	ssh -p $(ADMIN_VM_PORT) -i $(SSH_KEY) -o StrictHostKeyChecking=no sysadmin@$(GATEWAY_HOST)