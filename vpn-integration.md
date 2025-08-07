# NordVPN Integration for Snorlax Homelab

**Reference ADRs**: [VPN Strategy ADR](adrs/006-vpn-integration-strategy.md)  
**External Domain**: `snorlax.me`  
**Internal Domain**: `mumblescavern.local`  
**Last Updated**: 2025-01-07

## Overview

Integrate NordVPN across the Snorlax homelab infrastructure to provide security and anonymity while maintaining proper exclusions for work devices, printers, and other essential services that require direct internet access.

## VPN Architecture Strategy

### Centralized VPN Gateway Approach
Deploy a dedicated VPN gateway VM that handles all VPN routing with smart exclusion rules, rather than individual VPN clients on each device.

```
Internet
    ↓
UniFi Dream Machine Pro (Gateway)
    ↓
VPN Gateway VM (sleepy-vpn.mumblescavern.local)
    ↓ (VPN Traffic)        ↓ (Excluded Traffic)
NordVPN Servers ←→ Direct Internet Access
    ↓                      ↓
Protected Devices      Work/Printer Devices
```

## Device Classification & Routing

### VPN-Protected Devices (Through NordVPN)
```yaml
ALL devices get VPN protection by default, including:

Gaming VLAN (192.168.80.x):
  - mighty-snorlax (192.168.80.10) - Gaming PC
  - sleepy-deck (192.168.80.20) - Steam Deck
  - All gaming consoles (192.168.80.40+)

Services VLAN (192.168.20.x):
  - dreamy-pro (192.168.20.50) - MacBook Pro 16" (lab integration)
  - munchlax (192.168.20.51) - MacBook Air M4 (personal daily driver)
  - All homelab service VMs
  - Media streaming services

Default VLAN (192.168.1.x):
  - Personal mobile devices (phones, tablets)
  - Network printer (sleepy-printer)
  - Trusted personal devices

IoT VLAN (192.168.50.x) - NEW:
  - Google Home devices
  - Philips Hue bridge and lights
  - Elgato lights
  - Future smart home devices

Management VLAN (192.168.10.x):
  - Dell cluster nodes (management traffic can be VPN or direct)
  - NAS devices (configurable)
```

### VPN-Excluded Devices (Direct Internet)
```yaml
Work VLAN (192.168.30.x) - ONLY for work compliance:
  - work-macbook (192.168.30.10) - Work MacBook
  - work-windows (192.168.30.11) - Work Windows laptop
  - Any future work-issued devices

Note: These are the ONLY devices excluded from VPN protection
for corporate compliance requirements.
```

## IoT Device Security Architecture

### IoT VLAN Isolation Rules
```yaml
IoT Device Access Policy:
  Internet Access: Through VPN (anonymity + security)
  Homelab Access: Restricted to Home Assistant VM only
  Inter-VLAN Access: Blocked (cannot reach gaming/personal devices)
  Device Discovery: Limited to IoT VLAN only

Firewall Rules:
  IoT → Internet: Via VPN gateway ✅
  IoT → Home Assistant: Allow (future deployment) ✅  
  IoT → All other homelab: Block 🚫
  All other VLANs → IoT: Block 🚫
  IoT ↔ IoT: Allow (for device communication) ✅
```

## Implementation Architecture

### VPN Gateway VM Specification
```yaml
VM Name: sleepy-vpn.mumblescavern.local
Host: Any Dell cluster node (HA capable)
Resources:
  CPU: 2 cores
  RAM: 4GB
  Storage: 20GB
  Network: Bridge to all VLANs

Purpose:
  - NordVPN client connection
  - Traffic routing and filtering
  - DNS filtering and ad-blocking
  - Kill switch protection
  - Connection health monitoring
```

### VLAN Routing Configuration
```yaml
VPN Gateway Interfaces:
  eth0: Management VLAN (192.168.10.25)
  eth1: Gaming VLAN (192.168.80.1) - VPN route
  eth2: Services VLAN (192.168.20.1) - VPN route  
  eth3: Default VLAN (192.168.1.1) - VPN route
  eth4: Work VLAN (192.168.30.1) - Direct route
  eth5: IoT VLAN (192.168.50.1) - VPN route + Isolated
```

## Network Architecture Updates

