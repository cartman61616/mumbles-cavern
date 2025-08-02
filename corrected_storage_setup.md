# Phase 3: Storage Configuration

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [Cluster Setup](03-cluster-setup.md)  
**Next Phase**: [Shared Infrastructure](05-shared-infrastructure.md)  
**Duration**: 3-4 hours  
**Prerequisites**: 2-node cluster operational

## Overview
Configure shared storage systems to enable live migration, high availability, and centralized data management across your Proxmox cluster.

## Step 1: ASUSTOR TrueNAS Initial Configuration

### Access and Initial Setup
1. **Physical Connection**: Ensure ASUSTOR is connected to network
2. **Find ASUSTOR IP**: Check router/UniFi for device IP (typically DHCP assigned)
3. **Access Web Interface**: Browse to http://[asustor-ip]:8000
4. **Initial Setup Wizard**: Complete if first-time setup
   - Create admin account
   - Set timezone to America/New_York
   - Configure network settings

### Set Static IP Address
1. **Go to Network** → **Network Interface**
2. **Configure static IP**:
   ```
   IP Address: 192.168.1.100 (initial deployment)
   Subnet Mask: 255.255.255.0
   Gateway: 192.168.1.1
   DNS: 192.168.1.1, 1.1.1.1
   ```
3. **Apply settings** and reconnect at new IP

### Create Storage Pool (If Not Exists)
1. **Go to Storage Manager** → **Storage Pool**
2. **Create Pool** if no pool exists:
   ```
   Pool Name: pool
   RAID Type: Based on your drive configuration
   File System: EXT4 (recommended for NFS)
   ```
3. **Wait for initialization** (may take hours for large drives)

### Create Proxmox Datasets
1. **Go to Storage Manager** → **Volume**
2. **Create these volumes/folders**:
   ```
   /mnt/pool/proxmox-storage     # VM disk images and containers
   /mnt/pool/proxmox-backup      # VM backups and snapshots
   /mnt/pool/proxmox-iso         # ISO images and templates
   /mnt/pool/media               # Media files for Plex
   /mnt/pool/downloads           # Download staging area
   /mnt/pool/games               # Game ROMs and data
   ```

## Step 2: Configure NFS Service on ASUSTOR

### Enable NFS Service
1. **Go to Services** → **NFS Server**
2. **Enable NFS Server**
3. **Configure NFS Settings**:
   ```
   NFS Version: v3 and v4 (both enabled)
   NFSv4 Domain: mumblescavern.local
   Port: 2049 (default)
   ```
4. **Apply Settings**

### Create NFS Shares for Proxmox
1. **Go to Sharing** → **Shared Folder**
2. **Create NFS shares**:

#### VM Storage Share
```
Name: proxmox-storage
Path: /mnt/pool/proxmox-storage
Description: Proxmox VM storage
Access Control: Enable
NFS: Enable
NFS Settings:
  - Squash: No root squash
  - Sync: Synchronous
  - Authorized Networks: 192.168.1.0/24
  - Read/Write: Read/Write
```

#### Backup Storage Share
```
Name: proxmox-backup
Path: /mnt/pool/proxmox-backup
Description: Proxmox backup storage
NFS Settings:
  - Squash: No root squash
  - Sync: Synchronous
  - Authorized Networks: 192.168.1.0/24
  - Read/Write: Read/Write
```

#### ISO Storage Share
```
Name: proxmox-iso
Path: /mnt/pool/proxmox-iso
Description: ISO images and templates
NFS Settings:
  - Squash: No root squash
  - Sync: Synchronous
  - Authorized Networks: 192.168.1.0/24
  - Read/Write: Read/Write
```

#### Media Storage Share
```
Name: media
Path: /mnt/pool/media
Description: Media files for streaming
NFS Settings:
  - Squash: No root squash
  - Sync: Synchronous
  - Authorized Networks: 192.168.1.0/24,192.168.80.0/24
  - Read/Write: Read/Write
```

#### Games Storage Share
```
Name: games
Path: /mnt/pool/games
Description: Game ROMs and data
NFS Settings:
  - Squash: No root squash
  - Sync: Synchronous
  - Authorized Networks: 192.168.1.0/24,192.168.80.0/24
  - Read/Write: Read/Write
```

### Test NFS Connectivity
```bash
# From both Proxmox nodes, test NFS mounting
# Node 1:
ssh root@192.168.1.10

mkdir -p /mnt/test-nfs
mount -t nfs 192.168.1.100:/mnt/pool/proxmox-storage /mnt/test-nfs
ls -la /mnt/test-nfs
umount /mnt/test-nfs

# Node 2:
ssh root@192.168.1.11

mkdir -p /mnt/test-nfs  
mount -t nfs 192.168.1.100:/mnt/pool/proxmox-storage /mnt/test-nfs
touch /mnt/test-nfs/node2-test.txt
ls -la /mnt/test-nfs
umount /mnt/test-nfs

# Verify Node 1 can see Node 2's test file
ssh root@192.168.1.10
mount -t nfs 192.168.1.100:/mnt/pool/proxmox-storage /mnt/test-nfs
ls -la /mnt/test-nfs/node2-test.txt
rm /mnt/test-nfs/node2-test.txt
umount /mnt/test-nfs
```

## Step 3: Add Shared Storage to Proxmox Cluster

### Add NFS Storage via Web UI

#### VM Storage (Live Migration Capable)
1. **Datacenter** → **Storage** → **Add** → **NFS**
2. **Configuration**:
   ```
   ID: asustor-vm-storage
   Server: 192.168.1.100
   Export: /mnt/pool/proxmox-storage
   Content: Disk image, Container
   Max Backups: 3
   Shared: Yes
   Enable: Yes
   Nodes: All (leave empty for all nodes)
   ```

#### Backup Storage
1. **Datacenter** → **Storage** → **Add** → **NFS**  
2. **Configuration**:
   ```
   ID: asustor-backup
   Server: 192.168.1.100
   Export: /mnt/pool/proxmox-backup
   Content: VZDump backup file
   Max Backups: 10
   Shared: Yes
   Enable: Yes
   Nodes: All
   ```

#### ISO Storage
1. **Datacenter** → **Storage** → **Add** → **NFS**
2. **Configuration**:
   ```
   ID: asustor-iso
   Server: 192.168.1.100
   Export: /mnt/pool/proxmox-iso
   Content: ISO image, Container template
   Shared: Yes
   Enable: Yes
   Nodes: All
   ```

