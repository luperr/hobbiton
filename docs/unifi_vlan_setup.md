# Unifi — WiFi VLAN Setup Runbook

Configure IoT and Guest SSIDs mapped to VLANs 40 and 50.
Complete the [router_vlan_setup.md](router_vlan_setup.md) first — the VLANs must exist
on the ER-X before the APs can tag traffic into them.

The Unifi controller is at `192.168.1.91`.

---

## Step 1 — Create Networks in Unifi

**Settings → Networks → Create New Network** (repeat for each):

### IoT Network

| Field | Value |
|---|---|
| Name | IoT |
| Purpose | Corporate |
| VLAN | 40 |
| Gateway/Subnet | 10.10.40.1/24 |
| DHCP Mode | None (ER-X handles DHCP) |

### Guest Network

| Field | Value |
|---|---|
| Name | Guest |
| Purpose | Guest |
| VLAN | 50 |
| Gateway/Subnet | 10.10.50.1/24 |
| DHCP Mode | None (ER-X handles DHCP) |

> Use **Corporate** purpose for IoT (not Guest) — Guest purpose in Unifi adds client
> isolation which prevents IoT devices from communicating with each other on the same
> VLAN. For smart home devices that need to discover each other (Chromecast, HomeKit,
> etc.) use Corporate. For the actual guest WiFi use Guest purpose.

---

## Step 2 — Create WiFi Networks

**Settings → WiFi → Create New WiFi Network** (repeat for each):

### IoT WiFi

| Field | Value |
|---|---|
| Name (SSID) | `YourNetwork-IoT` (or similar) |
| Password | Strong password — IoT devices rarely need you to type this |
| Network | IoT (VLAN 40) |
| Security Protocol | WPA2 |
| Hide SSID | Optional — IoT devices can still connect if hidden |
| Band | 2.4 GHz preferred (most IoT devices don't support 5 GHz) |

### Guest WiFi

| Field | Value |
|---|---|
| Name (SSID) | `YourNetwork-Guest` |
| Password | Something easy to share verbally |
| Network | Guest (VLAN 50) |
| Security Protocol | WPA2 |
| Hide SSID | No — guests need to find it |
| Band | Both (2.4 + 5 GHz) |

---

## Step 3 — Apply to APs

**Devices → select each AP → Config → WiFi**

Confirm both new SSIDs appear and are enabled on the AP. Unifi pushes config
automatically after saving — the AP will re-provision (brief disconnect ~30s).

---

## Step 4 — Verify

1. Connect a phone to `YourNetwork-IoT`
   - Should get an IP in `10.10.40.100–200`
   - `ping 1.1.1.1` should work
   - `ping 192.168.1.1` should fail

2. Connect a phone to `YourNetwork-Guest`
   - Should get an IP in `10.10.50.100–200`
   - Browser should reach internet
   - `ping 192.168.1.1` should fail
   - `ping 10.10.20.1` should fail

---

## mDNS / device discovery across VLANs (optional)

By default, mDNS (used by Chromecast, AirPlay, HomeKit, etc.) does not cross VLAN
boundaries. If you want to control IoT devices from your main LAN (e.g. cast from your
laptop on the main LAN to a Chromecast on VLAN 40), you need an mDNS repeater.

Options:
- **Avahi** on the Pi-hole LXC (`apt install avahi-daemon`) — repeats mDNS between
  interfaces. Lightweight and already on a device that serves both networks.
- **Unifi mDNS** — newer Unifi firmware has a built-in mDNS repeater under
  Settings → Services → mDNS. Enable and select the networks to bridge.

If you don't need cross-VLAN device discovery, skip this entirely.
