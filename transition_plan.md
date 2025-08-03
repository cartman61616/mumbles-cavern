# Homelab Transition Plan - Current to Final State
*From Dell Cluster Foundation to Enterprise Snorlax Edition*

## Current State Assessment

### ✅ Infrastructure Completed (Excellent Progress!)
Based on your README progress report, you've successfully completed the foundational infrastructure:

```
✅ Hardware Preparation - BIOS updates and optimization complete
✅ Network Infrastructure - VLANs, QoS, and switch configuration complete  
✅ Primary Node - Proxmox installed and inter-VLAN routing working
✅ Cluster Formation - 2-node cluster created and operational
```

### 🔧 Current Infrastructure Status
**Dell Cluster (Operational Foundation)**:
- **pve-node1** (Drowzee): i5-7500T, 32GB RAM, Proxmox VE - Primary cluster node
- **pve-node2** (Sleepy): i5-7500T, 32GB RAM, Proxmox VE - Secondary cluster node  
- **Cluster Name**: "mumbles-cluster" (2-node operational)
- **Network**: VLANs configured with gaming optimization
- **Management**: UniFi Dream Machine Pro + Flex Mini switch

**Gaming Infrastructure (Two Separate Systems)**:
- **beast-rig**: Ryzen 5600X, RTX 3070, 64GB RAM - Primary gaming PC (Gaming VLAN)
- **Snorlax Prime**: Intel i5-6500, 32GB RAM, GTX 970 - Current homelab server (needs transition)

**Storage Infrastructure**:
- **ASUSTOR NAS**: TrueNAS with 24TB storage ready for NFS integration
- **Synology NAS**: Backup storage system

### 🎯 Immediate Next Steps (Focus on Dell Cluster)
Since we're starting fresh on the Dell cluster, priorities are:

1. **Deploy Storage Integration on Dell Cluster**
2. **Deploy Shared Infrastructure Services (PostgreSQL, Redis)**
3. **Implement Proper Node Naming Convention**
4. **Deploy Core Services (Plex, Media Stack)**
5. **Transition Snorlax Prime Role (homelab server → gaming/development)**

## Transition Strategy

### Phase 1: Dell Cluster Foundation & Naming (Week 1)
**Goal**: Complete storage integration and implement proper naming conventions

#### 1.1 Node Naming Standardization (Day 1)
**Current**: Generic Proxmox node names  
**Target**: Snorlax-themed naming convention

```bash
# Rename Proxmox nodes to match Snorlax theme
# On each node, update hostname and Proxmox node names

# Node 1 (Drowzee) → drowzee.mumblescavern.local
hostnamectl set-hostname drowzee
pvesh set /nodes/pve-node1 --node drowzee

# Node 2 (Sleepy) → sleepy.mumblescavern.local  
hostnamectl set-hostname sleepy
pvesh set /nodes/pve-node2 --node sleepy

# Update /etc/hosts on both nodes
echo "192.168.10.10 drowzee.mumblescavern.local drowzee" >> /etc/hosts
echo "192.168.10.11 sleepy.mumblescavern.local sleepy" >> /etc/hosts

# Update cluster configuration
pvecm updatecerts --force
systemctl restart pve-cluster
```

#### 1.2 Storage Integration Deployment (Day 2-4)
**Following**: `storage_setup.md`

```bash
# Deploy shared storage integration on Dell cluster
# Configure ASUSTOR TrueNAS NFS mounts
# Set up local NVMe storage pools  
# Create VM templates for service deployment
# Test storage performance and failover
```

#### 1.3 Network Services Integration (Day 5-7)
```bash
# Configure DNS entries for new naming
# Update DHCP reservations with proper hostnames
# Test inter-node connectivity with new names
# Validate VLAN routing and performance
```

### Phase 2: Core Services Deployment (Week 2)
**Goal**: Deploy fresh shared infrastructure and core services on Dell cluster

#### 2.1 Deploy Shared Infrastructure (Day 1-3)
**Following**: `shared_infrastructure.md`

```bash
# Deploy shared-infra VM on Dell cluster (192.168.10.15)
# Create fresh PostgreSQL + Redis services
# Configure database initialization scripts
# Set up monitoring and backup strategies
# No migration needed - fresh deployment
```

#### 2.2 Essential Services Deployment (Day 4-7)
**Following**: `essential_services.md`