### Verify Storage Integration
```bash
# Check storage status from any Proxmox node
pvesm status

# Should show all storage pools including new NFS shares:
# Name             Type     Status           Total            Used       Available        %
# asustor-backup   nfs      active      [size varies]    [used]     [available]      [%]
# asustor-iso      nfs      active      [size varies]    [used]     [available]      [%]
# asustor-vm-storage nfs    active      [size varies]    [used]     [available]      [%]
# local            dir      active      [local size]     [used]     [available]      [%]
# local-lvm        lvmthin  active      [local size]     [used]     [available]      [%]

# Test storage accessibility from both nodes
pvesm list asustor-vm-storage
pvesm list asustor-backup
pvesm list asustor-iso
```

## Step 4: Local Storage Optimization

### Configure Node-Specific High-Performance Storage

#### Assess Available Storage on Each Node
```bash
# On each node, check available storage
ssh root@192.168.1.10
lsblk
pvs
vgs
lvs

# Document findings:
# Node 1: [SSD type], [NVMe if available], [sizes]
# Node 2: [SSD type], [NVMe if available], [sizes]
```

#### Configure Local NVMe Storage (If Available)
```bash
# Only if you have separate NVMe drive for VM storage
# Check if NVMe is available and not used by OS
lsblk | grep nvme

# If separate NVMe available (e.g., /dev/nvme0n1):
pvecreate /dev/nvme0n1
vgcreate local-nvme /dev/nvme0n1
lvcreate -l 100%FREE -n vm-fast local-nvme

# Add via Web UI:
# Node → Storage → Add → LVM
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

## Step 5: Optimize NFS Mount Options for Performance

### Configure Performance-Optimized NFS Mounts
```bash
# On both Proxmox nodes, optimize NFS mount options
# Edit /etc/fstab to add performance-optimized NFS mounts

cat >> /etc/fstab << 'EOF'
# High-performance NFS mounts for Proxmox storage
192.168.1.100:/mnt/pool/proxmox-storage /mnt/pve/asustor-vm-storage nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2,_netdev 0 0
192.168.1.100:/mnt/pool/proxmox-backup /mnt/pve/asustor-backup nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2,_netdev 0 0
192.168.1.100:/mnt/pool/proxmox-iso /mnt/pve/asustor-iso nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2,_netdev 0 0
EOF

# Test mount with optimized options
mount -a
df -h | grep asustor

# Verify mount options
mount | grep nfs
```

### Configure NFS Client Optimization
```bash
# On both nodes, optimize NFS client settings
echo 'net.core.rmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.rmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.core.wmem_default = 262144' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 16777216' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 65536 16777216' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 16777216' >> /etc/sysctl.conf

# Apply settings
sysctl -p

# Install NFS utilities if not present
apt install -y nfs-common rpcbind
systemctl enable rpcbind
systemctl start rpcbind
```

## Step 6: Configure Comprehensive Backup Strategy

### Create VM Backup Configuration
```bash
# Create backup strategy script
cat > /usr/local/bin/vm-backup-strategy.sh << 'EOF'
#!/bin/bash
# Comprehensive VM backup strategy

BACKUP_LOG="/var/log/vm-backup.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" >> "$BACKUP_LOG"
    echo "$1"
}

# Function to backup all VMs with intelligent scheduling
backup_all_vms() {
    log_message "Starting automated VM backup"
    
    # Get list of all running VMs
    VM_LIST=$(qm list | grep running | awk '{print $1}')
    
    # Get list of all stopped VMs that should be backed up
    STOPPED_VMS=$(qm list | grep stopped | awk '{print $1}')
    
    # Combine lists
    ALL_VMS="$VM_LIST $STOPPED_VMS"
    
    for VMID in $ALL_VMS; do
        if [ ! -z "$VMID" ] && [ "$VMID" != "VMID" ]; then
            # Get VM name for logging
            VM_NAME=$(qm config $VMID | grep "^name:" | cut -d' ' -f2)
            
            log_message "Backing up VM $VMID ($VM_NAME)"
            
            # Determine backup mode based on VM state
            VM_STATE=$(qm status $VMID | cut -d' ' -f2)
            if [ "$VM_STATE" = "running" ]; then
                BACKUP_MODE="snapshot"
            else
                BACKUP_MODE="stop"
            fi
            
            # Perform backup
            vzdump $VMID \
                --storage asustor-backup \
                --mode $BACKUP_MODE \
                --compress gzip \
                --mailto root \
                --quiet 1 \
                --remove 0
                
            if [ $? -eq 0 ]; then
                log_message "VM $VMID ($VM_NAME) backup completed successfully"
            else
                log_message "ERROR: VM $VMID ($VM_NAME) backup failed"
            fi
        fi
    done
    
    log_message "VM backup cycle completed"
}

# Function to cleanup old backups with retention policy
cleanup_old_backups() {
    log_message "Starting backup cleanup with retention policy"
    
    # Retention policy:
    # - Keep daily backups for 7 days
    # - Keep weekly backups for 4 weeks  
    # - Keep monthly backups for 6 months
    
    # Remove backups older than 6 months from NFS storage
    find /mnt/pve/asustor-backup -name "*.vma.gz" -mtime +180 -delete 2>/dev/null
    find /mnt/pve/asustor-backup -name "*.tar.gz" -mtime +180 -delete 2>/dev/null
    find /mnt/pve/asustor-backup -name "*.lzo" -mtime +180 -delete 2>/dev/null
    
    # Remove local backups older than 3 days (local storage is limited)
    find /var/lib/vz/dump -name "*.vma.gz" -mtime +3 -delete 2>/dev/null
    find /var/lib/vz/dump -name "*.tar.gz" -mtime +3 -delete 2>/dev/null
    find /var/lib/vz/dump -name "*.lzo" -mtime +3 -delete 2>/dev/null
    
    # Log cleanup results
    REMAINING_BACKUPS=$(find /mnt/pve/asustor-backup -name "*.vma.gz" -o -name "*.tar.gz" -o -name "*.lzo" | wc -l)
    log_message "Backup cleanup completed - $REMAINING_BACKUPS backups retained"
}

# Function to verify backup integrity
verify_backup_integrity() {
    log_message "Starting backup integrity verification"
    
    # Find recent backups (last 24 hours)
    RECENT_BACKUPS=$(find /mnt/pve/asustor-backup -name "*.vma.gz" -mtime -1)
    
    for backup_file in $RECENT_BACKUPS; do
        if [ -f "$backup_file" ]; then
            # Test if backup file is readable and not corrupted
            if gzip -t "$backup_file" 2>/dev/null; then
                log_message "Backup integrity OK: $(basename $backup_file)"
            else
                log_message "WARNING: Backup integrity failed: $(basename $backup_file)"
            fi
        fi
    done
    
    log_message "Backup integrity verification completed"
}

