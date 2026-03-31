# =============================================================================
# Makefile — convenience targets for the Debian upgrade Ansible playbook
# Usage: make <target> [VERSION=12|13] [LIMIT=host] [TAGS=tag] [EXTRA="-e k=v"]
# =============================================================================

INVENTORY     ?= inventory/secret_hosts.yml
PLAYBOOK      ?= upgrade_debian.yml
REQUIREMENTS  ?= requirements.yml
LIMIT         ?= debian_hosts
VERSION       ?= 13
TAGS          ?=
SKIP_TAGS     ?=
EXTRA         ?=
ANSIBLE_ARGS  ?=

# Build optional flags dynamically
_LIMIT     = $(if $(LIMIT),--limit $(LIMIT),)
_TAGS      = $(if $(TAGS),--tags $(TAGS),)
_SKIP_TAGS = $(if $(SKIP_TAGS),--skip-tags $(SKIP_TAGS),)
_EXTRA     = $(if $(EXTRA),$(EXTRA),)
_VERSION   = -e debian_upgrade_target_version=$(VERSION)

COMMON_ARGS = -i $(INVENTORY) $(_LIMIT) $(_TAGS) $(_SKIP_TAGS) $(_VERSION) $(_EXTRA) $(ANSIBLE_ARGS)

.DEFAULT_GOAL := help

# -----------------------------------------------------------------------------
# Help
# -----------------------------------------------------------------------------
.PHONY: help
help: ## Show this help message
	@echo ""
	@echo "  Debian Major Version Upgrade Playbook"
	@echo ""
	@echo "  Usage:  make <target> [VERSION=12|13] [LIMIT=host] [TAGS=tag] [EXTRA='-e key=val']"
	@echo ""
	@echo "  VERSION controls which Debian release to upgrade TO (default: $(VERSION))."
	@echo "  The codename is resolved automatically from the codename map."
	@echo ""
	@awk 'BEGIN {FS = ":.*##"; section=""} \
	      /^## / {section=substr($$0,4); next} \
	      /^[a-zA-Z_0-9-]+:.*##/ { \
	        printf "  \033[36m%-28s\033[0m %s\n", $$1, $$2 \
	      }' $(MAKEFILE_LIST)
	@echo ""
	@echo "  Examples:"
	@echo "    make check                                      # dry-run upgrade to $(VERSION)"
	@echo "    make check VERSION=12                           # dry-run upgrade to 12 (bookworm)"
	@echo "    make run VERSION=13                             # upgrade to 13 (trixie)"
	@echo "    make run VERSION=12 LIMIT=debian-server-01      # upgrade one host to 12"
	@echo "    make run-no-reboot VERSION=13                   # upgrade without auto-reboot"
	@echo "    make run VERSION=13 EXTRA='-e debian_upgrade_allow_skip_release=true'  # skip-release"
	@echo ""

## Galaxy
.PHONY: install
install: install-roles install-collections ## Install all Galaxy roles AND collections

.PHONY: install-roles
install-roles: ## Install Galaxy roles from requirements.yml
	ansible-galaxy role install -r $(REQUIREMENTS)

.PHONY: install-collections
install-collections: ## Install Galaxy collections from requirements.yml
	ansible-galaxy collection install -r $(REQUIREMENTS)

.PHONY: install-force
install-force: ## Force-reinstall all Galaxy roles and collections (overwrite existing)
	ansible-galaxy role install -r $(REQUIREMENTS) --force
	ansible-galaxy collection install -r $(REQUIREMENTS) --force

## Connectivity
.PHONY: ping
ping: ## Ping all hosts in the inventory group (tests SSH + Python)
	ansible -i $(INVENTORY) $(LIMIT) -m ping

.PHONY: facts
facts: ## Gather and display facts from target hosts
	ansible -i $(INVENTORY) $(LIMIT) -m ansible.builtin.setup \
	  -a "gather_subset=distribution,distribution_version,distribution_release,mounts"

.PHONY: os-check
os-check: ## Show current OS version on all target hosts
	ansible -i $(INVENTORY) $(LIMIT) -m ansible.builtin.command -a "cat /etc/os-release"

