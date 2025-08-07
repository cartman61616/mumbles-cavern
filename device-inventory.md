# Mumbles Cavern Device Inventory & Tracking

**Last Updated**: 2025-01-07  
**Reference**: [Naming Conventions](naming-conventions.md) | [ADR-002](adrs/002-naming-convention-standard.md)

## Overview

Comprehensive tracking and monitoring document for all devices in the Mumbles Cavern homelab ecosystem. This document serves as the single source of truth for device status, configurations, and monitoring requirements.

## Infrastructure Status Dashboard

### ✅ Operational Devices
| Status | Device | Role | Issues |
|--------|--------|------|--------|
| 🟢 **ACTIVE** | Drowzee (Node 1) | Media Hub | None |
| 🟢 **ACTIVE** | Sleepy (Node 2) | Media Automation | None |
| 🔵 **PLANNED** | Snooze (Node 3) | Productivity Services | Awaiting deployment |
| 🔵 **PLANNED** | Nappy (Node 4) | Development Environment | Awaiting deployment |
| 🟢 **ACTIVE** | ASUSTOR NAS | Primary Storage | TrueNAS reset pending |
| 🟡 **TRANSITIONING** | Snorlax Prime | Current Homelab Server | Migration to gaming/dev role |
| 🟢 **ACTIVE** | Beast Rig | Primary Gaming PC | None |
| 🔵 **INTEGRATING** | MacBook Pro | Tri-Role Workstation | Awaiting integration |

## Core Infrastructure

### Dell OptiPlex Proxmox Cluster

#### Node 1: Drowzee (Media Hub)
```yaml
Hardware:
  Model: Dell OptiPlex 7050 Micro
  CPU: Intel i5-7500T (4 cores, 2.7-3.3GHz)
  RAM: 32GB DDR4
  Storage: NVMe SSD (local) + NFS (shared)
  GPU: Intel HD Graphics 630 (QuickSync capable)
  
Network:
  Hostname: drowzee.mumblescavern.local
  Management IP: 192.168.10.10
  MAC Address: [To be documented during setup]
  VLAN: Management (10)
  
Status:
  Deployment: ✅ Operational (2-node cluster)
  Services: Awaiting Plex Media Server deployment
  Health: Healthy
  Uptime: [Monitor post-deployment]
  
Planned Services:
  - Plex Media Server (hardware transcoding)
  - Tautulli (Plex monitoring)
  - Overseerr (media requests)
  
Monitoring:
  - Node Exporter: To be configured
  - Proxmox metrics: Available
  - Service health checks: Pending deployment
```

#### Node 2: Sleepy (Media Automation)
```yaml
Hardware:
  Model: Dell OptiPlex 7050 Micro
  CPU: Intel i5-7500T (4 cores, 2.7-3.3GHz)
  RAM: 32GB DDR4
  Storage: NVMe SSD (local) + NFS (shared)
  GPU: Intel HD Graphics 630
  
Network:
  Hostname: sleepy.mumblescavern.local
  Management IP: 192.168.10.11
  MAC Address: [To be documented during setup]
  VLAN: Management (10)
  
Status:
  Deployment: ✅ Operational (2-node cluster)
  Services: Awaiting *arr stack deployment
  Health: Healthy
  Uptime: [Monitor post-deployment]
  
Planned Services:
  - Sonarr (TV series management)
  - Radarr (Movie management)
  - Prowlarr (Indexer management)
  - qBittorrent (Download client)
  - Bazarr (Subtitle management)
  
Monitoring:
  - Node Exporter: To be configured
  - Download monitoring: Pending
  - Disk space alerts: Critical for downloads
```

#### Node 3: Snooze (Productivity Services)
```yaml
Hardware:
  Model: Dell OptiPlex 7050 Micro
  CPU: Intel i5-7500T (4 cores, 2.7-3.3GHz)
  RAM: 32GB DDR4
  Storage: NVMe SSD (needs upgrade)
  GPU: Intel HD Graphics 630
  
Network:
  Hostname: snooze.mumblescavern.local
  Management IP: 192.168.10.12
  MAC Address: [To be documented during setup]
  VLAN: Management (10)
  
Status:
  Deployment: 🔵 Planned for Week 3
  Services: Not yet configured
  Health: Not applicable
  Notes: Requires NVMe storage upgrade
  
Planned Services:
  - Nextcloud (file sync and collaboration)
  - Vaultwarden (password management)
  - Paperless-ngx (document management)
  - Navidrome (music streaming)
  
Prerequisites:
  - NVMe storage installation
  - Proxmox installation
  - Cluster joining procedure
```

