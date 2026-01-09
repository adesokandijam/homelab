# Network Troubleshooting Runbook

## Purpose
This runbook provides procedures for diagnosing and resolving common network issues in the homelab VLAN-segmented environment.

## Prerequisites
- Access to OpenWRT gateway (SSH or Web UI)
- Access to managed switch (Web UI)
- Access to Proxmox nodes
- Basic understanding of network topology and VLAN configuration

## Network Architecture Quick Reference

```
Internet → ISP Router → OpenWRT Gateway → Managed Switch → Proxmox Nodes
         192.168.55.1   192.168.55.95     (VLAN-aware)     (5 nodes)

VLANs:
- VLAN 1  (LAN):        192.168.2.0/24   - Untagged default
- VLAN 10 (Management): 192.168.10.0/24  - Proxmox, Switch
- VLAN 20 (Ops):        192.168.20.0/24  - Monitoring, DNS
- VLAN 30 (K8s):        192.168.30.0/24  - Kubernetes
- VLAN 50 (DMZ):        192.168.50.0/24  - Public services
```

---

## Common Issues and Resolution

### Issue 1: Cannot Access VM from Another VLAN

#### Symptoms
- Cannot ping VM from different VLAN
- Cannot access services across VLANs
- Connectivity works within same VLAN

#### Diagnosis
```bash
# From source VM/container
ping <destination-ip>

# Try to ping the gateway first
ping 192.168.XX.1  # Where XX is your VLAN

# Check routing
ip route
netstat -rn

# Check if firewall is blocking
# From OpenWRT gateway
iptables -L -v -n | grep <destination-ip>
```

#### Resolution Steps

1. **Verify gateway is reachable**:
   ```bash
   ping 192.168.XX.1  # Your VLAN gateway
   ```

2. **Check OpenWRT firewall rules**:
   ```bash
   # SSH to OpenWRT gateway
   ssh root@192.168.55.95
   
   # View firewall rules
   iptables -L FORWARD -v -n
   
   # Check zone forwarding
   cat /etc/config/firewall | grep -A 5 "config forwarding"
   ```

3. **Verify VLAN routing on OpenWRT**:
   ```bash
   # Check VLAN interfaces are up
   ip link show | grep eth1
   
   # Should see eth1.1, eth1.10, eth1.20, eth1.30, eth1.50
   ip addr show
   ```

4. **Check firewall zone configuration**:
   ```bash
   uci show firewall | grep zone
   
   # Verify forwarding rules exist
   uci show firewall | grep forwarding
   ```

5. **Temporarily disable firewall to test** (troubleshooting only):
   ```bash
   # Disable firewall temporarily
   /etc/init.d/firewall stop
   
   # Test connectivity
   ping <destination>
   
   # Re-enable firewall
   /etc/init.d/firewall start
   ```

6. **Add missing firewall rule** (if needed):
   ```bash
   # Example: Allow K8s VLAN to access Ops VLAN
   uci add firewall forwarding
   uci set firewall.@forwarding[-1].src='k8s'
   uci set firewall.@forwarding[-1].dest='ops'
   uci commit firewall
   /etc/init.d/firewall restart
   ```

---

### Issue 2: VM Not Getting DHCP Address

#### Symptoms
- VM has no IP address or shows APIPA (169.254.x.x)
- `dhclient` or DHCP request times out
- VM shows "No link" or "Cable unplugged"

#### Diagnosis
```bash
# From VM
ip addr show
ip link show

# Check DHCP client
sudo dhclient -v  # Verbose DHCP request
journalctl -u NetworkManager -f  # If using NetworkManager

# From OpenWRT
# Check DHCP logs
logread | grep dnsmasq
```

#### Resolution Steps

1. **Verify VM network configuration in Proxmox**:
   ```bash
   # From Proxmox node
   qm config <vmid> | grep net
   
   # Should show: net0: virtio=XX:XX:XX:XX:XX:XX,bridge=vmbr0,tag=XX
   # Verify VLAN tag is correct
   ```

2. **Check if VM network interface is up**:
   ```bash
   # In VM
   ip link show
   
   # Bring interface up if down
   sudo ip link set eth0 up
   ```

3. **Verify VLAN tag in Proxmox**:
   - Web UI: VM → Hardware → Network Device
   - Check "VLAN Tag" field matches intended VLAN

