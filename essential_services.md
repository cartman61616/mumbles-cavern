# Phase 5: Essential Services Deployment

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [Shared Infrastructure](05-shared-infrastructure.md)  
**Next Phase**: [Monitoring](07-monitoring.md)  
**Duration**: 4-6 hours  
**Prerequisites**: Shared infrastructure operational, storage configured

## Overview
Deploy core homelab applications including Plex media server and the *arr stack for media management, all integrated with your shared infrastructure.

## Step 1: Create VM Templates

### Download Cloud Images
```bash
# SSH to Node 1 (or use web shell)
ssh root@192.168.1.10

# Download cloud images to shared ISO storage
cd /mnt/pve/asustor-iso/

# Ubuntu 22.04 LTS cloud image
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Ubuntu 24.04 LTS cloud image  
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img

# Debian 12 cloud image
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2

# Verify downloads
ls -lh /mnt/pve/asustor-iso/
```

### Create Ubuntu 22.04 Template
```bash
# Create VM for template
qm create 9000 \
    --memory 2048 \
    --cores 2 \
    --name ubuntu-22-template \
    --net0 virtio,bridge=vmbr0 \
    --storage asustor-vm-storage

# Import cloud image disk
qm importdisk 9000 /mnt/pve/asustor-iso/jammy-server-cloudimg-amd64.img asustor-vm-storage

# Configure VM hardware
qm set 9000 --scsihw virtio-scsi-pci --scsi0 asustor-vm-storage:vm-9000-disk-0
qm set 9000 --ide2 asustor-vm-storage:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0

# Configure cloud-init
qm set 9000 --ciuser ubuntu
qm set 9000 --cipassword $(openssl rand -base64 12)
qm set 9000 --ipconfig0 ip=dhcp
qm set 9000 --nameserver 192.168.1.1
qm set 9000 --searchdomain mumblescavern.local

# Add SSH key for template access
cat /root/.ssh/vm_template_key.pub | qm set 9000 --sshkeys -

# Resize disk to reasonable size
qm resize 9000 scsi0 20G

# Start VM to customize
qm start 9000

# Wait for boot and get IP
sleep 30
VM_IP=$(qm guest cmd 9000 network-get-interfaces | grep -oP '(?<="ip-address":")[^"]*' | grep -v 127.0.0.1 | head -1)
echo "Template VM IP: $VM_IP"
```

### Customize Template
```bash
# SSH to template VM (use the IP from above)
ssh -i /root/.ssh/vm_template_key ubuntu@$VM_IP

# Update system
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y \
    curl \
    wget \
    git \
    vim \
    htop \
    tree \
    unzip \
    software-properties-common \
    apt-transport-https \
    ca-certificates \
    gnupg \
    lsb-release \
    prometheus-node-exporter

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu

# Configure automatic updates
sudo apt install -y unattended-upgrades
echo 'Unattended-Upgrade::Automatic-Reboot "false";' | sudo tee -a /etc/apt/apt.conf.d/50unattended-upgrades

# Clean up system
sudo apt autoremove -y
sudo apt autoclean
sudo rm -rf /tmp/*
sudo rm -rf /var/tmp/*

# Clear bash history and SSH host keys (will be regenerated)
history -c && history -w
sudo rm -f /etc/ssh/ssh_host_*
sudo rm -f /home/ubuntu/.ssh/known_hosts

# Shutdown template VM
sudo shutdown -h now
```

### Convert to Template
```bash
# Wait for shutdown, then convert to template
qm stop 9000 --timeout 30
qm template 9000

# Verify template creation
qm list | grep template
```

## Step 2: Deploy Plex Media Server

### Create Plex VM
```bash
# Clone template for Plex server
qm clone 9000 101 --name plex-server --full --storage asustor-vm-storage

# Configure for media server workload
qm set 101 --memory 8192 --cores 4
qm set 101 --ipconfig0 ip=192.168.1.20/24,gw=192.168.1.1

# Add additional storage for transcoding (use local fast storage if available)
qm set 101 --scsi1 local:32 --scsihw virtio-scsi-pci

# Enable hardware passthrough for transcoding (if available)
# Check if Intel iGPU is available
lspci | grep VGA

# If Intel graphics available, add GPU passthrough
# qm set 101 --hostpci0 00:02

# Start Plex VM
qm start 101

# Wait for boot and verify IP
sleep 30
qm guest cmd 101 network-get-interfaces
```

### Configure Plex VM
```bash
# SSH to Plex VM
ssh ubuntu@192.168.1.20

# Create Plex directory structure
mkdir -p /home/ubuntu/plex/{config,transcode}
cd /home/ubuntu/plex

# Mount media storage from NAS
sudo mkdir -p /mnt/media
sudo apt install -y nfs-common

# Add NFS mount to fstab for persistence
echo "192.168.10.20:/mnt/pool/media /mnt/media nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2 0 0" | sudo tee -a /etc/fstab

# Mount media storage
sudo mount -a
ls -la /mnt/media

# Create Docker network connection to shared infrastructure
docker network create shared-network --subnet=172.20.0.0/16 || echo "Network already exists"
```

