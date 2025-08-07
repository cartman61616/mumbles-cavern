# Dell Nodes 3 & 4 Integration Plan
**Snooze (Node 3) & Nappy (Node 4) - Cluster Expansion**

**Target Timeline**: Week 2-3 post-Plex deployment  
**Current Status**: 2-node cluster operational (drowzee + sleepy)  
**Goal**: Expand to 4-node cluster for full homelab capacity

## Overview

Expand our current 2-node Proxmox cluster (drowzee + sleepy) to a 4-node cluster by adding:
- **Node 3 (snooze.mumblescavern.local)**: Productivity services hub
- **Node 4 (nappy.mumblescavern.local)**: Development environment

## Hardware Specifications

### Node 3: Snooze (Dell OptiPlex 7050 Micro)
```yaml
Hardware:
  Model: Dell OptiPlex 7050 Micro  
  CPU: Intel i5-7500T (4 cores, 2.7-3.3GHz)
  RAM: 32GB DDR4
  Storage: NVMe SSD (needs upgrade/installation)
  GPU: Intel HD Graphics 630
  Network: Gigabit Ethernet

Network Configuration:
  Hostname: snooze.mumblescavern.local
  Management IP: 192.168.10.12
  VLAN: Management (10)
  Switch Port: flex-mini port 3

Planned Services:
  - Nextcloud (file sync and collaboration)
  - Vaultwarden (password management)  
  - Paperless-ngx (document management)
  - Navidrome (music streaming)
  - Development databases (PostgreSQL, Redis instances)
```

### Node 4: Nappy (Dell OptiPlex 6500T)
```yaml
Hardware:
  Model: Dell OptiPlex 6500T
  CPU: Intel i5-6500 (4 cores, 3.2-3.6GHz, 6th gen)
  RAM: 32GB DDR4
  Storage: SSD + development storage
  GPU: Intel HD Graphics 530
  Network: Gigabit Ethernet

Network Configuration:
  Hostname: nappy.mumblescavern.local
  Management IP: 192.168.10.13
  VLAN: Management (10)
  Switch Port: flex-mini port 4

Planned Services:
  - GitLab CE (code repositories and CI/CD)
  - Harbor (container registry)
  - Code-Server (VS Code in browser)
  - K3s (Kubernetes learning environment)
  - Development and testing workloads
```

## Pre-Integration Prerequisites

### Hardware Preparation
```bash
# Node 3 (Snooze) Requirements:
- [ ] NVMe SSD installation (if not present)
- [ ] RAM upgrade to 32GB (if needed)
- [ ] BIOS optimization (following existing guide)
- [ ] Network cable connection to flex-mini port 3

# Node 4 (Nappy) Requirements:  
- [ ] SSD installation/upgrade
- [ ] RAM upgrade to 32GB (if needed)
- [ ] BIOS optimization (6th gen specific settings)
- [ ] Network cable connection to flex-mini port 4
```

### Network Infrastructure Updates
```bash
# UniFi Configuration Updates:
- [ ] DHCP reservations for both nodes
- [ ] DNS entries in UniFi DNS
- [ ] Switch port configuration (if needed)
- [ ] Monitoring setup for new nodes

# IP Reservations:
192.168.10.12 - snooze.mumblescavern.local
192.168.10.13 - nappy.mumblescavern.local
```

## Integration Timeline

### Week 1: Node 3 (Snooze) Integration

#### Day 1: Hardware Preparation
```bash
# Physical setup
- Power off node, install NVMe if needed
- Connect to flex-mini port 3
- Power on and verify BIOS settings
- Update BIOS if needed (following existing guide)

# Network connectivity test
ping 192.168.10.12  # Should fail initially
```

#### Day 2: Proxmox Installation
```bash
# Install Proxmox VE on Node 3
# Download latest Proxmox ISO to USB
# Follow standard installation:
- Hostname: snooze.mumblescavern.local  
- IP: 192.168.10.12/24
- Gateway: 192.168.10.1
- DNS: 192.168.10.1

# Post-installation configuration
ssh root@192.168.10.12

# Update system
apt update && apt upgrade -y

# Configure repositories (match existing cluster)
# Add no-subscription repo if needed
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update
```

#### Day 3: Cluster Integration
```bash
# From existing cluster node (drowzee or sleepy):
ssh root@192.168.10.10

# Generate join token
pvecm add 192.168.10.12

# On new node (snooze):
ssh root@192.168.10.12

# Join cluster (get exact command from previous step)
pvecm add 192.168.10.10

# Verify cluster status
pvecm status
pvecm nodes

# Should show 3 nodes now
```

### Week 2: Node 4 (Nappy) Integration

#### Day 1-2: Hardware & Installation
```bash
# Repeat hardware preparation process for Node 4
# Note: 6th gen CPU may have slightly different BIOS options

# Proxmox installation:
- Hostname: nappy.mumblescavern.local
- IP: 192.168.10.13/24
- Gateway: 192.168.10.1
- DNS: 192.168.10.1
```

