# Homelab Network Architecture

## 1. Overview
**Goal:** A high-availability Kubernetes and AI cluster segregated from the home network.
**Topology:** "Router-on-a-Stick" (or Dual-NIC Gateway) managing a VLAN-aware cluster.

### Hardware Stack
* **Gateway:** Raspberry Pi 5 (OpenWRT)
    * `eth0`: WAN (ISP Uplink)
    * `eth1`: LAN Trunk (Downlink to Switch)
* **Switch:** TP-Link Managed Switch (802.1Q VLANs)
* **Compute:** 3x Mini PCs + 1x Laptop (Proxmox Cluster)

## 2. VLAN Topology
The network is segmented into isolated tiers. Traffic routing is handled by OpenWRT.

| VLAN ID | Name   | Subnet        | Gateway     | Use Case |
| :---    | :---   | :---          | :---        | :--- |
| **-** | `WAN`  | *ISP DHCP* | *ISP IP* | Internet Uplink (`eth0`) |
| **20** | `MGMT` | `10.0.20.0/24`| `10.0.20.1` | **Infra:** Proxmox Hosts, Switch, Ansible Node. |
| **30** | `OPS`  | `10.0.30.0/24`| `10.0.30.1` | **Services:** Bind9 DNS, Prometheus, Grafana. |
| **40** | `K8S`  | `10.0.40.0/24`| `10.0.40.1` | **Compute:** K8s Masters, Workers, AI Nodes. |

*(Note: VLAN 10 is reserved for Home/Personal devices directly on ISP Router)*

## 3. IP Allocation Strategy
**Rule:** DHCP is authoritative. Fixed IPs are handled via specific ranges or static leases.

* **.1**: Gateway (OpenWRT)
* **.2 - .99**: **Static Reserve** (Manually configured in VM/LXC `interfaces`)
* **.100 - .250**: **DHCP Pool** (Dynamic allocation for test VMs)
* **.253**: Network Switch Mgmt IP

## 4. Configuration Snippets

### A. OpenWRT (Router)
**Network Interfaces (`/etc/config/network` logic):**
* **WAN:** `eth0` -> Firewall Zone: `WAN`
* **LAN Trunk:** `eth1` (Physical)
* **VLAN Interfaces:**
    * `eth1.20` -> Firewall Zone: `MGMT`
    * `eth1.30` -> Firewall Zone: `OPS`
    * `eth1.40` -> Firewall Zone: `K8S`

**Firewall Rules (`/etc/config/firewall` logic):**
1.  `MGMT` -> Allow to `ALL` (Ansible access).
2.  `K8S` -> Allow to `OPS` (DNS/Metrics).
3.  `K8S` -> Allow to `WAN` (Image Pulls).
4.  `K8S` -> **BLOCK** to `MGMT` (Security boundary).

### B. Proxmox (Hypervisor)
**File:** `/etc/network/interfaces`
**Concept:** Bridge must be VLAN-Aware to pass tags from VMs to the Switch.

```bash
auto vmbr0
iface vmbr0 inet manual
    bridge-ports eno1        # Physical Port
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes    # <--- CRITICAL

# Host Management IP (Sitting on VLAN 20)
auto vmbr0.20
iface vmbr0.20 inet static
    address 10.0.20.x/24
    gateway 10.0.20.1