### Deploy Plex Container
```bash
# Create environment file for Plex
cat > .env << 'EOF'
# Plex Configuration
PLEX_CLAIM=claim-token-from-plex.tv-goes-here
PLEX_UID=1000
PLEX_GID=1000
TZ=America/New_York

# Shared Infrastructure
POSTGRES_HOST=192.168.1.15
POSTGRES_PORT=5432
REDIS_HOST=192.168.1.15
REDIS_PORT=6379
EOF

# Create Plex docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  plex:
    image: plexinc/pms-docker:latest
    container_name: plex
    restart: unless-stopped
    hostname: plex-server
    ports:
      - "32400:32400/tcp"
      - "3005:3005/tcp"
      - "8324:8324/tcp"
      - "32469:32469/tcp"
      - "1900:1900/udp"
      - "32410:32410/udp"
      - "32412:32412/udp"
      - "32413:32413/udp"
      - "32414:32414/udp"
    environment:
      - PLEX_CLAIM=${PLEX_CLAIM}
      - PLEX_UID=${PLEX_UID}
      - PLEX_GID=${PLEX_GID}
      - TZ=${TZ}
      - ADVERTISE_IP=http://192.168.1.20:32400/
    volumes:
      - ./config:/config
      - ./transcode:/transcode
      - /mnt/media:/data/media:ro
      - /dev/shm:/ram-transcode
    devices:
      - /dev/dri:/dev/dri  # Hardware transcoding (if available)
    networks:
      - shared-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:32400/web"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.plex.entrypoints=websecure"
      - "traefik.http.routers.plex.rule=Host(`plex.snorlax.me`)"
      - "traefik.http.routers.plex.tls=true"
      - "traefik.http.routers.plex.tls.certresolver=cloudflare"
      - "traefik.http.services.plex.loadbalancer.server.port=32400"

  tautulli:
    image: linuxserver/tautulli:latest
    container_name: tautulli
    restart: unless-stopped
    environment:
      - PUID=${PLEX_UID}
      - PGID=${PLEX_GID}
      - TZ=${TZ}
    volumes:
      - ./tautulli-config:/config
    ports:
      - "8181:8181"
    networks:
      - shared-network
    depends_on:
      - plex
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tautulli.entrypoints=websecure"
      - "traefik.http.routers.tautulli.rule=Host(`tautulli.snorlax.me`)"
      - "traefik.http.routers.tautulli.tls=true"
      - "traefik.http.routers.tautulli.tls.certresolver=cloudflare"

networks:
  shared-network:
    external: true
EOF

# Start Plex services
docker-compose up -d

# Check service status
docker-compose ps
docker-compose logs plex
```

### Configure Plex Server
1. **Access Plex Web UI:** http://192.168.1.20:32400/web
2. **Complete Plex setup:**
   - Sign in with Plex account
   - Claim server with token from plex.tv/claim
   - Add media libraries pointing to /data/media
   - Configure transcoding settings
   - Enable hardware acceleration (if available)

## Step 3: Deploy *arr Stack

### Create *arr Stack VM
```bash
# Clone template for *arr services
qm clone 9000 102 --name arr-stack --full --storage asustor-vm-storage

# Configure for media management workload
qm set 102 --memory 6144 --cores 3
qm set 102 --ipconfig0 ip=192.168.1.21/24,gw=192.168.1.1

# Add additional storage for download processing
qm set 102 --scsi1 local:50

# Start *arr VM
qm start 102

# Wait for boot
sleep 30
```

### Configure *arr VM
```bash
# SSH to *arr VM
ssh ubuntu@192.168.1.21

# Mount NFS shares for media and downloads
sudo mkdir -p /mnt/{downloads,media}
sudo apt install -y nfs-common

# Add NFS mounts to fstab
echo "192.168.10.20:/mnt/pool/downloads /mnt/downloads nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2 0 0" | sudo tee -a /etc/fstab
echo "192.168.10.20:/mnt/pool/media /mnt/media nfs rsize=131072,wsize=131072,hard,intr,timeo=14,retrans=2 0 0" | sudo tee -a /etc/fstab

# Mount shares
sudo mount -a

# Verify mounts
df -h | grep nfs

# Create Docker network
docker network create shared-network --subnet=172.20.0.0/16 || echo "Network already exists"

# Create *arr stack directory
mkdir -p /home/ubuntu/arr-stack
cd /home/ubuntu/arr-stack
```

