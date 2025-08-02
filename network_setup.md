# Phase 0.5: Network Infrastructure Setup (UniFi VLAN Configuration)

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [BIOS Optimization](00-bios-optimization.md)  
**Next Phase**: [Primary Node Setup](02-primary-node.md)  
**Duration**: 4-6 hours  
**Prerequisites**: UniFi Dream Machine Pro accessible, BIOS updates complete

## Overview
Configure network segmentation, VLANs, and gaming optimization to support the entire homelab infrastructure.

## Step 1: Plan VLAN Architecture

### VLAN Design for Mumbles Cavern Homelab
```
VLAN 1   (Default/Untagged): Management and trusted devices
VLAN 10  (Management): Infrastructure (Proxmox, NAS, switches)
VLAN 20  (Services): Production services (Plex, *arr stack)
VLAN 30  (Storage): High-bandwidth storage traffic (NFS, iSCSI)
VLAN 40  (Development): Testing and development VMs
VLAN 50  (IoT/Edge): Raspberry Pis, smart devices
VLAN 60  (Guest): Isolated guest network
VLAN 70  (Media): Dedicated media streaming traffic
VLAN 80  (Gaming): Gaming devices and services
VLAN 90  (Work): Development/management devices
```

### IP Address Allocation
```
VLAN 1  (Default):     192.168.1.0/24   (Current network)
VLAN 10 (Management):  192.168.10.0/24  (Infrastructure)
VLAN 20 (Services):    192.168.20.0/24  (Production apps)
VLAN 30 (Storage):     192.168.30.0/24  (Storage traffic)
VLAN 40 (Development): 192.168.40.0/24  (Dev/Test)
VLAN 50 (IoT):         192.168.50.0/24  (Edge devices)
VLAN 60 (Guest):       192.168.60.0/24  (Guest access)
VLAN 70 (Media):       192.168.70.0/24  (Media streaming)
VLAN 80 (Gaming):      192.168.80.0/24  (Gaming devices)
VLAN 90 (Work):        192.168.90.0/24  (Work devices)
```

## Step 2: Configure VLANs in UniFi Dream Machine Pro

