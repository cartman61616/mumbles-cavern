# Quick Hostname Updates
**Current State to Snorlax Names**

## Current Status ✅
- pve-node1: 192.168.10.10 (Management VLAN) 
- pve-node2: 192.168.10.11 (Management VLAN)
- VLANs: Already configured and operational
- Network: No changes needed!

## Simple Hostname Updates (30 minutes total)

### Node 1: pve-node1 → drowzee.mumblescavern.local

```bash
# SSH to node 1
ssh root@192.168.10.10

# Update hostname
hostnamectl set-hostname drowzee.mumblescavern.local
echo "drowzee.mumblescavern.local" > /etc/hostname

# Update hosts file
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

# No reboot needed - just restart services
systemctl restart pveproxy pvedaemon
```

### Node 2: pve-node2 → sleepy.mumblescavern.local

```bash
# SSH to node 2
ssh root@192.168.10.11

# Update hostname
hostnamectl set-hostname sleepy.mumblescavern.local
echo "sleepy.mumblescavern.local" > /etc/hostname

# Update hosts file
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

# Restart services
systemctl restart pveproxy pvedaemon
```

### Update UniFi DNS (5 minutes)
```yaml
UniFi Controller Updates:
- 192.168.10.10 → drowzee.mumblescavern.local
- 192.168.10.11 → sleepy.mumblescavern.local
```

### Verify Cluster (5 minutes)
```bash
# From either node
pvecm status
# Should show both nodes healthy

# Test new names
ping drowzee.mumblescavern.local
ping sleepy.mumblescavern.local
```

## Next: Deploy Plex
Once hostnames are updated, proceed with [Fast-Track Plex Deployment](fast-track-plex-deployment.md)

This is much simpler than full network migration! 🎉