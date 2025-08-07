# Device Naming Update Timing Strategy
**When to implement Snorlax-themed naming across infrastructure**

## Current State Analysis

### What We Have Now
```yaml
Proxmox Cluster (2 nodes):
  - pve-node1 → should become drowzee.mumblescavern.local
  - pve-node2 → should become sleepy.mumblescavern.local
  - Current cluster name: likely "mumbles-cluster" or generic

Network Infrastructure:
  - UniFi Dream Machine Pro (likely needs hostname update)
  - UniFi Flex Mini switch (likely needs hostname update)
  - ASUSTOR NAS (needs hostname update)
  - Synology NAS (needs hostname update)

Gaming/Workstation:
  - Gaming PC (needs to become mighty-snorlax.mumblescavern.local)
  - Current homelab server (needs to become snorlax-prime.mumblescavern.local)
```

## Optimal Timing Options

### Option A: NOW (Before Plex Deployment) ⭐ **RECOMMENDED**
**Timing**: Before deploying any services on the cluster

**Pros**:
- ✅ **Clean slate**: No services to migrate or reconfigure
- ✅ **Zero service disruption**: Nothing running yet to break
- ✅ **Proper foundation**: All new services deployed with correct names
- ✅ **No VM migrations needed**: Can rename nodes without moving VMs
- ✅ **Documentation alignment**: All guides can reference correct names immediately

**Cons**:
- ⚠️ **Brief cluster downtime**: ~10-15 minutes per node during rename
- ⚠️ **Potential re-clustering**: May need to rebuild cluster with new names

**Implementation**:
```bash
# 1. Backup cluster configuration
# 2. Rename nodes one at a time
# 3. Update DNS/DHCP reservations
# 4. Deploy services with correct naming
```

### Option B: During TrueNAS Reset (Parallel Work)
**Timing**: While TrueNAS is being reset and storage reconfigured

**Pros**:
- ✅ **Parallel efficiency**: Work on naming while storage is offline anyway
- ✅ **Natural maintenance window**: Expected downtime period
- ✅ **Clean infrastructure restart**: Everything gets proper naming together

**Cons**:
- ⚠️ **Multiple moving parts**: More complexity during maintenance window
- ⚠️ **Extended downtime**: Both storage AND compute affected

### Option C: After Basic Services (NOT RECOMMENDED)
**Timing**: After Plex and essential services are running

**Pros**:
- ✅ **Services remain available**: Less immediate disruption

**Cons**:
- ❌ **Complex migration**: Need to migrate VMs during rename
- ❌ **Service reconfiguration**: Plex, databases need hostname updates  
- ❌ **DNS propagation issues**: Services may become temporarily unreachable
- ❌ **Documentation confusion**: Guides reference old names while new ones exist
- ❌ **Certificate issues**: SSL certs may need regeneration

## Recommendation: Option A - Rename NOW

### Why Now is Best
1. **2-node cluster is minimal**: Easy to rename/rebuild
2. **No services deployed yet**: Nothing to break or migrate
3. **Before adding nodes 3&4**: They can join with correct names immediately
4. **Clean foundation**: All future work uses proper naming

### Implementation Plan (2-3 hours total)

#### Phase 1: Prepare for Rename (30 minutes)
```bash
# Backup current cluster config
pvecm status > cluster-backup-$(date +%Y%m%d).txt
cat /etc/hosts > hosts-backup-$(date +%Y%m%d).txt
cat /etc/hostname > hostname-backup-$(date +%Y%m%d).txt

# Document current IPs and settings
ip addr show > ip-config-backup-$(date +%Y%m%d).txt
```

#### Phase 2: Update Network Infrastructure (30 minutes)
```bash
# UniFi Configuration:
1. Update DNS entries:
   - 192.168.10.10 → drowzee.mumblescavern.local
   - 192.168.10.11 → sleepy.mumblescavern.local
   - 192.168.10.20 → asustor-nas.mumblescavern.local
   - 192.168.10.21 → synology-backup.mumblescavern.local

2. Update DHCP reservations with new hostnames
3. Update UniFi device hostnames in controller
```

#### Phase 3: Rename Node 1 (drowzee) (45 minutes)
```bash
# SSH to pve-node1
ssh root@192.168.10.10

# Stop cluster services temporarily
systemctl stop pve-cluster
systemctl stop corosync

# Update hostname
hostnamectl set-hostname drowzee.mumblescavern.local
echo "drowzee.mumblescavern.local" > /etc/hostname

# Update hosts file
cat > /etc/hosts << 'EOF'
127.0.0.1 localhost.localdomain localhost
192.168.10.10 drowzee.mumblescavern.local drowzee pvelocalhost
192.168.10.11 sleepy.mumblescavern.local sleepy

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF

# Update Proxmox configuration
# Edit /etc/pve/corosync.conf (if accessible)
# Update node name references

# Restart services
systemctl start corosync
systemctl start pve-cluster

# Verify
hostname -f
systemctl status pve-cluster
```

#### Phase 4: Rename Node 2 (sleepy) (45 minutes)
```bash
# Repeat similar process for pve-node2 → sleepy.mumblescavern.local
```

#### Phase 5: Rebuild Cluster if Needed (30 minutes)
```bash
# If cluster becomes unstable, rebuild:
# 1. Leave cluster from node 2
# 2. Recreate cluster on node 1 with new name
# 3. Join node 2 to cluster with new names

# New cluster creation (if needed):
pvecm create mumbles-cavern

# From sleepy node:
pvecm add 192.168.10.10
```

#### Phase 6: Update Documentation and SSH configs (15 minutes)
```bash
# Update SSH config files
# Update documentation references
# Test connectivity with new names
```

### Alternative: Quick Rebuild Approach (Faster)
If renaming proves complex, consider:

```bash
# 1. Fresh install Proxmox on both nodes with correct names
# 2. Create new cluster with proper naming
# 3. Import any essential VMs from backups if they exist

# This might be faster than in-place rename and ensures clean start
```

## Post-Rename Benefits

### Immediate Benefits
- ✅ All new services deploy with correct names
- ✅ Documentation matches reality
- ✅ SSH configs use proper hostnames
- ✅ Monitoring uses meaningful names

### Long-term Benefits  
- ✅ No future migration needed
- ✅ Nodes 3&4 join with proper names
- ✅ Certificate generation uses correct names
- ✅ Professional presentation for visitors

## Risk Mitigation

### Backup Plan
```bash
# If rename fails badly:
1. Use backup configs to restore original names
2. Fresh reinstall is always option (only ~2 hours)
3. No services are lost (none deployed yet)
```

### Testing Approach
```bash
# After rename, verify:
1. Both nodes communicate: pvecm status
2. Web UI accessible on both nodes
3. SSH works with new names
4. DNS resolution working
5. Can create/start test VM
```

## Final Recommendation

**Rename RIGHT NOW** before deploying any services. It's the cleanest, safest approach with minimal complexity and maximum long-term benefit. The cluster is young, no services to migrate, and it sets the proper foundation for everything that follows.

Would you like me to help execute the rename process?