### Access UniFi Network Controller
1. Browse to your UDM Pro IP (typically https://192.168.1.1)
2. Login with your UniFi credentials
3. Navigate to **Settings** → **Networks**

### Create Management VLAN (VLAN 10)
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

### Create Services VLAN (VLAN 20)
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

### Create Storage VLAN (VLAN 30)
```
Network Name: Storage
VLAN ID: 30
Gateway/Subnet: 192.168.30.1/24
DHCP: Enabled
DHCP Range: 192.168.30.100 - 192.168.30.150
Domain Name: storage.mumblescavern.local
Content Filtering: None
IGMP Snooping: Disabled (for storage performance)
```

### Create Development VLAN (VLAN 40)
```
Network Name: Development
VLAN ID: 40
Gateway/Subnet: 192.168.40.1/24
DHCP: Enabled
DHCP Range: 192.168.40.100 - 192.168.40.200
Domain Name: dev.mumblescavern.local
Content Filtering: None
IGMP Snooping: Enabled
```

### Create IoT/Maker VLAN (VLAN 50)
```
Network Name: IoT-Maker
VLAN ID: 50
Gateway/Subnet: 192.168.50.1/24
DHCP: Enabled
DHCP Range: 192.168.50.100 - 192.168.50.200
Domain Name: iot.mumblescavern.local
Content Filtering: Enabled (restrict internet access for security)
IGMP Snooping: Enabled
QoS: Medium Priority
```

### Create Guest VLAN (VLAN 60)
```
Network Name: Guest
VLAN ID: 60
Gateway/Subnet: 192.168.60.1/24
DHCP: Enabled
DHCP Range: 192.168.60.100 - 192.168.60.200
Domain Name: guest.mumblescavern.local
Content Filtering: Enabled
Guest Network: Enabled (isolate from other networks)
```

### Create Gaming VLAN (VLAN 80)
```
Network Name: Gaming
VLAN ID: 80
Gateway/Subnet: 192.168.80.1/24
DHCP: Enabled
DHCP Range: 192.168.80.100 - 192.168.80.200
Domain Name: gaming.mumblescavern.local
Content Filtering: None
IGMP Snooping: Disabled (for gaming performance)
QoS: Gaming profile (high priority, low latency)
```

### Create Work VLAN (VLAN 90)
```
Network Name: Work
VLAN ID: 90
Gateway/Subnet: 192.168.90.1/24
DHCP: Enabled
DHCP Range: 192.168.90.100 - 192.168.90.150
Domain Name: work.mumblescavern.local
Content Filtering: None
IGMP Snooping: Enabled
QoS: High Priority (development/management traffic)
```

## Step 3: Configure Gaming-Optimized Switch Profiles and QoS

### Gaming Network Optimization
```
QoS Profile: Gaming-Priority
- Gaming Traffic: Highest priority
- Real-time protocols: High priority (UDP gaming, voice)
- Streaming: Medium-high priority
- Bulk downloads: Lower priority

Traffic Rules:
- Gaming VLAN 80: Highest priority
- UDP ports 1024-65535: High priority (gaming)
- TCP ports for game launchers: Medium priority
- ICMP: High priority (ping/latency)
```

## Step 4: Device Assignment Strategy

### Gaming VLAN (VLAN 80 - Dream Realm) - Personal Gaming Devices
```
192.168.80.10: beast-rig (Ryzen 5600X, RTX 3070, 64GB RAM - main gaming PC)
192.168.80.20: sleepy-deck (Steam Deck)
192.168.80.21: sleepy-deck-dock (if using dock)
192.168.80.40: dreamy-console-1 (PS5/Xbox/Switch)
192.168.80.41: dreamy-console-2 (additional consoles)
192.168.80.60: drowsy-tablet (iPad/Android gaming)
```

### Work VLAN (VLAN 90 - Command Center) - Development/Management
```
192.168.90.10: work-laptop (Actual work laptop - if connecting to homelab)
192.168.90.11: admin-device (Additional management device if needed)
```

### Personal VLAN (VLAN 1 - Default) - Daily Use Device
```
192.168.1.50: sleepy-air (MacBook Air M3 - personal daily use laptop)
192.168.1.51: personal-devices (other personal devices)
```

### Homelab Gaming Infrastructure (VLAN 80 - Dream Realm)
```
192.168.80.100: retro-dreamer (RetroPie/Batocera server - on Snorlax Prime)
192.168.80.101: game-librarian (ROM organization service)
192.168.80.102: save-keeper (save state management)
192.168.80.103: dream-stream-server (Sunshine/Moonlight on Snorlax Prime)
```

### Work Devices (if connecting to homelab)
```
MAC: [work-laptop-mac] → IP: 192.168.90.10 → Name: work-laptop
```

## Step 5: Configure Switch Profiles

### Dell Cluster Switch Profile (UniFi Flex Mini)
```
Profile Name: Dell-Cluster-Profile
Native VLAN: Management (VLAN 10)
Tagged VLANs: 
- VLAN 20 (Services)
- VLAN 30 (Storage)
- VLAN 40 (Development)
- VLAN 70 (Media)

Port Configuration:
Port 1-4: Dell OptiPlex nodes
Port 5: Uplink to main network
```

### Apply Profile to UniFi Flex Mini
1. Go to **Devices** → **Switches**
2. Select your UniFi Flex Mini
3. Click **Port Manager**
4. Apply "Dell-Cluster-Profile" to ports 1-4
5. Set port 5 as "All" (trunk port to main switch)

## Step 6: Configure DHCP Reservations

### Infrastructure Devices (VLAN 10 - Management)
```
192.168.10.10: pve-node1 (Dell Node 1)
192.168.10.11: pve-node2 (Dell Node 2)
192.168.10.12: pve-node3 (Dell Node 3)
192.168.10.13: pve-node4 (Dell Node 4)
192.168.10.15: shared-infra (Shared services VM)
192.168.10.20: asustor-nas (ASUSTOR TrueNAS)
192.168.10.21: synology-nas (Synology NAS)
```

### Service VMs (VLAN 20 - Services)
```
192.168.20.20: plex-server
192.168.20.21: arr-stack
192.168.20.22: monitoring
192.168.20.23: traefik-cluster
192.168.20.25: nextcloud
192.168.20.26: immich
192.168.20.27: home-assistant
```

### Storage Network (VLAN 30)
```
192.168.30.20: asustor-storage (Storage interface)
192.168.30.21: synology-storage (Storage interface)
```

### Gaming Device DHCP Reservations

#### Find device MAC addresses
```bash
# On each gaming device, find MAC address:
# Windows: ipconfig /all
# Linux: ip link show
# Steam Deck: Settings → Internet → Advanced → Hardware Address
# Consoles: Network settings → View connection details
```

#### Create DHCP reservations in UniFi
```
Go to: Settings → Networks → Gaming → DHCP
Add reservations:

MAC: [beast-rig-mac] → IP: 192.168.80.10 → Name: beast-rig
MAC: [sleepy-deck-mac] → IP: 192.168.80.20 → Name: sleepy-deck  
MAC: [dreamy-console1-mac] → IP: 192.168.80.40 → Name: dreamy-ps5
MAC: [dreamy-console2-mac] → IP: 192.168.80.41 → Name: dreamy-xbox
MAC: [sleepy-switch-mac] → IP: 192.168.80.42 → Name: drowsy-switch
```

#### Personal devices (daily use - default VLAN)
```
MAC: [sleepy-air-mac] → IP: 192.168.1.50 → Name: sleepy-air
MAC: [personal-device-mac] → IP: 192.168.1.51 → Name: personal-device
```

#### IoT and 3D Printing devices
```
MAC: [bambu-p1s-mac] → IP: 192.168.50.20 → Name: bambu-p1s
MAC: [sleepyhead-pi-mac] → IP: 192.168.50.10 → Name: sleepyhead-pi
MAC: [dozer-pi-mac] → IP: 192.168.50.11 → Name: dozer-pi
MAC: [snuggle-pi-mac] → IP: 192.168.50.12 → Name: snuggle-pi
MAC: [pillow-pi-mac] → IP: 192.168.50.13 → Name: pillow-pi
```

## Step 7: Configure Gaming-Specific Firewall Rules

### Gaming VLAN Firewall Configuration

#### Default → Services (ALLOW - Personal Access)
```
Rule Name: Personal-to-Services
Action: Allow
Source: Default (VLAN 1)
Destination: Services (VLAN 20)
Protocol: TCP/UDP
Ports: 80, 443, 32400 (Plex), 8096 (Jellyfin), common service ports
Description: Allow personal devices to access homelab services
```

#### Work → All Infrastructure (ALLOW - Management Access)
```
Rule Name: Work-to-Infrastructure
Action: Allow
Source: Work (VLAN 90)
Destination: Management (VLAN 10), Services (VLAN 20), Development (VLAN 40)
Protocol: TCP/UDP
Ports: 22 (SSH), 443 (HTTPS), 8006 (Proxmox), 8080-8090 (various services)
Description: Allow work laptop full access to homelab infrastructure
```

#### Gaming → Services (ALLOW - Gaming Services)
```
Rule Name: Gaming-to-Services
Action: Allow
Source: Gaming (VLAN 80)
Destination: Services (VLAN 20)
Protocol: TCP/UDP
Ports: 32400 (Plex), 8096 (Jellyfin), game streaming ports
Description: Allow gaming devices to access media services
```

#### Gaming → Internet (ALLOW - High Priority)
```
Rule Name: Gaming-Internet-Priority  
Action: Allow
Source: Gaming (VLAN 80)
Destination: Internet
Protocol: Any
QoS: High Priority
Description: Prioritize gaming traffic to internet
```

#### Gaming Inter-Device Communication (ALLOW)
```
Rule Name: Gaming-Internal-Comms
Action: Allow  
Source: Gaming (VLAN 80)
Destination: Gaming (VLAN 80)
Protocol: Any
Description: Allow gaming devices to communicate (LAN play, streaming)
```

#### Block Gaming → Management/Storage
```
Rule Name: Gaming-Block-Infrastructure
Action: Block
Source: Gaming (VLAN 80)  
Destination: Management (VLAN 10), Storage (VLAN 30)
Protocol: Any
Description: Prevent gaming devices from accessing infrastructure
```

### Inter-VLAN Communication Rules

#### Management → All VLANs (ALLOW)
```
Rule Name: Management-to-All
Action: Allow
Source: Management (VLAN 10)
Destination: Any
Protocol: Any
Description: Allow management access to all networks
```

#### Services → Storage (ALLOW)
```
Rule Name: Services-to-Storage
Action: Allow
Source: Services (VLAN 20)
Destination: Storage (VLAN 30)
Protocol: Any
Description: Allow services to access storage
```

#### Development → Services (ALLOW - Limited)
```
Rule Name: Dev-to-Services
Action: Allow
Source: Development (VLAN 40)
Destination: Services (VLAN 20)
Protocol: TCP
Ports: 80, 443, 8080, 9090
Description: Allow dev access to service web interfaces
```

#### IoT → Internet Only (BLOCK Internal)
```
Rule Name: IoT-Block-Internal
Action: Block
Source: IoT (VLAN 50)
Destination: LAN Networks
Protocol: Any
Description: Block IoT devices from accessing internal networks
```

#### Guest → Internet Only (BLOCK Internal)
```
Rule Name: Guest-Block-Internal
Action: Block
Source: Guest (VLAN 60)
Destination: LAN Networks
Protocol: Any
Description: Block guest devices from accessing internal networks
```

## Step 8: Update DNS Configuration

### Create DNS Records in UniFi
```
# Management Network
pve-node1.mgmt.mumblescavern.local → 192.168.10.10
pve-node2.mgmt.mumblescavern.local → 192.168.10.11
pve-node3.mgmt.mumblescavern.local → 192.168.10.12
pve-node4.mgmt.mumblescavern.local → 192.168.10.13
shared-infra.mgmt.mumblescavern.local → 192.168.10.15

# Service Network
plex.svc.mumblescavern.local → 192.168.20.20
sonarr.svc.mumblescavern.local → 192.168.20.21
radarr.svc.mumblescavern.local → 192.168.20.21
grafana.svc.mumblescavern.local → 192.168.20.22
traefik.svc.mumblescavern.local → 192.168.20.23

# Storage Network
asustor.storage.mumblescavern.local → 192.168.30.20
synology.storage.mumblescavern.local → 192.168.30.21
```

## Step 9: Configure NAS Network Interfaces

### ASUSTOR TrueNAS Network Setup
```
Primary Interface (Management):
- IP: 192.168.10.20
- VLAN: 10 (Management)
- Purpose: Web interface, SSH, management

Storage Interface (Dedicated):
- IP: 192.168.30.20
- VLAN: 30 (Storage)
- Purpose: NFS, iSCSI, high-bandwidth storage
```

### Synology NAS Network Setup
```
Primary Interface (Management):
- IP: 192.168.10.21
- VLAN: 10 (Management)
- Purpose: Web interface, SSH, management

Storage Interface (Dedicated):
- IP: 192.168.30.21
- VLAN: 30 (Storage)
- Purpose: SMB, backup target, media storage
```

## Step 10: Test Network Connectivity

### Connectivity Test Matrix
```bash
# From management workstation, test VLAN routing:

# Test Management VLAN
ping 192.168.10.1  # Gateway
nslookup pve-node1.mgmt.mumblescavern.local

# Test Services VLAN
ping 192.168.20.1  # Gateway

# Test Storage VLAN  
ping 192.168.30.1  # Gateway

# Test inter-VLAN routing
# Should work: Management → Services
ping 192.168.20.1  # From VLAN 10 to VLAN 20

# Should be blocked: Guest → Management
# (Test from guest device)
```

### Performance Testing
```bash
# Test storage network performance
# From future Proxmox node to storage VLAN:
iperf3 -s  # On storage device
iperf3 -c 192.168.30.20 -t 30  # From client

# Should achieve near-gigabit speeds on storage VLAN
# Management traffic should not impact storage performance
```

## Step 11: Proxmox Network Configuration Plan

### Bridge Configuration (To be implemented during Proxmox install)
```
vmbr0 (Default):
- Bridge for VLAN 10 (Management)
- Proxmox host management traffic

vmbr10 (Management):
- Tagged VLAN 10
- Infrastructure VMs

vmbr20 (Services):
- Tagged VLAN 20
- Production service VMs

vmbr30 (Storage):
- Tagged VLAN 30
- Storage traffic only

vmbr40 (Development):
- Tagged VLAN 40
- Development and testing VMs
```

## Validation Checklist

### Network Validation
- [ ] All VLANs created in UniFi controller
- [ ] DHCP scopes configured for each VLAN
- [ ] Firewall rules implemented and tested
- [ ] DNS records created for infrastructure
- [ ] Switch profiles applied to UniFi Flex Mini
- [ ] DHCP reservations configured
- [ ] Inter-VLAN routing tested
- [ ] NAS devices accessible on both management and storage VLANs

### Gaming Optimization Validation
- [ ] Gaming VLAN QoS configured for lowest latency
- [ ] Gaming devices assigned to VLAN 80
- [ ] High priority traffic rules active
- [ ] Inter-gaming device communication working
- [ ] Gaming traffic isolated from infrastructure

## Troubleshooting

### VLAN Connectivity Issues
```bash
# Check VLAN tagging on switch ports
# Verify DHCP scope assignments
# Test with static IP assignment if DHCP fails
# Check firewall rule order (most specific first)
```

### Performance Issues
```bash
# Verify QoS settings are applied
# Check for IGMP snooping conflicts
# Test with simplified firewall rules
# Monitor switch port utilization
```

## Beast Rig (Gaming PC) Setup

### Network Adapter Configuration
```
Windows: Network & Internet → Ethernet → Change adapter options
- Set static IP: 192.168.80.10 or verify DHCP reservation
- DNS: 192.168.80.1 or 1.1.1.1
- Test latency: ping 8.8.8.8
```

### Game Launcher Optimization
```
Steam: Settings → Downloads → Download Region (nearest)
Epic Games: Settings → Downloads → Throttle Downloads (disable)
Battle.net: Settings → Game Install/Update → Network Bandwidth
```

## Completion Criteria
- [ ] All VLANs operational with proper IP assignments
- [ ] Firewall rules tested and working
- [ ] Gaming network optimized for low latency
- [ ] NAS devices accessible via both management and storage networks
- [ ] DNS resolution working for all planned hostnames
- [ ] DHCP reservations active for all devices
- [ ] Network performance meeting expectations

**Next Phase**: Proceed to [Primary Node Setup](02-primary-node.md) to install Proxmox on your first Dell node.