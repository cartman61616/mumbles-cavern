        GPU_TEMP=$(ssh ubuntu@192.168.80.103 "nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits" 2>/dev/null)
        if [ ! -z "$GPU_TEMP" ] && [ "$GPU_TEMP" != "N/A" ]; then
            log_message "Stream Server GPU Temp: ${GPU_TEMP}°C"
            
            if [ "$GPU_TEMP" -gt 85 ]; then
                log_message "WARNING: High GPU temperature on streaming server"
            fi
        fi
    fi
}

# Check game library server health
check_game_library() {
    log_message "=== Game Library Server Health ==="
    
    # Check RomM service
    if curl -s http://192.168.80.101:8080 > /dev/null; then
        log_message "Game Library (RomM): Healthy"
    else
        log_message "Game Library (RomM): Not responding"
    fi
    
    # Check game storage mount
    if ssh ubuntu@192.168.80.101 "mountpoint -q /mnt/games" 2>/dev/null; then
        GAMES_USAGE=$(ssh ubuntu@192.168.80.101 "df -h /mnt/games | tail -1 | awk '{print \$5}'" 2>/dev/null)
        log_message "Game Storage: Mounted, ${GAMES_USAGE} used"
    else
        log_message "Game Storage: Mount failed"
    fi
}

# Check gaming network performance
check_gaming_network() {
    log_message "=== Gaming Network Performance ==="
    
    # Test latency between gaming services
    STREAMING_LATENCY=$(ping -c 5 192.168.80.103 2>/dev/null | grep 'rtt' | cut -d'=' -f2 | cut -d'/' -f2)
    LIBRARY_LATENCY=$(ping -c 5 192.168.80.101 2>/dev/null | grep 'rtt' | cut -d'=' -f2 | cut -d'/' -f2)
    
    if [ ! -z "$STREAMING_LATENCY" ]; then
        log_message "Gaming Network Latency - Streaming: ${STREAMING_LATENCY}ms, Library: ${LIBRARY_LATENCY}ms"
        
        # Alert if latency is high
        if (( $(echo "$STREAMING_LATENCY > 5.0" | bc -l) )); then
            log_message "WARNING: High latency to streaming server"
        fi
    fi
    
    # Check gaming VLAN connectivity
    if ip route | grep -q "192.168.80.0/24"; then
        log_message "Gaming VLAN: Routing configured"
    else
        log_message "Gaming VLAN: Routing issue detected"
    fi
}

# Check external gaming device connectivity
check_gaming_devices() {
    log_message "=== Gaming Device Connectivity ==="
    
    GAMING_DEVICES=(
        "192.168.80.10:mighty-snorlax"
        "192.168.80.20:sleepy-deck"
        "192.168.80.40:dreamy-console-1"
    )
    
    ONLINE_DEVICES=0
    for device_info in "${GAMING_DEVICES[@]}"; do
        IP=$(echo "$device_info" | cut -d':' -f1)
        NAME=$(echo "$device_info" | cut -d':' -f2)
        
        if ping -c 1 -W 2 "$IP" > /dev/null 2>&1; then
            log_message "$NAME: Online"
            ((ONLINE_DEVICES++))
        else
            log_message "$NAME: Offline"
        fi
    done
    
    log_message "Gaming devices online: $ONLINE_DEVICES/3"
}

# Execute all health checks
check_streaming_server
check_game_library
check_gaming_network
check_gaming_devices

log_message "Gaming health check completed"

# Keep log manageable
tail -500 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x scripts/gaming-health-monitor.sh

# Schedule gaming health checks every 2 minutes
echo "*/2 * * * * /home/ubuntu/monitoring/scripts/gaming-health-monitor.sh" | crontab -l | { cat; echo "*/2 * * * * /home/ubuntu/monitoring/scripts/gaming-health-monitor.sh"; } | crontab -

# Test gaming health monitoring
./scripts/gaming-health-monitor.sh
cat /var/log/gaming-health.log
```

## Step 8: Configure Game Save and State Management

### Deploy Save State Management System
```bash
# SSH to game library VM for save management
ssh ubuntu@192.168.80.101

cd /home/ubuntu/game-library

# Create save state management system
mkdir -p save-management/{saves,states,backups}

cat > save-management/docker-compose.yml << 'EOF'
version: '3.8'

services:
  save-keeper:
    image: linuxserver/syncthing:latest
    container_name: save-keeper
    restart: unless-stopped
    hostname: save-keeper
    ports:
      - "8384:8384"   # Web UI
      - "22000:22000" # Sync protocol
      - "21027:21027/udp" # Discovery
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./syncthing-config:/config
      - ./saves:/saves
      - ./states:/states
      - /mnt/games/saves:/games-saves
    networks:
      - gaming-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.saves.entrypoints=websecure"
      - "traefik.http.routers.saves.rule=Host(`saves.snorlax.me`)"
      - "traefik.http.routers.saves.middlewares=security-headers@file"
      - "traefik.http.routers.saves.tls=true"
      - "traefik.http.routers.saves.tls.certresolver=cloudflare"

  save-backup:
    image: postgres:15
    container_name: save-backup
    restart: unless-stopped
    environment:
      - POSTGRES_HOST=192.168.1.15
      - POSTGRES_DB=save_states_db
      - POSTGRES_USER=save_user
      - POSTGRES_PASSWORD=save_secure_password
    volumes:
      - ./backups:/backups
      - ./scripts:/scripts
    networks:
      - gaming-network
      - shared-network
    command: >
      sh -c "
        while true; do
          sleep 21600
          echo 'Backing up save states at $(date)'
          tar -czf /backups/save_states_$(date +%Y%m%d_%H%M%S).tar.gz -C /saves .
          tar -czf /backups/game_states_$(date +%Y%m%d_%H%M%S).tar.gz -C /states .
          find /backups -name '*.tar.gz' -mtime +30 -delete
          echo 'Save backup completed at $(date)'
        done
      "

networks:
  gaming-network:
    external: true
  shared-network:
    external: true
EOF

# Start save management services
docker-compose up -d

# Create save state management script
cat > scripts/manage-saves.sh << 'EOF'
#!/bin/bash
# Game save and state management

SAVES_DIR="/home/ubuntu/game-library/save-management/saves"
STATES_DIR="/home/ubuntu/game-library/save-management/states"
LOG_FILE="/var/log/save-management.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$LOG_FILE"
}

# Backup individual game saves
backup_game_saves() {
    local GAME="$1"
    local SAVE_PATH="$2"
    
    if [ -d "$SAVE_PATH" ]; then
        BACKUP_FILE="$SAVES_DIR/${GAME}_$(date +%Y%m%d_%H%M%S).tar.gz"
        tar -czf "$BACKUP_FILE" -C "$(dirname "$SAVE_PATH")" "$(basename "$SAVE_PATH")"
        log_message "Backed up saves for $GAME"
    fi
}

