# ADR-008: Service Deployment Strategy

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

With the 2-node Proxmox cluster operational and network VLAN migration planned, the homelab needs a strategic approach to service deployment. The primary constraint is the upcoming TrueNAS reset, which requires essential services (particularly Plex) to be operational beforehand to maintain media access during storage reconfiguration.

Multiple deployment approaches are possible, from minimal fast-track deployments to comprehensive infrastructure-as-code implementations.

## Decision

Implement a **phased service deployment strategy** that prioritizes essential services before TrueNAS reset, followed by comprehensive infrastructure expansion.

### Phase 1: Fast-Track Essential Services (Pre-TrueNAS Reset)
Deploy minimal viable services on the 2-node cluster to maintain functionality during storage transition:

```yaml
Priority Services:
  Plex Media Server:
    - Host: drowzee.mumblescavern.local (Node 1)
    - Purpose: Maintain media access during NAS reset
    - Deployment: Docker container with NFS mount
    - Timeline: Before TrueNAS reset

  Basic Media Automation (Optional):
    - Sonarr/Radarr: sleepy.mumblescavern.local (Node 2) 
    - Purpose: Maintain download automation
    - Deployment: Docker containers
    - Timeline: If time permits before reset
```

### Phase 2: Infrastructure Expansion (Post-TrueNAS Reset)
Deploy comprehensive infrastructure with proper architecture:

```yaml
Node Specialization:
  Node 1 (drowzee) - Media Hub:
    - Plex Media Server (hardware transcoding)
    - Tautulli (Plex monitoring)
    - Overseerr (media requests)

  Node 2 (sleepy) - Media Automation:
    - *arr Stack (Sonarr, Radarr, Prowlarr)
    - qBittorrent (download client)
    - Bazarr (subtitle management)

  Node 3 (snooze) - Productivity Services:
    - Nextcloud (file sync and collaboration)
    - Vaultwarden (password management)
    - Paperless-ngx (document management)
    - Navidrome (music streaming)

  Node 4 (nappy) - Development Environment:
    - GitLab CE (code repositories and CI/CD)
    - Harbor (container registry)
    - Code-Server (VS Code in browser)
    - K3s (Kubernetes learning environment)
```

### Phase 3: Advanced Infrastructure
Deploy supporting infrastructure and automation:

```yaml
Shared Infrastructure:
  - PostgreSQL cluster (cross-node redundancy)
  - Redis cluster (caching and sessions)
  - Traefik (reverse proxy and SSL)
  - Monitoring stack (Prometheus + Grafana)

VPN Integration:
  - Centralized VPN gateway
  - Policy-based routing
  - IoT device security

Automation:
  - Ansible playbooks
  - Infrastructure as Code
  - Monitoring and alerting
```

## Alternatives Considered

### Option A: Infrastructure-First Approach
**Description**: Deploy shared infrastructure (databases, monitoring) before applications  
**Rejected because**: Delays essential service availability, complex rollback if TrueNAS reset fails

### Option B: Comprehensive Deployment
**Description**: Deploy all services simultaneously with full infrastructure  
**Rejected because**: Too complex for pre-TrueNAS reset timeline, higher failure risk

### Option C: Postpone Until 4-Node Cluster
**Description**: Wait until all nodes available before any service deployment  
**Rejected because**: Leaves gap in media access during TrueNAS reset

### Option D: Phased Deployment (Selected)
**Description**: Fast-track essentials, then comprehensive expansion  
**Selected because**: Balances immediate needs with long-term architecture

## Consequences

### Positive Impacts

#### Immediate Benefits
- Media services maintained during storage transition
- Reduced risk of service interruption
- Learning opportunity with minimal complexity
- Foundation for future expansion

#### Strategic Benefits
- Service deployment experience before 4-node expansion
- Container orchestration practice
- Network VLAN testing with real workloads
- Documentation refinement

#### Risk Mitigation
- Essential services preserved during infrastructure changes
- Incremental complexity management
- Rollback options at each phase
- Expertise building over time

### Negative Impacts

#### Technical Debt
- Initial deployments may need refactoring for full architecture
- Potential service migrations as infrastructure matures
- Docker-first approach may need Kubernetes conversion later

#### Resource Utilization
- 2-node cluster may be resource-constrained for full service load
- Storage performance dependent on NFS over network
- Limited high-availability options with 2 nodes

### Implementation Requirements

#### Phase 1 Prerequisites
- Network VLAN migration completed
- ASUSTOR NAS accessible from new VLAN
- Docker deployment capability on Proxmox nodes
- Basic monitoring for service health

#### Phase 2 Prerequisites  
- TrueNAS reset completed and optimized
- Nodes 3&4 integrated into cluster
- Shared infrastructure services deployed
- Comprehensive monitoring operational

#### Phase 3 Prerequisites
- Service deployment experience gained
- Infrastructure automation tools ready
- Advanced networking (VPN) implemented
- Production-ready backup and recovery

## Implementation Notes

### Fast-Track Deployment Approach
```yaml
Technology Choices:
  - Docker Compose for simplicity
  - Direct NFS mounts for storage
  - Static IP assignments
  - Basic container networking

Benefits:
  - Rapid deployment (1-2 hours)
  - Easy troubleshooting
  - Minimal dependencies
  - Clear migration path to full architecture
```

### Service Migration Strategy
```yaml
Upgrade Path:
  1. Fast-track services prove networking and storage
  2. Migrate to VM-based deployments for better isolation
  3. Add shared infrastructure (databases, monitoring)
  4. Implement proper service discovery and load balancing
  5. Add high availability and automation
```

### Quality Assurance
```yaml
Each Phase Includes:
  - Service health validation
  - Performance baseline establishment
  - Backup and recovery testing
  - Documentation updates
  - User acceptance testing
```

This phased approach ensures service continuity while building toward comprehensive homelab architecture, with each phase providing value and learning opportunities.