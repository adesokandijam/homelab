# Backup and Restore Runbook

## Purpose
This runbook provides procedures for executing backups and performing restore operations for VMs, containers, and Kubernetes resources in the homelab environment.

## Prerequisites
- Access to Proxmox web interface or SSH
- Access to backup storage
- Understanding of backup retention policies
- kubectl access for Kubernetes backups

## Backup Strategy Overview

Following the **3-2-1 Rule**:
- **3** copies of data (original + 2 backups)
- **2** different storage media types
- **1** offsite backup location

---

## Proxmox Backup Procedures

### 1. Manual VM/LXC Backup

#### 1.1 Single VM Backup
```bash
# Snapshot mode (VM stays running)
vzdump <vmid> --mode snapshot --storage <backup-storage>

# Example: Backup VM 100 to local storage
vzdump 100 --mode snapshot --storage local

# Stop mode (VM is stopped during backup)
vzdump <vmid> --mode stop --storage <backup-storage>

# Suspend mode (VM is suspended during backup)
vzdump <vmid> --mode suspend --storage <backup-storage>
```

#### 1.2 Multiple VM Backup
```bash
# Backup multiple VMs
vzdump 100 101 102 --mode snapshot --storage local

# Backup all VMs on a node
vzdump --all --mode snapshot --storage local --exclude <vmid-to-exclude>
```

#### 1.3 Backup with Compression
```bash
# Use zstd compression (recommended)
vzdump <vmid> --mode snapshot --compress zstd --storage local

# Use gzip (slower but more compatible)
vzdump <vmid> --mode snapshot --compress gzip --storage local
```

### 2. Scheduled Backup Configuration

#### 2.1 Via Web UI
1. Navigate to: **Datacenter → Backup**
2. Click **Add**
3. Configure:
   - **Schedule**: `0 2 * * *` (daily at 2 AM)
   - **Storage**: Select backup storage
   - **Mode**: snapshot
   - **Compression**: zstd
   - **Retention**: Keep last 30
4. Click **Create**

#### 2.2 Via Command Line
```bash
# Edit backup job configuration
cat >> /etc/pve/vzdump.cron << EOF
# Daily backup at 2 AM
0 2 * * * root vzdump --all --mode snapshot --compress zstd --storage local --exclude 999 --quiet 1
EOF

# Reload cron
systemctl reload cron
```

### 3. Backup Verification

#### 3.1 List Backups
```bash
# List all backups
vzdump list

# List backups for specific VM
vzdump list | grep "vmid-100"

# Via file system
ls -lh /var/lib/vz/dump/
```

#### 3.2 Verify Backup Integrity
```bash
# Check backup file integrity
zstd -t /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst

# Verify backup can be read
qmrestore --info /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst
```

### 4. Backup Cleanup

#### 4.1 Manual Cleanup
```bash
# Remove old backups (keeping last 30 days)
find /var/lib/vz/dump/ -name "vzdump-*.vma.zst" -mtime +30 -delete

# Remove backups for a specific VM
rm /var/lib/vz/dump/vzdump-qemu-100-*
```

#### 4.2 Automated Retention
```bash
# Configured in backup job (Web UI)
# Retention: Keep last 30 backups
# Or use vzdump with --prune-backups option
vzdump 100 --mode snapshot --storage local --prune-backups keep-last=30
```

---

## Proxmox Restore Procedures

### 1. Full VM Restore

#### 1.1 Restore to New VM ID
```bash
# Restore VM from backup to new ID
qmrestore /var/lib/vz/dump/vzdump-qemu-100-2026_01_09-02_00_00.vma.zst <new-vmid>

# Example: Restore VM 100 backup to VM 200
qmrestore /var/lib/vz/dump/vzdump-qemu-100-2026_01_09-02_00_00.vma.zst 200

# Specify storage location
qmrestore /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst 200 --storage local-lvm
```

#### 1.2 Restore via Web UI
1. Navigate to: **Storage** (backup storage) → **Backups**
2. Find the backup file
3. Click **Restore**
4. Configure:
   - **VM ID**: Enter new VM ID or leave for original
   - **Storage**: Select target storage
   - **Unique**: Check to regenerate MAC addresses
5. Click **Restore**

#### 1.3 Overwrite Existing VM (Caution!)
```bash
# Stop the existing VM
qm stop <vmid>

# Remove existing VM configuration
qm destroy <vmid>

# Restore from backup
qmrestore /var/lib/vz/dump/vzdump-qemu-<vmid>-*.vma.zst <vmid>

# Start the restored VM
qm start <vmid>
```

### 2. Container (LXC) Restore

#### 2.1 Restore LXC Container
```bash
# Restore container from backup
pct restore <new-ctid> /var/lib/vz/dump/vzdump-lxc-<ctid>-*.tar.zst --storage local-lvm

# Example
pct restore 200 /var/lib/vz/dump/vzdump-lxc-100-2026_01_09-02_00_00.tar.zst --storage local-lvm

# Start the container
pct start 200
```

### 3. Partial/File-Level Restore