### Deploy *arr Stack Services
```bash
# Create environment file
cat > .env << 'EOF'
# *arr Stack Configuration
PUID=1000
PGID=1000
TZ=America/New_York

# Shared Infrastructure
POSTGRES_HOST=192.168.1.15
POSTGRES_PORT=5432
REDIS_HOST=192.168.1.15
REDIS_PORT=6379
REDIS_PASSWORD=your-secure-redis-password-here

# Service-specific settings
SONARR_API_KEY=generate-this-after-first-run
RADARR_API_KEY=generate-this-after-first-run
PROWLARR_API_KEY=generate-this-after-first-run
EOF

# Create docker-compose.yml for *arr stack
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    ports:
      - "8989:8989"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/sonarr:/config
      - /mnt/media/tv:/tv
      - /mnt/downloads:/downloads
    networks:
      - shared-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8989/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.sonarr.entrypoints=websecure"
      - "traefik.http.routers.sonarr.rule=Host(`sonarr.snorlax.me`)"
      - "traefik.http.routers.sonarr.tls=true"
      - "traefik.http.routers.sonarr.tls.certresolver=cloudflare"

  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    ports:
      - "7878:7878"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/radarr:/config
      - /mnt/media/movies:/movies
      - /mnt/downloads:/downloads
    networks:
      - shared-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:7878/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.radarr.entrypoints=websecure"
      - "traefik.http.routers.radarr.rule=Host(`radarr.snorlax.me`)"
      - "traefik.http.routers.radarr.tls=true"
      - "traefik.http.routers.radarr.tls.certresolver=cloudflare"

  prowlarr:
    image: linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    ports:
      - "9696:9696"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/prowlarr:/config
    networks:
      - shared-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9696/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prowlarr.entrypoints=websecure"
      - "traefik.http.routers.prowlarr.rule=Host(`prowlarr.snorlax.me`)"
      - "traefik.http.routers.prowlarr.tls=true"
      - "traefik.http.routers.prowlarr.tls.certresolver=cloudflare"

  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
    volumes:
      - ./config/qbittorrent:/config
      - /mnt/downloads:/downloads
    networks:
      - shared-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.qbittorrent.entrypoints=websecure"
      - "traefik.http.routers.qbittorrent.rule=Host(`qbittorrent.snorlax.me`)"
      - "traefik.http.routers.qbittorrent.tls=true"
      - "traefik.http.routers.qbittorrent.tls.certresolver=cloudflare"

  bazarr:
    image: linuxserver/bazarr:latest
    container_name: bazarr
    restart: unless-stopped
    ports:
      - "6767:6767"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/bazarr:/config
      - /mnt/media/movies:/movies
      - /mnt/media/tv:/tv
    networks:
      - shared-network
    depends_on:
      - sonarr
      - radarr
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.bazarr.entrypoints=websecure"
      - "traefik.http.routers.bazarr.rule=Host(`bazarr.snorlax.me`)"
      - "traefik.http.routers.bazarr.tls=true"
      - "traefik.http.routers.bazarr.tls.certresolver=cloudflare"

networks:
  shared-network:
    external: true
EOF

# Start *arr stack
docker-compose up -d

# Check service status
docker-compose ps
docker-compose logs --tail=20
```

## Step 4: Configure Service Integration

### Prowlarr Configuration
1. **Access Prowlarr:** http://192.168.1.21:9696
2. **Add indexers:**
   - Configure public indexers (1337x, RARBG alternatives, etc.)
   - Add private indexers if you have accounts
3. **Configure applications:**
   - Add Sonarr: http://192.168.1.21:8989
   - Add Radarr: http://192.168.1.21:7878
   - Use API keys (generated after first access to each service)

### Sonarr Configuration  
1. **Access Sonarr:** http://192.168.1.21:8989
2. **Configure settings:**
   - Media Management → Add Root Folder: /tv
   - Download Clients → Add qBittorrent:
     - Host: 192.168.1.21
     - Port: 8080
     - Category: tv-sonarr
   - Connect → Add Plex:
     - Server Host: 192.168.1.20
     - Port: 32400
     - Update Library: Yes

### Radarr Configuration
1. **Access Radarr:** http://192.168.1.21:7878  
2. **Configure settings:**
   - Media Management → Add Root Folder: /movies
   - Download Clients → Add qBittorrent:
     - Host: 192.168.1.21
     - Port: 8080
     - Category: movies-radarr
   - Connect → Add Plex:
     - Server Host: 192.168.1.20
     - Port: 32400
     - Update Library: Yes

### qBittorrent Configuration
1. **Access qBittorrent:** http://192.168.1.21:8080
2. **Default login:** admin / adminadmin (change immediately)
3. **Configure settings:**
   - Downloads → Save files to: /downloads
   - Downloads → Create subfolder for: Category
   - Connection → Port: 6881
   - Speed → Alternative Rate Limits (optional)

## Step 5: Deploy Additional Essential Services


## Step 6: Service Health Monitoring

