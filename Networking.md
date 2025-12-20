# Homelab Network Architecture

## 1. Overview
**Goal:** A high-availability Kubernetes and AI cluster segregated from the home network.
**Topology:** "Router-on-a-Stick" gateway managing a VLAN-aware cluster behind the ISP router.

![Homelab Network Architecture Diagram](images/Homelab%20Networking.drawio.png)

### Hardware Stack
* **Edge Router:** ISP Home Router (Handles Home/Personal devices)
* **Lab Gateway:** Raspberry Pi 5 (OpenWRT)
    * `eth0`: WAN Uplink (to ISP Router)
    * `eth1`: LAN Trunk (Downlink to Switch)
* **Switch:** TP-Link 8-Port Managed Switch (802.1Q VLANs)
* **Compute:** 3x Mini PCs + 1x Laptop (Proxmox Cluster)

## 2. VLAN Topology
The lab network is segmented into isolated tiers. Traffic routing between these tiers is handled strictly by the OpenWRT gateway.

| VLAN ID | Name   | Subnet        | Gateway     | Use Case |
| :---    | :---   | :---          | :---        | :--- |
| **20** | `MGMT` | `10.0.20.0/24`| `10.0.20.1` | **Infra:** Proxmox Hosts, Switch, Ansible Node. |
| **30** | `OPS`  | `10.0.30.0/24`| `10.0.30.1` | **Services:** Bind9 DNS, Prometheus, Grafana. |
| **40** | `K8S`  | `10.0.40.0/24`| `10.0.40.1` | **Compute:** K8s Masters, Workers, AI Nodes. |

## 3. IP Allocation Strategy
**Rule:** DHCP is authoritative for the lab. Fixed IPs are handled via specific ranges or static leases within OpenWRT.

* **.1**: Gateway (OpenWRT Interface IP)
* **.2 - .99**: **Static Reserve** (Manually configured in VM/LXC `interfaces` or netplan)
* **.100 - .250**: **DHCP Pool** (Dynamic allocation for test VMs/containers)
* **.253**: Managed Switch IP (VLAN 20 only)

## 4. Configuration Snippets

### A. OpenWRT (Gateway)
**Network Interfaces (`/etc/config/network` logic):**
* **WAN:** `eth0` -> Firewall Zone: `WAN` (DHCP Client from ISP)
* **LAN Trunk:** `eth1` (Physical link to switch port 1)
* **VLAN Interfaces (Virtual):**
    * `eth1.20` -> Firewall Zone: `MGMT` (Static `10.0.20.1`)
    * `eth1.30` -> Firewall Zone: `OPS` (Static `10.0.30.1`)
    * `eth1.40` -> Firewall Zone: `K8S` (Static `10.0.40.1`)

**Firewall Rules (`/etc/config/firewall` logic):**
1.  **Global:** Masquerading (NAT) enabled on WAN zone.
2.  `MGMT` -> Allow to `ALL` (Ansible controller needs full reachability).
3.  `K8S` -> Allow to `OPS` (For DNS resolution and metric scraping).
4.  `K8S` -> Allow to `WAN` (For pulling container images).
5.  `K8S` -> **BLOCK** to `MGMT` (Security boundary to protect hypervisors).

### B. Proxmox (Hypervisor Nodes)
**File:** `/etc/network/interfaces`
**Concept:** The Linux bridge must be VLAN-Aware to accept tagged frames from the switch and pass them to VMs based on their configuration.

```bash
auto vmbr0
iface vmbr0 inet manual
    bridge-ports eno1        # Your Physical Ethernet Port ID
    bridge-stp off
    bridge-fd 0
    # CRITICAL: Enables 802.1Q tagging on the bridge
    bridge-vlan-aware yes

# Host Management Interface (Sitting logically on VLAN 20)
auto vmbr0.20
iface vmbr0.20 inet static
    address 10.0.20.X/24     # .11, .12, .13, etc.
    gateway 10.0.20.1        # Pointing to RPi OpenWRT