# Phase 0.5: Network Infrastructure Setup (UniFi VLAN Configuration)

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [BIOS Optimization](00-bios-optimization.md)  
**Next Phase**: [Primary Node Setup](02-primary-node.md)  
**Duration**: 4-6 hours  
**Prerequisites**: UniFi Dream Machine Pro accessible, BIOS updates complete

## Overview
Configure network segmentation, VLANs, and gaming optimization to support the entire homelab infrastructure. This phase establishes the network foundation that all subsequent phases will use.

## Step 1: Plan Network Migration Strategy

### Two-Phase Network Approach
Given the complexity of VLAN migration, we'll use a phased approach:

**Phase 1 (Immediate)**: Deploy on default VLAN for simplicity
- Proxmox nodes: 192.168.1.10-13 (default VLAN)
- Services: 192.168.1.15-30 (default VLAN)
- Gaming devices: 192.168.80.x (dedicated gaming VLAN)

**Phase 2 (After core services)**: Migrate to segmented VLANs
- Management: 192.168.10.x (Proxmox, NAS, infrastructure)
- Services: 192.168.20.x (Plex, *arr stack, etc.)
- Gaming: 192.168.80.x (gaming devices and services)

### Network Architecture Overview
```
VLAN 1   (Default): Initial deployment and user devices
VLAN 10  (Management): Infrastructure after migration
VLAN 20  (Services): Production services after migration
VLAN 30  (Storage): High-bandwidth storage traffic
VLAN 50  (IoT): Raspberry Pis, smart devices
VLAN 60  (Guest): Isolated guest network
VLAN 80  (Gaming): Gaming devices and services
VLAN 90  (Work): Development/management devices
```

## Step 2: Configure Base VLANs in UniFi Dream Machine Pro