# Restore game saves
restore_game_saves() {
    local BACKUP_FILE="$1"
    local RESTORE_PATH="$2"
    
    if [ -f "$BACKUP_FILE" ]; then
        tar -xzf "$BACKUP_FILE" -C "$RESTORE_PATH"
        log_message "Restored saves from $BACKUP_FILE"
    fi
}

# Sync saves across devices
sync_saves() {
    log_message "Starting save synchronization"
    
    # Use Syncthing API to trigger sync
    curl -X POST -H "X-API-Key: $(cat ./syncthing-config/config.xml | grep -o '<apikey>[^<]*' | cut -d'>' -f2)" \
         http://localhost:8384/rest/db/scan?folder=saves
    
    log_message "Save synchronization triggered"
}

case "$1" in
    "backup")
        backup_game_saves "$2" "$3"
        ;;
    "restore")
        restore_game_saves "$2" "$3"
        ;;
    "sync")
        sync_saves
        ;;
    *)
        echo "Usage: $0 {backup|restore|sync}"
        echo "  backup <game> <save-path>"
        echo "  restore <backup-file> <restore-path>"
        echo "  sync"
        ;;
esac
EOF

chmod +x scripts/manage-saves.sh
```

## Step 9: Configure Gaming External Access

### Update Cloudflare Tunnel for Gaming Services
Back in **Cloudflare Dashboard**, add gaming service hostnames to your tunnel:

```
gaming.snorlax.me → http://192.168.1.23:80 (Sunshine via Traefik)
retro.snorlax.me → http://192.168.1.23:80 (RetroArch via Traefik)
games.snorlax.me → http://192.168.1.23:80 (Game Library via Traefik)
saves.snorlax.me → http://192.168.1.23:80 (Save Management via Traefik)
```

### Configure Gaming Service Security
```bash
# SSH to Traefik VM to add gaming-specific security
ssh ubuntu@192.168.1.23

cd /home/ubuntu/traefik

# Create gaming-specific security middleware
cat > config/gaming-security.yml << 'EOF'
http:
  middlewares:
    # Gaming services - more permissive for performance
    gaming-access:
      chain:
        middlewares:
          - gaming-rate-limit@file
          - gaming-headers@file
          - compression@file

    # Gaming rate limiting (higher limits for streaming)
    gaming-rate-limit:
      rateLimit:
        burst: 200
        average: 100
        period: "1m"
        sourceCriterion:
          ipStrategy:
            depth: 1

    # Gaming-optimized headers (reduced security for performance)
    gaming-headers:
      headers:
        browserXssFilter: true
        contentTypeNosniff: true
        customResponseHeaders:
          X-Gaming-Server: "mumbles-cluster"
          Cache-Control: "no-cache, no-store, must-revalidate"

    # Streaming-specific middleware
    streaming-optimized:
      chain:
        middlewares:
          - streaming-headers@file
          - no-cache@file

    streaming-headers:
      headers:
        customResponseHeaders:
          X-Content-Type-Options: "nosniff"
          Access-Control-Allow-Origin: "*"
          Access-Control-Allow-Methods: "GET, POST, OPTIONS"

    no-cache:
      headers:
        customResponseHeaders:
          Cache-Control: "no-cache, no-store, must-revalidate"
          Pragma: "no-cache"
          Expires: "0"

  # Gaming service routers with optimized middleware
  routers:
    sunshine-gaming:
      rule: "Host(`gaming.snorlax.me`)"
      entrypoints:
        - websecure
      middlewares:
        - streaming-optimized@file
      tls:
        certResolver: cloudflare
      service: sunshine-service

    game-library:
      rule: "Host(`games.snorlax.me`)"
      entrypoints:
        - websecure
      middlewares:
        - gaming-access@file
      tls:
        certResolver: cloudflare
      service: game-library-service

  services:
    sunshine-service:
      loadBalancer:
        servers:
          - url: "http://192.168.80.103:47990"
        healthCheck:
          path: "/"
          interval: "30s"
          timeout: "5s"

    game-library-service:
      loadBalancer:
        servers:
          - url: "http://192.168.80.101:8080"
        healthCheck:
          path: "/"
          interval: "30s"
          timeout: "10s"
EOF

# Restart Traefik to load gaming configuration
docker-compose restart traefik

# Test gaming service access
sleep 30
curl -k https://gaming.snorlax.me
curl -k https://games.snorlax.me
```

## Step 10: Create Gaming Performance Optimization Scripts

### Configure Gaming VM Performance Tuning
```bash
# Create comprehensive gaming performance script
cat > scripts/gaming-performance-tuning.sh << 'EOF'
#!/bin/bash
# Comprehensive gaming performance tuning

LOG_FILE="/var/log/gaming-tuning.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Tune gaming VMs for performance
tune_gaming_vms() {
    log_message "Starting gaming VM performance tuning"
    
    # Gaming VMs to tune
    GAMING_VMS=(
        "110:dream-stream-server"
        "111:game-librarian"
    )
    
    for vm_info in "${GAMING_VMS[@]}"; do
        VMID=$(echo "$vm_info" | cut -d':' -f1)
        VM_NAME=$(echo "$vm_info" | cut -d':' -f2)
        
        log_message "Tuning VM $VMID ($VM_NAME)"
        
        # Set CPU governor to performance
        qm set "$VMID" --args '-cpu host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX'
        
        # Enable CPU pinning for consistent performance
        # qm set "$VMID" --cpus 0-1  # Pin to specific cores
        
        # Set memory ballooning off for consistent performance
        qm set "$VMID" --balloon 0
        
        log_message "VM $VMID tuning completed"
    done
}

# Optimize host system for gaming workloads
optimize_host_system() {
    log_message "Optimizing host system for gaming"
    
    # Set CPU governor to performance for gaming workloads
    echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
    
    # Disable CPU idle states for consistent latency
    for i in /sys/devices/system/cpu/cpu*/cpuidle/state*/disable; do
        echo 1 > "$i" 2>/dev/null || true
    done
    
    # Optimize interrupt handling
    echo 1 > /proc/sys/kernel/numa_balancing
    echo 0 > /proc/sys/kernel/timer_migration
    
    log_message "Host system optimization completed"
}

