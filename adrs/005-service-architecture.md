# ADR-005: Container Orchestration Strategy

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

The homelab requires a container orchestration strategy that balances simplicity with advanced features. With plans for 20+ services across 4 Dell nodes, the approach must support both immediate deployment needs and future scalability requirements.

## Decision

Implement a **mixed approach** starting with Docker Compose for core services, then gradually introducing Kubernetes (K3s) for advanced services and learning opportunities.

### Phase 1: Docker Compose Foundation
- Core services (Plex, *arr stack, databases) deployed via Docker Compose on VMs
- Simple, reliable, and well-documented approach
- Easy backup and restore procedures
- Minimal learning curve for immediate deployment

### Phase 2: Kubernetes Integration  
- K3s cluster on Node 4 (Nappy) for development and learning
- Advanced services (GitLab, Harbor, development tools) on Kubernetes
- Gradual migration of appropriate services to K3s
- Maintains Docker Compose for stable core services

### Service Distribution Strategy
```
Docker Compose (Stable Core):
├── Plex Media Server (Drowzee)
├── *arr Stack (Sleepy)  
├── Shared Infrastructure (Any node)
└── Monitoring Stack (Prometheus/Grafana)

Kubernetes (Advanced/Learning):
├── GitLab CE (Development)
├── Harbor Registry (Container management)
├── Development environments
└── Experimental/learning services
```

## Alternatives Considered

- **Option A**: Docker Compose only - Simple but limits advanced features and learning
- **Option B**: Kubernetes only - Complex learning curve and overkill for core services  
- **Option C**: Mixed approach (Selected) - Best of both worlds with gradual complexity
- **Option D**: VM-based services - Traditional but misses containerization benefits

## Consequences

### Positive Impacts
- Immediate deployment capability with Docker Compose
- Learning opportunities with Kubernetes
- Flexibility to choose right tool for each service
- Gradual complexity introduction
- Professional skill development with industry-standard tools
- Scalable foundation for growth

### Negative Impacts
- Managing two different orchestration platforms
- Increased complexity over single approach
- Need to learn both Docker Compose and Kubernetes
- Potential inconsistencies between platforms

### Implementation Requirements
- Deploy Docker Compose services on Proxmox VMs first
- Set up K3s cluster on Node 4 for learning and development
- Create migration procedures for services that benefit from K8s
- Maintain documentation for both platforms
- Implement monitoring across both orchestration types

## Implementation Notes

**Service Selection Criteria**:
- **Docker Compose**: Stable, simple services that rarely change (media stack)
- **Kubernetes**: Complex services, development tools, services requiring scaling

**Learning Path**: Start with simple K3s deployments, gradually migrate appropriate services as expertise grows.

**Backup Strategy**: Different approaches for each platform - Docker volumes vs. persistent volume claims.

**Monitoring**: Unified monitoring across both platforms using Prometheus/Grafana.

**Future Flexibility**: Architecture supports full K8s migration if desired, or maintaining hybrid approach indefinitely.