# RustFS AI Multi-Agent System Configuration

## System Overview

This document configures the AI Augmented Engineer orchestration system for RustFS - a high-performance distributed object storage system written in Rust.

## Project Context

| Property | Value |
|----------|-------|
| Project | RustFS |
| Language | Rust (Edition 2024, rust-version 1.85) |
| License | Apache 2.0 |
| Type | S3-Compatible Object Storage |
| Architecture | Microservices (27 crates) |

## Agent Roster

### Available Agents

| Agent | File | Expertise |
|-------|------|-----------|
| **Architect** | `.github/agents/rustfs-architect.agent.md` | System design, C4 diagrams, architecture decisions |
| **Developer** | `.github/agents/rustfs-developer.agent.md` | Rust implementation, TDD, async patterns |
| **Security** | `.github/agents/rustfs-security.agent.md` | IAM, KMS, encryption, audit logging |
| **Tester** | `.github/agents/rustfs-tester.agent.md` | Unit/integration/e2e/load testing |
| **DevOps** | `.github/agents/rustfs-devops.agent.md` | Docker, Kubernetes, CI/CD, observability |
| **Reviewer** | `.github/agents/rustfs-reviewer.agent.md` | Code quality, security review, best practices |

### Agent Selection Matrix

| Task Type | Primary Agent | Support Agents |
|-----------|---------------|----------------|
| New feature design | Architect | Developer, Security |
| Implementation | Developer | Tester |
| Bug fix | Developer | Tester, Reviewer |
| Security audit | Security | Reviewer |
| Performance tuning | Developer | Architect, Tester |
| CI/CD setup | DevOps | Developer |
| Code review | Reviewer | Security (if security-sensitive) |
| Test creation | Tester | Developer |
| Architecture analysis | Architect | - |
| Deployment | DevOps | Security |

## Technology Prompts

### Available Prompts

| Prompt | File | Domain |
|--------|------|--------|
| **Rust Core** | `.github/prompts/rust-core.prompt.md` | Error handling, types, traits, testing |
| **Rust Async** | `.github/prompts/rust-async.prompt.md` | Tokio, futures, concurrency patterns |
| **S3 API** | `.github/prompts/rust-s3api.prompt.md` | S3 handlers, XML, signatures |

### Prompt Application

```yaml
task_prompt_mapping:
  - task_type: "error_handling"
    prompt: "rust-core"
  - task_type: "async_implementation"
    prompt: "rust-async"
  - task_type: "s3_endpoint"
    prompt: "rust-s3api"
  - task_type: "concurrent_processing"
    prompt: "rust-async"
  - task_type: "xml_serialization"
    prompt: "rust-s3api"
```

## Orchestration Models

### Available Instructions

| Model | File | Use Case |
|-------|------|----------|
| **HIVE** | `.github/instructions/hive_hierarchical.instruction.md` | Complex projects with clear delegation |
| **TEAM** | `.github/instructions/team_mesh.instruction.md` | Collaborative tasks, brainstorming |
| **PIPELINE** | `.github/instructions/pipeline_ring.instruction.md` | Sequential workflows, CI/CD |

### Model Selection Rules

```yaml
model_selection:
  - condition: "Large feature (>3 crates affected)"
    model: HIVE
    reason: "Need clear coordination and quality gates"

  - condition: "Design discussion or exploration"
    model: TEAM
    reason: "Multiple perspectives beneficial"

  - condition: "Standard feature implementation"
    model: PIPELINE
    reason: "Well-defined sequential workflow"

  - condition: "Bug fix or hotfix"
    model: HIVE
    reason: "Fast track with minimal ceremony"

  - condition: "Performance investigation"
    model: TEAM
    reason: "Cross-domain analysis needed"

  - condition: "Security incident response"
    model: HIVE
    reason: "Centralized control, clear escalation"
```

## Quality Standards

### Pre-Commit Requirements

All code changes must pass before submission:

```bash
# Format check
cargo fmt --all --check

# Linting
cargo clippy --all-targets --all-features -- -D warnings

# Unit tests
cargo test --workspace --exclude e2e_test

# Or use the combined command
make pre-commit
```

### Code Review Criteria

| Category | Requirement |
|----------|-------------|
| Formatting | `cargo fmt` compliant |
| Linting | Zero Clippy warnings |
| Testing | Coverage > 80% |
| Documentation | Public APIs documented |
| Error Handling | No `unwrap()` in production code |
| Security | No hardcoded secrets |

### Test Requirements

| Change Type | Unit Tests | Integration | E2E |
|-------------|------------|-------------|-----|
| New feature | Required | Required | Required |
| Bug fix | Required | If applicable | Optional |
| Refactor | Maintain coverage | If applicable | Optional |
| Security fix | Required | Required | Required |

## RustFS-Specific Guidelines

