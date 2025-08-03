# Phase 2: Secondary Node Deployment and Cluster Formation

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [Primary Node Setup](02-primary-node.md)  
**Next Phase**: [Storage Setup](04-storage-setup.md)  
**Duration**: 2-3 hours  
**Prerequisites**: Node 1 operational and accessible

## Overview
Install Proxmox on the second Dell node and create a cluster foundation for high availability and shared management.

## Step 1: Install Proxmox on Node 2

### Hardware Preparation
1. Connect Node 2 to UniFi Flex Mini switch (port 2)
2. Connect power via Anker Prime USB-C
3. Verify BIOS settings match Node 1 configuration
4. Connect keyboard, mouse, monitor

### Installation Process
Use the same installation process as Node 1 with these network differences:

```
Network Configuration for Node 2:
IP Address: 192.168.10.11
Netmask: 255.255.255.0 (/24)
Gateway: 192.168.10.1
DNS: 192.168.10.1
Hostname: pve-node2.mumblescavern.local
```

**System Configuration:**
```
Timezone: America/New_York (same as Node 1)
Root Password: [SAME password as Node 1 for cluster compatibility]
Email: your-email@domain.com
```

### Post-Installation Network Configuration
After Proxmox installation, configure the network bridge for VLAN-aware operation:

```bash
# SSH to Node 2 
ssh root@192.168.10.11

# Edit network configuration for VLAN-aware bridge
nano /etc/network/interfaces

# Configure network interfaces:
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.11/24
    gateway 192.168.10.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Apply network configuration
ifreload -a

# Test connectivity
ping 192.168.10.1   # Test VLAN 10 gateway
ping 8.8.8.8        # Test internet
ping google.com     # Test DNS
```

**Important**: Ensure the gateway matches the VLAN subnet. Common mistake is using `192.168.0.1` (Default VLAN) when node is on `192.168.10.x` (Management VLAN).

### Post-Installation Updates
```bash
# Update system
apt update && apt full-upgrade -y

# Install essential packages
apt install -y \
    htop \
    iotop \
    ncdu \
    tree \
    curl \
    wget \
    git \
    vim \
    tmux \
    prometheus-node-exporter

# Enable node exporter
systemctl enable prometheus-node-exporter
systemctl start prometheus-node-exporter

# Reboot if kernel updated
reboot
```

### Configure Storage (If available)
```bash
# Check available storage
lsblk

# If Node 2 has NVMe, configure same as Node 1
# If you have unused NVMe (e.g., /dev/nvme0n1):
pvecreate /dev/nvme0n1
vgcreate vm-storage /dev/nvme0n1
lvcreate -l 100%FREE -n vm-data vm-storage

# Add to Proxmox storage via web UI:
# Node → local storage → Add → LVM
# ID: vm-nvme-storage
# Volume group: vm-storage
# Content: Disk image, Container
```

## Step 2: Create Proxmox Cluster

### From Node 1 (Primary)
```bash
# SSH to Node 1
ssh root@192.168.10.10

# Create cluster
pvecm create mumbles-cluster

# Check cluster status
pvecm status

# Should output something like:
# Cluster information
# -------------------
# Name:             mumbles-cluster
# Config Version:   1
# Transport:        knet
# Secure auth:      on
```

### From Node 2 (Join cluster)
```bash
# SSH to Node 2
ssh root@192.168.10.11

# Join the cluster (run from Node 2)
pvecm add 192.168.10.10

# You'll be prompted for Node 1's root password
# This establishes cluster communication and shared configuration
```

**Expected Output:**
```
Please enter superuser (root) password for '192.168.10.10': [enter password]
Establishing API connection with host '192.168.10.10'
The authenticity of host '192.168.10.10' can't be established.
X509 SHA256 key fingerprint is [fingerprint]
Are you sure you want to continue connecting (yes/no)? yes
Login succeeded.
Request addition of this node
Join request OK, finishing setup locally
stopping pve-cluster service
backup old database to '/var/lib/pve-cluster/backup/config-1234567890.sql.gz'
waiting for quorum...OK
(re)generate node files
(re)generate ssh keys
Generating public/private rsa key pair.
Your identification has been saved in /etc/pve/nodes/pve-node2/priv/pve-ssl-key
Your public key has been saved in /etc/pve/nodes/pve-node2/priv/pve-ssl-key.pub
(re)generate node certificate
(re)generate ssh known_hosts
Restart services...
Successfully added node to cluster.
```

