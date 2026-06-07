# remnawave-nodes

Ansible project for deploying **RemnaWave proxy nodes** and a **central monitoring server** (Consul + Prometheus + Grafana) without touching the RemnaWave panel — the panel stays yours, this repo only manages the nodes and observability layer.

---

## Architecture

```
                   ┌─────────────────────────────────────────┐
                   │  Monitoring server (monitoring1)         │
                   │                                          │
  HTTPS 443 ──────►│  Caddy  ──► grafana.atlas.gripe          │
                   │         ──► prometheus.atlas.gripe        │
                   │         ──► tofu.atlas.gripe (optional)  │
                   │                                          │
                   │  Grafana  :3000  (localhost)             │
                   │  Prometheus :9090  (localhost)           │
                   │  Consul Server :8500  (localhost)        │
                   │  Node Exporter :9100  (localhost)        │
                   └────────────────┬────────────────────────┘
                                    │ Consul SD / scrape
          ┌─────────────────────────┼─────────────────────────┐
          ▼                         ▼                         ▼
  ┌───────────────┐         ┌───────────────┐         ┌───────────────┐
  │   node1       │         │   node2       │         │   nodeN       │
  │               │         │               │         │               │
  │ RemnaNode:2222│         │ RemnaNode:2222│         │ RemnaNode:2222│
  │ Caddy:443     │         │ Caddy:443     │         │ Caddy:443     │
  │ Consul Agent  │         │ Consul Agent  │         │ Consul Agent  │
  │ Node Exp.:9100│         │ Node Exp.:9100│         │ Node Exp.:9100│
  └───────────────┘         └───────────────┘         └───────────────┘
```

**Consul** runs as a single-server cluster on the monitoring server. Every proxy node runs a Consul agent that joins the server; Prometheus uses Consul service discovery to auto-discover node exporters — no manual scrape target list needed.

---

## Project layout

```
remnawave-nodes/
├── ansible.cfg
├── requirements.yml           # Galaxy collections
├── Makefile                   # convenience targets
├── nodes.yml                  # playbook: deploy proxy nodes
├── monitoring.yml             # playbook: deploy monitoring server
├── inventory/
│   ├── hosts.yml              # fill in real IPs + SECRET_KEYs
│   └── group_vars/
│       ├── all.yml            # shared vars (system_user, consul_datacenter …)
│       ├── nodes.yml          # node-specific vars (app_port, monitoring_server_ip …)
│       └── monitoring.yml     # monitoring vars (domains, Grafana port …)
├── roles/
│   ├── common/                # base hardening applied to every host
│   ├── remnawave_node/        # RemnaNode + Caddy + Consul Agent + Node Exporter
│   └── remnawave_monitoring/  # Consul Server + Prometheus + Grafana + Caddy
└── molecule/
    ├── monitoring/            # tests for the monitoring scenario
    └── node/                  # tests for the node scenario
```

---

## Quick start

### 1. Prerequisites

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install ansible molecule molecule-plugins[docker] ansible-lint
```

Or if the parent repo already has a venv at `../.venv`:

```bash
make install    # installs Galaxy collections only
```

### 2. Edit inventory

**`inventory/hosts.yml`** — fill real IPs and `SECRET_KEY` values:

```yaml
nodes:
  hosts:
    node1:
      ansible_host: 1.2.3.4       # your node IP
      ansible_user: root
      SECRET_KEY: "GET_FROM_PANEL" # Nodes → Add Node → copy key
monitoring:
  hosts:
    monitoring1:
      ansible_host: 9.10.11.12    # your monitoring server IP
      ansible_user: root
```

**`inventory/group_vars/all.yml`**:

| Variable | Description |
|---|---|
| `system_user` | Non-root user created on every host |
| `ssh_public_key` | Your `~/.ssh/id_ed25519.pub` (pushed to all hosts) |
| `consul_datacenter` | Datacenter name for Consul (same everywhere) |

**`inventory/group_vars/nodes.yml`**:

| Variable | Description |
|---|---|
| `monitoring_server_ip` | IP of `monitoring1` — Consul agents join here |
| `monitoring_hostname` | Inventory hostname of monitoring server (for UFW delegation) |
| `app_port` | Port RemnaNode listens on (`2222` default) |

**`inventory/group_vars/monitoring.yml`**:

| Variable | Description |
|---|---|
| `monitoring_caddy_enabled` | `true` to deploy Caddy with Let's Encrypt SSL |
| `caddy_acme_email` | Email for Let's Encrypt notifications |
| `grafana_domain` | Public domain → Grafana (e.g. `grafana.atlas.gripe`) |
| `prometheus_domain` | Public domain → Prometheus (e.g. `prometheus.atlas.gripe`) |
| `prometheus_basic_auth_hash` | Optional bcrypt hash to protect Prometheus (see below) |
| `remna_node_domain` | Set when monitoring server doubles as a proxy node |
| `remnawave_panel_metrics_target` | `"host:3001"` to scrape panel metrics remotely |

### 3. Add node in RemnaWave panel

Before running the nodes playbook, add each node in the web panel:

1. Panel UI → **Nodes** → **Add Node**
2. Set **Port** to match `app_port` (default `2222`)
3. Copy the **Secret Key** shown → paste into `inventory/hosts.yml` as `SECRET_KEY`

### 4. Deploy

```bash
# Deploy monitoring server first (Consul Server must be up before agent joins)
make monitoring

# Deploy all proxy nodes
make nodes

