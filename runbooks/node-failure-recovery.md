# Node Failure Recovery Runbook

## Purpose
This runbook provides the procedure for handling and recovering from a Proxmox node failure in the homelab cluster.

## Prerequisites
- Access to Proxmox web interface or SSH
- Access to other healthy cluster nodes
- Understanding of VM/LXC locations and configurations

## Severity Classification

### P1 - Critical (Single node down, HA not functioning)
- **Response Time**: Immediate
- **Impact**: Loss of VMs/services if HA fails

### P2 - High (Single node down, HA working)
- **Response Time**: Within 1 hour
- **Impact**: Reduced capacity, no service impact

### P3 - Medium (Node maintenance)
- **Response Time**: Scheduled maintenance window
- **Impact**: Planned migration

---

## Procedure

### 1. Detection and Assessment

#### 1.1 Confirm Node Failure
```bash
# From any healthy Proxmox node
pvecm status

# Expected output shows node as offline
# Check node connectivity
ping 192.168.10.XX  # Replace XX with node IP
ssh root@192.168.10.XX
```

#### 1.2 Identify Affected VMs/Containers
```bash
# List VMs on the failed node
pvesh get /cluster/resources --type vm | grep "node-name"

# Or via web UI:
# Datacenter → Node → Summary (check VM list)
```

#### 1.3 Check HA Status
```bash
# Verify HA manager status
ha-manager status

# Check which VMs have HA enabled
pvesh get /cluster/ha/resources
```

### 2. Immediate Response (If HA Configured)

#### 2.1 Automatic Failover
- HA should automatically migrate VMs to healthy nodes within 2-5 minutes
- Monitor migration progress in web UI: Datacenter → HA → Resources

#### 2.2 Verify VM Status
```bash
# Check if VMs have been migrated
pvesh get /cluster/resources --type vm

# Verify VM is running
qm status <vmid>
```

### 3. Manual Recovery (If HA Not Configured)

#### 3.1 Fence the Failed Node
```bash
# From a healthy node
pvecm expected 1  # Adjust based on remaining nodes
```

#### 3.2 Option A: Restore from Backup (RTO: 30-60 minutes)
```bash
# List available backups
vzdump list

# Restore VM from backup
qmrestore /var/lib/vz/dump/vzdump-qemu-XXX-YYYY.vma.zst <new-vmid>

# Start the restored VM
qm start <new-vmid>
```

#### 3.3 Option B: Force Start VM on Another Node (if shared storage exists)
```bash
# This only works with shared storage (Ceph, NFS)
qm unlock <vmid>
qm start <vmid> --node <target-node>
```

### 4. Troubleshoot Failed Node

#### 4.1 Physical Checks
- [ ] Check power LED (is the device on?)
- [ ] Check network link lights
- [ ] Check display output if available
- [ ] Listen for POST beeps

#### 4.2 Remote Management (if available)
```bash
# IPMI or iLO access (if configured)
ipmitool -I lanplus -H <ipmi-ip> -U <user> -P <pass> power status
ipmitool -I lanplus -H <ipmi-ip> -U <user> -P <pass> chassis status
```

#### 4.3 Common Failure Scenarios

##### Network Connectivity Issue
```bash
# From another node, check network
ping 192.168.10.XX
traceroute 192.168.10.XX

# Check switch port status
# Log into managed switch and verify port is up
```

##### System Hang/Kernel Panic
```bash
# Hard reboot the node (last resort)
ipmitool -I lanplus -H <ipmi-ip> -U <user> -P <pass> power reset

# Or physical power cycle if no remote management
```

##### Storage Failure
```bash
# SSH to node (if accessible)
df -h
zpool status  # If using ZFS
lvs           # If using LVM
dmesg | grep -i error
```

### 5. Node Recovery

#### 5.1 Bring Node Back Online
```bash
# Once node is accessible
systemctl status pve-cluster
systemctl status pvedaemon
systemctl status pveproxy

# If services are down
systemctl restart pve-cluster
systemctl restart pvedaemon
systemctl restart pveproxy
```

#### 5.2 Rejoin Cluster
```bash
# Check cluster status
pvecm status

# If node shows as offline
pvecm add <master-node-ip>

# Verify quorum is restored
pvecm expected <total-nodes>
```

#### 5.3 Verify Node Health
```bash
# Check system resources
uptime
free -h
df -h

# Check for errors
journalctl -xe
dmesg | tail -50

# Verify Proxmox services
systemctl status pve-cluster pvedaemon pveproxy pvestatd
```

### 6. Post-Recovery Actions

#### 6.1 Migrate VMs Back (if desired)
```bash
# Online migration (VM stays running)
qm migrate <vmid> <target-node>

# Or via web UI:
# Right-click VM → Migrate → Select target node
```

#### 6.2 Rebalance Cluster
- Review VM distribution across nodes
- Ensure even resource utilization
- Update HA groups if needed

#### 6.3 Verify Backups
```bash
# Ensure backups are current
vzdump --mode snapshot --storage <backup-storage>

# Verify backup integrity
vzdump list | tail -20
```

---

## Validation

### Success Criteria
- [ ] All nodes show "online" in `pvecm status`
- [ ] Quorum is restored (expected votes matches total votes)
- [ ] All VMs are running on expected nodes
- [ ] No HA warnings in web UI
- [ ] Cluster health is "green"
- [ ] Backup schedule is current

### Health Checks
```bash
# Cluster status
pvecm status

# Resource usage
pvesh get /cluster/resources

# HA status
ha-manager status

# System health
cat /etc/pve/.version
pveversion -v
```

---

## Rollback

If node recovery causes issues:

1. **Fence the problematic node**:
   ```bash
   pvecm delnode <nodename>
   ```

2. **Keep VMs on current healthy nodes**: Do not migrate back

3. **Schedule maintenance**: Plan proper troubleshooting during maintenance window

---

## Common Issues and Solutions

### Issue: Split-brain Scenario
**Symptoms**: Multiple nodes think they are master, cluster instability

**Solution**:
```bash
# On all nodes except one
systemctl stop pve-cluster
systemctl stop pvedaemon
systemctl stop pveproxy

# On the master node
pvecm expected 1

# Slowly bring other nodes back online one by one
```

### Issue: VM Won't Start After Migration
**Symptoms**: VM shows "locked" status

**Solution**:
```bash
qm unlock <vmid>
qm start <vmid>
```

### Issue: Storage Not Available on Recovered Node
**Symptoms**: Can't see shared storage

**Solution**:
```bash
pvesm status
pvesm scan <storage-type>
```

---

## Related Documentation
- [Proxmox High Availability](https://pve.proxmox.com/wiki/High_Availability)
- [Backup and Restore](backup-restore.md)
- [Network Troubleshooting](network-troubleshooting.md)
- [Enterprise Plan - Disaster Recovery](../ENTERPRISE_PLAN.md#51-backup--disaster-recovery)

---

## Version History
- **v1.0** (January 2026): Initial runbook creation
- **Last Tested**: Not yet tested (documentation only)
- **Next Review**: April 2026