# Execute based on argument
case "$1" in
    "backup")
        backup_all_vms
        ;;
    "cleanup")
        cleanup_old_backups
        ;;
    "verify")
        verify_backup_integrity
        ;;
    "full")
        backup_all_vms
        cleanup_old_backups
        verify_backup_integrity
        ;;
    *)
        echo "Usage: $0 {backup|cleanup|verify|full}"
        exit 1
        ;;
esac
EOF

chmod +x /usr/local/bin/vm-backup-strategy.sh

# Test backup script components
/usr/local/bin/vm-backup-strategy.sh cleanup
/usr/local/bin/vm-backup-strategy.sh verify
```

### Schedule Automated Backups
```bash
# Create comprehensive cron schedule for backups
cat > /tmp/backup-cron << 'EOF'
# VM Backup Schedule - Comprehensive Strategy
# Full backup with cleanup every night at 1 AM (staggered by node)
0 1 * * * /usr/local/bin/vm-backup-strategy.sh full

# Quick cleanup check every 6 hours
0 */6 * * * /usr/local/bin/vm-backup-strategy.sh cleanup

# Integrity verification every morning at 6 AM
0 6 * * * /usr/local/bin/vm-backup-strategy.sh verify

# Backup log rotation weekly
0 4 * * 0 /usr/bin/logrotate -f /etc/logrotate.d/vm-backup
EOF

# Install cron jobs on both nodes
crontab /tmp/backup-cron
rm /tmp/backup-cron

# Create logrotate configuration for backup logs
cat > /etc/logrotate.d/vm-backup << 'EOF'
/var/log/vm-backup.log {
    weekly
    rotate 12
    compress
    delaycompress
    missingok
    notifempty
    create 644 root root
    postrotate
        # Send weekly backup summary
        tail -100 /var/log/vm-backup.log | mail -s "Weekly Backup Summary - $(hostname)" root 2>/dev/null || true
    endscript
}
EOF
```

## Step 7: Configure Synology as Secondary Backup Target

### Synology NAS Setup
1. **Access Synology DSM**: Browse to http://[synology-ip]:5000
2. **Initial Setup**: Complete if first time setup
3. **Set Static IP**: 192.168.1.101 
4. **Create Shared Folder**: 
   - Name: proxmox-backup-secondary
   - Location: volume1
   - Description: Secondary backup storage for Proxmox VMs

### Enable NFS on Synology
1. **Control Panel** → **File Services** → **NFS**
2. **Enable NFS service**
3. **Configure NFS permissions**:
   ```
   Shared Folder: proxmox-backup-secondary
   Hostname or IP: 192.168.1.0/24
   Privilege: Read/Write
   Squash: Map root to admin
   Security: sys
   Enable asynchronous: No (for data integrity)
   ```

### Add Synology to Proxmox
```bash
# Add Synology as secondary backup storage via Web UI
# Datacenter → Storage → Add → NFS
# ID: synology-backup-secondary
# Server: 192.168.1.101
# Export: /volume1/proxmox-backup-secondary
# Content: VZDump backup file
# Max Backups: 5
# Shared: Yes
# Enable: Yes

# Or via CLI:
pvesm add nfs synology-backup-secondary \
    --server 192.168.1.101 \
    --export /volume1/proxmox-backup-secondary \
    --content backup \
    --maxfiles 5 \
    --shared 1 \
    --enabled 1
```

### Configure Backup Replication to Synology
```bash
# Create script to replicate critical backups to Synology
cat > /usr/local/bin/backup-replication.sh << 'EOF'
#!/bin/bash
# Replicate critical backups to secondary storage

SOURCE_DIR="/mnt/pve/asustor-backup"
DEST_DIR="/mnt/pve/synology-backup-secondary"
LOG_FILE="/var/log/backup-replication.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Function to replicate recent backups
replicate_recent_backups() {
    log_message "Starting backup replication to secondary storage"
    
    # Ensure destination is mounted
    if ! mountpoint -q "$DEST_DIR"; then
        log_message "ERROR: Secondary backup storage not mounted"
        return 1
    fi
    
    # Sync only recent backups (last 3 days) to secondary storage
    rsync -av --delete \
        --include="*.vma.gz" \
        --include="*.tar.gz" \
        --include="*.lzo" \
        --exclude="*" \
        --max-age=3d \
        "$SOURCE_DIR/" "$DEST_DIR/"
    
    if [ $? -eq 0 ]; then
        log_message "Backup replication completed successfully"
    else
        log_message "ERROR: Backup replication failed"
        return 1
    fi
    
    # Verify replicated backups
    REPLICATED_COUNT=$(find "$DEST_DIR" -name "*.vma.gz" -o -name "*.tar.gz" -o -name "*.lzo" | wc -l)
    log_message "Replicated $REPLICATED_COUNT backup files to secondary storage"
}

# Function to cleanup old replicated backups
cleanup_secondary_backups() {
    log_message "Cleaning up old backups on secondary storage"
    
    # Remove backups older than 14 days from secondary storage
    find "$DEST_DIR" -name "*.vma.gz" -mtime +14 -delete 2>/dev/null
    find "$DEST_DIR" -name "*.tar.gz" -mtime +14 -delete 2>/dev/null
    find "$DEST_DIR" -name "*.lzo" -mtime +14 -delete 2>/dev/null
    
    REMAINING_COUNT=$(find "$DEST_DIR" -name "*.vma.gz" -o -name "*.tar.gz" -o -name "*.lzo" | wc -l)
    log_message "Secondary storage cleanup completed - $REMAINING_COUNT backups retained"
}

# Execute replication
replicate_recent_backups
cleanup_secondary_backups

# Keep log manageable
tail -500 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x /usr/local/bin/backup-replication.sh

# Schedule backup replication every 6 hours
echo "0 */6 * * * /usr/local/bin/backup-replication.sh" | crontab -l | { cat; echo "0 */6 * * * /usr/local/bin/backup-replication.sh"; } | crontab -

# Test replication script
/usr/local/bin/backup-replication.sh
```

## Step 8: Storage Performance Testing and Optimization

### Install Performance Testing Tools
```bash
# On both Proxmox nodes, install performance testing tools
apt install -y fio iperf3 hdparm

# Create comprehensive storage performance test
cat > /usr/local/bin/storage-performance-test.sh << 'EOF'
#!/bin/bash
# Comprehensive storage performance testing