4. **Check OpenWRT DHCP configuration**:
   ```bash
   # SSH to OpenWRT
   cat /etc/config/dhcp
   
   # Verify DHCP is enabled for the VLAN
   uci show dhcp | grep -A 10 "interface='vlan10'"
   ```

5. **Restart DHCP server on OpenWRT**:
   ```bash
   /etc/init.d/dnsmasq restart
   ```

6. **Check switch port configuration**:
   - Log into TP-Link switch web UI
   - Verify port is configured for 802.1Q VLAN tagging
   - Ensure VLAN is in allowed list for the port

7. **Test with static IP**:
   ```bash
   # Temporarily assign static IP to isolate DHCP issue
   sudo ip addr add 192.168.XX.100/24 dev eth0
   sudo ip route add default via 192.168.XX.1
   
   # Test connectivity
   ping 192.168.XX.1
   ping 8.8.8.8
   ```

---

### Issue 3: Inter-VLAN Routing Not Working

#### Symptoms
- Can ping gateway but not other VLANs
- Firewall rules look correct
- Same-VLAN communication works fine

#### Diagnosis
```bash
# From source VM
traceroute <destination-ip>

# Should see first hop as gateway (192.168.XX.1)
# If it stops there, routing issue on gateway

# Check routing table
ip route

# From OpenWRT
cat /proc/sys/net/ipv4/ip_forward
# Should return: 1
```

#### Resolution Steps

1. **Verify IP forwarding is enabled**:
   ```bash
   # On OpenWRT
   sysctl net.ipv4.ip_forward
   
   # If not enabled
   echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
   sysctl -p
   ```

2. **Check VLAN interfaces on OpenWRT**:
   ```bash
   ip addr show | grep -E "(eth1\.|vlan)"
   
   # Verify all VLAN interfaces have IP addresses
   # eth1.1  -> 192.168.2.1
   # eth1.10 -> 192.168.10.1
   # eth1.20 -> 192.168.20.1
   # eth1.30 -> 192.168.30.1
   # eth1.50 -> 192.168.50.1
   ```

3. **Test routing from OpenWRT**:
   ```bash
   # From OpenWRT gateway
   ping -I br-vlan10 192.168.20.10  # Ping from VLAN 10 to VLAN 20
   ```

4. **Check NAT/masquerading**:
   ```bash
   iptables -t nat -L POSTROUTING -v -n
   
   # Should see masquerading rule for WAN
   ```

5. **Review firewall forwarding chains**:
   ```bash
   iptables -L FORWARD -v -n
   
   # Look for DROP rules that might be blocking traffic
   ```

---

### Issue 4: Cannot Access Internet from VM

#### Symptoms
- Can ping gateway (192.168.XX.1)
- Cannot ping 8.8.8.8 or external IPs
- Cannot resolve DNS names

#### Diagnosis
```bash
# From VM
ping 192.168.XX.1      # Gateway - should work
ping 192.168.55.1      # ISP router - might fail
ping 8.8.8.8           # Internet - fails

# Check DNS
nslookup google.com
dig google.com

# Check routing
ip route
```

#### Resolution Steps

1. **Verify default route**:
   ```bash
   # From VM
   ip route | grep default
   
   # Should point to VLAN gateway
   # If missing, add it
   sudo ip route add default via 192.168.XX.1
   ```

2. **Test from OpenWRT gateway**:
   ```bash
   # SSH to OpenWRT
   ping 8.8.8.8
   
   # If this fails, issue is with ISP router or WAN connection
   ```

3. **Check OpenWRT WAN connection**:
   ```bash
   # From OpenWRT
   ip addr show eth0  # WAN interface
   
   # Should have IP from ISP router (192.168.55.x)
   ping 192.168.55.1  # ISP router
   ping 8.8.8.8       # Internet
   ```

4. **Verify NAT is working**:
   ```bash
   # From OpenWRT
   iptables -t nat -L POSTROUTING -v -n
   
   # Should see MASQUERADE rule for WAN zone
   ```

5. **Check DNS configuration**:
   ```bash
   # From VM
   cat /etc/resolv.conf
   
   # Should point to either:
   # - Gateway: 192.168.XX.1 (OpenWRT DNS)
   # - Bind9: 192.168.20.X (if configured)
   ```

6. **Restart network on OpenWRT**:
   ```bash
   /etc/init.d/network restart
   ```

---

### Issue 5: Proxmox Node Cannot Reach Internet

