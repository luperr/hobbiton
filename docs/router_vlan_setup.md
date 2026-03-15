# EdgeRouter X — VLAN Setup Runbook

This is a manual step. The EdgeRouter X is not managed by Ansible.
Complete this runbook **before** running the `configure_network.yml` Ansible playbook (Stage 8).

For Unifi AP / WiFi SSID configuration see [unifi_vlan_setup.md](unifi_vlan_setup.md).

---

## Network overview

| VLAN | Name | Subnet | Gateway | Purpose |
|---|---|---|---|---|
| untagged | Management | 192.168.1.0/24 | 192.168.1.1 | Main LAN — existing, unchanged |
| 10 | Isolated | 192.168.10.0/24 | 192.168.10.1 | Game server — existing, unchanged |
| 20 | Services | 10.10.20.0/24 | 10.10.20.1 | Docker app LXC on Proxmox |
| 30 | Monitoring | 10.10.30.0/24 | 10.10.30.1 | Prometheus/Grafana LXC on Proxmox |
| 40 | IoT | 10.10.40.0/24 | 10.10.40.1 | Smart home devices — WiFi via Unifi AP |
| 50 | Guest | 10.10.50.0/24 | 10.10.50.1 | Visitor internet — WiFi via Unifi AP |

### Trunk port assignment

| Port | Connected to | Carries VLANs |
|---|---|---|
| eth0 | WAN (ISP) | — |
| eth2 | Proxmox host (eno1) | 20, 30 |
| eth4 | Unifi AP (PoE) | 40, 50 |
| eth1/eth3 | Other LAN devices | untagged only |

VLANs 10, 40, 50 are not bridged into Proxmox — they route entirely through the ER-X.

---

## Firewall chains (reused across all new VLANs)

The existing `BLOCK_IN` and `BLOCK_LOCAL` chains already do the right thing:

| Chain | Behaviour |
|---|---|
| `BLOCK_IN` | Accepts established/related; drops new connections to all private ranges (10/8, 172.16/12, 192.168/16). Internet passes. |
| `BLOCK_LOCAL` | Drops all traffic to the router itself except DNS (53) and DHCP (67). |

All four new VLANs use these same chains. No new firewall rules needed.

**IoT note:** "Calling home" blocking is handled at the DNS layer via Pi-hole blocklists, not firewall rules. IoT devices get Pi-hole (`192.168.1.87`) as their DNS server. Blocked domains return NXDOMAIN — the device never gets an IP to connect to. See Pi-hole IoT blocklist recommendations at the bottom of this doc.

---

## Step 1 — Add VLAN trunk IDs to eth2 (Proxmox)

```bash
configure

set interfaces switch switch0 switch-port interface eth2 vlan vid 20
set interfaces switch switch0 switch-port interface eth2 vlan vid 30
```

---

## Step 2 — Add VLAN trunk IDs to eth4 (Unifi AP)

```bash
set interfaces switch switch0 switch-port interface eth4 vlan vid 40
set interfaces switch switch0 switch-port interface eth4 vlan vid 50
```

---

## Step 3 — Create VLAN interfaces

```bash
# VLAN 20 — Services (Proxmox)
set interfaces switch switch0 vif 20 address 10.10.20.1/24
set interfaces switch switch0 vif 20 description 'Services VLAN 20'
set interfaces switch switch0 vif 20 mtu 1500
set interfaces switch switch0 vif 20 firewall in name BLOCK_IN
set interfaces switch switch0 vif 20 firewall local name BLOCK_LOCAL

# VLAN 30 — Monitoring (Proxmox)
set interfaces switch switch0 vif 30 address 10.10.30.1/24
set interfaces switch switch0 vif 30 description 'Monitoring VLAN 30'
set interfaces switch switch0 vif 30 mtu 1500
set interfaces switch switch0 vif 30 firewall in name BLOCK_IN
set interfaces switch switch0 vif 30 firewall local name BLOCK_LOCAL

# VLAN 40 — IoT
set interfaces switch switch0 vif 40 address 10.10.40.1/24
set interfaces switch switch0 vif 40 description 'IoT VLAN 40'
set interfaces switch switch0 vif 40 mtu 1500
set interfaces switch switch0 vif 40 firewall in name BLOCK_IN
set interfaces switch switch0 vif 40 firewall local name BLOCK_LOCAL

# VLAN 50 — Guest
set interfaces switch switch0 vif 50 address 10.10.50.1/24
set interfaces switch switch0 vif 50 description 'Guest VLAN 50'
set interfaces switch switch0 vif 50 mtu 1500
set interfaces switch switch0 vif 50 firewall in name BLOCK_IN
set interfaces switch switch0 vif 50 firewall local name BLOCK_LOCAL
```

---

## Step 4 — DHCP scopes

