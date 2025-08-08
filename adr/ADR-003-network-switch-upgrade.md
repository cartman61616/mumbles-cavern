# ADR-003: Network Switch Upgrade from Flex Mini to USW-Lite-16-PoE

**Date**: 2025-08-08  
**Status**: Accepted  
**Authors**: Mumbles Cavern Infrastructure Team  
**Reviewers**: N/A (Single operator)  

## Context

During VLAN configuration testing for Proxmox VM deployment, we discovered that the UniFi Flex Mini switch does not support VLAN tagging properly. The switch was blocking all tagged VLAN traffic with no option to enable it, preventing proper network segmentation for the homelab infrastructure.

### Problem Statement
- VM deployed with VLAN 10 tagging could not obtain IP address
- UniFi Flex Mini showed "Tagged VLAN Management is limited on this device" 
- "Block All" was the only available option for tagged VLAN traffic
- This prevented proper network segmentation essential for homelab security and organization

### Discovery Process
1. VM worked on default VLAN (192.168.1.254) ✅
2. VM failed on VLAN 10 with proper Proxmox configuration ❌
3. Proxmox bridge and VLAN configuration verified as correct ✅
4. UniFi interface revealed VLAN tagging limitation ❌

## Decision

**Upgrade network switch from UniFi Flex Mini to UniFi USW-Lite-16-PoE**

### Rationale

**Technical Requirements:**
- Full VLAN trunking support for homelab segmentation
- Proper tagged VLAN management capabilities
- PoE support for future IoT devices and access points
- 16 ports for infrastructure expansion
- Full UniFi integration and management

**Architecture Benefits:**
- Direct connection to UDM Pro (removing Flex Mini bottleneck)
- Support for all planned VLANs (1, 10, 20, 30, 50, 80, 90)
- Proper trunk port configuration
- Advanced switch management features

## Implementation

### New Network Architecture
```
UDM Pro → USW-Lite-16-PoE → Dell OptiPlex nodes
```

### Port Allocation
- **Port 1**: Uplink to UDM Pro (All VLANs trunk)
- **Port 2**: drowzee (Dell OptiPlex 7050)
- **Port 3**: sleepy (Dell OptiPlex 7050)  
- **Port 4**: snorlax (Dell OptiPlex 6500T) - Future
- **Port 5**: jigglypuff (Dell OptiPlex 7050) - Future
- **Ports 6-16**: Available for expansion

### VLAN Configuration
- **VLAN 1** (Default): Initial deployment
- **VLAN 10** (Management): Proxmox, infrastructure
- **VLAN 20** (Services): Application services  
- **VLAN 30** (Storage): NAS and storage traffic
- **VLAN 80** (Gaming): Gaming devices
- **VLAN 90** (Work): Management devices

## Consequences

### Positive
- ✅ **Full VLAN Support**: Proper network segmentation now possible
- ✅ **PoE Capability**: Future IoT and AP deployment support
- ✅ **Expansion Ready**: 16 ports for infrastructure growth
- ✅ **Performance**: Direct UDM Pro connection eliminates bottleneck
- ✅ **Management**: Advanced switch features and monitoring

### Negative
- ❌ **Cost**: Additional hardware investment (~$179)
- ❌ **Complexity**: More advanced switch requires proper configuration
- ❌ **Redundancy**: Single switch point of failure (acceptable for homelab)

### Risks Mitigated
- **Network Segmentation**: Essential for security and traffic management
- **VLAN Testing**: Can now properly test all planned network architecture
- **Future Expansion**: Ready for additional services and devices

## Alternatives Considered

### Alternative 1: Use vmbr0.X Bridge Interfaces
- **Pros**: Works with existing Flex Mini
- **Cons**: Complex Proxmox bridge configuration, doesn't solve fundamental VLAN limitation

### Alternative 2: Keep Default VLAN for All Services  
- **Pros**: Simple configuration, works with current hardware
- **Cons**: No network segmentation, security concerns, traffic management issues

### Alternative 3: Use Separate Physical NICs
- **Pros**: Hardware-level separation
- **Cons**: Dell OptiPlex micro nodes only have single NIC, requires USB adapters

## Monitoring and Success Criteria

### Immediate Success Metrics
- [ ] VM can obtain IP address on VLAN 10
- [ ] All VLANs properly configured and accessible
- [ ] Inter-VLAN routing working as designed
- [ ] Switch properly adopted in UniFi controller

### Long-term Success Metrics  
- [ ] Network segmentation operational for all services
- [ ] Gaming VLAN providing optimized performance
- [ ] Management VLAN securing infrastructure access
- [ ] PoE supporting IoT device deployment

### Rollback Plan
If USW-Lite-16-PoE fails or causes issues:
1. Reconnect Flex Mini
2. Configure all VMs on default VLAN temporarily
3. Implement network segmentation at Proxmox bridge level
4. Plan alternative switch solution

## Implementation Timeline

- **Phase 1** (Day 1): Physical installation and UniFi adoption
- **Phase 2** (Day 1): VLAN configuration and port profiles  
- **Phase 3** (Day 1): VM testing and validation
- **Phase 4** (Day 2+): Service migration to proper VLANs

## Related Documents

- [Network Setup Guide](../network_setup.md) - Updated with USW-Lite-16-PoE configuration
- [Master Deployment Guide](../master_deployment_guide.md) - Network architecture overview
- [Cavern Learnings](../cavern_learnings.md) - VLAN troubleshooting lessons

## Notes

This ADR represents a critical infrastructure decision that enables proper network architecture for the homelab. The UniFi Flex Mini's VLAN limitations were discovered through systematic troubleshooting, highlighting the importance of hardware validation during infrastructure planning.

The USW-Lite-16-PoE upgrade provides the foundation for secure, segmented network architecture essential for a production-quality homelab environment.