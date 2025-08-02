# Phase 4: Shared Infrastructure Deployment

**Parent Guide**: [Master Deployment Guide](master_deployment_guide.md)  
**Previous Phase**: [Storage Setup](04-storage-setup.md)  
**Next Phase**: [Essential Services](06-essential-services.md)  
**Duration**: 2-3 hours  
**Prerequisites**: Shared storage operational, cluster stable

## Overview
Deploy foundational shared services (PostgreSQL, Redis, networking) that will support all applications across your homelab. This centralizes database management and reduces resource overhead.

## Step 1: Create Shared Infrastructure VM

### Deploy Shared Services VM
```bash
# Clone from template (if you have one) or create new VM
qm create 100 \
    --name shared-infra \
    --memory 4096 \
    --cores 2 \
    --net0 virtio,bridge=vmbr0 \
    --storage asustor-vm-storage

# Create VM disk on shared storage (enables migration)
qm set 100 --scsi0 asustor-vm-storage:32

# Set static IP for infrastructure consistency
qm set 100 --ipconfig0 ip=192.168.1.15/24,gw=192.168.1.1

# Start the VM
qm start 100

# Get console access if needed
qm terminal 100
```

### Initial VM Setup
```bash
# SSH to the new VM
ssh ubuntu@192.168.1.15

# Update system
sudo apt update && sudo apt upgrade -y

# Install Docker and essential tools
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu

# Install additional tools
sudo apt install -y \
    htop \
    iotop \
    ncdu \
    tree \
    curl \
    wget \
    git \
    vim \
    tmux \
    prometheus-node-exporter

# Enable monitoring
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus-node-exporter

# Logout and log back in to activate docker group
exit
ssh ubuntu@192.168.1.15
```

## Step 2: Deploy Shared Infrastructure Services

### Create Shared Infrastructure Directory Structure
```bash
# Create organized directory structure
mkdir -p /home/ubuntu/shared-infra/{config,data,logs,backups,scripts}
cd /home/ubuntu/shared-infra

# Create environment file with secure passwords
cat > .env << 'EOF'
# Database credentials
POSTGRES_USER=shared_admin
POSTGRES_PASSWORD=your-secure-postgres-password-here
POSTGRES_DB=shared_db

# Redis credentials  
REDIS_PASSWORD=your-secure-redis-password-here

# Timezone
TZ=America/New_York

# Network settings
POSTGRES_HOST=postgres
REDIS_HOST=redis
EOF

# Secure the environment file
chmod 600 .env
```

### Deploy Core Infrastructure Services
```bash
# Create docker-compose.yml for shared infrastructure
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15
    container_name: shared-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=en_US.UTF-8"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      - ./config/postgresql.conf:/etc/postgresql/postgresql.conf:ro
      - ./logs:/var/log/postgresql
    ports:
      - "5432:5432"
    networks:
      - shared-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    container_name: shared-redis
    restart: unless-stopped
    command: redis-server /usr/local/etc/redis/redis.conf
    environment:
      - TZ=${TZ}
    volumes:
      - redis_data:/data
      - ./config/redis.conf:/usr/local/etc/redis/redis.conf:ro
      - ./logs:/var/log/redis
    ports:
      - "6379:6379"
    networks:
      - shared-network
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  postgres-backup:
    image: postgres:15
    container_name: postgres-backup
    restart: unless-stopped
    depends_on:
      - postgres
    environment:
      POSTGRES_HOST: postgres
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      TZ: ${TZ}
    volumes:
      - ./backups:/backups
      - ./scripts:/scripts:ro
    networks:
      - shared-network
    command: >
      sh -c "
        echo 'Starting backup service...'
        while true; do
          sleep 86400
          echo 'Starting daily backup at $(date)'
          pg_dump -h postgres -U ${POSTGRES_USER} -d ${POSTGRES_DB} | gzip > /backups/shared_db_$(date +%Y%m%d_%H%M%S).sql.gz
          find /backups -name '*.sql.gz' -mtime +7 -delete
          echo 'Backup completed at $(date)'
        done
      "
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "2"

  redis-backup:
    image: redis:7-alpine
    container_name: redis-backup
    restart: unless-stopped
    depends_on:
      - redis
    environment:
      - TZ=${TZ}
    volumes:
      - ./backups:/backups
      - redis_data:/data:ro
    networks:
      - shared-network
    command: >
      sh -c "
        echo 'Starting Redis backup service...'
        while true; do
          sleep 21600
          echo 'Starting Redis backup at $(date)'
          cp /data/dump.rdb /backups/redis_backup_$(date +%Y%m%d_%H%M%S).rdb
          find /backups -name 'redis_backup_*.rdb' -mtime +3 -delete
          echo 'Redis backup completed at $(date)'
        done
      "

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  shared-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
EOF
```