### Verify Cluster Formation
```bash
# From either node:
pvecm status
pvecm nodes

# Web UI: Should now show both nodes in left panel
# Refresh browser if you don't see Node 2 immediately
```

## Step 3: Configure Cluster Networking

### Enable Cluster Network on Both Nodes
```bash
# On both nodes, verify cluster communication
# Check corosync status
systemctl status corosync

# Check cluster communication
pvecm status

# Should show both nodes as online
```

### Configure High Availability Prerequisites
```bash
# Enable cluster filesystem
# This allows shared configuration across nodes
systemctl enable pve-cluster
systemctl start pve-cluster

# Verify shared storage (we'll configure this in next phase)
df -h /etc/pve
```

## Step 4: Configure Cluster-Wide Settings

### Set Cluster-Wide Configurations via Web UI

#### Email Notifications
1. **Datacenter** → **Options** → **Email**
2. Configure mail server settings (if you have them)
3. Or skip for now (can configure later)

#### Update Repositories

**Manual Learning Approach for Node 1:**
1. SSH to Node 1 for hands-on learning:
   ```bash
   ssh root@192.168.10.10
   
   # Check what repos are currently configured
   ls -la /etc/apt/sources.list.d/
   cat /etc/apt/sources.list.d/pve-enterprise.list
   cat /etc/apt/sources.list.d/ceph.list
   
   # Comment out enterprise repo (manual edit)
   nano /etc/apt/sources.list.d/pve-enterprise.list
   # Add # to comment out the deb line
   
   # Comment out ceph enterprise repo (manual edit)
   nano /etc/apt/sources.list.d/ceph.list
   # Add # to comment out the deb line
   
   # Create no-subscription repo (manual)
   nano /etc/apt/sources.list.d/pve-no-subscription.list
   # Add: deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
   
   # Test the fix
   apt update
   apt full-upgrade -y
   ```

**Automated Script for Nodes 2-4:**
After learning the manual process, create automation for remaining nodes:
   ```bash
   # Create automation script for remaining nodes
   cat > /tmp/fix-repos.sh << 'EOF'
   #!/bin/bash
   sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
   sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list
   echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list
   apt update && apt full-upgrade -y
   EOF
   
   chmod +x /tmp/fix-repos.sh
   
   # Deploy to remaining nodes:
   scp /tmp/fix-repos.sh root@192.168.10.11:/tmp/
   ssh root@192.168.10.11 "/tmp/fix-repos.sh"
   ```

This gives you both: deep understanding from manual work + efficiency for scale!

#### Cluster Resource Management
1. **Datacenter** → **Resource Mappings**
2. Plan for future HA configuration
3. Note that full HA requires shared storage (next phase)

## Step 5: Network Bridge VLAN Configuration

### Configure VLAN Bridges for Future VMs
While logged into either node via web UI:

1. **Go to Node** → **System** → **Network**
2. **Create additional bridges for VLANs:**

```
Bridge: vmbr10 (Management VLAN)
- Bridge ports: (none - VLAN only)  
- VLAN ID: 10
- IP: (none - VMs will use this)
- Gateway: (none)
- Comment: Management VLAN bridge

Bridge: vmbr20 (Services VLAN)
- Bridge ports: (none - VLAN only)
- VLAN ID: 20  
- IP: (none - VMs will use this)
- Gateway: (none)
- Comment: Services VLAN bridge

Bridge: vmbr30 (Storage VLAN)
- Bridge ports: (none - VLAN only)
- VLAN ID: 30
- IP: (none - VMs will use this) 
- Gateway: (none)
- Comment: Storage VLAN bridge
```

**Note**: These bridges will be used by VMs to connect to specific VLANs. We'll use the main vmbr0 with VLAN tagging for most deployments.

## Step 6: Test Cluster Functionality

### Cluster Communication Test
```bash
# Test cluster messaging between nodes
# From Node 1:
pvecm nodes
# Should show both nodes with status "online"

# Test configuration synchronization
# Create a test file on Node 1:
echo "test" > /etc/pve/test-cluster-sync

# Check if file appears on Node 2:
ssh root@192.168.1.11 cat /etc/pve/test-cluster-sync

# Should output "test" - indicates cluster filesystem is working
# Clean up test file:
rm /etc/pve/test-cluster-sync
```

