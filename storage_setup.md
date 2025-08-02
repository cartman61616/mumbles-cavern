# Phase 3: Storage Configuration

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [Cluster Setup](03-cluster-setup.md)  
**Next Phase**: [Shared Infrastructure](05-shared-infrastructure.md)  
**Duration**: 3-4 hours  
**Prerequisites**: 2-node cluster operational

## Overview
Configure shared storage systems to enable live migration, high availability, and centralized data management across your Proxmox cluster.

## Step 1: ASUSTOR TrueNAS Configuration

### Access TrueNAS Web Interface
1. Connect to ASUSTOR via web UI (typically http://[asustor-ip]:8000)
2. Complete initial setup if not already done
3. Ensure storage pool is created and healthy

### Create Proxmox Storage Datasets
```bash
# Via TrueNAS web UI, create these datasets:
/mnt/pool/proxmox-storage     # VM disk images and containers
/mnt/pool/proxmox-backup      # VM backups and snapshots  
/mnt/pool/proxmox-iso         # ISO images and templates
/mnt/pool/media               # Media files for Plex
/mnt/pool/downloads           # Download staging area
```

### Configure NFS Shares for Proxmox
1. **Go to Sharing** → **Unix (NFS) Shares**
2. **Create share for VM storage:**
   ```
   Path: /mnt/pool/proxmox-storage
   Comment: Proxmox VM storage
   Authorized Networks: 192.168.1.0/24,192.168.10.0/24,192.168.30.0/24
   Options: rw,sync,no_root_squash,no_subtree_check
   ```

3. **Create share for backups:**
   ```
   Path: /mnt/pool/proxmox-backup  
   Comment: Proxmox backup storage
   Authorized Networks: 192.168.1.0/24,192.168.10.0/24
   Options: rw,sync,no_root_squash,no_subtree_check
   ```

4. **Create share for ISO images:**
   ```
   Path: /mnt/pool/proxmox-iso
   Comment: ISO images and templates
   Authorized Networks: 192.168.1.0/24,192.168.10.0/24
   Options: rw,sync,no_root_squash,no_subtree_check
   ```

5. **Create share for media:**
   ```
   Path: /mnt/pool/media
   Comment: Media files for streaming
   Authorized Networks: 192.168.1.0/24,192.168.20.0/24
   Options: rw,sync,no_root_squash,no_subtree_check
   ```

### Enable NFS Service
1. **Go to Services** → **NFS**
2. **Enable NFS service**
3. **Configure NFS settings:**
   ```
   NFS Protocol: v3, v4
   Enable NFSv4: Yes
   NFSv4 Domain: mumblescavern.local
   ```

### Test NFS Connectivity
```bash
# From both Proxmox nodes, test NFS mounting
# Node 1:
mkdir -p /mnt/test-nfs
mount -t nfs 192.168.10.20:/mnt/pool/proxmox-storage /mnt/test-nfs
ls -la /mnt/test-nfs
umount /mnt/test-nfs

# Node 2:
mkdir -p /mnt/test-nfs  
mount -t nfs 192.168.10.20:/mnt/pool/proxmox-storage /mnt/test-nfs
touch /mnt/test-nfs/node2-test.txt
ls -la /mnt/test-nfs
umount /mnt/test-nfs

# Verify Node 1 can see Node 2's test file
mount -t nfs 192.168.10.20:/mnt/pool/proxmox-storage /mnt/test-nfs
ls -la /mnt/test-nfs/node2-test.txt
rm /mnt/test-nfs/node2-test.txt
umount /mnt/test-nfs
```

## Step 2: Add Shared Storage to Proxmox Cluster

### Add NFS Storage via Web UI

#### VM Storage (Live Migration Capable)
1. **Datacenter** → **Storage** → **Add** → **NFS**
2. **Configuration:**
   ```
   ID: asustor-vm-storage
   Server: 192.168.10.20
   Export: /mnt/pool/proxmox-storage
   Content: Disk image, Container
   Max Backups: 3
   Shared: Yes
   Enable: Yes
   ```

#### Backup Storage
1. **Datacenter** → **Storage** → **Add** → **NFS**  
2. **Configuration:**
   ```
   ID: asustor-backup
   Server: 192.168.10.20
   Export: /mnt/pool/proxmox-backup
   Content: VZDump backup file
   Max Backups: 10
   Shared: Yes
   Enable: Yes
   ```

#### ISO Storage
1. **Datacenter** → **Storage** → **Add** → **NFS**
2. **Configuration:**
   ```
   ID: asustor-iso
   Server: 192.168.10.20  
   Export: /mnt/pool/proxmox-iso
   Content: ISO image, Container template
   Shared: Yes
   Enable: Yes
   ```

### Verify Storage Integration
```bash
# Check storage status
pvesm status

# Should show all storage pools including new NFS shares:
# local, local-lvm, asustor-vm-storage, asustor-backup, asustor-iso

# Test storage accessibility from both nodes
pvesm list asustor-vm-storage
pvesm list asustor-backup
```

## Step 3: Local Storage Optimization

### Configure Node-Specific High-Performance Storage

#### Node 1 & 2 (If dual storage available)
```bash
# If you have NVMe + SSD configuration:
# Use NVMe for high-performance VM storage
# Use SSD partition for local backups and cache

# Check current storage setup
lsblk
pvs
vgs
lvs

# If not already configured, create local high-performance storage
# (Only if you have separate NVMe drive)
pvecreate /dev/nvme0n1
vgcreate local-nvme /dev/nvme0n1
lvcreate -l 100%FREE -n vm-fast local-nvme

# Add via Web UI:
# Node → local storage → Add → LVM
# ID: local-nvme-fast
# Volume group: local-nvme
# Content: Disk image, Container
# Shared: No (node-specific fast storage)
```

#### Create Local Cache Storage
```bash
# Create cache directory for frequently accessed data
mkdir -p /var/cache/proxmox-vm
chmod 755 /var/cache/proxmox-vm

# Add cache storage via Web UI:
# Node → Storage → Add → Directory
# ID: local-cache
# Directory: /var/cache/proxmox-vm
# Content: Disk image (for templates and frequently used images)
# Shared: No
```

## Step 4: Configure Storage Network Optimization

### Dedicated Storage Network (VLAN 30)
```bash
# Configure storage-specific network interface
# This will be used later for high-bandwidth storage traffic

# Edit network configuration to prepare for storage VLAN
# For now, document the plan - we'll implement during VM deployment

# Storage Network Plan:
# Node 1 Storage IP: 192.168.30.10
# Node 2 Storage IP: 192.168.30.11  
# ASUSTOR Storage IP: 192.168.30.20
# Synology Storage IP: 192.168.30.21
```

### Optimize NFS Mount Options
```bash
# Create optimized NFS mount configuration
# Edit /etc/fstab to add performance-optimized NFS mounts
cat >> /etc/fstab << 'EOF'
# High-performance NFS mounts for Proxmox storage
192.168.10.20:/mnt/pool/proxmox-storage /mnt/pve/asustor-vm-storage nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2 0 0
192.168.10.20:/mnt/pool/proxmox-backup /mnt/pve/asustor-backup nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2 0 0
192.168.10.20:/mnt/pool/proxmox-iso /mnt/pve/asustor-iso nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2 0 0
EOF

# Test mount with optimized options
mount -a
df -h | grep asustor
```

## Step 5: Configure Backup Strategy

### Automated VM Backup Configuration
```bash
# Create comprehensive backup script
cat > /usr/local/bin/vm-backup-strategy.sh << 'EOF'
#!/bin/bash
# Comprehensive VM backup strategy

BACKUP_LOG="/var/log/vm-backup.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" >> "$BACKUP_LOG"
    echo "$1"
}

# Function to backup all VMs
backup_all_vms() {
    log_message "Starting automated VM backup"
    
    # Get list of all VMs
    VM_LIST=$(qm list | grep -v VMID | awk '{print $1}')
    
    for VMID in $VM_LIST; do
        if [ ! -z "$VMID" ]; then
            log_message "Backing up VM $VMID"
            vzdump $VMID \
                --storage asustor-backup \
                --mode snapshot \
                --compress gzip \
                --mailto your-email@domain.com \
                --quiet 1 \
                --remove 0
                
            if [ $? -eq 0 ]; then
                log_message "VM $VMID backup completed successfully"
            else
                log_message "ERROR: VM $VMID backup failed"
            fi
        fi
    done
    
    log_message "VM backup cycle completed"
}

# Function to cleanup old backups
cleanup_old_backups() {
    log_message "Starting backup cleanup"
    
    # Remove backups older than 14 days from NFS storage
    find /mnt/pve/asustor-backup -name "*.vma.gz" -mtime +14 -delete 2>/dev/null
    find /mnt/pve/asustor-backup -name "*.tar.gz" -mtime +14 -delete 2>/dev/null
    
    # Remove local backups older than 7 days
    find /backup-local -name "*.vma.gz" -mtime +7 -delete 2>/dev/null
    
    log_message "Backup cleanup completed"
}

# Execute based on argument
case "$1" in
    "backup")
        backup_all_vms
        ;;
    "cleanup")
        cleanup_old_backups
        ;;
    "full")
        backup_all_vms
        cleanup_old_backups
        ;;
    *)
        echo "Usage: $0 {backup|cleanup|full}"
        exit 1
        ;;
esac
EOF

chmod +x /usr/local/bin/vm-backup-strategy.sh

# Test backup script
/usr/local/bin/vm-backup-strategy.sh cleanup
```

### Schedule Automated Backups
```bash
# Create cron jobs for automated backups
cat > /tmp/backup-cron << 'EOF'
# VM Backup Schedule
# Full backup with cleanup every night at 1 AM
0 1 * * * /usr/local/bin/vm-backup-strategy.sh full

# Quick cleanup check every 6 hours
0 */6 * * * /usr/local/bin/vm-backup-strategy.sh cleanup

# Cluster health and backup log rotation
0 4 * * 0 /usr/bin/logrotate -f /etc/logrotate.d/vm-backup
EOF

# Install cron jobs
crontab /tmp/backup-cron
rm /tmp/backup-cron

# Create logrotate configuration for backup logs
cat > /etc/logrotate.d/vm-backup << 'EOF'
/var/log/vm-backup.log {
    weekly
    rotate 4
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
}
EOF
```

## Step 6: Configure Synology as Secondary Backup Target

### Synology NAS Setup
1. **Access Synology DSM** (typically http://[synology-ip]:5000)
2. **Create shared folder:** "proxmox-backup-secondary"
3. **Enable NFS:** Control Panel → File Services → NFS → Enable NFS
4. **Configure NFS permissions:**
   ```
   Shared Folder: proxmox-backup-secondary
   Authorized Networks: 192.168.1.0/24,192.168.10.0/24
   Privilege: Read/Write
   Squash: Map root to admin
   ```

### Add Synology to Proxmox
```bash
# Add Synology as secondary backup storage
# Web UI: Datacenter → Storage → Add → NFS
# ID: synology-backup-secondary
# Server: 192.168.10.21
# Export: /volume1/proxmox-backup-secondary
# Content: VZDump backup file
# Max Backups: 5
# Shared: Yes
```

### Configure Offsite Backup Sync
```bash
# Create script to sync critical backups to Synology
cat > /usr/local/bin/offsite-backup-sync.sh << 'EOF'
#!/bin/bash
# Sync critical backups to secondary location

SOURCE_DIR="/mnt/pve/asustor-backup"
DEST_DIR="/mnt/pve/synology-backup-secondary"
LOG_FILE="/var/log/offsite-backup.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Sync only recent backups (last 3 days) to secondary storage
rsync -av --delete \
    --include="*.vma.gz" \
    --include="*.tar.gz" \
    --exclude="*" \
    --newer-than="3 days ago" \
    "$SOURCE_DIR/" "$DEST_DIR/"

echo "$DATE: Offsite backup sync completed" >> "$LOG_FILE"

# Keep log manageable
tail -100 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x /usr/local/bin/offsite-backup-sync.sh

# Schedule offsite sync every 6 hours
echo "0 */6 * * * /usr/local/bin/offsite-backup-sync.sh" | crontab -l | { cat; echo "0 */6 * * * /usr/local/bin/offsite-backup-sync.sh"; } | crontab -
```

## Step 7: Storage Performance Testing

### Network Storage Performance Test
```bash
# Test NFS performance from both nodes
# Install performance testing tools
apt install -y fio iperf3

# Test NFS write performance
fio --name=nfs-write-test \
    --filename=/mnt/pve/asustor-vm-storage/fio-test \
    --size=1G \
    --bs=4k \
    --rw=write \
    --direct=1 \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting

# Test NFS read performance  
fio --name=nfs-read-test \
    --filename=/mnt/pve/asustor-vm-storage/fio-test \
    --size=1G \
    --bs=4k \
    --rw=read \
    --direct=1 \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting

# Test mixed workload (simulates VM disk activity)
fio --name=nfs-mixed-test \
    --filename=/mnt/pve/asustor-vm-storage/fio-mixed-test \
    --size=2G \
    --bs=4k \
    --rw=randrw \
    --rwmixread=70 \
    --direct=1 \
    --numjobs=2 \
    --runtime=120 \
    --group_reporting

# Clean up test files
rm /mnt/pve/asustor-vm-storage/fio-*test*
```

### Local Storage Performance Baseline
```bash
# Test local NVMe performance (if available)
# This gives you a comparison point for shared vs local storage

# Local NVMe write test
fio --name=local-nvme-write \
    --filename=/tmp/fio-local-test \
    --size=1G \
    --bs=4k \
    --rw=write \
    --direct=1 \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting

# Local NVMe read test
fio --name=local-nvme-read \
    --filename=/tmp/fio-local-test \
    --size=1G \
    --bs=4k \
    --rw=read \
    --direct=1 \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting

rm /tmp/fio-local-test
```

## Step 8: Configure Storage-Based High Availability

### Enable HA Prerequisites
```bash
# Verify shared storage is accessible from both nodes
pvesm status | grep asustor

# Check cluster quorum
pvecm status | grep -i quorum

# Both nodes should show shared storage as available
# Quorum should show 2/2 nodes
```

### Create HA Resource Groups
```bash
# Via Web UI: Datacenter → HA → Groups → Add
# Group 1: critical-services
# - Nodes: pve-node1:pve-node2  
# - Restricted: No
# - No Failback: No
# - Comment: Critical homelab services

# Group 2: media-services  
# - Nodes: pve-node1:pve-node2
# - Restricted: No
# - No Failback: Yes (prefer specific node for media transcoding)
# - Comment: Media and entertainment services
```

## Step 9: Storage Monitoring and Alerting

### Install Storage Monitoring Tools
```bash
# Install NFS monitoring tools
apt install -y nfs-common rpcbind

# Create storage health monitoring script
cat > /usr/local/bin/storage-health-check.sh << 'EOF'
#!/bin/bash
# Storage health monitoring

LOG_FILE="/var/log/storage-health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" >> "$LOG_FILE"
}

# Check NFS mount points
check_nfs_mounts() {
    FAILED_MOUNTS=""
    
    for MOUNT in /mnt/pve/asustor-vm-storage /mnt/pve/asustor-backup /mnt/pve/asustor-iso; do
        if ! mountpoint -q "$MOUNT"; then
            FAILED_MOUNTS="$FAILED_MOUNTS $MOUNT"
        fi
    done
    
    if [ ! -z "$FAILED_MOUNTS" ]; then
        log_message "ERROR: Failed NFS mounts:$FAILED_MOUNTS"
        return 1
    else
        log_message "All NFS mounts healthy"
        return 0
    fi
}

# Check storage space
check_storage_space() {
    # Check shared storage space
    ASUSTOR_USAGE=$(df -h /mnt/pve/asustor-vm-storage | tail -1 | awk '{print $5}' | sed 's/%//')
    
    if [ "$ASUSTOR_USAGE" -gt 85 ]; then
        log_message "WARNING: ASUSTOR storage usage at ${ASUSTOR_USAGE}%"
    else
        log_message "ASUSTOR storage usage: ${ASUSTOR_USAGE}%"
    fi
    
    # Check local storage
    LOCAL_USAGE=$(df -h / | tail -1 | awk '{print $5}' | sed 's/%//')
    if [ "$LOCAL_USAGE" -gt 80 ]; then
        log_message "WARNING: Local storage usage at ${LOCAL_USAGE}%"
    else
        log_message "Local storage usage: ${LOCAL_USAGE}%"
    fi
}

# Check storage performance
check_storage_performance() {
    # Simple write test to shared storage
    TEST_FILE="/mnt/pve/asustor-vm-storage/.storage-test-$(hostname)"
    
    START_TIME=$(date +%s.%N)
    dd if=/dev/zero of="$TEST_FILE" bs=1M count=100 oflag=direct 2>/dev/null
    END_TIME=$(date +%s.%N)
    
    WRITE_TIME=$(echo "$END_TIME - $START_TIME" | bc -l)
    WRITE_SPEED=$(echo "scale=2; 100 / $WRITE_TIME" | bc -l)
    
    rm -f "$TEST_FILE"
    
    log_message "Storage write speed: ${WRITE_SPEED} MB/s"
    
    # Alert if performance is poor (less than 50MB/s)
    if (( $(echo "$WRITE_SPEED < 50" | bc -l) )); then
        log_message "WARNING: Storage performance degraded (${WRITE_SPEED} MB/s)"
    fi
}

# Run all checks
check_nfs_mounts
check_storage_space  
check_storage_performance

# Keep log size manageable
tail -200 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x /usr/local/bin/storage-health-check.sh

# Schedule storage health checks every 15 minutes
echo "*/15 * * * * /usr/local/bin/storage-health-check.sh" | crontab -l | { cat; echo "*/15 * * * * /usr/local/bin/storage-health-check.sh"; } | crontab -

# Test the storage health check
/usr/local/bin/storage-health-check.sh
cat /var/log/storage-health.log
```

## Step 10: Test Live Migration Capability

### Create Test VM for Migration
```bash
# Create small test VM to verify migration works
qm create 999 \
    --name migration-test \
    --memory 1024 \
    --cores 1 \
    --net0 virtio,bridge=vmbr0 \
    --storage asustor-vm-storage

# Create small disk for test
qm set 999 --scsi0 asustor-vm-storage:8

# Start test VM
qm start 999

# Check VM status
qm status 999
```

### Test Online Migration
```bash
# Migrate test VM from Node 1 to Node 2
# Via Web UI: Select VM → More → Migrate
# Or via command line:
qm migrate 999 pve-node2 --online

# Check migration status
qm status 999

# Should now show running on pve-node2
# Test migrating back
qm migrate 999 pve-node1 --online

# Clean up test VM
qm stop 999
qm destroy 999
```

## Validation and Testing

### Storage Integration Validation
- [ ] **NFS shares mounted** on both nodes
- [ ] **Shared storage visible** in web UI from both nodes  
- [ ] **VM creation possible** on shared storage
- [ ] **Backup storage configured** and accessible
- [ ] **ISO storage available** for templates
- [ ] **Live migration working** between nodes
- [ ] **Storage monitoring** operational

### Performance Validation
```bash
# Expected performance targets:
# NFS Sequential Read: >100 MB/s
# NFS Sequential Write: >80 MB/s  
# NFS Random Read IOPS: >1000
# NFS Random Write IOPS: >500
# Local NVMe (if available): >500 MB/s sequential

# Network utilization during storage operations should not exceed 80%
```

### Backup System Validation
- [ ] **Automated backups** scheduled and tested
- [ ] **Backup retention** policies working
- [ ] **Secondary backup target** (Synology) configured
- [ ] **Backup monitoring** logging properly
- [ ] **Storage health checks** running every 15 minutes

## Troubleshooting Common Issues

### NFS Mount Failures
```bash
# Check NFS service on ASUSTOR
showmount -e 192.168.10.20

# Check network connectivity to storage VLAN
ping 192.168.10.20

# Check NFS mount options
mount | grep nfs

# Remount with debug options
mount -t nfs -o rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2,_netdev 192.168.10.20:/mnt/pool/proxmox-storage /mnt/pve/asustor-vm-storage
```

### Storage Performance Issues
```bash
# Check network utilization
iftop -i eno1

# Check NFS server performance
nfsstat -c  # Client statistics
nfsstat -s  # Server statistics (run on ASUSTOR)

# Check for network errors
cat /proc/net/dev | grep eno1

# Optimize mount options if needed
umount /mnt/pve/asustor-vm-storage
mount -t nfs -o rsize=262144,wsize=262144,hard,intr,timeo=20,retrans=3 192.168.10.20:/mnt/pool/proxmox-storage /mnt/pve/asustor-vm-storage
```

### Live Migration Failures
```bash
# Check shared storage accessibility
pvesm list asustor-vm-storage

# Check SSH connectivity between nodes
ssh root@192.168.1.11 "echo migration-test"

# Check VM disk location
qm config [VMID] | grep scsi0

# Ensure VM is using shared storage, not local storage
```

## Advanced Storage Configuration

### Storage Replication Setup (Optional)
```bash
# Configure storage replication between nodes
# This provides additional redundancy for critical VMs

# Via Web UI: Datacenter → Replication → Add
# Target: pve-node2 (if configuring from pve-node1)
# Schedule: */15 (every 15 minutes)
# Rate limit: 10 (MB/s limit to not saturate network)
```

### Storage Snapshot Management
```bash
# Create snapshot management script
cat > /usr/local/bin/snapshot-management.sh << 'EOF'
#!/bin/bash
# Automated snapshot management

LOG_FILE="/var/log/snapshot-management.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Create daily snapshots for important VMs
create_daily_snapshots() {
    for VMID in $(qm list | grep running | awk '{print $1}'); do
        SNAPSHOT_NAME="auto-daily-$(date +%Y%m%d)"
        
        echo "$DATE: Creating snapshot $SNAPSHOT_NAME for VM $VMID" >> "$LOG_FILE"
        qm snapshot "$VMID" "$SNAPSHOT_NAME" --description "Automated daily snapshot"
    done
}

# Clean up old snapshots (keep 7 days)
cleanup_old_snapshots() {
    CUTOFF_DATE=$(date -d '7 days ago' +%Y%m%d)
    
    for VMID in $(qm list | awk 'NR>1 {print $1}'); do
        # List snapshots and remove old ones
        qm listsnapshot "$VMID" | grep "auto-daily-" | while read snapshot_line; do
            SNAPSHOT_NAME=$(echo "$snapshot_line" | awk '{print $2}')
            SNAPSHOT_DATE=$(echo "$SNAPSHOT_NAME" | grep -o '[0-9]\{8\}')
            
            if [ "$SNAPSHOT_DATE" -lt "$CUTOFF_DATE" ]; then
                echo "$DATE: Removing old snapshot $SNAPSHOT_NAME from VM $VMID" >> "$LOG_FILE"
                qm delsnapshot "$VMID" "$SNAPSHOT_NAME"
            fi
        done
    done
}

case "$1" in
    "create")
        create_daily_snapshots
        ;;
    "cleanup")
        cleanup_old_snapshots
        ;;
    "full")
        create_daily_snapshots
        cleanup_old_snapshots
        ;;
    *)
        echo "Usage: $0 {create|cleanup|full}"
        exit 1
        ;;
esac
EOF

chmod +x /usr/local/bin/snapshot-management.sh

# Schedule daily snapshots at 23:30
echo "30 23 * * * /usr/local/bin/snapshot-management.sh full" | crontab -l | { cat; echo "30 23 * * * /usr/local/bin/snapshot-management.sh full"; } | crontab -
```

## Step 11: Storage Security Configuration

### Configure NFS Security
```bash
# Create NFS security policy
# Limit NFS access to specific networks and protocols

# Update NFS exports on ASUSTOR with security options:
# Add these options to existing NFS shares:
# - no_root_squash (already configured)
# - sync (already configured)  
# - secure (require privileged ports)
# - subtree_check (verify subdirectory access)

# Test secure NFS mounting
umount /mnt/pve/asustor-vm-storage
mount -t nfs -o rsize=131072,wsize=131072,hard,intr,secure 192.168.10.20:/mnt/pool/proxmox-storage /mnt/pve/asustor-vm-storage
```

### Backup Encryption Configuration
```bash
# Configure encrypted backups for sensitive VMs
# This will be used when creating backup jobs via Web UI

# Create encryption key for backups
openssl rand -base64 32 > /etc/pve/backup-encryption.key
chmod 600 /etc/pve/backup-encryption.key

# Document encryption key location in password manager
echo "Backup encryption key stored in: /etc/pve/backup-encryption.key"
```

## Step 12: Storage Disaster Recovery Planning

### Create Storage Disaster Recovery Documentation
```bash
# Document storage configuration for disaster recovery
cat > /etc/pve/storage-dr-info.txt << 'EOF'
# Storage Disaster Recovery Information
# Generated: $(date)

## NFS Storage Configuration
Primary NFS Server: 192.168.10.20 (ASUSTOR)
Secondary Backup: 192.168.10.21 (Synology)

## Mount Points and Exports
/mnt/pve/asustor-vm-storage → 192.168.10.20:/mnt/pool/proxmox-storage
/mnt/pve/asustor-backup → 192.168.10.20:/mnt/pool/proxmox-backup
/mnt/pve/asustor-iso → 192.168.10.20:/mnt/pool/proxmox-iso
/mnt/pve/synology-backup-secondary → 192.168.10.21:/volume1/proxmox-backup-secondary

## Recovery Procedures
1. If ASUSTOR fails: Restore latest backups from Synology to replacement NAS
2. If network storage unavailable: VMs can run from local storage temporarily
3. If single node fails: Cluster continues on remaining node with shared storage

## Critical Files for DR
- Cluster configuration: /etc/pve/ (automatically replicated)
- Backup encryption key: /etc/pve/backup-encryption.key
- Storage health logs: /var/log/storage-health.log
- VM configurations: /etc/pve/qemu-server/ (automatically replicated)
EOF
```

### Test Disaster Recovery Scenarios
```bash
# Test 1: Simulate NFS server failure
# Temporarily stop NFS on ASUSTOR and verify:
# - Existing VMs continue running from local cache
# - New VM creation gracefully fails with clear error
# - Cluster remains operational

# Test 2: Simulate single node failure  
# Shutdown Node 2 and verify:
# - Cluster maintains quorum (with 2 nodes, losing 1 breaks quorum)
# - VMs can be manually started on remaining node
# - Shared storage remains accessible

# Test 3: Storage failover
# Switch VM storage from ASUSTOR to Synology
# Verify backup restore procedures work
```

## Documentation and Monitoring Setup

### Storage Performance Baseline Documentation
```bash
# Create storage performance baseline document
cat > /etc/pve/storage-baseline.txt << 'EOF'
# Storage Performance Baseline
# Measured: $(date)

## NFS Performance (ASUSTOR)
Sequential Read: [Record your fio test results]
Sequential Write: [Record your fio test results]  
Random Read IOPS: [Record your fio test results]
Random Write IOPS: [Record your fio test results]

## Local Storage Performance (if available)
Local NVMe Sequential Read: [Record results]
Local NVMe Sequential Write: [Record results]

## Network Performance
Node-to-Node: [iperf3 results] Gbps
Node-to-Storage: [iperf3 results] Gbps

## Utilization Thresholds
Storage Alert: >85% full
Performance Alert: <50 MB/s sustained
Network Alert: >80% utilization
EOF
```

### Configure Storage Alerts
```bash
# Create storage alert script
cat > /usr/local/bin/storage-alerts.sh << 'EOF'
#!/bin/bash
# Storage alerting system

ALERT_EMAIL="your-email@domain.com"
LOG_FILE="/var/log/storage-alerts.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

send_alert() {
    local SUBJECT="$1"
    local MESSAGE="$2"
    
    echo "$DATE: ALERT - $SUBJECT" >> "$LOG_FILE"
    echo "$MESSAGE" >> "$LOG_FILE"
    
    # Send email if mail is configured
    if command -v mail >/dev/null 2>&1; then
        echo "$MESSAGE" | mail -s "Proxmox Storage Alert: $SUBJECT" "$ALERT_EMAIL"
    fi
}

# Check for storage issues and send alerts
if ! /usr/local/bin/storage-health-check.sh; then
    send_alert "Storage Health Check Failed" "One or more storage health checks failed. Check /var/log/storage-health.log for details."
fi
EOF

chmod +x /usr/local/bin/storage-alerts.sh

# Schedule alert checks every hour
echo "0 * * * * /usr/local/bin/storage-alerts.sh" | crontab -l | { cat; echo "0 * * * * /usr/local/bin/storage-alerts.sh"; } | crontab -
```

## Completion Criteria
- [ ] **NFS shares** configured and mounted on both nodes
- [ ] **Shared VM storage** available from both nodes  
- [ ] **Backup storage** configured with retention policies
- [ ] **Live migration** tested and working
- [ ] **Storage monitoring** scripts operational
- [ ] **Performance baseline** established and documented
- [ ] **Disaster recovery** procedures documented
- [ ] **Storage health checks** running automatically
- [ ] **Backup automation** scheduled and tested
- [ ] **Secondary backup target** configured

### Pre-Next Phase Checklist
- [ ] Storage performance meets expectations (>80 MB/s NFS)
- [ ] Live migration working reliably
- [ ] Backup jobs completing successfully
- [ ] Storage monitoring showing healthy status
- [ ] Both nodes can access all shared storage
- [ ] Storage space utilization <50% (room for growth)

**Next Phase**: Proceed to [Shared Infrastructure](05-shared-infrastructure.md) to deploy foundational services like PostgreSQL and Redis that will support your applications.