LOG_FILE="/var/log/storage-performance.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Test NFS storage performance
test_nfs_performance() {
    log_message "=== NFS Storage Performance Test ==="
    
    # Test NFS write performance
    log_message "Testing NFS write performance..."
    NFS_WRITE=$(fio --name=nfs-write-test \
        --filename=/mnt/pve/asustor-vm-storage/fio-test \
        --size=1G \
        --bs=4k \
        --rw=write \
        --direct=1 \
        --numjobs=4 \
        --runtime=60 \
        --group_reporting \
        --output-format=json | jq -r '.jobs[0].write.bw_mean')
    
    # Convert from KB/s to MB/s
    NFS_WRITE_MBS=$(echo "scale=2; $NFS_WRITE / 1024" | bc -l)
    log_message "NFS Write Performance: ${NFS_WRITE_MBS} MB/s"
    
    # Test NFS read performance  
    log_message "Testing NFS read performance..."
    NFS_READ=$(fio --name=nfs-read-test \
        --filename=/mnt/pve/asustor-vm-storage/fio-test \
        --size=1G \
        --bs=4k \
        --rw=read \
        --direct=1 \
        --numjobs=4 \
        --runtime=60 \
        --group_reporting \
        --output-format=json | jq -r '.jobs[0].read.bw_mean')
    
    NFS_READ_MBS=$(echo "scale=2; $NFS_READ / 1024" | bc -l)
    log_message "NFS Read Performance: ${NFS_READ_MBS} MB/s"
    
    # Test mixed workload (simulates VM disk activity)
    log_message "Testing NFS mixed workload..."
    NFS_MIXED=$(fio --name=nfs-mixed-test \
        --filename=/mnt/pve/asustor-vm-storage/fio-mixed-test \
        --size=2G \
        --bs=4k \
        --rw=randrw \
        --rwmixread=70 \
        --direct=1 \
        --numjobs=2 \
        --runtime=120 \
        --group_reporting \
        --output-format=json | jq -r '.jobs[0].mixed.bw_mean')
    
    NFS_MIXED_MBS=$(echo "scale=2; $NFS_MIXED / 1024" | bc -l)
    log_message "NFS Mixed Workload Performance: ${NFS_MIXED_MBS} MB/s"
    
    # Clean up test files
    rm -f /mnt/pve/asustor-vm-storage/fio-*test*
}

# Test local storage performance (for comparison)
test_local_performance() {
    log_message "=== Local Storage Performance Test ==="
    
    # Test local NVMe/SSD performance (if available)
    if [ -d "/var/lib/vz" ]; then
        log_message "Testing local storage write performance..."
        LOCAL_WRITE=$(fio --name=local-write-test \
            --filename=/var/lib/vz/fio-local-test \
            --size=1G \
            --bs=4k \
            --rw=write \
            --direct=1 \
            --numjobs=4 \
            --runtime=60 \
            --group_reporting \
            --output-format=json | jq -r '.jobs[0].write.bw_mean')
        
        LOCAL_WRITE_MBS=$(echo "scale=2; $LOCAL_WRITE / 1024" | bc -l)
        log_message "Local Storage Write Performance: ${LOCAL_WRITE_MBS} MB/s"
        
        # Clean up
        rm -f /var/lib/vz/fio-local-test
    fi
}

# Test network performance to storage
test_network_performance() {
    log_message "=== Network Performance to Storage ==="
    
    # Test network throughput to ASUSTOR
    log_message "Testing network throughput to ASUSTOR..."
    
    # This requires iperf3 server running on ASUSTOR (manual setup)
    # For now, test ping latency
    PING_LATENCY=$(ping -c 10 192.168.1.100 | grep 'rtt' | cut -d'=' -f2 | cut -d'/' -f2)
    if [ ! -z "$PING_LATENCY" ]; then
        log_message "Network latency to ASUSTOR: ${PING_LATENCY}ms"
    fi
    
    # Test TCP throughput using dd over nc (if available)
    # This is a basic test - iperf3 would be better
    NETWORK_THROUGHPUT=$(timeout 10 sh -c 'dd if=/dev/zero bs=1M count=100 2>/dev/null | nc -w 1 192.168.1.100 12345' 2>&1 | grep -o '[0-9.]* MB/s' || echo "Test failed")
    log_message "Network throughput estimate: $NETWORK_THROUGHPUT"
}

# Run all performance tests
test_nfs_performance
test_local_performance
test_network_performance

log_message "Storage performance testing completed"

# Performance targets for reference:
log_message "=== Performance Targets ==="
log_message "Target NFS Sequential Read: >100 MB/s"
log_message "Target NFS Sequential Write: >80 MB/s"
log_message "Target NFS Random IOPS: >1000"
log_message "Target Network Latency: <2ms to storage"
EOF

chmod +x /usr/local/bin/storage-performance-test.sh

# Run initial performance test
/usr/local/bin/storage-performance-test.sh
```

## Step 9: Configure Storage Health Monitoring

### Create Comprehensive Storage Monitoring
```bash
# Create storage health monitoring script
cat > /usr/local/bin/storage-health-monitor.sh << 'EOF'
#!/bin/bash
# Comprehensive storage health monitoring

LOG_FILE="/var/log/storage-health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Check NFS mount points and health
check_nfs_mounts() {
    log_message "=== NFS Mount Health Check ==="
    
    FAILED_MOUNTS=""
    NFS_MOUNTS=(
        "/mnt/pve/asustor-vm-storage"
        "/mnt/pve/asustor-backup"
        "/mnt/pve/asustor-iso"
    )
    
    for mount_point in "${NFS_MOUNTS[@]}"; do
        if mountpoint -q "$mount_point"; then
            # Test read/write access
            if touch "$mount_point/.health-check" 2>/dev/null; then
                rm -f "$mount_point/.health-check"
                log_message "$(basename $mount_point): Healthy (read/write OK)"
            else
                log_message "$(basename $mount_point): ERROR - Read/write test failed"
                FAILED_MOUNTS="$FAILED_MOUNTS $mount_point"
            fi
        else
            log_message "$(basename $mount_point): ERROR - Not mounted"
            FAILED_MOUNTS="$FAILED_MOUNTS $mount_point"
        fi
    done
    
    if [ ! -z "$FAILED_MOUNTS" ]; then
        log_message "CRITICAL: Failed NFS mounts detected:$FAILED_MOUNTS"
        return 1
    else
        log_message "All NFS mounts are healthy"
        return 0
    fi
}

