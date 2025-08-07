# Mumbles Cavern Naming Conventions

**Reference ADR**: [ADR-002: Infrastructure Naming Convention Standard](adrs/002-naming-convention-standard.md)  
**Domain**: `mumblescavern.local`  
**Theme**: Snorlax/Sleep-related names  
**Last Updated**: 2025-01-07

## Core Naming Principles

### 1. FQDN Structure
All devices use the format: `[hostname].mumblescavern.local`

### 2. Theme Consistency
Sleep/rest-themed names that are memorable and fun while maintaining professionalism.

### 3. Role Identification
Names should suggest the device's primary function or capability.

### 4. Scalability
Naming scheme supports 20+ devices with thematic variations.

## Infrastructure Naming

### Dell OptiPlex Proxmox Cluster

| Hostname | FQDN | Role | Hardware | IP Address |
|----------|------|------|----------|------------|
| `drowzee` | `drowzee.mumblescavern.local` | Media Hub | Dell 7050 Micro (i5-7500T) | 192.168.10.10 |
| `sleepy` | `sleepy.mumblescavern.local` | Media Automation | Dell 7050 Micro (i5-7500T) | 192.168.10.11 |
| `snooze` | `snooze.mumblescavern.local` | Productivity Services | Dell 7050 Micro (i5-7500T) | 192.168.10.12 |
| `nappy` | `nappy.mumblescavern.local` | Development Environment | Dell 6500T (i5-6500) | 192.168.10.13 |

### Storage Infrastructure

| Hostname | FQDN | Role | Hardware | IP Address |
|----------|------|------|----------|------------|
| `asustor-nas` | `asustor-nas.mumblescavern.local` | Primary NFS Storage | ASUSTOR TrueNAS | 192.168.10.20 |
| `synology-backup` | `synology-backup.mumblescavern.local` | Backup Storage | Synology NAS | 192.168.10.21 |

### Network Infrastructure

| Hostname | FQDN | Role | Hardware | IP Address |
|----------|------|------|----------|------------|
| `udmp` | `udmp.mumblescavern.local` | Router/Security Gateway | UniFi Dream Machine Pro | 192.168.1.1 |
| `flex-mini` | `flex-mini.mumblescavern.local` | Managed Switch | UniFi Flex Mini | 192.168.10.5 |

### Gaming & Workstations

| Hostname | FQDN | Role | Hardware | IP Address |
|----------|------|------|----------|------------|
| `mighty-snorlax` | `mighty-snorlax.mumblescavern.local` | Primary Gaming PC | Ryzen 5600X, RTX 3070 | 192.168.80.10 |
| `snorlax-prime` | `snorlax-prime.mumblescavern.local` | Gaming/Dev Support | Intel i5-6500, GTX 970 | 192.168.80.15 |
| `sleepy-deck` | `sleepy-deck.mumblescavern.local` | Portable Gaming | Steam Deck | 192.168.80.20 |
| `dreamy-pro` | `dreamy-pro.mumblescavern.local` | Lab Integration Workstation | MacBook Pro 16" (2019) | 192.168.20.50 |

### Personal Devices (Munchlax Theme)

| Hostname | FQDN | Role | Hardware | IP Address |
|----------|------|------|----------|------------|
| `munchlax` | `munchlax.mumblescavern.local` | Personal Daily Driver | MacBook Air M4 (2024) | 192.168.20.51 |

### IoT Devices (Smart Home Theme)

| Hostname | FQDN | Role | Hardware | IP Address |
|----------|------|------|----------|------------|
| `sleepy-assistant` | `sleepy-assistant.mumblescavern.local` | Voice Assistant | Google Home Hub | 192.168.50.10 |
| `dreamy-light-hub` | `dreamy-light-hub.mumblescavern.local` | Smart Lighting Hub | Philips Hue Bridge | 192.168.50.20 |
| `cozy-light-*` | `cozy-light-01.mumblescavern.local` | Smart Lights | Philips Hue Bulbs | 192.168.50.21+ |
| `bright-light-*` | `bright-light-01.mumblescavern.local` | Content Lighting | Elgato Lights | 192.168.50.30+ |

## Service Naming

### Virtual Machine Naming
Format: `[service]-[node].mumblescavern.local`

| Service VM | FQDN | Host Node | Purpose |
|------------|------|-----------|---------|
| `plex-drowzee` | `plex-drowzee.mumblescavern.local` | Drowzee | Plex Media Server |
| `arr-sleepy` | `arr-sleepy.mumblescavern.local` | Sleepy | *arr Stack Services |
| `shared-infra` | `shared-infra-cluster.mumblescavern.local` | Any Node | PostgreSQL + Redis |
| `monitoring` | `monitoring-cluster.mumblescavern.local` | Any Node | Prometheus + Grafana |
| `traefik-proxy` | `traefik-cluster.mumblescavern.local` | Any Node | Reverse Proxy |
| `productivity` | `productivity-snooze.mumblescavern.local` | Snooze | Nextcloud, Vaultwarden |
| `development` | `development-nappy.mumblescavern.local` | Nappy | GitLab, Harbor, K3s |

