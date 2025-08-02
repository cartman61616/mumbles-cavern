# Pre-Phase 0: Dell OptiPlex BIOS Updates and CPU Optimization

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Next Phase**: [Network Setup](01-network-setup.md)  
**Duration**: 1-2 days  
**Prerequisites**: Physical access to all Dell nodes

## Overview
This phase resolves common Dell OptiPlex issues, particularly the 800MHz CPU frequency lock, and prepares hardware for optimal virtualization performance.

## Step 1: Diagnose CPU Frequency Issues

### Common Causes of 800MHz CPU Lock
- Outdated BIOS with power management bugs
- Thermal throttling due to dust/thermal paste issues
- Power adapter not providing sufficient wattage
- Intel SpeedStep/C-States misconfigured
- Windows power management remnants (if migrating from Windows)

### Pre-Update Diagnosis
Boot each Dell node with Ubuntu Live USB and run diagnostics:

```bash
# Check current CPU frequency and thermal state
cat /proc/cpuinfo | grep MHz
# or
sudo cpupower frequency-info

# Check thermal throttling
sudo dmesg | grep -i thermal
sensors  # (may need: sudo apt install lm-sensors)

# Check power management state
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq

# Document current BIOS version
sudo dmidecode -s bios-version
sudo dmidecode -s bios-release-date
```

## Step 2: Download Latest BIOS Updates

### Dell OptiPlex 7050 Micro BIOS
1. Go to Dell Support website: support.dell.com
2. Enter service tag for each node (found on bottom label)
3. Download latest BIOS version for OptiPlex 7050

### Expected Latest Versions (as of 2025)
- OptiPlex 7050: BIOS version 1.21.0 or newer
- File format: `.exe` (Windows) or BIOS Update Utility

### Alternative Download Method
```bash
# Find your exact model and service tag
sudo dmidecode -s system-serial-number  # Service tag
sudo dmidecode -s system-product-name   # Confirm it's OptiPlex 7050

# Use Dell Command Update (Linux version) if available
wget -q -O - https://linux.dell.com/repo/hardware/dsu/public.key | sudo apt-key add -
echo 'deb https://linux.dell.com/repo/hardware/dsu/ubuntu focal main' | sudo tee /etc/apt/sources.list.d/linux.dell.com.sources.list
sudo apt update
sudo apt install dell-system-update
```

## Step 3: BIOS Update Process (Per Node)

### Method 1: USB BIOS Update (Recommended)

#### Prepare Update USB
```bash
# Format USB drive as FAT32
sudo mkfs.vfat /dev/sdX1  # Replace X with your USB drive

# Mount USB drive
sudo mkdir -p /mnt/usb
sudo mount /dev/sdX1 /mnt/usb

# Copy BIOS files to USB root
# Download from Dell website and copy .exe file to USB root
```

#### BIOS Update Steps (For each Dell node)
1. **Power down** Dell node completely
2. **Insert USB** with BIOS update file
3. **Boot to BIOS** (F2 during startup)
4. **Navigate to**: Maintenance → BIOS Update
5. **Select USB device** and choose .exe file
6. **Confirm update** (this takes 5-10 minutes)
7. **DO NOT POWER OFF** during update process
8. **System will reboot** automatically when complete

### Method 2: Linux BIOS Update (If supported)
```bash
# If Dell System Update is available
sudo dsu --category=bios --apply-upgrades

# Or manual method with fwupd (if supported)
sudo apt install fwupd
sudo fwupdmgr get-devices
sudo fwupdmgr get-updates
sudo fwupdmgr update
```

## Step 4: Post-Update BIOS Configuration

### Access BIOS Setup
- Boot and press **F2** repeatedly during Dell logo
- Or **F12** for boot menu → BIOS Setup

### Critical BIOS Settings for Homelab Use

#### Power Management Settings
```
Advanced → Power Management:
├── AC Power Recovery: Power On (auto-start after power loss)
├── Auto Power On: Disabled (unless you want scheduled boots)
├── Deep Sleep Control: Disabled (prevents wake issues)
├── Block Sleep: S3/S4/S5 → Enabled (improves reliability)
└── USB Wake Support: Enabled (for remote management)

Advanced → CPU Configuration:
├── Intel SpeedStep: Enabled (allows dynamic CPU scaling)
├── C-States: Enabled (power efficiency)
├── Intel Turbo Boost: Enabled (performance boost)
├── Hyper-Threading: Enabled (if available on your model)
├── Virtualization Technology (VT-x): Enabled
└── VT for Directed I/O (VT-d): Enabled
```

#### Thermal Management
```
Advanced → Thermal Management:
├── Thermal Management: Enabled
├── Fan Control: Automatic (not Quiet mode)
└── Thermal Profile: Optimized (balanced performance/noise)
```

#### Storage Configuration
```
Advanced → Storage:
├── SATA Operation mode: AHCI (not RAID)
├── NVMe Support: Enabled
└── M.2 Slot Configuration: Enabled (both slots if available)
```

#### Boot Configuration
```
Boot Configuration:
├── UEFI Boot: Enabled
├── Legacy Boot: Disabled (cleaner for Linux)
├── Secure Boot: Disabled (can cause Linux issues)
├── Fast Boot: Disabled (allows easier BIOS access)
└── USB Boot: Enabled
```

