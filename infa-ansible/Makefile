ANSIBLE_CONFIG := $(shell pwd)/ansible.cfg
export ANSIBLE_CONFIG

INVENTORY := inventories/inventory
PLAYBOOKS_DIR := playbooks

.PHONY: help pfsense ping

help:
	@echo "Usage:"
	@echo "  make pfsense       - Run pfsense_network playbook"
	@echo "  make pfsense-rules - Run only firewall rules"
	@echo "  make vault         - Install Vault on admin_vm"
	@echo "  make lan           - Install OpenSearch + Netbox on lan_vm"
	@echo "  make ping          - Test connectivity to all hosts"

pfsense:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml

pfsense-rules:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/pfsense_network.yml --tags rules

vault:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/vault.yml --ask-become-pass

lan:
	ansible-playbook -i $(INVENTORY) $(PLAYBOOKS_DIR)/lan.yml --ask-become-pass

ping:
	ansible all -i $(INVENTORY) -m ping
