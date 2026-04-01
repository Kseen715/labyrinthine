# Debian Upgrade Playbook

Ansible playbook to automatically upgrade **Debian** to a chosen target release (default: **Debian 13 — Trixie**) using roles from Ansible Galaxy. The target version is configurable and the role enforces safe, sequential upgrade paths.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Inventory Setup](#inventory-setup)
- [Variables Reference](#variables-reference)
- [Upgrade Path Safety](#upgrade-path-safety)
- [Tags](#tags)
- [Galaxy Dependencies](#galaxy-dependencies)
- [Upgrade Process](#upgrade-process)
- [Examples](#examples)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [License](#license)

---

## Overview

This playbook performs a safe, automated major version upgrade of Debian Linux. It:

1. Resolves the target codename automatically from `debian_upgrade_target_version` using a built-in codename map
2. Validates the upgrade path — skips hosts already at or above the target, blocks downgrades, and enforces sequential (N→N+1) upgrades by default
3. Fully updates all packages on the current version before switching sources
4. Backs up `/etc/apt/sources.list` and all `sources.list.d` files
5. Rewrites APT sources to point to the target version's repositories
6. Runs `apt-get upgrade` followed by `apt-get dist-upgrade`
7. Removes obsolete packages with `autoremove`
8. Reboots the host (configurable) and waits for it to return
9. Verifies the upgraded OS version and APT health post-reboot

Upgrades are performed **one host at a time** (`serial: 1`) to prevent simultaneous mass outages.

---

## Requirements

| Requirement | Version |
|---|---|
| Ansible | >= 2.14 |
| Python | >= 3.9 |
| Target OS | Debian 8+ (any version below the chosen target) |
| Target user | `root` or a user with passwordless `sudo` |
| Free disk space | >= 4 GB on `/` (configurable) |

Install Ansible on the control node:

```
pip install "ansible>=2.14" ansible-lint
```

---

## Project Structure

```
debian-upgrade/
├── requirements.yml                   # Ansible Galaxy roles
├── upgrade_debian.yml                 # Main playbook
├── Makefile                           # Convenience targets
├── inventory/
│   └── hosts.yml                      # Inventory example
└── roles/
    └── debian_upgrade/                # Custom upgrade role
        ├── defaults/
        │   └── main.yml               # All configurable variables with defaults
        ├── handlers/
        │   └── main.yml               # Reboot & apt-cache handlers
        ├── meta/
        │   └── main.yml               # Galaxy metadata & role dependencies
        ├── tasks/
        │   └── main.yml               # Full upgrade task list
        ├── templates/
        │   └── sources.list.j2        # Jinja2 template for new APT sources
        └── molecule/
            ├── requirements.txt       # Molecule test dependencies
            ├── default/               # 12→13 sources rewrite scenario
            ├── already_upgraded/      # Skip-on-target scenario
            ├── skip_release/          # 11→13 blocked scenario
            └── downgrade_protection/  # 13→12 skip scenario
```

---

## Quick Start

### 1. Install Galaxy dependencies

```
ansible-galaxy role install -r requirements.yml
```

### 1a. (Alternative) Use the Makefile

```
make install        # installs Galaxy roles
make install-force  # force-reinstall (useful after requirements.yml changes)
```

### 2. Configure your inventory

Edit `inventory/hosts.yml` and replace the example IPs with your actual hosts:

```
debian_hosts:
  hosts:
    my-server:
      ansible_host: 192.168.1.10
      ansible_user: root
```

### 3. Test connectivity

```
ansible -i inventory/hosts.yml debian_hosts -m ping
# or via Makefile:
make ping INVENTORY=inventory/hosts.yml
```

### 4. Dry-run (no changes)

```
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml --check --diff
# or via Makefile:
make check INVENTORY=inventory/hosts.yml
```

### 5. Run the upgrade

```
# Upgrade to Debian 13 (default)
make run

# Upgrade to Debian 12
make run VERSION=12

# Upgrade to Debian 14
make run VERSION=14

# Or use ansible-playbook directly:
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_target_version=12"
```

---

## Makefile

A `Makefile` is included for convenience. All targets accept optional overrides:

```
INVENTORY   — path to inventory file  (default: inventory/secret_hosts.yml)
LIMIT       — host or group to target (default: debian_hosts)
VERSION     — target Debian version   (default: 13)
TAGS        — comma-separated tags    (default: none)
SKIP_TAGS   — tags to skip            (default: none)
EXTRA       — extra -e arguments      (default: none)
```

| Target | Description |
|---|---|
| `make install` | Install Galaxy roles |
| `make install-roles` | Install Galaxy roles only |
| `make install-force` | Force-reinstall everything |
| `make ping` | Test SSH connectivity (`ansible -m ping`) |
| `make facts` | Show OS facts from all hosts |
| `make os-check` | Print `/etc/os-release` from all hosts |
| `make check` | Dry-run with `--check --diff` |
| `make check VERSION=12` | Dry-run targeting Debian 12 |
| `make check-12` | Dry-run targeting Debian 12 (shorthand) |
| `make check-13` | Dry-run targeting Debian 13 (shorthand) |
| `make check-14` | Dry-run targeting Debian 14 (shorthand) |
| `make run` | Apply the full upgrade playbook (default: Debian 13) |
| `make run VERSION=12` | Apply the full upgrade to Debian 12 |
| `make run VERSION=14` | Apply the full upgrade to Debian 14 |
| `make upgrade-to-12` | Upgrade to Debian 12 (shorthand) |
| `make upgrade-to-13` | Upgrade to Debian 13 (shorthand) |
| `make upgrade-to-14` | Upgrade to Debian 14 (shorthand) |
| `make run-no-reboot` | Full upgrade, skip automatic reboot |
| `make preflight` | Run pre-flight checks only (check mode) |
| `make sources` | Rewrite APT sources to target version only |
| `make dist-upgrade` | Run dist-upgrade stage only |
| `make verify` | Post-upgrade verification only |
| `make reboot-hosts` | Manually reboot hosts and wait |
| `make lint` | Lint with `ansible-lint` |
| `make syntax-check` | Syntax check (no SSH needed) |
| `make clean-galaxy` | Remove installed roles |
| `make test` | Run all Molecule test scenarios |
| `make test-default` | Run the `default` Molecule scenario (12→13) |
| `make test-already-upgraded` | Run the `already_upgraded` scenario |
| `make test-skip-release` | Run the `skip_release` scenario (11→13 blocked) |
| `make test-downgrade` | Run the `downgrade_protection` scenario |
| `make test-converge` | Run Molecule converge only (no verify/destroy) |
| `make test-lint` | Run linting inside the Molecule environment |

### Examples

```
# Dry-run on a single host
make check LIMIT=debian-server-01

# Upgrade a single host to Debian 12
make run VERSION=12 LIMIT=debian-server-01

# Full upgrade to Debian 13, no reboot
make run LIMIT=debian-server-01 EXTRA='-e debian_upgrade_allow_reboot=false'

# Only rewrite sources (skip upgrade)
make tags TAGS=sources LIMIT=debian-server-01

# Use a custom inventory
make run INVENTORY=inventory/secret_hosts.yml

# Run all Molecule tests
make test
```

---

## Inventory Setup

### Basic inventory (`inventory/hosts.yml`)

```
all:
  vars:
    ansible_user: root
    ansible_python_interpreter: /usr/bin/python3

  children:
    debian_hosts:
      hosts:
        debian-server-01:
          ansible_host: 192.168.1.10
        debian-server-02:
          ansible_host: 192.168.1.11
```

### Production inventory (manual reboot)

```
debian_hosts_production:
  vars:
    debian_upgrade_allow_reboot: false  # reboot manually at a maintenance window
  hosts:
    prod-01:
      ansible_host: 10.0.0.10
```

### Using SSH jump host

```
all:
  vars:
    ansible_ssh_common_args: "-o ProxyJump=jumphost.example.com"
```

---

## Variables Reference

All variables are defined in `roles/debian_upgrade/defaults/main.yml` and can be overridden via inventory `group_vars`, `host_vars`, playbook `vars`, or `-e` on the command line.

### Target version

| Variable | Default | Description |
|---|---|---|
| `debian_upgrade_target_version` | `"13"` | Target Debian major version (primary input) |
| `debian_upgrade_target_codename` | *(auto-derived)* | Target Debian codename — automatically derived from `debian_upgrade_codenames[debian_upgrade_target_version]`. Normally you do not need to set this manually. |
| `debian_upgrade_codenames` | `{8: jessie, 9: stretch, 10: buster, 11: bullseye, 12: bookworm, 13: trixie, 14: forky}` | Codename lookup map used to resolve `debian_upgrade_target_codename` from the version number |
| `debian_upgrade_allow_skip_release` | `false` | Allow skipping releases (e.g. 11→13). When `false`, only sequential N→N+1 upgrades are permitted. |

### APT sources

| Variable | Default | Description |
|---|---|---|
| `debian_upgrade_apt_mirror` | `deb.debian.org` | Main APT mirror hostname |
| `debian_upgrade_apt_security_mirror` | `security.debian.org` | Security APT mirror |
| `debian_upgrade_apt_components` | `main contrib non-free non-free-firmware` | APT components |
| `debian_upgrade_sources_list_path` | `/etc/apt/sources.list` | Path to sources.list |
| `debian_upgrade_sources_list_d_path` | `/etc/apt/sources.list.d` | Path to sources.list.d |

### Backup

| Variable | Default | Description |
|---|---|---|
| `debian_upgrade_backup_sources` | `true` | Backup sources before modification |
| `debian_upgrade_backup_path` | `/etc/apt/sources.list.bak` | Backup destination |

### Upgrade behaviour

| Variable | Default | Description |
|---|---|---|
| `debian_upgrade_pre_update` | `true` | Fully update packages before switching sources |
| `debian_upgrade_dist_upgrade` | `true` | Run `dist-upgrade` (as opposed to `safe-upgrade` only) |
| `debian_upgrade_autoremove` | `true` | Remove obsolete packages after upgrade |
| `debian_upgrade_autoclean` | `true` | Clean apt cache after upgrade |
| `debian_upgrade_fail_if_already_upgraded` | `false` | Fail (vs. skip) if already on target version |
| `debian_upgrade_held_packages` | `[]` | Packages to mark as held before upgrade |
| `debian_upgrade_extra_packages_pre` | `[]` | Packages to install before upgrade |
| `debian_upgrade_extra_packages_post` | `[]` | Packages to install after upgrade |

### Reboot

| Variable | Default | Description |
|---|---|---|
| `debian_upgrade_allow_reboot` | `true` | Automatically reboot after dist-upgrade |
| `debian_upgrade_reboot_timeout` | `600` | Seconds to wait for host to come back |
| `debian_upgrade_reboot_delay` | `5` | Seconds to wait before initiating reboot |

### Safety checks

| Variable | Default | Description |
|---|---|---|
| `debian_upgrade_check_disk_space` | `true` | Check free disk space before upgrading |
| `debian_upgrade_min_free_disk_mb` | `4096` | Minimum required free space on `/` in MB |
| `debian_upgrade_verify_after_upgrade` | `true` | Verify OS version and APT health after reboot |

### APT reliability

| Variable | Default | Description |
|---|---|---|
| `debian_upgrade_apt_timeout` | `3600` | Timeout in seconds for long apt operations |
| `debian_upgrade_apt_retries` | `3` | Number of retries on transient apt failures |
| `debian_upgrade_apt_retry_delay` | `30` | Delay in seconds between retries |

---

## Upgrade Path Safety

Debian only supports upgrading **one major version at a time** (N→N+1). This role enforces that constraint by default to prevent broken systems.

### Sequential upgrade enforcement

When `debian_upgrade_allow_skip_release` is `false` (the default), the role ensures that the target version is exactly one major version above the current version. For example:

- **Debian 11→12** ✅ — sequential, allowed
- **Debian 12→13** ✅ — sequential, allowed
- **Debian 11→13** ❌ — blocked (skips version 12)

If you need to go from Debian 11 to Debian 13, run the playbook twice — first targeting version 12, then targeting version 13.

### Skip-release override

Set `debian_upgrade_allow_skip_release: true` to bypass the sequential enforcement:

```
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_target_version=13" \
  -e "debian_upgrade_allow_skip_release=true"
```

> **Warning:** Skipping releases is **unsupported by Debian** and may break your system. Use at your own risk.

### Already-at-target and downgrade protection

The role handles edge cases gracefully:

- **Already at target version** — the host is skipped with a message (no error, no changes).
- **Above target version** — if a host is already running a version newer than the target (e.g. host is on Debian 13 and target is 12), the host is skipped gracefully. Downgrades are never attempted.

---

## Tags

You can run individual stages using Ansible tags:

| Tag | Stage |
|---|---|
| `preflight` | Pre-flight checks (OS version, disk space, apt lock) |
| `pre-upgrade` | Update current packages before switching sources |
| `sources` | Rewrite APT sources to target version |
| `dist-upgrade` | Run `apt-get upgrade` + `apt-get dist-upgrade` |
| `reboot` | Reboot the host |
| `verify` | Post-upgrade version and health checks |

### Examples

```
# Only run pre-flight checks
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml --tags preflight

# Only rewrite sources (skip upgrade)
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml --tags sources

# Skip the reboot stage
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml --skip-tags reboot
```

---

## Galaxy Dependencies

Dependencies are declared in `requirements.yml` and in the role's `meta/main.yml`.

| Name | Type | Version | Purpose |
|---|---|---|---|
| `robertdebock.update` | Role | `3.1.10` | Fully updates all packages on the current OS before the major upgrade |
| `robertdebock.reboot` | Role | `3.2.7` | Used internally for safe, wait-aware reboots |
| `robertdebock.bootstrap` | Role | `7.1.7` | Ensures Python and basic tooling are present on the target |

No external collections are required — every module used in this playbook is from `ansible.builtin`
(e.g. `apt`, `dpkg_selections`, `template`, `copy`, `command`, `reboot`, `setup`, `assert`).

Install roles:

```
ansible-galaxy role install -r requirements.yml
```

Or via the Makefile:

```
make install
```

---

## Upgrade Process

```
┌──────────────────────────────────────────────────────┐
│                Resolve & Validate                    │
│  • Resolve target codename from version number       │
│    (debian_upgrade_target_version → codename map)    │
│  • Already at target version? → skip host            │
│  • Above target version? → skip host (no downgrade)  │
│  • Sequential path check (N→N+1 enforced by default) │
│    — blocked if skip-release not allowed              │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│                  Pre-flight Checks                   │
│  • Assert OS = Debian                                │
│  • Check free disk space (>= 4 GB)                   │
│  • Verify apt is not locked                          │
│  • Print upgrade plan summary                        │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│              Pre-upgrade (Galaxy: update)             │
│  • Install prerequisites                             │
│  • apt-get update                                    │
│  • apt-get full-upgrade (current version)            │
│  • Reboot if kernel update pending                   │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│                  Rewrite APT Sources                 │
│  • Backup sources.list + sources.list.d/*            │
│  • Replace codename in sources.list.d files          │
│  • Deploy new sources.list (Jinja2 template)         │
│  • apt-get update with new sources                   │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│                    dist-upgrade                       │
│  • apt-get upgrade  (safe, phase 1)                  │
│  • apt-get dist-upgrade  (full, phase 2)             │
│  • apt-get autoremove --purge                        │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│                       Reboot                         │
│  • Reboot + wait for host to come back               │
│  • (or print manual reboot reminder)                 │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│              Post-upgrade Verification               │
│  • Re-gather facts                                   │
│  • Assert target version / codename                  │
│  • apt-get check                                     │
│  • dpkg --audit                                      │
│  • Print final status report                         │
└──────────────────────────────────────────────────────┘
```

---

## Examples

### Upgrade a single host

```
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  --limit debian-server-01
```

### Upgrade to a specific Debian version

```
# Upgrade to Debian 12
make run VERSION=12

# Upgrade to Debian 13 (default)
make run

# Upgrade to Debian 14
make run VERSION=14

# Or with ansible-playbook directly:
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_target_version=12"
```

### Upgrade without automatic reboot

```
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_allow_reboot=false"
```

### Use a local mirror

```
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_apt_mirror=mirror.yandex.ru/debian" \
  -e "debian_upgrade_apt_security_mirror=mirror.yandex.ru/debian-security"
```

### Hold back the kernel during upgrade

```
# host_vars/debian-server-01.yml
debian_upgrade_held_packages:
  - linux-image-amd64
  - linux-headers-amd64
```

### Run in CI (skip interactive pause, no reboot)

```
CI=true ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_allow_reboot=false"
```

### Only verify (after a manual upgrade)

```
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml --tags verify
```

---

## Testing

[Molecule](https://molecule.readthedocs.io/) is used for integration testing of the `debian_upgrade` role.

### Test scenarios

| Scenario | Description |
|---|---|
| `default` | Debian 12→13 sources rewrite — verifies the full upgrade workflow |
| `already_upgraded` | Host is already at the target version — verifies graceful skip |
| `skip_release` | Debian 11→13 with `allow_skip_release=false` — verifies the upgrade is blocked |
| `downgrade_protection` | Debian 13→12 — verifies the host is skipped (no downgrade) |

### Prerequisites

Install the test dependencies:

```
pip install molecule molecule-docker docker ansible-lint yamllint
```

Or use the bundled requirements file:

```
pip install -r roles/debian_upgrade/molecule/requirements.txt
```

### Running tests

```
# Run all Molecule scenarios
make test

# Run individual scenarios
make test-default              # default (12→13 sources rewrite)
make test-already-upgraded     # already_upgraded (skip on target)
make test-skip-release         # skip_release (11→13 blocked)
make test-downgrade            # downgrade_protection (13→12 skip)

# Utility targets
make test-converge             # Run converge only (no verify/destroy)
make test-lint                 # Run linting inside the Molecule environment
```

---

## Troubleshooting

### `apt` is locked

```
TASK [Pre-flight | Verify apt is not locked] FAILED
```

Another process is holding the dpkg lock. On the target host, run:

```
ps aux | grep -E 'apt|dpkg|unattended'
# Kill the offending process, then remove stale lock files if needed:
rm -f /var/lib/dpkg/lock-frontend /var/lib/dpkg/lock /var/cache/apt/archives/lock
dpkg --configure -a
```

### Not enough disk space

```
TASK [Pre-flight | Check available disk space on /] FAILED
```

Free up space before retrying:

```
apt-get clean
apt-get autoremove --purge
journalctl --vacuum-size=500M
```

Or lower the threshold temporarily (not recommended for production):

```
-e "debian_upgrade_min_free_disk_mb=2048"
```

### Post-upgrade: broken packages

```
TASK [Post-upgrade | Fail if dpkg audit reports broken packages] FAILED
```

On the target host:

```
dpkg --configure -a
apt-get -f install
apt-get dist-upgrade
```

### Host does not come back after reboot

Increase the reboot timeout:

```
-e "debian_upgrade_reboot_timeout=1200"
```

### `robertdebock.*` roles not found

Run the Galaxy install step:

```
ansible-galaxy install -r requirements.yml
```

If you use a private Automation Hub or a local roles path, add `--roles-path`:

```
ansible-galaxy install -r requirements.yml --roles-path ./roles
```

---

## Security Considerations

- The playbook runs as `root` (or via `sudo`). Restrict access to the control node and the private key file.
- All APT sources are fetched over **HTTP** by default (matching the official Debian mirror layout). Debian package integrity is verified via GPG signatures regardless of transport.
- To use HTTPS mirrors, replace `http://` with `https://` in `debian_upgrade_apt_mirror` and update your `sources.list.j2` template accordingly. Note that not all Debian mirrors support HTTPS.
- Backup files (`.bak`) are left on the remote host. Remove them after confirming the upgrade is stable:

```
rm /etc/apt/sources.list.bak /etc/apt/sources.list.d/*.list.bak
```

---

## License

MIT