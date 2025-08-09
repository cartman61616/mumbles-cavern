# Nuclear Cluster Rebuild Plan - Drowzee & Sleepy

## Overview
Plan for complete cluster rebuild to resolve naming inconsistency and establish proper Snorlax-themed infrastructure foundation.

## Current State (2025-08-09)

### Cluster Status
- **Backend**: Healthy 2-node cluster communication
- **Nodes**: drowzee (192.168.10.10), sleepy (192.168.10.11)
- **Issue**: Mixed naming in cluster filesystem (`pve-node1/2` + `drowzee/sleepy`)
- **Impact**: Primary node UI cannot see secondary node

### Services Deployed
- **VMs**: Test VM 101 only
- **Containers**: None
- **Storage**: Local storage only, no shared storage
- **Data**: ASUSTOR migration 70% complete (external to cluster)

### Network Configuration (PRESERVE)
- **VLANs**: 1 (default), 10 (management), 20 (services), 80 (gaming)
- **IPs**: Static assignments confirmed and working
- **Switch**: USW-Lite-16-PoE with proper VLAN trunking
- **Routing**: Inter-VLAN communication validated

## Rebuild Timeline

### Phase 1: Pre-Rebuild Preparation
**Duration**: 30 minutes
**Status**: Ready when ASUSTOR migration completes

#### Checklist
- [ ] ASUSTOR data migration completed (currently 70%)
- [ ] Network configuration documented
- [ ] VLAN settings backed up in UniFi controller
- [ ] Node hardware confirmed operational
- [ ] Gaming session completed 🎮

#### Documentation Backup
```bash
# Current network config to preserve
Node 1 (drowzee): 192.168.10.10/24, gateway 192.168.10.1
Node 2 (sleepy):  192.168.10.11/24, gateway 192.168.10.1
Bridge: vmbr0 with VLAN-aware enabled
VLANs: 1,10,20,80 confirmed working
```

### Phase 2: Cluster Destruction
**Duration**: 15 minutes
**Risk**: Low (no production services)

#### Process
1. **Remove secondary node from cluster**
   ```bash
   # On drowzee
   pvecm delnode sleepy
   ```

2. **Destroy cluster on primary node**
   ```bash
   # On drowzee
   systemctl stop pve-cluster pvedaemon pveproxy
   pmxcfs -l  # Leave cluster mode
   rm -rf /etc/pve/cluster.conf
   rm -rf /etc/corosync/
   ```

3. **Clean secondary node**
   ```bash
   # On sleepy
   systemctl stop pve-cluster pvedaemon pveproxy
   rm -rf /etc/pve/
   rm -rf /etc/corosync/
   ```

### Phase 3: Clean Proxmox Reinstall
**Duration**: 45 minutes per node (can be parallel)

#### Process
1. **Preserve network configuration**
   ```bash
   # Backup /etc/network/interfaces on both nodes
   cp /etc/network/interfaces /etc/network/interfaces.backup
   ```

2. **Reinstall Proxmox** (if needed) or clean reset
   - Boot from Proxmox installer
   - Use proper hostnames from installation: `drowzee`, `sleepy`
   - Configure same IP addresses: .10, .11
   - Restore network configuration

3. **Validate individual nodes**
   ```bash
   # Test each node independently
   ping 192.168.10.1  # Gateway test
   ping 8.8.8.8       # Internet test
   # Access web UI: https://192.168.10.10:8006
   # Access web UI: https://192.168.10.11:8006
   ```

### Phase 4: Clean Cluster Creation
**Duration**: 30 minutes

#### Process
1. **Create cluster on primary node**
   ```bash
   # On drowzee
   pvecm create mumbles-cluster --bindnet0_addr 192.168.10.10
   ```

2. **Join secondary node to cluster**
   ```bash
   # On sleepy
   pvecm add 192.168.10.10 --use_ssh
   ```

3. **Validate cluster health**
   ```bash
   # On either node
   pvecm status
   pvecm nodes
   # Both should show consistent drowzee/sleepy naming
   ```

### Phase 5: Validation & Testing
**Duration**: 30 minutes

#### Success Criteria
- [ ] Both nodes visible in single web UI
- [ ] Cluster status shows consistent naming
- [ ] `pvecm nodes` shows drowzee/sleepy only
- [ ] Test VM creation and migration between nodes
- [ ] Network connectivity confirmed across all VLANs

#### Test Commands
```bash
# Cluster validation
pvecm status | grep -E "(drowzee|sleepy|pve-node)"
pvecm nodes | grep -E "(drowzee|sleepy|pve-node)"

# UI validation - check both nodes appear in:
# https://192.168.10.10:8006 (should show both drowzee & sleepy)

# Network validation
ping 192.168.10.1   # Management gateway
ping 192.168.20.1   # Services gateway
ping 192.168.80.1   # Gaming gateway
```

## Post-Rebuild Actions

### Immediate (Same Day)
- [ ] Update README.md progress status
- [ ] Test storage integration readiness
- [ ] Create VM template for service deployment
- [ ] Document lessons learned in cavern_learnings.md

### Next Phase (Following Day)
- [ ] Deploy ASUSTOR TrueNAS integration
- [ ] Configure shared storage pools
- [ ] Deploy shared infrastructure services
- [ ] Begin essential services deployment

## Risk Mitigation

### Backup Strategy
- **Network Config**: Preserved in UniFi controller
- **Documentation**: All IP assignments documented
- **Data**: ASUSTOR migration external to cluster
- **Rollback**: Can recreate current mixed-name cluster if needed

### Communication Plan
- **Timeline**: Execute during planned maintenance window
- **Status Updates**: Update README.md and commit progress
- **Success Metrics**: Clear UI display and consistent naming

### Emergency Procedures
If rebuild fails:
1. **Fallback**: Recreate current mixed-name cluster
2. **Network**: Restore from UniFi controller backup
3. **Documentation**: Revert README.md changes
4. **Timeline**: Retry during next maintenance window

## Expected Outcomes

### Infrastructure Quality
- **Professional Standards**: Consistent Snorlax naming throughout
- **Management Simplicity**: Single UI managing both nodes
- **Future-Proofing**: Clean foundation for storage and services
- **Documentation Accuracy**: Guides match actual infrastructure

### Technical Benefits
- **UI Functionality**: Complete cluster visibility
- **Service Deployment**: Ready for immediate storage integration
- **Troubleshooting**: Consistent naming simplifies diagnostics
- **Scaling**: Proper foundation for nodes 3 & 4 addition

---

**Total Estimated Time**: 2.5-3 hours
**Risk Level**: Low (clean boxes, no production services)
**Success Probability**: High (well-documented network config)

*Execute when ASUSTOR migration reaches 100% and gaming session complete* 🎮 ➡️ 🔧