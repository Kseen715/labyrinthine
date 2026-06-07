# vps-benchmark

Ansible playbook and role that installs benchmark tools and runs CPU, RAM, disk,
and network tests on Debian-based VPS servers (Debian 11/12, Ubuntu 20.04/22.04/24.04).

Benchmark results are printed in the Ansible output **and** saved as plain-text
files on the remote host.

## What it does

| Benchmark | Tool | Output file |
|---|---|---|
| CPU (1-thread & all-thread) | `sysbench cpu` | `cpubench.txt` |
| RAM throughput | `sysbench memory` | `membench.txt` |
| System disk read speed | `hdparm -Tt` | `drivebench.txt` |
| Network / bandwidth | `bench.tlab.pw` | `netbench.txt` |

> `TERM=dumb` is set for the entire play, so benchmark scripts that open
> alternate terminal screen buffers (e.g. `btop`, interactive tools) are safe
> to run over SSH without corrupting the terminal.

## Requirements

- Python 3.10+
- Ansible 2.14+ (tested on 2.20)
- Target hosts: Debian 11/12 or Ubuntu 20.04/22.04/24.04
- `become: true` (sudo) on target hosts

Install Python dependencies:

```bash
pip install -r ../requirements.txt
# or, to also get Molecule for testing:
pip install 'molecule-plugins[docker]' ansible-lint
```

## Directory structure

```
vps-benchmark/
├── ansible.cfg
├── benchmark.yml              # Main playbook
├── Makefile                   # Convenience targets
├── requirements.yml
├── inventory/
│   └── hosts.yml              # Default inventory (localhost)
├── roles/
│   └── vps_benchmark/
│       ├── defaults/main.yml  # Role variables
│       ├── meta/main.yml
│       └── tasks/main.yml
└── molecule/
    └── default/               # Molecule test scenario
        ├── molecule.yml
        ├── converge.yml
        └── verify.yml
```

## Quick start

### Run against localhost

```bash
cd vps-benchmark
make run
```

### Run against a remote VPS

Edit `inventory/hosts.yml`:

```yaml
all:
  hosts:
    my-vps:
      ansible_host: 1.2.3.4
      ansible_user: root
```

Then:

```bash
make run HOST=my-vps
```

Or directly with `ansible-playbook`:

```bash
ansible-playbook benchmark.yml -i inventory/hosts.yml --limit my-vps -v
```

## Variables

| Variable | Default | Description |
|---|---|---|
| `vps_benchmark_run_netbench` | `true` | Run the network benchmark (`bench.tlab.pw`). Set to `false` in air-gapped or CI environments. |
| `vps_benchmark_output_dir` | `/tmp` | Directory on the remote host where `.txt` result files are written. |

Override on the command line:

```bash
# Skip the network test
make run-no-netbench

# Or directly
ansible-playbook benchmark.yml -e vps_benchmark_run_netbench=false

# Custom output directory
ansible-playbook benchmark.yml -e vps_benchmark_output_dir=/root/bench
```

## Makefile targets

```
make help              Show all available targets
make install-deps      Install Ansible + Molecule Python deps
make run               Run the full benchmark (localhost by default)
make run-no-netbench   Run without the network test
make syntax            Check playbook syntax
make lint              Lint with ansible-lint
make molecule-test     Full Molecule test suite (create → converge → verify → destroy)
make molecule-converge Apply the role inside test containers
make molecule-verify   Run Molecule verifier only
make molecule-destroy  Destroy Molecule containers
```

## Testing with Molecule

The Molecule scenario tests the role against **Debian 12** and **Ubuntu 22.04**
Docker containers. The network benchmark is disabled in CI
(`vps_benchmark_run_netbench: false`) to avoid downloading remote scripts.

```bash
# Full test cycle
make molecule-test

# Step by step
make molecule-converge
make molecule-verify
make molecule-destroy
```

Docker images used (pre-built, systemd-enabled):

- `geerlingguy/docker-debian12-ansible`
- `geerlingguy/docker-ubuntu2204-ansible`

Pull them in advance if the test environment has limited internet access:

```bash
docker pull geerlingguy/docker-debian12-ansible:latest
docker pull geerlingguy/docker-ubuntu2204-ansible:latest
```

## Result files

After a successful run, result files are available on the remote host at
`vps_benchmark_output_dir` (default `/tmp`):

```
/tmp/cpubench.txt     — CPU events-per-second (1-thread and all-thread)
/tmp/membench.txt     — RAM MiB transferred / MiB per second
/tmp/drivebench.txt   — Disk cached and buffered read speed
/tmp/netbench.txt     — Network benchmark report (if enabled)
```

Results are also printed directly in the Ansible output via `debug` tasks at the
end of the play, so you see them without logging into the remote host.
