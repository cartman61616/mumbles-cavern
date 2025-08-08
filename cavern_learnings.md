# Learnings from the Cavern - Deployment Insights & Troubleshooting

## Overview
This document captures key lessons learned, troubleshooting patterns, and optimization insights from the Mumbles Cavern homelab deployment. These learnings complement the main deployment guides and help avoid common pitfalls.

## 🚨 Critical Network Configuration Issues

### UniFi Flex Mini VLAN Limitations (August 2025)
**Issue**: VMs with VLAN tagging could not obtain IP addresses despite correct Proxmox configuration  
**Root Cause**: UniFi Flex Mini blocks all tagged VLAN traffic with no override option available

#### The Problem
```bash
# VM Configuration (Correct):
qm config 101 | grep net
# Result: net0: virtio=BC:24:11:8A:7C:05,bridge=vmbr0,tag=10

# Proxmox Bridge (Correct):
bridge vlan show
# Result: tap101i0    10 PVID Egress Untagged

# VM Network Status (Failed):
# Inside VM: ip addr show
# Result: eth0 has no IP address - DHCP fails completely
```

#### The Discovery Process
1. **VM works on default VLAN** (192.168.1.254) ✅
2. **VM fails on any tagged VLAN** (10, 20, 80) ❌  
3. **Host can reach VLAN gateways** (192.168.10.1) ✅
4. **UniFi interface shows**: "Tagged VLAN Management is limited on this device" ❌
5. **Only option available**: "Block All" for tagged VLANs ❌

#### The Solution: Hardware Upgrade
**Replaced**: UniFi Flex Mini → UniFi USW-Lite-16-PoE
```bash
# New Network Architecture:
UDM Pro → USW-Lite-16-PoE → Dell OptiPlex nodes

# Port Configuration:
Port 1: Uplink to UDM Pro (All VLANs trunk)
Port 2-5: Dell nodes (All VLANs)
Port 6-16: Expansion ports
```

#### Key Lessons
- **"Smart managed" ≠ "Fully managed"** - Flex Mini has significant VLAN limitations
- **Test VLAN functionality early** - Don't assume hardware supports planned features  
- **Hardware research critical** - Verify switch capabilities match network design
- **Systematic troubleshooting works** - Layer-by-layer diagnosis identified root cause
- **Investment in proper networking pays off** - USW-Lite-16-PoE enables proper VLAN architecture

#### Verification Commands After Fix
```bash
# Test VM on VLAN 10
qm stop 101
qm set 101 --net0 virtio,bridge=vmbr0,tag=10
qm set 101 --ipconfig0 ip=192.168.10.50/24,gw=192.168.10.1
qm start 101
sleep 30
nmap -sn 192.168.10.0/24
# Expected: VM appears at 192.168.10.50 ✅
```

## 🚨 Critical Network Configuration Issues

### Inter-VLAN Routing Gateway Mismatch
**Issue**: Primary Node (Phase 1) had inter-VLAN routing failure  
**Root Cause**: Gateway configuration mismatch between node VLAN and gateway VLAN

#### The Problem
```bash
# Node configuration:
Node IP: 192.168.10.10  (Management VLAN 10)
Gateway:  192.168.0.1   (Default VLAN 1)

# Result: No routing between VLANs = network failure
```

#### The Solution
```bash
# Correct configuration:
Node IP: 192.168.10.10  (Management VLAN 10)
Gateway:  192.168.10.1  (Management VLAN 10)

# Edit /etc/network/interfaces:
auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.10.1    # <-- Must match VLAN subnet
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094
```

#### Verification Commands
```bash
# Test VLAN gateway
ping 192.168.10.1

# Test internet connectivity
ping 8.8.8.8

# Test DNS resolution
ping google.com

# Check routing table
ip route show
# Should show: default via 192.168.10.1 dev vmbr0
```

**Prevention**: Always ensure gateway IP matches the node's VLAN subnet. Double-check network architecture documentation before configuration.

### VPN Interference with Local Network Configuration
**Issue**: Network timeouts and routing failures during homelab deployment  
**Root Cause**: VPN client interfering with local network routing and DNS resolution

#### The Problem
```bash
# VPN active symptoms:
- Timeouts connecting to local devices (192.168.10.x)
- DNS resolution going through VPN servers
- Default route pointing to VPN gateway instead of local
- Inter-VLAN communication failures
- Proxmox web UI timeouts
```

