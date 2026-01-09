# Incident Response Runbook

## Purpose
This runbook provides procedures for responding to security incidents and operational emergencies in the homelab environment.

## Prerequisites
- Access to all critical systems (Proxmox, OpenWRT, Kubernetes)
- Access to monitoring and logging systems
- Understanding of security architecture and policies
- Incident response authorization

## Incident Severity Classification

| Severity | Response Time | Examples | Notification |
|----------|--------------|----------|--------------|
| **P1 - Critical** | Immediate (< 15 min) | Data breach, ransomware, complete outage | Immediate |
| **P2 - High** | < 30 minutes | Security vulnerability, partial outage | Within 1 hour |
| **P3 - Medium** | < 4 hours | Degraded performance, minor security issue | Within 4 hours |
| **P4 - Low** | < 24 hours | Informational, planned maintenance | Daily summary |

---

## Incident Response Process

### Phase 1: Detection and Triage (0-15 minutes)

#### 1.1 Initial Detection
Incidents may be detected via:
- Monitoring alerts (Prometheus/Grafana)
- Security scanning tools
- Log analysis (Loki)
- User reports
- External notifications

#### 1.2 Immediate Assessment
```bash
# Check overall system health
# From monitoring system
curl -s http://prometheus.homelab.local:9090/api/v1/query?query=up

# Check critical services status
kubectl get nodes
kubectl get pods --all-namespaces | grep -v Running

# Check Proxmox cluster
pvecm status

# Check for security alerts
grep -i "authentication failure" /var/log/auth.log
```

#### 1.3 Classify Incident
- Determine severity (P1-P4)
- Identify affected systems
- Assess business impact
- Determine if security-related

#### 1.4 Activate Response Team
- For P1/P2: Initiate immediate response
- Document incident start time
- Create incident tracking number: `INC-YYYYMMDD-XXX`

---

### Phase 2: Containment (15-60 minutes)

#### 2.1 Security Incident Containment

##### Suspected Compromise - Immediate Actions
```bash
# Isolate affected VM/container
# Option 1: Shutdown (preserves state)
qm shutdown <vmid>

# Option 2: Network isolation (keeps system running for analysis)
qm set <vmid> -net0 model=virtio,bridge=vmbr0,firewall=1,link_down=1

# For Kubernetes pod
kubectl scale deployment <deployment-name> --replicas=0 -n <namespace>

# Or delete pod
kubectl delete pod <pod-name> -n <namespace> --force
```

##### Block Malicious IP Address
```bash
# On OpenWRT firewall
# SSH to OpenWRT
ssh root@192.168.55.95

# Add IP to blocklist
iptables -I INPUT -s <malicious-ip> -j DROP
iptables -I FORWARD -s <malicious-ip> -j DROP

# Make persistent
/etc/init.d/firewall restart
```

##### Revoke Compromised Credentials
```bash
# Disable user account
# On affected system
usermod -L <username>  # Lock account
passwd -l <username>   # Lock password

# Revoke SSH keys
mv ~/.ssh/authorized_keys ~/.ssh/authorized_keys.disabled

# For Kubernetes service account
kubectl delete serviceaccount <sa-name> -n <namespace>

# Rotate compromised secrets
kubectl delete secret <secret-name> -n <namespace>
kubectl create secret generic <secret-name> --from-literal=key=new-value
```

##### Preserve Evidence
```bash
# Create forensic snapshot before any changes
# For VM
qm snapshot <vmid> forensic-$(date +%Y%m%d-%H%M%S)

# For container
docker commit <container-id> forensic-image:$(date +%Y%m%d-%H%M%S)

# Capture network traffic
tcpdump -i any -w /tmp/incident-capture-$(date +%Y%m%d-%H%M%S).pcap

# Collect logs
journalctl --since "1 hour ago" > /tmp/incident-logs-$(date +%Y%m%d-%H%M%S).log
```

#### 2.2 Operational Incident Containment

##### Service Outage
```bash
# Check dependent services
systemctl status <service-name>

# Review recent changes
git log --since="24 hours ago" --oneline

# Rollback recent deployment (if applicable)
kubectl rollout undo deployment/<deployment-name> -n <namespace>

# Scale up replacement instances
kubectl scale deployment <deployment-name> --replicas=5 -n <namespace>
```

##### Resource Exhaustion
```bash
# Identify resource hog
# CPU
top -b -n 1 | head -20

# Memory
ps aux --sort=-%mem | head -20

# Disk
du -sh /* | sort -hr | head -10

# Kill problematic process (if safe)
kill -9 <pid>

# Or reduce replicas
kubectl scale deployment <deployment-name> --replicas=1 -n <namespace>
```

---

### Phase 3: Investigation (Parallel with Containment)

#### 3.1 Security Incident Investigation

