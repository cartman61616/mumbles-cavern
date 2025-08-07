# Network & VLAN Migration Plan
**From Default Network to Proper VLAN Segmentation**

## Current State
```yaml
Current Network Setup:
  - Everything on default VLAN 1: 192.168.1.x (or 192.168.0.x)
  - ASUSTOR NAS: 192.168.0.93
  - Proxmox nodes: Likely 192.168.1.x or 192.168.0.x
  - All devices on single flat network

Target Network Architecture:
  - VLAN 1 (Default): 192.168.1.x - Personal devices, printer
  - VLAN 10 (Management): 192.168.10.x - Infrastructure (Proxmox, NAS)
  - VLAN 20 (Services): 192.168.20.x - Homelab services (Plex, VMs)
  - VLAN 30 (Work): 192.168.30.x - Work devices (direct internet)
  - VLAN 50 (IoT): 192.168.50.x - Smart home devices (VPN + isolated)
  - VLAN 80 (Gaming): 192.168.80.x - Gaming devices (VPN protected)
```

## Migration Strategy: Phased Approach

### Phase 1: Create VLANs in UniFi (30 minutes)
```yaml
UniFi Configuration Steps:
1. Create new networks in UniFi controller:
   - Management (VLAN 10): 192.168.10.0/24
   - Services (VLAN 20): 192.168.20.0/24
   - Work (VLAN 30): 192.168.30.0/24
   - IoT (VLAN 50): 192.168.50.0/24
   - Gaming (VLAN 80): 192.168.80.0/24

2. Configure DHCP pools:
   - VLAN 10: 192.168.10.100-199
   - VLAN 20: 192.168.20.100-199
   - VLAN 30: 192.168.30.100-199
   - VLAN 50: 192.168.50.100-199
   - VLAN 80: 192.168.80.100-199

3. Create WiFi networks:
   - "Snorlax-Services" → VLAN 20
   - "Snorlax-Work" → VLAN 30
   - "Snorlax-Gaming" → VLAN 80
   - Keep existing WiFi on VLAN 1 for transition
```

### Phase 2: Infrastructure Device Migration (1 hour)
```yaml
Device Migration Order (Critical First):
1. Proxmox Nodes → VLAN 10 (Management)
2. ASUSTOR NAS → VLAN 10 (Management) 
3. Synology NAS → VLAN 10 (Management)
4. Network equipment stays on VLAN 1 (UniFi management)

Reserved IP Assignments:
  - 192.168.10.10: drowzee.mumblescavern.local
  - 192.168.10.11: sleepy.mumblescavern.local
  - 192.168.10.12: snooze.mumblescavern.local (future)
  - 192.168.10.13: nappy.mumblescavern.local (future)
  - 192.168.10.20: asustor-nas.mumblescavern.local
  - 192.168.10.21: synology-backup.mumblescavern.local
```

### Phase 3: Combined Naming + Network Migration

#### Step 3A: Migrate & Rename Node 1 (45 minutes)
```bash
# Current node (probably pve-node1 on 192.168.1.x or 192.168.0.x)
# Target: drowzee.mumblescavern.local on 192.168.10.10

# SSH to current node
ssh root@<current-node1-ip>

# Create DHCP reservation in UniFi FIRST:
# MAC of node1 → 192.168.10.10 → drowzee.mumblescavern.local

# Update network configuration for VLAN 10
cat > /etc/network/interfaces << 'EOF'
auto lo
iface lo inet loopback

iface eno1 inet manual

auto vmbr0
iface vmbr0 inet static
    address 192.168.10.10/24
    gateway 192.168.10.1
    bridge-ports eno1
    bridge-stp off
    bridge-fd 0
EOF

# Update hostname
hostnamectl set-hostname drowzee.mumblescavern.local
echo "drowzee.mumblescavern.local" > /etc/hostname

# Update DNS configuration
echo "nameserver 192.168.10.1" > /etc/resolv.conf

# Update hosts file
cat > /etc/hosts << 'EOF'
127.0.0.1 localhost.localdomain localhost
192.168.10.10 drowzee.mumblescavern.local drowzee pvelocalhost

# IPv6
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF

# Reboot to apply network changes
reboot
```