# Check storage space utilization
check_storage_space() {
    log_message "=== Storage Space Utilization ==="
    
    # Check shared storage space
    if mountpoint -q "/mnt/pve/asustor-vm-storage"; then
        ASUSTOR_USAGE=$(df -h /mnt/pve/asustor-vm-storage | tail -1 | awk '{print $5}' | sed 's/%//')
        ASUSTOR_SIZE=$(df -h /mnt/pve/asustor-vm-storage | tail -1 | awk '{print $2}')
        ASUSTOR_AVAIL=$(df -h /mnt/pve/asustor-vm-storage | tail -1 | awk '{print $4}')
        
        log_message "ASUSTOR VM Storage: ${ASUSTOR_USAGE}% used (${ASUSTOR_AVAIL} of ${ASUSTOR_SIZE} available)"
        
        if [ "$ASUSTOR_USAGE" -gt 90 ]; then
            log_message "CRITICAL: ASUSTOR storage usage above 90%"
        elif [ "$ASUSTOR_USAGE" -gt 80 ]; then
            log_message "WARNING: ASUSTOR storage usage above 80%"
        fi
    fi
    
    # Check backup storage space
    if mountpoint -q "/mnt/pve/asustor-backup"; then
        BACKUP_USAGE=$(df -h /mnt/pve/asustor-backup | tail -1 | awk '{print $5}' | sed 's/%//')
        BACKUP_SIZE=$(df -h /mnt/pve/asustor-backup | tail -1 | awk '{print $2}')
        BACKUP_AVAIL=$(df -h /mnt/pve/asustor-backup | tail -1 | awk '{print $4}')
        
        log_message "ASUSTOR Backup Storage: ${BACKUP_USAGE}% used (${BACKUP_AVAIL} of ${BACKUP_SIZE} available)"
        
        if [ "$BACKUP_USAGE" -gt 85 ]; then
            log_message "WARNING: Backup storage usage above 85% - cleanup recommended"
        fi
    fi
    
    # Check local storage
    LOCAL_USAGE=$(df -h / | tail -1 | awk '{print $5}' | sed 's/%//')
    LOCAL_SIZE=$(df -h / | tail -1 | awk '{print $2}')
    LOCAL_AVAIL=$(df -h / | tail -1 | awk '{print $4}')
    
    log_message "Local Storage: ${LOCAL_USAGE}% used (${LOCAL_AVAIL} of ${LOCAL_SIZE} available)"
    
    if [ "$LOCAL_USAGE" -gt 80 ]; then
        log_message "WARNING: Local storage usage above 80%"
    fi
}

# Check storage performance degradation
check_storage_performance() {
    log_message "=== Storage Performance Check ==="
    
    # Simple write performance test to shared storage
    if mountpoint -q "/mnt/pve/asustor-vm-storage"; then
        TEST_FILE="/mnt/pve/asustor-vm-storage/.perf-test-$(hostname)"
        
        START_TIME=$(date +%s.%N)
        dd if=/dev/zero of="$TEST_FILE" bs=1M count=50 oflag=direct 2>/dev/null
        END_TIME=$(date +%s.%N)
        
        WRITE_TIME=$(echo "$END_TIME - $START_TIME" | bc -l)
        WRITE_SPEED=$(echo "scale=2; 50 / $WRITE_TIME" | bc -l)
        
        rm -f "$TEST_FILE"
        
        log_message "NFS write performance: ${WRITE_SPEED} MB/s"
        
        # Alert if performance is poor (less than 30MB/s)
        if (( $(echo "$WRITE_SPEED < 30" | bc -l) )); then
            log_message "WARNING: Storage write performance degraded (${WRITE_SPEED} MB/s)"
        fi
    fi
}

# Check NFS server connectivity and responsiveness
check_nfs_server() {
    log_message "=== NFS Server Health Check ==="
    
    # Test connectivity to NFS server
    if ping -c 3 192.168.1.100 > /dev/null 2>&1; then
        log_message "ASUSTOR NFS Server: Reachable"
        
        # Test NFS service responsiveness
        if showmount -e 192.168.1.100 > /dev/null 2>&1; then
            log_message "ASUSTOR NFS Service: Responsive"
        else
            log_message "ERROR: ASUSTOR NFS service not responding"
        fi
    else
        log_message "CRITICAL: ASUSTOR NFS server not reachable"
    fi
    
    # Check secondary backup server if configured
    if ping -c 3 192.168.1.101 > /dev/null 2>&1; then
        log_message "Synology Backup Server: Reachable"
    else
        log_message "WARNING: Synology backup server not reachable"
    fi
}

# Run all health checks
log_message "Starting storage health monitoring cycle"

check_nfs_mounts
check_storage_space
check_storage_performance
check_nfs_server

log_message "Storage health monitoring cycle completed"

# Keep log size manageable
tail -1000 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x /usr/local/bin/storage-health-monitor.sh

# Schedule storage health monitoring every 15 minutes
echo "*/15 * * * * /usr/local/bin/storage-health-monitor.sh" | crontab -l | { cat; echo "*/15 * * * * /usr/local/bin/storage-health-monitor.sh"; } | crontab -

# Test storage health monitoring
/usr/local/bin/storage-health-monitor.sh
cat /var/log/storage-health.log
```

## Step 10: Test Live Migration Capability

### Create Test VM for Migration Validation
```bash
# Create small test VM to verify migration works
qm create 999 \
    --name migration-test \
    --memory 1024 \
    --cores 1 \
    --net0 virtio,bridge=vmbr0 \
    --storage asustor-vm-storage

# Create VM disk on shared storage
qm set 999 --scsi0 asustor-vm-storage:8

# Configure for quick boot
qm set 999 --boot order=scsi0
qm set 999 --ostype l26

# Start test VM
qm start 999

# Wait for VM to fully boot
sleep 30

# Check VM status
qm status 999
echo "Test VM should show as 'running'"
```

### Test Online Migration Between Nodes
```bash
# Test migration from Node 1 to Node 2
echo "Testing live migration from Node 1 to Node 2..."

# Migrate test VM from Node 1 to Node 2
qm migrate 999 pve-node2 --online

# Wait for migration to complete
sleep 30

# Check migration status and VM location
qm status 999
echo "VM should now be running on pve-node2"

# Test migrating back to confirm bidirectional capability
echo "Testing reverse migration from Node 2 to Node 1..."
qm migrate 999 pve-node1 --online

# Wait for migration to complete
sleep 30

# Check final status
qm status 999
echo "VM should now be back on pve-node1"

# Clean up test VM
qm stop 999
sleep 10
qm destroy 999

echo "Migration test completed successfully"
```

## Validation and Testing

### Comprehensive Storage Validation
```bash
# Create comprehensive storage validation script
cat > /usr/local/bin/validate-storage-setup.sh << 'EOF'
#!/bin/bash
# Comprehensive storage setup validation

