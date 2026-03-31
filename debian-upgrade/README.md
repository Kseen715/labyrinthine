# Debian Upgrade Playbook

Ansible playbook to automatically upgrade **Debian 11 (Bullseye)** or **Debian 12 (Bookworm)** to **Debian 13 (Trixie)** using roles from Ansible Galaxy.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Inventory Setup](#inventory-setup)
- [Variables Reference](#variables-reference)
- [Tags](#tags)
- [Galaxy Dependencies](#galaxy-dependencies)
- [Upgrade Process](#upgrade-process)
- [Examples](#examples)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)
- [License](#license)

---

## Overview

This playbook performs a safe, automated major version upgrade of Debian Linux. It:

1. Verifies the current OS is Debian 11 or 12
2. Skips hosts already on Debian 13 without failing
3. Fully updates all packages on the current version before switching sources
4. Backs up `/etc/apt/sources.list` and all `sources.list.d` files
5. Rewrites APT sources to point to the Debian 13 (Trixie) repositories
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
| Target OS | Debian 11 (Bullseye) or Debian 12 (Bookworm) |
| Target user | `root` or a user with passwordless `sudo` |
| Free disk space | >= 4 GB on `/` (configurable) |

Install Ansible on the control node:

```bash
pip install "ansible>=2.14" ansible-lint
```

---

## Project Structure

```
debian-upgrade/
├── requirements.yml                   # Ansible Galaxy roles & collections
├── upgrade_debian.yml                 # Main playbook
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
        └── templates/
            └── sources.list.j2        # Jinja2 template for new APT sources
```

---

## Quick Start

### 1. Install Galaxy dependencies

Roles and collections must be installed **separately**:

```bash
# Install Galaxy roles
ansible-galaxy role install -r requirements.yml

# Install Galaxy collections
ansible-galaxy collection install -r requirements.yml
```

> **Why two commands?** `ansible-galaxy install` only installs roles.
> Collections require `ansible-galaxy collection install`.

### 1a. (Alternative) Use the Makefile

```bash
make install        # installs both roles and collections in one shot
make install-force  # force-reinstall (useful after requirements.yml changes)
```

### 2. Configure your inventory

Edit `inventory/hosts.yml` and replace the example IPs with your actual hosts:

```yaml
debian_hosts:
  hosts:
    my-server:
      ansible_host: 192.168.1.10
      ansible_user: root
```

### 3. Test connectivity

```bash
ansible -i inventory/hosts.yml debian_hosts -m ping
# or via Makefile:
make ping INVENTORY=inventory/hosts.yml
```

### 4. Dry-run (no changes)

```bash
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml --check --diff
# or via Makefile:
make check INVENTORY=inventory/hosts.yml
```

### 5. Run the upgrade

```bash
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml
# or via Makefile:
make run INVENTORY=inventory/hosts.yml
```

---

## Makefile

A `Makefile` is included for convenience. All targets accept optional overrides:

```
INVENTORY   — path to inventory file  (default: inventory/secret_hosts.yml)
LIMIT       — host or group to target (default: debian_hosts)
TAGS        — comma-separated tags    (default: none)
SKIP_TAGS   — tags to skip            (default: none)
EXTRA       — extra -e arguments      (default: none)
```

| Target | Description |
|---|---|
| `make install` | Install Galaxy roles **and** collections |
| `make install-roles` | Install Galaxy roles only |
| `make install-collections` | Install Galaxy collections only |
| `make install-force` | Force-reinstall everything |
| `make ping` | Test SSH connectivity (`ansible -m ping`) |
| `make facts` | Show OS facts from all hosts |
| `make os-check` | Print `/etc/os-release` from all hosts |
| `make check` | Dry-run with `--check --diff` |
| `make run` | Apply the full upgrade playbook |
| `make run-no-reboot` | Full upgrade, skip automatic reboot |
| `make preflight` | Run pre-flight checks only (check mode) |
| `make sources` | Rewrite APT sources to Trixie only |
| `make dist-upgrade` | Run dist-upgrade stage only |
| `make verify` | Post-upgrade verification only |
| `make reboot-hosts` | Manually reboot hosts and wait |
| `make lint` | Lint with `ansible-lint` |
| `make syntax-check` | Syntax check (no SSH needed) |
| `make clean-galaxy` | Remove installed roles and collections |

### Examples

```bash
# Dry-run on a single host
make check LIMIT=debian-server-01

# Full upgrade, no reboot
make run LIMIT=debian-server-01 EXTRA='-e debian_upgrade_allow_reboot=false'

# Only rewrite sources (skip upgrade)
make tags TAGS=sources LIMIT=debian-server-01

# Use a custom inventory
make run INVENTORY=inventory/secret_hosts.yml
```

---

## Inventory Setup

### Basic inventory (`inventory/hosts.yml`)

```yaml
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

```yaml
debian_hosts_production:
  vars:
    debian_upgrade_allow_reboot: false  # reboot manually at a maintenance window
  hosts:
    prod-01:
      ansible_host: 10.0.0.10
```

### Using SSH jump host

```yaml
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
| `debian_upgrade_target_version` | `"13"` | Target Debian major version |
| `debian_upgrade_target_codename` | `"trixie"` | Target Debian codename |
| `debian_upgrade_supported_versions` | `["11", "12"]` | Allowed source versions |
| `debian_upgrade_codenames` | `{11: bullseye, 12: bookworm, 13: trixie}` | Codename lookup map |

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

## Tags

You can run individual stages using Ansible tags:

| Tag | Stage |
|---|---|
| `preflight` | Pre-flight checks (OS version, disk space, apt lock) |
| `pre-upgrade` | Update current packages before switching sources |
| `sources` | Rewrite APT sources to Trixie |
| `dist-upgrade` | Run `apt-get upgrade` + `apt-get dist-upgrade` |
| `reboot` | Reboot the host |
| `verify` | Post-upgrade version and health checks |

### Examples

```bash
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

```bash
ansible-galaxy role install -r requirements.yml
```

Or via the Makefile:

```bash
make install
```

---

## Upgrade Process

```
┌──────────────────────────────────────────────────────┐
│                  Pre-flight Checks                   │
│  • Assert OS = Debian                                │
│  • Assert version ∈ {11, 12}                         │
│  • Check free disk space (>= 4 GB)                   │
│  • Verify apt is not locked                          │
│  • Print upgrade plan summary                        │
└────────────────────────┬─────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────┐
│              Pre-upgrade (Galaxy: update)            │
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
│                    dist-upgrade                      │
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
│  • Assert Debian 13 / trixie                         │
│  • apt-get check                                     │
│  • dpkg --audit                                      │
│  • Print final status report                         │
└──────────────────────────────────────────────────────┘
```

---

## Examples

### Upgrade a single host

```bash
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  --limit debian-server-01
```

### Upgrade without automatic reboot

```bash
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_allow_reboot=false"
```

### Use a local mirror

```bash
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_apt_mirror=mirror.yandex.ru/debian" \
  -e "debian_upgrade_apt_security_mirror=mirror.yandex.ru/debian-security"
```

### Hold back the kernel during upgrade

```yaml
# host_vars/debian-server-01.yml
debian_upgrade_held_packages:
  - linux-image-amd64
  - linux-headers-amd64
```

### Run in CI (skip interactive pause, no reboot)

```bash
CI=true ansible-playbook -i inventory/hosts.yml upgrade_debian.yml \
  -e "debian_upgrade_allow_reboot=false"
```

### Only verify (after a manual upgrade)

```bash
ansible-playbook -i inventory/hosts.yml upgrade_debian.yml --tags verify
```

---

## Troubleshooting

### `apt` is locked

```
TASK [Pre-flight | Verify apt is not locked] FAILED
```

Another process is holding the dpkg lock. On the target host, run:

```bash
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

```bash
apt-get clean
apt-get autoremove --purge
journalctl --vacuum-size=500M
```

Or lower the threshold temporarily (not recommended for production):

```bash
-e "debian_upgrade_min_free_disk_mb=2048"
```

### Post-upgrade: broken packages

```
TASK [Post-upgrade | Fail if dpkg audit reports broken packages] FAILED
```

On the target host:

```bash
dpkg --configure -a
apt-get -f install
apt-get dist-upgrade
```

### Host does not come back after reboot

Increase the reboot timeout:

```bash
-e "debian_upgrade_reboot_timeout=1200"
```

### `robertdebock.*` roles not found

Run the Galaxy install step:

```bash
ansible-galaxy install -r requirements.yml
```

If you use a private Automation Hub or a local roles path, add `--roles-path`:

```bash
ansible-galaxy install -r requirements.yml --roles-path ./roles
```

---

## Security Considerations

- The playbook runs as `root` (or via `sudo`). Restrict access to the control node and the private key file.
- All APT sources are fetched over **HTTP** by default (matching the official Debian mirror layout). Debian package integrity is verified via GPG signatures regardless of transport.
- To use HTTPS mirrors, replace `http://` with `https://` in `debian_upgrade_apt_mirror` and update your `sources.list.j2` template accordingly. Note that not all Debian mirrors support HTTPS.
- Backup files (`.bak`) are left on the remote host. Remove them after confirming the upgrade is stable:

```bash
rm /etc/apt/sources.list.bak /etc/apt/sources.list.d/*.list.bak
```

---

## License

MIT