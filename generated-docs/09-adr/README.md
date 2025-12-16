# Architecture Decision Records (ADRs)

## Overview

This directory contains Architecture Decision Records (ADRs) documenting significant architectural decisions made in the RustFS project. ADRs help maintain historical context and rationale for design choices.

## ADR Format

Each ADR follows this structure:

```markdown
# ADR-XXX: [Title]

## Status

[Proposed | Accepted | Deprecated | Superseded]

## Context

What is the issue that we're seeing that is motivating this decision or change?

## Decision

What is the change that we're proposing and/or doing?

## Consequences

What becomes easier or more difficult to do because of this change?

## Alternatives Considered

What other options were evaluated?

## References

- Links to related issues, discussions, or documentation
```

## Index of ADRs

| ADR | Title | Status | Date |
|-----|-------|--------|------|
| [ADR-001](ADR-001-rust-language.md) | Use Rust as Primary Language | Accepted | 2024-01 |
| [ADR-002](ADR-002-erasure-coding.md) | Reed-Solomon Erasure Coding | Accepted | 2024-01 |
| [ADR-003](ADR-003-axum-framework.md) | Axum for HTTP Server | Accepted | 2024-02 |
| [ADR-004](ADR-004-memory-allocator.md) | Memory Allocator Selection | Accepted | 2024-02 |
| [ADR-005](ADR-005-cargo-workspace.md) | Cargo Workspace Architecture | Accepted | 2024-02 |
| [ADR-006](ADR-006-kms-design.md) | Key Management Service Design | Accepted | 2024-03 |
| [ADR-007](ADR-007-iam-architecture.md) | IAM Architecture | Accepted | 2024-03 |
| [ADR-008](ADR-008-async-runtime.md) | Tokio as Async Runtime | Accepted | 2024-02 |
| [ADR-009](ADR-009-error-handling.md) | Error Handling Strategy | Accepted | 2024-02 |
| [ADR-010](ADR-010-observability.md) | Observability Stack | Accepted | 2024-03 |

## Creating New ADRs

### When to Create an ADR

Create an ADR when making decisions about:
- Architecture patterns or frameworks
- Technology choices
- API design principles
- Security approaches
- Performance trade-offs
- Significant refactoring

### ADR Workflow

1. **Propose**: Create draft ADR with "Proposed" status
2. **Discuss**: Review with team, gather feedback
3. **Decide**: Update based on feedback
4. **Accept**: Change status to "Accepted" once decision is final
5. **Implement**: Reference ADR in related PRs
6. **Review**: Update if circumstances change

### Superseding ADRs

When a decision changes:
1. Create new ADR explaining the new decision
2. Reference the old ADR
3. Mark old ADR as "Superseded by ADR-XXX"
4. Keep old ADR for historical context

## ADR Categories

### Language and Tooling
- Programming language selection
- Build tools and configuration
- Development workflows

### Architecture
- System architecture patterns
- Component boundaries
- Communication protocols

### Security
- Authentication and authorization
- Encryption approaches
- Key management

### Performance
- Optimization strategies
- Caching approaches
- Concurrency models

### Operational
- Deployment strategies
- Monitoring and observability
- Disaster recovery

## Best Practices

1. **Be Specific**: Clearly state the decision and rationale
2. **Include Context**: Explain why the decision was needed
3. **Document Trade-offs**: What are the pros and cons?
4. **Consider Alternatives**: What else was evaluated?
5. **Keep Updated**: Update status if decision changes
6. **Reference Evidence**: Link to benchmarks, research, or discussions

## Template

Use this template for new ADRs:

```markdown
# ADR-XXX: [Title]

## Status

Proposed

## Context

[Describe the context and problem statement]

### Current Situation

[What is the current state?]

### Problem

[What problem are we trying to solve?]

### Requirements

- [Requirement 1]
- [Requirement 2]

## Decision

[Describe the decision]

### Key Points

- [Key decision point 1]
- [Key decision point 2]

### Implementation

[How will this be implemented?]

## Consequences

### Positive

- [Benefit 1]
- [Benefit 2]

### Negative

- [Trade-off 1]
- [Trade-off 2]

### Neutral

- [Neutral consequence 1]

## Alternatives Considered

### Alternative 1: [Name]

**Pros:**
- [Pro 1]

**Cons:**
- [Con 1]

**Why not chosen:**
[Explanation]

### Alternative 2: [Name]

**Pros:**
- [Pro 1]

**Cons:**
- [Con 1]

**Why not chosen:**
[Explanation]

## References

- [Link to issue]
- [Link to discussion]
- [Link to benchmark]
- [External documentation]

## Decision Makers

- [Name] - [Role]
- [Name] - [Role]

## Date

YYYY-MM-DD
```

---

**Maintained by:** RustFS Architecture Team  
**Last Updated:** December 16, 2025