### Container Naming
Simple service names without hostname suffixes:
- `plex`, `sonarr`, `radarr`, `prowlarr`, `qbittorrent`
- `postgres`, `redis`, `traefik`
- `prometheus`, `grafana`, `alertmanager`
- `nextcloud`, `vaultwarden`, `gitlab`

## IP Address Allocation

### VLAN 1 (Default) - 192.168.1.x
- `192.168.1.1`: UniFi Dream Machine Pro (Gateway)
- `192.168.1.2-9`: Reserved for network equipment
- `192.168.1.10-19`: Reserved for Proxmox nodes (legacy, migrating to VLAN 10)
- `192.168.1.20-49`: Service VMs (legacy, migrating to VLAN 20)
- `192.168.1.50-99`: User devices
- `192.168.1.100-199`: DHCP pool for guests/temporary devices

### VLAN 10 (Management) - 192.168.10.x
- `192.168.10.1`: UniFi Dream Machine Pro (VLAN interface)
- `192.168.10.5`: UniFi Flex Mini switch
- `192.168.10.10-13`: Dell Proxmox cluster nodes
- `192.168.10.20-29`: Storage infrastructure (NAS devices)
- `192.168.10.30-49`: Network infrastructure and management
- `192.168.10.50-99`: Reserved for future infrastructure

### VLAN 20 (Services) - 192.168.20.x
- `192.168.20.1`: UniFi Dream Machine Pro (VLAN interface)
- `192.168.20.10-19`: Core service VMs
- `192.168.20.20-39`: Media service VMs
- `192.168.20.40-49`: Productivity service VMs
- `192.168.20.50-59`: Development workstations (MacBook Pro)
- `192.168.20.60-99`: Additional service VMs

### VLAN 80 (Gaming) - 192.168.80.x
- `192.168.80.1`: UniFi Dream Machine Pro (VLAN interface)
- `192.168.80.10`: Beast-rig (primary gaming PC)
- `192.168.80.15`: Snorlax Prime (gaming/dev support)
- `192.168.80.20`: Steam Deck
- `192.168.80.30-49`: Gaming consoles and devices
- `192.168.80.50-99`: Additional gaming infrastructure

## DNS Configuration

### Internal DNS (UniFi)
All devices registered with their FQDN in UniFi DNS for internal resolution.

### External DNS (Future)
When external access is configured:
- `*.mumblescavern.com` - External domain for public services
- Cloudflare DNS management for external resolution
- Traefik for internal/external service routing

## Implementation Checklist

### Phase 1: Dell Cluster Renaming
- [ ] Rename `pve-node1` → `drowzee`
- [ ] Rename `pve-node2` → `sleepy`
- [ ] Update Proxmox cluster configuration
- [ ] Update SSH configurations and keys
- [ ] Update documentation references

### Phase 2: DNS & DHCP Updates
- [ ] Create DHCP reservations for all devices
- [ ] Configure UniFi DNS entries
- [ ] Update local DNS configurations
- [ ] Test name resolution across VLANs

### Phase 3: Service VM Creation
- [ ] Deploy VMs with proper naming convention
- [ ] Configure service containers with simple names
- [ ] Update service discovery and monitoring
- [ ] Document all service endpoints

### Phase 4: Future Device Integration
- [ ] Add Node 3 (Snooze) and Node 4 (Nappy) when available
- [ ] Configure MacBook Pro with proper hostname
- [ ] Update all documentation and procedures

## Expansion Names (Future Use)

### Additional Sleep-Themed Names
For future infrastructure expansion:
- `dozing`, `drowsy`, `resting`, `slumber`, `dreamy`
- `cozy`, `peaceful`, `tranquil`, `serene`, `calm`
- `lazy`, `relaxed`, `comfortable`, `soft`, `gentle`

### Service-Specific Themes
- **Database servers**: `mem-*` (memory-related: `memorable`, `mindful`)
- **Monitoring**: `watch-*` (`watchful`, `vigilant`)  
- **Security**: `guard-*` (`guardian`, `secure`)
- **Backup**: `safe-*` (`safekeep`, `vault`)

## Benefits of This Naming Convention

1. **Memorable**: Easy to remember device purposes
2. **Consistent**: Clear pattern across all infrastructure  
3. **Professional**: Suitable for documentation and presentations
4. **Scalable**: Theme supports significant expansion
5. **Fun**: Makes infrastructure management more enjoyable
6. **Functional**: Names hint at device roles and capabilities

This naming convention provides a solid foundation for the growing homelab infrastructure while maintaining both professionalism and personality.