## Step 3: Create Configuration Files

### PostgreSQL Configuration
```bash
# Create PostgreSQL configuration directory
mkdir -p config

# Create optimized PostgreSQL configuration
cat > config/postgresql.conf << 'EOF'
# PostgreSQL configuration for shared homelab infrastructure
# Optimized for multiple applications with moderate load

# Connection Settings
listen_addresses = '*'
port = 5432
max_connections = 100

# Memory Settings (adjust based on VM memory allocation)
shared_buffers = 512MB
effective_cache_size = 1536MB
work_mem = 16MB
maintenance_work_mem = 128MB

# Checkpoint Settings
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100

# Logging
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_statement = 'mod'
log_min_duration_statement = 1000
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '

# Performance
random_page_cost = 1.1
effective_io_concurrency = 200
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4

# Security
ssl = off
password_encryption = scram-sha-256
EOF
```

### Redis Configuration
```bash
# Create Redis configuration
cat > config/redis.conf << 'EOF'
# Redis configuration for shared homelab infrastructure

# Network
bind 0.0.0.0
port 6379
protected-mode yes

# Authentication
requirepass your-secure-redis-password-here

# Memory management
maxmemory 1gb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000
dbfilename dump.rdb
dir /data

# Logging
loglevel notice
logfile /var/log/redis/redis-server.log

# Performance
tcp-keepalive 300
timeout 0
tcp-backlog 511
databases 16

# Security
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command KEYS ""
rename-command CONFIG "CONFIG_b835729c8b6f4444ac7b6b7c6503e7ad"
EOF

# Update Redis password in config file to match .env
sed -i "s/your-secure-redis-password-here/$(grep REDIS_PASSWORD .env | cut -d'=' -f2)/" config/redis.conf
```

## Step 4: Create Database Initialization Scripts

```bash
# Create database initialization directory
mkdir -p init-scripts

# Create script to initialize databases for different services
cat > init-scripts/01-create-databases.sql << 'EOF'
-- Create databases for different services
CREATE DATABASE nextcloud_db;
CREATE DATABASE immich_db;
CREATE DATABASE grafana_db;
CREATE DATABASE gitea_db;
CREATE DATABASE authentik_db;
CREATE DATABASE paperless_db;

-- Create users for services (optional, can use shared user)
CREATE USER nextcloud_user WITH PASSWORD 'nextcloud_secure_password_change_me';
CREATE USER immich_user WITH PASSWORD 'immich_secure_password_change_me';
CREATE USER grafana_user WITH PASSWORD 'grafana_secure_password_change_me';
CREATE USER gitea_user WITH PASSWORD 'gitea_secure_password_change_me';
CREATE USER authentik_user WITH PASSWORD 'authentik_secure_password_change_me';
CREATE USER paperless_user WITH PASSWORD 'paperless_secure_password_change_me';

-- Grant permissions
GRANT ALL PRIVILEGES ON DATABASE nextcloud_db TO nextcloud_user;
GRANT ALL PRIVILEGES ON DATABASE immich_db TO immich_user;
GRANT ALL PRIVILEGES ON DATABASE grafana_db TO grafana_user;
GRANT ALL PRIVILEGES ON DATABASE gitea_db TO gitea_user;
GRANT ALL PRIVILEGES ON DATABASE authentik_db TO authentik_user;
GRANT ALL PRIVILEGES ON DATABASE paperless_db TO paperless_user;

-- Create extensions that services commonly need
\c nextcloud_db;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

\c immich_db;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

\c authentik_db;
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";
EOF

# Create Redis database allocation script  
cat > init-scripts/02-redis-databases.txt << 'EOF'
# Redis Database Allocation
# Redis supports 16 databases (0-15) by default

Database 0: Default/General purpose
Database 1: Session storage
Database 2: Cache storage  
Database 3: Queue/Job processing
Database 4: Nextcloud cache
Database 5: Immich cache
Database 6: Grafana sessions
Database 7: Authentik sessions
Database 8: Application metrics
Database 9: Temporary data
Database 10-15: Reserved for future use
EOF
```

## Step 5: Deploy and Start Infrastructure Services

### Create External Network
```bash
# Create Docker network for service communication
docker network create shared-network --driver bridge --subnet=172.20.0.0/16

# Verify network creation
docker network ls | grep shared-network
```