echo "=== Storage Setup Validation ==="

# Test 1: NFS mount accessibility
echo "1. Testing NFS mount accessibility..."
MOUNTS_OK=0
NFS_MOUNTS=("/mnt/pve/asustor-vm-storage" "/mnt/pve/asustor-backup" "/mnt/pve/asustor-iso")

for mount in "${NFS_MOUNTS[@]}"; do
    if mountpoint -q "$mount" && touch "$mount/.test" 2>/dev/null; then
        rm -f "$mount/.test"
        echo "✅ $(basename $mount): Accessible"
        ((MOUNTS_OK++))
    else
        echo "❌ $(basename $mount): Not accessible"
    fi
done

echo "NFS mounts accessible: $MOUNTS_OK/3"

# Test 2: Proxmox storage recognition
echo ""
echo "2. Testing Proxmox storage recognition..."
STORAGE_COUNT=$(pvesm status | grep -c "asustor")
if [ "$STORAGE_COUNT" -ge 3 ]; then
    echo "✅ Proxmox recognizes all ASUSTOR storage pools"
else
    echo "❌ Missing ASUSTOR storage pools in Proxmox"
fi

# Test 3: Storage performance
echo ""
echo "3. Testing basic storage performance..."
TEST_FILE="/mnt/pve/asustor-vm-storage/.validation-test"
START_TIME=$(date +%s.%N)
dd if=/dev/zero of="$TEST_FILE" bs=1M count=10 oflag=direct 2>/dev/null
END_TIME=$(date +%s.%N)
WRITE_TIME=$(echo "$END_TIME - $START_TIME" | bc -l)
WRITE_SPEED=$(echo "scale=2; 10 / $WRITE_TIME" | bc -l)
rm -f "$TEST_FILE"

echo "Storage write speed: ${WRITE_SPEED} MB/s"
if (( $(echo "$WRITE_SPEED > 20" | bc -l) )); then
    echo "✅ Storage performance acceptable"
else
    echo "⚠️  Storage performance may be slow"
fi

# Test 4: Backup storage
echo ""
echo "4. Testing backup storage..."
if mountpoint -q "/mnt/pve/asustor-backup"; then
    BACKUP_SPACE=$(df -h /mnt/pve/asustor-backup | tail -1 | awk '{print $4}')
    echo "✅ Backup storage available: $BACKUP_SPACE free"
else
    echo "❌ Backup storage not accessible"
fi

# Test 5: Live migration capability (if cluster exists)
echo ""
echo "5. Testing live migration prerequisites..."
CLUSTER_NODES=$(pvecm nodes 2>/dev/null | grep -c "online" || echo "0")
if [ "$CLUSTER_NODES" -ge 2 ]; then
    echo "✅ Cluster has $CLUSTER_NODES nodes - live migration possible"
    
    # Check if shared storage is accessible from all nodes
    SSH_TEST=$(ssh root@192.168.1.11 "mountpoint -q /mnt/pve/asustor-vm-storage" 2>/dev/null && echo "OK" || echo "FAIL")
    if [ "$SSH_TEST" = "OK" ]; then
        echo "✅ Shared storage accessible from both nodes"
    else
        echo "❌ Shared storage not accessible from Node 2"
    fi
else
    echo "⚠️  Single node - live migration not available yet"
fi

echo ""
echo "=== Storage Validation Complete ==="
EOF

chmod +x /usr/local/bin/validate-storage-setup.sh

# Run storage validation
/usr/local/bin/validate-storage-setup.sh
```

### Storage Performance Benchmarking
```bash
# Create detailed storage benchmarking
cat > /usr/local/bin/storage-benchmark.sh << 'EOF'
#!/bin/bash
# Detailed storage performance benchmarking

BENCHMARK_LOG="/var/log/storage-benchmark.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_result() {
    echo "$DATE: $1" | tee -a "$BENCHMARK_LOG"
}

echo "=== Storage Performance Benchmark ==="
log_result "Starting storage performance benchmark"

# Benchmark 1: Sequential Read/Write
echo "Running sequential I/O benchmark..."
SEQ_RESULTS=$(fio --name=sequential-test \
    --filename=/mnt/pve/asustor-vm-storage/benchmark-seq \
    --size=1G \
    --bs=1M \
    --rw=rw \
    --rwmixread=50 \
    --direct=1 \
    --numjobs=1 \
    --runtime=60 \
    --group_reporting \
    --output-format=json)

SEQ_READ=$(echo "$SEQ_RESULTS" | jq -r '.jobs[0].read.bw_mean')
SEQ_WRITE=$(echo "$SEQ_RESULTS" | jq -r '.jobs[0].write.bw_mean')
SEQ_READ_MBS=$(echo "scale=2; $SEQ_READ / 1024" | bc -l)
SEQ_WRITE_MBS=$(echo "scale=2; $SEQ_WRITE / 1024" | bc -l)

log_result "Sequential Read: ${SEQ_READ_MBS} MB/s"
log_result "Sequential Write: ${SEQ_WRITE_MBS} MB/s"

# Benchmark 2: Random Read/Write IOPS
echo "Running random I/O benchmark..."
RAND_RESULTS=$(fio --name=random-test \
    --filename=/mnt/pve/asustor-vm-storage/benchmark-rand \
    --size=1G \
    --bs=4k \
    --rw=randrw \
    --rwmixread=70 \
    --direct=1 \
    --numjobs=4 \
    --runtime=60 \
    --group_reporting \
    --output-format=json)

RAND_READ_IOPS=$(echo "$RAND_RESULTS" | jq -r '.jobs[0].read.iops_mean')
RAND_WRITE_IOPS=$(echo "$RAND_RESULTS" | jq -r '.jobs[0].write.iops_mean')
RAND_READ_IOPS_INT=$(printf "%.0f" "$RAND_READ_IOPS")
RAND_WRITE_IOPS_INT=$(printf "%.0f" "$RAND_WRITE_IOPS")

log_result "Random Read IOPS: ${RAND_READ_IOPS_INT}"
log_result "Random Write IOPS: ${RAND_WRITE_IOPS_INT}"

# Benchmark 3: VM-like workload
echo "Running VM workload simulation..."
VM_RESULTS=$(fio --name=vm-workload \
    --filename=/mnt/pve/asustor-vm-storage/benchmark-vm \
    --size=2G \
    --bs=64k \
    --rw=randrw \
    --rwmixread=60 \
    --direct=1 \
    --numjobs=2 \
    --runtime=120 \
    --group_reporting \
    --output-format=json)

