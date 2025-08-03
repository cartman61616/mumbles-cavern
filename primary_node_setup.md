# Phase 1: Primary Node Deployment (Node 1)

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [Network Setup](01-network-setup.md)  
**Next Phase**: [Cluster Setup](03-cluster-setup.md)  
**Duration**: 2-3 hours  
**Prerequisites**: BIOS optimized, network VLANs configured

## Overview
Install and configure the first Proxmox node, which will become the foundation of your cluster.

## Step 1: Hardware Assessment and Documentation

### Storage Audit (Do this first on each node)
Boot each Dell node with a Linux live USB (Ubuntu/Pop!_OS works well) and document storage:

```bash
lsblk -f
# This shows all drives with filesystems

fdisk -l
# Shows partition tables and exact drive sizes

smartctl -a /dev/nvme0n1
# Shows NVMe drive health and exact model (if nvme drive exists)

smartctl -a /dev/sda
# Shows SATA drive health and model
```

### Create Hardware Inventory Document
```
Node 1 (7500T): 
- SATA SSD: [Size] [Model] [Health Status]
- NVMe: [Size] [Model] [Health Status]
- RAM: 32GB
- Network MAC: [Record this]

Node 2 (7500T):
- [Same format]

Node 3 (7500T):
- [Same format]

Node 4 (6500T):
- [Same format]
```

## Step 2: Download and Prepare Installation Media

### Download Proxmox VE ISO
```bash
# On your local machine, download latest Proxmox VE
wget https://www.proxmox.com/en/downloads/category/iso-images-pve
# Current version: proxmox-ve_8.2-1.iso (verify latest on website)
```

### Upload ISO to JetKVM Storage
- Upload Proxmox VE ISO to JetKVM's virtual storage
- Verify SHA256 checksum of downloaded ISO
- Configure JetKVM to mount ISO as virtual CD/DVD drive

### Prepare JetKVM and Installation Notes
```
JetKVM Setup:
- Connect JetKVM to Node 1 via USB-C/HDMI
- Upload Proxmox ISO to JetKVM storage
- Configure virtual CD/DVD mount
- Verify remote KVM access working

Each node will need:
- Root password: [Use strong password, document in password manager]
- Email: your-email@domain.com
- Timezone: America/New_York (or your timezone)
- Keyboard: us
- Network: Static IP as planned in network setup
```

## Step 3: Hardware Preparation

### Physical Setup
1. Connect Node 1 to UniFi Flex Mini switch
2. Connect Anker Prime USB-C to barrel jack power
3. Connect JetKVM to Node 1 (USB-C + HDMI)
4. Verify power LED and network activity
5. Access Node 1 remotely via JetKVM web interface

### BIOS Configuration (Final Check)
1. Boot and press F2/F12 to enter BIOS
2. Verify these settings are enabled:
   ```
   Advanced → CPU Configuration:
   - Intel Virtualization Technology: Enabled
   - VT-d: Enabled
   
   Advanced → Storage:
   - SATA Mode: AHCI
   - NVMe Support: Enabled
   
   Security:
   - Secure Boot: Disabled
   
   Boot:
   - Legacy Boot: Disabled  
   - UEFI Boot: Enabled
   - USB Boot: Enabled
   ```
3. Save and exit

## Step 4: Proxmox Installation

### Boot from JetKVM Virtual Storage
1. Mount Proxmox ISO via JetKVM virtual CD/DVD
2. Power cycle Node 1 via JetKVM or physically
3. Boot and select "Install Proxmox VE" from virtual CD/DVD
4. Accept license agreement

### Target Disk Selection (CRITICAL)
```
If Node 1 has SSD + NVMe:
- Install Proxmox OS on SATA SSD (usually /dev/sda)
- Leave NVMe untouched for VM storage pool
- File system: ext4 (reliable for OS)

If only one drive available:
- Use entire drive for now
- Plan to add second drive later
```

### Network Configuration
```
IP Address: 192.168.10.10
Netmask: 255.255.255.0 (/24)
Gateway: 192.168.0.1 (your UDM Pro)
DNS: 192.168.0.1
Hostname: pve-node1.mumblescavern.local
```