##### Identify Attack Vector
```bash
# Check authentication logs
grep "Failed password" /var/log/auth.log | tail -50
grep "Accepted publickey" /var/log/auth.log | tail -50

# Check web server access logs
tail -100 /var/log/nginx/access.log | grep -E "POST|GET"

# Check for suspicious processes
ps aux | grep -v "\[" | sort

# Check active connections
netstat -tupan | grep ESTABLISHED
ss -tunap
```

##### Check for Malware/Backdoors
```bash
# Check for suspicious files
find /tmp -type f -mtime -1
find /var/tmp -type f -mtime -1
find /home -name "*.sh" -mtime -1

# Check for unauthorized users
cat /etc/passwd | grep -v nologin

# Check cron jobs
cat /etc/crontab
ls -la /etc/cron.*
crontab -l

# Check startup scripts
ls -la /etc/rc*.d/
systemctl list-unit-files | grep enabled
```

##### Analyze Logs
```bash
# Centralized log query (Loki)
logcli query '{namespace="production"}' --since=1h

# Check Kubernetes audit logs
kubectl logs -n kube-system kube-apiserver-* | grep -i audit

# Check system logs
journalctl -xe --since "2 hours ago"
dmesg | tail -100
```

#### 3.2 Operational Incident Investigation

##### Check Monitoring Dashboards
- Access Grafana: `http://grafana.homelab.local`
- Review recent alerts
- Check resource usage trends
- Identify anomalies

##### Review Change History
```bash
# Recent Git commits
git log --all --since="24 hours ago" --pretty=format:"%h - %an, %ar : %s"

# Kubernetes events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Check Proxmox task log
# Via Web UI: Datacenter → Tasks
```

##### Application Logs
```bash
# Check application logs
kubectl logs <pod-name> -n <namespace> --tail=100

# Previous container logs (if crashed)
kubectl logs <pod-name> -n <namespace> --previous

# All pods in deployment
kubectl logs -l app=<label> -n <namespace> --tail=100
```

---

### Phase 4: Eradication (Post-containment)

#### 4.1 Remove Threat

##### Malware Removal
```bash
# Restore from clean backup
qmrestore /var/lib/vz/dump/vzdump-qemu-<vmid>-*.vma.zst <new-vmid>

# Or patch running system
# Update packages
apt update && apt upgrade -y

# Remove malicious files
rm -f /path/to/malicious/file

# Kill malicious processes
pkill -9 -f malicious-process
```

##### Patch Vulnerabilities
```bash
# Update all systems
# On Debian/Ubuntu
apt update && apt upgrade -y

# On Proxmox
apt update && apt dist-upgrade -y

# Update Kubernetes components
kubectl get nodes -o wide  # Check versions
# Follow upgrade procedure for your K8s distribution

# Update container images
kubectl set image deployment/<name> container=image:new-tag -n <namespace>
```

##### Harden Configuration
```bash
# Disable unnecessary services
systemctl disable <service-name>
systemctl stop <service-name>

# Update firewall rules
iptables -A INPUT -p tcp --dport <vulnerable-port> -j DROP

# Enforce security policies
# Apply Pod Security Standards
kubectl label namespace <namespace> pod-security.kubernetes.io/enforce=restricted
```

#### 4.2 Verify Eradication
```bash
# Scan for remaining vulnerabilities
# Run security scanner
trivy image <image-name>

# Check for persistence mechanisms
systemctl list-unit-files | grep enabled
crontab -l

# Verify no backdoor accounts
cat /etc/passwd
cat /etc/shadow

# Check network connections
netstat -tupan | grep LISTEN
```

---

### Phase 5: Recovery (After eradication verified)

#### 5.1 Restore Services

##### Bring Systems Back Online
```bash
# Start VM
qm start <vmid>

# Start service
systemctl start <service-name>

# Scale up deployment
kubectl scale deployment <deployment-name> --replicas=3 -n <namespace>
```

##### Validate Functionality
```bash
# Check service health
curl -I http://service.homelab.local/health

# Check pod status
kubectl get pods -n <namespace>

# Run smoke tests
kubectl run test-pod --image=curlimages/curl --rm -it --restart=Never -- \
  curl http://service.homelab.local/api/status
```

##### Monitor for Issues
- Watch dashboards for 30+ minutes
- Check error rates
- Verify no degradation
- Monitor resource usage

#### 5.2 Restore Access
```bash
# Re-enable user accounts (if disabled)
usermod -U <username>

# Restore SSH access with new keys
cat new-public-key >> ~/.ssh/authorized_keys

# Recreate service accounts
kubectl create serviceaccount <sa-name> -n <namespace>
```

---

### Phase 6: Post-Incident Activities

#### 6.1 Document Incident

Create incident report including:
- **Incident ID**: INC-YYYYMMDD-XXX
- **Timeline**: Detailed chronology of events
- **Impact**: Affected systems and users
- **Root Cause**: What caused the incident
- **Actions Taken**: Step-by-step response
- **Resolution**: How it was fixed
- **Lessons Learned**: What can be improved

