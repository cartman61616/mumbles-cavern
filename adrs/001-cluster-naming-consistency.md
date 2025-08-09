# ADR-001: Cluster Naming Consistency Issue and Nuclear Rebuild Decision

## Status
Proposed

## Date
2025-08-09

## Context
During cluster formation phase, we discovered a critical naming inconsistency in the Proxmox cluster that affects UI functionality and long-term maintainability.

### Current State Discovery
- **Backend Cluster**: Healthy and functional (confirmed via `pvecm status`)
- **UI Display Issue**: Primary node (drowzee) web UI cannot see secondary node (sleepy)
- **Root Cause**: Mixed naming scheme in cluster filesystem
  - Legacy names: `pve-node1`, `pve-node2` (from initial cluster creation)
  - Hostname names: `drowzee`, `sleepy` (from post-creation renaming)
  - Filesystem shows both: `/etc/pve/nodes/` contains `drowzee`, `pve-node1`, `pve-node2`, `sleepy`

### Technical Impact
- **Functional**: Backend cluster communication works perfectly
- **Management**: UI inconsistency makes cluster management confusing
- **Future Risk**: Mixed naming could complicate storage integration and service deployment
- **Professional Standards**: Inconsistent naming violates infrastructure best practices

### Current Services Status
- **VMs**: Minimal test VM (101) only
- **Containers**: None deployed
- **Storage**: Local storage only, no shared storage configured
- **Services**: Only Proxmox base installation
- **Data Migration**: ASUSTOR data migration in progress (external to cluster)

## Decision
**Nuclear cluster rebuild** with proper naming from inception.

### Rationale
1. **Clean Slate Opportunity**: No production services or data at risk
2. **Future-Proofing**: Establish proper naming standards before complexity grows
3. **Professional Standards**: Infrastructure should follow consistent naming conventions
4. **Risk Minimization**: Fix now vs. complex migration later with services deployed

### Timing Decision
**Wait for ASUSTOR data migration completion** before executing rebuild to avoid:
- Network disruption during data transfer
- System resource contention
- Risk of data corruption

## Consequences

### Positive
- Clean, consistent cluster naming from foundation
- Professional-grade infrastructure standards
- Simplified future management and troubleshooting
- Proper foundation for storage and service deployment
- Clean documentation and procedures

### Negative
- Short-term delay in storage integration
- Need to recreate cluster (minimal effort with clean boxes)
- Brief infrastructure downtime during rebuild

### Risk Mitigation
- **Timing**: Execute during planned maintenance window post-migration
- **Preparation**: Document current network configuration for rapid rebuild
- **Testing**: Validate cluster functionality immediately after rebuild
- **Documentation**: Update all procedures with correct naming

## Implementation Plan

### Pre-Rebuild Checklist
- [ ] ASUSTOR data migration completed
- [ ] Network configuration documented
- [ ] VLAN settings backed up
- [ ] Node hardware validated

### Rebuild Process
1. **Destruction Phase**
   - Destroy existing cluster on both nodes
   - Clean Proxmox installation (preserve network config)
   
2. **Recreation Phase**
   - Install Proxmox with proper hostnames (drowzee, sleepy)
   - Create cluster with consistent Snorlax naming
   - Validate both nodes visible in single UI
   
3. **Validation Phase**
   - Confirm cluster health and communication
   - Test VM creation and management
   - Validate storage readiness
   - Update documentation

### Success Criteria
- Single web UI shows both nodes consistently
- Cluster commands show consistent naming
- Ready for immediate storage integration
- Professional naming standards established

## References
- **Technical Discovery**: Session analysis revealed `/etc/pve/nodes/` inconsistency
- **Cluster Health**: `pvecm status` confirmed backend functionality
- **UI Issue**: Primary node web interface missing secondary node display
- **Service Status**: Minimal deployment makes nuclear option low-risk

---

*This ADR documents a critical infrastructure decision made during the cluster formation phase to ensure long-term maintainability and professional standards.*