```
Fresh Service Deployment Strategy:
├── Drowzee (Node 1): Plex Media Server
│   └── Optimize for Intel QuickSync transcoding
├── Sleepy (Node 2): *arr Stack + qBittorrent  
│   └── Media automation and download management
├── Shared Infrastructure: PostgreSQL + Redis
│   └── Supporting all services with shared databases
└── Storage: ASUSTOR NAS mounted via NFS
    └── Media library + VM storage + backups
```

### Phase 3: Gaming Integration & Node Expansion (Week 3)
**Goal**: Deploy gaming services and add remaining Dell nodes

#### 3.1 Gaming Infrastructure Deployment
**Following**: `gaming_setup.md`

```
Gaming Services Deployment:
├── beast-rig (192.168.80.10): Primary gaming workstation (separate from homelab)
│   └── Ryzen 5600X, RTX 3070 - dedicated gaming machine
├── Snorlax Prime: Transition from homelab server to gaming/development support
│   └── Intel i5-6500, GTX 970 - can host retro gaming or development
├── Gaming Services on Dell Cluster:
│   ├── Sunshine/Moonlight Server (for streaming from beast-rig)
│   ├── RetroPie/Batocera Server (could run on Snorlax Prime or Dell cluster)
│   ├── Game Library Management
│   └── Save State Synchronization
└── Gaming Device Integration:
    ├── sleepy-deck (192.168.80.20) - Steam Deck
    └── gaming consoles (192.168.80.40+)
```

#### 3.2 Expand Dell Cluster (Day 4-7)
```bash
# Add Node 3 (Snooze) and Node 4 (Nappy) to cluster
# Configure storage upgrade for Node 3 (add NVMe)
# Deploy specialized workloads:
#   - Node 3 (Snooze): Productivity services
#   - Node 4 (Nappy): Development environment
```

### Phase 4: Monitoring & External Access (Week 4)
**Goal**: Deploy monitoring and secure external access

#### 4.1 Monitoring Stack
**Following**: Monitoring documentation (from project knowledge)

```bash
# Deploy on Dell cluster
├── Prometheus (metrics collection)
├── Grafana (dashboards and visualization) 
├── Node Exporters (all nodes and services)
└── Alert Manager (notifications)
```

#### 4.2 External Access Migration
**Following**: `external_access.md`

```bash
# Migrate Traefik to Dell cluster
# Configure Cloudflare Tunnel on new infrastructure
# Test SSL certificate automation
# Implement security hardening
```

### Phase 5: Service Migration Execution (Month 2)
**Goal**: Migrate remaining services from Snorlax Prime to Dell cluster

#### 5.1 Authentication Migration (HIGH RISK)
```
Keycloak Migration Strategy:
1. Deploy new Keycloak on Dell cluster
2. Export users/groups from current Keycloak
3. Import to new Keycloak instance
4. Test authentication against all services
5. Update service configurations
6. Cutover with rollback plan
```

#### 5.2 Database Migration
```
PostgreSQL Migration:
1. pg_dump all databases from Snorlax Prime
2. Import to Dell cluster shared PostgreSQL
3. Update connection strings in services
4. Test all database connectivity
5. Decommission old databases
```

#### 5.3 Reverse Proxy Migration
```
Traefik Migration:
1. Deploy Traefik on Dell cluster VM
2. Configure identical routing rules
3. Test certificate automation
4. DNS cutover to new load balancer
5. Monitor traffic and performance
```

## Detailed Transition Timeline

### Week 1: Dell Cluster Foundation
```
Day 1-2: Node Naming & Organization
├── Rename Proxmox nodes to Snorlax theme
├── Update DNS entries and documentation
├── Configure proper hostnames and certificates
└── Validate cluster communication with new names

Day 3-5: Storage Integration
├── Deploy shared storage on Dell cluster
├── Configure ASUSTOR NFS integration
├── Set up local storage pools on each node
├── Create VM templates with proper naming
└── Validate storage performance

Day 6-7: Foundation Validation
├── Test cluster failover and recovery
├── Benchmark storage and network performance
├── Document current configurations
└── Prepare for service deployment
```

### Week 2: Core Services Deployment
```
Day 1-3: Shared Infrastructure
├── Deploy shared-infra VM (fresh deployment)
├── Configure PostgreSQL + Redis cluster
├── Set up monitoring and backups
├── Test database connectivity and performance
└── Create service deployment templates

Day 4-7: Media Services
├── Deploy Plex on Drowzee (Node 1)
├── Configure hardware transcoding (QuickSync)
├── Deploy *arr stack on Sleepy (Node 2)
├── Set up media automation pipeline
└── Test complete media workflow end-to-end
```

