# Check ACME certificate status
check_acme_status() {
    if [ -f "data/acme.json" ]; then
        CERT_COUNT=$(jq -r '.cloudflare.Certificates | length' data/acme.json 2>/dev/null || echo "0")
        log_message "ACME: $CERT_COUNT certificates managed"
        
        # Check if any certificates are near expiry
        jq -r '.cloudflare.Certificates[] | select(.certificate != null) | .domain.main' data/acme.json 2>/dev/null | while read domain; do
            check_cert_expiration "$domain"
        done
    else
        log_message "ACME file not found"
    fi
}

# Force certificate renewal if needed
force_cert_renewal() {
    log_message "Forcing certificate renewal"
    docker exec traefik traefik version
    # Certificate renewal is automatic, but we can restart Traefik to trigger checks
    docker-compose restart traefik
}

# Monitor certificate health
check_acme_status

# Force renewal if any certs expire within 7 days (emergency renewal)
if grep -q "expires in [0-7] days" "$LOG_FILE"; then
    force_cert_renewal
fi

# Keep log manageable
tail -200 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x scripts/cert-management.sh

# Schedule certificate monitoring daily
echo "0 6 * * * /home/ubuntu/traefik/scripts/cert-management.sh" | crontab -l | { cat; echo "0 6 * * * /home/ubuntu/traefik/scripts/cert-management.sh"; } | crontab -

# Test certificate management
./scripts/cert-management.sh
```

### Configure Certificate Backup
```bash
# Create certificate backup script
cat > scripts/cert-backup.sh << 'EOF'
#!/bin/bash
# Backup SSL certificates and Traefik configuration

BACKUP_DIR="/home/ubuntu/traefik/backups"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/cert-backup.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$LOG_FILE"
}

mkdir -p "$BACKUP_DIR"

# Backup ACME certificates
backup_certificates() {
    log_message "Starting certificate backup"
    
    if [ -f "data/acme.json" ]; then
        # Backup ACME file
        cp data/acme.json "$BACKUP_DIR/acme_$DATE.json"
        chmod 600 "$BACKUP_DIR/acme_$DATE.json"
        
        # Create readable certificate info
        jq -r '.cloudflare.Certificates[] | "Domain: " + .domain.main + " | Expires: " + .certificate' data/acme.json > "$BACKUP_DIR/cert_info_$DATE.txt" 2>/dev/null
        
        log_message "Certificate backup completed"
    else
        log_message "No ACME file found"
    fi
}

# Backup Traefik configuration
backup_traefik_config() {
    log_message "Starting Traefik configuration backup"
    
    tar -czf "$BACKUP_DIR/traefik_config_$DATE.tar.gz" \
        docker-compose.yml \
        .env \
        config/ \
        scripts/
    
    log_message "Traefik configuration backup completed"
}

# Cleanup old backups
cleanup_old_backups() {
    log_message "Starting backup cleanup"
    
    # Remove backups older than 60 days
    find "$BACKUP_DIR" -name "*.json" -mtime +60 -delete
    find "$BACKUP_DIR" -name "*.tar.gz" -mtime +60 -delete
    find "$BACKUP_DIR" -name "*.txt" -mtime +60 -delete
    
    log_message "Backup cleanup completed"
}

# Execute backup functions
backup_certificates
backup_traefik_config
cleanup_old_backups

# Copy critical backups to shared storage
cp "$BACKUP_DIR"/acme_$DATE.json /mnt/pve/asustor-backup/ 2>/dev/null || true
cp "$BACKUP_DIR"/traefik_config_$DATE.tar.gz /mnt/pve/asustor-backup/ 2>/dev/null || true

log_message "Certificate backup cycle completed"
EOF

chmod +x scripts/cert-backup.sh

# Schedule certificate backups weekly
echo "0 5 * * 0 /home/ubuntu/traefik/scripts/cert-backup.sh" | crontab -l | { cat; echo "0 5 * * 0 /home/ubuntu/traefik/scripts/cert-backup.sh"; } | crontab -
```

## Step 9: Configure Load Balancing and High Availability

### Configure Service Load Balancing
```bash
# Create load balancing configuration for redundant services
cat > config/load-balancing.yml << 'EOF'
http:
  services:
    # Load balance Plex if you deploy multiple instances
    plex-cluster:
      loadBalancer:
        servers:
          - url: "http://192.168.1.20:32400"
          # - url: "http://192.168.1.24:32400"  # Future second Plex instance
        healthCheck:
          path: "/identity"
          interval: "30s"
          timeout: "10s"

    # Load balance monitoring services
    grafana-cluster:
      loadBalancer:
        servers:
          - url: "http://192.168.1.22:3000"
        healthCheck:
          path: "/api/health"
          interval: "30s"
          timeout: "10s"

    # Failover configuration for critical services
    shared-infrastructure:
      loadBalancer:
        servers:
          - url: "http://192.168.1.15:5432"
        healthCheck:
          path: "/"
          interval: "30s"
          timeout: "10s"

  routers:
    # Example of load-balanced router
    plex-lb:
      rule: "Host(`plex-lb.snorlax.me`)"
      entrypoints:
        - websecure
      middlewares:
        - public-access@file
      tls:
        certResolver: cloudflare
      service: plex-cluster
EOF
```

### Configure Health Checks and Failover
```bash
# Create service health monitoring for load balancing
cat > scripts/service-health-lb.sh << 'EOF'
#!/bin/bash
# Monitor service health for load balancing decisions

LOG_FILE="/var/log/service-health-lb.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Check service health and update Traefik if needed
check_service_health() {
    local SERVICE_NAME="$1"
    local SERVICE_URL="$2"
    local HEALTH_ENDPOINT="$3"
    
    if curl -s -f "$SERVICE_URL$HEALTH_ENDPOINT" > /dev/null; then
        log_message "$SERVICE_NAME: Healthy"
        return 0
    else
        log_message "$SERVICE_NAME: Unhealthy - removing from load balancer"
        return 1
    fi
}

# Monitor key services
SERVICES=(
    "plex:http://192.168.1.20:32400:/identity"
    "grafana:http://192.168.1.22:3000:/api/health"
    "prometheus:http://192.168.1.22:9090:/-/healthy"
)

for service_info in "${SERVICES[@]}"; do
    SERVICE_NAME=$(echo "$service_info" | cut -d':' -f1)
    SERVICE_URL=$(echo "$service_info" | cut -d':' -f2-3)
    HEALTH_ENDPOINT=$(echo "$service_info" | cut -d':' -f4)
    
    check_service_health "$SERVICE_NAME" "$SERVICE_URL" "$HEALTH_ENDPOINT"
done

# Check Traefik API health
if curl -s http://localhost:8080/ping | grep -q "OK"; then
    log_message "Traefik API: Healthy"
else
    log_message "Traefik API: Unhealthy"
fi

log_message "Service health check completed"
EOF

chmod +x scripts/service-health-lb.sh

# Schedule health checks every 2 minutes
echo "*/2 * * * * /home/ubuntu/traefik/scripts/service-health-lb.sh" | crontab -l | { cat; echo "*/2 * * * * /home/ubuntu/traefik/scripts/service-health-lb.sh"; } | crontab -
```

## Step 10: Configure Security Monitoring and Intrusion Detection

### Set Up Access Logging and Analysis
```bash
# Create access log analysis script
cat > scripts/analyze-access-logs.sh << 'EOF'
#!/bin/bash
# Analyze Traefik access logs for security threats

