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
│   ├── cloudflared    192.168.1.154  LXC 106
│   └── miniflux       192.168.1.177  LXC 113
├── vmbr1  — Services VLAN 20 (10.10.20.0/24)
│   └── docker         10.10.20.10    LXC 107
├── vmbr2  — Monitoring VLAN 30 (10.10.30.0/24)
│   ├── prometheus     10.10.30.10    LXC 103
│   ├── grafana        10.10.30.11    LXC 108
│   ├── pve-exporter   10.10.30.12    LXC 109
│   ├── loki           10.10.30.13    LXC 110
│   └── umami          10.10.30.14    LXC 111
└── vmbr3  — ExileMail VLAN 60 (10.10.60.0/24)
    └── opentrashmail  10.10.60.10    LXC 112

Network: EdgeRouter X — VLAN-aware APs (Unifi)
Public access: Cloudflare tunnel — no inbound ports open
```

## Prerequisites

- Python 3 + virtualenv

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
ansible-galaxy collection install -r requirements.yml
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
| miniflux | lxc_containers | 192.168.1.177 | 113 |
| docker | lxc_services | 10.10.20.10 | 107 |
| prometheus | lxc_monitoring | 10.10.30.10 | 103 |
| grafana | lxc_monitoring | 10.10.30.11 | 108 |
| pve-exporter | lxc_monitoring | 10.10.30.12 | 109 |
| loki | lxc_monitoring | 10.10.30.13 | 110 |
| umami | lxc_monitoring | 10.10.30.14 | 111 |
| opentrashmail | lxc_exilemail | 10.10.60.10 | 112 |

## Playbooks

| Playbook | Targets | What it does |
|---|---|---|
| `site.yml` | all | Full stack — harden + network + storage + lxc hardening |
| `harden_proxmox_host.yml` | pvenodes | SSH, fail2ban, UFW, unattended upgrades |
| `configure_network.yml` | pvenodes | VLAN bridges (vmbr1–vmbr3) |
| `configure_storage.yml` | pvenodes | LXC bind mounts (prerequisite for deploy_opentrashmail.yml) |
| `harden_lxc.yml` | lxc_all | SSH, fail2ban, unattended upgrades |
| `harden_docker.yml` | lxc_services | Docker daemon hardening |
| `deploy_opentrashmail.yml` | opentrashmail | Full OpenTrashMail service deploy (requires configure_storage.yml first) |
| `deploy_monitoring.yml` | prometheus, pvenodes | Prometheus scrape config + Proxmox API user |
| `pve_onboard.yml` | all | Bootstrap ansible user (run once as root per new host) |

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

## Known gaps

- **miniflux** — in inventory but no deploy playbook or role yet
- **Monitoring stack** — Prometheus, Grafana, Loki, and cAdvisor are provisioned manually; see [docs/monitoring_setup.md](docs/monitoring_setup.md)
- **UFW on lxc_containers** — SSH-hardened but no firewall applied to pihole, unifi, homeassistant, paperless, cloudflared, miniflux
- **No CI/CD** — playbooks are run locally; no linting or check-mode pipeline