### Crate Organization

```
crates/
├── ahm/          # Async hash map utilities
├── appauth/      # Application authentication
├── audit/        # Audit logging
├── checksums/    # Checksum utilities
├── common/       # Shared types and utilities
├── config/       # Configuration management
├── crypto/       # Cryptographic operations
├── ecstore/      # Erasure coding storage engine
├── filemeta/     # File metadata management
├── iam/          # Identity and Access Management
├── kms/          # Key Management Service
├── lock/         # Distributed locking
├── madmin/       # Admin API
├── mcp/          # MinIO Client Protocol
├── notify/       # Event notifications
├── obs/          # Object storage operations
├── policy/       # IAM policy evaluation
├── protos/       # Protocol buffer definitions
├── rio/          # Rust I/O utilities
├── s3select-api/ # S3 Select API
├── s3select-query/ # S3 Select query engine
├── signer/       # Request signing
├── targets/      # Storage targets
├── utils/        # General utilities
├── workers/      # Background workers
├── zip/          # ZIP archive support
└── e2e_test/     # End-to-end tests
```

### Memory Allocators

```rust
// Linux x86_64 GNU: Jemalloc
#[cfg(all(target_os = "linux", target_arch = "x86_64", target_env = "gnu"))]
#[global_allocator]
static GLOBAL: tikv_jemallocator::Jemalloc = tikv_jemallocator::Jemalloc;

// Other platforms: MiMalloc
#[cfg(not(all(target_os = "linux", target_arch = "x86_64", target_env = "gnu")))]
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

### Error Handling Pattern

```rust
// Use thiserror for error definitions
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StorageError {
    #[error("Object not found: {key}")]
    NotFound { key: String },

    #[error("Access denied for operation: {operation}")]
    AccessDenied { operation: String },

    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

// Bubble errors with Result, never use unwrap() in production
pub async fn get_object(key: &str) -> Result<Object, StorageError> {
    // Implementation
}
```

### Async Patterns

```rust
// Use Tokio for async runtime
use tokio::sync::{RwLock, Mutex, mpsc};

// Spawn blocking for CPU-intensive work
let result = tokio::task::spawn_blocking(move || {
    compute_expensive_operation()
}).await?;

// Use channels for async communication
let (tx, mut rx) = mpsc::channel(100);
```

## Communication Format

### Task Request

```yaml
task:
  id: "TASK-001"
  title: "Implement bucket versioning"
  type: "feature"
  priority: "high"
  requester: "user"
  description: |
    Implement S3-compatible bucket versioning to support
    object version history and recovery.
  acceptance_criteria:
    - "Enable/disable versioning per bucket"
    - "Version IDs for all objects"
    - "List object versions API"
    - "Delete markers for deleted objects"
  constraints:
    - "S3 API compatible"
    - "Performance impact < 5%"
```

### Status Update

```yaml
status:
  task_id: "TASK-001"
  phase: "implementation"
  progress: 60%
  current_activity: "Implementing version storage"
  completed:
    - "Design document"
    - "API handlers"
  in_progress:
    - "Storage layer changes"
  remaining:
    - "Tests"
    - "Documentation"
  blockers: []
  eta: "2 hours"
```

### Completion Report

```yaml
completion:
  task_id: "TASK-001"
  status: "completed"
  summary: |
    Implemented bucket versioning with full S3 compatibility.
    Added 3 new API endpoints and modified storage layer.
  artifacts:
    - "crates/ecstore/src/versioning/"
    - "crates/e2e_test/src/versioning_test.rs"
  metrics:
    files_created: 5
    files_modified: 8
    lines_added: 1247
    lines_removed: 45
    test_coverage: 86%
  quality_checks:
    fmt: pass
    clippy: pass
    tests: pass
  notes: |
    Consider adding version lifecycle rules in future iteration.
```

## Quick Reference

### Common Commands

```bash
# Development
cargo check --all-targets          # Fast validation
cargo build --release              # Release build
./build-rustfs.sh --dev            # Dev build

# Testing
cargo test --workspace --exclude e2e_test  # Unit/integration
cargo test -p e2e_test                      # E2E tests

# Quality
cargo fmt --all                    # Format code
cargo clippy --all-targets -- -D warnings  # Lint
make pre-commit                    # All quality checks

# Run
./scripts/dev_rustfs.sh            # Development server
```

### Key Configuration Files

| File | Purpose |
|------|---------|
| `Cargo.toml` | Workspace dependencies |
| `rustfmt.toml` | Code formatting rules |
| `rust-toolchain.toml` | Rust version |
| `Makefile` | Build automation |
| `.github/workflows/` | CI/CD pipelines |

### Getting Help

1. Check memory files for project context
2. Consult agent files for domain expertise
3. Reference prompt files for coding patterns
4. Review instruction files for workflow guidance