### Week 3: Gaming & Expansion
```
Day 1-3: Gaming Integration
├── Configure gaming VLAN optimization
├── Deploy game streaming services on Dell cluster
├── Set up retro gaming infrastructure
├── Test gaming device connectivity
└── Optimize for low latency gaming

Day 4-7: Node Expansion
├── Add Node 3 (Snooze) to cluster
├── Add Node 4 (Nappy) to cluster
├── Deploy productivity services on Snooze
├── Set up development environment on Nappy
└── Test 4-node cluster operations
```

### Week 4: Monitoring & Access
```
Day 1-3: Monitoring Stack
├── Deploy Prometheus + Grafana stack
├── Configure dashboards and alerting
├── Set up comprehensive monitoring
├── Test alert notifications
└── Create performance baselines

Day 4-7: External Access
├── Deploy Traefik reverse proxy
├── Configure Cloudflare Tunnel
├── Test SSL certificate automation
├── Implement security policies
└── Validate external connectivity
```

## Service Deployment Matrix

### Fresh Deployment Strategy (No Migration Needed)

| Service | Deployment Location | Node Assignment | Priority | Configuration Source |
|---------|-------------------|-----------------|----------|---------------------|
| **Shared Infrastructure** | Dell Cluster VM | Any Node | 🔴 CRITICAL | `shared_infrastructure.md` |
| **Plex Media Server** | Drowzee (Node 1) | Fixed | 🟡 HIGH | `essential_services.md` |
| **Media Automation** | Sleepy (Node 2) | Fixed | 🟡 HIGH | `essential_services.md` |
| **Traefik Proxy** | Dell Cluster VM | Any Node | 🟡 HIGH | `external_access.md` |
| **Monitoring Stack** | Dell Cluster VM | Any Node | 🟢 MEDIUM | Project knowledge |
| **Gaming Services** | Dell Cluster VMs | Any Node | 🟢 MEDIUM | `gaming_setup.md` |
| **Productivity Suite** | Snooze (Node 3) | Fixed | 🔵 LOW | Final state plans |
| **Development Env** | Nappy (Node 4) | Fixed | 🔵 LOW | Final state plans |

### Node Specialization Plan
```
Drowzee (i5-7500T, Node 1): Media Hub
├── Plex Media Server (hardware transcoding)
├── Tautulli (Plex monitoring)
├── Overseerr (media requests)
└── Hardware: Optimized for QuickSync transcoding

Sleepy (i5-7500T, Node 2): Media Automation
├── Sonarr/Radarr/Prowlarr (*arr stack)
├── qBittorrent (download client)
├── Bazarr (subtitle management)
└── Hardware: Optimized for automation workloads

Snooze (i5-7500T, Node 3): Productivity Services
├── Nextcloud (file sync and collaboration)
├── Vaultwarden (password management)
├── Paperless-ngx (document management)
├── Navidrome (music streaming)
└── Hardware: Needs NVMe storage upgrade

Nappy (i5-6500T, Node 4): Development Environment
├── GitLab CE (code repositories and CI/CD)
├── Harbor (container registry)
├── Code-Server (VS Code in browser)
├── K3s (Kubernetes learning environment)
└── Hardware: Development and testing workloads
```

## Hardware Utilization Plan

### Current State
```
beast-rig (Primary Gaming PC):
├── Role: Dedicated gaming workstation (separate from homelab)
├── CPU: Ryzen 5600X (high performance gaming)
├── GPU: RTX 3070 (modern gaming, VR, streaming)
├── RAM: 64GB (content creation, development)
└── Network: Gaming VLAN (192.168.80.10)

Snorlax Prime (Current Homelab Server):
├── Role: Homelab server (transitioning to gaming support)
├── CPU: Intel i5-6500 (older but capable)
├── GPU: GTX 970 (retro gaming, light transcoding)
├── RAM: 32GB (adequate for support services)
└── Future: Gaming support or development workstation

Dell Cluster (Fresh Homelab):
├── Drowzee (Node 1): i5-7500T, QuickSync ready
├── Sleepy (Node 2): i5-7500T, automation workloads
├── Snooze (Node 3): i5-7500T, needs storage upgrade
└── Nappy (Node 4): i5-6500T, development platform
```