## Playbook — version-aware targets
.PHONY: check
check: ## Dry-run upgrade to VERSION (--check --diff)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) --check --diff

.PHONY: diff
diff: check ## Alias for 'check' (dry-run with diff output)

.PHONY: run
run: ## Run the full upgrade to VERSION (applies real changes)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS)

.PHONY: run-verbose
run-verbose: ## Run the full upgrade to VERSION with verbose output (-vvv)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) -vvv

.PHONY: run-no-reboot
run-no-reboot: ## Upgrade to VERSION but skip automatic reboot
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) -e "debian_upgrade_allow_reboot=false"

## Version-specific shortcuts
.PHONY: upgrade-to-12
upgrade-to-12: ## Shortcut: upgrade to Debian 12 (bookworm)
	$(MAKE) run VERSION=12

.PHONY: upgrade-to-13
upgrade-to-13: ## Shortcut: upgrade to Debian 13 (trixie)
	$(MAKE) run VERSION=13

.PHONY: upgrade-to-14
upgrade-to-14: ## Shortcut: upgrade to Debian 14 (forky)
	$(MAKE) run VERSION=14

.PHONY: check-12
check-12: ## Shortcut: dry-run upgrade to Debian 12 (bookworm)
	$(MAKE) check VERSION=12

.PHONY: check-13
check-13: ## Shortcut: dry-run upgrade to Debian 13 (trixie)
	$(MAKE) check VERSION=13

.PHONY: check-14
check-14: ## Shortcut: dry-run upgrade to Debian 14 (forky)
	$(MAKE) check VERSION=14

## Tag-based execution
.PHONY: tags
tags: ## Run only specific stages. Usage: make tags TAGS=preflight,sources
	@test -n "$(TAGS)" || (echo "Error: TAGS is required. Example: make tags TAGS=preflight" && exit 1)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS)

.PHONY: skip
skip: ## Skip specific stages. Usage: make skip SKIP_TAGS=reboot
	@test -n "$(SKIP_TAGS)" || (echo "Error: SKIP_TAGS is required. Example: make skip SKIP_TAGS=reboot" && exit 1)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS)

## Staged execution
.PHONY: preflight
preflight: ## Run only pre-flight checks for VERSION (no changes)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) --tags preflight --check

.PHONY: pre-upgrade
pre-upgrade: ## Run only the pre-upgrade package update stage
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) --tags pre-upgrade

.PHONY: sources
sources: ## Rewrite APT sources to target release only (no dist-upgrade)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) --tags sources

.PHONY: dist-upgrade
dist-upgrade: ## Run the dist-upgrade stage only (sources must already be updated)
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) --tags dist-upgrade

.PHONY: verify
verify: ## Run post-upgrade verification only
	ansible-playbook $(PLAYBOOK) $(COMMON_ARGS) --tags verify

## Reboot helpers
.PHONY: reboot-hosts
reboot-hosts: ## Reboot target hosts and wait for them to come back
	ansible -i $(INVENTORY) $(LIMIT) -m ansible.builtin.reboot \
	  -a "reboot_timeout=600 pre_reboot_delay=5 post_reboot_delay=30 test_command=uptime" \
	  --become

## Linting
.PHONY: lint
lint: ## Lint the playbook and role with ansible-lint
	ansible-lint $(PLAYBOOK)

.PHONY: syntax-check
syntax-check: ## Run Ansible syntax check (no SSH needed)
	ansible-playbook $(PLAYBOOK) -i $(INVENTORY) --syntax-check

## Cleanup
.PHONY: clean-galaxy
clean-galaxy: ## Remove locally installed Galaxy roles and collections
	@echo "Removing installed roles..."
	@for role in robertdebock.update robertdebock.reboot robertdebock.bootstrap; do \
	  ansible-galaxy role remove $$role 2>/dev/null || true; \
	done
	@echo "Removing installed collections..."
	@for col in community.general ansible.posix; do \
	  ansible-galaxy collection remove $$col 2>/dev/null || true; \
	done
	@echo "Done."

.PHONY: clean-retry
clean-retry: ## Remove any leftover .retry files from failed playbook runs
	find . -name "*.retry" -delete
	@echo "Removed all .retry files."