### Start Infrastructure Services
```bash
# Start all infrastructure services
docker-compose up -d

# Check service status
docker-compose ps

# View logs to ensure healthy startup
docker-compose logs postgres
docker-compose logs redis

# Should see successful startup messages and health checks passing
```

### Verify Service Health
```bash
# Test PostgreSQL connectivity
docker exec shared-postgres pg_isready -U shared_admin -d shared_db

# Test Redis connectivity  
docker exec shared-redis redis-cli --raw incr ping

# Check if databases were created
docker exec shared-postgres psql -U shared_admin -d shared_db -c "\l"

# Should show all databases created by init script
```

## Step 6: Configure Service Discovery and Networking

### Create Service Discovery Configuration
```bash
# Create service discovery script for other VMs
cat > scripts/service-discovery.sh << 'EOF'
#!/bin/bash
# Service discovery helper for connecting to shared infrastructure

SHARED_INFRA_IP="192.168.1.15"
POSTGRES_PORT="5432"
REDIS_PORT="6379"

# Function to test PostgreSQL connectivity
test_postgres() {
    echo "Testing PostgreSQL connectivity..."
    pg_isready -h $SHARED_INFRA_IP -p $POSTGRES_PORT -U shared_admin
    if [ $? -eq 0 ]; then
        echo "✅ PostgreSQL is accessible"
        return 0
    else
        echo "❌ PostgreSQL is not accessible"
        return 1
    fi
}

# Function to test Redis connectivity
test_redis() {
    echo "Testing Redis connectivity..."
    if redis-cli -h $SHARED_INFRA_IP -p $REDIS_PORT -a your-redis-password ping > /dev/null 2>&1; then
        echo "✅ Redis is accessible"
        return 0
    else
        echo "❌ Redis is not accessible"  
        return 1
    fi
}

# Function to get connection strings
get_connection_info() {
    echo "=== Shared Infrastructure Connection Information ==="
    echo "PostgreSQL Host: $SHARED_INFRA_IP"
    echo "PostgreSQL Port: $POSTGRES_PORT"
    echo "PostgreSQL User: shared_admin"
    echo "PostgreSQL Available Databases:"
    echo "  - nextcloud_db (user: nextcloud_user)"
    echo "  - immich_db (user: immich_user)"  
    echo "  - grafana_db (user: grafana_user)"
    echo "  - gitea_db (user: gitea_user)"
    echo "  - authentik_db (user: authentik_user)"
    echo "  - paperless_db (user: paperless_user)"
    echo ""
    echo "Redis Host: $SHARED_INFRA_IP"
    echo "Redis Port: $REDIS_PORT"
    echo "Redis Password: [use REDIS_PASSWORD env var]"
    echo "Redis Databases:"
    echo "  - DB 0: Default/General"
    echo "  - DB 1: Sessions"
    echo "  - DB 2: Cache"
    echo "  - DB 3: Queues"
    echo "  - DB 4-15: Service-specific"
}

# Main execution
case "$1" in
    "test")
        test_postgres
        test_redis
        ;;
    "info")
        get_connection_info
        ;;
    *)
        echo "Usage: $0 {test|info}"
        echo "  test - Test connectivity to shared services"
        echo "  info - Display connection information"
        exit 1
        ;;
esac
EOF

chmod +x scripts/service-discovery.sh

# Test the script
./scripts/service-discovery.sh test
./scripts/service-discovery.sh info
```

