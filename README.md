# hobbiton

Ansible-managed home server infrastructure running on Proxmox.

## Architecture

```
Physical Host — Proxmox (192.168.1.85)
├── vmbr0  — Management LAN (192.168.1.0/24)
│   ├── pihole         192.168.1.87   LXC 100
│   ├── unifi          192.168.1.91   LXC 101
│   ├── homeassistant  192.168.1.93   LXC 102
│   ├── paperless      192.168.1.116  LXC 104
│   └── cloudflared    192.168.1.154  LXC 106
├── vmbr1  — Services VLAN 20 (10.10.20.0/24)
│   └── docker         10.10.20.10    LXC 107
└── vmbr2  — Monitoring VLAN 30 (10.10.30.0/24)
    ├── prometheus     10.10.30.10    LXC 103
    ├── grafana        10.10.30.11    LXC 108
    └── pve-exporter   10.10.30.12    LXC 109

Network: EdgeRouter X — VLAN-aware APs (Unifi)
Public access: Cloudflare tunnel — no inbound ports open
```

## Prerequisites

- Python 3 + virtualenv
- Ansible installed in `.venv`
- `community.general` collection: `ansible-galaxy collection install community.general`
- SSH key deployed to all hosts via `pve_onboard.yml`

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install ansible
```

## Inventory

| Host | Group | IP | VMID |
|---|---|---|---|
| pve-node-1 | pvenodes | 192.168.1.85 | — |
| pihole | lxc_containers | 192.168.1.87 | 100 |
| unifi | lxc_containers | 192.168.1.91 | 101 |
| homeassistant | lxc_containers | 192.168.1.93 | 102 |
| paperless | lxc_containers | 192.168.1.116 | 104 |
| cloudflared | lxc_containers | 192.168.1.154 | 106 |
| docker | lxc_services | 10.10.20.10 | 107 |
| prometheus | lxc_monitoring | 10.10.30.10 | 103 |
| grafana | lxc_monitoring | 10.10.30.11 | 108 |
| pve-exporter | lxc_monitoring | 10.10.30.12 | 109 |

## Playbooks

| Playbook | Targets | What it does |
|---|---|---|
| `site.yml` | all | Full stack — run all playbooks in order |
| `harden_proxmox_host.yml` | pvenodes | SSH, fail2ban, UFW, unattended upgrades |
| `configure_network.yml` | pvenodes | VLAN bridges (vmbr1, vmbr2) |
| `harden_lxc.yml` | lxc_all | SSH, fail2ban, unattended upgrades |
| `harden_docker.yml` | lxc_services | Docker daemon hardening |
| `pve_onboard.yml` | all | Bootstrap ansible user (run once as root) |

## Usage

```bash
source .venv/bin/activate

# Dry run
ansible-playbook site.yml -i inventory --check --diff

# Apply
ansible-playbook site.yml -i inventory

# Single host
ansible-playbook harden_lxc.yml -i inventory --limit pihole

# Bootstrap a new container (run once)
# Step 1 — copy your SSH key into the container from the Proxmox host
cat ~/.ssh/id_ed25519.pub | pct exec <vmid> -- tee /root/.ssh/authorized_keys

# Step 2 — run onboarding
ansible-playbook pve_onboard.yml -i inventory --limit <host> -e "ansible_user=root"
```

## Network setup

Manual steps required before running `configure_network.yml`:

- [EdgeRouter X VLAN setup](docs/router_vlan_setup.md)
- [Unifi WiFi VLAN setup](docs/unifi_vlan_setup.md)
- [Monitoring stack setup](docs/monitoring_setup.md)

## Pending

- Deploy monitoring stack — see [docs/monitoring_setup.md](docs/monitoring_setup.md)
- GitHub Actions lint workflow (GitHub-hosted runner)
- GitHub Actions check workflow (`--check --diff` on PR, self-hosted runner)
- GitHub Actions deploy workflow (manual trigger, self-hosted runner)
- Self-hosted runner LXC (`runner_lxc` role + `provision_runner.yml`)
- Re-evaluate Cloudflare tunnel setup (dual-tunnel issues between LXC 106 and Docker compose tunnel)
