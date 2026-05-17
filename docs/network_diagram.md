# Network Diagram

```mermaid
flowchart TD
    Internet(["Internet"])
    CF(["Cloudflare Edge\ntunnel.exilemail.xyz\ngrafana.*"])

    subgraph ER["EdgeRouter X — 192.168.1.1"]
        WAN["eth0 WAN\n103.115.113.77"]
        ETH2["eth2 trunk → Proxmox\nVLAN 20 / 30 / 60"]
        ETH4["eth4 trunk → Unifi AP\nVLAN 40 / 50"]
    end

    subgraph MGMT["Management LAN — 192.168.1.0/24  (untagged)"]
        pve["pve-node-1\n192.168.1.85"]
        pihole["pihole\n192.168.1.87"]
        unifi["unifi\n192.168.1.91"]
        ha["homeassistant\n192.168.1.93"]
        paperless["paperless\n192.168.1.116"]
        cloudflared["cloudflared\n192.168.1.154"]
    end

    subgraph PROXMOX["Proxmox — pve-node-1"]
        subgraph V20["VLAN 20 — Services  10.10.20.0/24  vmbr1"]
            docker["docker LXC 107\n10.10.20.10"]
        end
        subgraph V30["VLAN 30 — Monitoring  10.10.30.0/24  vmbr2"]
            prometheus["prometheus LXC 103\n10.10.30.10"]
            grafana["grafana LXC 108\n10.10.30.11"]
            pveexp["pve-exporter LXC 109\n10.10.30.12"]
            loki["loki LXC 110\n10.10.30.13"]
            umami["umami LXC 111\n10.10.30.14"]
        end
        subgraph V60["VLAN 60 — exilemail  10.10.60.0/24  vmbr3"]
            otm["opentrashmail LXC 112\n10.10.60.10\n:25 SMTP  :8080 web UI"]
        end
    end

    subgraph WIFI["Unifi AP"]
        subgraph V40["VLAN 40 — IoT  10.10.40.0/24"]
            iot["IoT devices\n10.10.40.100–200"]
        end
        subgraph V50["VLAN 50 — Guest  10.10.50.0/24"]
            guest["Guest devices\n10.10.50.100–200"]
        end
    end

    %% Internet edge
    Internet <-->|"SMTP :25 port-forward"| WAN
    Internet <--> CF
    CF <-->|"outbound tunnel"| cloudflared

    %% Router to segments
    WAN --- ER
    ER --- MGMT
    ETH2 --> PROXMOX
    ETH4 --> WIFI

    %% Cloudflare tunnel → services
    cloudflared -->|":8080 HTTP"| otm

    %% Monitoring data flows
    prometheus -->|":9221 metrics"| pveexp
    prometheus -->|":8080 cAdvisor"| docker
    pveexp -->|"Proxmox API :8006"| pve
    docker -->|":3100 logs"| loki

    %% DNS
    pihole -. "DNS" .-> MGMT
    pihole -. "DNS" .-> V20
    pihole -. "DNS" .-> V30
    pihole -. "DNS" .-> V40
    pihole -. "DNS" .-> V60
```

---

## VLAN reference

| VLAN | Name | Subnet | Gateway | Trunk port | Firewall |
|------|------|--------|---------|------------|----------|
| untagged | Management | 192.168.1.0/24 | 192.168.1.1 | all | none |
| 10 | Isolated | 192.168.10.0/24 | 192.168.10.1 | — | `IOT_IN` |
| 20 | Services | 10.10.20.0/24 | 10.10.20.1 | eth2 | `BLOCK_IN` |
| 30 | Monitoring | 10.10.30.0/24 | 10.10.30.1 | eth2 | `BLOCK_IN` |
| 40 | IoT | 10.10.40.0/24 | 10.10.40.1 | eth4 | `IOT_IN` |
| 50 | Guest | 10.10.50.0/24 | 10.10.50.1 | eth4 | `BLOCK_IN` |
| 60 | exilemail | 10.10.60.0/24 | 10.10.60.1 | eth2 | `BLOCK_IN` |

## Host reference

| Host | VLAN | IP | VMID | Role |
|------|------|----|------|------|
| pve-node-1 | Management | 192.168.1.85 | — | Proxmox hypervisor |
| pihole | Management | 192.168.1.87 | 100 | DNS / ad blocking |
| unifi | Management | 192.168.1.91 | 101 | Unifi controller |
| homeassistant | Management | 192.168.1.93 | 102 | Home automation |
| paperless | Management | 192.168.1.116 | 104 | Document management |
| cloudflared | Management | 192.168.1.154 | 106 | Cloudflare tunnel daemon |
| docker | VLAN 20 | 10.10.20.10 | 107 | Docker app host |
| prometheus | VLAN 30 | 10.10.30.10 | 103 | Metrics scraping |
| grafana | VLAN 30 | 10.10.30.11 | 108 | Metrics dashboards |
| pve-exporter | VLAN 30 | 10.10.30.12 | 109 | Proxmox → Prometheus bridge |
| loki | VLAN 30 | 10.10.30.13 | 110 | Log aggregation |
| umami | VLAN 30 | 10.10.30.14 | 111 | Web analytics |
| opentrashmail | VLAN 60 | 10.10.60.10 | 112 | Disposable email (exilemail.xyz) |

## Cross-VLAN firewall exceptions

Configured in `BLOCK_IN` on the EdgeRouter. All other cross-VLAN new connections are dropped.

| Rule | Source | Destination | Port | Purpose |
|------|--------|-------------|------|---------|
| 3 | 10.10.30.0/24 | 192.168.1.85 | 8006 | Prometheus → Proxmox API |
| 4 | 10.10.30.0/24 | 10.10.20.10 | 8080 | Prometheus → cAdvisor |
| 5 | 10.10.20.0/24 | 10.10.30.13 | 3100 | Docker LXC → Loki |