### Configure Service Health Monitoring
```bash
# Create health monitoring script for infrastructure services
cat > scripts/infra-health-monitor.sh << 'EOF'
#!/bin/bash
# Infrastructure service health monitoring

LOG_FILE="/var/log/infra-health.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

log_message() {
    echo "$DATE: $1" | tee -a "$LOG_FILE"
}

# Check PostgreSQL health
check_postgres() {
    if docker exec shared-postgres pg_isready -U shared_admin -d shared_db > /dev/null 2>&1; then
        # Check database sizes
        DB_SIZES=$(docker exec shared-postgres psql -U shared_admin -d shared_db -t -c "
            SELECT datname, pg_size_pretty(pg_database_size(datname)) 
            FROM pg_database 
            WHERE datname NOT IN ('template0', 'template1', 'postgres');
        " 2>/dev/null)
        
        log_message "PostgreSQL: Healthy - Databases: $(echo "$DB_SIZES" | wc -l)"
        return 0
    else
        log_message "PostgreSQL: ERROR - Service not responding"
        return 1
    fi
}

# Check Redis health
check_redis() {
    if docker exec shared-redis redis-cli --raw incr healthcheck > /dev/null 2>&1; then
        REDIS_MEM=$(docker exec shared-redis redis-cli info memory | grep used_memory_human | cut -d: -f2 | tr -d '\r')
        REDIS_KEYS=$(docker exec shared-redis redis-cli dbsize)
        
        log_message "Redis: Healthy - Memory: $REDIS_MEM, Keys: $REDIS_KEYS"
        return 0
    else
        log_message "Redis: ERROR - Service not responding"
        return 1
    fi
}

# Check Docker network connectivity
check_network() {
    if docker network inspect shared-network > /dev/null 2>&1; then
        CONNECTED_CONTAINERS=$(docker network inspect shared-network | grep -c "\"Name\":")
        log_message "Network: Healthy - Connected containers: $CONNECTED_CONTAINERS"
        return 0
    else
        log_message "Network: ERROR - shared-network not available"
        return 1
    fi
}

# Check resource usage
check_resources() {
    MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.1f", $3/$2 * 100.0}')
    CPU_LOAD=$(uptime | awk -F'load average:' '{print $2}' | awk '{print $1}' | sed 's/,//')
    DISK_USAGE=$(df -h / | grep -E "[0-9]+%" | awk '{print $5}' | sed 's/%//')
    
    log_message "Resources: Memory ${MEMORY_USAGE}%, CPU Load ${CPU_LOAD}, Disk ${DISK_USAGE}%"
    
    # Alert thresholds
    if (( $(echo "$MEMORY_USAGE > 85" | bc -l) )); then
        log_message "WARNING: High memory usage (${MEMORY_USAGE}%)"
    fi
    
    if (( $(echo "$DISK_USAGE > 80" | bc -l) )); then
        log_message "WARNING: High disk usage (${DISK_USAGE}%)"
    fi
}

# Run all health checks
POSTGRES_OK=0
REDIS_OK=0
NETWORK_OK=0

check_postgres && POSTGRES_OK=1
check_redis && REDIS_OK=1  
check_network && NETWORK_OK=1
check_resources

# Overall health status
if [ $POSTGRES_OK -eq 1 ] && [ $REDIS_OK -eq 1 ] && [ $NETWORK_OK -eq 1 ]; then
    log_message "Infrastructure: All services healthy"
else
    log_message "Infrastructure: Some services have issues - check logs"
fi

# Keep log size manageable
tail -500 "$LOG_FILE" > "$LOG_FILE.tmp" && mv "$LOG_FILE.tmp" "$LOG_FILE"
EOF

chmod +x scripts/infra-health-monitor.sh

# Schedule health monitoring every 5 minutes
echo "*/5 * * * * /home/ubuntu/shared-infra/scripts/infra-health-monitor.sh" | crontab -

# Test the health monitor
./scripts/infra-health-monitor.sh
cat /var/log/infra-health.log
```

## Step 7: Configure Database Security and Performance

### PostgreSQL Security Hardening
```bash
# Create PostgreSQL security configuration
cat > scripts/postgres-security-setup.sql << 'EOF'
-- PostgreSQL security configuration

-- Create read-only monitoring user
CREATE USER monitoring_user WITH PASSWORD 'monitoring_secure_password_change_me';
GRANT CONNECT ON DATABASE shared_db TO monitoring_user;
GRANT pg_monitor TO monitoring_user;

-- Create application-specific roles with limited permissions
CREATE ROLE app_read_role;
CREATE ROLE app_write_role;

-- Grant appropriate permissions to roles
GRANT CONNECT ON DATABASE shared_db TO app_read_role;
GRANT USAGE ON SCHEMA public TO app_read_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_read_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO app_read_role;

GRANT CONNECT ON DATABASE shared_db TO app_write_role;  
GRANT USAGE ON SCHEMA public TO app_write_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_write_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_write_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_write_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT USAGE, SELECT ON SEQUENCES TO app_write_role;

-- Configure connection limits
ALTER USER shared_admin CONNECTION LIMIT 10;
ALTER USER nextcloud_user CONNECTION LIMIT 15;
ALTER USER immich_user CONNECTION LIMIT 15;
ALTER USER grafana_user CONNECTION LIMIT 5;
ALTER USER gitea_user CONNECTION LIMIT 10;
ALTER USER authentik_user CONNECTION LIMIT 10;
ALTER USER paperless_user CONNECTION LIMIT 5;
ALTER USER monitoring_user CONNECTION LIMIT 3;
EOF

# Apply security configuration
docker exec shared-postgres psql -U shared_admin -d shared_db -f /scripts/postgres-security-setup.sql
```