#### Node 4: Nappy (Development Environment)
```yaml
Hardware:
  Model: Dell OptiPlex 6500T
  CPU: Intel i5-6500 (4 cores, 3.2-3.6GHz, 6th gen)
  RAM: 32GB DDR4
  Storage: SSD + development storage
  GPU: Intel HD Graphics 530
  
Network:
  Hostname: nappy.mumblescavern.local
  Management IP: 192.168.10.13
  MAC Address: [To be documented during setup]
  VLAN: Management (10)
  
Status:
  Deployment: 🔵 Planned for Week 3
  Services: Development environment
  Health: Not applicable
  
Planned Services:
  - GitLab CE (code repositories and CI/CD)
  - Harbor (container registry)
  - Code-Server (VS Code in browser)
  - K3s (Kubernetes learning environment)
  
Special Notes:
  - Older CPU (6th gen vs 7th gen on other nodes)
  - Development and testing workloads
  - K3s cluster deployment target
```

## Storage Infrastructure

### ASUSTOR NAS (Primary Storage)
```yaml
Hardware:
  Model: ASUSTOR [Model TBD]
  Capacity: 24TB usable storage
  Interface: Gigabit Ethernet
  
Network:
  Hostname: asustor-nas.mumblescavern.local
  IP: 192.168.10.20
  VLAN: Management (10)
  
Status:
  Current: 🟡 TrueNAS reset pending
  Services: Currently hosting existing media library
  Health: Functional, needs reorganization
  
Configuration:
  OS: TrueNAS
  Shares: 
    - /mnt/pool/media (NFS export for Plex)
    - /mnt/pool/downloads (NFS export for *arr stack)
    - /mnt/pool/backups (Configuration and VM backups)
    - /mnt/pool/documents (MacBook Pro NFS mount)
  
Critical Action:
  - Data migration to temporary storage before reset
  - TrueNAS configuration optimization
  - NFS performance tuning
```

### Synology NAS (Backup Storage)
```yaml
Hardware:
  Model: Synology [Model TBD]
  Role: Backup and archival storage
  Interface: Gigabit Ethernet
  
Network:
  Hostname: synology-backup.mumblescavern.local
  IP: 192.168.10.21
  VLAN: Management (10)
  
Status:
  Current: 🟢 Active backup system
  Health: Stable
  
Services:
  - Automated backups from ASUSTOR
  - Configuration backups
  - Long-term archival storage
```

## Network Infrastructure

### UniFi Dream Machine Pro
```yaml
Hardware:
  Model: UniFi Dream Machine Pro
  Role: Router, firewall, controller
  Ports: Multiple gigabit + SFP+
  
Network:
  Hostname: udmp.mumblescavern.local
  Management IP: 192.168.1.1
  Controller: Built-in UniFi Network Application
  
Status:
  Current: 🟢 Operational
  VLANs: Configured and tested
  Performance: Excellent
  
Configuration:
  - VLAN 1 (Default): 192.168.1.x
  - VLAN 10 (Management): 192.168.10.x
  - VLAN 20 (Services): 192.168.20.x  
  - VLAN 80 (Gaming): 192.168.80.x
  - Inter-VLAN routing: Configured
  - QoS: Gaming optimization enabled
```

### UniFi Flex Mini Switch
```yaml
Hardware:
  Model: UniFi Flex Mini
  Ports: 5 gigabit ports
  Power: PoE powered from UDMP
  
Network:
  Hostname: flex-mini.mumblescavern.local
  Management IP: 192.168.10.5
  VLAN: Management (10)
  
Status:
  Current: 🟢 Operational
  Port utilization: 4/5 ports active
  
Port Configuration:
  Port 1: Drowzee (Node 1)
  Port 2: Sleepy (Node 2)
  Port 3: Reserved for Snooze (Node 3)
  Port 4: Reserved for Nappy (Node 4)
  Port 5: Uplink to UDMP
```