### System Configuration
```
Timezone: America/New_York (or appropriate)
Root Password: [Strong password - document this!]
Email: your-email@domain.com
```

### Complete Installation
1. Verify all settings on summary screen
2. Click "Install"
3. Installation takes 5-10 minutes
4. Unmount virtual CD/DVD via JetKVM when prompted
5. Reboot

## Step 5: Initial Configuration

### First Boot and Web Access
1. Note the IP address shown on console: https://192.168.10.10:8006
2. From your management computer, browse to https://192.168.10.10:8006
3. Accept self-signed certificate warning
4. Login: root / [your password]

### Update System
```bash
# SSH to node or use web shell
ssh root@192.168.10.10

# Update package lists
apt update

# Upgrade system
apt full-upgrade -y

# Reboot if kernel updated
reboot
```

### Configure Storage (If NVMe available)
```bash
# Check available storage
lsblk

# If you have unused NVMe (e.g., /dev/nvme0n1):
# Create storage pool for VMs
pvecreate /dev/nvme0n1
vgcreate vm-storage /dev/nvme0n1
lvcreate -l 100%FREE -n vm-data vm-storage

# Add to Proxmox storage config via web UI:
# Datacenter → Storage → Add → LVM
# ID: vm-nvme-storage
# Volume group: vm-storage
# Content: Disk image, Container
```