### Updated VLAN Structure
```yaml
VLAN 1 (Default): 192.168.1.x - Personal trusted devices, printer (VPN)
VLAN 10 (Management): 192.168.10.x - Infrastructure (VPN)
VLAN 20 (Services): 192.168.20.x - Homelab services (VPN)
VLAN 30 (Work): 192.168.30.x - Work devices ONLY (Direct) - NEW
VLAN 50 (IoT): 192.168.50.x - Smart home devices (VPN + Isolated) - NEW
VLAN 80 (Gaming): 192.168.80.x - Gaming devices (VPN)

Note: Only VLAN 30 (Work) bypasses VPN. All others VPN protected.
```

### UniFi Configuration Updates
```yaml
New VLANs Required:
  VLAN 30 (Work):
    - Subnet: 192.168.30.0/24
    - Gateway: 192.168.30.1
    - DHCP: 192.168.30.100-199
    - Internet: Direct (no VPN)
    - Firewall: Restricted access to homelab

  VLAN 40 (Printer):
    - Subnet: 192.168.40.0/24
    - Gateway: 192.168.40.1
    - DHCP: 192.168.40.100-150
    - Internet: Direct (for cloud printing)
    - Access: All VLANs can print
```

## VPN Gateway Deployment

### Step 1: Create VPN Gateway VM
```bash
# SSH to any Dell cluster node
ssh drowzee.mumblescavern.local

# Create VPN gateway VM
qm create 201 \
    --memory 4096 \
    --cores 2 \
    --name sleepy-vpn \
    --net0 virtio,bridge=vmbr0,tag=10 \
    --net1 virtio,bridge=vmbr0,tag=1 \
    --net2 virtio,bridge=vmbr0,tag=20 \
    --net3 virtio,bridge=vmbr0,tag=30 \
    --net4 virtio,bridge=vmbr0,tag=40 \
    --net5 virtio,bridge=vmbr0,tag=80 \
    --storage asustor-vm-storage

# Import Ubuntu cloud image
qm importdisk 201 /mnt/pve/asustor-iso/jammy-server-cloudimg-amd64.img asustor-vm-storage

# Configure VM hardware
qm set 201 --scsihw virtio-scsi-pci --scsi0 asustor-vm-storage:vm-201-disk-0
qm set 201 --ide2 asustor-vm-storage:cloudinit
qm set 201 --boot c --bootdisk scsi0
qm set 201 --serial0 socket --vga serial0

# Configure cloud-init
qm set 201 --ciuser ubuntu
qm set 201 --cipassword $(openssl rand -base64 12)
qm set 201 --ipconfig0 ip=192.168.10.25/24,gw=192.168.10.1
qm set 201 --nameserver 192.168.10.1
qm set 201 --searchdomain mumblescavern.local

# Add SSH key
cat ~/.ssh/authorized_keys | qm set 201 --sshkeys -

# Resize disk and start
qm resize 201 scsi0 20G
qm start 201
```

### Step 2: Configure VPN Gateway Software
```bash
# SSH to VPN gateway
ssh ubuntu@192.168.10.25

# Update system
sudo apt update && sudo apt upgrade -y

# Install required packages
sudo apt install -y \
    openvpn \
    iptables-persistent \
    unbound \
    curl \
    wget \
    net-tools \
    htop \
    fail2ban \
    ufw

# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Create NordVPN configuration directory
sudo mkdir -p /etc/openvpn/nordvpn
cd /etc/openvpn/nordvpn
```

### Step 3: Configure NordVPN Connection
```bash
# Download NordVPN configuration files
# Get your NordVPN service credentials first (not your regular login)
# Go to https://my.nordaccount.com/dashboard/nordvpn/ -> Manual setup

# Create credentials file
sudo tee /etc/openvpn/nordvpn/credentials << EOF
your-nordvpn-service-username
your-nordvpn-service-password
EOF
sudo chmod 600 /etc/openvpn/nordvpn/credentials

# Download optimal server configuration
# Choose server based on location preference
sudo wget https://downloads.nordcdn.com/configs/archives/servers/ovpn.zip
sudo unzip ovpn.zip
sudo cp ovpn_udp/us*.nordvpn.com.udp.ovpn /etc/openvpn/nordvpn/nordvpn.conf

# Modify configuration for credentials
sudo sed -i 's/auth-user-pass/auth-user-pass \/etc\/openvpn\/nordvpn\/credentials/' /etc/openvpn/nordvpn/nordvpn.conf

# Add custom configuration
sudo tee -a /etc/openvpn/nordvpn/nordvpn.conf << EOF

# Custom configuration for homelab
script-security 2
up /etc/openvpn/nordvpn/up.sh
down /etc/openvpn/nordvpn/down.sh

# DNS configuration
dhcp-option DNS 103.86.96.100
dhcp-option DNS 103.86.99.100

# Logging
log /var/log/openvpn-nordvpn.log
log-append
verb 3
EOF
```