### Configure Connection Pooling
```bash
# Install PgBouncer for connection pooling
cat > pgbouncer-compose.yml << 'EOF'
version: '3.8'

services:
  pgbouncer:
    image: pgbouncer/pgbouncer:latest
    container_name: pgbouncer
    restart: unless-stopped
    environment:
      DATABASES_HOST: postgres
      DATABASES_PORT: 5432
      DATABASES_USER: shared_admin
      DATABASES_PASSWORD: ${POSTGRES_PASSWORD}
      DATABASES_DBNAME: shared_db
      POOL_MODE: transaction
      SERVER_RESET_QUERY: DISCARD ALL
      MAX_CLIENT_CONN: 100
      DEFAULT_POOL_SIZE: 20
      MIN_POOL_SIZE: 5
      RESERVE_POOL_SIZE: 5
      MAX_DB_CONNECTIONS: 50
    ports:
      - "6432:6432"
    networks:
      - shared-network
    depends_on:
      - postgres
    healthcheck:
      test: ["CMD", "psql", "-h", "127.0.0.1", "-p", "6432", "-U", "shared_admin", "-d", "shared_db", "-c", "SELECT 1"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  shared-network:
    external: true
EOF

# Start PgBouncer
docker-compose -f pgbouncer-compose.yml up -d

# Test PgBouncer connectivity
docker exec pgbouncer psql -h 127.0.0.1 -p 6432 -U shared_admin -d shared_db -c "SELECT version();"
```

## Step 8: Create Service Templates and Documentation

### Create Connection Templates for Services
```bash
# Create connection string templates for easy service integration
cat > scripts/connection-templates.txt << 'EOF'
# Database Connection Templates for Services

## PostgreSQL Direct Connection
Host: 192.168.1.15
Port: 5432
Database: [service_name]_db
Username: [service_name]_user
Password: [from password manager]

## PostgreSQL via PgBouncer (Recommended for high-traffic services)
Host: 192.168.1.15
Port: 6432
Database: [service_name]_db  
Username: [service_name]_user
Password: [from password manager]

## Redis Connection
Host: 192.168.1.15
Port: 6379
Password: [REDIS_PASSWORD from .env]
Database: [0-15, see redis database allocation]

## Docker Compose Network Integration
Add to your service's docker-compose.yml:

networks:
  shared-network:
    external: true

And in your service configuration:
    networks:
      - shared-network

Then use container names for connection:
- PostgreSQL Host: shared-postgres (or via pgbouncer container)
- Redis Host: shared-redis

## Environment Variables Template
Add to your service's .env file:

# Database Configuration
POSTGRES_HOST=192.168.1.15
POSTGRES_PORT=5432
POSTGRES_DB=[service]_db
POSTGRES_USER=[service]_user
POSTGRES_PASSWORD=[service_password]

# Redis Configuration  
REDIS_HOST=192.168.1.15
REDIS_PORT=6379
REDIS_PASSWORD=[redis_password]
REDIS_DB=[assigned_db_number]

# PgBouncer Configuration (optional)
PGBOUNCER_HOST=192.168.1.15
PGBOUNCER_PORT=6432
EOF
```

### Create Service Deployment Helper Scripts
```bash
# Create helper script for deploying new services with shared infrastructure
cat > scripts/deploy-new-service.sh << 'EOF'
#!/bin/bash
# Helper script for deploying new services with shared infrastructure

SERVICE_NAME="$1"
if [ -z "$SERVICE_NAME" ]; then
    echo "Usage: $0 <service-name>"
    echo "Example: $0 nextcloud"
    exit 1
fi

DB_NAME="${SERVICE_NAME}_db"
DB_USER="${SERVICE_NAME}_user"
DB_PASSWORD="$(openssl rand -base64 16)"

echo "Setting up database for service: $SERVICE_NAME"

# Create database and user
docker exec shared-postgres psql -U shared_admin -d shared_db -c "
CREATE DATABASE $DB_NAME;
CREATE USER $DB_USER WITH PASSWORD '$DB_PASSWORD';
GRANT ALL PRIVILEGES ON DATABASE $DB_NAME TO $DB_USER;
"

# Display connection information
echo ""
echo "=== Database Setup Complete ==="
echo "Service: $SERVICE_NAME"
echo "Database: $DB_NAME"
echo "Username: $DB_USER"  
echo "Password: $DB_PASSWORD"
echo "Host: 192.168.1.15"
echo "Port: 5432"
echo ""
echo "Add these to your service's .env file:"
echo "POSTGRES_HOST=192.168.1.15"
echo "POSTGRES_PORT=5432"
echo "POSTGRES_DB=$DB_NAME"
echo "POSTGRES_USER=$DB_USER"
echo "POSTGRES_PASSWORD=$DB_PASSWORD"
echo ""
echo "⚠️  IMPORTANT: Save the password in your password manager!"
EOF

chmod +x scripts/deploy-new-service.sh

# Test script (optional - creates test database)
# ./scripts/deploy-new-service.sh test-service
```