### Create Service Health Dashboard
```bash
# On *arr VM, create service monitoring script
cd /home/ubuntu/arr-stack

cat > scripts/service-health-check.sh << 'EOF'
#!/bin/bash
# *arr stack health monitoring

LOG_FILE="/var/log/arr-health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Check service health
check_service_health() {
    local SERVICE_NAME="$1"
    local PORT="$2"
    local ENDPOINT="$3"
    
    if curl -f -s "http://localhost:$PORT$ENDPOINT" > /dev/null; then
        log_message "$SERVICE_NAME: Healthy"
        return 0
    else
        log_message "$SERVICE_NAME: ERROR - Not responding"
        return 1
    fi
}

# Check all services
SERVICES_OK=0

check_service_health "Sonarr" "8989" "/ping" && ((SERVICES_OK++))
check_service_health "Radarr" "7878" "/ping" && ((SERVICES_OK++))
check_service_health "Prowlarr" "9696" "/ping" && ((SERVICES_OK++))
check_service_health "qBittorrent" "8080" "/" && ((SERVICES_OK++))
check_service_health "Bazarr" "6767" "/" && ((SERVICES_OK++))

# Check download space
DOWNLOAD_USAGE=$(df -h /mnt/downloads | tail -1 | awk '{print $5}' | sed 's/%//')
if [ "$DOWNLOAD_USAGE" -gt 85 ]; then
    log_message "WARNING: Download storage at ${DOWNLOAD_USAGE}%"
fi

# Check media space  
MEDIA_USAGE=$(df -h /mnt/media | tail -1 | awk '{print $5}' | sed 's/%//')
log_message "Storage: Downloads ${DOWNLOAD_USAGE}%, Media ${MEDIA_USAGE}%"

log_message "Health check completed: $SERVICES_OK/5 services healthy"

# Keep log manageable
tail -200 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

mkdir -p scripts
chmod +x scripts/service-health-check.sh

# Schedule health checks every 10 minutes
echo "*/10 * * * * /home/ubuntu/arr-stack/scripts/service-health-check.sh" | crontab -

# Test health check
./scripts/service-health-check.sh
```

### Create Service Integration Tests
```bash
# Create integration test script
cat > scripts/integration-test.sh << 'EOF'
#!/bin/bash
# Integration testing for *arr stack

echo "=== *arr Stack Integration Testing ==="

# Test Prowlarr → Sonarr integration
echo "Testing Prowlarr → Sonarr integration..."
SONARR_API_KEY=$(grep SONARR_API_KEY .env | cut -d'=' -f2)
if [ ! -z "$SONARR_API_KEY" ]; then
    curl -s "http://localhost:8989/api/v3/system/status?apikey=$SONARR_API_KEY" | grep -q '"version"' && echo "✅ Sonarr API accessible" || echo "❌ Sonarr API failed"
else
    echo "⚠️  Sonarr API key not configured yet"
fi

# Test Prowlarr → Radarr integration  
echo "Testing Prowlarr → Radarr integration..."
RADARR_API_KEY=$(grep RADARR_API_KEY .env | cut -d'=' -f2)
if [ ! -z "$RADARR_API_KEY" ]; then
    curl -s "http://localhost:7878/api/v3/system/status?apikey=$RADARR_API_KEY" | grep -q '"version"' && echo "✅ Radarr API accessible" || echo "❌ Radarr API failed"
else
    echo "⚠️  Radarr API key not configured yet"
fi

# Test qBittorrent connectivity
echo "Testing qBittorrent..."
if curl -s http://localhost:8080 | grep -q qBittorrent; then
    echo "✅ qBittorrent web UI accessible"
else
    echo "❌ qBittorrent web UI failed"
fi

# Test NFS mount accessibility
echo "Testing storage mounts..."
if [ -d "/mnt/downloads" ] && mountpoint -q /mnt/downloads; then
    echo "✅ Downloads mount accessible"
else
    echo "❌ Downloads mount failed"
fi

if [ -d "/mnt/media" ] && mountpoint -q /mnt/media; then
    echo "✅ Media mount accessible"
else
    echo "❌ Media mount failed"  
fi

# Test Plex connectivity
echo "Testing Plex connectivity..."
if curl -s http://192.168.1.20:32400/identity | grep -q PlexMediaServer; then
    echo "✅ Plex server accessible"
else
    echo "❌ Plex server not accessible"
fi

echo "=== Integration Testing Complete ==="
EOF

chmod +x scripts/integration-test.sh

# Run integration test
./scripts/integration-test.sh
```

## Step 7: Configure Media Library Management

### Create Media Directory Structure
```bash
# Create standardized media directory structure on NAS
# SSH to ASUSTOR or use web interface to create:

/mnt/pool/media/
├── movies/
│   ├── Movie Name (Year)/
│   └── Movie Name (Year)/
├── tv/
│   ├── Series Name/
│   │   ├── Season 01/
│   │   └── Season 02/
├── music/
│   ├── Artist/
│   │   └── Album/
└── audiobooks/
    ├── Author/
    │   └── Book Title/

/mnt/pool/downloads/
├── complete/
│   ├── movies/
│   ├── tv/
│   └── music/
├── incomplete/
└── torrents/
```