### Access UniFi Network Controller
1. Browse to your UDM Pro IP (typically https://192.168.1.1)
2. Login with your UniFi credentials
3. Navigate to **Settings** → **Networks**

### Create Gaming VLAN (VLAN 80) - Priority Setup
```
Network Name: Gaming
VLAN ID: 80
Gateway/Subnet: 192.168.80.1/24
DHCP: Enabled
DHCP Range: 192.168.80.100 - 192.168.80.200
Domain Name: gaming.mumblescavern.local
Content Filtering: None
IGMP Snooping: Disabled (for gaming performance)
QoS: Gaming profile (highest priority, lowest latency)
```

### Create Management VLAN (VLAN 10) - For Future Migration
```
Network Name: Management
VLAN ID: 10
Gateway/Subnet: 192.168.10.1/24
DHCP: Enabled
DHCP Range: 192.168.10.100 - 192.168.10.200
Domain Name: mgmt.mumblescavern.local
Content Filtering: None
IGMP Snooping: Enabled
```

### Create Services VLAN (VLAN 20) - For Future Migration
```
Network Name: Services
VLAN ID: 20
Gateway/Subnet: 192.168.20.1/24
DHCP: Enabled
DHCP Range: 192.168.20.100 - 192.168.20.200
Domain Name: svc.mumblescavern.local
Content Filtering: None
IGMP Snooping: Enabled
```

### Create Additional VLANs
```
# Storage VLAN (VLAN 30)
Network Name: Storage
VLAN ID: 30
Gateway/Subnet: 192.168.30.1/24
DHCP: Enabled
DHCP Range: 192.168.30.100 - 192.168.30.150

# IoT VLAN (VLAN 50)
Network Name: IoT-Maker
VLAN ID: 50
Gateway/Subnet: 192.168.50.1/24
DHCP: Enabled
DHCP Range: 192.168.50.100 - 192.168.50.200

# Work VLAN (VLAN 90)
Network Name: Work
VLAN ID: 90
Gateway/Subnet: 192.168.90.1/24
DHCP: Enabled
DHCP Range: 192.168.90.100 - 192.168.90.150
```

## Step 3: Configure Switch Profiles and Port Management

### Dell Cluster Switch Profile (UniFi USW-Lite-16-PoE)
```
Profile Name: Dell-Cluster-Profile
Native VLAN: Default (VLAN 1) - Initial deployment
Tagged VLANs: 
- VLAN 10 (Management) - Proxmox infrastructure
- VLAN 20 (Services) - Application services
- VLAN 30 (Storage) - NAS and storage traffic
- VLAN 80 (Gaming) - Gaming devices and services

Port Configuration:
Port 1: Uplink to UDM Pro (All VLANs trunk)
Port 2: Dell Node 1 (drowzee) - All VLANs
Port 3: Dell Node 2 (sleepy) - All VLANs  
Port 4: Dell Node 3 (snorlax) - Future
Port 5: Dell Node 4 (jigglypuff) - Future
Ports 6-16: Available for expansion
```

### Apply Profile to USW-Lite-16-PoE
1. Go to **Devices** → **Switches**
2. Select your UniFi USW-Lite-16-PoE
3. Click **Port Manager**
4. Set Port 1 as "All" (trunk to UDM Pro)
5. Apply "Dell-Cluster-Profile" to ports 2-5 (All VLANs)
6. Configure individual VLANs as needed for specific workloads

> **⚠️ Important**: UniFi Flex Mini has limited VLAN support and blocks tagged traffic by default. USW-Lite-16-PoE provides full managed switch capabilities required for proper VLAN trunking.

## Step 4: Configure Gaming Network Optimization

### Gaming QoS Configuration
```
QoS Profile: Gaming-Priority
- Gaming Traffic (VLAN 80): Highest priority
- Real-time protocols: High priority (UDP gaming, voice)
- Management Traffic: Medium priority
- Bulk downloads: Lower priority

Traffic Rules:
- Gaming VLAN 80: Highest priority, guaranteed bandwidth
- UDP ports 1024-65535: High priority (gaming)
- TCP ports for game launchers: Medium priority
- ICMP: High priority (ping/latency optimization)
```

### Gaming Device DHCP Reservations
```
# Gaming VLAN (192.168.80.x) Assignments
192.168.80.10: mighty-snorlax (Primary gaming PC)
192.168.80.20: sleepy-deck (Steam Deck)
192.168.80.21: sleepy-deck-dock (Steam Deck dock)
192.168.80.40: dreamy-console-1 (PS5/Xbox/Switch)
192.168.80.41: dreamy-console-2 (Additional consoles)

# Gaming Infrastructure (deployed later)
192.168.80.100: retro-dreamer (RetroPie/Batocera server)
192.168.80.101: game-librarian (ROM organization service)
192.168.80.103: dream-stream-server (Sunshine/Moonlight server)
```

## Step 5: Configure Initial DHCP Reservations

### Default VLAN Reservations (Initial Deployment)
```
# Proxmox Cluster (Initial deployment on default VLAN)
192.168.1.10: pve-node1 (Dell Node 1)
192.168.1.11: pve-node2 (Dell Node 2)
192.168.1.12: pve-node3 (Dell Node 3) - Future
192.168.1.13: pve-node4 (Dell Node 4) - Future

# Infrastructure Services (Initial deployment)
192.168.1.15: shared-infra (PostgreSQL, Redis, shared services)
192.168.1.20: plex-server (Plex Media Server)
192.168.1.21: arr-stack (*arr services)
192.168.1.22: monitoring (Prometheus, Grafana)
192.168.1.23: traefik-cluster (Reverse proxy)

# Storage (Initial deployment)
192.168.1.100: asustor-nas (ASUSTOR NAS - management interface)
192.168.1.101: synology-nas (Synology NAS - management interface)
```

### Personal Device Reservations
```
# Personal devices remain on default VLAN
192.168.1.50: sleepy-air (MacBook Air M3)
192.168.1.51: personal-device-1
192.168.1.52: personal-device-2

# Work devices on work VLAN
192.168.90.10: work-laptop (if connecting to homelab)
```

### IoT Device Reservations
```
# IoT devices on dedicated VLAN
192.168.50.10: sleepyhead-pi (Raspberry Pi)
192.168.50.11: dozer-pi (Raspberry Pi)
192.168.50.12: snuggle-pi (Raspberry Pi)
192.168.50.13: pillow-pi (Raspberry Pi)
192.168.50.20: bambu-p1s (3D Printer)
```

## Step 6: Configure Inter-VLAN Firewall Rules

### Gaming VLAN Security Rules

#### Gaming → Services (ALLOW - Gaming Access)
```
Rule Name: Gaming-to-Services
Action: Allow
Source: Gaming (VLAN 80)
Destination: Default (VLAN 1) - Services on 192.168.1.20-30
Protocol: TCP/UDP
Ports: 32400 (Plex), 8096 (Jellyfin), 8989 (Sonarr), etc.
Description: Allow gaming devices to access media services
```

#### Gaming → Internet (ALLOW - High Priority)
```
Rule Name: Gaming-Internet-Priority
Action: Allow
Source: Gaming (VLAN 80)
Destination: Internet
Protocol: Any
QoS: Highest Priority
Description: Prioritize gaming traffic to internet
```

#### Gaming Internal Communication (ALLOW)
```
Rule Name: Gaming-Internal-Comms
Action: Allow
Source: Gaming (VLAN 80)
Destination: Gaming (VLAN 80)
Protocol: Any
Description: Allow gaming devices to communicate (LAN play, streaming)
```

### Management and Security Rules

#### Default → Services (ALLOW - User Access)
```
Rule Name: Users-to-Services
Action: Allow
Source: Default (VLAN 1)
Destination: Default (VLAN 1) - Services subnet
Protocol: TCP/UDP
Ports: 80, 443, 32400, 8096, common service ports
Description: Allow user devices to access homelab services
```

#### Work → All Infrastructure (ALLOW - Management Access)
```
Rule Name: Work-to-Infrastructure
Action: Allow
Source: Work (VLAN 90)
Destination: Default (VLAN 1), Management (VLAN 10)
Protocol: TCP/UDP
Ports: 22 (SSH), 443 (HTTPS), 8006 (Proxmox), 8080-8090
Description: Allow work devices full infrastructure access
```

#### IoT → Internet Only (BLOCK Internal)
```
Rule Name: IoT-Block-Internal
Action: Block
Source: IoT (VLAN 50)
Destination: LAN Networks (all VLANs except IoT)
Protocol: Any
Description: Block IoT devices from accessing internal networks
```

## Step 7: Configure DNS and Network Services

### DNS Configuration in UniFi
```
# Initial DNS records (default VLAN)
pve-node1.mumblescavern.local → 192.168.1.10
pve-node2.mumblescavern.local → 192.168.1.11
shared-infra.mumblescavern.local → 192.168.1.15
plex.mumblescavern.local → 192.168.1.20
monitoring.mumblescavern.local → 192.168.1.22

# Gaming DNS records
gaming.mumblescavern.local → 192.168.80.103
retro.mumblescavern.local → 192.168.80.100
games.mumblescavern.local → 192.168.80.101
```

## Step 8: Prepare Network Bridges for Proxmox

### Create Bridge Preparation Script
```bash
# Create script to configure Proxmox network bridges
# This will be run during Proxmox installation

cat > /tmp/proxmox-bridge-setup.sh << 'EOF'
#!/bin/bash
# Proxmox network bridge configuration

# Main bridge (default VLAN) - initial deployment
cat >> /etc/network/interfaces << 'BRIDGE_EOF'

# Default bridge for initial deployment
auto vmbr0
iface vmbr0 inet static
    address 192.168.1.10/24  # Node-specific IP
    gateway 192.168.1.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

# Gaming VLAN bridge
auto vmbr80
iface vmbr80 inet manual
    bridge-ports eno1.80
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 80
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward

# Management VLAN bridge (for future migration)
auto vmbr10
iface vmbr10 inet manual
    bridge-ports eno1.10
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 10

# Services VLAN bridge (for future migration)
auto vmbr20
iface vmbr20 inet manual
    bridge-ports eno1.20
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 20

# Storage VLAN bridge
auto vmbr30
iface vmbr30 inet manual
    bridge-ports eno1.30
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 30
BRIDGE_EOF

echo "Proxmox bridges configured for VLAN support"
EOF

chmod +x /tmp/proxmox-bridge-setup.sh
```

## Step 9: Configure NAS Network Interfaces

### ASUSTOR TrueNAS Network Setup
**Initial Configuration (Default VLAN):**
```
Primary Interface:
- IP: 192.168.1.100
- VLAN: 1 (Default)
- Purpose: Initial management and NFS access
```

**Future Migration Plan:**
```
Management Interface:
- IP: 192.168.10.20
- VLAN: 10 (Management)
- Purpose: Web interface, SSH, management

Storage Interface:
- IP: 192.168.30.20
- VLAN: 30 (Storage)
- Purpose: NFS, iSCSI, high-bandwidth storage
```

### Synology NAS Network Setup
**Initial Configuration:**
```
Primary Interface:
- IP: 192.168.1.101
- VLAN: 1 (Default)
- Purpose: Initial setup and backup target
```

## Step 10: Network Performance Testing and Validation

### Create Network Validation Script
```bash
# Create comprehensive network testing script
cat > /tmp/network-validation.sh << 'EOF'
#!/bin/bash
# Network infrastructure validation

echo "=== Network Infrastructure Validation ==="

# Test VLAN gateways
echo "Testing VLAN gateways..."
VLANS=(
    "1:192.168.1.1"
    "10:192.168.10.1"
    "20:192.168.20.1"
    "30:192.168.30.1"
    "50:192.168.50.1"
    "80:192.168.80.1"
    "90:192.168.90.1"
)

for vlan_info in "${VLANS[@]}"; do
    VLAN=$(echo "$vlan_info" | cut -d':' -f1)
    GATEWAY=$(echo "$vlan_info" | cut -d':' -f2)
    
    if ping -c 1 -W 2 "$GATEWAY" > /dev/null 2>&1; then
        echo "✅ VLAN $VLAN Gateway ($GATEWAY): Reachable"
    else
        echo "❌ VLAN $VLAN Gateway ($GATEWAY): Not reachable"
    fi
done

# Test DNS resolution
echo ""
echo "Testing DNS resolution..."
TEST_DOMAINS=(
    "google.com"
    "mumblescavern.local"
)

for domain in "${TEST_DOMAINS[@]}"; do
    if nslookup "$domain" > /dev/null 2>&1; then
        echo "✅ DNS resolution for $domain: Working"
    else
        echo "❌ DNS resolution for $domain: Failed"
    fi
done

# Test inter-VLAN routing
echo ""
echo "Testing inter-VLAN routing..."
echo "This will be tested after services are deployed"

echo ""
echo "=== Network Validation Complete ==="
EOF

chmod +x /tmp/network-validation.sh
```

## Step 11: Create Network Migration Plan

### VLAN Migration Strategy (For Later Implementation)
```bash
# Create VLAN migration script for future use
cat > /tmp/vlan-migration-plan.sh << 'EOF'
#!/bin/bash
# VLAN migration plan - Run after core services are stable

echo "=== VLAN Migration Plan ==="
echo "This script provides the migration strategy for moving from"
echo "default VLAN deployment to segmented VLAN architecture."
echo ""

echo "Migration Steps:"
echo "1. Verify all services are stable on default VLAN"
echo "2. Update DHCP reservations for management VLAN"
echo "3. Migrate Proxmox nodes to management VLAN"
echo "4. Migrate shared infrastructure to management VLAN"
echo "5. Migrate services to services VLAN"
echo "6. Update firewall rules for new VLANs"
echo "7. Test all service connectivity"
echo "8. Update monitoring and external access"
echo ""

echo "Prerequisites for migration:"
echo "- All services operational and stable"
echo "- Backup of all configurations completed"
echo "- Maintenance window scheduled"
echo "- Rollback procedures documented"
echo ""

echo "Migration will be implemented in Phase 9 (Post-Gaming Setup)"
EOF

chmod +x /tmp/vlan-migration-plan.sh
```

## Validation and Testing

### Network Configuration Validation
- [ ] **All VLANs created** in UniFi controller
- [ ] **DHCP scopes configured** for each VLAN
- [ ] **Gaming VLAN optimized** with highest QoS priority
- [ ] **Switch profiles applied** to UniFi Flex Mini
- [ ] **DHCP reservations configured** for known devices
- [ ] **Firewall rules implemented** for inter-VLAN communication
- [ ] **DNS records created** for infrastructure services

### Gaming Network Validation
- [ ] **Gaming VLAN (80) operational** with QoS optimization
- [ ] **Gaming device reservations** configured
- [ ] **Low-latency settings** applied (no IGMP snooping)
- [ ] **Gaming traffic prioritization** configured
- [ ] **Gaming bridge preparation** completed for Proxmox VMs

### Performance Validation
```bash
# Run network validation script
/tmp/network-validation.sh

# Test gaming network performance when devices are connected
# Expected results:
# - Gaming VLAN latency: <1ms local, <5ms to services
# - Inter-VLAN routing: Functional with proper firewall rules
# - QoS prioritization: Gaming traffic gets highest priority
# - DNS resolution: All domains resolve correctly
```

## Troubleshooting Common Issues

### VLAN Communication Problems
```bash
# Check VLAN configuration
ip link show | grep vlan

# Check routing table
ip route show

# Test VLAN tagging
tcpdump -i eno1 -n vlan

# Verify firewall rules
iptables -L -n -v
```

### Gaming Performance Issues
```bash
# Check QoS configuration in UniFi
# Monitor traffic classification
# Verify gaming devices are on correct VLAN
# Test latency between gaming VLAN and services
```

### Switch Configuration Issues
```bash
# Verify switch port configuration in UniFi
# Check for VLAN tagging on trunk ports
# Ensure proper profile application
# Monitor switch port utilization
```

## Completion Criteria
- [ ] **All VLANs operational** with proper IP assignments
- [ ] **Gaming network optimized** for lowest latency
- [ ] **DHCP reservations active** for all planned devices
- [ ] **Firewall rules configured** and tested
- [ ] **DNS resolution working** for all domains
- [ ] **Switch profiles applied** and functional
- [ ] **Network performance** meeting expectations
- [ ] **Migration plan documented** for future VLAN segmentation

### Pre-Next Phase Checklist
- [ ] Gaming VLAN (80) fully operational
- [ ] Default VLAN ready for Proxmox deployment
- [ ] All device MAC addresses documented
- [ ] Network validation script passes all tests
- [ ] UniFi configuration backed up
- [ ] Migration strategy documented

**Next Phase**: Proceed to [Primary Node Setup](02-primary-node.md) to install Proxmox on your first Dell node using the network infrastructure you just configured.
