# ADR-006: NordVPN Integration Strategy

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

The Snorlax homelab requires VPN integration for security and anonymity while maintaining proper functionality for work devices, network printers, and essential services. The solution must provide comprehensive protection without disrupting work compliance or breaking network services like printing.

## Decision

Implement a **centralized VPN gateway approach** using NordVPN with smart routing that automatically routes homelab traffic through VPN while excluding work devices and printers via dedicated VLANs.

### Core Architecture
- **VPN Gateway VM**: `sleepy-vpn.mumblescavern.local` handling all VPN routing
- **VLAN-based Routing**: Different internet access policies per VLAN
- **Smart Exclusions**: Work and printer traffic bypass VPN automatically
- **Kill Switch Protection**: Block internet if VPN connection fails

### Network Segmentation Strategy
```
VPN-Protected VLANs (Through NordVPN):
- VLAN 1 (Default): Personal devices → VPN
- VLAN 20 (Services): Homelab services → VPN  
- VLAN 80 (Gaming): Gaming devices → VPN

VPN-Excluded VLANs (Direct Internet):
- VLAN 10 (Management): Infrastructure → Direct
- VLAN 30 (Work): Work devices → Direct (NEW)
- VLAN 40 (Printer): Network printers → Direct (NEW)
```

## Alternatives Considered

- **Option A**: Individual VPN clients on each device - Complex management, inconsistent protection
- **Option B**: Router-level VPN for entire network - Would affect work compliance and printing
- **Option C**: Centralized VPN gateway with smart routing (Selected) - Best balance of security and functionality
- **Option D**: No VPN integration - Misses security and anonymity benefits

## Consequences

### Positive Impacts
- **Comprehensive Security**: Gaming, media, and personal traffic anonymized
- **Work Compliance**: Work devices maintain direct internet (corporate friendly)
- **Network Services**: Printers and management traffic unaffected
- **Gaming Performance**: VPN optimized for gaming with NordVPN gaming servers
- **Centralized Management**: Single point of control for all VPN routing
- **Automatic Protection**: New devices automatically get VPN based on VLAN assignment

### Negative Impacts
- **Additional Infrastructure**: Requires dedicated VPN gateway VM
- **Network Complexity**: More VLANs and routing rules to manage
- **Single Point of Failure**: VPN gateway failure affects protected devices
- **Initial Setup Complexity**: Requires careful network reconfiguration

### Implementation Requirements
- Deploy VPN gateway VM with multi-VLAN interfaces
- Create two new VLANs (Work, Printer) in UniFi
- Configure NordVPN client with kill switch protection
- Implement policy-based routing rules
- Update firewall rules for VLAN isolation
- Set up health monitoring and automatic reconnection

## Implementation Notes

**VLAN Assignment Strategy**:
- Device VLAN assignment determines VPN routing automatically
- Work laptop connects to Work VLAN → Direct internet
- Gaming devices connect to Gaming VLAN → Through VPN
- Network printer on Printer VLAN → Direct internet with access from all VLANs

**Security Considerations**:
- Kill switch prevents traffic leaks if VPN fails
- Work VLAN isolated from homelab networks
- Management traffic remains direct for NAS cloud services
- VPN gateway has health monitoring and auto-recovery

**Performance Optimization**:
- Gaming traffic uses NordVPN gaming-optimized servers
- Printer traffic bypasses VPN for low latency
- Management traffic direct for remote access and backups
- VPN gateway provisioned with adequate resources for throughput

**Future Flexibility**:
- Easy to add/remove devices from VPN protection via VLAN assignment
- Can implement per-device exceptions within VLANs if needed
- Support for multiple VPN providers by changing gateway configuration
- Monitoring integration for network visibility