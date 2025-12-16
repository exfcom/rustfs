# RustFS Architecture Documentation

> Auto-generated architecture documentation for RustFS - A high-performance, distributed object storage system built in Rust.

**Generated:** December 16, 2025  
**Version:** 0.0.5  
**License:** Apache 2.0

## Overview

This directory contains comprehensive architecture documentation for the RustFS project, including system design diagrams, component relationships, data flows, and architectural decision records.

## Document Index

### System Architecture
- [System Overview](01-system-overview.md) - High-level system architecture and design principles
- [C4 Architecture Diagrams](02-c4-architecture.md) - Context, Container, Component, and Code diagrams

### Component Documentation
- [Crate Architecture](03-crate-architecture.md) - Detailed crate organization and dependencies
- [Storage Layer Design](04-storage-layer.md) - Erasure coding and storage implementation
- [Security Architecture](05-security-architecture.md) - IAM, KMS, and encryption design

### Behavioral Diagrams
- [Sequence Diagrams](06-sequence-diagrams.md) - Critical workflow sequences
- [Data Flow Diagrams](07-data-flow.md) - Data movement and transformations
- [State Diagrams](08-state-diagrams.md) - Object and system state transitions

### Design Decisions
- [Architecture Decision Records](09-adr/README.md) - ADRs documenting key architectural choices
- [Design Patterns](10-design-patterns.md) - Common patterns used throughout the codebase
- [Technology Stack](11-tech-stack.md) - Technology choices and rationale

## Architecture at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                    RustFS Architecture                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────┐  ┌───────────┐  ┌────────────────────────┐ │
│  │  S3 API   │  │ Admin API │  │   MCP Server (QUIC)    │ │
│  │  (Axum)   │  │ (madmin)  │  │                        │ │
│  └─────┬─────┘  └─────┬─────┘  └───────────┬────────────┘ │
│        │              │                    │              │
│  ┌─────┴──────────────┴────────────────────┴────────────┐ │
│  │              Core Services Layer                      │ │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────────────┐ │ │
│  │  │  IAM   │ │  KMS   │ │ Policy │ │    Notify      │ │ │
│  │  └────────┘ └────────┘ └────────┘ └────────────────┘ │ │
│  └───────────────────────┬──────────────────────────────┘ │
│                          │                                │
│  ┌───────────────────────┴──────────────────────────────┐ │
│  │              Storage Layer (ecstore)                 │ │
│  │  ┌─────────────┐  ┌──────────┐  ┌─────────────────┐ │ │
│  │  │ Erasure Code│  │ Metadata │  │ Distributed Lock│ │ │
│  │  └─────────────┘  └──────────┘  └─────────────────┘ │ │
│  └──────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Key Design Principles

1. **Performance First**: Rust's zero-cost abstractions and memory safety
2. **S3 Compatibility**: 100% compatible with S3 API
3. **Distributed by Design**: Horizontal scalability and fault tolerance
4. **Security by Default**: Zero-trust architecture with IAM and KMS
5. **Observability Built-in**: Prometheus metrics and distributed tracing
6. **Modular Architecture**: 27 independent crates with clear boundaries

## Quick Navigation

### For Architects
Start with [System Overview](01-system-overview.md) → [C4 Architecture](02-c4-architecture.md) → [ADRs](09-adr/README.md)

### For Developers
Start with [Crate Architecture](03-crate-architecture.md) → [Design Patterns](10-design-patterns.md) → [Sequence Diagrams](06-sequence-diagrams.md)

### For Security Engineers
Start with [Security Architecture](05-security-architecture.md) → [IAM Design](05-security-architecture.md#iam) → [KMS Design](05-security-architecture.md#kms)

### For DevOps Engineers
Start with [System Overview](01-system-overview.md) → [Data Flow](07-data-flow.md) → [Technology Stack](11-tech-stack.md)

## Diagram Tools

All PlantUML diagrams can be rendered using:
- [PlantUML Online Server](http://www.plantuml.com/plantuml/uml/)
- VS Code with PlantUML extension
- IntelliJ IDEA with PlantUML plugin
- Command line: `plantuml <filename>.puml`

## Contributing

When making significant architectural changes:
1. Create an ADR documenting the decision
2. Update relevant architecture diagrams
3. Add sequence diagrams for new workflows
4. Update crate dependency documentation

## Maintenance

This documentation should be updated when:
- New crates are added to the workspace
- Major architectural patterns change
- New external dependencies are introduced
- API boundaries are modified
- Security architecture evolves

---

**Maintained by:** RustFS Architecture Team  
**Last Updated:** December 16, 2025  
**Contact:** [GitHub Issues](https://github.com/rustfs/rustfs/issues)