# Deploy a single node
make node HOST=node1
```

---

## SSL / HTTPS setup

Caddy on the monitoring server automatically obtains Let's Encrypt certificates for `grafana_domain` and `prometheus_domain`. **DNS A records must point to the monitoring server's IP before you run the playbook.**

```yaml
# inventory/group_vars/monitoring.yml
monitoring_caddy_enabled: true
caddy_acme_email: "admin@atlas.gripe"
grafana_domain: "grafana.atlas.gripe"
prometheus_domain: "prometheus.atlas.gripe"
```

After deployment:

| Service | URL |
|---|---|
| Grafana | `https://grafana.atlas.gripe` |
| Prometheus | `https://prometheus.atlas.gripe` |
| Consul UI | `http://<monitoring-ip>:8500` (localhost-only, use SSH tunnel) |

> **First login:** Grafana default credentials are `admin / admin`. You will be prompted to change the password on first login.

### Optional: protect Prometheus with HTTP basic-auth

Grafana has its own login screen. Prometheus has no built-in auth; add one via Caddy:

```bash
# Generate bcrypt hash
caddy hash-password --plaintext <your_password>
```

```yaml
# inventory/group_vars/monitoring.yml
prometheus_basic_auth_user: "prometheus"
prometheus_basic_auth_hash: "$2a$14$..."   # paste output above
```

---

## Combo mode — monitoring server as a proxy node

When you want a single server to run both monitoring services **and** act as a RemnaWave proxy node:

1. Add the host to **both** `nodes` and `monitoring` groups in `inventory/hosts.yml`.
2. Create `inventory/host_vars/<hostname>.yml`:

   ```yaml
   # Disable the remnawave_node role's own Caddy – monitoring Caddy handles port 443.
   caddy_enabled: false
   ```

3. Set `remna_node_domain` in `monitoring.yml` variables:

   ```yaml
   # inventory/group_vars/monitoring.yml  (or host_vars)
   remna_node_domain: "tofu.atlas.gripe"
   ```

The monitoring Caddy will then serve all three domains on port 443:

| Domain | Destination |
|---|---|
| `grafana.atlas.gripe` | → Grafana (:3000) |
| `prometheus.atlas.gripe` | → Prometheus (:9090) |
| `tofu.atlas.gripe` | → SNI connection templates (client config) |

---

## Panel metrics scraping

To have Prometheus collect metrics from the RemnaWave panel:

1. In the panel UI → **Settings → Metrics** — enable metrics and set a username/password.
2. On the panel server, allow the monitoring server to reach port `3001`:

   ```bash
   ufw allow from <monitoring_ip> to any port 3001
   ```

3. Set in `inventory/group_vars/monitoring.yml`:

   ```yaml
   remnawave_panel_metrics_target: "panel.example.com:3001"
   prometheus_remnawave_username: "admin"
   prometheus_remnawave_password: "your_metrics_password"
   ```

---

## Per-node custom Caddyfile

Each node's Caddy serves SNI camouflage templates by default. To use a custom Caddyfile for a specific node, create:

```
roles/remnawave_node/templates/caddyfiles/<inventory_hostname>.j2
```

If that file exists it takes precedence; otherwise `default.j2` is used.

---

## Makefile targets

| Target | Description |
|---|---|
| `make install` | Install Galaxy collections |
| `make monitoring` | Deploy monitoring server |
| `make nodes` | Deploy all proxy nodes |
| `make node HOST=<name>` | Deploy a single node |
| `make test` | Run all Molecule tests |
| `make test-monitoring` | Run monitoring Molecule scenario |
| `make test-node` | Run node Molecule scenario |
| `make lint` | Run ansible-lint |

---

## Role reference

### `common`
Applied to every host. Covers: timezone auto-detection, user creation, SSH hardening, UFW, Fail2Ban, Docker CE, sysctl tuning (BBR, conntrack), swap, logrotate, RKN-scanner IP blacklist cronjob.

| Key variable | Default | Description |
|---|---|---|
| `enable_sysctl_optimization` | `true` | BBR + TCP tuning |
| `enable_swap` | `true` | Auto-size swapfile |
| `enable_fail2ban` | `true` | SSH brute-force protection |
| `enable_ufw_blacklist` | `true` | Daily RKN scanner IP blocklist |
| `docker_services_enabled` | `true` | Whether to start docker-compose services |

### `remnawave_node`
Deploys RemnaNode, Caddy (SNI proxy), Consul agent, Node Exporter.

| Key variable | Default | Description |
|---|---|---|
| `app_port` | `2222` | Port RemnaNode listens on |
| `SECRET_KEY` | — | Must be set per-host (from panel) |
| `caddy_enabled` | `true` | Deploy Caddy for this node |
| `consul_server_ip` | `{{ monitoring_server_ip }}` | Consul server to join |

### `remnawave_monitoring`
Deploys Consul Server, Prometheus, Grafana, Node Exporter and optionally Caddy.

| Key variable | Default | Description |
|---|---|---|
| `monitoring_caddy_enabled` | `false` | Deploy Caddy for SSL domains |
| `grafana_domain` | `""` | HTTPS domain for Grafana |
| `prometheus_domain` | `""` | HTTPS domain for Prometheus |
| `remna_node_domain` | `""` | HTTPS domain for SNI templates (combo mode) |
| `caddy_acme_email` | `""` | Let's Encrypt contact email |
| `prometheus_basic_auth_hash` | `""` | Bcrypt hash to protect Prometheus |
