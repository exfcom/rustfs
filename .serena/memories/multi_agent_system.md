# RustFS Multi-Agent System Configuration

## Overview

A comprehensive multi-agent AI system has been configured for RustFS development, providing specialized agents, technology prompts, and orchestration models tailored to Rust-based distributed storage development.

## Directory Structure

```
.github/
├── agents/                                    # Agent definitions
│   ├── INSTRUCTION.md                         # Agent roster overview
│   ├── rustfs-architect.agent.md              # Solution Architect
│   ├── rustfs-developer.agent.md              # Software Engineer
│   ├── rustfs-security.agent.md               # Security Engineer
│   ├── rustfs-tester.agent.md                 # QA Engineer
│   ├── rustfs-devops.agent.md                 # DevOps Engineer
│   └── rustfs-reviewer.agent.md               # Code Reviewer
├── prompts/                                   # Technology prompts
│   ├── INSTRUCTION.md                         # Prompts overview
│   ├── rust-core.prompt.md                    # Core Rust patterns
│   ├── rust-async.prompt.md                   # Async Rust patterns
│   └── rust-s3api.prompt.md                   # S3 API patterns
├── instructions/                              # Orchestration models
│   ├── INSTRUCTION.md                         # Instructions overview
│   ├── hive_hierarchical.instruction.md       # HIVE model
│   ├── team_mesh.instruction.md               # TEAM model
│   └── pipeline_ring.instruction.md           # PIPELINE model
└── copilot-instructions.md                    # Main orchestrator config
```

## Agent Capabilities

| Agent | Domain | Key Responsibilities |
|-------|--------|----------------------|
| **Architect** | System Design | C4 diagrams, HLD/LLD, ADRs, pattern analysis |
| **Developer** | Implementation | TDD, Rust patterns, async code, testing |
| **Security** | Security | IAM/KMS review, encryption, audit logging |
| **Tester** | Quality Assurance | Unit/integration/e2e/load testing |
| **DevOps** | Infrastructure | Docker, K8s, CI/CD, observability |
| **Reviewer** | Code Quality | Code review, best practices, security audit |

## Orchestration Models

### HIVE (Hierarchical)
- Central orchestrator coordinates all agents
- Clear delegation and quality gates
- Best for: Complex features, structured workflows

### TEAM (Mesh)
- All agents communicate directly
- Collaborative decision making
- Best for: Design discussions, exploration, debugging

### PIPELINE (Ring)
- Sequential processing through stages
- Automated handoffs and gates
- Best for: Standard features, CI/CD integration

## Technology Prompts

### rust-core.prompt.md
- Error handling with `thiserror`
- Type system patterns (newtype, builder)
- Trait design and implementation
- Testing patterns (unit, property-based)

### rust-async.prompt.md
- Tokio runtime patterns
- Channel communication (mpsc, broadcast)
- Concurrency primitives (RwLock, Mutex)
- Cancellation and timeout handling

### rust-s3api.prompt.md
- S3 API handler patterns
- XML serialization with `quick-xml`
- SigV4 signature verification
- Error response formatting

## Usage Guide

### Selecting the Right Model

| Scenario | Recommended Model |
|----------|-------------------|
| New feature (>3 crates) | HIVE |
| Bug fix | HIVE (fast track) |
| Design discussion | TEAM |
| Standard feature | PIPELINE |
| Security incident | HIVE |
| Performance investigation | TEAM |

### Selecting the Right Agent

| Task | Primary Agent | Support |
|------|---------------|---------|
| Architecture analysis | Architect | - |
| Implementation | Developer | Tester |
| Security review | Security | Reviewer |
| Testing | Tester | Developer |
| Deployment | DevOps | Security |
| Code review | Reviewer | Security |

### Applying Prompts

| Domain | Prompt |
|--------|--------|
| Error handling | rust-core |
| Async/concurrency | rust-async |
| S3 endpoints | rust-s3api |
| Type design | rust-core |
| Testing | rust-core |

## Quality Gates

All agents enforce these quality standards:

1. **Pre-commit**: `cargo fmt`, `cargo clippy -D warnings`, `cargo test`
2. **Code Review**: Coverage > 80%, no unwrap in production
3. **Security**: No hardcoded secrets, proper authorization
4. **Documentation**: Public APIs documented

## Integration with Serena

The multi-agent system integrates with Serena through:

1. **Memory files**: Project context and conventions
2. **Agent files**: Specialized expertise definitions
3. **Prompt files**: Technology-specific patterns
4. **Instruction files**: Workflow orchestration

When working on RustFS:
1. Activate the project with Serena
2. Read relevant memory files for context
3. Select appropriate agent(s) for the task
4. Apply relevant prompts for technical guidance
5. Follow instruction model for workflow coordination