## Step 9: Configure Monitoring Integration

### Expose Metrics for Prometheus
```bash
# Install PostgreSQL exporter
cat > postgres-exporter-compose.yml << 'EOF'
version: '3.8'

services:
  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: postgres-exporter
    restart: unless-stopped
    environment:
      DATA_SOURCE_NAME: "postgresql://monitoring_user:monitoring_secure_password_change_me@postgres:5432/shared_db?sslmode=disable"
    ports:
      - "9187:9187"
    networks:
      - shared-network
    depends_on:
      - postgres

  redis-exporter:
    image: oliver006/redis_exporter:latest
    container_name: redis-exporter
    restart: unless-stopped
    environment:
      REDIS_ADDR: "redis://shared-redis:6379"
      REDIS_PASSWORD: "${REDIS_PASSWORD}"
    ports:
      - "9121:9121"
    networks:
      - shared-network
    depends_on:
      - redis

networks:
  shared-network:
    external: true
EOF

# Start exporters
docker-compose -f postgres-exporter-compose.yml up -d

# Test metrics endpoints
curl -s localhost:9187/metrics | head -10
curl -s localhost:9121/metrics | head -10
```

### Create Infrastructure Monitoring Dashboard Data
```bash
# Create monitoring configuration for Grafana integration
cat > config/monitoring-targets.yml << 'EOF'
# Prometheus targets for shared infrastructure
# Add these to your Prometheus configuration

- job_name: 'shared-infrastructure'
  static_configs:
    - targets:
        - '192.168.1.15:9100'  # Node exporter
        - '192.168.1.15:9187'  # PostgreSQL exporter
        - '192.168.1.15:9121'  # Redis exporter
  scrape_interval: 15s
  metrics_path: /metrics

# Alerting rules for infrastructure services
- alert: PostgreSQLDown
  expr: pg_up == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "PostgreSQL instance is down"
    description: "PostgreSQL on {{ $labels.instance }} has been down for more than 1 minute"

- alert: RedisDown  
  expr: redis_up == 0
  for: 1m
  labels:
    severity: critical
  annotations:
    summary: "Redis instance is down"
    description: "Redis on {{ $labels.instance }} has been down for more than 1 minute"

- alert: HighDatabaseConnections
  expr: pg_stat_database_numbackends / pg_settings_max_connections * 100 > 80
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "High database connection usage"
    description: "Database connection usage is {{ $value }}% on {{ $labels.instance }}"
EOF
```

## Step 10: Configure Backup and Disaster Recovery

### Create Infrastructure Backup Strategy
```bash
# Create comprehensive backup script for infrastructure services
cat > scripts/infra-backup.sh << 'EOF'
#!/bin/bash
# Comprehensive infrastructure backup

BACKUP_DIR="/home/ubuntu/shared-infra/backups"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/infra-backup.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S'): $1" | tee -a "$LOG_FILE"
}

# Backup PostgreSQL
backup_postgres() {
    log_message "Starting PostgreSQL backup"
    
    # Create compressed backup of all databases
    docker exec shared-postgres pg_dumpall -U shared_admin | gzip > "$BACKUP_DIR/postgres_full_backup_$DATE.sql.gz"
    
    # Individual database backups
    for DB in nextcloud_db immich_db grafana_db gitea_db authentik_db paperless_db; do
        docker exec shared-postgres pg_dump -U shared_admin -d "$DB" | gzip > "$BACKUP_DIR/postgres_${DB}_backup_$DATE.sql.gz"
    done
    
    log_message "PostgreSQL backup completed"
}

# Backup Redis
backup_redis() {
    log_message "Starting Redis backup"
    
    # Force Redis to save current state
    docker exec shared-redis redis-cli --raw BGSAVE
    
    # Wait for background save to complete
    sleep 10
    
    # Copy Redis dump file
    docker cp shared-redis:/data/dump.rdb "$BACKUP_DIR/redis_backup_$DATE.rdb"
    
    log_message "Redis backup completed"
}

# Backup Docker configurations
backup_configs() {
    log_message "Starting configuration backup"
    
    # Create tarball of all configuration files
    tar -czf "$BACKUP_DIR/infra_configs_$DATE.tar.gz" \
        docker-compose.yml \
        pgbouncer-compose.yml \
        postgres-exporter-compose.yml \
        .env \
        config/ \
        scripts/ \
        init-scripts/
    
    log_message "Configuration backup completed"
}

# Cleanup old backups
cleanup_old_backups() {
    log_message "Starting backup cleanup"
    
    # Remove backups older than 14 days
    find "$BACKUP_DIR" -name "*.sql.gz" -mtime +14 -delete
    find "$BACKUP_DIR" -name "*.rdb" -mtime +14 -delete
    find "$BACKUP_DIR" -name "*.tar.gz" -mtime +14 -delete
    
    log_message "Backup cleanup completed"
}

# Execute backup
backup_postgres
backup_redis
backup_configs
cleanup_old_backups

log_message "Infrastructure backup cycle completed"

# Copy critical backups to Proxmox backup storage
cp "$BACKUP_DIR"/postgres_full_backup_$DATE.sql.gz /mnt/pve/asustor-backup/ 2>/dev/null || true
cp "$BACKUP_DIR"/infra_configs_$DATE.tar.gz /mnt/pve/asustor-backup/ 2>/dev/null || true
EOF

chmod +x scripts/infra-backup.sh

# Schedule infrastructure backups daily at 2 AM
echo "0 2 * * * /home/ubuntu/shared-infra/scripts/infra-backup.sh" | crontab -l | { cat; echo "0 2 * * * /home/ubuntu/shared-infra/scripts/infra-backup.sh"; } | crontab -

# Test backup script
./scripts/infra-backup.sh
ls -la backups/
```