#### 3.1 Extract Files from VM Backup
```bash
# Mount the backup archive
mkdir -p /tmp/restore
vzdump extract /var/lib/vz/dump/vzdump-qemu-100-*.vma.zst /tmp/restore

# Extract specific files
# (Manual inspection and copy)

# Cleanup
rm -rf /tmp/restore
```

---

## Kubernetes Backup Procedures (Using Velero)

### 1. Install Velero (One-time Setup)

```bash
# Download Velero
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# Install Velero in cluster with MinIO backend (S3-compatible)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket velero \
  --secret-file ./credentials-velero \
  --use-volume-snapshots=false \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio.homelab.local:9000
```

### 2. Create Kubernetes Backups

#### 2.1 Backup Entire Cluster
```bash
# Create full cluster backup
velero backup create full-backup-$(date +%Y%m%d)

# Backup specific namespace
velero backup create app-backup --include-namespaces production

# Backup with TTL (time to live)
velero backup create daily-backup --ttl 720h  # 30 days
```

#### 2.2 Scheduled Backups
```bash
# Create daily backup schedule
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h

# Create hourly backup for critical namespace
velero schedule create production-hourly \
  --schedule="0 * * * *" \
  --include-namespaces production \
  --ttl 168h  # 7 days
```

#### 2.3 Check Backup Status
```bash
# List all backups
velero backup get

# Get backup details
velero backup describe <backup-name>

# Check backup logs
velero backup logs <backup-name>
```

### 3. Restore from Kubernetes Backup

#### 3.1 Full Cluster Restore
```bash
# Restore entire cluster
velero restore create --from-backup <backup-name>

# Restore specific namespace
velero restore create --from-backup <backup-name> \
  --include-namespaces production

# Restore to different namespace
velero restore create --from-backup <backup-name> \
  --namespace-mappings old-namespace:new-namespace
```

#### 3.2 Selective Resource Restore
```bash
# Restore specific resources
velero restore create --from-backup <backup-name> \
  --include-resources deployments,services

# Exclude specific resources
velero restore create --from-backup <backup-name> \
  --exclude-resources secrets
```

#### 3.3 Check Restore Status
```bash
# List restores
velero restore get

# Get restore details
velero restore describe <restore-name>

# Check restore logs
velero restore logs <restore-name>
```

### 4. etcd Snapshot (Kubernetes Control Plane)

#### 4.1 Create etcd Snapshot
```bash
# On Kubernetes master node
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify snapshot
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-*.db
```

#### 4.2 Restore from etcd Snapshot
```bash
# Stop Kubernetes services
systemctl stop kubelet

# Restore etcd snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-*.db \
  --data-dir=/var/lib/etcd-restore

# Update etcd configuration to use restored data
# (specific to K8s distribution)

# Restart services
systemctl start kubelet
```

---

## Database Backups

### 1. PostgreSQL Backup
```bash
# Backup single database
kubectl exec -n production postgres-0 -- \
  pg_dump -U postgres mydb > /backup/mydb-$(date +%Y%m%d).sql

# Backup all databases
kubectl exec -n production postgres-0 -- \
  pg_dumpall -U postgres > /backup/postgres-all-$(date +%Y%m%d).sql
```

### 2. MongoDB Backup
```bash
# Backup MongoDB
kubectl exec -n production mongodb-0 -- \
  mongodump --out=/backup/mongodb-$(date +%Y%m%d)
```

---

## Validation

### Success Criteria
- [ ] Backup completes without errors
- [ ] Backup file exists and is readable
- [ ] Backup size is reasonable (not 0 bytes)
- [ ] Test restore succeeds
- [ ] Restored VM/application functions correctly
- [ ] Backup is stored in multiple locations (3-2-1 rule)

### Test Restore Procedure
```bash
# Monthly: Perform test restore
# 1. Select random backup from previous month
# 2. Restore to test VM ID (900-999 range)
# 3. Verify VM boots and application works
# 4. Document test results
# 5. Delete test VM
```

---

## Rollback

If restore causes issues:

1. **Stop the restored VM/container**
2. **Revert to previous known-good state**
3. **Investigate backup corruption or restore errors**
4. **Try alternate backup from different date**

---

## Troubleshooting

### Issue: Backup Fails with "No Space"
**Solution**:
```bash
df -h /var/lib/vz/dump
# Clean old backups or expand storage
```

### Issue: Restore Fails with "Invalid Archive"
**Solution**:
```bash
# Verify backup integrity
zstd -t /path/to/backup.vma.zst
# Try different backup file
```

### Issue: Velero Backup Pending
**Solution**:
```bash
# Check Velero logs
kubectl logs -n velero deployment/velero
# Check storage connection
```

---

## Related Documentation
- [Enterprise Plan - Backup Strategy](../ENTERPRISE_PLAN.md#51-backup--disaster-recovery)
- [Node Failure Recovery](node-failure-recovery.md)
- [Proxmox Backup Documentation](https://pve.proxmox.com/wiki/Backup_and_Restore)
- [Velero Documentation](https://velero.io/docs/)

---

## Version History
- **v1.0** (January 2026): Initial runbook creation
- **Last Tested**: Not yet tested (documentation only)
- **Next Review**: April 2026
