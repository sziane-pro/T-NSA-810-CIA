ANSIBLE_CONFIG := $(shell pwd)/ansible.cfg
export ANSIBLE_CONFIG

INVENTORY := inventories/inventory
PLAYBOOKS_DIR := playbooks

-include .env
export

.PHONY: help install pfsense ping vpn pki

help:
	@echo "Usage:"
	@echo "  make install       - Install all Ansible collections + Python dependencies"
	@echo "  make pfsense       - Run pfsense_network playbook"
	@echo "  make pfsense-rules - Run only firewall rules"
	@echo "  make pfsense-vpn   - Run only VPN rules on pfSense"
	@echo "  make vault         - Install Vault on admin_vm"
	@echo "  make lan           - Install OpenSearch + Netbox on lan_vm"
	@echo "  make vpn           - Generate PKI + configure OpenVPN on pfSense"
	@echo "  make pki           - Generate PKI only (on admin_vm)"
	@echo "  make ping          - Test connectivity to all hosts"

install:
	ansible-galaxy collection install -r requirements.yml
	python3 -m pip install -r requirements.txt --break-system-packages

pfsense:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml

pfsense-rules:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags rules

pfsense-vpn:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags vpn

vault:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/vault.yml --ask-become-pass

lan:
	@ssh -f -N -p $(GATEWAY_PORT) -i $(SSH_KEY) \
		-L 8200:192.168.30.101:8200 \
		-o ExitOnForwardFailure=yes \
		-o StrictHostKeyChecking=no \
		$(GATEWAY_USER)@$(GATEWAY_HOST) 2>/dev/null || true
	@sleep 1
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/lan.yml --ask-become-pass; \
	pkill -f "8200:192.168.30.101:8200" 2>/dev/null || true

vpn:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml --ask-become-pass

pki:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/vpn.yml --ask-become-pass --tags pki

ping:
	ansible all -i $(INVENTORY) -m ping
