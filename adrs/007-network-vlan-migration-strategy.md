# ADR-007: Network VLAN Migration Strategy

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

The current homelab infrastructure is deployed on a flat network (192.168.0.x/192.168.1.x) with all devices sharing the same broadcast domain. This creates security risks, performance issues, and lacks the network segmentation required for proper homelab architecture. Additionally, device naming follows generic conventions rather than the established Snorlax theme.

The infrastructure needs to migrate from this flat network to proper VLAN segmentation while simultaneously implementing the Snorlax naming convention, all before deploying production services.

## Decision

Implement a **coordinated network and naming migration** that moves all infrastructure to proper VLAN segmentation with Snorlax-themed naming in a single orchestrated effort.

### Target VLAN Architecture (Already Implemented)

```yaml
VLAN 1 (Default): 192.168.1.x
  - Personal trusted devices, network printer
  - Internet access through VPN (future)

VLAN 10 (Management): 192.168.10.x  
  - Proxmox cluster nodes (pve-node1 already at 192.168.10.10)
  - Storage infrastructure (ASUSTOR, Synology)
  - Network management interfaces

VLAN 20 (Services): 192.168.20.x
  - Homelab service VMs (Plex, databases, web apps)
  - MacBook Pro lab integration
  - Personal daily driver (VPN protected)

VLAN 30 (Storage): 192.168.30.x
  - High-bandwidth storage traffic
  - IGMP snooping disabled for performance

VLAN 50 (IoT): 192.168.50.x
  - Smart home devices (VPN + isolated)
  - Home Assistant integration only

VLAN 80 (Gaming): 192.168.80.x
  - Gaming devices and consoles (Nintendo Switches already configured)
  - Gaming-optimized QoS and VPN protection
  - IGMP snooping disabled for performance

VLAN 90 (Work): 192.168.90.x
  - Work devices only (direct internet, compliance)
  - Isolated from homelab infrastructure
```

### Migration Timing Strategy

Execute migration **immediately before service deployment** to:
- Avoid service disruption (no services running yet)
- Eliminate future complex migrations
- Provide clean foundation for all deployments

## Alternatives Considered

### Option A: Gradual VLAN Migration
**Description**: Migrate devices to VLANs over time while keeping flat network operational  
**Rejected because**: Creates hybrid complexity, requires multiple maintenance windows, potential service disruptions

### Option B: Service-First, Network-Later  
**Description**: Deploy services on flat network, migrate to VLANs later  
**Rejected because**: Complex service reconfiguration, DNS issues, certificate regeneration, documentation confusion

### Option C: VLAN-Only Migration (Keep Generic Names)
**Description**: Implement VLANs but delay naming updates  
**Rejected because**: Misses opportunity for clean slate, documentation becomes inconsistent

### Option D: Coordinated Migration (Selected)
**Description**: Single orchestrated migration of both network segmentation and naming  
**Selected because**: Clean slate approach, single maintenance window, proper foundation for growth

## Consequences

### Positive Impacts

#### Security Benefits
- Network segmentation isolates work from personal traffic
- IoT devices contained with limited access  
- Infrastructure protected in management VLAN
- Gaming traffic separated and optimized

#### Operational Benefits
- Professional naming convention (drowzee, sleepy, snooze, nappy)
- Clear role identification through hostnames
- Scalable IP addressing scheme
- Documentation alignment with reality

#### Future-Proofing Benefits
- VPN integration architecture ready
- Node 3&4 can join with proper configuration
- Service deployment follows proper network design
- Monitoring uses meaningful device names

### Negative Impacts

#### Migration Complexity
- Coordinated change affects entire infrastructure
- Requires careful planning and execution
- Potential for network connectivity issues during migration
- Learning curve for VLAN management

#### Temporary Disruption
- 2-3 hour maintenance window required
- Cluster communication may need rebuilding
- SSH configurations require updates
- DNS propagation delays possible

### Implementation Requirements

#### Pre-Migration Preparation
- Create all VLANs in UniFi controller
- Configure DHCP pools and reservations
- Document all device MAC addresses
- Plan rollback procedures

#### Migration Execution
- Update Proxmox node network configurations
- Migrate ASUSTOR NAS during TrueNAS reset
- Update cluster configuration files
- Verify inter-VLAN routing rules

#### Post-Migration Validation
- Test connectivity between all VLANs
- Verify cluster communication
- Validate service access patterns
- Update documentation and SSH configs

## Implementation Notes

### Migration Phases
1. **UniFi VLAN Creation** (30 minutes)
2. **Node 1 Migration** (drowzee.mumblescavern.local → 192.168.10.10)
3. **Node 2 Migration** (sleepy.mumblescavern.local → 192.168.10.11)  
4. **Cluster Reconfiguration** (rebuild if necessary)
5. **ASUSTOR Migration** (during TrueNAS reset)
6. **Device Migration** (ongoing as needed)

### Critical Success Factors
- Execute before any services are deployed
- Maintain console access to all nodes
- Have rollback plan ready
- Test connectivity thoroughly post-migration

### Risk Mitigation
- Fresh Proxmox installation option if migration fails
- No service dependencies to break (clean slate advantage)
- Console access for recovery
- Documented rollback procedures

This migration provides the security, performance, and organizational foundation required for professional homelab operations while implementing our Snorlax naming theme from day one.