# Configure gaming-specific cgroups
configure_gaming_cgroups() {
    log_message "Configuring gaming cgroups"
    
    # Create gaming cgroup for prioritization
    mkdir -p /sys/fs/cgroup/gaming
    
    # Set higher CPU priority for gaming processes
    echo "100000 100000" > /sys/fs/cgroup/gaming/cpu.max
    echo "1000" > /sys/fs/cgroup/gaming/cpu.weight
    
    log_message "Gaming cgroups configured"
}

# Run optimization functions
tune_gaming_vms
optimize_host_system
configure_gaming_cgroups

log_message "Gaming performance tuning completed"
EOF

chmod +x scripts/gaming-performance-tuning.sh

# Run gaming performance tuning on both nodes
./scripts/gaming-performance-tuning.sh

# Apply to Node 2 as well
scp scripts/gaming-performance-tuning.sh root@192.168.1.11:/tmp/
ssh root@192.168.1.11 "chmod +x /tmp/gaming-performance-tuning.sh && /tmp/gaming-performance-tuning.sh"
```

### Create Gaming Load Testing
```bash
# Create gaming infrastructure load testing
cat > scripts/gaming-load-test.sh << 'EOF'
#!/bin/bash
# Gaming infrastructure load testing

echo "=== Gaming Infrastructure Load Test ==="

# Test game streaming server under load
test_streaming_load() {
    echo "Testing game streaming server load..."
    
    # Simulate multiple streaming connections
    for i in {1..5}; do
        curl -s http://192.168.80.103:47990 > /dev/null &
    done
    
    # Monitor CPU usage during simulated load
    ssh ubuntu@192.168.80.103 "top -bn1 | grep 'Cpu(s)'" 2>/dev/null
    
    wait
    echo "Streaming load test completed"
}

# Test game library under load
test_library_load() {
    echo "Testing game library server load..."
    
    # Simulate multiple library requests
    for i in {1..20}; do
        curl -s http://192.168.80.101:8080 > /dev/null &
    done
    
    wait
    echo "Library load test completed"
}

# Test gaming network under load
test_network_load() {
    echo "Testing gaming network load..."
    
    # Test bandwidth between gaming services
    ssh ubuntu@192.168.80.103 "iperf3 -s -D" 2>/dev/null
    sleep 2
    
    BANDWIDTH=$(ssh ubuntu@192.168.80.101 "iperf3 -c 192.168.80.103 -t 10 -f M" 2>/dev/null | grep "sender" | awk '{print $7, $8}')
    echo "Gaming network bandwidth: $BANDWIDTH"
    
    ssh ubuntu@192.168.80.103 "pkill iperf3" 2>/dev/null
}

# Run load tests
test_streaming_load
test_library_load
test_network_load

echo "=== Gaming Load Test Complete ==="
EOF

chmod +x scripts/gaming-load-test.sh
# ./scripts/gaming-load-test.sh  # Run when needed
```

## Step 11: Gaming Service Integration and Automation

### Create Gaming Service Automation
```bash
# Create gaming service automation script
cat > scripts/gaming-automation.sh << 'EOF'
#!/bin/bash
# Gaming service automation and management

LOG_FILE="/var/log/gaming-automation.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Auto-start gaming services based on device activity
auto_start_services() {
    log_message "Checking for gaming device activity"
    
    # Check if gaming devices are online
    ACTIVE_DEVICES=0
    GAMING_IPS=("192.168.80.10" "192.168.80.20" "192.168.80.40")
    
    for ip in "${GAMING_IPS[@]}"; do
        if ping -c 1 -W 2 "$ip" > /dev/null 2>&1; then
            ((ACTIVE_DEVICES++))
        fi
    done
    
    log_message "Active gaming devices: $ACTIVE_DEVICES"
    
    # Start/stop services based on activity
    if [ "$ACTIVE_DEVICES" -gt 0 ]; then
        # Ensure gaming services are running
        ssh ubuntu@192.168.80.103 "cd /home/ubuntu/game-streaming && docker-compose up -d" 2>/dev/null
        ssh ubuntu@192.168.80.101 "cd /home/ubuntu/game-library && docker-compose up -d" 2>/dev/null
        log_message "Gaming services started due to device activity"
    else
        # Consider stopping non-essential services to save power
        log_message "No gaming activity detected"
    fi
}

# Optimize gaming services based on load
optimize_services() {
    log_message "Optimizing gaming services"
    
    # Check current load on streaming server
    if command -v ssh >/dev/null; then
        CPU_LOAD=$(ssh ubuntu@192.168.80.103 "uptime | awk -F'load average:' '{print \$2}' | awk '{print \$1}' | sed 's/,//'" 2>/dev/null)
        
        if [ ! -z "$CPU_LOAD" ]; then
            # If load is high, prioritize streaming processes
            if (( $(echo "$CPU_LOAD > 2.0" | bc -l) )); then
                ssh ubuntu@192.168.80.103 "sudo renice -10 \$(pgrep sunshine)" 2>/dev/null
                log_message "Increased priority for streaming processes due to high load"
            fi
        fi
    fi
}

# Clean up gaming temporary files
cleanup_gaming_data() {
    log_message "Cleaning up gaming temporary data"
    
    # Clean up streaming temp files
    ssh ubuntu@192.168.80.103 "rm -rf /tmp/sunshine_* 2>/dev/null" 2>/dev/null
    
    # Clean up old game saves (keep 90 days)
    ssh ubuntu@192.168.80.101 "find /home/ubuntu/game-library/save-management/backups -name '*.tar.gz' -mtime +90 -delete" 2>/dev/null
    
    log_message "Gaming cleanup completed"
}

# Execute automation functions
auto_start_services
optimize_services
cleanup_gaming_data

log_message "Gaming automation cycle completed"
EOF

chmod +x scripts/gaming-automation.sh

# Schedule gaming automation every 15 minutes
echo "*/15 * * * * /home/ubuntu/monitoring/scripts/gaming-automation.sh" | crontab -l | { cat; echo "*/15 * * * * /home/ubuntu/monitoring/scripts/gaming-automation.sh"; } | crontab -
```

### Create Gaming Service Discovery
```bash
# Create gaming service discovery for monitoring
cat > scripts/gaming-service-discovery.sh << 'EOF'
#!/bin/bash
# Gaming service discovery for dynamic monitoring

PROMETHEUS_CONFIG="/home/ubuntu/monitoring/prometheus/gaming-targets.json"

# Discover active gaming services
discover_gaming_services() {
    cat > "$PROMETHEUS_CONFIG" << 'TARGET_EOF'
[
  {
    "targets": [
      "192.168.80.103:9100",
      "192.168.80.101:9100"
    ],
    "labels": {
      "job": "gaming-infrastructure",
      "environment": "gaming"
    }
  },
  {
    "targets": [
      "192.168.80.103:47990",
      "192.168.80.101:8080"
    ],
    "labels": {
      "job": "gaming-services",
      "environment": "gaming"
    }
  }
]
TARGET_EOF
    
    echo "Gaming service targets updated"
}

