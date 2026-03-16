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
    └── (future monitoring stack)

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
ansible-playbook pve_onboard.yml -i inventory --limit <host> --user root --ask-pass
```

## Network setup

Manual steps required before running `configure_network.yml`:

- [EdgeRouter X VLAN setup](docs/router_vlan_setup.md)
- [Unifi WiFi VLAN setup](docs/unifi_vlan_setup.md)

## Pending

- Split cloudflared: management tunnel to Proxmox host, app tunnel to Docker LXC, decommission LXC 106
- Add monitoring LXC on VLAN 30 (Prometheus + Grafana)
