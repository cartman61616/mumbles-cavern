# ADR-001: VLAN Segmentation Strategy

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

The homelab infrastructure requires network segmentation for security, performance optimization, and service organization. With gaming devices, productivity services, and management interfaces all sharing network resources, proper VLAN segmentation is essential for both security isolation and Quality of Service (QoS) optimization.

## Decision

Implement immediate comprehensive VLAN segmentation using UniFi networking equipment with the following structure:

- **VLAN 1 (Default)**: 192.168.1.x - Initial deployment and general user devices
- **VLAN 10 (Management)**: 192.168.10.x - Infrastructure management (Proxmox, NAS, switches)
- **VLAN 20 (Services)**: 192.168.20.x - Homelab services (Plex, *arr stack, productivity apps)
- **VLAN 80 (Gaming)**: 192.168.80.x - Gaming devices with optimized QoS

## Alternatives Considered

- **Option A**: Single flat network - Simple but lacks security and QoS capabilities
- **Option B**: Deploy on default VLAN initially, migrate later - Risk of service disruption during migration
- **Option C**: Immediate segmentation (Selected) - More complex setup but proper foundation

## Consequences

### Positive Impacts
- Enhanced security through network isolation
- Gaming optimization with dedicated VLAN and QoS
- Clear service organization and troubleshooting
- Scalable foundation for future growth
- Professional-grade network architecture

### Negative Impacts
- Increased initial configuration complexity
- Requires proper inter-VLAN routing configuration
- More firewall rules to manage
- Additional DNS/DHCP configuration

### Implementation Requirements
- Configure UniFi VLANs and firewall rules
- Set up DHCP reservations for all devices
- Configure inter-VLAN routing for necessary communication
- Update Proxmox network configuration
- Implement QoS policies for gaming optimization

## Implementation Notes

- Gaming VLAN gets highest QoS priority for low latency
- Management VLAN restricted access from other networks
- Services VLAN allows controlled access from gaming and default VLANs
- All infrastructure follows this segmentation from deployment day one