### Step 4: Configure Routing and Firewall Rules
```bash
# Create interface configuration
sudo tee /etc/netplan/99-vpn-gateway.yaml << EOF
network:
  version: 2
  ethernets:
    # Management interface (no VPN)
    eth0:
      dhcp4: false
      addresses: [192.168.10.25/24]
      routes:
        - to: 192.168.10.0/24
          via: 192.168.10.1
        - to: default
          via: 192.168.10.1
          metric: 200
    
    # Gaming VLAN (through VPN)
    eth5:
      dhcp4: false
      addresses: [192.168.80.254/24]
    
    # Services VLAN (through VPN)  
    eth2:
      dhcp4: false
      addresses: [192.168.20.254/24]
    
    # Default VLAN (through VPN)
    eth1:
      dhcp4: false
      addresses: [192.168.1.254/24]
    
    # Work VLAN (direct internet)
    eth3:
      dhcp4: false
      addresses: [192.168.30.254/24]
    
    # Printer VLAN (direct internet)
    eth4:
      dhcp4: false
      addresses: [192.168.40.254/24]
EOF

sudo netplan apply
```

### Step 5: Create VPN Routing Scripts
```bash
# Create VPN up script
sudo tee /etc/openvpn/nordvpn/up.sh << 'EOF'
#!/bin/bash
# VPN connection established - configure routing

# VPN interface (usually tun0)
VPN_IF="$1"
VPN_GW="$4"

# Log connection
echo "$(date): VPN connected on $VPN_IF with gateway $VPN_GW" >> /var/log/vpn-routing.log

# Route VPN traffic through VPN interface
# Gaming VLAN
iptables -t nat -A POSTROUTING -s 192.168.80.0/24 -o "$VPN_IF" -j MASQUERADE
ip route add 192.168.80.0/24 dev "$VPN_IF" table 100

# Services VLAN  
iptables -t nat -A POSTROUTING -s 192.168.20.0/24 -o "$VPN_IF" -j MASQUERADE
ip route add 192.168.20.0/24 dev "$VPN_IF" table 100

# Default VLAN (selective routing)
iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o "$VPN_IF" -j MASQUERADE
ip route add 192.168.1.0/24 dev "$VPN_IF" table 100

# Add default route for VPN traffic
ip route add default via "$VPN_GW" dev "$VPN_IF" table 100

# Configure policy routing
ip rule add from 192.168.80.0/24 table 100 priority 100
ip rule add from 192.168.20.0/24 table 100 priority 100  
ip rule add from 192.168.1.0/24 table 100 priority 100

# Kill switch - drop traffic if VPN is down
iptables -I FORWARD -i eth5 -o eth0 -j DROP  # Block gaming->internet
iptables -I FORWARD -i eth2 -o eth0 -j DROP  # Block services->internet  
iptables -I FORWARD -i eth1 -o eth0 -j DROP  # Block default->internet

echo "VPN routing configured successfully"
EOF

sudo chmod +x /etc/openvpn/nordvpn/up.sh
```

```bash
# Create VPN down script
sudo tee /etc/openvpn/nordvpn/down.sh << 'EOF'
#!/bin/bash
# VPN connection lost - clean up routing

echo "$(date): VPN disconnected - cleaning routing rules" >> /var/log/vpn-routing.log

# Remove VPN NAT rules
iptables -t nat -D POSTROUTING -s 192.168.80.0/24 -o tun0 -j MASQUERADE 2>/dev/null
iptables -t nat -D POSTROUTING -s 192.168.20.0/24 -o tun0 -j MASQUERADE 2>/dev/null
iptables -t nat -D POSTROUTING -s 192.168.1.0/24 -o tun0 -j MASQUERADE 2>/dev/null

# Remove policy routing rules
ip rule del from 192.168.80.0/24 table 100 2>/dev/null
ip rule del from 192.168.20.0/24 table 100 2>/dev/null
ip rule del from 192.168.1.0/24 table 100 2>/dev/null

# Keep kill switch active - traffic should remain blocked until VPN reconnects

echo "VPN routing cleanup completed"
EOF

sudo chmod +x /etc/openvpn/nordvpn/down.sh
```

