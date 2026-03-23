# Monitoring Stack Setup

Prometheus and Grafana run as native systemd services in separate LXCs on VLAN 30 (10.10.30.0/24).

| Host | IP | VMID | Port |
|---|---|---|---|
| prometheus | 10.10.30.10 | 103 | 9090 |
| grafana | 10.10.30.11 | 108 | 3000 |

## Exporter strategy

| Scope | Tool | Where it runs |
|---|---|---|
| All Proxmox LXCs + VMs | pve_exporter | LXC 109 (10.10.30.12) |
| Docker containers in LXC 107 | cAdvisor | Docker compose service in LXC 107 |

- **pve_exporter** queries the Proxmox API — gives uptime/status for every LXC and VM from a single install, no agent needed inside containers
- **cAdvisor** runs alongside the app containers and exposes per-container CPU, memory, network, and restart metrics
- **node_exporter** not used — pve_exporter covers LXC-level metrics via the API

---

## Prerequisites

- Both LXCs created via Proxmox community scripts (plain Debian)
- Bridged to vmbr2, gateway 10.10.30.1
- Bootstrapped with `pve_onboard.yml`
- Hardened with `harden_lxc.yml`


## Step 4 — Add cAdvisor to Docker compose (LXC 107)

Add to `docker-compose.yml` in the app repo:
```yaml
cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  restart: unless-stopped
  privileged: true
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
  expose:
    - "8080"
```

---

## Step 5 — Configure Prometheus scrape targets

Edit `/etc/prometheus/prometheus.yml` on LXC 103:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: proxmox
    metrics_path: /pve
    params:
      target: ['192.168.1.85']
    static_configs:
      - targets: ['10.10.30.12:9221']   # pve_exporter LXC 109

  - job_name: docker_containers
    static_configs:
      - targets: ['10.10.20.10:8080']    # cAdvisor in LXC 107
```

Reload Prometheus:
```bash
systemctl restart prometheus
```

---

## Step 6 — Connect Grafana to Prometheus

1. Open Grafana at `http://10.10.30.11:3000` (default login: admin/admin — change immediately)
2. Connections → Data Sources → Add → Prometheus
3. URL: `http://10.10.30.10:9090`
4. Save & Test

---

## Step 7 — EdgeRouter firewall exceptions

Prometheus (VLAN 30) needs to reach pve_exporter on the management LAN and cAdvisor on VLAN 20.

On the EdgeRouter:
```bash
configure

set firewall name BLOCK_IN rule 3 action accept
set firewall name BLOCK_IN rule 3 description 'Allow Prometheus scraping from VLAN 30'
set firewall name BLOCK_IN rule 3 source address 10.10.30.0/24
set firewall name BLOCK_IN rule 3 destination port 9221,8080
set firewall name BLOCK_IN rule 3 protocol tcp

commit
save
```

---

## Step 8 — Expose Grafana via Cloudflare tunnel

1. In LXC 106 tunnel config, add public hostname:
   - Subdomain: `grafana`
   - Service: `http://10.10.30.11:3000`
2. In Cloudflare Zero Trust → Access → Applications, add Grafana hostname with email policy

---

## Importing dashboards

Recommended Grafana dashboard IDs (import via Dashboards → Import):

| Dashboard | ID |
|---|---|
| Proxmox via pve_exporter | 10347 |
| cAdvisor Container Metrics | 14282 |
| Prometheus Stats | 2 |