#### 6.2 Root Cause Analysis

Use the "5 Whys" technique:
1. Why did the incident occur?
2. Why did that happen?
3. Why wasn't it prevented?
4. Why wasn't it detected earlier?
5. Why wasn't the impact limited?

#### 6.3 Implement Improvements

##### Update Security Controls
```bash
# Add monitoring alerts
# Create Prometheus alert rule
cat > /etc/prometheus/rules/security-alerts.yml <<EOF
groups:
- name: security
  rules:
  - alert: HighFailedLoginRate
    expr: rate(ssh_failed_login_total[5m]) > 5
    for: 5m
    annotations:
      summary: "High rate of failed SSH logins"
EOF
```

##### Update Runbooks
- Document new procedures discovered
- Update existing runbooks with improvements
- Add preventive measures

##### Update Firewall Rules
```bash
# Add new firewall rules based on incident
# On OpenWRT
uci add firewall rule
uci set firewall.@rule[-1].name='Block known bad actor'
uci set firewall.@rule[-1].src='wan'
uci set firewall.@rule[-1].src_ip='<malicious-ip>'
uci set firewall.@rule[-1].target='DROP'
uci commit firewall
/etc/init.d/firewall restart
```

##### Implement Additional Monitoring
```bash
# Add log monitoring rule
# Create alert for specific log pattern
# In Loki
{job="syslog"} |~ "suspicious pattern" | rate > 10
```

#### 6.4 Conduct Post-Mortem Meeting

Schedule meeting within 48 hours to:
- Review timeline
- Discuss what went well
- Identify improvement areas
- Assign action items
- Update documentation

---

## Common Incident Types

### 1. Ransomware Attack

**Immediate Actions**:
1. Disconnect affected systems from network
2. DO NOT pay ransom
3. Restore from clean backups
4. Identify entry point and patch

### 2. Data Breach

**Immediate Actions**:
1. Identify scope of data exposed
2. Preserve evidence
3. Contain the breach
4. Notify affected parties (if required)
5. Review access logs

### 3. DDoS Attack

**Immediate Actions**:
1. Identify attack source
2. Rate limit traffic
3. Block malicious IPs
4. Enable CloudFlare (if available)
5. Scale up resources temporarily

### 4. Unauthorized Access

**Immediate Actions**:
1. Lock compromised accounts
2. Force password reset
3. Review access logs
4. Check for lateral movement
5. Rotate all credentials

### 5. Service Outage

**Immediate Actions**:
1. Check infrastructure status
2. Review recent changes
3. Rollback if needed
4. Scale resources
5. Implement workaround

---

## Communication Templates

### Initial Incident Notification
```
INCIDENT ALERT - [P1/P2/P3/P4]

Incident ID: INC-YYYYMMDD-XXX
Time Detected: HH:MM UTC
Severity: [P1/P2/P3/P4]
Status: [Investigating/Contained/Resolved]

Summary: [Brief description]

Impact: [What is affected]

Actions: [What is being done]

Next Update: [Expected time]
```

### Incident Update
```
INCIDENT UPDATE - INC-YYYYMMDD-XXX

Status: [Current status]
Time: HH:MM UTC

Update: [What has changed]

Current Actions: [What is being done now]

Next Steps: [What's planned]

Next Update: [Expected time]
```

### Incident Resolution
```
INCIDENT RESOLVED - INC-YYYYMMDD-XXX

Resolution Time: HH:MM UTC
Duration: [Total time]

Root Cause: [Brief explanation]

Resolution: [How it was fixed]

Preventive Actions: [What will prevent recurrence]

Post-Mortem: [Link to detailed report]
```

---

## Validation

### Incident Response Checklist
- [ ] Incident detected and classified
- [ ] Response team notified
- [ ] Affected systems identified
- [ ] Containment measures implemented
- [ ] Evidence preserved
- [ ] Threat eradicated
- [ ] Systems recovered
- [ ] Functionality validated
- [ ] Incident documented
- [ ] Root cause identified
- [ ] Improvements implemented
- [ ] Post-mortem completed

---

## Related Documentation
- [Enterprise Plan - Security Framework](../ENTERPRISE_PLAN.md#4-security-framework)
- [Network Troubleshooting](network-troubleshooting.md)
- [Node Failure Recovery](node-failure-recovery.md)
- [Backup and Restore](backup-restore.md)

---

## Version History
- **v1.0** (January 2026): Initial runbook creation
- **Last Tested**: Not yet tested (documentation only)
- **Next Review**: April 2026

---

## Emergency Contacts

| Role | Contact Method | Use Case |
|------|----------------|----------|
| Infrastructure Owner | Primary | All incidents |
| ISP Support | Phone: XXX-XXX-XXXX | Internet connectivity issues |
| Hardware Vendor | Email: support@vendor.com | Hardware failures |

**Note**: Update contact information as needed.