#### Day 3: Cluster Integration
```bash
# Add Node 4 to cluster
pvecm add 192.168.10.13

# Final cluster verification
pvecm status
# Should show 4 nodes: drowzee, sleepy, snooze, nappy
```

## Storage Configuration

### Shared Storage Access
```bash
# On both new nodes, configure access to existing storage:

# ASUSTOR NAS integration
# Add NFS storage to both nodes via Proxmox web UI:
- Storage ID: asustor-vm-storage
- Server: 192.168.10.20
- Export: /mnt/pool/proxmox-vms
- Content: VZ images, ISO images, VZ snapshots, VZ templates

# Verify storage accessibility
pvesm status
```

### Local Storage Optimization
```bash
# Node 3 (Snooze) - Productivity workloads
# Optimize for document storage and database performance

# Node 4 (Nappy) - Development workloads  
# Optimize for code compilation and container builds
# Consider enabling caching for Docker builds
```

## Service Distribution Strategy

### Current 2-Node Distribution
```yaml
Node 1 (Drowzee):
  - Plex Media Server (hardware transcoding)
  - Tautulli (Plex monitoring)
  - Overseerr (media requests)

Node 2 (Sleepy):
  - *arr Stack (Sonarr, Radarr, Prowlarr)
  - qBittorrent (download client)
  - Bazarr (subtitle management)
```

### 4-Node Target Distribution
```yaml
Node 1 (Drowzee) - Media Hub:
  - Plex Media Server (hardware transcoding)
  - Tautulli (Plex monitoring)
  - Overseerr (media requests)
  - Media-related automation

Node 2 (Sleepy) - Media Automation:
  - *arr Stack (Sonarr, Radarr, Prowlarr)
  - qBittorrent (download client)
  - Bazarr (subtitle management)
  - Download automation

Node 3 (Snooze) - Productivity Services:
  - Nextcloud (file sync and collaboration)
  - Vaultwarden (password management)
  - Paperless-ngx (document management)
  - Navidrome (music streaming)
  - Productivity PostgreSQL instance

Node 4 (Nappy) - Development Environment:
  - GitLab CE (code repositories and CI/CD)
  - Harbor (container registry)
  - Code-Server (VS Code in browser)
  - K3s (Kubernetes learning environment)
  - Development databases and testing
```

## High Availability Considerations

### Cluster Quorum
```bash
# With 4 nodes, cluster can survive 1 node failure
# Quorum: 3 out of 4 nodes required

# Configure proper corosync settings
# Verify voting configuration:
corosync-cmapctl | grep members
```

### VM Migration Strategy
```bash
# Configure VM migration between nodes
# Ensure shared storage access from all nodes
# Test live migration between nodes

# Example migration test:
qm migrate 101 snooze   # Migrate Plex VM to new node if needed
```

### Service HA Configuration
```bash
# Consider HA configuration for critical services:
# - PostgreSQL with replication
# - Shared file storage
# - Load balancing for web services
```

## Monitoring and Management

### Node Monitoring
```bash
# Add new nodes to monitoring stack
# Update Prometheus configuration
# Add Grafana dashboards for new nodes
# Configure alerts for all 4 nodes
```

### Resource Allocation
```yaml
Total Cluster Resources (4 nodes):
  CPU Cores: 16 physical (32 threads with HT)
  RAM: 128GB total
  Storage: Shared NFS + local fast storage
  
Recommended VM Allocation:
  - Reserve 20% resources for Proxmox overhead
  - Distribute workloads based on service requirements
  - Plan for growth and development needs
```

## Testing and Validation

### Cluster Health Tests
```bash
# After both nodes integrated:

# Test cluster communication
pvecm status

# Test shared storage from all nodes
pvesm status

# Test VM creation on new nodes
qm create 200 --name test-vm --memory 1024 --cores 1 --storage asustor-vm-storage

# Test VM migration between all nodes
qm migrate 200 sleepy
qm migrate 200 snooze  
qm migrate 200 nappy
qm migrate 200 drowzee

# Clean up test VM
qm stop 200 && qm destroy 200
```

### Service Deployment Testing
```bash
# Deploy test services on new nodes
# Verify network connectivity between services
# Test backup and restore procedures
# Validate monitoring and alerting
```

## Benefits of 4-Node Cluster

### Capacity Benefits
- **4x compute capacity** for expanded services
- **Improved resource utilization** with workload distribution
- **Development environment** separation from production

### Reliability Benefits  
- **N-1 redundancy** (cluster survives single node failure)
- **Rolling updates** possible without service interruption
- **Load distribution** reduces single points of failure

### Organizational Benefits
- **Service specialization** (media, productivity, development)
- **Clear resource boundaries** for different workload types
- **Scalability foundation** for future growth

This plan provides a structured approach to expanding your cluster while maintaining service availability and following established naming/network conventions!