### Create Disaster Recovery Procedures
```bash
# Create disaster recovery documentation
cat > scripts/disaster-recovery.md << 'EOF'
# Shared Infrastructure Disaster Recovery

## Quick Recovery Steps

### PostgreSQL Recovery
1. **Stop current PostgreSQL container:**
   ```bash
   docker-compose stop postgres
   ```

2. **Restore from backup:**
   ```bash
   # Restore full backup
   zcat backups/postgres_full_backup_[DATE].sql.gz | docker exec -i shared-postgres psql -U shared_admin

   # Or restore individual database
   zcat backups/postgres_[DATABASE]_backup_[DATE].sql.gz | docker exec -i shared-postgres psql -U shared_admin -d [DATABASE]
   ```

3. **Restart services:**
   ```bash
   docker-compose up -d postgres
   ```

### Redis Recovery
1. **Stop Redis container:**
   ```bash
   docker-compose stop redis
   ```

2. **Restore Redis data:**
   ```bash
   docker cp backups/redis_backup_[DATE].rdb shared-redis:/data/dump.rdb
   ```

3. **Restart Redis:**
   ```bash
   docker-compose up -d redis
   ```

### Complete Infrastructure Recovery
1. **Restore configuration files:**
   ```bash
   tar -xzf backups/infra_configs_[DATE].tar.gz
   ```

2. **Recreate Docker network:**
   ```bash
   docker network create shared-network
   ```

3. **Restore all services:**
   ```bash
   docker-compose up -d
   docker-compose -f pgbouncer-compose.yml up -d
   docker-compose -f postgres-exporter-compose.yml up -d
   ```

## Recovery Testing
- Test recovery procedures monthly
- Validate backup integrity weekly
- Document any changes to recovery procedures
EOF
```

## Validation and Testing