#### The Solution
```bash
# Disconnect VPN during homelab work:
# macOS: Disconnect from VPN in Network Settings
# Windows: Disconnect VPN client
# Linux: sudo wg-quick down [config] or disconnect client

# Verify local routing restored:
ip route show
# Should show local gateway (192.168.10.1) as default

# Test local connectivity:
ping 192.168.10.1    # Local gateway
ping 192.168.10.10   # Node 1
nslookup google.com  # Should use local DNS
```

#### VPN Best Practices for Homelab
1. **Disconnect VPN** during infrastructure deployment
2. **Use VPN selectively** - only when accessing external resources
3. **Split tunneling** if VPN supports it (route only specific traffic)
4. **Document VPN impact** when troubleshooting network issues
5. **Test without VPN first** when diagnosing connectivity problems

**Detection**: If experiencing unexplained network timeouts or routing issues, check VPN status first before deep network troubleshooting.

## 🔧 Proxmox Repository Configuration

### Enterprise Repository Subscription Errors
**Issue**: Proxmox `apt update` fails with subscription warnings/errors  
**Root Cause**: Default installation includes enterprise repositories without subscription

#### The Problem
```bash
# Default Proxmox installation includes:
/etc/apt/sources.list.d/pve-enterprise.list
/etc/apt/sources.list.d/ceph.list

# Both contain enterprise repos requiring paid subscription
# Result: apt update fails or shows warnings
```

#### The Solution - Manual Learning Approach
**Philosophy**: Learn manually first (Node 1), then automate (Nodes 2-4)

```bash
# Manual process for understanding (Node 1):
ssh root@192.168.10.10

# Investigate current repository configuration
ls -la /etc/apt/sources.list.d/
cat /etc/apt/sources.list.d/pve-enterprise.list
cat /etc/apt/sources.list.d/ceph.list

# Comment out enterprise repos (manual editing)
nano /etc/apt/sources.list.d/pve-enterprise.list
# Add # to comment out the deb line
nano /etc/apt/sources.list.d/ceph.list  
# Add # to comment out the deb line

# Create no-subscription repo
nano /etc/apt/sources.list.d/pve-no-subscription.list
# Add: deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription

# Test the fix
apt update && apt full-upgrade -y
```

#### Automation Script (Nodes 2-4)
```bash
# Create automation script after learning manually
cat > /tmp/fix-repos.sh << 'EOF'
#!/bin/bash
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
apt update && apt full-upgrade -y
EOF

chmod +x /tmp/fix-repos.sh

# Deploy to remaining nodes
scp /tmp/fix-repos.sh root@192.168.10.11:/tmp/
ssh root@192.168.10.11 "/tmp/fix-repos.sh"
```

#### Manual-First Benefits Realized
- **Deep Understanding**: Learned both enterprise repos need fixing (not just PVE)
- **Troubleshooting Skills**: Can debug repository issues in future
- **Reliable Automation**: Script works perfectly because manual process was tested
- **Documentation Quality**: Real-world tested procedures

**Key Learning**: Both `pve-enterprise.list` AND `ceph.list` need to be commented out - automation scripts that only fix one will fail.

## 📊 Performance & Time Optimizations

### Network Setup Phase Performance
**Original Estimate**: 4-6 hours  
**Actual Time**: ~2-3 hours  
**Efficiency Gain**: 150-200% faster than expected

#### Success Factors
- **Strategic VLAN Approach**: Implemented proper segmentation from day one
- **Smart Device Pre-configuration**: Gaming devices configured during network setup
- **Quality Equipment**: UniFi gear simplified VLAN management
- **Preparation**: BIOS optimization completed first avoided delays

### Hardware Preparation Lessons
- **JetKVM vs USB**: JetKVM significantly faster than USB installs
- **BIOS First**: Complete BIOS optimization before network setup saves troubleshooting time
- **Power Planning**: Anker Prime USB-C charger works excellently for Dell OptiPlex

## 🔧 Troubleshooting Patterns

### Network Connectivity Debugging Sequence
**FIRST**: Check if VPN is active - disconnect if so  
1. **VPN Status Check**: Verify VPN is disconnected
2. **Gateway Test**: `ping [gateway-ip]`
3. **Internet Test**: `ping 8.8.8.8`
4. **DNS Test**: `ping google.com`
5. **Route Check**: `ip route show`
6. **Interface Check**: `ip addr show`

