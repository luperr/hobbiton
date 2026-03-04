# Hobbiton

Home server infrastructure automation using Ansible, targeting a Proxmox Virtual Environment (PVE) with multiple LXC containers for self-hosted services.

## Architecture

**Physical Host:**
- `pve-node-1` (192.168.1.85) — Proxmox VE hypervisor

**LXC Containers:**

| Service | Hostname | IP |
|---|---|---|
| Home Assistant | homeassistant | 192.168.1.93 |
| Pi-hole | piehole | 192.168.1.87 |
| UniFi Controller | unifi | 192.168.1.91 |
| Paperless-ngx | paperless | 192.168.1.116 |

## Setup

### Prerequisites

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### Initial Server Onboarding

Run once on a fresh server to create the `ansible` user and configure SSH access:

```bash
ansible-playbook pve_onboard.yml -u <initial-user> -k --ask-become-pass
```

### Proxmox Node Setup

Install `proxmoxer` on PVE nodes for Proxmox API interaction:

```bash
ansible-playbook pve_install_proxmoxer.yml
```

## Playbooks

| Playbook | Targets | Purpose |
|---|---|---|
| `pve_onboard.yml` | all | Bootstrap ansible user with SSH + sudo access |
| `pve_install_proxmoxer.yml` | pvenodes | Install proxmoxer Python library on PVE nodes |