### Web UI Configuration
1. **Datacenter → Cluster**: Note cluster information (don't create cluster yet)
2. **System → Network**: Verify network configuration
3. **System → Updates**: Ensure all updates applied
4. **Local Storage**: Configure storage pools

## Step 6: Network Bridge Configuration

### Configure Network Bridges for VLANs
```bash
# Edit network configuration
nano /etc/network/interfaces

# Add VLAN-aware bridge configuration:
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.0.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Apply network configuration
ifreload -a
```

### Web UI Network Configuration
1. Go to **System** → **Network**
2. Verify vmbr0 shows as "VLAN Aware: Yes"
3. Test connectivity to gateway: `ping 192.168.0.1`

## Step 7: Install Essential Packages

### Install Monitoring and Management Tools
```bash
# Install useful packages
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

# Enable node exporter for monitoring
systemctl enable prometheus-node-exporter
systemctl start prometheus-node-exporter

# Verify node exporter is running
curl localhost:9100/metrics | head -20
```

### Configure SSH Keys (Optional but recommended)
```bash
# Create SSH directory if not exists
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Add your public key (replace with your actual key)
echo "ssh-rsa AAAAB3NzaC1yc2E... your-email@domain.com" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Test SSH key login from your workstation
# ssh root@192.168.10.10
```

## Step 8: Configure Backup Storage

### Create Backup Directory
```bash
# Create local backup storage
mkdir -p /backup-local
chmod 755 /backup-local

# Add via Web UI: 
# Datacenter → Storage → Add → Directory
# ID: local-backup
# Directory: /backup-local  
# Content: VZDump backup file
# Enabled: Yes
# Shared: No
```

### Configure Automatic Backup Retention
```bash
# Create backup cleanup script
cat > /usr/local/bin/cleanup-backups.sh << 'EOF'
#!/bin/bash
# Clean up backups older than 7 days
find /backup-local -name "*.vma.gz" -mtime +7 -delete
find /backup-local -name "*.tar.gz" -mtime +7 -delete
find /backup-local -name "*.tar.lzo" -mtime +7 -delete
EOF

chmod +x /usr/local/bin/cleanup-backups.sh

# Add to crontab for daily execution
echo "0 3 * * * /usr/local/bin/cleanup-backups.sh" | crontab -
```

## Step 9: Initial Security Hardening

### Configure Firewall
```bash
# Enable Proxmox firewall
# Web UI: Datacenter → Firewall → Options
# Enable: Yes

# Configure basic rules via Web UI:
# Datacenter → Firewall → Add
# Direction: in
# Action: ACCEPT
# Protocol: tcp
# Dest. port: 8006 (Proxmox Web UI)
# Source: 192.168.0.0/24,192.168.10.0/24,192.168.90.0/24
# Comment: Allow Proxmox web access from management networks
```

### Disable Unused Services
```bash
# Check running services
systemctl list-unit-files --state=enabled

# Disable unnecessary services (be careful!)
systemctl disable postfix  # If not using email alerts
systemctl stop postfix
```

## Step 10: Performance Tuning

### CPU Governor Configuration
```bash
# Set CPU governor for virtualization workload
echo 'GOVERNOR="ondemand"' > /etc/default/cpufrequtils

# Apply immediately
cpufreq-set -r -g ondemand

# Verify governor is set
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
```

### Memory and Swap Configuration
```bash
# Check current memory usage
free -h

# Configure swap (if not already optimal)
# For 32GB RAM, 2-4GB swap is sufficient
swapon --show

# Adjust swappiness for server workload
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p
```

## Validation and Testing

### System Health Check
```bash
# Check system resources
htop

# Check disk usage
df -h
ncdu /

# Check network connectivity
ping 8.8.8.8
ping 192.168.0.1

# Check virtualization support
egrep -c '(vmx|svm)' /proc/cpuinfo
# Should return number > 0

# Check VT-d support  
dmesg | grep -i dmar
# Should show DMAR/IOMMU initialization
```

### Web UI Functionality Test
- [ ] Login successful at https://192.168.10.10:8006
- [ ] All system information displayed correctly
- [ ] Storage pools visible and healthy
- [ ] Network configuration matches plan
- [ ] System updates completed
- [ ] No critical errors in logs

### Performance Baseline
```bash
# CPU performance test
stress --cpu 4 --timeout 60s &
watch -n 1 "cat /proc/cpuinfo | grep MHz"

# Memory test
stress --vm 2 --vm-bytes 4G --timeout 60s

# Storage I/O test (be careful on production systems)
dd if=/dev/zero of=/tmp/test.img bs=1M count=1024 oflag=dsync
rm /tmp/test.img

# Network throughput (if you have another machine)
# On another machine: iperf3 -s
# On Proxmox node: iperf3 -c [other-machine-ip]
```

## Documentation

### Record Configuration Details
Save these details in your password manager or documentation system:

```
Node 1 Configuration:
- IP Address: 192.168.10.10
- Root Password: [password]
- SSH Keys: [location of private key]
- Storage Configuration: [local storage + NVMe pool]
- BIOS Version: [version from Step 1]
- MAC Address: [for DHCP reservations]
- Service Tag: [for Dell support]
```

## Troubleshooting Common Issues

### Installation Issues
- **Installer hangs**: Check RAM seating, try with single RAM stick
- **Network not detected**: Verify Ethernet cable, check switch port
- **Storage not detected**: Verify SATA cables, check BIOS storage settings

### Post-Installation Issues
- **Web UI not accessible**: Check firewall, verify IP configuration
- **Poor performance**: Verify CPU governor, check thermal throttling
- **Storage errors**: Check disk health with smartctl

### Network Connectivity Problems
```bash
# Test basic connectivity
ping 192.168.0.1  # Gateway

# Check network interface
ip addr show

# Check routing table
ip route show

# Test DNS resolution
nslookup google.com
```

### JetKVM-Specific Troubleshooting
- **JetKVM not accessible**: Verify JetKVM power and network connectivity
- **Virtual CD/DVD not mounting**: Check ISO file integrity and JetKVM storage
- **Boot priority issues**: Ensure UEFI boot order includes virtual CD/DVD
- **Remote console lag**: Check network latency between management machine and JetKVM
- **Installation media not detected**: Verify ISO is properly mounted in JetKVM interface
- **Power control issues**: Use JetKVM power management or physical power cycle

## Completion Criteria
- [ ] Proxmox VE installed and accessible via web UI
- [ ] System fully updated
- [ ] Storage configured appropriately
- [ ] Network connectivity validated
- [ ] SSH access working (if configured)
- [ ] Backup storage configured
- [ ] Performance baseline established
- [ ] Configuration documented

**Next Phase**: Proceed to [Cluster Setup](03-cluster-setup.md) to install the second node and form your cluster.