## Gaming & Workstation Infrastructure

### Beast Rig (Primary Gaming PC)
```yaml
Hardware:
  CPU: AMD Ryzen 5600X
  GPU: NVIDIA RTX 3070
  RAM: 64GB DDR4
  Storage: NVMe SSDs
  
Network:
  Hostname: beast-rig.mumblescavern.local
  IP: 192.168.80.10
  VLAN: Gaming (80)
  Connection: Wired gigabit
  
Status:
  Current: 🟢 Active gaming workstation
  Role: Primary gaming and content creation
  Performance: Excellent
  
Services:
  - Steam gaming library
  - Content creation software
  - Game streaming (Sunshine server)
  
Monitoring:
  - Gaming performance metrics
  - Temperature monitoring
  - Network latency to gaming services
```

### Snorlax Prime (Transitioning Server)
```yaml
Hardware:
  CPU: Intel i5-6500
  GPU: NVIDIA GTX 970
  RAM: 32GB DDR4
  Storage: Multiple drives
  
Network:
  Hostname: snorlax-prime.mumblescavern.local
  Current IP: [Legacy configuration]
  Target IP: 192.168.80.15 (Gaming VLAN)
  
Status:
  Current: 🟡 Running legacy homelab services
  Target: Gaming support / development workstation
  Migration: In progress to Dell cluster
  
Current Services (To be migrated):
  - Existing Plex server
  - *arr stack services
  - Authentication services
  - Reverse proxy
  
Future Role:
  - Gaming support services
  - Development environment
  - RetroPie/retro gaming server
  - Testing and experimentation
```

### Steam Deck
```yaml
Hardware:
  Model: Steam Deck (64GB/256GB/512GB)
  Connectivity: Wi-Fi 802.11ac
  
Network:
  Hostname: sleepy-deck.mumblescavern.local
  IP: 192.168.80.20
  VLAN: Gaming (80)
  Connection: Wi-Fi
  
Status:
  Current: 🟢 Active portable gaming
  Services: Steam client, emulation
  
Gaming Services:
  - Steam Remote Play from Beast Rig
  - Local gaming library
  - Emulation (RetroArch, etc.)
  - Network game streaming
```

### MacBook Pro (Tri-Role Workstation)
```yaml
Hardware:
  Model: MacBook Pro [Specific model TBD]
  CPU: Apple Silicon / Intel
  RAM: [To be specified]
  Storage: SSD
  
Network:
  Hostname: macbook-cavern.mumblescavern.local
  Primary IP: 192.168.20.50
  VLAN: Services (20) - Primary
  Additional Access: All VLANs via routing
  
Status:
  Current: 🔵 Integration planned
  Roles: Development + Media Production + Management
  
Role 1 - Development Workstation:
  - Infrastructure code development
  - Container orchestration
  - Database access to shared infrastructure
  - Git repository management
  
Role 2 - Media Production Hub:
  - Video editing and encoding
  - Audio production
  - Content creation
  - Direct NFS access to media storage
  
Role 3 - Mobile Management Station:
  - Homelab web interface access
  - SSH to all infrastructure
  - VPN for remote administration
  - Documentation and monitoring
  
Storage Integration:
  - Local: Active projects and cache
  - NFS: /Volumes/asustor-media
  - NFS: /Volumes/asustor-documents
  - Cloud sync: Documentation and configs
```

## Service Virtual Machines

### Shared Infrastructure VM
```yaml
Target Host: Any Dell node (high availability)
Hostname: shared-infra-cluster.mumblescavern.local
IP: 192.168.20.15
Resources:
  CPU: 4 cores
  RAM: 8GB
  Storage: 50GB + database storage
  
Services:
  - PostgreSQL (shared database)
  - Redis (caching and sessions)
  - Backup services
  
Status: 🔵 Planned deployment
Priority: 🔴 Critical (required for other services)
```

### Monitoring Stack VM
```yaml
Target Host: Any Dell node
Hostname: monitoring-cluster.mumblescavern.local  
IP: 192.168.20.16
Resources:
  CPU: 4 cores
  RAM: 6GB
  Storage: 100GB (metrics retention)
  
Services:
  - Prometheus (metrics collection)
  - Grafana (dashboards)
  - AlertManager (notifications)
  - Node exporters (all nodes)
  
Status: 🔵 Planned deployment
Priority: 🟡 High (operational visibility)
```