# Check for gaming devices and add to monitoring if available
discover_gaming_devices() {
    ACTIVE_TARGETS=()
    
    # Check if Mighty Snorlax has monitoring
    if ping -c 1 -W 2 192.168.80.10 > /dev/null 2>&1; then
        if curl -s http://192.168.80.10:9100/metrics > /dev/null 2>&1; then
            ACTIVE_TARGETS+=("192.168.80.10:9100")
        fi
    fi
    
    # Check if Steam Deck has monitoring  
    if ping -c 1 -W 2 192.168.80.20 > /dev/null 2>&1; then
        if curl -s http://192.168.80.20:9100/metrics > /dev/null 2>&1; then
            ACTIVE_TARGETS+=("192.168.80.20:9100")
        fi
    fi
    
    if [ ${#ACTIVE_TARGETS[@]} -gt 0 ]; then
        # Add gaming devices to monitoring targets
        cat >> "$PROMETHEUS_CONFIG" << TARGET_EOF
,
  {
    "targets": [
$(printf '      "%s",\n' "${ACTIVE_TARGETS[@]}" | sed '$ s/,$//')
    ],
    "labels": {
      "job": "gaming-devices",
      "environment": "gaming"
    }
  }
TARGET_EOF
        echo "Added ${#ACTIVE_TARGETS[@]} gaming devices to monitoring"
    fi
}

# Execute discovery
discover_gaming_services
discover_gaming_devices

# Signal Prometheus to reload configuration
curl -X POST http://192.168.1.22:9090/-/reload 2>/dev/null
EOF

chmod +x scripts/gaming-service-discovery.sh

# Schedule gaming service discovery every 5 minutes
echo "*/5 * * * * /home/ubuntu/monitoring/scripts/gaming-service-discovery.sh" | crontab -l | { cat; echo "*/5 * * * * /home/ubuntu/monitoring/scripts/gaming-service-discovery.sh"; } | crontab -

# Run initial discovery
./scripts/gaming-service-discovery.sh
```

## Step 12: Create Gaming Documentation and User Guides

### Create Gaming Infrastructure Documentation
```bash
# Create comprehensive gaming documentation
cat > docs/gaming-infrastructure-guide.md << 'EOF'
# Gaming Infrastructure Guide

## Gaming Network Overview
- **VLAN**: 80 (Gaming - Dream Realm)
- **Subnet**: 192.168.80.0/24
- **Gateway**: 192.168.80.1
- **QoS**: Highest priority for lowest latency

## Gaming Services
- **Game Streaming**: https://gaming.snorlax.me (Sunshine/Moonlight)
- **Retro Gaming**: https://retro.snorlax.me (RetroArch web interface)
- **Game Library**: https://games.snorlax.me (ROM management)
- **Save Management**: https://saves.snorlax.me (Save state sync)

## Gaming Device Configuration

### Mighty Snorlax (192.168.80.10)
- **Purpose**: Primary gaming PC
- **Network**: Gaming VLAN for optimal performance
- **Streaming**: Can host games via Sunshine
- **Monitoring**: Optional node exporter installation

### Steam Deck (192.168.80.20)
- **Purpose**: Portable gaming device
- **Network**: Gaming Wi-Fi (VLAN 80)
- **Streaming**: Moonlight client for remote gaming
- **Saves**: Synced via save management system

### Gaming Consoles (192.168.80.40+)
- **Purpose**: Console gaming
- **Network**: Gaming VLAN via Ethernet/Wi-Fi
- **Features**: UPnP support, low-latency routing

## Performance Optimization

### Network Optimization
- **BBR congestion control**: Enabled for optimal throughput
- **Low latency settings**: TCP timestamps disabled
- **Gaming QoS**: Highest priority traffic classification
- **Jitter reduction**: fq_codel queue discipline

### Gaming VM Optimization
- **CPU**: Host passthrough with gaming-specific flags
- **Memory**: Balloon disabled for consistent performance
- **Storage**: NVMe for fastest loading times
- **Network**: Direct VLAN access for minimal overhead

### Streaming Optimization
- **Hardware encoding**: GPU acceleration enabled
- **Audio**: Low-latency PulseAudio configuration
- **Video**: Optimized encoding settings
- **Network**: Dedicated gaming VLAN bandwidth

## Monitoring and Maintenance

### Performance Metrics
- **Latency**: <5ms between gaming services
- **Bandwidth**: Full 1Gbps available for streaming
- **CPU**: <50% average on streaming server
- **Temperature**: <80°C CPU, <85°C GPU
- **Uptime**: 99.9% availability target

### Maintenance Tasks
- **Daily**: Check gaming service health
- **Weekly**: Update game libraries and saves
- **Monthly**: Performance optimization review
- **Quarterly**: Hardware temperature and performance audit

## Troubleshooting

### High Latency Issues
1. Check gaming VLAN QoS settings
2. Verify network path optimization
3. Check for network congestion
4. Review CPU governor settings

### Streaming Performance Issues
1. Check GPU hardware acceleration
2. Verify encoding settings
3. Monitor CPU/GPU temperatures
4. Check network bandwidth utilization

### Service Connectivity Issues
1. Verify gaming VLAN configuration
2. Check Docker network settings
3. Test internal service connectivity
4. Validate external access via Traefik
EOF

# Create gaming user manual
cat > docs/gaming-user-manual.md << 'EOF'
# Gaming Services User Manual

## Getting Started

### Setting Up Game Streaming
1. **Install Moonlight** on your client device
2. **Access**: https://gaming.snorlax.me
3. **Pair device** using PIN from Sunshine web interface
4. **Add games** from Mighty Snorlax or streaming server
5. **Start streaming** with optimized settings

### Using Retro Gaming
1. **Access**: https://retro.snorlax.me
2. **Upload ROMs** to appropriate console folders
3. **Configure controllers** in RetroArch settings
4. **Save states** are automatically synced
5. **Achievements** tracked via RetroAchievements

### Managing Game Library
1. **Access**: https://games.snorlax.me
2. **Organize ROMs** by console and region
3. **Add metadata** and artwork automatically
4. **Create playlists** for favorite games
5. **Download game** info from IGDB database

### Save State Management
1. **Access**: https://saves.snorlax.me
2. **Sync saves** across multiple devices
3. **Backup saves** automatically every 6 hours
4. **Restore saves** from any backup point
5. **Share saves** between family members

## Optimal Settings

### Moonlight Streaming Settings
- **Resolution**: 1920x1080 (adjust for bandwidth)
- **FPS**: 60 (or 120 if supported)
- **Bitrate**: 20 Mbps (adjust for quality vs latency)
- **Hardware Decoding**: Enabled
- **VSync**: Disabled for lowest latency

### RetroArch Settings  
- **Video Driver**: Vulkan (if supported) or OpenGL
- **Audio Driver**: PulseAudio
- **Latency**: Set to minimum values
- **Shaders**: Use lightweight shaders only
- **Rewind**: Disabled for best performance

## Gaming Device Setup Instructions

### Windows Gaming PC (Mighty Snorlax)
```cmd
# Set static IP
netsh interface ip set address "Ethernet" static 192.168.80.10 255.255.255.0 192.168.80.1

# Install gaming software
- Steam, Epic Games, Battle.net, GOG Galaxy
- MSI Afterburner for GPU monitoring
- OBS Studio for local recording
- Sunshine for game hosting

# Optimize Windows for gaming
- Set power plan to High Performance
- Disable Windows Game Mode
- Set priority for gaming processes
- Disable Windows Update during gaming hours
```

### Steam Deck Setup
```bash
# Switch to Desktop Mode# Phase 8: Gaming Infrastructure Deployment

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [External Access](08-external-access.md)  
**Duration**: 2-3 hours  
**Prerequisites**: External access operational, gaming VLAN configured

## Overview
Deploy gaming-specific infrastructure including game streaming servers, retro gaming systems, and performance optimization for your gaming network (VLAN 80).

## Step 1: Optimize Gaming Network Performance

### Configure Gaming VLAN QoS on Dell Cluster
```bash
# SSH to primary Proxmox node
ssh root@192.168.1.10

# Create gaming network bridge for VLAN 80
cat >> /etc/network/interfaces << 'EOF'

# Gaming VLAN bridge
auto vmbr80
iface vmbr80 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 80
    post-up echo 1 > /proc/sys/net/ipv4/ip_forward
    post-up iptables -t nat -A POSTROUTING -s '192.168.80.0/24' -o vmbr0 -j MASQUERADE
EOF

# Apply network configuration
ifreload -a

# Verify gaming bridge
ip link show vmbr80
```

### Optimize Network Stack for Gaming
```bash
# Apply gaming-optimized network settings on all Proxmox nodes
cat > /tmp/gaming-network-optimization.sh << 'EOF'
#!/bin/bash
# Gaming network optimization

# TCP congestion control optimized for gaming
echo 'net.core.default_qdisc = fq_codel' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_congestion_control = bbr' >> /etc/sysctl.conf

# Reduce network latency
echo 'net.core.netdev_max_backlog = 5000' >> /etc/sysctl.conf
echo 'net.core.netdev_budget = 600' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_no_metrics_save = 1' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_moderate_rcvbuf = 1' >> /etc/sysctl.conf

# Gaming-specific optimizations
echo 'net.ipv4.tcp_timestamps = 0' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_sack = 1' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_fack = 1' >> /etc/sysctl.conf

# Apply settings
sysctl -p

echo "Gaming network optimization applied"
EOF

chmod +x /tmp/gaming-network-optimization.sh
/tmp/gaming-network-optimization.sh

# Apply to Node 2 as well
scp /tmp/gaming-network-optimization.sh root@192.168.1.11:/tmp/
ssh root@192.168.1.11 "chmod +x /tmp/gaming-network-optimization.sh && /tmp/gaming-network-optimization.sh"
```

## Step 2: Deploy Game Streaming Server (Sunshine/Moonlight)

### Create Game Streaming VM
```bash
# Create VM for game streaming on Node 1 (better CPU for encoding)
qm create 110 \
    --name dream-stream-server \
    --memory 4096 \
    --cores 3 \
    --net0 virtio,bridge=vmbr80,tag=80 \
    --storage asustor-vm-storage

# Create VM disk
qm set 110 --scsi0 asustor-vm-storage:32

# Configure for gaming workload
qm set 110 --cpu cputype=host
qm set 110 --args '-cpu host,+kvm_pv_unhalt,+kvm_pv_eoi,hv_vendor_id=NV43FIX,kvm=off'

# Set static IP in gaming VLAN
qm set 110 --ipconfig0 ip=192.168.80.103/24,gw=192.168.80.1

# Add GPU passthrough if available (check first)
lspci | grep VGA
# If Intel iGPU available:
# qm set 110 --hostpci0 00:02,pcie=1

# Start gaming VM
qm start 110

# Wait for boot
sleep 45
```

### Configure Game Streaming VM
```bash
# SSH to game streaming VM (use gaming VLAN IP)
ssh ubuntu@192.168.80.103

# Update system and install dependencies
sudo apt update && sudo apt upgrade -y

# Install Sunshine game streaming server
curl -fsSL https://github.com/LizardByte/Sunshine/releases/latest/download/sunshine-ubuntu-22.04-amd64.deb -o sunshine.deb
sudo apt install -y ./sunshine.deb

# Install additional gaming dependencies
sudo apt install -y \
    vainfo \
    mesa-utils \
    vulkan-tools \
    ffmpeg \
    x11vnc \
    xorg \
    openbox \
    pulseaudio

# Create gaming user
sudo useradd -m -s /bin/bash gaming
sudo usermod -aG audio,video,input gaming

# Configure Sunshine
mkdir -p /home/ubuntu/game-streaming
cd /home/ubuntu/game-streaming
```

### Deploy Sunshine Configuration
```bash
# Create Sunshine docker-compose for easier management
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  sunshine:
    image: lizardbyte/sunshine:latest
    container_name: sunshine
    restart: unless-stopped
    hostname: dream-stream-server
    ports:
      - "47984:47984"     # HTTPS Web UI
      - "47989:47989"     # HTTPS Web UI
      - "47990:47990/udp" # Control
      - "48010:48010"     # RTSP
      - "48000:48000/udp" # Video
      - "48001:48001/udp" # Audio
      - "48002:48002/udp" # Video (backup)
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - DISPLAY=:0
    volumes:
      - ./config:/config
      - ./logs:/var/log/sunshine
      - /tmp/.X11-unix:/tmp/.X11-unix:rw
      - /dev/input:/dev/input:ro
      - /run/udev:/run/udev:ro
    devices:
      - /dev/dri:/dev/dri
    network_mode: host
    privileged: true
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.sunshine.entrypoints=websecure"
      - "traefik.http.routers.sunshine.rule=Host(`gaming.snorlax.me`)"
      - "traefik.http.routers.sunshine.middlewares=admin-whitelist@file,security-headers@file"
      - "traefik.http.routers.sunshine.tls=true"
      - "traefik.http.routers.sunshine.tls.certresolver=cloudflare"
      - "traefik.http.services.sunshine.loadbalancer.server.port=47990"

  retropie:
    image: dtcooper/raspberrypi-os:latest
    container_name: retro-dreamer
    restart: unless-stopped
    hostname: retro-dreamer
    ports:
      - "22022:22"        # SSH
      - "80:80"           # EmulationStation Web UI
      - "8080:8080"       # RetroArch Web UI
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
    volumes:
      - ./retropie/config:/opt/retropie/configs
      - ./retropie/roms:/home/pi/RetroPie/roms
      - ./retropie/bios:/home/pi/RetroPie/BIOS
      - ./retropie/saves:/home/pi/RetroPie/saves
    devices:
      - /dev/input:/dev/input
      - /dev/dri:/dev/dri
    networks:
      - gaming-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.retropie.entrypoints=websecure"
      - "traefik.http.routers.retropie.rule=Host(`retro.snorlax.me`)"
      - "traefik.http.routers.retropie.middlewares=security-headers@file"
      - "traefik.http.routers.retropie.tls=true"
      - "traefik.http.routers.retropie.tls.certresolver=cloudflare"

networks:
  gaming-network:
    driver: bridge
    ipam:
      config:
        - subnet: 192.168.80.0/24
          gateway: 192.168.80.1
EOF

# Create gaming network
docker network create gaming-network \
    --driver=bridge \
    --subnet=192.168.80.0/24 \
    --gateway=192.168.80.1 || echo "Network already exists"

# Start gaming services
docker-compose up -d

# Check service status
docker-compose ps
```

## Step 3: Configure Retro Gaming Infrastructure

### Set Up RetroPie/Batocera Alternative
```bash
# Since we're using Docker, create a custom retro gaming setup
mkdir -p retropie/{config,roms,bios,saves}

# Create RetroArch configuration for web-based retro gaming
cat > retropie/config/retroarch.cfg << 'EOF'
# RetroArch configuration for web-based gaming

# Video settings
video_driver = "gl"
video_fullscreen = false
video_windowed_fullscreen = false
video_smooth = true
video_threaded = true

# Audio settings
audio_driver = "pulse"
audio_enable = true
audio_out_rate = 48000

# Input settings
input_driver = "udev"
input_joypad_driver = "udev"

# Network settings
network_cmd_enable = true
netplay_enable = true

# Web interface
web_ui_enable = true
web_ui_port = 8080

# Logging
log_verbosity = true
EOF

# Create ROM organization script
cat > scripts/organize-roms.sh << 'EOF'
#!/bin/bash
# ROM organization and management

ROMS_DIR="/home/ubuntu/game-streaming/retropie/roms"
LOG_FILE="/var/log/rom-management.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$LOG_FILE"
}

# Create console directories
create_console_dirs() {
    CONSOLES=(
        "arcade"
        "atari2600"
        "nes" 
        "snes"
        "gb"
        "gbc"
        "gba"
        "n64"
        "genesis"
        "segacd"
        "saturn"
        "dreamcast"
        "psx"
        "ps2"
        "psp"
        "nds"
        "3ds"
    )
    
    for console in "${CONSOLES[@]}"; do
        mkdir -p "$ROMS_DIR/$console"
        log_message "Created directory for $console"
    done
}

# Scan for ROM files and organize
organize_roms() {
    log_message "Starting ROM organization"
    
    # Count ROMs by system
    for dir in "$ROMS_DIR"/*; do
        if [ -d "$dir" ]; then
            CONSOLE=$(basename "$dir")
            ROM_COUNT=$(find "$dir" -name "*.zip" -o -name "*.7z" -o -name "*.rom" -o -name "*.iso" | wc -l)
            
            if [ "$ROM_COUNT" -gt 0 ]; then
                log_message "Console $CONSOLE: $ROM_COUNT ROMs"
            fi
        fi
    done
    
    log_message "ROM organization completed"
}

# Create console directories and organize
create_console_dirs
organize_roms
EOF

chmod +x scripts/organize-roms.sh
./scripts/organize-roms.sh
```

## Step 4: Deploy Game Library Management

### Create Game Library VM
```bash
# Create VM for game library management
qm create 111 \
    --name game-librarian \
    --memory 2048 \
    --cores 2 \
    --net0 virtio,bridge=vmbr80,tag=80 \
    --storage asustor-vm-storage

# Create VM disk
qm set 111 --scsi0 asustor-vm-storage:20

# Set static IP in gaming VLAN
qm set 111 --ipconfig0 ip=192.168.80.101/24,gw=192.168.80.1

# Start game library VM
qm start 111

# Wait for boot
sleep 30
```

### Configure Game Library Management
```bash
# SSH to game library VM
ssh ubuntu@192.168.80.101

# Create game library management system
mkdir -p /home/ubuntu/game-library
cd /home/ubuntu/game-library

# Deploy RomM (ROM Manager) for game library organization
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  romm:
    image: rommapp/romm:latest
    container_name: romm
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - DB_HOST=192.168.1.15
      - DB_PORT=5432
      - DB_NAME=romm_db
      - DB_USER=romm_user
      - DB_PASSWD=romm_secure_password_change_me
      - IGDB_CLIENT_ID=your-igdb-client-id
      - IGDB_CLIENT_SECRET=your-igdb-client-secret
      - STEAMGRIDDB_API_KEY=your-steamgriddb-api-key
    volumes:
      - ./config:/romm/config
      - ./library:/romm/library
      - ./resources:/romm/resources
      - ./logs:/romm/logs
      - /mnt/games:/romm/assets
    networks:
      - gaming-network
      - shared-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.romm.entrypoints=websecure"
      - "traefik.http.routers.romm.rule=Host(`games.snorlax.me`)"
      - "traefik.http.routers.romm.middlewares=security-headers@file"
      - "traefik.http.routers.romm.tls=true"
      - "traefik.http.routers.romm.tls.certresolver=cloudflare"

  gameserver:
    image: linuxserver/gameserver:latest
    container_name: gameserver
    restart: unless-stopped
    ports:
      - "25565:25565"     # Minecraft
      - "7777:7777/udp"   # Generic game server
      - "27015:27015"     # Steam/Source games
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/New_York
      - GAME_ID=minecraft
      - GAME_PARAMS=-Xmx2G -Xms1G
    volumes:
      - ./gameserver:/config
      - ./worlds:/worlds
    networks:
      - gaming-network

networks:
  gaming-network:
    external: true
  shared-network:
    external: true
EOF

# Mount games storage from NAS
sudo mkdir -p /mnt/games
sudo apt install -y nfs-common

# Add games NFS mount
echo "192.168.10.20:/mnt/pool/games /mnt/games nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2 0 0" | sudo tee -a /etc/fstab
sudo mount -a

# Create shared and gaming networks
docker network create gaming-network --driver=bridge --subnet=192.168.80.0/24 || echo "Network exists"
docker network create shared-network --driver=bridge --subnet=172.20.0.0/16 || echo "Network exists"

# Set up game library database
ssh ubuntu@192.168.1.15 "cd /home/ubuntu/shared-infra && ./scripts/deploy-new-service.sh romm"

# Start game library services
docker-compose up -d
```

## Step 5: Configure Gaming Performance Monitoring

### Deploy Gaming-Specific Monitoring
```bash
# SSH back to monitoring VM to add gaming metrics
ssh ubuntu@192.168.1.22

cd /home/ubuntu/monitoring

# Add gaming infrastructure to Prometheus configuration
cat >> prometheus/prometheus.yml << 'EOF'

  # Gaming infrastructure
  - job_name: 'gaming-infrastructure'
    static_configs:
      - targets:
        - '192.168.80.103:9100'  # Game streaming server
        - '192.168.80.101:9100'  # Game library server
        - '192.168.80.10:9100'   # Beast rig (if node exporter installed)
    scrape_interval: 15s
    metrics_path: /metrics

  # Gaming services
  - job_name: 'gaming-services'
    static_configs:
      - targets:
        - '192.168.80.103:9200'  # Sunshine metrics
        - '192.168.80.101:8080'  # Game library web UI
    scrape_interval: 30s
    timeout: 10s
EOF

# Create gaming performance dashboard
cat > grafana/dashboard-configs/gaming-performance.json << 'EOF'
{
  "dashboard": {
    "id": null,
    "title": "Gaming Infrastructure Performance",
    "description": "Monitor gaming services, streaming performance, and network latency",
    "tags": ["gaming", "performance"],
    "timezone": "browser",
    "refresh": "10s",
    "time": {
      "from": "now-30m",
      "to": "now"
    },
    "panels": [
      {
        "id": 1,
        "title": "Gaming Service Status",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=\"gaming-infrastructure\"}",
            "legendFormat": "{{ instance }}"
          }
        ],
        "gridPos": {"h": 4, "w": 24, "x": 0, "y": 0},
        "options": {
          "colorMode": "background",
          "graphMode": "none"
        },
        "fieldConfig": {
          "defaults": {
            "mappings": [
              {"options": {"0": {"text": "DOWN", "color": "red"}}, "type": "value"},
              {"options": {"1": {"text": "UP", "color": "green"}}, "type": "value"}
            ]
          }
        }
      },
      {
        "id": 2,
        "title": "Gaming Network Latency",
        "type": "timeseries",
        "targets": [
          {
            "expr": "ping_rtt_milliseconds{instance=~\"192.168.80.*\"}",
            "legendFormat": "{{ instance }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4}
      },
      {
        "id": 3,
        "title": "Stream Server Performance",
        "type": "timeseries",
        "targets": [
          {
            "expr": "100 - (avg(irate(node_cpu_seconds_total{instance=\"192.168.80.103:9100\",mode=\"idle\"}[5m])) * 100)",
            "legendFormat": "CPU Usage %"
          },
          {
            "expr": "(node_memory_MemTotal_bytes{instance=\"192.168.80.103:9100\"} - node_memory_MemAvailable_bytes{instance=\"192.168.80.103:9100\"}) / node_memory_MemTotal_bytes{instance=\"192.168.80.103:9100\"} * 100",
            "legendFormat": "Memory Usage %"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 4}
      }
    ]
  }
}
EOF

# Restart Prometheus to load new configuration
docker-compose restart prometheus

# Import gaming dashboard to Grafana
sleep 30
curl -X POST \
    -H "Content-Type: application/json" \
    -u "admin:${GRAFANA_ADMIN_PASSWORD}" \
    -d @grafana/dashboard-configs/gaming-performance.json \
    http://localhost:3000/api/dashboards/db
```

### Create Gaming Performance Testing
```bash
# Create gaming performance testing suite
cat > scripts/gaming-performance-test.sh << 'EOF'
#!/bin/bash
# Gaming infrastructure performance testing

LOG_FILE="/var/log/gaming-performance.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Test network latency to gaming services
test_gaming_latency() {
    log_message "=== Gaming Latency Test ==="
    
    GAMING_TARGETS=(
        "192.168.80.103"  # Game streaming server
        "192.168.80.101"  # Game library server
        "192.168.80.10"   # Beast rig (if accessible)
    )
    
    for target in "${GAMING_TARGETS[@]}"; do
        if ping -c 1 "$target" > /dev/null 2>&1; then
            LATENCY=$(ping -c 10 "$target" | grep 'rtt' | cut -d'=' -f2 | cut -d'/' -f2)
            log_message "$target: ${LATENCY}ms average latency"
        else
            log_message "$target: Not reachable"
        fi
    done
}

# Test gaming service responsiveness
test_gaming_services() {
    log_message "=== Gaming Services Test ==="
    
    # Test Sunshine (if running)
    if curl -s http://192.168.80.103:47990 > /dev/null; then
        log_message "Sunshine: Responsive"
    else
        log_message "Sunshine: Not responding"
    fi
    
    # Test game library
    if curl -s http://192.168.80.101:8080 > /dev/null; then
        log_message "Game Library: Responsive"
    else
        log_message "Game Library: Not responding"
    fi
}

# Test streaming performance
test_streaming_performance() {
    log_message "=== Streaming Performance Test ==="
    
    # Check CPU usage on streaming server
    if command -v ssh >/dev/null; then
        CPU_USAGE=$(ssh ubuntu@192.168.80.103 "top -bn1 | grep 'Cpu(s)' | cut -d',' -f1 | cut -d':' -f2 | tr -d ' %us'" 2>/dev/null)
        if [ ! -z "$CPU_USAGE" ]; then
            log_message "Stream Server CPU: ${CPU_USAGE}%"
        fi
        
        # Check GPU usage (if available)
        GPU_USAGE=$(ssh ubuntu@192.168.80.103 "nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader,nounits" 2>/dev/null || echo "N/A")
        log_message "Stream Server GPU: ${GPU_USAGE}%"
    fi
}

# Run all gaming performance tests
test_gaming_latency
test_gaming_services
test_streaming_performance

log_message "Gaming performance test completed"
EOF

chmod +x scripts/gaming-performance-test.sh

# Schedule gaming performance tests every 5 minutes
echo "*/5 * * * * /home/ubuntu/monitoring/scripts/gaming-performance-test.sh" | crontab -l | { cat; echo "*/5 * * * * /home/ubuntu/monitoring/scripts/gaming-performance-test.sh"; } | crontab -
```

## Step 6: Configure Gaming Device Integration

### Set Up Gaming Device Management
```bash
# Create gaming device configuration script
cat > scripts/gaming-device-setup.sh << 'EOF'
#!/bin/bash
# Gaming device integration and management

echo "=== Gaming Device Setup Guide ==="

# Mighty Snorlax Configuration
echo "Mighty Snorlax (192.168.80.10) Configuration:"
echo "1. Install Moonlight client for game streaming"
echo "2. Configure static IP: 192.168.80.10"
echo "3. Install Steam, Epic Games, Battle.net launchers"
echo "4. Configure Sunshine for hosting (if desired)"
echo "5. Install MSI Afterburner for GPU monitoring"
echo ""

# Steam Deck Configuration  
echo "Steam Deck (192.168.80.20) Configuration:"
echo "1. Add to gaming VLAN via Wi-Fi settings"
echo "2. Install Moonlight for streaming from desktop"
echo "3. Configure RetroArch for emulation"
echo "4. Set up ROM sync with game library server"
echo "5. Install Tailscale for external access (optional)"
echo ""

# Console Configuration
echo "Gaming Console Configuration:"
echo "1. PS5/Xbox (192.168.80.40): Configure for low-latency gaming"
echo "2. Switch (192.168.80.42): Connect to gaming Wi-Fi"
echo "3. Enable UPnP if needed for console online features"
echo "4. Configure QoS for console traffic"
echo ""

# Create device monitoring script
cat > /tmp/gaming-device-monitor.sh << 'DEVICE_EOF'
#!/bin/bash
# Monitor gaming devices on network

GAMING_DEVICES=(
    "192.168.80.10:mighty-snorlax"
    "192.168.80.20:sleepy-deck"
    "192.168.80.40:dreamy-console-1"
    "192.168.80.42:drowsy-switch"
)

echo "Gaming Device Status Check:"
for device_info in "${GAMING_DEVICES[@]}"; do
    IP=$(echo "$device_info" | cut -d':' -f1)
    NAME=$(echo "$device_info" | cut -d':' -f2)
    
    if ping -c 1 -W 2 "$IP" > /dev/null 2>&1; then
        echo "✅ $NAME ($IP): Online"
    else
        echo "❌ $NAME ($IP): Offline"
    fi
done
DEVICE_EOF

chmod +x /tmp/gaming-device-monitor.sh
/tmp/gaming-device-monitor.sh
EOF

chmod +x scripts/gaming-device-setup.sh
./scripts/gaming-device-setup.sh
```

### Configure Game Streaming Optimization
```bash
# SSH back to game streaming server
ssh ubuntu@192.168.80.103

cd /home/ubuntu/game-streaming

# Create streaming optimization script
cat > scripts/optimize-streaming.sh << 'EOF'
#!/bin/bash
# Optimize game streaming performance

# GPU acceleration settings (if available)
setup_gpu_acceleration() {
    echo "Setting up GPU acceleration..."
    
    # Check for Intel iGPU
    if lspci | grep -i "intel.*graphics" > /dev/null; then
        sudo apt install -y intel-media-va-driver-non-free intel-gpu-tools
        export LIBVA_DRIVER_NAME=iHD
        echo "Intel GPU acceleration configured"
    fi
    
    # Check for NVIDIA GPU  
    if lspci | grep -i nvidia > /dev/null; then
        # Install NVIDIA drivers if not present
        sudo apt install -y nvidia-driver-470 nvidia-settings
        echo "NVIDIA GPU acceleration configured"
    fi
}

# Network optimization for streaming
optimize_network() {
    echo "Optimizing network for streaming..."
    
    # Increase network buffers for streaming
    sudo sysctl -w net.core.rmem_max=134217728
    sudo sysctl -w net.core.wmem_max=134217728
    sudo sysctl -w net.ipv4.tcp_rmem="4096 16384 134217728"
    sudo sysctl -w net.ipv4.tcp_wmem="4096 16384 134217728"
    
    # Reduce network latency
    sudo sysctl -w net.ipv4.tcp_low_latency=1
    sudo sysctl -w net.ipv4.tcp_timestamps=0
    
    echo "Network optimization applied"
}

# Audio optimization
optimize_audio() {
    echo "Optimizing audio for streaming..."
    
    # Configure PulseAudio for low latency
    cat > ~/.pulse/daemon.conf << 'PULSE_EOF'
default-sample-format = s16le
default-sample-rate = 48000
default-sample-channels = 2
default-fragments = 2
default-fragment-size-msec = 5
enable-lfe-remixing = no
high-priority = yes
nice-level = -11
realtime-scheduling = yes
realtime-priority = 9
rlimit-rtprio = 9
rlimit-rttime = 200000
PULSE_EOF
    
    # Restart PulseAudio
    pulseaudio -k
    pulseaudio --start
    
    echo "Audio optimization applied"
}

# Execute optimizations
setup_gpu_acceleration
optimize_network  
optimize_audio

echo "Game streaming optimization completed"
EOF

chmod +x scripts/optimize-streaming.sh
./scripts/optimize-streaming.sh
```

## Step 7: Create Gaming Service Health Monitoring

### Configure Gaming Service Monitoring
```bash
# Create gaming service health monitoring
cat > scripts/gaming-health-monitor.sh << 'EOF'
#!/bin/bash
# Gaming infrastructure health monitoring

LOG_FILE="/var/log/gaming-health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Check game streaming server health
check_streaming_server() {
    log_message "=== Game Streaming Server Health ==="
    
    # Check Sunshine service
    if curl -s http://192.168.80.103:47990 > /dev/null; then
        log_message "Sunshine: Healthy"
    else
        log_message "Sunshine: Not responding"
    fi
    
    # Check system resources on streaming server
    if command -v ssh >/dev/null; then
        CPU_TEMP=$(ssh ubuntu@192.168.80.103 "sensors 2>/dev/null | grep 'Core 0' | awk '{print \$3}' | cut -d'+' -f2 | cut -d'°' -f1" 2>/dev/null)
        if [ ! -z "$CPU_TEMP" ]; then
            log_message "Stream Server CPU Temp: ${CPU_TEMP}°C"
            
            if [ "$CPU_TEMP" -gt 80 ]; then
                log_message "WARNING: High CPU temperature on streaming server"
            fi
        fi
        
        # Check GPU temperature (if available)
        GPU_TEMP=$(ssh ubuntu@192.168.80.103 "nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader,nounits" 2>/dev/null)
        if [ ! -z "$GPU_TEMP" ] && [ "$GPU_