# =============================================================================
# Makefile — convenience targets for the VPS benchmark Ansible playbook
# Usage: make <target> [HOST=<host>] [EXTRA="-e key=val"]
# =============================================================================

PLAYBOOK      ?= benchmark.yml
INVENTORY     ?= inventory/hosts.yml
HOST          ?= all
EXTRA         ?=
ANSIBLE_ARGS  ?=

MOLECULE_SCENARIO ?= default

_HOST  = $(if $(HOST),--limit $(HOST),)
_EXTRA = $(if $(EXTRA),$(EXTRA),)
COMMON_ARGS = -i $(INVENTORY) $(_HOST) $(_EXTRA) $(ANSIBLE_ARGS)

.DEFAULT_GOAL := help

# -----------------------------------------------------------------------------
# Help
# -----------------------------------------------------------------------------
.PHONY: help
help: ## Show this help message
	@echo ""
	@echo "VPS Benchmark — available targets:"
	@echo ""
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) \
		| awk 'BEGIN {FS = ":.*?## "}; {printf "  \033[36m%-22s\033[0m %s\n", $$1, $$2}'
	@echo ""

# -----------------------------------------------------------------------------
# Dependencies
# -----------------------------------------------------------------------------
.PHONY: install-deps
install-deps: ## Install Python deps for Ansible + Molecule
	pip install --upgrade molecule molecule-docker ansible-lint

# -----------------------------------------------------------------------------
# Syntax & lint
# -----------------------------------------------------------------------------
.PHONY: syntax lint
syntax: ## Check playbook syntax
	ansible-playbook $(PLAYBOOK) --syntax-check

lint: ## Lint the playbook and role with ansible-lint
	ansible-lint $(PLAYBOOK)

# -----------------------------------------------------------------------------
# Run benchmark
# -----------------------------------------------------------------------------
.PHONY: run
run: ## Run the full benchmark playbook (default: localhost)
	ansible-playbook $(PLAYBOOK) -v $(COMMON_ARGS)

run-no-netbench: ## Run benchmark without the network test
	ansible-playbook $(PLAYBOOK) -v $(COMMON_ARGS) -e vps_benchmark_run_netbench=false

# -----------------------------------------------------------------------------
# Molecule
# -----------------------------------------------------------------------------
.PHONY: molecule-test molecule-converge molecule-verify molecule-destroy
molecule-test: ## Run full molecule test suite (create → converge → verify → destroy)
	cd "$(CURDIR)" && molecule test -s $(MOLECULE_SCENARIO)

molecule-converge: ## Molecule converge only (create + apply role)
	cd "$(CURDIR)" && molecule converge -s $(MOLECULE_SCENARIO)

molecule-verify: ## Molecule verify only
	cd "$(CURDIR)" && molecule verify -s $(MOLECULE_SCENARIO)

molecule-destroy: ## Molecule destroy (clean up containers)
	cd "$(CURDIR)" && molecule destroy -s $(MOLECULE_SCENARIO)
