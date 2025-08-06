# Phase 6: Proxmox 9.0 Upgrade

## Overview
This phase upgrades your existing Proxmox VE 8.x cluster to the latest Proxmox VE 9.0, which includes enhanced features like snapshots for thick-provisioned LVM storage, improved HA rules, and enhanced Software-Defined Networking (SDN) capabilities.

## Prerequisites
- [ ] Proxmox VE 8.4.1 or newer installed on all nodes
- [ ] All VMs and containers backed up and tested
- [ ] Cluster in healthy state with all nodes online
- [ ] External storage plugins compatible with Proxmox VE 9.0 (if used)
- [ ] Console or SSH access to all nodes

## Estimated Duration
**Total Time**: 2-3 hours for a 2-node cluster
**Per Node**: 45-60 minutes of active upgrade time
**Downtime**: Minimal for VMs if migrated; 10-15 minutes per node

## Hardware Requirements
- **RAM**: Adequate for running existing VMs during upgrade
- **Storage**: At least 2GB free space on each node
- **Network**: Stable connection for package downloads

## Pre-Upgrade Validation

### Step 1: Run Upgrade Checklist
Check for potential compatibility issues:

```bash
# Download and run the upgrade checklist script
wget https://enterprise.proxmox.com/pve8to9 -O pve8to9
chmod +x pve8to9
./pve8to9
```

Review all warnings and address any critical issues before proceeding.

### Step 2: Validate Container Compatibility
Check for containers that may not be compatible with cgroupv2:

```bash
# List all containers and their OS versions
for vmid in $(pct list | awk 'NR>1 {print $1}'); do
    echo "=== CT $vmid ==="
    pct exec $vmid -- cat /etc/os-release 2>/dev/null || echo "Could not read OS info"
done
```

**Important**: Containers running systemd 230 or older (e.g., CentOS 7, Ubuntu 16.04) will NOT work with Proxmox VE 9.0.

### Step 3: Backup Verification
Ensure all critical data is backed up:

```bash
# Create fresh backup of all VMs/containers
vzdump --all --compress gzip --storage local

# Verify backup integrity
ls -la /var/lib/vz/dump/
```

### Step 4: LVM Storage Preparation
If using shared LVM storage, run the migration script:

```bash
/usr/share/pve-manager/migrations/pve-lvm-disable-autoactivation
```

## Upgrade Process

### Step 1: Prepare Cluster for Upgrade
On all nodes, start with the primary node:

```bash
# Start a screen session for upgrade safety
screen -S proxmox-upgrade

# Update current repositories
apt update && apt full-upgrade -y

# Reboot if kernel was updated
reboot
```

### Step 2: Update Repository Sources
Update from Debian Bookworm to Trixie:

```bash
# Update main sources.list
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list

# Update Proxmox enterprise repository (if using)
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list.d/pve-enterprise.list

# Update Proxmox no-subscription repository (if using)
sed -i 's/bookworm/trixie/g' /etc/apt/sources.list.d/pve-no-subscription.list
```

### Step 3: Update Package Lists
```bash
# Update package information
apt update

# Check for any repository issues
apt-get check
```

### Step 4: Perform In-Place Upgrade
```bash
# Start the distribution upgrade
apt full-upgrade -y

# This process may take 20-30 minutes per node
# Monitor for any prompts requiring user input
```

### Step 5: Upgrade Proxmox Packages
```bash
# Upgrade to Proxmox VE 9.0 packages
apt install proxmox-ve=9.0* -y

# Reboot to complete the upgrade
reboot
```

## Post-Upgrade Validation

### Step 1: Verify System Status
```bash
# Check Proxmox version
pveversion -v

# Verify cluster status
pvecm status

# Check all nodes are online
pvecm nodes
```

### Step 2: Validate Services
```bash
# Check critical services
systemctl status pveproxy
systemctl status pvedaemon
systemctl status pve-cluster
systemctl status corosync

# Verify web interface accessibility
curl -k https://localhost:8006
```

### Step 3: Test VM Operations
```bash
# Test VM start/stop operations
qm start 100  # Replace with test VM ID
qm stop 100
qm start 100

# Test container operations
pct start 101  # Replace with test container ID
pct stop 101
pct start 101
```