```bash
# VLAN 20 — Services
set service dhcp-server shared-network-name VLAN20 authoritative enable
set service dhcp-server shared-network-name VLAN20 subnet 10.10.20.0/24 default-router 10.10.20.1
set service dhcp-server shared-network-name VLAN20 subnet 10.10.20.0/24 dns-server 192.168.1.87
set service dhcp-server shared-network-name VLAN20 subnet 10.10.20.0/24 lease 86400
set service dhcp-server shared-network-name VLAN20 subnet 10.10.20.0/24 start 10.10.20.100 stop 10.10.20.200

# VLAN 30 — Monitoring
set service dhcp-server shared-network-name VLAN30 authoritative enable
set service dhcp-server shared-network-name VLAN30 subnet 10.10.30.0/24 default-router 10.10.30.1
set service dhcp-server shared-network-name VLAN30 subnet 10.10.30.0/24 dns-server 192.168.1.87
set service dhcp-server shared-network-name VLAN30 subnet 10.10.30.0/24 lease 86400
set service dhcp-server shared-network-name VLAN30 subnet 10.10.30.0/24 start 10.10.30.100 stop 10.10.30.200

# VLAN 40 — IoT (Pi-hole as DNS for blocking)
set service dhcp-server shared-network-name VLAN40 authoritative enable
set service dhcp-server shared-network-name VLAN40 subnet 10.10.40.0/24 default-router 10.10.40.1
set service dhcp-server shared-network-name VLAN40 subnet 10.10.40.0/24 dns-server 192.168.1.87
set service dhcp-server shared-network-name VLAN40 subnet 10.10.40.0/24 lease 86400
set service dhcp-server shared-network-name VLAN40 subnet 10.10.40.0/24 start 10.10.40.100 stop 10.10.40.200

# VLAN 50 — Guest (Pi-hole or use 1.1.1.1 for clean public DNS)
set service dhcp-server shared-network-name VLAN50 authoritative enable
set service dhcp-server shared-network-name VLAN50 subnet 10.10.50.0/24 default-router 10.10.50.1
set service dhcp-server shared-network-name VLAN50 subnet 10.10.50.0/24 dns-server 1.1.1.1
set service dhcp-server shared-network-name VLAN50 subnet 10.10.50.0/24 dns-server 1.0.0.1
set service dhcp-server shared-network-name VLAN50 subnet 10.10.50.0/24 lease 3600
set service dhcp-server shared-network-name VLAN50 subnet 10.10.50.0/24 start 10.10.50.100 stop 10.10.50.200
```

> Guest lease time is 1 hour (3600s) vs 24 hours for other VLANs — reduces stale leases from transient devices.
> Guest DNS uses Cloudflare (1.1.1.1) directly rather than Pi-hole — keeps guest traffic out of your internal DNS.

---

## Step 5 — DNS forwarding on new VLANs

```bash
set service dns forwarding listen-on switch0.20
set service dns forwarding listen-on switch0.30
set service dns forwarding listen-on switch0.40
# switch0.50 omitted — guest uses 1.1.1.1 directly, not the router DNS
```

---

## Step 6 — Commit and save

```bash
commit ; save
exit
```

---

## Step 7 — Verify

```bash
# On ER-X — confirm all vif interfaces are up
show interfaces switch switch0

# Check DHCP leases after connecting a test device to each VLAN
show dhcp leases
```

From a device on any new VLAN:
```bash
ping 1.1.1.1          # Should succeed — internet reachable
ping 192.168.1.1      # Should fail — main LAN blocked
ping 10.10.20.1       # Should fail from VLAN 30/40/50 — cross-VLAN blocked
```

---

## Rollback

```bash
configure

delete interfaces switch switch0 vif 20
delete interfaces switch switch0 vif 30
delete interfaces switch switch0 vif 40
delete interfaces switch switch0 vif 50
delete interfaces switch switch0 switch-port interface eth2 vlan vid 20
delete interfaces switch switch0 switch-port interface eth2 vlan vid 30
delete interfaces switch switch0 switch-port interface eth4 vlan vid 40
delete interfaces switch switch0 switch-port interface eth4 vlan vid 50
delete service dhcp-server shared-network-name VLAN20
delete service dhcp-server shared-network-name VLAN30
delete service dhcp-server shared-network-name VLAN40
delete service dhcp-server shared-network-name VLAN50
delete service dns forwarding listen-on switch0.20
delete service dns forwarding listen-on switch0.30
delete service dns forwarding listen-on switch0.40

commit ; save
exit
```

Existing LAN, VLAN 10 (game server), and all current devices are unaffected.

---

## IP reference

| Host | VLAN | IP | Notes |
|---|---|---|---|
| ER-X gateway | 20 | 10.10.20.1 | |
| ER-X gateway | 30 | 10.10.30.1 | |
| ER-X gateway | 40 | 10.10.40.1 | |
| ER-X gateway | 50 | 10.10.50.1 | |
| Proxmox host (vmbr1) | 20 | 10.10.20.2 | Static, set in /etc/network/interfaces |
| Proxmox host (vmbr2) | 30 | 10.10.30.2 | Static, set in /etc/network/interfaces |
| docker-host LXC | 20 | 10.10.20.10 | Static, set in Proxmox |
| monitoring LXC | 30 | 10.10.30.10 | Static, set in Proxmox |
| IoT devices | 40 | 10.10.40.100–200 | DHCP |
| Guest devices | 50 | 10.10.50.100–200 | DHCP |

---

## Pi-hole IoT blocklists

Add these to Pi-hole (Group Management → Adlists) for IoT telemetry blocking:

| List | Targets |
|---|---|
| `https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts` | General ads/tracking (already default) |
| `https://raw.githubusercontent.com/StevenBlack/hosts/master/alternates/fakenews-gambling-porn/hosts` | Extended |
| `https://perflyst.github.io/PiHoleBlocklist/SmartTV.txt` | Smart TV telemetry |
| `https://raw.githubusercontent.com/crazy-max/WindowsSpyBlocker/master/data/hosts/spy.txt` | Windows/IoT telemetry |

After adding, run **Update Gravity** in Pi-hole to activate.