#### Security Settings
```
Security:
├── Admin Password: Set strong password and document it
├── Trusted Platform Module (TPM): Enabled (for future use)
└── Computrace: Disabled (not needed for homelab)
```

## Step 5: Post-BIOS Validation

### Boot Test with Live USB
```bash
# After BIOS update, boot with Ubuntu Live USB
# Verify CPU frequency scaling works properly

# Check CPU frequency range
sudo cpupower frequency-info
# Should show proper min/max frequencies (not locked at 800MHz)

# Check available governors
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors
# Should include: conservative, ondemand, performance, powersave

# Test frequency scaling
# Install stress testing tool
sudo apt update && sudo apt install stress

# Monitor frequencies while under load
watch -n 1 "cat /proc/cpuinfo | grep MHz"

# In another terminal, run stress test
sudo stress --cpu 4 --timeout 60s

# Frequencies should scale up under load, then back down
```

### Thermal Validation
```bash
# Install monitoring tools
sudo apt install lm-sensors stress

# Detect sensors
sudo sensors-detect  # Answer 'yes' to safe questions

# Monitor temperatures during stress test
sensors
# CPU temps should be reasonable (under 80°C under load)

# Run extended stress test
sudo stress --cpu 4 --timeout 300s &
watch -n 2 sensors

# No thermal throttling messages should appear in dmesg
dmesg | grep -i thermal
```

## Step 6: Document Configuration

### Create Hardware Documentation
```bash
# For each node, document:
echo "=== Node [1-4] Hardware Info ===" > node-X-info.txt
echo "Service Tag: $(sudo dmidecode -s system-serial-number)" >> node-X-info.txt
echo "BIOS Version: $(sudo dmidecode -s bios-version)" >> node-X-info.txt
echo "BIOS Date: $(sudo dmidecode -s bios-release-date)" >> node-X-info.txt
echo "CPU Model: $(cat /proc/cpuinfo | grep 'model name' | head -1 | cut -d: -f2)" >> node-X-info.txt
echo "CPU Frequency Range: $(sudo cpupower frequency-info | grep 'hardware limits')" >> node-X-info.txt
echo "Available Governors: $(cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors)" >> node-X-info.txt
echo "Memory: $(free -h | grep Mem:)" >> node-X-info.txt
echo "Storage: $(lsblk)" >> node-X-info.txt
echo "Network: $(ip addr show | grep -E 'ether|inet ')" >> node-X-info.txt
```

## Step 7: Performance Validation Checklist

### Per Node Validation
- [ ] **BIOS Version**: Latest available from Dell
- [ ] **CPU Frequency**: Scales properly (not stuck at 800MHz)
- [ ] **Thermal Management**: No throttling under normal load
- [ ] **Virtualization**: VT-x and VT-d enabled
- [ ] **Power Management**: Proper sleep/wake behavior
- [ ] **Storage**: All drives detected properly
- [ ] **Network**: Ethernet auto-negotiates to 1Gbps
- [ ] **USB**: All ports functional
- [ ] **Memory**: Full 32GB detected and tested

### Power Consumption Validation
With Anker Prime charger LCD display:
- Idle: ~15-20W per node
- Light load: ~25-30W per node  
- Full load: ~35W per node (should not exceed this)
- Total cluster idle: ~60-80W
- Total cluster loaded: ~140W

## Troubleshooting Common Issues

### If CPU still stuck at 800MHz after BIOS update

#### Check power adapter
- Ensure Anker Prime is providing adequate power
- Check individual port power limits on LCD display
- Try different USB-C ports

#### Reset power management
```bash
# Boot with Ubuntu Live USB
sudo apt install cpufrequtils

# Set performance governor temporarily
sudo cpufreq-set -r -g performance

# Check if frequencies increase
cat /proc/cpuinfo | grep MHz

# If this works, issue is likely software-related
```

### Thermal issues
- Open case and clean dust from fans/heatsinks
- Check thermal paste condition (may need replacement on older units)
- Ensure adequate ventilation around units

### BIOS corruption recovery
- Dell OptiPlex has BIOS recovery via USB
- Hold Ctrl+Esc during power-on with BIOS file on USB
- System will automatically recover BIOS

## Timeline for BIOS Updates

### Recommended Schedule
- **Day 1**: Update Nodes 1 & 2 (your primary Proxmox nodes)
- **Day 2**: Update Nodes 3 & 4 (validate process before full deployment)
- **Day 3**: Final validation and documentation
- **Day 4**: Begin Proxmox deployment on updated nodes

### Safety Considerations
- Update one node at a time
- Keep one node powered off during updates (in case of issues)
- Document working configurations before making changes
- Have USB recovery tools prepared

## Completion Criteria
- [ ] All nodes updated to latest BIOS
- [ ] CPU frequency scaling working properly on all nodes
- [ ] Thermal management optimized
- [ ] Virtualization features enabled
- [ ] Hardware documentation complete
- [ ] Power consumption within expected ranges

**Next Phase**: Proceed to [Network Setup](01-network-setup.md) to configure VLANs and network infrastructure.