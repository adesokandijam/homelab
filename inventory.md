# Homelab Inventory

## Network Equipment

### Router
- **Raspberry Pi 5** - 2GB RAM, 64-bit Quad-Core SBC
  - **HAT:** ZDE ZP593 PCIe to 2.5G Ethernet Network Port + M.2 Key M NVMe SSD HAT Adapter
    - Supports M.2 NVMe 2230/2242/2260/2280 SSD
    - 2.5GbE network port
  - **Storage:** 128GB NVMe SSD
  - **OS:** OpenWrt

### Switch
- **TP-Link TL-SG608E** - Smart Managed Ethernet Switch
  - 8 Ports RJ45 Gigabit

---

## Compute Nodes

### Laptop
- **Lenovo ThinkPad T480s**
  - CPU: Intel i5-8250U (8th Gen)
  - RAM: 20GB
  - Storage: 512GB SSD

### Mini PCs
- **Lenovo ThinkCentre M720q**
  - CPU: Intel i5-8400T (8th Gen)
  - RAM: 16GB
  - Storage: 500GB SSD

- **HP EliteDesk 800 G4**
  - CPU: Intel i5-8500 @ 3.00GHz (8th Gen)
  - RAM: 16GB (8GB original + 8GB added)
  - Storage: 512GB NVMe SSD

- **GMKtec NucBox M5 Plus**
  - CPU: AMD Ryzen 7 5825U
  - RAM: 32GB (upgraded from 16GB)
  - Storage: 1TB SSD (512GB original + 512GB added)

### Workstation
- **HP Z440**
  - CPU: Intel 12-Core
  - RAM: 48GB
  - Storage: 240GB SSD + PCI to M2.NVME 512GB Drive
  - GPU: NVIDIA RTX 3060
  - AI Accelerator: NVIDIA Tesla P4
  - OS: Windows 10
  - Purpose: AI Workloads and DB on K8s

---

## Summary

| Category | Device Count |
|----------|--------------|
| Network Equipment | 2 |
| Compute Nodes | 5 |
| **Total Devices** | **7** |

### Total Resources
- **CPU Cores:** 50+ cores
- **RAM:** 112GB
- **Storage:** ~2.9TB SSD/NVMe
- **Network:** 2.5GbE routing + Gigabit switching
- **GPU:** RTX 3060 + Tesla P4