### Web UI Cluster Management
1. Login to either node's web UI
2. Left panel should show both nodes
3. Click between nodes - should switch context
4. Verify you can manage both nodes from either web interface

### Resource Pool Testing
```bash
# Check cluster resources
pvesh get /cluster/resources

# Should show resources from both nodes:
# - CPU, memory, storage from each node
# - Network interfaces
# - Node status
```

## Step 7: Backup and Recovery Configuration

### Configure Cluster Backup Strategy
```bash
# Create shared backup script for cluster-wide backups
cat > /usr/local/bin/cluster-backup.sh << 'EOF'
#!/bin/bash
# Cluster-wide backup script

BACKUP_DIR="/backup-local"
DATE=$(date +%Y%m%d_%H%M%S)

# Backup cluster configuration
cp -r /etc/pve "$BACKUP_DIR/cluster-config-$DATE"

# Create cluster resource dump
pvesh get /cluster/resources > "$BACKUP_DIR/cluster-resources-$DATE.json"

# Log backup completion
echo "$(date): Cluster backup completed to $BACKUP_DIR/cluster-config-$DATE" >> /var/log/cluster-backup.log
EOF

chmod +x /usr/local/bin/cluster-backup.sh

# Test the backup script
/usr/local/bin/cluster-backup.sh
ls -la /backup-local/

# Schedule daily cluster backups
echo "0 2 * * * /usr/local/bin/cluster-backup.sh" | crontab -
```

### Test VM Migration Capability (Preparation)
```bash
# Verify migration prerequisites are met
# (We'll test actual migration after VM deployment)

# Check SSH connectivity between nodes
ssh root@192.168.1.11 "echo 'Migration test successful'"

# Check shared storage readiness (will configure in next phase)
# For now, just verify both nodes can access same network storage location
```

## Step 8: Cluster Security Hardening

### Configure Cluster Firewall
```bash
# Enable cluster firewall on both nodes
# Web UI: Datacenter → Firewall → Options
# Firewall: Yes
# Input Policy: DROP
# Output Policy: ACCEPT
# Log level: info
```

### Essential Firewall Rules (via Web UI)
```
1. SSH Access:
   Direction: in
   Action: ACCEPT  
   Protocol: tcp
   Dest. port: 22
   Source: 192.168.10.0/24,192.168.20.0/24,192.168.90.0/24
   Comment: SSH access from management networks

2. Proxmox Web UI:
   Direction: in
   Action: ACCEPT
   Protocol: tcp  
   Dest. port: 8006
   Source: 192.168.10.0/24,192.168.20.0/24,192.168.90.0/24
   Comment: Proxmox web interface

3. Cluster Communication:
   Direction: in
   Action: ACCEPT
   Protocol: udp
   Dest. port: 5404-5405
   Source: 192.168.10.10,192.168.10.11
   Comment: Corosync cluster communication

4. Proxmox Cluster Communication:
   Direction: in  
   Action: ACCEPT
   Protocol: tcp
   Dest. port: 22,5900-5999,8006
   Source: 192.168.10.10,192.168.10.11  
   Comment: Proxmox inter-node communication

5. Node Exporter (Monitoring):
   Direction: in
   Action: ACCEPT
   Protocol: tcp
   Dest. port: 9100
   Source: 192.168.10.0/24,192.168.20.0/24
   Comment: Prometheus monitoring access
```

### SSH Key Distribution (Enhance Security)
```bash
# Generate cluster SSH keys for migration
# On Node 1:
ssh-keygen -t rsa -b 4096 -f /root/.ssh/cluster_rsa -N ""

# Copy public key to Node 2
ssh-copy-id -i /root/.ssh/cluster_rsa.pub root@192.168.10.11

# Copy private key to Node 2 for bidirectional migration
scp /root/.ssh/cluster_rsa root@192.168.10.11:/root/.ssh/

# Test passwordless SSH both directions
ssh -i /root/.ssh/cluster_rsa root@192.168.10.11 "hostname"
# Should return: pve-node2
```

## Step 9: Configure Resource Limits and Policies

