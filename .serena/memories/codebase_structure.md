# Codebase Structure

## Workspace Organization
This is a Cargo workspace using resolver = "2".

### Root Directory Structure
- **Cargo.toml**: Workspace configuration defining all member crates
- **rustfs/**: Core service binary, main entry at `rustfs/src/main.rs`
- **crates/**: Reusable library crates
- **deploy/**: Deployment manifests and configurations
- **scripts/**: Automation scripts
- **docs/**: Project documentation
- **helm/**: Kubernetes Helm charts
- **test_standalone/**: Local fixtures for standalone flows

### Core Crates (in crates/ directory)
1. **ahm**: Asynchronous Hash Map for concurrent data structures
2. **appauth**: Application authentication and authorization
3. **audit**: Audit target management system with multi-target fan-out
4. **checksums**: Client checksums
5. **common**: Shared utilities and data structures
6. **config**: Configuration management
7. **crypto**: Cryptography and security features
8. **ecstore**: Erasure coding storage implementation
9. **e2e_test**: End-to-end test suite
10. **filemeta**: File metadata management
11. **iam**: Identity and Access Management
12. **kms**: Key Management Service
13. **lock**: Distributed locking implementation
14. **madmin**: Management dashboard and admin API interface
15. **mcp**: MCP server for S3 operations
16. **notify**: Notification system for events
17. **obs**: Observability utilities (Prometheus, Jaeger, logging)
18. **policy**: Policy management
19. **protos**: Protocol Buffer definitions
20. **rio**: Rust I/O utilities and abstractions
21. **s3select-api**: S3 Select API interface
22. **s3select-query**: S3 Select query engine
23. **signer**: Client signer
24. **targets**: Target-specific configurations and utilities
25. **utils**: Utility functions and helpers
26. **workers**: Worker thread pools and task scheduling
27. **zip**: ZIP file handling and compression

### Import Rules
Each crate is defined in the workspace (rustfs-*) with unified version 0.0.5.
Before contributing changes, skim each crate's README or module documentation.

### Key Files
- **build-rustfs.sh**: Multi-platform build script
- **docker-buildx.sh**: Multi-architecture Docker builds
- **Makefile**: Common development tasks
- **rustfmt.toml**: Code formatting configuration
- **rust-toolchain.toml**: Rust toolchain version