### Final State
```
beast-rig (Gaming Workstation):
├── Role: Primary gaming and content creation
├── Services: Gaming clients, streaming, VR
├── Connection: Gaming VLAN (192.168.80.10)
└── Purpose: High-performance gaming and content creation

Snorlax Prime (Gaming Support/Development):
├── Role: Gaming support services OR development workstation
├── Options: RetroPie server, development environment, or gaming VM host
├── Connection: Gaming VLAN or Development VLAN
└── Purpose: Support gaming ecosystem or development workflows

Dell Cluster (Production Homelab):
├── Drowzee: Media hub (Plex, transcoding)
├── Sleepy: Media automation (*arr stack)
├── Snooze: Productivity services (Nextcloud, etc.)
├── Nappy: Development (K3s, GitLab, testing)
└── Shared: Authentication, databases, monitoring

Edge Computing:
├── Raspberry Pi cluster for IoT and edge services
├── iPad Pro dashboard for homelab management
└── Smart home integration via Home Assistant
```

## Repository Evolution Strategy

### Current Repository Structure (GitHub)
```
mumbles-cavern-docs/ (Current)
├── README.md (progress tracking)
├── master_deployment_guide.md
├── Phase guides (network, cluster, storage, etc.)
├── corrected_*.md files (updates)
└── Infrastructure documentation
```

### Evolved Repository Strategy (Post-Migration)
```
Repository Expansion Plan:
├── mumbles-cavern-docs/ (Enhanced documentation)
├── mumbles-cavern-infrastructure/ (IaC and automation)
├── mumbles-cavern-services/ (Service configurations)
├── mumbles-cavern-monitoring/ (Observability stack)
└── mumbles-cavern-gaming/ (Gaming infrastructure)
```

### Repository Transition Actions

#### 1. Enhance Current Repository
```bash
# Add to existing mumbles-cavern-docs repository:
├── final_state_plans.md (comprehensive roadmap)
├── transition_plan.md (this document)
├── service_migration_guides/ (detailed migration steps)
├── infrastructure_as_code/ (Ansible playbooks, scripts)
├── monitoring_configs/ (Prometheus, Grafana configs)
└── disaster_recovery/ (backup and recovery procedures)
```

#### 2. Create Infrastructure Repository
```bash
# New repository: mumbles-cavern-infrastructure
├── ansible/ (server automation and configuration)
├── terraform/ (infrastructure provisioning)
├── docker-compose/ (service deployment files)
├── kubernetes/ (K3s manifests for advanced services)
├── scripts/ (deployment and maintenance automation)
└── backups/ (backup strategies and scripts)
```

#### 3. Service Configuration Repository
```bash
# New repository: mumbles-cavern-services  
├── media/ (Plex, *arr stack configurations)
├── productivity/ (Nextcloud, Vaultwarden, etc.)
├── development/ (GitLab, Harbor, code-server)
├── monitoring/ (Prometheus rules, Grafana dashboards)
├── gaming/ (RetroArch, Sunshine, game management)
└── authentication/ (Keycloak realm exports, SSO config)
```

## Critical Decision Points

### Decision 1: Migration Strategy
**Options**:
- **A**: Gradual migration (recommended) - Keep Snorlax Prime running during transition
- **B**: Big bang migration - Complete cutover in one weekend
- **C**: Hybrid approach - Migrate non-critical services first

**Recommendation**: **Option A (Gradual)** for safety and learning

### Decision 2: Snorlax Prime Future Role
**Options**:
- **A**: Pure gaming workstation (recommended)
- **B**: Development environment + gaming  
- **C**: Backup homelab infrastructure
- **D**: Decommission and use for parts

**Recommendation**: **Option B (Development + Gaming)** - Leverage the powerful hardware

### Decision 3: Service Architecture
**Options**:
- **A**: Docker Compose on VMs (current path, simpler)
- **B**: Kubernetes cluster (future roadmap, more complex)
- **C**: Mixed approach (Docker for core, K8s for advanced)

**Recommendation**: **Option C (Mixed)** - Docker first, K8s later

## Immediate Action Items (This Week)

### Priority 1: Fix Current Infrastructure ⚠️
```bash
# On Snorlax Prime:
1. Backup all current configurations
2. Fix shared-infra docker-compose.yml YAML error
3. Add Cloudflare API tokens to Traefik
4. Test all services and external access
5. Document working configurations
```

### Priority 2: Complete Dell Cluster Storage 🔧
```bash
# On Dell Cluster:
1. Follow storage_setup.md
2. Configure ASUSTOR NFS integration  
3. Set up local storage pools
4. Create and test VM templates
5. Validate storage performance
```

