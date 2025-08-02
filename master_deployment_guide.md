# Mumbles Cavern Dell Cluster - Master Deployment Guide

## Overview
This guide walks you through deploying a complete Proxmox homelab cluster using Dell OptiPlex nodes. The deployment is broken into manageable phases, each with its own detailed guide.

## Hardware Inventory
- **Node 1-3**: Dell OptiPlex 7050 Micro (i5-7500T, 32GB RAM)
- **Node 4**: Dell OptiPlex 6500T (6th gen i5, 32GB RAM)
- **Power**: Anker Prime USB-C charger with barrel jack adapters
- **Networking**: UniFi Dream Machine Pro + UniFi Flex Mini switch
- **Storage**: ASUSTOR TrueNAS (shared) + local NVMe/SSD per node

## Prerequisites
- [ ] All Dell nodes physically connected and powered
- [ ] UniFi network equipment configured and accessible
- [ ] Password manager setup for credential storage
- [ ] Basic Linux command line familiarity

## Deployment Phases

### Pre-Phase 0: Hardware Preparation
**File**: `bios_optimization.md`
**Duration**: 1-2 days  
**Purpose**: Update BIOS, resolve CPU frequency issues, optimize hardware settings
- BIOS updates for all Dell nodes
- CPU frequency optimization (fix 800MHz lock)
- Thermal and power management configuration
- Hardware validation and documentation

### Phase 0.5: Network Infrastructure
**File**: `network_setup.md`
**Duration**: 4-6 hours  
**Purpose**: Configure VLANs, security, and gaming optimization
- VLAN architecture design and implementation
- Gaming network optimization with QoS
- Device assignment and DHCP reservations
- Firewall rules and inter-VLAN routing
- DNS configuration

### Phase 1: Primary Node Setup
**File**: `primary_node_setup.md`
**Duration**: 2-3 hours  
**Purpose**: Install and configure first Proxmox node
- Hardware preparation and BIOS final config
- Proxmox VE installation on Node 1
- Initial system configuration and updates
- Storage pool creation (if dual storage available)
- Web UI access and validation

### Phase 2: Cluster Formation
**File**: `cluster_setup.md`
**Duration**: 2-3 hours  
**Purpose**: Add second node and create cluster
- Node 2 Proxmox installation
- Cluster creation and node joining
- Cluster validation and health checks
- Shared storage configuration planning

### Phase 3: Storage Configuration
**File**: `storage_setup.md`
**Duration**: 3-4 hours  
**Purpose**: Configure shared and local storage systems
- ASUSTOR TrueNAS NFS configuration
- Proxmox storage integration
- Local storage optimization
- Backup storage planning
- Storage performance testing

### Phase 4: Shared Infrastructure
**File**: `shared_infrastructure.md`
**Duration**: 2-3 hours  
**Purpose**: Deploy foundational services
- Shared PostgreSQL and Redis deployment
- Network configuration for shared services
- Service health monitoring setup
- Database initialization and security

### Phase 5: Essential Services
**File**: `essential_services.md`
**Duration**: 4-6 hours  
**Purpose**: Deploy core homelab applications
- VM template creation
- Plex media server deployment
- *arr stack deployment (Sonarr, Radarr, Prowlarr)
- Download client configuration (qBittorrent)
- Service integration testing

### Phase 6: Monitoring & Management
**File**: `07-monitoring.md`  
**Duration**: 3-4 hours  
**Purpose**: Deploy monitoring and observability
- Prometheus deployment and configuration
- Grafana setup with shared PostgreSQL
- Node exporter installation on all nodes
- Dashboard creation and alerting
- Performance baseline establishment

### Phase 7: External Access & Security
**File**: `08-external-access.md`  
**Duration**: 3-4 hours  
**Purpose**: Configure secure remote access
- Traefik reverse proxy deployment
- Cloudflare Tunnel configuration
- SSL certificate automation
- Service routing and load balancing
- Security hardening

### Phase 8: Gaming Infrastructure
**File**: `09-gaming-setup.md`  
**Duration**: 2-3 hours  
**Purpose**: Deploy gaming-specific services
- Gaming VLAN optimization
- Game streaming server (Sunshine/Moonlight)
- Retro gaming server (RetroPie/Batocera)
- Gaming device integration
- Performance optimization for low latency

## Quick Start Checklist

### Before You Begin
- [ ] Read through all phase documents
- [ ] Prepare 4 USB drives with Proxmox ISO
- [ ] Document all MAC addresses for DHCP reservations
- [ ] Set up password manager entries
- [ ] Plan IP address assignments
- [ ] Verify internet connectivity and Cloudflare account

### Phase Completion Validation
Each phase includes validation steps. Mark completion:
- [ ] Pre-Phase 0: BIOS Updated, CPU frequencies normal
- [ ] Phase 0.5: VLANs configured, networking tested
- [ ] Phase 1: Node 1 accessible via web UI
- [ ] Phase 2: Cluster formed, both nodes visible
- [ ] Phase 3: Storage accessible, NFS mounted
- [ ] Phase 4: Shared services healthy and accessible
- [ ] Phase 5: Plex and *arr stack functional
- [ ] Phase 6: Monitoring dashboards operational
- [ ] Phase 7: External access working, SSL certificates valid
- [ ] Phase 8: Gaming services deployed and optimized

## Emergency Procedures

### If Something Goes Wrong
1. **Single node failure**: Remaining cluster nodes continue operation
2. **Network connectivity issues**: Check VLAN configuration in `network_setup.md`
3. **Storage problems**: Refer to troubleshooting in `storage_setup.md`
4. **Service failures**: Check shared infrastructure health in `shared_infrastructure.md`

### Rollback Procedures
- Each phase includes rollback instructions
- VM snapshots taken before major changes
- Configuration backups stored in password manager
- Network settings documented for quick restoration

## Support Resources
- **Proxmox Documentation**: https://pve.proxmox.com/pve-docs/
- **UniFi Documentation**: https://help.ui.com/
- **Cloudflare Tunnel Guide**: https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/
- **Hardware Specific**: Dell support documentation per node service tag

## Timeline Estimate
- **Weekend 1**: Pre-Phase 0 through Phase 2 (Basic cluster)
- **Weekend 2**: Phase 3 through Phase 5 (Core services)
- **Weekend 3**: Phase 6 through Phase 8 (Monitoring and gaming)
- **Total**: 20-30 hours spread across 3 weekends

Each phase is designed to leave you with a functional system, so you can take breaks between phases without losing progress.

---

**Next Step**: Begin with `bios_optimization.md` to prepare your hardware for optimal performance.