LOG_FILE="/var/log/security-analysis.log"
ACCESS_LOG="/home/ubuntu/traefik/logs/access.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Analyze access patterns
analyze_access_patterns() {
    if [ ! -f "$ACCESS_LOG" ]; then
        log_message "Access log not found"
        return
    fi
    
    # Count requests in last hour
    HOUR_AGO=$(date -d '1 hour ago' '+%Y-%m-%dT%H')
    REQUESTS_LAST_HOUR=$(grep "$HOUR_AGO" "$ACCESS_LOG" | wc -l)
    
    log_message "Requests in last hour: $REQUESTS_LAST_HOUR"
    
    # Check for suspicious activity
    FAILED_REQUESTS=$(grep -E '"(4[0-9]{2}|5[0-9]{2})"' "$ACCESS_LOG" | grep "$HOUR_AGO" | wc -l)
    if [ "$FAILED_REQUESTS" -gt 100 ]; then
        log_message "WARNING: High number of failed requests: $FAILED_REQUESTS"
    fi
    
    # Check for potential attacks
    ATTACK_PATTERNS=(
        "\.\./"
        "/etc/passwd"
        "/proc/"
        "admin.*admin"
        "SELECT.*FROM"
        "<script"
    )
    
    for pattern in "${ATTACK_PATTERNS[@]}"; do
        ATTACK_COUNT=$(grep -i "$pattern" "$ACCESS_LOG" | grep "$HOUR_AGO" | wc -l)
        if [ "$ATTACK_COUNT" -gt 0 ]; then
            log_message "SECURITY: Detected $ATTACK_COUNT requests matching pattern: $pattern"
        fi
    done
    
    # Top IPs by request count
    TOP_IPS=$(grep "$HOUR_AGO" "$ACCESS_LOG" | jq -r '.ClientHost' | sort | uniq -c | sort -nr | head -5)
    log_message "Top IPs in last hour:"
    echo "$TOP_IPS" | while read count ip; do
        log_message "  $ip: $count requests"
    done
}

# Check for blocked countries (if geo-blocking enabled)
check_geo_blocks() {
    GEO_BLOCKS=$(grep "geoblock" "$ACCESS_LOG" | grep "$HOUR_AGO" | wc -l)
    if [ "$GEO_BLOCKS" -gt 0 ]; then
        log_message "Geo-blocking: $GEO_BLOCKS requests blocked"
    fi
}

# Run analysis
analyze_access_patterns
check_geo_blocks

log_message "Security analysis completed"
EOF

chmod +x scripts/analyze-access-logs.sh

# Schedule security analysis every hour
echo "0 * * * * /home/ubuntu/traefik/scripts/analyze-access-logs.sh" | crontab -l | { cat; echo "0 * * * * /home/ubuntu/traefik/scripts/analyze-access-logs.sh"; } | crontab -
```

### Configure Fail2Ban for Additional Protection
```bash
# Install and configure Fail2Ban
sudo apt install -y fail2ban

# Create Fail2Ban configuration for Traefik
sudo cat > /etc/fail2ban/filter.d/traefik.conf << 'EOF'
[Definition]
failregex = ^.*"ClientHost":"<HOST>".*"RequestMethod":".*".*"RequestURI":".*".*"OriginStatus":(4\d\d|5\d\d).*$
ignoreregex =
EOF

# Configure Fail2Ban jail for Traefik
sudo cat > /etc/fail2ban/jail.d/traefik.conf << 'EOF'
[traefik]
enabled = true
port = 80,443
filter = traefik
logpath = /home/ubuntu/traefik/logs/access.log
maxretry = 5
bantime = 3600
findtime = 600
banaction = iptables-multiport
EOF

# Restart Fail2Ban
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban

# Check Fail2Ban status
sudo fail2ban-client status
sudo fail2ban-client status traefik
```

## Step 11: Configure External Service Monitoring

### Add External Access Monitoring to Prometheus
```bash
# SSH to monitoring VM to add external monitoring
ssh ubuntu@192.168.1.22

cd /home/ubuntu/monitoring

# Create external service monitoring configuration
cat > prometheus/external-services.yml << 'EOF'
# External service monitoring via Traefik

scrape_configs:
  - job_name: 'traefik'
    static_configs:
      - targets:
        - '192.168.1.23:8080'
    scrape_interval: 15s
    metrics_path: /metrics

  - job_name: 'external-services-health'
    static_configs:
      - targets:
        - '192.168.1.23:9091'  # Custom external health exporter
    scrape_interval: 30s
    metrics_path: /metrics
EOF

# Create external service health exporter
cat > scripts/external-health-exporter.py << 'EOF'
#!/usr/bin/env python3
# External service health monitoring exporter

import time
import requests
import subprocess
from prometheus_client import start_http_server, Gauge, Counter

# Define metrics
external_service_response_time = Gauge('external_service_response_time_seconds', 'External service response time', ['service', 'domain'])
external_service_status = Gauge('external_service_status', 'External service status', ['service', 'domain'])
external_certificate_expiry = Gauge('external_certificate_expiry_days', 'Days until certificate expiry', ['domain'])

def check_external_service(domain, service_name, path="/"):
    """Check external service health and response time"""
    
    url = f"https://{domain}{path}"
    
    try:
        start_time = time.time()
        response = requests.get(url, timeout=10, verify=True)
        response_time = time.time() - start_time
        
        # Record metrics
        external_service_response_time.labels(service=service_name, domain=domain).set(response_time)
        external_service_status.labels(service=service_name, domain=domain).set(1 if response.status_code < 400 else 0)
        
        return True
        
    except Exception as e:
        external_service_response_time.labels(service=service_name, domain=domain).set(0)
        external_service_status.labels(service=service_name, domain=domain).set(0)
        return False

def check_certificate_expiry(domain):
    """Check SSL certificate expiration"""
    
    try:
        result = subprocess.run([
            'openssl', 's_client', '-servername', domain, '-connect', f'{domain}:443'
        ], input='', capture_output=True, text=True, timeout=10)
        
        if result.returncode == 0:
            cert_result = subprocess.run([
                'openssl', 'x509', '-noout', '-enddate'
            ], input=result.stdout, capture_output=True, text=True)
            
            if cert_result.returncode == 0:
                expiry_str = cert_result.stdout.strip().split('=')[1]
                expiry_time = subprocess.run([
                    'date', '-d', expiry_str, '+%s'
                ], capture_output=True, text=True)
                
                if expiry_time.returncode == 0:
                    expiry_epoch = int(expiry_time.stdout.strip())
                    current_epoch = int(time.time())
                    days_until_expiry = (expiry_epoch - current_epoch) / 86400
                    
                    external_certificate_expiry.labels(domain=domain).set(days_until_expiry)
                    return True
    except:
        pass
    
    external_certificate_expiry.labels(domain=domain).set(-1)
    return False

def main():
    """Main monitoring loop"""
    
    # Start metrics server
    start_http_server(9091)
    print("External health exporter started on port 9091")
    
    # Define services to monitor
    services = [
        ("plex.snorlax.me", "plex", "/identity"),
        ("grafana.snorlax.me", "grafana", "/api/health"),
        ("sonarr.snorlax.me", "sonarr", "/ping"),
        ("radarr.snorlax.me", "radarr", "/ping"),
        ("traefik.snorlax.me", "traefik", "/ping"),
    ]
    
    while True:
        try:
            for domain, service, path in services:
                check_external_service(domain, service, path)
                check_certificate_expiry(domain)
                
        except Exception as e:
            print(f"Error in monitoring loop: {e}")
        
        time.sleep(60)  # Check every minute