### Configuration File Priority
When troubleshooting network issues, check in this order:
1. `/etc/network/interfaces` - Primary network config
2. `systemctl status networking` - Service status
3. `journalctl -u networking` - Service logs
4. UniFi controller - VLAN/switch configuration
5. UDM Pro routing - Inter-VLAN rules

## 📐 Architecture Decisions & Rationale

### VLAN Strategy: Approach 2 (Immediate Segmentation)
**Decision**: Implement proper VLAN segmentation from day one  
**Alternative Considered**: Deploy on default VLAN, migrate later  
**Rationale**: Avoid migration complexity, professional architecture from start

#### VLAN Architecture
```
VLAN 10 (Management): 192.168.10.x - Infrastructure (Proxmox, NAS)
VLAN 20 (Services):   192.168.20.x - Homelab services (Plex, *arr)
VLAN 30 (Storage):    192.168.30.x - High-bandwidth storage traffic
VLAN 80 (Gaming):     192.168.80.x - Gaming devices and services
```

**Benefits Realized**:
- No future migration needed
- Clean network segmentation
- Gaming optimization from start
- Professional-grade setup

### Network Performance Optimizations
- **Smart Queues**: Enabled with 875/850 Mbps downrates
- **IGMP Snooping**: Disabled on Gaming/Storage VLANs for performance
- **Gaming QoS**: Automatic traffic prioritization achieved near-gigabit speeds

## 🛠️ Common Mistakes & Prevention

### Network Configuration Mistakes
1. **VPN Interference**: Always disconnect VPN during homelab work
2. **Gateway VLAN Mismatch**: Always match gateway to node's VLAN
3. **DNS Configuration**: Use VLAN gateway as DNS when possible
4. **Bridge VLAN-Aware**: Don't forget `bridge-vlan-aware yes`
5. **VLAN Range**: Include full range `bridge-vids 2-4094`

### Installation Sequence Mistakes
1. **Network Before BIOS**: Complete BIOS optimization first
2. **Skipping Connectivity Tests**: Always verify network before proceeding
3. **Documentation Drift**: Update guides with actual IP addresses used

## 🎯 Deployment Velocity Improvements

### Phase Completion Acceleration
- **Hardware Prep**: Complete on all nodes simultaneously
- **Network Setup**: Configure all VLANs and devices in single session
- **Documentation**: Update progress immediately, not in batches

### Tool & Equipment Insights
- **JetKVM**: Game-changer for remote installations
- **UniFi Equipment**: Simplified VLAN management significantly
- **Dell OptiPlex**: Excellent homelab hardware, minimal issues
- **Power Delivery**: USB-C PD works perfectly for micro form factor

## 📋 Quality Control Checklist

### Before Proceeding to Next Phase
- [ ] Network connectivity tests pass (gateway, internet, DNS)
- [ ] Web UI accessible without issues
- [ ] IP routing table shows correct default gateway
- [ ] Documentation updated with actual configuration
- [ ] Progress tracking updated in README.md

### Network Configuration Validation
- [ ] Gateway IP matches node VLAN subnet
- [ ] VLAN-aware bridge configured
- [ ] Connectivity tests pass from node CLI
- [ ] Web UI accessible from management network
- [ ] DNS resolution working

## 🔮 Future Optimization Opportunities

### Automation Potential
- **Ansible Playbooks**: Network configuration automation
- **Cloud-Init Templates**: Standardized VM deployment
- **Monitoring Integration**: Prometheus/Grafana for cluster health

### Architecture Evolution
- **Additional Nodes**: Plan for Nodes 3-4 expansion
- **Storage Tiers**: NVMe vs SSD optimization
- **Backup Strategy**: Multi-tier backup implementation
- **Disaster Recovery**: Site replication planning

## 📚 Reference Quick Links

### Network Troubleshooting
- **UniFi Controller**: Check VLAN/port configuration
- **UDM Pro**: Verify inter-VLAN routing rules
- **Proxmox Networking**: `/etc/network/interfaces` configuration
- **Gateway Testing**: Use ping tests in sequence

### Documentation Cross-References
- **Network Architecture**: `network_setup.md` - VLAN design
- **Primary Node**: `primary_node_setup.md` - Network bridge config
- **Cluster Setup**: `cluster_setup.md` - Multi-node networking
- **Progress Tracking**: `README.md` - Deployment status

---

*Document started during Phase 1 completion - January 2025*  
*Living document: Updated with each phase completion and significant learning*