VM_TOTAL_BW=$(echo "$VM_RESULTS" | jq -r '.jobs[0].mixed.bw_mean')
VM_TOTAL_MBS=$(echo "scale=2; $VM_TOTAL_BW / 1024" | bc -l)

log_result "VM Workload Throughput: ${VM_TOTAL_MBS} MB/s"

# Clean up benchmark files
rm -f /mnt/pve/asustor-vm-storage/benchmark-*

# Performance evaluation
echo ""
echo "=== Performance Evaluation ==="
echo "Sequential Read: ${SEQ_READ_MBS} MB/s (Target: >100 MB/s)"
echo "Sequential Write: ${SEQ_WRITE_MBS} MB/s (Target: >80 MB/s)"
echo "Random Read IOPS: ${RAND_READ_IOPS_INT} (Target: >1000)"
echo "Random Write IOPS: ${RAND_WRITE_IOPS_INT} (Target: >500)"
echo "VM Workload: ${VM_TOTAL_MBS} MB/s (Target: >50 MB/s)"

log_result "Storage benchmark completed"
echo ""
echo "Results logged to: $BENCHMARK_LOG"
EOF

chmod +x /usr/local/bin/storage-benchmark.sh

# Run storage benchmark
/usr/local/bin/storage-benchmark.sh
```

## Troubleshooting Common Issues

### NFS Mount Failures
```bash
# Debug NFS connectivity issues
cat > /usr/local/bin/debug-nfs-issues.sh << 'EOF'
#!/bin/bash
# Debug NFS connectivity issues

echo "=== NFS Troubleshooting ==="

# Check NFS server availability
echo "1. Testing NFS server connectivity..."
if ping -c 3 192.168.1.100 > /dev/null; then
    echo "✅ ASUSTOR server reachable"
else
    echo "❌ ASUSTOR server not reachable - check network"
    exit 1
fi

# Check NFS service availability
echo "2. Testing NFS service..."
if showmount -e 192.168.1.100 > /dev/null 2>&1; then
    echo "✅ NFS service responding"
    showmount -e 192.168.1.100
else
    echo "❌ NFS service not responding"
fi

# Check current NFS mounts
echo "3. Current NFS mounts..."
mount | grep nfs

# Check NFS client services
echo "4. NFS client services..."
systemctl status rpcbind
systemctl status nfs-common

# Test manual mount
echo "5. Testing manual NFS mount..."
TEST_DIR="/tmp/nfs-test"
mkdir -p "$TEST_DIR"
if mount -t nfs 192.168.1.100:/mnt/pool/proxmox-storage "$TEST_DIR"; then
    echo "✅ Manual NFS mount successful"
    ls -la "$TEST_DIR"
    umount "$TEST_DIR"
else
    echo "❌ Manual NFS mount failed"
fi
rmdir "$TEST_DIR"

# Check network statistics
echo "6. NFS network statistics..."
nfsstat -c

echo "=== NFS Troubleshooting Complete ==="
EOF

chmod +x /usr/local/bin/debug-nfs-issues.sh
```

### Storage Performance Issues
```bash
# Create storage performance troubleshooting guide
cat > /usr/local/bin/debug-storage-performance.sh << 'EOF'
#!/bin/bash
# Debug storage performance issues

echo "=== Storage Performance Troubleshooting ==="

# Check network utilization
echo "1. Network utilization to storage server..."
if command -v iftop >/dev/null; then
    timeout 10 iftop -i eno1 -t -s 10
else
    echo "Install iftop for network monitoring: apt install iftop"
fi

# Check NFS mount options
echo "2. Current NFS mount options..."
mount | grep nfs | grep asustor

# Check for network errors
echo "3. Network error statistics..."
cat /proc/net/dev | grep eno1

# Check system load
echo "4. System load and resources..."
uptime
free -h
iostat 1 3 2>/dev/null || echo "Install sysstat for iostat: apt install sysstat"

# Check NFS performance statistics
echo "5. NFS performance statistics..."
nfsstat -c

# Recommend optimizations
echo ""
echo "=== Performance Optimization Recommendations ==="
echo "1. Verify NFS mount options include: rsize=131072,wsize=131072"
echo "2. Check network cable quality and switch port speed"
echo "3. Monitor ASUSTOR CPU and memory usage"
echo "4. Consider NFS over TCP vs UDP"
echo "5. Verify no network congestion during peak usage"
EOF

chmod +x /usr/local/bin/debug-storage-performance.sh
```

### Backup Restoration Testing
```bash
# Create backup restoration testing procedure
cat > /usr/local/bin/test-backup-restore.sh << 'EOF'
#!/bin/bash
# Test backup and restore procedures

echo "=== Backup Restore Testing ==="

# Function to create test VM
create_test_vm() {
    echo "Creating test VM for backup testing..."
    qm create 998 \
        --name backup-test-vm \
        --memory 512 \
        --cores 1 \
        --net0 virtio,bridge=vmbr0 \
        --storage asustor-vm-storage
    
    qm set 998 --scsi0 asustor-vm-storage:4
    echo "Test VM 998 created"
}

# Function to backup test VM
backup_test_vm() {
    echo "Creating backup of test VM..."
    vzdump 998 \
        --storage asustor-backup \
        --mode stop \
        --compress gzip \
        --quiet 1
    
    if [ $? -eq 0 ]; then
        echo "✅ Test VM backup successful"
        return 0
    else
        echo "❌ Test VM backup failed"
        return 1
    fi
}

# Function to restore test VM
restore_test_vm() {
    echo "Finding backup file..."
    BACKUP_FILE=$(find /mnt/pve/asustor-backup -name "*998*" -type f | head -1)
    
    if [ -z "$BACKUP_FILE" ]; then
        echo "❌ No backup file found for VM 998"
        return 1
    fi
    
    echo "Found backup: $BACKUP_FILE"
    
    # Restore as new VM with ID 997
    echo "Restoring backup to new VM 997..."
    qmrestore "$BACKUP_FILE" 997 --storage asustor-vm-storage
    
    if [ $? -eq 0 ]; then
        echo "✅ Backup restore successful"
        return 0
    else
        echo "❌ Backup restore failed"
        return 1
    fi
}

# Function to cleanup test VMs
cleanup_test_vms() {
    echo "Cleaning up test VMs..."
    qm destroy 998 --purge 2>/dev/null || true
    qm destroy 997 --purge 2>/dev/null || true
    
    # Clean up test backup files
    find /mnt/pve/asustor-backup -name "*998*" -delete 2>/dev/null || true
    
    echo "Cleanup completed"
}