### Configure Automated Media Processing
```bash
# Create media processing script for post-download actions
cat > scripts/media-processor.sh << 'EOF'
#!/bin/bash
# Automated media processing and organization

LOG_FILE="/var/log/media-processing.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Function to clean up completed downloads
cleanup_completed_downloads() {
    log_message "Starting download cleanup"
    
    # Remove completed torrents older than 7 days from downloads
    find /mnt/downloads/complete -name "*.torrent" -mtime +7 -delete
    
    # Clean up empty directories
    find /mnt/downloads/complete -type d -empty -delete
    
    log_message "Download cleanup completed"
}

# Function to monitor download space
monitor_download_space() {
    DOWNLOAD_USAGE=$(df /mnt/downloads | tail -1 | awk '{print $5}' | sed 's/%//')
    
    if [ "$DOWNLOAD_USAGE" -gt 90 ]; then
        log_message "CRITICAL: Download space at ${DOWNLOAD_USAGE}% - cleaning old files"
        
        # Emergency cleanup - remove files older than 3 days
        find /mnt/downloads/complete -type f -mtime +3 -delete
        find /mnt/downloads/incomplete -name "*.!qB" -mtime +1 -delete
        
    elif [ "$DOWNLOAD_USAGE" -gt 80 ]; then
        log_message "WARNING: Download space at ${DOWNLOAD_USAGE}%"
    else
        log_message "Download space usage: ${DOWNLOAD_USAGE}%"
    fi
}

# Function to validate media organization
validate_media_structure() {
    # Count media files
    MOVIE_COUNT=$(find /mnt/media/movies -name "*.mkv" -o -name "*.mp4" -o -name "*.avi" | wc -l)
    TV_COUNT=$(find /mnt/media/tv -name "*.mkv" -o -name "*.mp4" -o -name "*.avi" | wc -l)
    
    log_message "Media inventory: Movies $MOVIE_COUNT, TV episodes $TV_COUNT"
    
    # Check for unorganized files in root directories
    UNORGANIZED_MOVIES=$(find /mnt/media/movies -maxdepth 1 -name "*.mkv" -o -name "*.mp4" -o -name "*.avi" | wc -l)
    UNORGANIZED_TV=$(find /mnt/media/tv -maxdepth 1 -name "*.mkv" -o -name "*.mp4" -o -name "*.avi" | wc -l)
    
    if [ "$UNORGANIZED_MOVIES" -gt 0 ] || [ "$UNORGANIZED_TV" -gt 0 ]; then
        log_message "WARNING: Found unorganized media files - Movies: $UNORGANIZED_MOVIES, TV: $UNORGANIZED_TV"
    fi
}

# Execute functions
cleanup_completed_downloads
monitor_download_space
validate_media_structure

log_message "Media processing cycle completed"
EOF

chmod +x scripts/media-processor.sh

# Schedule media processing every hour
echo "0 * * * * /home/ubuntu/arr-stack/scripts/media-processor.sh" | crontab -l | { cat; echo "0 * * * * /home/ubuntu/arr-stack/scripts/media-processor.sh"; } | crontab -
```

## Step 8: Configure Service Authentication and Security

### Set Up Service API Keys
```bash
# Create script to extract and configure API keys
cat > scripts/configure-api-keys.sh << 'EOF'
#!/bin/bash
# Extract and configure API keys for service integration

echo "=== Configuring API Keys ==="

# Function to extract API key from service config
extract_api_key() {
    local SERVICE="$1"
    local CONFIG_FILE="$2"
    local API_KEY=""
    
    if [ -f "$CONFIG_FILE" ]; then
        API_KEY=$(grep -o '<ApiKey>[^<]*' "$CONFIG_FILE" | cut -d'>' -f2)
        if [ ! -z "$API_KEY" ]; then
            echo "Found $SERVICE API key: $API_KEY"
            return 0
        fi
    fi
    
    echo "$SERVICE API key not found - service may not be initialized yet"
    return 1
}

# Wait for services to initialize (first run creates config files)
echo "Waiting for services to initialize..."
sleep 60

# Extract API keys
SONARR_API_KEY=""
RADARR_API_KEY=""
PROWLARR_API_KEY=""

if extract_api_key "Sonarr" "./config/sonarr/config.xml"; then
    SONARR_API_KEY=$(grep -o '<ApiKey>[^<]*' ./config/sonarr/config.xml | cut -d'>' -f2)
fi

if extract_api_key "Radarr" "./config/radarr/config.xml"; then
    RADARR_API_KEY=$(grep -o '<ApiKey>[^<]*' ./config/radarr/config.xml | cut -d'>' -f2)
fi

if extract_api_key "Prowlarr" "./config/prowlarr/config.xml"; then
    PROWLARR_API_KEY=$(grep -o '<ApiKey>[^<]*' ./config/prowlarr/config.xml | cut -d'>' -f2)
fi

# Update .env file with API keys
if [ ! -z "$SONARR_API_KEY" ]; then
    sed -i "s/SONARR_API_KEY=.*/SONARR_API_KEY=$SONARR_API_KEY/" .env
fi

if [ ! -z "$RADARR_API_KEY" ]; then
    sed -i "s/RADARR_API_KEY=.*/RADARR_API_KEY=$RADARR_API_KEY/" .env
fi

if [ ! -z "$PROWLARR_API_KEY" ]; then
    sed -i "s/PROWLARR_API_KEY=.*/PROWLARR_API_KEY=$PROWLARR_API_KEY/" .env
fi

echo "API key configuration completed"
echo "Updated .env file with extracted keys"
EOF

chmod +x scripts/configure-api-keys.sh

# Run after services have been running for a few minutes
sleep 120
./scripts/configure-api-keys.sh
```