### Traefik Reverse Proxy VM
```yaml
Target Host: Any Dell node (HA considerations)
Hostname: traefik-cluster.mumblescavern.local
IP: 192.168.20.17
Resources:
  CPU: 2 cores
  RAM: 4GB
  Storage: 20GB
  
Services:
  - Traefik reverse proxy
  - SSL certificate management
  - Load balancing
  - External access gateway
  
Status: 🔵 Planned deployment  
Priority: 🟡 High (external access)
```

## Monitoring & Alerting Strategy

### Critical Metrics to Monitor

#### Infrastructure Health
```yaml
Dell Cluster Nodes:
  - CPU utilization and temperature
  - Memory usage and availability
  - Disk space and I/O performance
  - Network utilization and errors
  - Proxmox service health
  
Storage Systems:
  - NFS mount availability
  - Storage capacity and growth trends
  - Backup completion status
  - RAID health (if applicable)
  
Network Infrastructure:
  - Inter-VLAN routing performance
  - Internet connectivity and speed
  - Switch port utilization
  - Wi-Fi performance for mobile devices
```

#### Service Health
```yaml
Media Services:
  - Plex server availability and transcoding load
  - *arr services download queue and failures
  - Media library growth and disk space
  - Download bandwidth utilization
  
Shared Infrastructure:
  - Database connection pool and performance
  - Cache hit rates and memory usage
  - Backup job completion status
  
Gaming Infrastructure:
  - Game streaming latency and quality
  - Gaming device network performance
  - Server resource utilization during gaming
```

### Alert Thresholds
```yaml
Critical Alerts (Immediate):
  - Node down or unreachable
  - Storage >90% full
  - Service completely unavailable
  - Network connectivity loss
  - Security breach indicators
  
Warning Alerts (24h response):
  - CPU >80% for 10+ minutes
  - Memory >85% sustained
  - Storage >75% full
  - Service degraded performance
  - Backup job failures
  
Info Alerts (Weekly review):
  - Storage growth trends
  - Performance baseline changes
  - New device connections
  - Service usage statistics
```

## Maintenance Schedule

### Daily Automated
- Health checks for all services
- Backup verification
- Security updates (unattended)
- Log rotation and cleanup

### Weekly Manual
- Review monitoring dashboards
- Check storage utilization trends
- Verify backup integrity
- Update documentation if needed

### Monthly Manual
- Review and update device inventory
- Performance optimization review
- Security audit and updates
- Plan capacity expansion if needed

### Quarterly Manual
- Complete infrastructure review
- Hardware health assessment
- Disaster recovery testing
- Documentation comprehensive update

## Future Expansion Planning

### Short-term (3 months)
- Complete Dell cluster deployment (4 nodes)
- Deploy all planned services
- Implement comprehensive monitoring
- Complete Snorlax Prime role transition
- MacBook Pro full integration

### Medium-term (6 months)
- Raspberry Pi cluster for edge computing
- Advanced K3s deployment
- Smart home integration (Home Assistant)
- External access optimization
- Performance benchmarking and optimization

### Long-term (12 months)
- Additional storage expansion
- Networking equipment upgrades
- Advanced security implementations
- Disaster recovery site considerations
- Professional-grade monitoring and alerting

## Emergency Procedures

### Node Failure
1. Identify failed node via monitoring
2. Migrate VMs to healthy nodes
3. Investigate and repair hardware
4. Re-join node to cluster
5. Verify service restoration

### Network Outage
1. Check internet connectivity
2. Verify UDMP and switch status
3. Test VLAN routing and DNS
4. Restart network equipment if needed
5. Monitor service restoration

### Storage Failure
1. Verify NFS mount status
2. Check ASUSTOR NAS health
3. Activate backup storage if needed
4. Plan data recovery procedures
5. Update backup strategies

### Complete Infrastructure Loss
1. Activate disaster recovery procedures
2. Restore from off-site backups
3. Rebuild core infrastructure first
4. Restore services in priority order
5. Verify complete functionality

This device inventory serves as the living documentation for all aspects of the Mumbles Cavern homelab ecosystem, providing operational visibility and maintenance guidance for reliable infrastructure management.