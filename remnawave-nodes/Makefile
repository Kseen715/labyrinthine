VENV      ?= ../.venv
PYTHON    ?= python3
AP        := $(VENV)/bin/ansible-playbook
GALAXY    := $(VENV)/bin/ansible-galaxy
MOLECULE  := $(VENV)/bin/molecule
LINT      := $(VENV)/bin/ansible-lint

.PHONY: install nodes monitoring test lint

install:
	$(GALAXY) collection install -r requirements.yml

nodes:
	$(AP) nodes.yml

monitoring:
	$(AP) monitoring.yml

## Run one node: make node HOST=node1
node:
	$(AP) nodes.yml --limit $(HOST)

test:
	cd $(CURDIR) && $(MOLECULE) test

test-monitoring:
	cd $(CURDIR) && $(MOLECULE) test -s monitoring

test-node:
	cd $(CURDIR) && $(MOLECULE) test -s node

lint:
	$(LINT) nodes.yml monitoring.yml