### Configure Service Security
```bash
# Create security hardening script for services
cat > scripts/harden-services.sh << 'EOF'
#!/bin/bash
# Security hardening for *arr services

echo "Hardening service security..."

# Configure qBittorrent security
cat > ./config/qbittorrent/qBittorrent/qBittorrent.conf << 'INI_EOF'
[BitTorrent]
Session\DisableAutoTMMByDefault=false
Session\DisableAutoTMMTriggers\CategorySavePathChanged=true
Session\DisableAutoTMMTriggers\DefaultSavePathChanged=true

[Network]
PortRangeMin=6881
PortRangeMax=6881
UPnP=false

[Preferences]
Connection\PortRangeMin=6881
Connection\PortRangeMax=6881
Connection\UPnP=false
Downloads\SavePath=/downloads/
Downloads\TempPathEnabled=true
Downloads\TempPath=/downloads/incomplete/
General\Locale=en
WebUI\Address=*
WebUI\Port=8080
WebUI\UseUPnP=false
WebUI\CSRFProtection=true
WebUI\ClickjackingProtection=true
WebUI\SecureCookie=true
INI_EOF

# Restart qBittorrent to apply security settings
docker-compose restart qbittorrent

echo "Service security hardening completed"
EOF

chmod +x scripts/harden-services.sh
./scripts/harden-services.sh
```

## Step 9: Create Service Backup Strategy

### Configure Service-Specific Backups
```bash
# Create backup script for service configurations
cat > scripts/service-backup.sh << 'EOF'
#!/bin/bash
# Backup service configurations and data

BACKUP_DIR="/home/ubuntu/arr-stack/backups"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/service-backup.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$LOG_FILE"
}

mkdir -p "$BACKUP_DIR"

# Backup service configurations
backup_service_configs() {
    log_message "Starting service configuration backup"
    
    # Create tarball of all service configurations
    tar -czf "$BACKUP_DIR/arr-configs_$DATE.tar.gz" \
        docker-compose.yml \
        .env \
        config/ \
        scripts/
    
    # Backup individual service databases/configs
    for SERVICE in sonarr radarr prowlarr bazarr; do
        if [ -d "config/$SERVICE" ]; then
            tar -czf "$BACKUP_DIR/${SERVICE}_config_$DATE.tar.gz" config/$SERVICE/
            log_message "Backed up $SERVICE configuration"
        fi
    done
    
    log_message "Service configuration backup completed"
}

# Backup qBittorrent data
backup_qbittorrent() {
    log_message "Starting qBittorrent backup"
    
    # Backup torrent files and fastresume data
    if [ -d "config/qbittorrent" ]; then
        tar -czf "$BACKUP_DIR/qbittorrent_data_$DATE.tar.gz" config/qbittorrent/
        log_message "qBittorrent backup completed"
    fi
}

# Cleanup old backups
cleanup_old_backups() {
    log_message "Starting backup cleanup"
    
    # Remove backups older than 30 days
    find "$BACKUP_DIR" -name "*.tar.gz" -mtime +30 -delete
    
    log_message "Backup cleanup completed"
}

# Execute backup functions
backup_service_configs
backup_qbittorrent
cleanup_old_backups

# Copy critical backups to shared storage
cp "$BACKUP_DIR"/arr-configs_$DATE.tar.gz /mnt/pve/asustor-backup/ 2>/dev/null || true

log_message "Service backup cycle completed"
EOF

chmod +x scripts/service-backup.sh

# Schedule service backups daily at 3 AM
echo "0 3 * * * /home/ubuntu/arr-stack/scripts/service-backup.sh" | crontab -l | { cat; echo "0 3 * * * /home/ubuntu/arr-stack/scripts/service-backup.sh"; } | crontab -
```

## Step 10: Service Performance Optimization

### Configure Service Resource Limits
```bash
# Update docker-compose.yml with resource limits
cat > docker-compose-optimized.yml << 'EOF'
version: '3.8'

services:
  sonarr:
    image: linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    ports:
      - "8989:8989"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/sonarr:/config
      - /mnt/media/tv:/tv
      - /mnt/downloads:/downloads
    networks:
      - shared-network
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8989/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  radarr:
    image: linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    ports:
      - "7878:7878"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/radarr:/config
      - /mnt/media/movies:/movies
      - /mnt/downloads:/downloads
    networks:
      - shared-network
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
        reservations:
          memory: 512M
          cpus: '0.5'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:7878/ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  prowlarr:
    image: linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    ports:
      - "9696:9696"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
    volumes:
      - ./config/prowlarr:/config
    networks:
      - shared-network
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'

  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "6881:6881"
      - "6881:6881/udp"
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=${TZ}
      - WEBUI_PORT=8080
    volumes:
      - ./config/qbittorrent:/config
      - /mnt/downloads:/downloads
    networks:
      - shared-network
    deploy:
      resources:
        limits:
          memory: 2G
          cpus: '2.0'
        reservations:
          memory: 1G
          cpus: '1.0'

networks:
  shared-network:
    external: true
EOF

# Apply optimized configuration
docker-compose down
cp docker-compose-optimized.yml docker-compose.yml
docker-compose up -d
```