### Step 4: Verify Storage Access
```bash
# Check storage status
pvesm status

# Test storage operations
pvesm list local
```

## New Features Configuration

### Step 1: Dark Mode Default
The web interface now defaults to dark mode. No configuration needed.

### Step 2: Enhanced HA Rules (if using HA)
Configure node and resource affinity for HA services:

```bash
# View current HA configuration
ha-manager config

# Configure node affinity (example)
ha-manager add vm:100 --group production-nodes
```

### Step 3: Snapshot Support for LVM
Test new snapshot functionality for thick-provisioned LVM:

```bash
# Create test snapshot (if using shared LVM storage)
qm snapshot 100 test-snapshot-v9
qm listsnapshot 100
```

## Cluster-Wide Upgrade

### For Multi-Node Clusters
Upgrade nodes one at a time:

1. **Migrate VMs** from the node to be upgraded
2. **Upgrade the node** following steps above
3. **Validate the node** functionality
4. **Migrate VMs back** if desired
5. **Repeat** for next node

```bash
# Migrate all VMs from current node to other nodes
for vmid in $(qm list | awk 'NR>1 {print $1}'); do
    qm migrate $vmid <target-node>
done

# Wait for migrations to complete before upgrading
```

## Troubleshooting

### Common Issues

#### Repository Errors
```bash
# If enterprise repository causes issues and you don't have subscription
rm /etc/apt/sources.list.d/pve-enterprise.list

# Use no-subscription repository
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
```

#### Container Boot Issues
If containers fail to start after upgrade:
```bash
# Check container configuration
pct config <vmid>

# Try starting with debugging
pct start <vmid> --debug

# If cgroupv1 issues, consider container OS upgrade
```

#### Storage Mount Issues
```bash
# Remount storage if needed
umount /mnt/pve/<storage-name>
mount /mnt/pve/<storage-name>

# Refresh storage configuration
pvesm set <storage-id>
```

## Rollback Procedure

### If Upgrade Fails
1. **Boot from rescue media** or previous kernel
2. **Restore from backup** if system is unrecoverable
3. **Revert repository sources**:
   ```bash
   sed -i 's/trixie/bookworm/g' /etc/apt/sources.list
   sed -i 's/trixie/bookworm/g' /etc/apt/sources.list.d/pve-*.list
   apt update
   apt full-upgrade
   ```

### Emergency Recovery
If cluster becomes unstable:
```bash
# Stop cluster services
systemctl stop pve-cluster

# Start in local mode
pmxcfs -l

# Restore from backup and rebuild cluster
```

## Validation Checklist

### System Level
- [ ] Proxmox version shows 9.0.x
- [ ] All cluster nodes online and communicating
- [ ] Web interface accessible on all nodes
- [ ] SSH access working to all nodes

### Storage Level
- [ ] All storage pools accessible
- [ ] VM disk operations working
- [ ] Backup/restore functionality tested
- [ ] Snapshot operations working (for supported storage)

### VM/Container Level
- [ ] All VMs can start/stop successfully
- [ ] All containers can start/stop successfully
- [ ] VM migration between nodes working
- [ ] Console access working for all VMs

### Network Level
- [ ] Inter-node communication stable
- [ ] VM networking operational
- [ ] Corosync cluster communication healthy
- [ ] External connectivity maintained

## Performance Optimization

### Post-Upgrade Tuning
```bash
# Clear package cache
apt autoremove -y
apt autoclean

# Update VM machine types if desired
# (Review each VM individually)
qm config <vmid> | grep machine
```

### New Feature Adoption
- Review new SDN fabric options for complex networking
- Configure HA affinity rules for production workloads  
- Test new LVM snapshot functionality for backups

## Next Steps

Upon successful completion:
- [ ] All nodes upgraded to Proxmox VE 9.0
- [ ] Cluster stability validated
- [ ] New features tested and configured
- [ ] Documentation updated with new version

**Continue to**: [Phase 7: Monitoring & Management](07-monitoring.md)

## Support Resources
- **Proxmox VE 9.0 Documentation**: https://pve.proxmox.com/pve-docs/
- **Upgrade Guide**: https://pve.proxmox.com/wiki/Upgrade_from_8_to_9
- **Forum Support**: https://forum.proxmox.com/
- **Release Notes**: https://www.proxmox.com/en/about/company-details/press-releases/