#### Step 3B: Migrate & Rename Node 2 (45 minutes)
```bash
# Wait for Node 1 to come back online, then repeat for Node 2
# Target: sleepy.mumblescavern.local on 192.168.10.11

# SSH to Node 2 (from new Node 1 IP)
ssh root@192.168.10.10
ssh root@<current-node2-ip>

# Create DHCP reservation: Node2 MAC → 192.168.10.11

# Similar network config for Node 2:
cat > /etc/network/interfaces << 'EOF'
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
EOF

# Update hostname  
hostnamectl set-hostname sleepy.mumblescavern.local
echo "sleepy.mumblescavern.local" > /etc/hostname

# Update hosts file with both nodes
cat > /etc/hosts << 'EOF'
127.0.0.1 localhost.localdomain localhost
192.168.10.10 drowzee.mumblescavern.local drowzee
192.168.10.11 sleepy.mumblescavern.local sleepy pvelocalhost

::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF

# Reboot Node 2
reboot
```

#### Step 3C: Fix Cluster Communication (30 minutes)
```bash
# After both nodes reboot, fix cluster
# SSH to drowzee
ssh root@192.168.10.10

# Update cluster configuration
# Edit /etc/pve/corosync.conf to reflect new IPs and names

# Update /etc/hosts on Node 1
cat > /etc/hosts << 'EOF'
127.0.0.1 localhost.localdomain localhost
192.168.10.10 drowzee.mumblescavern.local drowzee pvelocalhost
192.168.10.11 sleepy.mumblescavern.local sleepy

::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
ff02::3 ip6-allhosts
EOF

# Test cluster status
pvecm status

# If cluster is broken, might need to recreate:
# pvecm create mumbles-cavern  (on drowzee)
# pvecm add 192.168.10.10     (on sleepy)
```

### Phase 4: ASUSTOR NAS Migration (During TrueNAS Reset)
```yaml
ASUSTOR Migration (Perfect timing with reset):
Current: 192.168.0.93 on default network
Target: 192.168.10.20 (asustor-nas.mumblescavern.local)

Process:
1. During TrueNAS reset, configure new IP
2. Set static IP: 192.168.10.20/24
3. Set gateway: 192.168.10.1
4. Set DNS: 192.168.10.1
5. Update hostname to: asustor-nas

UniFi DNS Entry:
- 192.168.10.20 → asustor-nas.mumblescavern.local
```

### Phase 5: Other Devices Migration (Ongoing)
```yaml
Gaming Devices → VLAN 80:
- mighty-snorlax: 192.168.80.10
- snorlax-prime: 192.168.80.15 (after role transition)
- sleepy-deck: 192.168.80.20

Personal Devices → VLAN 20 (VPN Protected):
- munchlax (MacBook Air): 192.168.20.51
- dreamy-pro (MacBook Pro): 192.168.20.50

Work Devices → VLAN 30:
- work-macbook: 192.168.30.10
- work-windows: 192.168.30.11

IoT Devices → VLAN 50:
- sleepy-assistant: 192.168.50.10
- dreamy-light-hub: 192.168.50.20
```

## Inter-VLAN Routing Rules

### UniFi Firewall Configuration
```yaml
Required Rules:
1. Gaming VLAN → Services VLAN (for Plex access)
2. Default VLAN → Services VLAN (for homelab access)
3. Services VLAN → Management VLAN (for NAS access)
4. All VLANs → Internet (via VPN gateway when implemented)
5. Work VLAN isolated from other homelab VLANs
6. IoT VLAN isolated except for specific Home Assistant access
```

## Migration Benefits

### Immediate Benefits
```yaml
Network Segmentation:
- Security isolation between work and personal
- IoT device containment  
- Gaming traffic optimization
- Infrastructure protection

Proper Naming:
- Professional appearance
- Easier troubleshooting
- Documentation alignment
- Future scalability
```

### Service Deployment Benefits
```yaml
Clean Foundation:
- Plex deployed on proper VLAN from start
- VPN integration architecture ready
- Monitoring can use proper hostnames
- Future nodes join with correct setup
```

## Rollback Plan

### If Migration Goes Wrong
```bash
# Emergency rollback to flat network:
1. Set all devices back to DHCP on VLAN 1
2. Use generic hostnames temporarily
3. Focus on functionality first
4. Retry migration during planned maintenance window

# Always have console access to Proxmox nodes
# Keep backup of original network configs
```

## Execution Timeline

### Day 1 (Setup): 2-3 hours
1. **Create VLANs in UniFi** (30 min)
2. **Migrate Node 1** (45 min) 
3. **Migrate Node 2** (45 min)
4. **Fix cluster communication** (30 min)
5. **Verify connectivity** (15 min)

### Day 2 (TrueNAS Reset): Parallel work
1. **ASUSTOR network migration** during reset
2. **Test NAS connectivity** from new Proxmox IPs

### Day 3+ (Ongoing): Device migration
1. **Move devices to appropriate VLANs** as needed
2. **Deploy services** on proper network segments

This plan gets you proper naming AND network segmentation in one coordinated effort! Ready to start with the UniFi VLAN creation?