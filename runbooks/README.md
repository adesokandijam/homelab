# Operational Runbooks

This directory contains operational procedures and runbooks for managing the homelab infrastructure.

## Available Runbooks

### Infrastructure Management
- [Node Failure Recovery](node-failure-recovery.md) - Procedure for handling Proxmox node failures
- [Backup and Restore](backup-restore.md) - Backup execution and restoration procedures
- [Network Troubleshooting](network-troubleshooting.md) - Common network issues and resolutions

### Kubernetes Operations
- [Pod Troubleshooting](k8s-pod-troubleshooting.md) - Debugging failing pods and containers
- [Certificate Renewal](certificate-renewal.md) - SSL/TLS certificate management
- [Scaling Operations](scaling-operations.md) - Scaling applications and cluster nodes

### Security Operations
- [Incident Response](incident-response.md) - Security incident handling procedures
- [Vulnerability Management](vulnerability-management.md) - Patch management and vulnerability remediation
- [Access Review](access-review.md) - Periodic access control review process

## Runbook Standards

Each runbook should include:

1. **Purpose**: What this runbook accomplishes
2. **Prerequisites**: Required access, tools, or knowledge
3. **Procedure**: Step-by-step instructions
4. **Validation**: How to verify success
5. **Rollback**: How to undo changes if needed
6. **Related Documentation**: Links to relevant docs

## Usage

1. Identify the issue or task
2. Find the relevant runbook
3. Follow the procedure step-by-step
4. Document any deviations or issues
5. Update the runbook if improvements are found

## Contributing

When creating new runbooks:
- Use clear, concise language
- Include example commands with expected output
- Test procedures before documenting
- Keep runbooks up-to-date with infrastructure changes