### Step 6: Configure Direct Internet Access Rules
```bash
# Create firewall rules for direct internet access
sudo tee /etc/iptables/rules.v4 << 'EOF'
*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]

# Direct internet access for work and printer VLANs
-A POSTROUTING -s 192.168.30.0/24 -o eth0 -j MASQUERADE
-A POSTROUTING -s 192.168.40.0/24 -o eth0 -j MASQUERADE

# Management VLAN direct access (for NAS, Proxmox management)
-A POSTROUTING -s 192.168.10.0/24 -o eth0 -j MASQUERADE

COMMIT

*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# Allow established and related connections
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
-A INPUT -i lo -j ACCEPT

# Allow SSH from management network
-A INPUT -i eth0 -p tcp --dport 22 -j ACCEPT

# Allow inter-VLAN communication where needed
# Work devices can access printer VLAN
-A FORWARD -s 192.168.30.0/24 -d 192.168.40.0/24 -j ACCEPT

# All VLANs can access printer VLAN
-A FORWARD -d 192.168.40.0/24 -j ACCEPT

# Gaming devices can access services (for Plex, etc.)
-A FORWARD -s 192.168.80.0/24 -d 192.168.20.0/24 -j ACCEPT

# Block work VLAN from accessing other homelab networks
-A FORWARD -s 192.168.30.0/24 -d 192.168.80.0/24 -j DROP
-A FORWARD -s 192.168.30.0/24 -d 192.168.20.0/24 -j DROP  
-A FORWARD -s 192.168.30.0/24 -d 192.168.1.0/24 -j DROP

# Default drop for unmatched traffic
-A INPUT -j DROP
-A FORWARD -j DROP

COMMIT
EOF

# Apply firewall rules
sudo iptables-restore < /etc/iptables/rules.v4
sudo netfilter-persistent save
```

## Printer Integration

### Network Printer Setup
```bash
# Configure network printer on Printer VLAN
# Printer should be configured with:
IP: 192.168.40.10
Subnet: 255.255.255.0
Gateway: 192.168.40.1
DNS: 192.168.40.1

# Snorlax-themed hostname: sleepy-printer.mumblescavern.local
```

### Printer Access Configuration
```yaml
Printer Access Rules:
  From Gaming VLAN: ✅ Allowed
  From Services VLAN: ✅ Allowed  
  From Default VLAN: ✅ Allowed
  From Work VLAN: ✅ Allowed
  From Management VLAN: ✅ Allowed

Internet Access: Direct (no VPN)
  - Cloud printing services
  - Driver updates
  - Printer manufacturer services
```

## Device Assignment Strategy

### Work Laptop Configuration
```yaml
Work Laptop Setup:
  Network: Connect to "Work" SSID → VLAN 30
  IP Assignment: 192.168.30.10 (DHCP reservation)
  Internet: Direct (no VPN)
  Homelab Access: Blocked (security)
  Printer Access: Allowed
  
Benefits:
  - Work traffic never goes through VPN
  - Maintains corporate compliance
  - Can access company resources normally
  - Still can print to homelab printer
```

### MacBook Air M4 (munchlax) Configuration
```yaml
Dual Network Setup:
  Personal Mode: Connect to "Services" SSID → Through VPN (Primary)
  Work Mode: Connect to "Work" SSID → Direct internet (When needed)
  
Implementation:
  - Two Wi-Fi network profiles
  - Easy switching between personal/work modes
  - VPN protection for personal activities by default
  - Work compliance mode available when required
```

### MacBook Pro 16" (dreamy-pro) Configuration
```yaml
Homelab Integration:
  Primary: Services VLAN → Through VPN
  Role: Lab development, media production, management
  Access: All homelab services and external resources
  Protection: Full VPN coverage for lab activities
```

## VPN Service Management

