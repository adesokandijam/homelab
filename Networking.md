# Homelab Network Architecture

## 1. Overview

**Goal:** A high-availability Kubernetes and AI cluster segregated from the home network.

**Topology:** "Router-on-a-Stick" gateway managing a VLAN-aware cluster behind the ISP router.

![Homelab Network Architecture Diagram](images/Homelab%20Networking.drawio.png)

### Hardware Stack

* **Edge Router:** ISP Home Router (Handles Home/Personal devices - `192.168.55.1`)
* **Lab Gateway:** Raspberry Pi 5 with USB Ethernet Adapter (OpenWRT - `192.168.55.95`)
  * `eth0`: WAN Uplink (to ISP Router)
  * `eth1` (Hat Ethernet Port): LAN Trunk (Downlink to Switch)
* **Switch:** TP-Link TL-SG108E 8-Port Managed Switch (802.1Q VLANs)
* **Compute:** 5x Proxmox Nodes (Mini PCs/Laptops)

---

## 2. VLAN Topology

The lab network is segmented into isolated tiers. Traffic routing between these tiers is handled strictly by the OpenWRT gateway.

| VLAN ID | Name         | Subnet            | Gateway        | Use Case |
| :------ | :----------- | :---------------- | :------------- | :------- |
| **1**   | `LAN`        | `192.168.2.0/24`  | `192.168.2.1`  | **Untagged:** Default VLAN for unmanaged devices. |
| **10**  | `Management` | `192.168.10.0/24` | `192.168.10.1` | **Infra:** Proxmox Hosts, Switch Management, Ansible Node. |
| **20**  | `Ops`        | `192.168.20.0/24` | `192.168.20.1` | **Services:** Bind9 DNS, Prometheus, Grafana, Monitoring. |
| **30**  | `K8s`        | `192.168.30.0/24` | `192.168.30.1` | **Compute:** K8s Masters, Workers, AI Nodes. |
| **50**  | `DMZ`        | `192.168.50.0/24` | `192.168.50.1` | **Public:** Internet-facing services, reverse proxies. |

---

## 3. IP Allocation Strategy

**Rule:** DHCP is authoritative for the lab. Fixed IPs are handled via static leases within OpenWRT or static configuration on hosts.

### Per-VLAN IP Ranges

* **.1**: Gateway (OpenWRT Interface IP)
* **.2 - .99**: **Static Reserve** (Manually configured in VM/LXC)
* **.100 - .250**: **DHCP Pool** (Dynamic allocation for test VMs/containers)
* **.253**: Managed Switch IP (VLAN 10 only)

### Proxmox Node Assignments (VLAN 10)

| Node Name | Management IP    | External Access URL               |
| :-------- | :--------------- | :-------------------------------- |
| **Node 1** | `192.168.10.10` | `https://192.168.55.95:8010`     |
| **Node 2** | `192.168.10.20` | `https://192.168.55.95:8020`     |
| **Node 3** | `192.168.10.30` | `https://192.168.55.95:8030`     |
| **Node 4** | `192.168.10.40` | `https://192.168.55.95:8040`     |
| **Node 5** | `192.168.10.50` | `https://192.168.55.95:8050`     |

*Note: External access uses port forwarding configured on OpenWRT to access Proxmox from upstream network.*

---

## 4. Switch Configuration (TP-Link TL-SG108E)

**802.1Q VLAN Configuration:**

* **Port 1-5 (Proxmox Nodes):** TAGGED for VLANs 1, 10, 20, 30, 50
* **Port 6 (K8S Worker):** Access Port VLAN 30
* **Ports 8 (Router Uplink - Router on a Stick for Multiple VLANs):** TAGGED for VLANs 1, 10, 20, 30, 50
  * *Important: Ports must be Tagged because Proxmox is "VLAN Aware" and handles tagging internally.*
* **Ports 7:** Available for expansion

---

## 5. Configuration Snippets

### A. OpenWRT (Gateway)

**Network Interfaces** (`/etc/config/network` logic):

* **WAN:** `eth0` → Firewall Zone: `WAN` (DHCP Client from ISP Router `192.168.55.1`)
* **LAN Trunk:** `eth1` (USB Green adapter - Physical link to switch port 1)
* **VLAN Interfaces (Virtual):**
  * `eth1.1` → Firewall Zone: `LAN` (Static `192.168.2.1`)
  * `eth1.10` → Firewall Zone: `Management` (Static `192.168.10.1`)
  * `eth1.20` → Firewall Zone: `Ops` (Static `192.168.20.1`)
  * `eth1.30` → Firewall Zone: `K8s` (Static `192.168.30.1`)
  * `eth1.50` → Firewall Zone: `DMZ` (Static `192.168.50.1`)

**Firewall Rules** (`/etc/config/firewall` logic):

1. **Global:** Masquerading (NAT) enabled on WAN zone
2. `Management` → Allow to `ALL` (Ansible controller needs full reachability)
3. `K8s` → Allow to `Ops` (For DNS resolution and metric scraping)
4. `K8s` → Allow to `WAN` (For pulling container images)
5. `K8s` → **BLOCK** to `Management` (Security boundary to protect hypervisors)
6. `DMZ` → Allow to `WAN` (Public services can access internet)
7. `DMZ` → **BLOCK** to `Management` (Security isolation)
8. `Ops` → Allow to `Management` (Monitoring can access infrastructure)

