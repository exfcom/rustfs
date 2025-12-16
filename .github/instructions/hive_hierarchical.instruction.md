---
model: "HIVE"
topology: "Hierarchical"
project: "RustFS"
language: "Rust"
---

# HIVE Hierarchical Orchestration for RustFS

## Orchestration Model

```
                    +──────────────────────────────────────+
                    │      AI AUGMENTED ENGINEER           │
                    │          (QUEEN AGENT)               │
                    │                                      │
                    │  • Coordinates all sub-agents        │
                    │  • Maintains project context         │
                    │  • Enforces quality gates            │
                    │  • Makes final decisions             │
                    +─────────────────┬────────────────────+
                                      │
        +─────────────+───────────────┼───────────────+─────────────+
        │             │               │               │             │
        ▼             ▼               ▼               ▼             ▼
   ┌─────────┐  ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ARCHITECT│  │DEVELOPER│    │SECURITY │    │ TESTER  │    │REVIEWER │
   │  Agent  │  │  Agent  │    │  Agent  │    │  Agent  │    │  Agent  │
   └─────────┘  └─────────┘    └─────────┘    └─────────┘    └─────────┘
```

## Delegation Protocol

### Phase 1: Analysis & Planning
1. **Orchestrator** receives task request
2. **Orchestrator** → **Architect**: Analyze architectural impact
3. **Architect** returns: Component diagram, affected crates, risk assessment
4. **Orchestrator** creates task plan with quality gates

### Phase 2: Design
1. **Orchestrator** → **Architect**: Create HLD if needed
2. **Orchestrator** → **Security**: Review security implications
3. Gate: Design approval required before implementation

### Phase 3: Implementation
1. **Orchestrator** → **Developer**: Implement feature with TDD
2. **Developer** writes tests first, then implementation
3. **Developer** runs local quality checks
4. Gate: All tests must pass

### Phase 4: Quality Assurance
1. **Orchestrator** → **Reviewer**: Code review
2. **Orchestrator** → **Security**: Security audit
3. **Orchestrator** → **Tester**: Additional test coverage
4. Gate: All reviews approved, coverage > 80%

### Phase 5: Finalization
1. **Orchestrator** verifies all gates passed
2. **Orchestrator** generates summary
3. Documentation updated

## Agent Communication Format

### Task Delegation
```yaml
delegation:
  from: orchestrator
  to: developer
  task_id: TASK-001
  type: implementation
  context:
    feature: "Bucket lifecycle management"
    affected_crates: ["rustfs-ecstore", "rustfs-policy"]
    requirements:
      - "S3-compatible lifecycle rules"
      - "Background task scheduling"
  acceptance_criteria:
    - "Unit tests for all lifecycle actions"
    - "Integration test with mock storage"
    - "Documentation updated"
  deadline: null
  priority: high
```

### Status Report
```yaml
status:
  from: developer
  to: orchestrator
  task_id: TASK-001
  status: completed
  artifacts:
    - path: "crates/ecstore/src/lifecycle.rs"
      type: implementation
    - path: "crates/ecstore/tests/lifecycle_test.rs"
      type: test
  quality_metrics:
    tests_passed: 15
    coverage: 87%
    clippy_warnings: 0
  notes: "Implementation complete, ready for review"
```

## Quality Gates

### Gate 1: Design Review
- [ ] Architecture impact assessed
- [ ] Security implications reviewed
- [ ] HLD/LLD created if significant change
- **Decision**: Orchestrator approval

### Gate 2: Implementation Complete
- [ ] All acceptance criteria met
- [ ] Unit tests written and passing
- [ ] `cargo fmt --all --check` passes
- [ ] `cargo clippy -D warnings` passes
- **Decision**: Automatic if all checks pass

### Gate 3: Code Review
- [ ] Reviewer approved changes
- [ ] Security audit passed (for security-sensitive code)
- [ ] No critical issues found
- **Decision**: Reviewer approval

### Gate 4: Final Quality
- [ ] Coverage > 80%
- [ ] All tests pass
- [ ] Documentation updated
- [ ] `make pre-commit` passes
- **Decision**: Orchestrator approval

## Escalation Protocol

### When to Escalate
1. **Conflicting requirements**: Escalate to orchestrator for resolution
2. **Security concerns**: Immediately notify security agent and orchestrator
3. **Architecture changes**: Require architect review
4. **Blocked tasks**: Report to orchestrator with blocker details

### Escalation Format
```yaml
escalation:
  from: developer
  to: orchestrator
  severity: high
  type: conflict
  description: "Performance optimization conflicts with security requirement"
  options:
    - "Option A: Prioritize performance, accept security trade-off"
    - "Option B: Maintain security, accept performance impact"
  recommendation: "Option B with mitigation strategy"
  requires_decision: true
```

## RustFS-Specific Rules

### Crate Modification Protocol
1. Single crate changes: Developer can proceed
2. Cross-crate changes: Architect review required
3. API changes: Full team review required
4. Security crate changes: Security agent must approve

### Test Requirements
| Change Type | Unit Tests | Integration Tests | E2E Tests |
|-------------|------------|-------------------|-----------|
| Bug fix | Required | If applicable | Optional |
| New feature | Required | Required | Required |
| Refactor | Maintain coverage | If applicable | Optional |
| Security fix | Required | Required | Required |

### Documentation Requirements
| Change Type | Code Docs | README | Architecture Docs |
|-------------|-----------|--------|-------------------|
| New public API | Required | If user-facing | If architectural |
| Config change | Required | Required | Optional |
| New crate | Required | Required | Required |
