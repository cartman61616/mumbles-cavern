# ADR-003: MacBook Pro Tri-Role Integration

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

The MacBook Pro represents significant compute capability that could enhance the homelab infrastructure. Rather than treating it as a separate device, integration into the homelab ecosystem would maximize resource utilization and create a powerful mobile management/development platform.

## Decision

Implement **tri-role MacBook Pro 16" integration** as `dreamy-pro.mumblescavern.local` with the following roles:

### Role 1: Development Workstation (Primary)
- **Network**: Services VLAN (192.168.20.50)
- **Function**: Infrastructure development, code editing, container orchestration
- **Integration**: Direct access to shared infrastructure (PostgreSQL/Redis)
- **Tools**: IDEs, Git, Docker, Kubernetes, Ansible, Terraform

### Role 2: Media Production Hub
- **Function**: High-performance video editing, audio production, content creation
- **Integration**: Direct NFS access to ASUSTOR storage for media workflows
- **Output**: Automated upload to media library with Plex API integration
- **Tools**: Final Cut Pro, Logic Pro, DaVinci Resolve, HandBrake

### Role 3: Mobile Management Station  
- **Function**: Portable homelab administration and monitoring
- **Access**: All homelab web interfaces, SSH to Dell cluster, VPN capability
- **Use Cases**: Remote troubleshooting, documentation updates, field diagnostics
- **Tools**: Web browsers, SSH clients, monitoring dashboards

## Alternatives Considered

- **Option A**: Single role (development only) - Underutilizes powerful hardware
- **Option B**: Dual role (development + media) - Good but misses management opportunities  
- **Option C**: Tri-role integration (Selected) - Maximizes hardware utilization
- **Option D**: Separate from homelab - Wastes integration opportunities

## Consequences

### Positive Impacts
- Maximum utilization of high-performance hardware
- Portable development environment for infrastructure
- Professional media production capabilities
- Mobile administration and troubleshooting
- Seamless integration with centralized storage
- Reduced need for additional hardware purchases

### Negative Impacts
- Increased complexity in network configuration
- Battery life impact from background services
- Potential resource conflicts between roles
- Additional security considerations for multi-role access

### Implementation Requirements
- Configure static IP reservation (192.168.20.50)
- Set up NFS mounts to ASUSTOR storage
- Install development toolchain and containers
- Configure SSH keys for Dell cluster access
- Set up VPN access for remote management
- Create role-switching workflows/profiles

## Implementation Notes

**Network Priority**: Services VLAN provides optimal access to homelab resources while maintaining security separation.

**Role Switching**: Use macOS Spaces and profiles to quickly switch between development, media production, and management contexts.

**Storage Strategy**: 
- Local SSD: Active development projects and media cache
- NFS mounts: Media library and infrastructure configurations  
- Cloud sync: Documentation and collaborative work

**Security**: Implement proper SSH key management and VPN configurations for secure remote access.

**Performance**: Configure Docker Desktop resource limits to prevent interference with media production tasks.

**Future Enhancement**: Potential render farm integration with Dell cluster for distributed media processing.