if __name__ == '__main__':
    main()
EOF

# Install Python dependencies and create systemd service
sudo apt install -y python3-pip
pip3 install prometheus_client requests

# Create systemd service
sudo cat > /etc/systemd/system/external-health-exporter.service << 'EOF'
[Unit]
Description=External Service Health Exporter
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/monitoring
ExecStart=/usr/bin/python3 /home/ubuntu/monitoring/scripts/external-health-exporter.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# Enable and start external health exporter
sudo systemctl daemon-reload
sudo systemctl enable external-health-exporter
sudo systemctl start external-health-exporter

# Test external health metrics
sleep 30
curl -s localhost:9091/metrics | grep external_
```

## Step 12: Create Emergency Access Procedures

### Configure Emergency Access Methods
```bash
# Create emergency access documentation and procedures
cat > docs/emergency-access.md << 'EOF'
# Emergency Access Procedures

## When External Access Fails

### Scenario 1: Cloudflare Tunnel Down
1. **Check tunnel status**: Cloudflare dashboard → Tunnels
2. **Restart cloudflared**: `docker-compose restart cloudflared`
3. **Check token validity**: Verify CLOUDFLARE_TUNNEL_TOKEN in .env
4. **Alternative access**: Use direct IP with port forwarding (emergency only)

### Scenario 2: Traefik Down
1. **Check Traefik status**: `docker-compose ps traefik`
2. **Check logs**: `docker-compose logs traefik`
3. **Restart Traefik**: `docker-compose restart traefik`
4. **Direct service access**: Use service IPs directly (internal network only)

### Scenario 3: Certificate Issues
1. **Check certificate status**: `./scripts/cert-management.sh`
2. **Force renewal**: `docker exec traefik traefik --acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory`
3. **Emergency certificate**: Use Let's Encrypt staging for testing
4. **Manual intervention**: SSH to Traefik VM and investigate acme.json

### Emergency Direct Access (Internal Network Only)
- Plex: http://192.168.1.20:32400
- Sonarr: http://192.168.1.21:8989
- Radarr: http://192.168.1.21:7878
- Grafana: http://192.168.1.22:3000
- Proxmox Node 1: https://192.168.1.10:8006
- Proxmox Node 2: https://192.168.1.11:8006

### VPN Access Setup (Future)
Consider setting up WireGuard VPN for secure remote access:
- Deploy WireGuard on Proxmox VM
- Configure client certificates
- Route traffic through VPN when external access fails
EOF

# Create emergency access script
cat > scripts/emergency-access.sh << 'EOF'
#!/bin/bash
# Emergency access configuration script

echo "=== Emergency Access Configuration ==="

# Show current service status
echo "Service Status:"
docker-compose ps

# Show certificate status
echo ""
echo "Certificate Status:"
if [ -f "data/acme.json" ]; then
    jq -r '.cloudflare.Certificates[] | "Domain: " + .domain.main' data/acme.json
else
    echo "No certificates found"
fi

# Show tunnel status
echo ""
echo "Cloudflare Tunnel Status:"
docker logs cloudflared --tail=10

# Show current routes
echo ""
echo "Current Traefik Routes:"
curl -s http://localhost:8080/api/http/routers | jq -r '.[] | .name + " -> " + .rule'

echo ""
echo "Direct Access URLs (Internal Network):"
echo "  Traefik Dashboard: http://192.168.1.23:8080"
echo "  Plex: http://192.168.1.20:32400"
echo "  Grafana: http://192.168.1.22:3000"
echo "  Proxmox: https://192.168.1.10:8006"
EOF

chmod +x scripts/emergency-access.sh
```

## Validation and Testing

### External Access Validation
```bash
# Create comprehensive external access test
cat > scripts/validate-external-access.sh << 'EOF'
#!/bin/bash
echo "=== External Access Validation ==="

# Define services to test
SERVICES=(
    "whoami.snorlax.me"
    "plex.snorlax.me"
    "grafana.snorlax.me"
    "sonarr.snorlax.me"
    "radarr.snorlax.me"
    "traefik.snorlax.me"
)

# Test external accessibility
echo "Testing external service accessibility..."
for service in "${SERVICES[@]}"; do
    echo -n "Testing $service: "
    
    # Test HTTP redirect to HTTPS
    HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "http://$service" --max-time 10)
    if [ "$HTTP_STATUS" = "301" ] || [ "$HTTP_STATUS" = "302" ]; then
        echo -n "✅ HTTP redirect "
    else
        echo -n "❌ HTTP redirect ($HTTP_STATUS) "
    fi
    
    # Test HTTPS access
    HTTPS_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://$service" --max-time 10)
    if [ "$HTTPS_STATUS" = "200" ]; then
        echo "✅ HTTPS working"
    else
        echo "❌ HTTPS failed ($HTTPS_STATUS)"
    fi
done

# Test SSL certificate validity
echo ""
echo "Testing SSL certificates..."
for service in "${SERVICES[@]}"; do
    echo -n "Certificate for $service: "
    
    CERT_INFO=$(echo | openssl s_client -servername "$service" -connect "$service:443" 2>/dev/null | openssl x509 -noout -dates 2>/dev/null)
    if [ $? -eq 0 ]; then
        EXPIRY=$(echo "$CERT_INFO" | grep notAfter | cut -d= -f2)
        EXPIRY_EPOCH=$(date -d "$EXPIRY" +%s)
        CURRENT_EPOCH=$(date +%s)
        DAYS_LEFT=$(( (EXPIRY_EPOCH - CURRENT_EPOCH) / 86400 ))
        
        if [ "$DAYS_LEFT" -gt 30 ]; then
            echo "✅ Valid ($DAYS_LEFT days left)"
        elif [ "$DAYS_LEFT" -gt 7 ]; then
            echo "⚠️  Expires soon ($DAYS_LEFT days left)"
        else
            echo "❌ Expires very soon ($DAYS_LEFT days left)"
        fi
    else
        echo "❌ Certificate check failed"
    fi
done

# Test Cloudflare Tunnel status
echo ""
echo "Testing Cloudflare Tunnel..."
TUNNEL_STATUS=$(docker logs cloudflared --tail=5 2>&1 | grep -i "registered tunnel connection")
if [ ! -z "$TUNNEL_STATUS" ]; then
    echo "✅ Cloudflare Tunnel connected"
else
    echo "❌ Cloudflare Tunnel connection issues"
fi

