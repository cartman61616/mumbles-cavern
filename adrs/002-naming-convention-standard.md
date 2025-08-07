# ADR-002: Infrastructure Naming Convention Standard

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

The homelab infrastructure currently uses generic naming conventions (pve-node1, pve-node2) that don't provide meaningful context or personality to the environment. With expansion to 4 Dell nodes plus additional devices, a consistent and memorable naming strategy is needed for easier management, documentation, and troubleshooting.

## Decision

Implement a **Snorlax-themed naming convention** with clear device type identification and consistent FQDN structure using the domain `mumblescavern.local`.

### Core Infrastructure Naming

**Dell OptiPlex Proxmox Cluster**:
- `drowzee.mumblescavern.local` (Node 1) - Media hub with transcoding
- `sleepy.mumblescavern.local` (Node 2) - Media automation
- `snooze.mumblescavern.local` (Node 3) - Productivity services  
- `nappy.mumblescavern.local` (Node 4) - Development environment

**Storage & Network**:
- `asustor-nas.mumblescavern.local` - Main NFS storage
- `synology-backup.mumblescavern.local` - Backup storage
- `udmp.mumblescavern.local` - UniFi Dream Machine Pro
- `flex-mini.mumblescavern.local` - UniFi Flex Mini switch

**Gaming & Workstation**:
- `mighty-snorlax.mumblescavern.local` - Primary gaming workstation
- `snorlax-prime.mumblescavern.local` - Current homelab server (transitioning)
- `sleepy-deck.mumblescavern.local` - Steam Deck
- `dreamy-mac.mumblescavern.local` - MacBook Pro tri-role station

### Service Naming Convention

**Virtual Machines**: `[service]-[node].mumblescavern.local`
- `plex-drowzee.mumblescavern.local`
- `arr-sleepy.mumblescavern.local`
- `shared-infra-cluster.mumblescavern.local`

**Containers**: `[service]-[function]` 
- Container names: `plex`, `sonarr`, `radarr`, `prometheus`, etc.

## Alternatives Considered

- **Option A**: Keep generic naming (pve-node1, server1) - Easy but unmemorable and unprofessional
- **Option B**: Technical naming (cpu-gpu-01, storage-02) - Descriptive but boring and rigid
- **Option C**: Snorlax theme (Selected) - Memorable, consistent, and personality-driven
- **Option D**: Mixed naming schemes - Inconsistent and confusing

## Consequences

### Positive Impacts
- Memorable and fun device identification
- Clear role identification through themed names
- Consistent FQDN structure for DNS/certificates
- Professional domain structure for external access
- Easier troubleshooting and documentation
- Scalable naming scheme for future devices

### Negative Impacts
- Requires immediate rename of existing infrastructure
- Need to update all documentation and configurations
- DNS and DHCP reservation updates required
- Temporary service interruptions during rename

### Implementation Requirements
- Update Proxmox node hostnames and cluster configuration
- Configure DNS entries for all devices
- Update DHCP reservations with proper hostnames
- Update SSH configs and documentation
- Regenerate SSL certificates with new names

## Implementation Notes

**Timing**: Implement now while infrastructure is young (2-node cluster) to minimize disruption.

**Theme Rationale**: Snorlax theme fits the "sleepy/relaxed" nature of a home environment while maintaining professionalism. Sleep-related names are memorable and appropriate for always-on infrastructure.

**FQDN Structure**: `.mumblescavern.local` provides clear domain separation and supports future external DNS/certificates.

**Scalability**: Theme can accommodate 20+ devices with variations (sleepy, drowsy, dozing, napping, resting, etc.).