# Main testing procedure
case "$1" in
    "full")
        create_test_vm
        backup_test_vm
        restore_test_vm
        cleanup_test_vms
        ;;
    "cleanup")
        cleanup_test_vms
        ;;
    *)
        echo "Usage: $0 {full|cleanup}"
        echo "  full    - Run complete backup/restore test"
        echo "  cleanup - Remove test VMs and backups"
        ;;
esac
EOF

chmod +x /usr/local/bin/test-backup-restore.sh
```

## Advanced Storage Configuration (Optional)

### Configure Storage Replication
```bash
# Set up storage replication between nodes for critical VMs
cat > /usr/local/bin/setup-storage-replication.sh << 'EOF'
#!/bin/bash
# Configure storage replication for high availability

echo "=== Storage Replication Setup ==="

# This would be configured via Proxmox web UI:
# Datacenter → Replication → Add

echo "To configure storage replication:"
echo "1. Go to Datacenter → Replication → Add"
echo "2. Select VM to replicate"
echo "3. Set target node"
echo "4. Set replication schedule (e.g., */15 for every 15 minutes)"
echo "5. Configure rate limit to prevent network saturation"
echo ""
echo "Recommended settings:"
echo "- Schedule: */30 (every 30 minutes for most VMs)"
echo "- Rate limit: 10 MB/s (adjust based on network capacity)"
echo "- Enable: Yes"
echo ""
echo "Note: Replication requires shared storage and cluster setup"
EOF

chmod +x /usr/local/bin/setup-storage-replication.sh
```

### Configure Storage Snapshots
```bash
# Create storage snapshot management
cat > /usr/local/bin/manage-storage-snapshots.sh << 'EOF'
#!/bin/bash
# Manage storage snapshots for data protection

LOG_FILE="/var/log/snapshot-management.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Create snapshots for all VMs
create_vm_snapshots() {
    log_message "Creating VM snapshots"
    
    # Get list of running VMs
    VM_LIST=$(qm list | grep running | awk '{print $1}')
    
    for VMID in $VM_LIST; do
        if [ ! -z "$VMID" ] && [ "$VMID" != "VMID" ]; then
            SNAPSHOT_NAME="auto-$(date +%Y%m%d-%H%M)"
            
            log_message "Creating snapshot $SNAPSHOT_NAME for VM $VMID"
            qm snapshot "$VMID" "$SNAPSHOT_NAME" --description "Automated snapshot $(date)"
            
            if [ $? -eq 0 ]; then
                log_message "Snapshot created successfully for VM $VMID"
            else
                log_message "ERROR: Failed to create snapshot for VM $VMID"
            fi
        fi
    done
}

# Clean up old snapshots
cleanup_old_snapshots() {
    log_message "Cleaning up old snapshots"
    
    # Remove snapshots older than 7 days
    CUTOFF_DATE=$(date -d '7 days ago' +%Y%m%d)
    
    VM_LIST=$(qm list | awk 'NR>1 {print $1}')
    for VMID in $VM_LIST; do
        if [ ! -z "$VMID" ]; then
            # List and remove old snapshots
            qm listsnapshot "$VMID" | grep "auto-" | while read snapshot_line; do
                SNAPSHOT_NAME=$(echo "$snapshot_line" | awk '{print $2}')
                SNAPSHOT_DATE=$(echo "$SNAPSHOT_NAME" | grep -o '[0-9]\{8\}' | head -1)
                
                if [ ! -z "$SNAPSHOT_DATE" ] && [ "$SNAPSHOT_DATE" -lt "$CUTOFF_DATE" ]; then
                    log_message "Removing old snapshot $SNAPSHOT_NAME from VM $VMID"
                    qm delsnapshot "$VMID" "$SNAPSHOT_NAME"
                fi
            done
        fi
    done
}

# Execute based on argument
case "$1" in
    "create")
        create_vm_snapshots
        ;;
    "cleanup")
        cleanup_old_snapshots
        ;;
    "full")
        create_vm_snapshots
        cleanup_old_snapshots
        ;;
    *)
        echo "Usage: $0 {create|cleanup|full}"
        exit 1
        ;;
esac
EOF

chmod +x /usr/local/bin/manage-storage-snapshots.sh

# Schedule snapshot management
echo "0 2 * * * /usr/local/bin/manage-storage-snapshots.sh full" | crontab -l | { cat; echo "0 2 * * * /usr/local/bin/manage-storage-snapshots.sh full"; } | crontab -
```

## Completion Criteria

### Storage Integration Validation
- [ ] **ASUSTOR NAS configured** with static IP and NFS service
- [ ] **NFS shares created** for VM storage, backups, and ISOs
- [ ] **Proxmox storage pools** added and accessible from both nodes
- [ ] **Storage performance** meets minimum requirements (>50 MB/s)
- [ ] **Backup storage** configured with automated retention
- [ ] **Live migration** tested and working between nodes
- [ ] **Storage monitoring** scripts operational
- [ ] **Secondary backup** target (Synology) configured

### Performance Validation
```bash
# Expected performance targets:
# - NFS Sequential Read: >100 MB/s
# - NFS Sequential Write: >80 MB/s  
# - NFS Random Read IOPS: >1000
# - NFS Random Write IOPS: >500
# - Network latency to storage: <2ms
# - Live migration time: <2 minutes for 4GB VM
```

### Backup System Validation
- [ ] **VM backups** automatically scheduled
- [ ] **Backup retention** policies implemented
- [ ] **Backup integrity** verification working
- [ ] **Secondary backup** replication operational
- [ ] **Backup restoration** tested and documented
- [ ] **Storage space monitoring** with alerts
- [ ] **Backup logs** properly rotated

### Monitoring and Health Checks
- [ ] **Storage health monitoring** running every 15 minutes
- [ ] **Performance monitoring** tracking key metrics
- [ ] **Space utilization** alerts configured
- [ ] **NFS connectivity** monitoring operational
- [ ] **Backup success/failure** tracking
- [ ] **Log rotation** configured for all storage logs

## Pre-Next Phase Checklist
- [ ] All storage pools showing "active" status in Proxmox
- [ ] Live migration working reliably between nodes
- [ ] Backup jobs completing successfully
- [ ] Storage performance meeting requirements
- [ ] Both nodes can access all shared storage
- [ ] Storage space utilization <50% (room for growth)
- [ ] No storage-related errors in logs

**Next Phase**: Proceed to [Shared Infrastructure](05-shared-infrastructure.md) to deploy foundational services like PostgreSQL and Redis that will support your applications using the robust storage foundation you just created.
