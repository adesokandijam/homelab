# Homelab - Enterprise Data Center

This repository contains documentation and configuration for an enterprise-grade homelab infrastructure, designed and operated with data center standards.

## 📚 Documentation

- **[Enterprise Plan](ENTERPRISE_PLAN.md)** - Comprehensive planning document with enterprise data center architecture
- **[Network Architecture](Networking.md)** - Detailed network design with VLAN topology and firewall rules
- **[Hardware Inventory](inventory.md)** - Complete hardware specifications and resource allocation

## 🎯 Overview

This homelab operates as a production-grade platform featuring:

- **Kubernetes Orchestration**: Multi-node K8s cluster for container workloads
- **High Availability**: 5-node Proxmox VE cluster with HA configuration
- **AI/ML Capabilities**: GPU-accelerated computing with NVIDIA RTX 3060 and Tesla P4
- **Enterprise Networking**: VLAN segmentation, zero-trust security, managed switching
- **Automation**: Infrastructure as Code using Terraform, Ansible, and GitOps
- **Observability**: Comprehensive monitoring with Prometheus, Grafana, and Loki

## 🏗️ Infrastructure

### Compute
- 5× Proxmox VE nodes (50+ CPU cores, 112GB RAM)
- 1× Windows AI workstation (48GB RAM, dual GPUs)
- Total capacity: 164GB RAM, 3.9TB+ storage

### Network
- OpenWRT gateway (Raspberry Pi 5 with 2.5GbE)
- TP-Link managed switch (802.1Q VLAN support)
- Multi-VLAN architecture (Management, Ops, K8s, DMZ)

### Services
- Kubernetes cluster (K3s/RKE2)
- Monitoring stack (Prometheus, Grafana, Loki)
- CI/CD pipeline (GitLab/ArgoCD)
- Container registry (Harbor)
- Identity management (FreeIPA/Keycloak)

## 🚀 Quick Links

- [Implementation Roadmap](ENTERPRISE_PLAN.md#8-implementation-roadmap)
- [Security Framework](ENTERPRISE_PLAN.md#4-security-framework)
- [Disaster Recovery Plan](ENTERPRISE_PLAN.md#51-backup--disaster-recovery)
- [Monitoring Strategy](ENTERPRISE_PLAN.md#52-monitoring--alerting-strategy)

## 📁 Repository Structure

```
homelab/
├── ENTERPRISE_PLAN.md    # Comprehensive enterprise planning document
├── Networking.md         # Network architecture and VLAN design
├── inventory.md          # Hardware inventory and specifications
├── openwrt/             # OpenWRT configuration files
│   ├── network          # Network interface configuration
│   ├── firewall         # Firewall rules and zones
│   └── dhcp             # DHCP and DNS settings
├── pihole/              # Pi-hole DNS configuration
└── images/              # Network diagrams and architecture images
```

## 🎓 Operating Principles

This homelab follows enterprise data center best practices:

1. **Infrastructure as Code**: All configurations versioned and automated
2. **Zero-Trust Security**: Network segmentation and least privilege access
3. **High Availability**: Redundancy and automated failover
4. **Observability**: Comprehensive monitoring, logging, and alerting
5. **Disaster Recovery**: Regular backups and tested recovery procedures
6. **Change Management**: Documented and controlled change processes
7. **Continuous Improvement**: Regular reviews and optimization

## 📊 Current Status

**Phase**: Foundation Complete, Core Infrastructure In Progress

See the [Enterprise Plan](ENTERPRISE_PLAN.md) for detailed implementation status and roadmap.

---

**Maintained by**: Homelab Infrastructure Team  
**Last Updated**: January 2026
