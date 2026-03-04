# Roadmap

Prioritized next steps for the Hobbiton project, based on the current state of the codebase.

## High Priority

### 1. Service Playbooks

The inventory defines four LXC containers but there are no playbooks to configure them. Create a dedicated playbook for each:

- `lxc_homeassistant.yml` — update HA, manage configuration, restart service
- `lxc_pihole.yml` — update blocklists, manage DNS settings, sync configuration
- `lxc_unifi.yml` — controller updates, backup schedule
- `lxc_paperless.yml` — OCR settings, storage mounts, service management

### 2. Role-Based Structure

Refactor playbooks into reusable Ansible roles so logic can be shared and tested independently:

```
roles/
  common/         # baseline: OS updates, NTP, unattended-upgrades
  homeassistant/
  pihole/
  unifi/
  paperless/
```

A `common` role applied to all hosts would handle baseline hardening and package updates, reducing duplication across service playbooks.

### 3. Re-enable Host Key Checking

`ansible.cfg` currently sets `host_key_checking=False`. Populate a `known_hosts` file (or use `ssh-keyscan` in a bootstrap step) and remove this override to restore the security check.

## Medium Priority

### 4. Group and Host Variables

Move configuration out of playbooks and into structured variable files:

```
group_vars/
  all.yml             # vars shared by all hosts
  lxc-containers.yml  # vars for all containers
  pvenodes.yml        # vars for PVE nodes
host_vars/
  homeassistant.yml
  piehole.yml
  unifi.yml
  paperless.yml
```

This separates data from logic and makes the playbooks reusable across different environments.

### 5. Ansible Vault for Secrets

Any passwords, API tokens, or other sensitive values should be encrypted with `ansible-vault` rather than stored in plaintext. Set up a vault password file (gitignored) or integrate with an external secrets manager.

### 6. LXC Provisioning Playbook

Use the already-installed `proxmoxer` library to automate LXC container creation on `pve-node-1`, rather than only managing containers that were created manually. This would make the infrastructure fully reproducible from scratch.

## Lower Priority

### 7. CI/CD Pipeline

Add a GitHub Actions (or Gitea Actions) workflow to:

- Run `ansible-lint` on every push
- Run `ansible-playbook --syntax-check` for all playbooks
- Optionally run [Molecule](https://molecule.readthedocs.io/) tests against Docker containers to validate role behavior

### 8. Backup Automation

A playbook (or cron-triggered role) to trigger Proxmox Backup Server snapshots or `vzdump` backups on a schedule, and verify their integrity.

### 9. Automated Updates

A scheduled playbook that runs `apt upgrade` across all containers, restarts affected services, and reports results — removing the need to SSH into each host manually.

### 10. Monitoring

Automate the setup of a lightweight monitoring stack such as:
- **Uptime Kuma** — simple HTTP/service uptime monitoring
- **Prometheus + Node Exporter + Grafana** — full metrics and dashboards

Both can be deployed as LXC containers and configured entirely via Ansible.