### Configure Plex Performance Optimization
```bash
# SSH to Plex VM for performance tuning
ssh ubuntu@192.168.1.20

cd /home/ubuntu/plex

# Create optimized Plex configuration
cat > docker-compose-optimized.yml << 'EOF'
version: '3.8'

services:
  plex:
    image: plexinc/pms-docker:latest
    container_name: plex
    restart: unless-stopped
    hostname: plex-server
    ports:
      - "32400:32400/tcp"
      - "3005:3005/tcp"
      - "8324:8324/tcp"
      - "32469:32469/tcp"
      - "1900:1900/udp"
      - "32410:32410/udp"
      - "32412:32412/udp"
      - "32413:32413/udp"
      - "32414:32414/udp"
    environment:
      - PLEX_CLAIM=${PLEX_CLAIM}
      - PLEX_UID=${PLEX_UID}
      - PLEX_GID=${PLEX_GID}
      - TZ=${TZ}
      - ADVERTISE_IP=http://192.168.1.20:32400/
      - NVIDIA_VISIBLE_DEVICES=all
      - NVIDIA_DRIVER_CAPABILITIES=compute,video,utility
    volumes:
      - ./config:/config
      - ./transcode:/transcode
      - /mnt/media:/data/media:ro
      - /dev/shm:/ram-transcode
    devices:
      - /dev/dri:/dev/dri
    networks:
      - shared-network
    deploy:
      resources:
        limits:
          memory: 6G
          cpus: '3.0'
        reservations:
          memory: 2G
          cpus: '1.0'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:32400/web"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  tautulli:
    image: linuxserver/tautulli:latest
    container_name: tautulli
    restart: unless-stopped
    environment:
      - PUID=${PLEX_UID}
      - PGID=${PLEX_GID}
      - TZ=${TZ}
    volumes:
      - ./tautulli-config:/config
    ports:
      - "8181:8181"
    networks:
      - shared-network
    depends_on:
      - plex
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
        reservations:
          memory: 256M
          cpus: '0.25'

networks:
  shared-network:
    external: true
EOF

# Apply optimized configuration
docker-compose down
cp docker-compose-optimized.yml docker-compose.yml
docker-compose up -d
```

## Validation and Testing

### Service Accessibility Test
```bash
# Create comprehensive service test
cat > /tmp/service-test.sh << 'EOF'
#!/bin/bash
echo "=== Essential Services Accessibility Test ==="

# Test Plex
echo "Testing Plex Media Server..."
if curl -s http://192.168.1.20:32400/identity | grep -q PlexMediaServer; then
    echo "✅ Plex server accessible"
else
    echo "❌ Plex server not accessible"
fi

# Test Sonarr
echo "Testing Sonarr..."
if curl -s http://192.168.1.21:8989/ping | grep -q "OK"; then
    echo "✅ Sonarr accessible"
else
    echo "❌ Sonarr not accessible"
fi

# Test Radarr  
echo "Testing Radarr..."
if curl -s http://192.168.1.21:7878/ping | grep -q "OK"; then
    echo "✅ Radarr accessible"
else
    echo "❌ Radarr not accessible"
fi

# Test Prowlarr
echo "Testing Prowlarr..."
if curl -s http://192.168.1.21:9696/ping | grep -q "OK"; then
    echo "✅ Prowlarr accessible"
else
    echo "❌ Prowlarr not accessible"
fi

# Test qBittorrent
echo "Testing qBittorrent..."
if curl -s http://192.168.1.21:8080 | grep -q qBittorrent; then
    echo "✅ qBittorrent accessible"
else
    echo "❌ qBittorrent not accessible"
fi

# Test Tautulli
echo "Testing Tautulli..."
if curl -s http://192.168.1.20:8181 | grep -q Tautulli; then
    echo "✅ Tautulli accessible"
else
    echo "❌ Tautulli not accessible"
fi


echo "=== Service Test Complete ==="
EOF

chmod +x /tmp/service-test.sh
/tmp/service-test.sh
```

### Media Processing Workflow Test
```bash
# Create test workflow to verify end-to-end functionality
cat > /tmp/workflow-test.sh << 'EOF'
#!/bin/bash
echo "=== Media Processing Workflow Test ==="

# Test 1: Storage accessibility
echo "Test 1: Storage accessibility"
if mountpoint -q /mnt/media && mountpoint -q /mnt/downloads; then
    echo "✅ Storage mounts accessible"
else
    echo "❌ Storage mounts failed"
fi

# Test 2: Service communication
echo "Test 2: Service communication"
# Test if Sonarr can communicate with qBittorrent
SONARR_QB_TEST=$(curl -s "http://192.168.1.21:8989/api/v3/downloadclient" -H "X-Api-Key: $(grep SONARR_API_KEY /home/ubuntu/arr-stack/.env | cut -d'=' -f2)" 2>/dev/null)
if echo "$SONARR_QB_TEST" | grep -q qbittorrent; then
    echo "✅ Sonarr → qBittorrent communication working"
else
    echo "⚠️  Sonarr → qBittorrent communication not configured yet"
fi

# Test 3: Plex library access
echo "Test 3: Plex library access"
if [ -d "/mnt/media/movies" ] && [ -d "/mnt/media/tv" ]; then
    MOVIE_COUNT=$(find /mnt/media/movies -name "*.mkv" -o -name "*.mp4" | wc -l)
    TV_COUNT=$(find /mnt/media/tv -name "*.mkv" -o -name "*.mp4" | wc -l)
    echo "✅ Media library accessible - Movies: $MOVIE_COUNT, TV: $TV_COUNT"
else
    echo "❌ Media library structure missing"
fi

# Test 4: Download processing
echo "Test 4: Download processing"
if [ -d "/mnt/downloads/complete" ] && [ -d "/mnt/downloads/incomplete" ]; then
    echo "✅ Download directories accessible"
else
    echo "❌ Download directory structure missing"
fi

echo "=== Workflow Test Complete ==="
EOF

chmod +x /tmp/workflow-test.sh
/tmp/workflow-test.sh
```