# Test Traefik health
echo ""
echo "Testing Traefik health..."
if curl -s http://localhost:8080/ping | grep -q "OK"; then
    echo "✅ Traefik API responding"
    
    ROUTES=$(curl -s http://localhost:8080/api/http/routers | jq -r 'length')
    echo "✅ Traefik managing $ROUTES routes"
else
    echo "❌ Traefik API not responding"
fi

echo ""
echo "=== External Access Validation Complete ==="
EOF

chmod +x scripts/validate-external-access.sh

# Run validation (may need to wait for DNS propagation)
sleep 60
./scripts/validate-external-access.sh
```

### Security Testing
```bash
# Create security testing script
cat > scripts/security-test.sh << 'EOF'
#!/bin/bash
# Security testing for external access

echo "=== Security Testing ==="

# Test rate limiting
echo "Testing rate limiting..."
for i in {1..60}; do
    curl -s -o /dev/null "https://whoami.snorlax.me" &
done
wait

echo "Rate limiting test completed (check access logs for blocks)"

# Test security headers
echo ""
echo "Testing security headers..."
HEADERS=$(curl -s -I "https://whoami.snorlax.me")
echo "Security Headers Check:"

if echo "$HEADERS" | grep -qi "X-Frame-Options"; then
    echo "✅ X-Frame-Options present"
else
    echo "❌ X-Frame-Options missing"
fi

if echo "$HEADERS" | grep -qi "Strict-Transport-Security"; then
    echo "✅ HSTS present"
else
    echo "❌ HSTS missing"
fi

if echo "$HEADERS" | grep -qi "X-Content-Type-Options"; then
    echo "✅ X-Content-Type-Options present"
else
    echo "❌ X-Content-Type-Options missing"
fi

# Test admin access restrictions
echo ""
echo "Testing admin access restrictions..."
ADMIN_STATUS=$(curl -s -o /dev/null -w "%{http_code}" "https://traefik.snorlax.me" --max-time 10)
if [ "$ADMIN_STATUS" = "401" ] || [ "$ADMIN_STATUS" = "403" ]; then
    echo "✅ Admin interface properly restricted"
else
    echo "⚠️  Admin interface response: $ADMIN_STATUS"
fi

echo ""
echo "=== Security Testing Complete ==="
EOF

chmod +x scripts/security-test.sh
# ./scripts/security-test.sh  # Run after services are accessible externally
```

## Troubleshooting Common Issues

### Traefik Configuration Issues
```bash
# Debug Traefik configuration problems

# Check Traefik configuration syntax
docker exec traefik traefik version

# Check routes and services
curl -s http://localhost:8080/api/http/routers | jq '.[] | {name: .name, rule: .rule, status: .status}'
curl -s http://localhost:8080/api/http/services | jq '.[] | {name: .name, status: .status}'

# Check certificate status
curl -s http://localhost:8080/api/http/routers | jq '.[] | {name: .name, tls: .tls}'

# View real-time logs
docker-compose logs -f traefik
```

### Cloudflare Tunnel Issues
```bash
# Debug Cloudflare Tunnel problems

# Check tunnel registration
docker logs cloudflared | grep -i "registered tunnel"

# Check tunnel configuration
docker exec cloudflared cloudflared tunnel info

# Test tunnel connectivity
docker exec cloudflared cloudflared tunnel run --dry-run

# Restart tunnel with verbose logging
docker-compose stop cloudflared
docker run --rm cloudflare/cloudflared:latest tunnel --no-autoupdate run --token $CLOUDFLARE_TUNNEL_TOKEN
```

### SSL Certificate Issues
```bash
# Debug SSL certificate problems

# Check ACME challenge logs
docker logs traefik | grep -i acme

# Test DNS challenge manually
dig TXT _acme-challenge.snorlax.me

# Check Cloudflare API access
curl -X GET "https://api.cloudflare.com/client/v4/zones" \
     -H "Authorization: Bearer $CF_DNS_API_TOKEN" \
     -H "Content-Type: application/json"

# Force certificate regeneration (testing only)
# rm data/acme.json && docker-compose restart traefik
```

## Performance Optimization

### Configure Traefik Performance Tuning
```bash
# Create performance tuning configuration
cat > config/performance.yml << 'EOF'
# Performance optimization for Traefik

entryPoints:
  web:
    transport:
      lifeCycle:
        graceTimeOut: "10s"
      respondingTimeouts:
        readTimeout: "60s"
        writeTimeout: "60s"
        idleTimeout: "180s"

  websecure:
    transport:
      lifeCycle:
        graceTimeOut: "10s"
      respondingTimeouts:
        readTimeout: "60s"
        writeTimeout: "60s"
        idleTimeout: "180s"

# Connection limits and timeouts
serversTransport:
  default:
    dialTimeout: "30s"
    responseHeaderTimeout: "60s"
    idleConnTimeout: "90s"
    maxIdleConnsPerHost: 100

# Enable compression for better performance
http:
  middlewares:
    compression:
      compress:
        excludedContentTypes:
          - "text/event-stream"
EOF
```

## Completion Criteria
- [ ] **Traefik deployed** and accessible via web UI
- [ ] **Cloudflare Tunnel** operational and connected
- [ ] **SSL certificates** automatically generated and valid
- [ ] **All services** accessible externally via HTTPS
- [ ] **Security headers** configured and validated
- [ ] **Rate limiting** implemented and tested
- [ ] **Access logging** configured and monitored
- [ ] **Fail2Ban** configured for additional protection
- [ ] **External monitoring** operational
- [ ] **Emergency access** procedures documented

### Security Validation
- [ ] **HTTP to HTTPS redirect** working for all services
- [ ] **Security headers** present in all responses
- [ ] **Admin interfaces** restricted to internal networks
- [ ] **Rate limiting** active and blocking excessive requests
- [ ] **Geo-blocking** configured (if desired)
- [ ] **SSL certificates** valid and auto-renewing
- [ ] **Access logs** being analyzed for threats
- [ ] **Fail2Ban** actively monitoring and blocking IPs

### Performance Validation
- [ ] **External response times** <2 seconds for all services
- [ ] **SSL handshake** completing quickly
- [ ] **Compression** working for appropriate content types
- [ ] **Connection pooling** optimized
- [ ] **Certificate validation** not causing delays
- [ ] **Tunnel latency** minimal (test from external network)

### Monitoring Integration
- [ ] **Traefik metrics** being collected by Prometheus
- [ ] **External service health** monitored and alerting
- [ ] **Certificate expiry** alerts configured
- [ ] **Access patterns** being analyzed
- [ ] **Security events** logged and monitored

## Final Configuration Tasks

### Update Service Configurations
Replace `snorlax.me` with your actual domain in all configuration files:

```bash
# Update all domain references
find /home/ubuntu/traefik -name "*.yml" -o -name "*.yaml" | xargs sed -i 's/yourdomain\.com/your-actual-domain.com/g'
find /home/ubuntu/traefik -name "*.env" | xargs sed -i 's/yourdomain\.com/your-actual-domain.com/g'

# Update Cloudflare credentials with real values
nano .env
# Fill in actual CF_API_EMAIL, CF_DNS_API_TOKEN, and CLOUDFLARE_TUNNEL_TOKEN
```

### Test Complete External Workflow
```bash
# Create end-to-end external access test
cat > scripts/e2e-external-test.sh << 'EOF'
#!/bin/bash
echo "=== End-to-End External Access Test ==="

# Test complete user workflow from external access
echo "Testing complete external user workflow..."

# 1. Test Plex access and media streaming capability
echo "1. Testing Plex external access..."
PLEX_RESPONSE=$(curl -s "https://plex.snorlax.me/identity")
if echo "$PLEX_RESPONSE" | grep -q "PlexMediaServer"; then
    echo "✅ Plex externally accessible"
else
    echo "❌ Plex external access failed"
fi

# 2. Test Grafana dashboard access
echo "2. Testing Grafana external access..."
GRAFANA_RESPONSE=$(curl -s "https://grafana.snorlax.me/api/health")
if echo "$GRAFANA_RESPONSE" | grep -q "ok"; then
    echo "✅ Grafana externally accessible"
else
    echo "❌ Grafana external access failed"
fi

# 3. Test *arr service management
echo "3. Testing *arr services external access..."
for service in sonarr radarr prowlarr; do
    RESPONSE=$(curl -s "https://$service.snorlax.me/ping")
    if echo "$RESPONSE" | grep -q "OK"; then
        echo "✅ $service externally accessible"
    else
        echo "❌ $service external access failed"
    fi
done

# 4. Test security restrictions
echo "4. Testing security restrictions..."
ADMIN_RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "https://traefik.snorlax.me")
if [ "$ADMIN_RESPONSE" = "401" ] || [ "$ADMIN_RESPONSE" = "403" ]; then
    echo "✅ Admin services properly restricted"