**Port Forwarding Rules** (Remote Access):

* `WAN:8010` → `192.168.10.10:8006` (Proxmox Node 1)
* `WAN:8020` → `192.168.10.20:8006` (Proxmox Node 2)
* `WAN:8030` → `192.168.10.30:8006` (Proxmox Node 3)
* `WAN:8040` → `192.168.10.40:8006` (Proxmox Node 4)
* `WAN:8050` → `192.168.10.50:8006` (Proxmox Node 5)

### B. Proxmox (Hypervisor Nodes)

**File:** `/etc/network/interfaces`

**Concept:** The Linux bridge must be VLAN-Aware to accept tagged frames from the switch and pass them to VMs based on their configuration.

```bash
auto lo
iface lo inet loopback

# Physical Interface (Do NOT assign IP here)
iface nic0 inet manual

# VLAN-Aware Bridge
auto vmbr0
iface vmbr0 inet manual
    bridge-ports nic0          # Your Physical Ethernet Port ID
    bridge-stp off
    bridge-fd 0
    # CRITICAL: Enables 802.1Q tagging on the bridge
    bridge-vlan-aware yes
    # Allow these VLAN tags to pass through
    bridge-vids 1 10 20 30 50

# Management Interface (VLAN 10)
auto vmbr0.10
iface vmbr0.10 inet static
    address 192.168.10.XX/24   # Replace XX: 10, 20, 30, 40, 50
    gateway 192.168.10.1       # Pointing to RPi OpenWRT
```

**VM/LXC Configuration in Proxmox GUI:**

When creating VMs or containers, assign them to the appropriate VLAN:

* **Bridge:** `vmbr0`
* **VLAN Tag:** Enter `10`, `20`, `30`, or `50` depending on the service tier
* The VM will receive an IP from that VLAN's DHCP pool or use static configuration

---

## 6. Security Boundaries

The firewall implements a zero-trust model between tiers:

```
┌─────────────┐
│   Internet  │
└──────┬──────┘
       │
┌──────▼──────────────────────┐
│   WAN (ISP Router)          │
└──────┬──────────────────────┘
       │
┌──────▼──────────────────────┐
│   OpenWRT Gateway (RPi)     │
│   - NAT/Firewall            │
│   - Inter-VLAN Routing      │
└──────┬──────────────────────┘
       │ (Trunk - All VLANs)
┌──────▼──────────────────────┐
│   Managed Switch            │
└─┬───┬───┬───┬───┬───────────┘
  │   │   │   │   │
  ▼   ▼   ▼   ▼   ▼
 Node1-5 (Proxmox Cluster)
```

**Traffic Flow Rules:**

* ✅ Management → Everything (Admin access)
* ✅ Ops → Management (Monitoring)
* ✅ K8s → Ops (DNS, Metrics)
* ✅ K8s → WAN (Internet)
* ❌ K8s → Management (Isolation)
* ✅ DMZ → WAN (Public services)
* ❌ DMZ → Management (Isolation)
* ❌ DMZ → K8s (Isolation)

---

## 7. DNS and Service Discovery

**Primary DNS:** Bind9 running in `Ops` VLAN (`192.168.20.x`)

**DNS Zones:**

* `homelab.local` - Internal services
* Reverse zones for all VLANs

**Key Records:**

```
# Management
pve1.homelab.local    → 192.168.10.10
pve2.homelab.local    → 192.168.10.20
...

# Operations
prometheus.homelab.local → 192.168.20.10
grafana.homelab.local    → 192.168.20.20

# Kubernetes
k8s-master.homelab.local → 192.168.30.10
```

---

## 8. Troubleshooting Guide

### Cannot reach Proxmox web interface from WAN

1. Verify OpenWRT port forwarding is configured
2. Check firewall allows WAN → Management on port 8006
3. Confirm Proxmox is listening on correct interface: `netstat -tlnp | grep 8006`

### VMs not getting DHCP addresses

1. Verify VLAN tag is set correctly in Proxmox VM config
2. Check OpenWRT DHCP is enabled for that VLAN
3. Confirm switch port is tagged for the VLAN

### Inter-VLAN routing not working

1. Test gateway reachability: `ping 192.168.XX.1` from VM
2. Check OpenWRT firewall rules: `iptables -L -v -n`
3. Verify VLAN interfaces are up on OpenWRT: `ip link show`

### Bridge not passing tagged traffic

1. Confirm `bridge-vlan-aware yes` is set in `/etc/network/interfaces`
2. Verify VLANs are in `bridge-vids`: `bridge vlan show`
3. Restart networking: `systemctl restart networking`

---

## 9. Future Enhancements

* **HA OpenWRT:** Add second RPi5 with VRRP/CARP for gateway redundancy
* **10GbE Uplink:** Upgrade switch and node NICs for higher throughput
* **External DNS:** Expose internal services via Cloudflare Tunnels
* **VPN Access:** WireGuard for secure remote access to Management VLAN
* **Additional VLANs:** 
  * VLAN 60 - IoT devices
  * VLAN 70 - Guest network