# Architecture Decision Records (ADRs)

This directory contains Architecture Decision Records for the Mumbles Cavern homelab infrastructure project.

## ADR Format

Each ADR follows a simple structured format:

```markdown
# ADR-XXX: [Decision Title]

**Status**: [Accepted | Rejected | Superseded | Deprecated]  
**Date**: YYYY-MM-DD  
**Decision Owner**: Jonathan Davis

## Context
Brief description of the problem or situation requiring a decision.

## Decision
What was decided and why.

## Alternatives Considered
- Option A: Description and why it was rejected
- Option B: Description and why it was rejected
- Option C: Selected option

## Consequences
- Positive impacts
- Negative impacts or trade-offs
- Implementation requirements

## Implementation Notes
Key steps or considerations for implementation.
```

## ADR Index

| ADR | Title | Status | Date |
|-----|-------|---------|------|
| [001](001-vlan-segmentation-strategy.md) | VLAN Segmentation Strategy | Accepted | 2025-01-07 |
| [002](002-naming-convention-standard.md) | Infrastructure Naming Convention | Accepted | 2025-01-07 |
| [003](003-macbook-pro-integration.md) | MacBook Pro Tri-Role Integration | Accepted | 2025-01-07 |
| [004](004-migration-strategy.md) | Service Migration Strategy | Accepted | 2025-01-07 |
| [005](005-service-architecture.md) | Container Orchestration Strategy | Accepted | 2025-01-07 |
| [006](006-vpn-integration-strategy.md) | NordVPN Integration Strategy | Accepted | 2025-01-07 |
| [007](007-network-vlan-migration-strategy.md) | Network VLAN Migration Strategy | Accepted | 2025-01-07 |
| [008](008-service-deployment-strategy.md) | Service Deployment Strategy | Accepted | 2025-01-07 |

## Guidelines

### When to Create an ADR
- Infrastructure architecture changes
- Technology stack selections
- Security policy decisions
- Network design choices
- Hardware procurement decisions
- Service deployment strategies

### ADR Numbering
- Use 3-digit zero-padded numbers (001, 002, 003...)
- Assign numbers sequentially
- Update the index table above

### Status Updates
- **Accepted**: Decision is active and implemented
- **Rejected**: Decision was considered but not chosen
- **Superseded**: Replaced by a newer ADR
- **Deprecated**: No longer relevant but kept for historical context