### Start/Stop/Monitor Scripts
```bash
# Create VPN management script
sudo tee /usr/local/bin/nordvpn-manager << 'EOF'
#!/bin/bash
# NordVPN management for Snorlax homelab

case "$1" in
    start)
        echo "Starting NordVPN connection..."
        sudo systemctl start openvpn@nordvpn
        sleep 5
        sudo systemctl status openvpn@nordvpn
        ;;
    stop)
        echo "Stopping NordVPN connection..."
        sudo systemctl stop openvpn@nordvpn
        ;;
    restart)
        echo "Restarting NordVPN connection..."
        sudo systemctl restart openvpn@nordvpn
        sleep 5
        sudo systemctl status openvpn@nordvpn
        ;;
    status)
        echo "=== VPN Status ==="
        sudo systemctl status openvpn@nordvpn
        echo ""
        echo "=== Connection Info ==="
        ip addr show tun0 2>/dev/null || echo "VPN interface not active"
        echo ""
        echo "=== External IP ==="
        curl -s ifconfig.me
        echo ""
        ;;
    test)
        echo "=== VPN Test ==="
        echo "External IP: $(curl -s ifconfig.me)"
        echo "VPN Interface:"
        ip addr show tun0 2>/dev/null || echo "No VPN interface"
        echo ""
        echo "Testing connectivity from different VLANs:"
        # Test from each VLAN network
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status|test}"
        exit 1
        ;;
esac
EOF

sudo chmod +x /usr/local/bin/nordvpn-manager

# Enable VPN service
sudo systemctl enable openvpn@nordvpn
```

### Health Monitoring
```bash
# Create VPN health monitor
sudo tee /etc/cron.d/nordvpn-health << 'EOF'
# Monitor VPN connection every 5 minutes
*/5 * * * * root /usr/local/bin/vpn-health-check.sh

# Test external connectivity every hour
0 * * * * root /usr/local/bin/vpn-connectivity-test.sh
EOF

# Create health check script
sudo tee /usr/local/bin/vpn-health-check.sh << 'EOF'
#!/bin/bash
# VPN health monitoring

LOG_FILE="/var/log/vpn-health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Check if VPN service is running
if systemctl is-active --quiet openvpn@nordvpn; then
    log_message "VPN service is running"
else
    log_message "ERROR: VPN service is down, attempting restart"
    systemctl start openvpn@nordvpn
    sleep 10
    if systemctl is-active --quiet openvpn@nordvpn; then
        log_message "VPN service restarted successfully"
    else
        log_message "CRITICAL: VPN service failed to restart"
    fi
fi

# Check if tun0 interface exists
if ip link show tun0 > /dev/null 2>&1; then
    log_message "VPN interface tun0 is active"
else
    log_message "WARNING: VPN interface tun0 is not active"
fi

# Test external connectivity
EXTERNAL_IP=$(curl -s --max-time 10 ifconfig.me)
if [ ! -z "$EXTERNAL_IP" ]; then
    log_message "External IP: $EXTERNAL_IP"
else
    log_message "WARNING: Could not determine external IP"
fi

# Keep log manageable
tail -200 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

sudo chmod +x /usr/local/bin/vpn-health-check.sh
```

## UniFi Configuration Updates

### New Network Creation
```yaml
Required UniFi Changes:

1. Create Work Network:
   - VLAN ID: 30
   - Subnet: 192.168.30.0/24
   - DHCP: 192.168.30.100-199
   - WiFi: "Snorlax-Work" (WPA3)

2. Create Printer Network:
   - VLAN ID: 40  
   - Subnet: 192.168.40.0/24
   - DHCP: 192.168.40.100-150
   - Wired: Dedicated switch port

3. Update Firewall Rules:
   - Allow all → Printer VLAN
   - Block Work → Gaming/Services/Default
   - Allow Work → Internet (direct)

4. DHCP Reservations:
   - 192.168.30.10: Work laptop MAC
   - 192.168.40.10: Network printer MAC
```

## Benefits of This Architecture

### Security Benefits
- **Anonymized Traffic**: Gaming, media, and personal devices through VPN
- **Work Compliance**: Work devices bypass VPN (corporate policy friendly)
- **Network Segmentation**: Work isolated from homelab
- **Kill Switch**: Internet blocked if VPN fails

### Performance Benefits  
- **Gaming Optimization**: VPN with gaming-optimized NordVPN servers
- **Work Performance**: Direct internet for work (no VPN overhead)
- **Printer Efficiency**: Local network printing without VPN latency

### Management Benefits
- **Centralized Control**: Single VPN gateway manages all routing
- **Easy Exclusions**: Simple device VLAN assignment
- **Monitoring**: Comprehensive health checks and logging
- **Flexibility**: Easy to add/remove devices from VPN

This architecture provides the security and anonymity you want while maintaining work compliance and printer functionality! 🛡️