else
    echo "⚠️  Admin service restriction needs review"
fi

# 5. Test performance
echo "5. Testing performance..."
for service in plex grafana sonarr; do
    RESPONSE_TIME=$(curl -s -o /dev/null -w "%{time_total}" "https://$service.snorlax.me" --max-time 10)
    if (( $(echo "$RESPONSE_TIME < 3.0" | bc -l) )); then
        echo "✅ $service response time: ${RESPONSE_TIME}s"
    else
        echo "⚠️  $service slow response time: ${RESPONSE_TIME}s"
    fi
done

echo ""
echo "=== External Access Test Complete ==="
echo ""
echo "If all tests pass, your external access is properly configured!"
echo "Services are accessible at:"
echo "  https://plex.snorlax.me"
echo "  https://grafana.snorlax.me"
echo "  https://sonarr.snorlax.me"
echo "  https://radarr.snorlax.me"
EOF

chmod +x scripts/e2e-external-test.sh
# Run this after DNS has propagated (may take 5-10 minutes)
```

### Create Maintenance Procedures
```bash
# Create maintenance and update procedures
cat > docs/maintenance-procedures.md << 'EOF'
# External Access Maintenance Procedures

## Regular Maintenance Tasks

### Weekly Tasks
- [ ] **Review access logs** for suspicious activity
- [ ] **Check certificate expiry** dates
- [ ] **Verify external service** accessibility
- [ ] **Review security alerts** and failed login attempts
- [ ] **Update Traefik** if new version available

### Monthly Tasks
- [ ] **Review and update** security middleware configurations
- [ ] **Test emergency access** procedures
- [ ] **Backup Traefik** configuration and certificates
- [ ] **Review Cloudflare Tunnel** usage and performance
- [ ] **Update firewall rules** if needed

### Quarterly Tasks
- [ ] **Security audit** of all external services
- [ ] **Update SSL/TLS** configuration for new best practices
- [ ] **Review access patterns** and optimize performance
- [ ] **Test disaster recovery** procedures
- [ ] **Update documentation** with any configuration changes

## Update Procedures

### Updating Traefik
1. **Backup current configuration**:
   ```bash
   ./scripts/cert-backup.sh
   ```

2. **Test new version** in staging:
   ```bash
   docker pull traefik:latest
   # Test with new image before updating production
   ```

3. **Update production**:
   ```bash
   docker-compose pull traefik
   docker-compose up -d traefik
   ```

4. **Verify functionality**:
   ```bash
   ./scripts/validate-external-access.sh
   ```

### Updating Cloudflare Tunnel
1. **Check for updates**: Cloudflare automatically updates
2. **Monitor logs**: `docker logs cloudflared`
3. **Restart if needed**: `docker-compose restart cloudflared`

## Security Incident Response

### Suspicious Activity Detected
1. **Immediate**: Check access logs for attack patterns
2. **Block sources**: Add malicious IPs to Fail2Ban or Cloudflare
3. **Review services**: Check if any unauthorized access occurred
4. **Update security**: Strengthen middleware if needed
5. **Document**: Record incident and response actions

### Certificate Compromise
1. **Immediate**: Revoke compromised certificate
2. **Generate new**: Force new certificate generation
3. **Update services**: Ensure all services use new certificate
4. **Monitor**: Watch for continued suspicious activity
5. **Audit**: Review how compromise occurred

### Service Compromise
1. **Isolate**: Disable external access to compromised service
2. **Investigate**: Check logs for unauthorized activity
3. **Rebuild**: Restore service from known good backup
4. **Harden**: Implement additional security measures
5. **Re-enable**: Restore external access with enhanced security
EOF

# Create incident response script
cat > scripts/incident-response.sh << 'EOF'
#!/bin/bash
# Security incident response script

INCIDENT_LOG="/var/log/security-incidents.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_incident() {
    echo "$DATE: INCIDENT - $1" | tee -a "$INCIDENT_LOG"
}

# Function to block IP address
block_ip() {
    local IP="$1"
    local REASON="$2"
    
    log_incident "Blocking IP $IP - Reason: $REASON"
    
    # Add to Fail2Ban
    sudo fail2ban-client set traefik banip "$IP"
    
    # Add to iptables as backup
    sudo iptables -I INPUT -s "$IP" -j DROP
    
    log_incident "IP $IP blocked successfully"
}

# Function to disable service external access
disable_service_external() {
    local SERVICE="$1"
    
    log_incident "Disabling external access for $SERVICE"
    
    # Remove Traefik labels to disable routing
    docker-compose -f "/home/ubuntu/$SERVICE/docker-compose.yml" down
    
    log_incident "External access disabled for $SERVICE"
}

# Check for common attack indicators
check_attack_indicators() {
    ACCESS_LOG="/home/ubuntu/traefik/logs/access.log"
    
    if [ -f "$ACCESS_LOG" ]; then
        # Check for SQL injection attempts
        SQL_ATTACKS=$(grep -i "SELECT\|UNION\|DROP\|INSERT" "$ACCESS_LOG" | tail -10)
        if [ ! -z "$SQL_ATTACKS" ]; then
            log_incident "SQL injection attempts detected"
        fi
        
        # Check for directory traversal
        TRAVERSAL_ATTACKS=$(grep -E "\.\./|\.\.%2F" "$ACCESS_LOG" | tail -10)
        if [ ! -z "$TRAVERSAL_ATTACKS" ]; then
            log_incident "Directory traversal attempts detected"
        fi
        
        # Check for brute force
        BRUTE_FORCE=$(grep '"OriginStatus":401' "$ACCESS_LOG" | grep "$(date +%Y-%m-%d)" | wc -l)
        if [ "$BRUTE_FORCE" -gt 100 ]; then
            log_incident "Potential brute force attack detected: $BRUTE_FORCE failed logins today"
        fi
    fi
}

case "$1" in
    "block")
        if [ ! -z "$2" ]; then
            block_ip "$2" "${3:-Manual block}"
        else
            echo "Usage: $0 block <ip> [reason]"
        fi
        ;;
    "disable")
        if [ ! -z "$2" ]; then
            disable_service_external "$2"
        else
            echo "Usage: $0 disable <service>"
        fi
        ;;
    "check")
        check_attack_indicators
        ;;
    *)
        echo "Usage: $0 {block|disable|check}"
        echo "  block <ip> [reason] - Block IP address"
        echo "  disable <service> - Disable external access for service"
        echo "  check - Check for attack indicators"
        ;;
esac
EOF

chmod +x scripts/incident-response.sh
```

## Documentation and Handoff

### Create External Access Documentation
```bash
# Create comprehensive documentation
cat > docs/external-access-guide.md << 'EOF'
# External Access Configuration Guide

## Service URLs
- **Plex Media Server**: https://plex.snorlax.me
- **Grafana Monitoring**: https://grafana.snorlax.me
- **Sonarr (TV)**: https://sonarr.snorlax.me
- **Radarr (Movies)**: https://radarr.snorlax.me
- **Prowlarr (Indexers)**: https://prowlarr.snorlax.me
- **Tautulli (Plex Stats)**: https://tautulli.snorlax.me
- **Traefik Dashboard**: https://traefik.snorlax.me (admin only)

