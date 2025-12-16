# RustFS Multi-Agent System - Agent Directory

## Overview

This directory contains specialized AI agents for the RustFS distributed object storage system. Each agent has domain expertise tailored to Rust development and S3-compatible storage systems.

## Agent Roster

| Agent File | Role | Domain Expertise |
|------------|------|------------------|
| `rustfs-architect.agent.md` | Solution Architect | System architecture, erasure coding, distributed systems |
| `rustfs-developer.agent.md` | Software Engineer | Rust implementation, async programming, storage APIs |
| `rustfs-security.agent.md` | Security Engineer | IAM, KMS, encryption, access control |
| `rustfs-tester.agent.md` | QA Engineer | Unit tests, integration tests, e2e tests, load testing |
| `rustfs-devops.agent.md` | DevOps Engineer | Docker, Kubernetes, CI/CD, deployment |
| `rustfs-reviewer.agent.md` | Code Reviewer | Code quality, Clippy, security audits |

## How to Use

1. **Direct Invocation**: Reference an agent by name in your prompt:
   ```
   @rustfs-developer implement a new bucket lifecycle feature
   ```

2. **Orchestrated Mode**: Let the AI Augmented Engineer delegate to appropriate agents automatically

3. **Specialized Tasks**: Use domain-specific agents for focused work:
   - Architecture changes → `rustfs-architect`
   - Security features → `rustfs-security`
   - Performance testing → `rustfs-tester`

## Agent Capabilities Matrix

| Capability | Architect | Developer | Security | Tester | DevOps | Reviewer |
|------------|-----------|-----------|----------|--------|--------|----------|
| Code Writing | ⚪ | ✅ | ✅ | ✅ | ✅ | ⚪ |
| Architecture | ✅ | ⚪ | ⚪ | ⚪ | ⚪ | ⚪ |
| Security Audit | ⚪ | ⚪ | ✅ | ⚪ | ⚪ | ✅ |
| Testing | ⚪ | ✅ | ✅ | ✅ | ⚪ | ⚪ |
| Deployment | ⚪ | ⚪ | ⚪ | ⚪ | ✅ | ⚪ |
| Code Review | ⚪ | ⚪ | ✅ | ⚪ | ⚪ | ✅ |
| Documentation | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

Legend: ✅ Primary capability | ⚪ Secondary/Advisory

## RustFS-Specific Context

All agents share knowledge of:
- **27 workspace crates** in `crates/` directory
- **Rust edition 2024** with rust-version 1.85
- **Async runtime**: Tokio with Axum web framework
- **Key patterns**: Erasure coding, distributed locking, event-driven architecture
- **Quality gates**: `cargo fmt`, `cargo clippy -D warnings`, `cargo test`