### Memory Overcommit Configuration
```bash
# Configure reasonable memory overcommit
# Web UI: Datacenter → Options → Memory
# Memory overcommit: 1.0 (no overcommit initially)
# Can adjust later based on usage patterns
```

### CPU Resource Management
```bash
# Configure CPU allocation policies
# Web UI: Datacenter → Options → CPU
# CPU type: host (best performance for homelab)
# Enable NUMA: Yes (if nodes have multiple CPU sockets)
```

### Migration Settings
```bash
# Configure migration policies
# Web UI: Datacenter → Options → Migration
# Migration type: secure (encrypted migration)
# Bandwidth limit: 0 (unlimited - you have 1Gbps)
# Migration network: default (can optimize later)
```

## Step 10: Cluster Health Monitoring

### Install Cluster Monitoring Tools
```bash
# Install cluster status monitoring
# On both nodes:
apt install -y crmsh pcs

# Configure cluster status checks
cat > /usr/local/bin/cluster-health-check.sh << 'EOF'
#!/bin/bash
# Cluster health monitoring script

LOG_FILE="/var/log/cluster-health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# Check cluster status
CLUSTER_STATUS=$(pvecm status 2>&1)
CLUSTER_NODES=$(pvecm nodes 2>&1)

# Check if all expected nodes are online
EXPECTED_NODES=2
ONLINE_NODES=$(echo "$CLUSTER_NODES" | grep -c "online")

if [ "$ONLINE_NODES" -eq "$EXPECTED_NODES" ]; then
    echo "$DATE: Cluster healthy - all $ONLINE_NODES nodes online" >> "$LOG_FILE"
else
    echo "$DATE: WARNING - Only $ONLINE_NODES/$EXPECTED_NODES nodes online" >> "$LOG_FILE"
    echo "$CLUSTER_NODES" >> "$LOG_FILE"
fi

# Check resource usage
MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100.0}')
CPU_LOAD=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
DISK_USAGE=$(df -h / | grep -E "[0-9]+%" | awk '{print $5}' | sed 's/%//')

echo "$DATE: Resources - Memory: ${MEMORY_USAGE}%, CPU Load: ${CPU_LOAD}, Disk: ${DISK_USAGE}%" >> "$LOG_FILE"

# Keep log size manageable
tail -500 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x /usr/local/bin/cluster-health-check.sh

# Schedule health checks every 5 minutes
echo "*/5 * * * * /usr/local/bin/cluster-health-check.sh" | crontab -

# Test the health check
/usr/local/bin/cluster-health-check.sh
cat /var/log/cluster-health.log
```

## Step 11: Prepare for VM Template Creation

### Download Cloud Images
```bash
# On Node 1 (will be shared via cluster filesystem)
cd /var/lib/vz/template/iso/

# Download Ubuntu 22.04 LTS cloud image
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Download Ubuntu 24.04 LTS cloud image  
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Download Debian 12 cloud image
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2

# Verify downloads
ls -lh /var/lib/vz/template/iso/
```

### Prepare SSH Keys for Templates
```bash
# Create SSH key for VM template access
ssh-keygen -t ed25519 -f /root/.ssh/vm_template_key -N ""

# Display public key for template configuration
cat /root/.ssh/vm_template_key.pub
# Copy this key - you'll need it for cloud-init templates
```

## Validation and Testing

### Cluster Status Validation
```bash
# Comprehensive cluster check
pvecm status
pvecm nodes  
pvesh get /cluster/resources
pvesh get /version

# Check cluster filesystem
df -h /etc/pve
ls -la /etc/pve/

# Verify both nodes can read shared configuration
cat /etc/pve/corosync.conf
```

### Network Connectivity Test
```bash
# Test connectivity between nodes
# From Node 1:
ping 192.168.10.11

# From Node 2:
ping 192.168.10.10

# Test cluster port connectivity
# From Node 1:
nc -zv 192.168.10.11 5405

# Should show: Connection succeeded
```

### Web UI Cluster Management Test
1. **Access both web UIs:**
   - Node 1: https://192.168.10.10:8006
   - Node 2: https://192.168.10.11:8006

2. **Test cluster-wide management:**
   - Login to Node 1's web UI
   - Left panel should show both nodes
   - Click on Node 2 in left panel
   - Should switch to managing Node 2
   - Verify you can see Node 2's resources and status