### Infrastructure Service Validation
```bash
# Comprehensive validation script
cat > scripts/validate-infrastructure.sh << 'EOF'
#!/bin/bash
# Comprehensive infrastructure validation

echo "=== Shared Infrastructure Validation ==="

# Test PostgreSQL
echo "Testing PostgreSQL..."
if docker exec shared-postgres pg_isready -U shared_admin -d shared_db; then
    echo "✅ PostgreSQL responsive"
    
    # Test database access
    DB_COUNT=$(docker exec shared-postgres psql -U shared_admin -d shared_db -t -c "SELECT count(*) FROM pg_database WHERE datname NOT IN ('template0', 'template1', 'postgres');" | tr -d ' ')
    echo "✅ PostgreSQL databases: $DB_COUNT"
else
    echo "❌ PostgreSQL not responsive"
fi

# Test Redis
echo "Testing Redis..."
if docker exec shared-redis redis-cli --raw ping | grep -q PONG; then
    echo "✅ Redis responsive"
    
    # Test Redis memory usage
    REDIS_MEM=$(docker exec shared-redis redis-cli info memory | grep used_memory_human | cut -d: -f2 | tr -d '\r')
    echo "✅ Redis memory usage: $REDIS_MEM"
else
    echo "❌ Redis not responsive"
fi

# Test PgBouncer
echo "Testing PgBouncer..."
if docker exec pgbouncer psql -h 127.0.0.1 -p 6432 -U shared_admin -d shared_db -c "SELECT 1;" > /dev/null 2>&1; then
    echo "✅ PgBouncer connection pooling working"
else
    echo "❌ PgBouncer not working"
fi

# Test monitoring endpoints
echo "Testing monitoring endpoints..."
if curl -s localhost:9100/metrics | head -1 | grep -q "#"; then
    echo "✅ Node exporter metrics available"
else
    echo "❌ Node exporter not responding"
fi

if curl -s localhost:9187/metrics | head -1 | grep -q "#"; then
    echo "✅ PostgreSQL exporter metrics available"
else
    echo "❌ PostgreSQL exporter not responding"
fi

if curl -s localhost:9121/metrics | head -1 | grep -q "#"; then
    echo "✅ Redis exporter metrics available"
else
    echo "❌ Redis exporter not responding"  
fi

# Test backup systems
echo "Testing backup systems..."
if [ -f "scripts/infra-backup.sh" ] && [ -x "scripts/infra-backup.sh" ]; then
    echo "✅ Backup scripts configured"
else
    echo "❌ Backup scripts missing or not executable"
fi

# Test network connectivity
echo "Testing network connectivity..."
if docker network inspect shared-network > /dev/null 2>&1; then
    echo "✅ Shared network available"
else
    echo "❌ Shared network not available"
fi

echo ""
echo "=== Validation Complete ==="
EOF

chmod +x scripts/validate-infrastructure.sh

# Run validation
./scripts/validate-infrastructure.sh
```

### Performance and Load Testing
```bash
# Create load testing script
cat > scripts/load-test-infrastructure.sh << 'EOF'
#!/bin/bash
# Load testing for shared infrastructure

echo "Starting infrastructure load testing..."

# PostgreSQL load test
echo "Testing PostgreSQL under load..."
for i in {1..10}; do
    docker exec shared-postgres psql -U shared_admin -d shared_db -c "
        CREATE TABLE IF NOT EXISTS load_test_$i (
            id SERIAL PRIMARY KEY,
            data TEXT,
            created_at TIMESTAMP DEFAULT NOW()
        );
        INSERT INTO load_test_$i (data) 
        SELECT 'test data ' || generate_series(1, 1000);
        SELECT COUNT(*) FROM load_test_$i;
        DROP TABLE load_test_$i;
    " > /dev/null &
done

wait

# Redis load test  
echo "Testing Redis under load..."
for i in {1..100}; do
    docker exec shared-redis redis-cli SET "test_key_$i" "test_value_$i" > /dev/null
done

for i in {1..100}; do
    docker exec shared-redis redis-cli GET "test_key_$i" > /dev/null
done

for i in {1..100}; do
    docker exec shared-redis redis-cli DEL "test_key_$i" > /dev/null  
done

echo "Load testing completed. Check service responsiveness."

# Check service health after load test
./scripts/infra-health-monitor.sh
EOF

chmod +x scripts/load-test-infrastructure.sh

# Run load test
./scripts/load-test-infrastructure.sh
```

## Completion Criteria
- [ ] **Shared PostgreSQL** deployed and accessible
- [ ] **Shared Redis** deployed and accessible  
- [ ] **PgBouncer** connection pooling working
- [ ] **Database monitoring** exporters operational
- [ ] **Service databases** created and configured
- [ ] **Backup systems** automated and tested
- [ ] **Health monitoring** scripts running
- [ ] **Connection templates** documented
- [ ] **Disaster recovery** procedures documented
- [ ] **Load testing** completed successfully

### Service Integration Checklist
- [ ] Docker network `shared-network` created
- [ ] Service discovery scripts working
- [ ] Connection string templates ready
- [ ] Database users and permissions configured
- [ ] Redis database allocation documented
- [ ] Monitoring metrics exposed and accessible

### Security Checklist
- [ ] Database users have minimal required permissions
- [ ] Redis authentication enabled with strong password
- [ ] PostgreSQL connections limited per user
- [ ] Monitoring user has read-only access
- [ ] Environment files properly secured (600 permissions)
- [ ] Dangerous Redis commands disabled/renamed

### Performance Baseline
- [ ] PostgreSQL responding in <100ms for simple queries
- [ ] Redis responding in <10ms for key operations
- [ ] Memory usage <50% under normal load
- [ ] CPU usage <25% under normal load
- [ ] Network connectivity stable at 1Gbps
- [ ] Backup operations completing in <15 minutes

**Next Phase**: Proceed to [Essential Services](06-essential-services.md) to deploy Plex, *arr stack, and other core homelab applications using the shared infrastructure you just created.