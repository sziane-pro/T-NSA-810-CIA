ANSIBLE_CONFIG := $(shell pwd)/ansible.cfg
export ANSIBLE_CONFIG

PLAYBOOKS_DIR := playbooks

-include .env
export

SITE1_INVENTORY := inventories/site1
SITE2_INVENTORY := inventories/site2

.PHONY: help install \
        site1-pfsense site1-pfsense-rules site1-pfsense-vpn site1-vault site1-lan site1-vpn site1-pki \
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
	@echo "  make site1-pki           - Genere la PKI Site 1 uniquement"
	@echo ""
	@echo "  make ping                - Teste la connectivite vers tous les hotes"

install:
	ansible-galaxy collection install -r requirements.yml
	python3 -m pip install -r requirements.txt --break-system-packages

# --- Site 1 ---

site1-pfsense:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml

site1-pfsense-rules:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags rules

site1-pfsense-vpn:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags vpn

site1-vault:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vault.yml --ask-become-pass

site1-lan:
	@ssh -f -N -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8200:192.168.30.101:8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/lan.yml --ask-become-pass; \
	pkill -f "8200:192.168.30.101:8200" 2>/dev/null || true

site1-vpn:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml --ask-become-pass

site1-pki:
	ansible-playbook -i $(SITE1_INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml --ask-become-pass --tags pki

# --- Site 2 ---
# Les cibles Site 2 sont a implementer par l'equipe Site 2.
# L'inventaire est disponible dans inventories/site2/ avec les variables pre-remplies.
# Pour ajouter les cibles, suivre le meme modele que Site 1 en remplacant
# $(SITE1_INVENTORY) par $(SITE2_INVENTORY).
#
# Exemple :
#
# site2-pfsense:
# 	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml
#
# site2-pfsense-rules:
# 	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags rules
#
# site2-lan:
# 	ansible-playbook -i $(SITE2_INVENTORY) $(PLAYBOOKS_DIR)/lan.yml --ask-become-pass

# --- Global ---

ping:
	ansible all -i $(SITE1_INVENTORY) -m ping
	ansible all -i $(SITE2_INVENTORY) -m ping