3. **Test shared configuration:**
   - Make a configuration change on one node
   - Verify it appears on the other node
   - Example: Create a user account on Node 1, check if visible from Node 2

### Resource Pool Verification
```bash
# Check total cluster resources
pvesh get /cluster/resources --type node

# Should show combined resources:
# - Total CPU cores from both nodes
# - Total memory from both nodes  
# - Storage from both nodes
# - Network interfaces from both nodes
```

## Troubleshooting Common Issues

### Cluster Join Failures
```bash
# If Node 2 fails to join cluster:

# Check network connectivity
ping 192.168.10.10  # From Node 2

# Check SSH connectivity  
ssh root@192.168.10.10 "echo test"

# Check if cluster service is running on Node 1
systemctl status pve-cluster

# Check corosync status
systemctl status corosync

# Check for firewall blocking
iptables -L

# Check time synchronization
timedatectl status
# Nodes should have synchronized time
```

### Cluster Communication Issues
```bash
# Check corosync ring status
corosync-quorumtool -s

# Check cluster logs
journalctl -u corosync -f
journalctl -u pve-cluster -f

# Restart cluster services if needed (careful!)
systemctl restart corosync
systemctl restart pve-cluster
```

### Split-Brain Prevention
```bash
# Verify quorum configuration  
pvecm expected 2

# Check quorum status
pvecm status | grep -i quorum

# Should show: Quorum information (expected 2, total 2)
```

## Performance Optimization

### Cluster Performance Tuning
```bash
# Optimize corosync for low-latency network
# Edit corosync configuration (automatically synchronized)
# Web UI: Datacenter → Cluster → Edit

# Or via command line (advanced):
# nano /etc/pve/corosync.conf
# Add under totem section:
# token: 3000
# consensus: 1500
# join: 50
# max_messages: 20
```

### Network Performance Optimization
```bash
# On both nodes, optimize network for virtualization
echo 'net.core.default_qdisc = fq' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_congestion_control = bbr' >> /etc/sysctl.conf
echo 'net.core.netdev_max_backlog = 5000' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 16384 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 16384 134217728' >> /etc/sysctl.conf

# Apply settings
sysctl -p

# Test network performance between nodes
# On Node 2: iperf3 -s
# On Node 1: iperf3 -c 192.168.10.11 -t 30
# Should achieve close to 1Gbps
```

## Documentation

### Record Cluster Configuration
Save in your password manager or documentation system:

```
Cluster Configuration:
- Cluster Name: mumbles-cluster
- Node 1: 192.168.10.10 (pve-node1.mumblescavern.local)
- Node 2: 192.168.10.11 (pve-node2.mumblescavern.local)  
- Cluster Network: 192.168.10.0/24
- Quorum: 2 nodes required
- Migration Network: default (192.168.1.0/24)
- Backup Schedule: Daily at 2:00 AM
- Health Check: Every 5 minutes
```

### Migration Capabilities Matrix
```
Current Migration Support:
- Offline migration: ✅ (shared configuration)
- Online migration: ⚠️ (requires shared storage - next phase)
- Automatic failover: ❌ (requires 3+ nodes and shared storage)

After Storage Setup:
- Offline migration: ✅  
- Online migration: ✅
- Automatic failover: ⚠️ (still requires 3rd node)
```

## Completion Criteria
- [ ] Node 2 Proxmox installation successful
- [ ] Cluster "mumbles-cluster" created with 2 nodes
- [ ] Both nodes showing "online" in cluster status
- [ ] Cluster filesystem operational (/etc/pve shared)
- [ ] Web UI accessible from both nodes
- [ ] Cluster-wide management working
- [ ] SSH connectivity between nodes working
- [ ] Firewall configured for cluster communication
- [ ] Monitoring and backup scripts operational
- [ ] Cloud images downloaded for template creation
- [ ] Network bridges prepared for VLAN deployment

### Pre-Next Phase Checklist
- [ ] Cluster communication stable for 24+ hours
- [ ] No errors in corosync or pve-cluster logs
- [ ] Both nodes accessible via web UI
- [ ] Resource reporting accurate for both nodes
- [ ] Network performance between nodes optimized

**Next Phase**: Proceed to [Storage Setup](04-storage-setup.md) to configure shared storage and enable advanced cluster features like live migration.