## Security Features
- **SSL/TLS**: Automatic certificates via Let's Encrypt + Cloudflare
- **Security Headers**: HSTS, X-Frame-Options, CSP, etc.
- **Rate Limiting**: 50 requests per minute burst, 20 average
- **IP Restrictions**: Admin interfaces limited to internal networks
- **Fail2Ban**: Automatic IP blocking for suspicious activity
- **Access Logging**: All requests logged and analyzed

## Monitoring
- **Uptime Monitoring**: External health checks every minute
- **Certificate Monitoring**: Daily expiry checks
- **Performance Monitoring**: Response time tracking
- **Security Monitoring**: Attack pattern detection

## Maintenance
- **Certificate Renewal**: Automatic via ACME/Let's Encrypt
- **Security Updates**: Manual Traefik updates as needed
- **Log Rotation**: Automatic cleanup of old logs
- **Backup Schedule**: Weekly configuration and certificate backups

## Emergency Access
If external access fails, use internal URLs:
- Traefik: http://192.168.1.23:8080
- Plex: http://192.168.1.20:32400
- Services: http://192.168.1.21:[port]
- Monitoring: http://192.168.1.22:3000

## Support Contacts
- **Cloudflare Issues**: Check Cloudflare status page
- **DNS Issues**: Verify domain registrar settings
- **Certificate Issues**: Check Let's Encrypt status
- **Service Issues**: Refer to service-specific documentation
EOF
```

## Completion Criteria Summary
- [ ] **Traefik reverse proxy** deployed and configured
- [ ] **Cloudflare Tunnel** established and routing traffic
- [ ] **SSL certificates** automatically managed
- [ ] **All services** accessible externally with HTTPS
- [ ] **Security middleware** configured and tested
- [ ] **Rate limiting and protection** active
- [ ] **Access monitoring** and logging operational
- [ ] **Emergency procedures** documented and tested
- [ ] **Performance optimization** applied
- [ ] **Backup and recovery** procedures implemented

### Final External Access Test
Run these final validation steps:

```bash
# Final comprehensive test
./scripts/validate-external-access.sh
./scripts/security-test.sh
./scripts/e2e-external-test.sh

# Check all services are externally accessible:
echo "Final External Access URLs:"
echo "  https://plex.snorlax.me"
echo "  https://grafana.snorlax.me" 
echo "  https://sonarr.snorlax.me"
echo "  https://radarr.snorlax.me"
echo "  https://prowlarr.snorlax.me"
echo "  https://tautulli.snorlax.me"
echo ""
echo "Admin URLs (internal network only):"
echo "  https://traefik.snorlax.me"
echo "  https://proxmox1.snorlax.me"
echo "  https://proxmox2.snorlax.me"
```

**Next Phase**: Proceed to [Gaming Setup](09-gaming-setup.md) to deploy gaming-specific infrastructure including game streaming servers and retro gaming systems.# Phase 7: External Access and Security

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [Monitoring](07-monitoring.md)  
**Next Phase**: [Gaming Setup](09-gaming-setup.md)  
**Duration**: 3-4 hours  
**Prerequisites**: Monitoring operational, all services accessible internally

## Overview
Configure secure external access to your homelab services using Traefik reverse proxy and Cloudflare Tunnel, enabling remote access without exposing your internal network.

## Step 1: Create Traefik VM

### Deploy Traefik VM
```bash
# Clone template for Traefik reverse proxy
qm clone 9000 105 --name traefik-cluster --full --storage asustor-vm-storage

# Configure for reverse proxy workload
qm set 105 --memory 2048 --cores 2
qm set 105 --ipconfig0 ip=192.168.1.23/24,gw=192.168.1.1

# Start Traefik VM
qm start 105

# Wait for boot
sleep 30
```

### Configure Traefik VM
```bash
# SSH to Traefik VM
ssh ubuntu@192.168.1.23

# Create Traefik directory structure
mkdir -p /home/ubuntu/traefik/{config,data,logs,rules}
cd /home/ubuntu/traefik

# Create acme.json file for SSL certificates
touch data/acme.json
chmod 600 data/acme.json

# Create Docker network
docker network create traefik-network --subnet=172.21.0.0/16
docker network create shared-network --subnet=172.20.0.0/16 || echo "Shared network already exists"
```

## Step 2: Configure Cloudflare Integration

### Prepare Cloudflare API Credentials
1. **Login to Cloudflare Dashboard**
2. **Go to My Profile → API Tokens**
3. **Create Token with these permissions:**
   ```
   Zone:Zone:Read
   Zone:DNS:Edit
   Account:Cloudflare Tunnel:Edit
   ```
4. **Note your Zone ID** from domain overview page

### Create Cloudflare Tunnel
1. **Go to Zero Trust → Networks → Tunnels**
2. **Create new tunnel:** "mumbles-cluster-tunnel"
3. **Note the tunnel token** (keep secure!)
4. **Configure Public Hostnames** (we'll add these during deployment)

### Create Environment Configuration
```bash
# Create environment file with Cloudflare credentials
cat > .env << 'EOF'
# Cloudflare Configuration
CF_API_EMAIL=your-email@domain.com
CF_DNS_API_TOKEN=your-cloudflare-dns-api-token
CF_ZONE_ID=your-cloudflare-zone-id
CLOUDFLARE_TUNNEL_TOKEN=your-tunnel-token-here

# Domain Configuration
DOMAIN=snorlax.me
TRAEFIK_DOMAIN=traefik.snorlax.me

# Traefik Configuration
TRAEFIK_API_USER=admin
TRAEFIK_API_PASSWORD=your-secure-traefik-password

# Let's Encrypt
ACME_EMAIL=your-email@domain.com

# Security
TRAEFIK_PILOT_TOKEN=optional-traefik-pilot-token
EOF

# Secure environment file
chmod 600 .env
```

## Step 3: Configure Traefik Static Configuration

### Create Traefik Static Configuration
```bash
# Create traefik.yml static configuration
cat > config/traefik.yml << 'EOF'
global:
  checkNewVersion: false
  sendAnonymousUsage: false

api:
  dashboard: true
  debug: true
  insecure: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entrypoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    address: ":443"
    http:
      tls:
        options: default

  traefik:
    address: ":8080"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: traefik-network
    watch: true

  file:
    directory: /etc/traefik/dynamic
    watch: true

certificatesResolvers:
  cloudflare:
    acme:
      email: your-email@domain.com
      storage: /etc/traefik/acme/acme.json
      dnsChallenge:
        provider: cloudflare
        delayBeforeCheck: 60
        resolvers:
          - "1.1.1.1:53"
          - "1.0.0.1:53"

log:
  level: INFO
  filePath: "/var/log/traefik/traefik.log"
  format: json

accessLog:
  filePath: "/var/log/traefik/access.log"
  format: json
  filters:
    statusCodes:
      - "400-499"
      - "500-599"

metrics:
  prometheus:
    addEntryPointsLabels: true
    addServicesLabels: true
    addRoutersLabels: true
    buckets:
      - 0.1
      - 0.3
      - 1.2
      - 5.0

ping:
  entryPoint: "traefik"