#### Symptoms
- Proxmox node (management VLAN) cannot update packages
- Cannot ping external IPs from node
- VMs on the node can access internet

#### Diagnosis
```bash
# From Proxmox node
ping 192.168.10.1      # VLAN 10 gateway
ping 192.168.55.1      # ISP router
ping 8.8.8.8           # Internet

# Check network configuration
cat /etc/network/interfaces
ip addr show
ip route
```

#### Resolution Steps

1. **Verify Proxmox network configuration**:
   ```bash
   cat /etc/network/interfaces
   
   # Should have vmbr0.10 with correct IP and gateway
   # auto vmbr0.10
   # iface vmbr0.10 inet static
   #     address 192.168.10.XX/24
   #     gateway 192.168.10.1
   ```

2. **Check if management interface is up**:
   ```bash
   ip link show vmbr0.10
   
   # If down
   ifup vmbr0.10
   ```

3. **Verify routing table**:
   ```bash
   ip route
   
   # Should have default route via 192.168.10.1
   # If missing
   ip route add default via 192.168.10.1
   ```

4. **Check if gateway is reachable**:
   ```bash
   ping 192.168.10.1
   arping -I vmbr0.10 192.168.10.1
   ```

5. **Restart networking**:
   ```bash
   systemctl restart networking
   
   # Or
   ifreload -a
   ```

---

### Issue 6: Switch Port Not Passing VLAN Traffic

#### Symptoms
- Physical connection exists (link lights on)
- No traffic passing through
- Works when connected to different port

#### Diagnosis
- Check switch web UI for port status
- Verify VLAN membership
- Check port configuration

#### Resolution Steps

1. **Log into TP-Link switch web UI**:
   - URL: `http://192.168.10.253` (from Management VLAN)
   - Check port status: Switching → Port Monitoring

2. **Verify 802.1Q VLAN configuration**:
   - Navigate to: VLAN → 802.1Q VLAN
   - Verify port is member of required VLANs
   - Check if port is Tagged or Untagged correctly

3. **For Proxmox nodes (Ports 1-5)**:
   - Should be TAGGED for VLANs: 1, 10, 20, 30, 50
   - PVID: 1

4. **For Router uplink (Port 8)**:
   - Should be TAGGED for VLANs: 1, 10, 20, 30, 50
   - PVID: 1

5. **Save and reboot switch if needed**:
   - Save → System → Save Config
   - System Tools → Reboot

---

## Validation

### Network Health Checks

```bash
# From any VM in VLAN 30 (K8s)
ping 192.168.30.1      # ✓ Gateway should respond
ping 192.168.20.10     # ✓ Should reach Ops VLAN (allowed)
ping 192.168.10.10     # ✗ Should be blocked (K8s → Management)
ping 8.8.8.8           # ✓ Should reach Internet

# From Proxmox node (VLAN 10 - Management)
ping 192.168.10.1      # ✓ Gateway
ping 192.168.20.10     # ✓ Can reach Ops
ping 192.168.30.10     # ✓ Can reach K8s
ping 8.8.8.8           # ✓ Can reach Internet

# From OpenWRT gateway
ping 192.168.55.1      # ✓ ISP router
ping 8.8.8.8           # ✓ Internet
ip link | grep UP      # ✓ All VLAN interfaces should be UP
```

---

## Diagnostic Tools

### Packet Capture
```bash
# On OpenWRT
tcpdump -i eth1.10 -n  # Capture VLAN 10 traffic

# On Proxmox
tcpdump -i vmbr0.10 -n icmp  # Capture ICMP on VLAN 10

# In VM
sudo tcpdump -i eth0 -n  # Capture all traffic
```

### Network Performance Testing
```bash
# Install iperf3
# On server VM
iperf3 -s

# On client VM
iperf3 -c <server-ip>
```

### DNS Testing
```bash
# Test DNS resolution
nslookup homelab.local 192.168.20.10
dig @192.168.20.10 pve1.homelab.local

# Test reverse DNS
dig -x 192.168.10.10
```

---

## Related Documentation
- [Network Architecture](../Networking.md)
- [Enterprise Plan - Network Layer](../ENTERPRISE_PLAN.md#12-network-layer)
- [OpenWRT Configuration Files](../openwrt/)

---

## Version History
- **v1.0** (January 2026): Initial runbook creation
- **Last Tested**: Not yet tested (documentation only)
- **Next Review**: April 2026
