# Fast-Track Plex Deployment 
**Before TrueNAS Reset - Minimal Viable Product**

**Duration**: 1-2 hours  
**Goal**: Get Plex Media Server operational on Dell cluster before TrueNAS reset  
**Current Status**: 2-node Proxmox cluster (drowzee + sleepy) operational

## Prerequisites Check

Verify we have:
- [x] 2-node Proxmox cluster operational (drowzee.mumblescavern.local + sleepy.mumblescavern.local)
- [x] ASUSTOR NAS accessible with media library
- [x] Network connectivity between cluster and NAS
- [ ] Basic VM template (we'll create minimal if needed)

## Quick VM Creation Method

### Option A: Quick Ubuntu VM (No Template)
If no template exists, create directly:

```bash
# SSH to drowzee (primary node)
ssh root@192.168.10.10  # or whatever the current management IP is

# Create minimal Plex VM quickly
qm create 101 \
    --name plex-drowzee \
    --memory 4096 \
    --cores 4 \
    --net0 virtio,bridge=vmbr0,tag=20 \
    --storage local-lvm

# Download and attach Ubuntu ISO for quick install
cd /var/lib/vz/template/iso
wget -O ubuntu-22.04-live-server.iso "https://releases.ubuntu.com/22.04/ubuntu-22.04.3-live-server-amd64.iso"

# Attach ISO and create disk
qm set 101 --ide2 local:iso/ubuntu-22.04-live-server.iso,media=cdrom
qm set 101 --scsi0 local-lvm:32

# Start VM for installation
qm start 101
```

### Option B: Cloud Image Quick Deploy (Faster)
```bash
# Download cloud image for instant deployment
cd /tmp
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Create VM with cloud image
qm create 101 \
    --name plex-drowzee \
    --memory 4096 \
    --cores 4 \
    --net0 virtio,bridge=vmbr0,tag=20 \
    --storage local-lvm

# Import cloud image
qm importdisk 101 /tmp/jammy-server-cloudimg-amd64.img local-lvm
qm set 101 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-101-disk-0
qm set 101 --ide2 local-lvm:cloudinit
qm set 101 --boot c --bootdisk scsi0
qm set 101 --serial0 socket --vga serial0

# Configure network (Services VLAN)
qm set 101 --ipconfig0 ip=192.168.20.10/24,gw=192.168.20.1
qm set 101 --nameserver 192.168.20.1
qm set 101 --searchdomain mumblescavern.local

# Configure user
qm set 101 --ciuser ubuntu
qm set 101 --cipassword "ChangeMe123!"  # Change this immediately

# Resize disk and start
qm resize 101 scsi0 +20G
qm start 101
```

## Rapid Plex Configuration

Once VM is running:

### Step 1: Basic System Setup
```bash
# SSH to Plex VM (adjust IP as needed)
ssh ubuntu@192.168.20.10

# Quick system update
sudo apt update && sudo apt install -y curl docker.io nfs-common

# Add user to docker group
sudo usermod -aG docker ubuntu
newgrp docker

# Create media mount point
sudo mkdir -p /mnt/media
```

### Step 2: Mount NAS Storage
```bash
# Test NFS mount from ASUSTOR (adjust IP/path as needed)
sudo mount -t nfs 192.168.10.20:/mnt/pool/media /mnt/media

# Verify media is accessible
ls -la /mnt/media

# Make mount persistent (add to fstab)
echo "192.168.10.20:/mnt/pool/media /mnt/media nfs defaults 0 0" | sudo tee -a /etc/fstab
```

### Step 3: Deploy Plex with Docker
```bash
# Create Plex config directory
mkdir -p ~/plex/config
cd ~/plex

# Quick Plex deployment with Docker
docker run -d \
    --name plex \
    --restart unless-stopped \
    -p 32400:32400/tcp \
    -p 3005:3005/tcp \
    -p 8324:8324/tcp \
    -p 32469:32469/tcp \
    -p 1900:1900/udp \
    -p 32410:32410/udp \
    -p 32412:32412/udp \
    -p 32413:32413/udp \
    -p 32414:32414/udp \
    -e TZ="America/New_York" \
    -e PLEX_CLAIM="claim-XXXXXXXXXX" \
    -e PLEX_UID=1000 \
    -e PLEX_GID=1000 \
    -v ~/plex/config:/config \
    -v /mnt/media:/data:ro \
    -v /tmp/plex-transcode:/transcode \
    --tmpfs /tmp/plex-transcode \
    plexinc/pms-docker:latest
```

### Step 4: Get Claim Token
```bash
# Visit https://www.plex.tv/claim/ to get your claim token
# Replace "claim-XXXXXXXXXX" in the docker run command above
```

### Step 5: Access and Configure Plex
1. Open web browser to `http://192.168.20.10:32400/web`
2. Complete Plex setup wizard
3. Add media libraries pointing to `/data`
4. Test playback

## Network Access Configuration

Ensure Plex is accessible from other VLANs:

### UniFi Firewall Rules
Add rules to allow:
- Gaming VLAN (192.168.80.x) → Services VLAN (192.168.20.x) port 32400
- Default VLAN (192.168.1.x) → Services VLAN (192.168.20.x) port 32400

### Test Connectivity
```bash
# From gaming PC or other device, test:
curl -I http://192.168.20.10:32400/web
```

## Post-Deployment Validation

### Quick Health Check
```bash
# Check Plex container status
docker ps | grep plex

# Check container logs
docker logs plex

# Verify NFS mount
df -h | grep media

# Test from external device
# Open browser to http://192.168.20.10:32400/web
```

### Backup Critical Data
Before TrueNAS reset, ensure:
```bash
# Backup Plex config
sudo tar -czf ~/plex-config-backup.tar.gz ~/plex/config

# Copy backup to safe location (maybe to Synology backup NAS)
# scp ~/plex-config-backup.tar.gz user@synology-ip:/backup/
```

## Minimal *arr Stack (Optional)

If time permits, deploy basic media automation:

```bash
# Quick Sonarr deployment
docker run -d \
    --name sonarr \
    --restart unless-stopped \
    -p 8989:8989 \
    -e TZ="America/New_York" \
    -v ~/sonarr/config:/config \
    -v /mnt/media:/data \
    linuxserver/sonarr:latest

# Quick Radarr deployment  
docker run -d \
    --name radarr \
    --restart unless-stopped \
    -p 7878:7878 \
    -e TZ="America/New_York" \
    -v ~/radarr/config:/config \
    -v /mnt/media:/data \
    linuxserver/radarr:latest
```

Access:
- Sonarr: `http://192.168.20.10:8989`
- Radarr: `http://192.168.20.10:7878`

## Post-TrueNAS Reset Migration Plan

After TrueNAS reset:
1. Update NFS mount points in `/etc/fstab`
2. Restart containers: `docker restart plex sonarr radarr`
3. Update media library paths in Plex if needed
4. Verify all services operational

This fast-track deployment gets Plex running quickly while preserving the ability to properly architect the full solution later!