EOF
```

### Create Dynamic Configuration
```bash
# Create dynamic configuration for middleware and TLS
cat > config/dynamic.yml << 'EOF'
http:
  middlewares:
    # Security headers
    security-headers:
      headers:
        frameDeny: true
        sslRedirect: true
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsPreload: true
        stsSeconds: 31536000
        customRequestHeaders:
          X-Forwarded-Proto: "https"
        customResponseHeaders:
          X-Robots-Tag: "noindex,nofollow,nosnippet,noarchive,notranslate,noimageindex"

    # Default headers for all services
    default-headers:
      headers:
        frameDeny: true
        browserXssFilter: true
        contentTypeNosniff: true
        forceSTSHeader: true
        stsIncludeSubdomains: true
        stsSeconds: 31536000

    # IP whitelist for admin interfaces
    admin-whitelist:
      ipWhiteList:
        sourceRange:
          - "192.168.1.0/24"
          - "192.168.10.0/24"
          - "192.168.90.0/24"
          - "10.0.0.0/8"
          - "172.16.0.0/12"

    # Authentication for sensitive services
    basic-auth:
      basicAuth:
        users:
          - "admin:$2y$10$your-hashed-password-here"

    # Rate limiting
    rate-limit:
      rateLimit:
        burst: 100
        average: 50

    # Compress responses
    compression:
      compress: {}

  # TLS configuration
  tls:
    options:
      default:
        sslProtocols:
          - "TLSv1.2"
          - "TLSv1.3"
        cipherSuites:
          - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
          - "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305"
          - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
          - "TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA"
        curvePreferences:
          - "CurveP521"
          - "CurveP384"
        minVersion: "VersionTLS12"

# TCP routers for non-HTTP services (if needed)
tcp:
  routers: {}
  services: {}