### Priority 3: Plan Migration Strategy 📋
```bash
# Planning and preparation:
1. Document all current service configurations
2. Create migration checklists for each service
3. Set up backup procedures for all data
4. Plan testing procedures for migrated services
5. Create rollback procedures for each migration
```

## Success Metrics

### Phase Completion Criteria

#### Week 1 Success Metrics
- [ ] Snorlax Prime services stable and backed up
- [ ] Dell cluster storage operational and performant
- [ ] All current external access working
- [ ] Migration plan documented and reviewed

#### Month 1 Success Metrics  
- [ ] Core services migrated to Dell cluster
- [ ] Monitoring operational across all infrastructure
- [ ] Gaming services deployed and optimized
- [ ] External access secured and reliable

#### Month 3 Success Metrics
- [ ] All services migrated from Snorlax Prime
- [ ] Advanced features deployed (productivity services)
- [ ] High availability configured and tested
- [ ] Disaster recovery procedures validated

### Performance Targets
```
Media Services:
├── Plex: 6-8 concurrent 1080p transcodes using QuickSync
├── *arr Stack: Automated media management working
└── Download: Full bandwidth utilization (1Gbps)

Gaming Performance:
├── Local latency: <5ms for gaming devices
├── Streaming: 4K60fps via Sunshine/Moonlight
└── Retro gaming: Stable 60fps for all supported systems

Infrastructure:
├── Uptime: 99.9% for critical services
├── Recovery: <4 hours for complete restoration  
├── Response: <200ms for web interfaces
└── Storage: No I/O bottlenecks under normal load
```

## Risk Mitigation

### High-Risk Activities
1. **Authentication Migration**: Users locked out if Keycloak fails
2. **DNS Cutover**: External access lost if Traefik migration fails
3. **Database Migration**: Data loss if backup/restore fails

### Mitigation Strategies
1. **Parallel Deployment**: Keep old services running during transition
2. **Staged Testing**: Test each component thoroughly before cutover
3. **Rollback Procedures**: Documented and tested for each service
4. **Backup Validation**: Test all backups before relying on them
5. **Monitoring**: Deploy monitoring early to catch issues

### Emergency Procedures
```bash
# Emergency rollback to Snorlax Prime
1. Revert DNS records to original configuration
2. Restart services on Snorlax Prime  
3. Verify external access restored
4. Document failure for post-mortem
5. Plan retry with lessons learned
```

## Long-Term Vision Integration

### Repository Strategy (Month 3+)
Transform your current documentation repository into a comprehensive homelab ecosystem:

```
GitHub Organization: mumbles-cavern
├── docs/ (Enhanced documentation with transition notes)
├── infrastructure/ (IaC, automation, deployment scripts)
├── services/ (Service configurations and templates)
├── monitoring/ (Observability configs and dashboards)
├── gaming/ (Gaming infrastructure and optimization)
└── security/ (Security policies and configurations)
```

### Advanced Features Roadmap (Month 6+)
After successful migration, deploy advanced features from your roadmap:

```
Advanced Services:
├── Kubernetes cluster (K3s on Node 4)
├── GitOps workflow (ArgoCD)
├── CI/CD pipeline (GitLab + Harbor)
├── Development environment (Code-Server)
├── Productivity suite (Nextcloud, Paperless, etc.)
└── Smart home integration (Home Assistant)

Edge Computing:
├── Raspberry Pi cluster deployment
├── IoT device integration
├── iPad dashboard backend
└── Smart home automation
```

## Next Actions

### This Week (Immediate)
1. **Implement proper naming** for Dell cluster nodes
2. **Deploy storage integration** on Dell cluster
3. **Create VM templates** with Snorlax naming scheme
4. **Deploy shared infrastructure** (fresh, no migration)
5. **Test cluster operations** with proper naming

### Next Week (Core Services)
1. **Deploy media services** (Plex, *arr stack) on Dell cluster
2. **Configure gaming VLAN** optimization and services
3. **Deploy monitoring** across all infrastructure
4. **Test performance** and optimize configurations
5. **Create backup procedures** for new services

### Month 2 (Full Deployment)
1. **Add remaining nodes** (Snooze, Nappy) to cluster
2. **Deploy productivity** and development services
3. **Implement external** access and security
4. **Configure advanced** gaming infrastructure
5. **Validate complete** homelab functionality

This transition plan bridges your current progress with the comprehensive final state vision, ensuring a smooth migration path while maintaining service continuity and minimizing risk.