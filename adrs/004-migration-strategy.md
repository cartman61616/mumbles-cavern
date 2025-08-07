# ADR-004: Service Migration Strategy

**Status**: Accepted  
**Date**: 2025-01-07  
**Decision Owner**: Jonathan Davis

## Context

Current services are running on Snorlax Prime (i5-6500, GTX 970) and need to be migrated to the new Dell OptiPlex Proxmox cluster. The migration must maintain service availability for end users while transitioning to a more robust, scalable infrastructure.

## Decision

Implement a **gradual migration strategy** with fresh service deployments on the Dell cluster, running services in parallel during transition, and maintaining Snorlax Prime as backup during the migration period.

### Migration Approach

**Phase 1**: Fresh deployment of core services on Dell cluster
- Deploy Plex and *arr stack on Dell cluster first
- Keep existing services running on Snorlax Prime
- Run services in parallel to ensure continuity

**Phase 2**: Data and configuration migration
- Export configurations from existing services  
- Import to new Dell cluster services
- Test functionality and performance

**Phase 3**: DNS/traffic cutover
- Update DNS entries to point to Dell cluster services
- Monitor performance and user experience
- Keep Snorlax Prime services as hot backup

**Phase 4**: Snorlax Prime role transition
- Decommission old services after stable operation
- Transition to development/gaming support role
- Repurpose hardware for enhanced capabilities

## Alternatives Considered

- **Option A**: Gradual migration (Selected) - Safest approach with rollback capability
- **Option B**: Big bang migration - Complete cutover in one weekend, higher risk
- **Option C**: Hybrid approach - Migrate non-critical services first, then critical
- **Option D**: Keep services on Snorlax Prime - Misses benefits of new infrastructure

## Consequences

### Positive Impacts
- Minimal service disruption for end users
- Ability to rollback if issues arise
- Time to properly configure and optimize new services
- Learning opportunity without pressure
- Maintains service availability throughout transition

### Negative Impacts
- Longer migration timeline
- Temporary resource overhead (running dual services)
- More complex DNS and configuration management
- Increased monitoring and maintenance during transition

### Implementation Requirements
- Deploy new services on Dell cluster first
- Create comprehensive testing procedures
- Set up monitoring for both old and new services
- Plan DNS cutover strategy with rollback capability
- Document configurations and procedures thoroughly

## Implementation Notes

**Timeline**: 4-6 weeks for complete migration to allow proper testing and optimization.

**Rollback Plan**: DNS-based rollback to Snorlax Prime services if critical issues arise.

**User Communication**: Inform family/users of planned improvements but maintain transparency about any temporary issues.

**Data Safety**: Full backups before any migration steps, with tested restore procedures.

**Performance Validation**: Ensure new services meet or exceed current performance before cutover.

**Final State**: Snorlax Prime transitions to development/gaming support role, leveraging its capabilities in a new way.