EOF
```

## Step 4: Deploy Traefik and Cloudflare Services

### Create Traefik Docker Compose
```bash
# Create docker-compose.yml for Traefik and Cloudflare
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    networks:
      - traefik-network
      - shared-network
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    environment:
      - CF_API_EMAIL=${CF_API_EMAIL}
      - CF_DNS_API_TOKEN=${CF_DNS_API_TOKEN}
      - CF_ZONE_API_TOKEN=${CF_DNS_API_TOKEN}
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./config/traefik.yml:/etc/traefik/traefik.yml:ro
      - ./config/dynamic.yml:/etc/traefik/dynamic/dynamic.yml:ro
      - ./data:/etc/traefik/acme:rw
      - ./logs:/var/log/traefik:rw
    healthcheck:
      test: ["CMD", "traefik", "healthcheck", "--ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.entrypoints=websecure"
      - "traefik.http.routers.traefik.rule=Host(`${TRAEFIK_DOMAIN}`)"
      - "traefik.http.routers.traefik.middlewares=admin-whitelist@file,security-headers@file"
      - "traefik.http.routers.traefik.tls=true"
      - "traefik.http.routers.traefik.tls.certresolver=cloudflare"
      - "traefik.http.routers.traefik.service=api@internal"

  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    networks:
      - traefik-network
    command: tunnel --no-autoupdate run --token ${CLOUDFLARE_TUNNEL_TOKEN}
    environment:
      - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
    healthcheck:
      test: ["CMD", "cloudflared", "tunnel", "info", "${CLOUDFLARE_TUNNEL_TOKEN}"]
      interval: 60s
      timeout: 30s
      retries: 3
      start_period: 60s

  whoami:
    image: traefik/whoami:latest
    container_name: whoami
    restart: unless-stopped
    networks:
      - traefik-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.rule=Host(`whoami.${DOMAIN}`)"
      - "traefik.http.routers.whoami.middlewares=security-headers@file"
      - "traefik.http.routers.whoami.tls=true"
      - "traefik.http.routers.whoami.tls.certresolver=cloudflare"

networks:
  traefik-network:
    external: true
  shared-network:
    external: true
EOF

# Start Traefik services
docker-compose up -d

# Check service status
docker-compose ps
docker-compose logs traefik
docker-compose logs cloudflared
```

## Step 5: Configure Service Integration with Traefik

### Update Plex for Traefik Integration
```bash
# SSH to Plex VM
ssh ubuntu@192.168.1.20

cd /home/ubuntu/plex

# Backup existing configuration
cp docker-compose.yml docker-compose.yml.backup

# Update Plex docker-compose with Traefik labels
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
      - ADVERTISE_IP=http://192.168.1.20:32400/,https://plex.snorlax.me:443
    volumes:
      - ./config:/config
      - ./transcode:/transcode
      - /mnt/media:/data/media:ro
      - /dev/shm:/ram-transcode
    devices:
      - /dev/dri:/dev/dri
    networks:
      - traefik-network
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
      - "traefik.http.routers.plex.middlewares=security-headers@file"
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
      - traefik-network
      - shared-network
    depends_on:
      - plex
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.tautulli.entrypoints=websecure"
      - "traefik.http.routers.tautulli.rule=Host(`tautulli.snorlax.me`)"
      - "traefik.http.routers.tautulli.middlewares=security-headers@file"
      - "traefik.http.routers.tautulli.tls=true"
      - "traefik.http.routers.tautulli.tls.certresolver=cloudflare"

networks:
  traefik-network:
    external: true
  shared-network:
    external: true
EOF

# Create external network and restart Plex
docker network create traefik-network --subnet=172.21.0.0/16 || echo "Network already exists"
docker-compose down && docker-compose up -d

# Verify Plex is accessible via Traefik
curl -k https://localhost/web
```

### Update *arr Stack for Traefik Integration
```bash
# SSH to *arr VM
ssh ubuntu@192.168.1.21

cd /home/ubuntu/arr-stack

# Backup existing configuration
cp docker-compose.yml docker-compose.yml.backup

# Update with Traefik labels (showing Sonarr as example - repeat pattern for all services)
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
      - traefik-network
      - shared-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.sonarr.entrypoints=websecure"
      - "traefik.http.routers.sonarr.rule=Host(`sonarr.snorlax.me`)"
      - "traefik.http.routers.sonarr.middlewares=security-headers@file"
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
      - traefik-network
      - shared-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.radarr.entrypoints=websecure"
      - "traefik.http.routers.radarr.rule=Host(`radarr.snorlax.me`)"
      - "traefik.http.routers.radarr.middlewares=security-headers@file"
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
      - traefik-network
      - shared-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.prowlarr.entrypoints=websecure"
      - "traefik.http.routers.prowlarr.rule=Host(`prowlarr.snorlax.me`)"
      - "traefik.http.routers.prowlarr.middlewares=security-headers@file"
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
      - traefik-network
      - shared-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.qbittorrent.entrypoints=websecure"
      - "traefik.http.routers.qbittorrent.rule=Host(`qbittorrent.snorlax.me`)"
      - "traefik.http.routers.qbittorrent.middlewares=admin-whitelist@file,security-headers@file"
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
      - traefik-network
      - shared-network
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.bazarr.entrypoints=websecure"
      - "traefik.http.routers.bazarr.rule=Host(`bazarr.snorlax.me`)"
      - "traefik.http.routers.bazarr.middlewares=security-headers@file"
      - "traefik.http.routers.bazarr.tls=true"
      - "traefik.http.routers.bazarr.tls.certresolver=cloudflare"

networks:
  traefik-network:
    external: true
  shared-network:
    external: true
EOF

# Restart with new configuration
docker network create traefik-network --subnet=172.21.0.0/16 || echo "Network already exists"
docker-compose down && docker-compose up -d
```

### Update Monitoring for Traefik Integration
```bash
# SSH to monitoring VM
ssh ubuntu@192.168.1.22

cd /home/ubuntu/monitoring

# Add Traefik labels to monitoring services
cat >> docker-compose.yml << 'EOF'

# Add labels to existing services
    labels:
      # Grafana labels
      - "traefik.enable=true"
      - "traefik.http.routers.grafana.entrypoints=websecure"
      - "traefik.http.routers.grafana.rule=Host(`grafana.snorlax.me`)"
      - "traefik.http.routers.grafana.middlewares=security-headers@file"
      - "traefik.http.routers.grafana.tls=true"
      - "traefik.http.routers.grafana.tls.certresolver=cloudflare"
      - "traefik.http.services.grafana.loadbalancer.server.port=3000"

# Add to networks section
    networks:
      - traefik-network
      - shared-network
EOF

# Create external network and restart monitoring
docker network create traefik-network --subnet=172.21.0.0/16 || echo "Network already exists"

# Update the services to include the network (you'll need to edit the compose file properly)
# For now, restart to test connectivity
docker-compose restart grafana
```

## Step 6: Configure Cloudflare Tunnel Public Hostnames

### Configure Public DNS and Routing
Back in **Cloudflare Dashboard → Zero Trust → Networks → Tunnels**:

1. **Edit your tunnel** and add public hostnames:
   ```
   plex.snorlax.me → http://192.168.1.23:80
   sonarr.snorlax.me → http://192.168.1.23:80
   radarr.snorlax.me → http://192.168.1.23:80
   prowlarr.snorlax.me → http://192.168.1.23:80
   qbittorrent.snorlax.me → http://192.168.1.23:80
   tautulli.snorlax.me → http://192.168.1.23:80
   bazarr.snorlax.me → http://192.168.1.23:80
   grafana.snorlax.me → http://192.168.1.23:80
   traefik.snorlax.me → http://192.168.1.23:80
   whoami.snorlax.me → http://192.168.1.23:80
   ```

2. **Configure additional headers** (optional):
   ```
   HTTP Settings:
   - No TLS Verify: Off (keep TLS verification)
   - HTTP2 Connection: On
   - Enable compression: On
   ```

### Test External Access
```bash
# Test external access to services
# From outside your network or using a VPN:

# Test whoami service first (simplest test)
curl -k https://whoami.snorlax.me

# Test Plex external access
curl -k https://plex.snorlax.me/identity

# Test Grafana
curl -k https://grafana.snorlax.me/api/health

# All should return appropriate responses without errors
```

## Step 7: Configure Advanced Security

### Create Security Middleware for Sensitive Services
```bash
# SSH back to Traefik VM
ssh ubuntu@192.168.1.23

cd /home/ubuntu/traefik

# Create additional security configurations
cat > config/security-advanced.yml << 'EOF'
http:
  middlewares:
    # Geo-blocking (block requests from certain countries)
    geo-restrict:
      plugin:
        geoblock:
          allowedCountries:
            - "US"
            - "CA"
          logLocalRequests: false
          logAllowedRequests: false
          logApiRequests: false

    # Advanced rate limiting for public services
    public-rate-limit:
      rateLimit:
        burst: 50
        average: 20
        period: "1m"
        sourceCriterion:
          ipStrategy:
            depth: 1

    # Admin-only services (Proxmox, Traefik dashboard, etc.)
    admin-only:
      chain:
        middlewares:
          - admin-whitelist@file
          - basic-auth@file
          - security-headers@file

    # Public services (Plex, etc.)
    public-access:
      chain:
        middlewares:
          - public-rate-limit@file
          - security-headers@file
          - compression@file

    # Management services (monitoring, etc.)
    management-access:
      chain:
        middlewares:
          - admin-whitelist@file
          - security-headers@file

  # Service-specific routers with enhanced security
  routers:
    # Proxmox web interface (admin only)
    proxmox-node1:
      rule: "Host(`proxmox1.snorlax.me`)"
      entrypoints:
        - websecure
      middlewares:
        - admin-only@file
      tls:
        certResolver: cloudflare
      service: proxmox-node1

    proxmox-node2:
      rule: "Host(`proxmox2.snorlax.me`)"
      entrypoints:
        - websecure
      middlewares:
        - admin-only@file
      tls:
        certResolver: cloudflare
      service: proxmox-node2

  services:
    proxmox-node1:
      loadBalancer:
        servers:
          - url: "https://192.168.1.10:8006"
        serversTransport: insecure-transport

    proxmox-node2:
      loadBalancer:
        servers:
          - url: "https://192.168.1.11:8006"
        serversTransport: insecure-transport

  serversTransports:
    insecure-transport:
      insecureSkipVerify: true
EOF
```

### Generate Secure Passwords and Hashes
```bash
# Create password generation and hashing script
cat > scripts/generate-passwords.sh << 'EOF'
#!/bin/bash
# Generate secure passwords and hashes for authentication

# Function to generate secure password
generate_password() {
    openssl rand -base64 32 | tr -d "=+/" | cut -c1-25
}

# Function to hash password for basic auth
hash_password() {
    local PASSWORD="$1"
    echo "$PASSWORD" | htpasswd -nBC 10 admin | cut -d: -f2
}

# Generate new admin password
ADMIN_PASSWORD=$(generate_password)
ADMIN_HASH=$(hash_password "$ADMIN_PASSWORD")

echo "Generated Admin Credentials:"
echo "Username: admin"
echo "Password: $ADMIN_PASSWORD"
echo "Hash: $ADMIN_HASH"
echo ""
echo "Add this hash to your dynamic.yml basic-auth configuration:"
echo "users:"
echo "  - \"admin:$ADMIN_HASH\""
echo ""
echo "⚠️  Save the password in your password manager!"

# Update dynamic.yml with new hash
sed -i "s/\$2y\$10\$your-hashed-password-here/$ADMIN_HASH/" config/dynamic.yml
EOF

# Install htpasswd for password hashing
sudo apt install -y apache2-utils

chmod +x scripts/generate-passwords.sh
./scripts/generate-passwords.sh
```

## Step 8: Configure SSL Certificate Management

### Configure Automatic Certificate Renewal
```bash
# Create certificate management script
cat > scripts/cert-management.sh << 'EOF'
#!/bin/bash
# SSL certificate management and monitoring

LOG_FILE="/var/log/cert-management.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Check certificate expiration
check_cert_expiration() {
    local DOMAIN="$1"
    local THRESHOLD_DAYS="30"
    
    if command -v openssl >/dev/null 2>&1; then
        EXPIRY_DATE=$(echo | openssl s_client -servername "$DOMAIN" -connect "$DOMAIN:443" 2>/dev/null | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2)
        
        if [ ! -z "$EXPIRY_DATE" ]; then
            EXPIRY_EPOCH=$(date -d "$EXPIRY_DATE" +%s)
            CURRENT_EPOCH=$(date +%s)
            DAYS_UNTIL_EXPIRY=$(( (EXPIRY_EPOCH - CURRENT_EPOCH) / 86400 ))
            
            if [ "$DAYS_UNTIL_EXPIRY" -lt "$THRESHOLD_DAYS" ]; then
                log_message "WARNING: Certificate for $DOMAIN expires in $DAYS_UNTIL_EXPIRY days"
            else
                log_message "Certificate for $DOMAIN expires in $DAYS_UNTIL_EXPIRY days"
            fi
        else
            log_message "Could not check certificate for $DOMAIN"
        fi
    fi
}

# Check ACME certificate status