### Hardware Transcoding Verification
```bash
# Test hardware transcoding capability on Plex VM
ssh ubuntu@192.168.1.20

# Check if Intel GPU is available for transcoding
lspci | grep VGA
ls -la /dev/dri/

# Check Plex transcoding capabilities
docker exec plex /usr/lib/plexmediaserver/Plex\ Transcoder -encoders 2>/dev/null | grep -i hardware

# If hardware transcoding available:
echo "Hardware transcoding devices available:"
ls -la /dev/dri/
```

## Troubleshooting Common Issues

### Service Startup Issues
```bash
# Debug service startup problems

# Check Docker logs
docker-compose logs [service-name]

# Check container resource usage
docker stats

# Check network connectivity between containers
docker exec sonarr ping redis
docker exec sonarr ping shared-postgres

# Restart problematic services
docker-compose restart [service-name]
```

### Storage Mount Issues
```bash
# Debug NFS mount problems

# Check NFS server availability
showmount -e 192.168.10.20

# Test manual mount
sudo umount /mnt/media
sudo mount -t nfs -v 192.168.10.20:/mnt/pool/media /mnt/media

# Check NFS statistics
nfsstat -c

# Check for network errors
dmesg | grep nfs
```

### Performance Issues
```bash
# Monitor service performance
htop
iotop -ao

# Check Docker resource usage
docker stats --no-stream

# Test storage performance
dd if=/dev/zero of=/mnt/downloads/test.img bs=1M count=1024 oflag=dsync
rm /mnt/downloads/test.img

# Check network utilization
iftop -i ens18
```

## Service Configuration Guidelines

### Initial Service Setup Checklist

#### Plex Configuration
- [ ] **Server claimed** with Plex account
- [ ] **Media libraries** added (Movies, TV Shows)
- [ ] **Remote access** configured
- [ ] **Hardware transcoding** enabled (if supported)
- [ ] **Scheduled tasks** configured (library updates, optimization)

#### Sonarr Configuration
- [ ] **Root folder** configured (/tv)
- [ ] **Quality profiles** set up
- [ ] **Download client** (qBittorrent) configured
- [ ] **Indexers** added via Prowlarr
- [ ] **Notification** to Plex configured

#### Radarr Configuration  
- [ ] **Root folder** configured (/movies)
- [ ] **Quality profiles** set up
- [ ] **Download client** (qBittorrent) configured
- [ ] **Indexers** added via Prowlarr
- [ ] **Notification** to Plex configured

#### Prowlarr Configuration
- [ ] **Indexers** added (public and private)
- [ ] **Applications** synced (Sonarr, Radarr)
- [ ] **Sync profiles** configured
- [ ] **Categories** mapped correctly

#### qBittorrent Configuration
- [ ] **Download directories** configured
- [ ] **Categories** set up (tv-sonarr, movies-radarr)
- [ ] **Speed limits** configured (if needed)
- [ ] **VPN** configured (if using)

## Completion Criteria
- [ ] **VM templates** created and ready for cloning
- [ ] **Plex server** deployed and accessible
- [ ] **Complete *arr stack** operational
- [ ] **Service integration** configured and tested
- [ ] **Media storage** mounted and accessible
- [ ] **Download processing** working
- [ ] **Service health monitoring** operational
- [ ] **Backup strategies** implemented
- [ ] **Performance optimization** applied
- [ ] **Security hardening** completed

### Performance Validation
- [ ] **Plex streaming** working smoothly (local and remote)
- [ ] **Transcoding** utilizing available hardware acceleration
- [ ] **Download speeds** meeting expectations
- [ ] **Service response times** <2 seconds for web interfaces
- [ ] **Storage I/O** not causing bottlenecks
- [ ] **Memory usage** within allocated limits

### Integration Validation
- [ ] **Prowlarr** successfully syncing indexers to Sonarr/Radarr
- [ ] **Download clients** properly configured in *arr apps
- [ ] **Plex notifications** working from Sonarr/Radarr
- [ ] **Media library updates** automatic after downloads
- [ ] **Quality profiles** and naming conventions working
- [ ] **Failed download handling** configured

**Next Phase**: Proceed to [Monitoring](07-monitoring.md) to deploy comprehensive monitoring and observability for your entire homelab infrastructure.