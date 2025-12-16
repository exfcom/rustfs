---
model: "TEAM"
topology: "Mesh"
project: "RustFS"
language: "Rust"
---

# TEAM Mesh Orchestration for RustFS

## Orchestration Model

```
         ┌─────────────┐
         │  ARCHITECT  │
         └──────┬──────┘
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│DEVELOPER│◄─►│ TESTER │◄─►│REVIEWER │
└────┬────┘ └────┬────┘ └────┬────┘
     │           │           │
     └───────────┼───────────┘
                 │
         ┌──────┴──────┐
         │  SECURITY   │
         └─────────────┘

All agents can communicate directly (mesh topology)
```

## When to Use TEAM Model

### Best For
- **Exploratory tasks**: "How should we implement X?"
- **Design discussions**: "What's the best approach?"
- **Complex refactoring**: Multiple perspectives needed
- **Debugging**: Cross-domain investigation
- **Knowledge sharing**: Learning from different expertise

### Not Ideal For
- Simple bug fixes (use HIVE)
- Well-defined feature implementation (use PIPELINE)
- Urgent security patches (use HIVE with priority)

## Collaboration Protocol

### Phase 1: Gather Perspectives
All relevant agents analyze the task simultaneously:

```yaml
collaboration_request:
  task: "Design multipart upload optimization"
  participants:
    - architect: "Analyze architectural impact"
    - developer: "Assess implementation complexity"
    - security: "Review security implications"
    - tester: "Consider testability"
  timeout: "async"
  consensus_required: true
```

### Phase 2: Discussion Round
Agents share findings and discuss:

```yaml
perspective:
  from: architect
  topic: "Multipart upload optimization"
  analysis: |
    Current implementation uses sequential part processing.
    Recommendation: Parallel processing with configurable concurrency.
  concerns:
    - "Memory pressure with large concurrent uploads"
    - "Need to maintain part ordering guarantees"
  questions_for:
    developer: "What's the current memory footprint per part?"
    security: "Any concerns with parallel processing?"
```

### Phase 3: Consensus Building
Agents work toward shared understanding:

```yaml
consensus:
  topic: "Multipart upload optimization"
  decision: "Implement bounded parallel processing"
  details:
    - "Max 4 concurrent parts by default"
    - "Configurable via environment variable"
    - "Memory limit per upload: 256MB"
  agreed_by: [architect, developer, security, tester]
  action_items:
    - developer: "Implement parallel processor"
    - tester: "Create load tests for memory pressure"
    - security: "Review final implementation"
```

## Direct Communication Patterns

### Developer ↔ Architect
```yaml
query:
  from: developer
  to: architect
  type: design_clarification
  question: "Should lifecycle rules support regex patterns?"
  context: "S3 supports prefix matching, considering regex extension"
```

### Developer ↔ Tester
```yaml
query:
  from: developer
  to: tester
  type: test_strategy
  question: "How should we test concurrent lifecycle execution?"
  proposed_approach: "Mock time advancement + synthetic workload"
```

### Developer ↔ Security
```yaml
query:
  from: developer
  to: security
  type: security_review
  question: "Is this encryption approach sufficient?"
  code_reference: "crates/crypto/src/envelope.rs:45-80"
```

### Tester ↔ Reviewer
```yaml
query:
  from: tester
  to: reviewer
  type: coverage_discussion
  question: "Should we add property-based tests for serialization?"
  current_coverage: "78% line coverage on serde impls"
```

## Brainstorming Sessions

### Session Structure
```yaml
brainstorm:
  topic: "Performance optimization for GetObject"
  facilitator: architect
  participants: [developer, tester, security]
  phases:
    1_diverge:
      duration: "each agent proposes ideas independently"
      output: "list of potential optimizations"
    2_discuss:
      duration: "evaluate each idea together"
      output: "pros/cons for each"
    3_converge:
      duration: "select best approaches"
      output: "prioritized action list"
```

### Idea Contribution Format
```yaml
idea:
  from: developer
  topic: "GetObject performance"
  proposal: "Implement zero-copy streaming with io_uring"
  benefits:
    - "Eliminate memory copies"
    - "Reduce syscall overhead"
  challenges:
    - "Linux-only (io_uring)"
    - "Complexity increase"
  effort_estimate: "2 weeks"
  risk: medium
```

## Conflict Resolution

### When Agents Disagree
```yaml
conflict:
  type: technical_disagreement
  parties: [developer, security]
  topic: "Caching strategy for IAM policies"
  positions:
    developer: "Cache for 5 minutes to reduce latency"
    security: "No caching, always fetch fresh"
  resolution_process:
    1: "Both parties present full rationale"
    2: "Architect moderates discussion"
    3: "Seek compromise (e.g., 30-second cache with invalidation)"
    4: "If no consensus, escalate to user for decision"
```

### Compromise Template
```yaml
compromise:
  topic: "IAM policy caching"
  agreed_solution: |
    - Cache policies for 30 seconds (configurable)
    - Implement cache invalidation on policy update
    - Add cache-control headers for clients
  satisfies:
    developer: "Reduces latency for repeated requests"
    security: "Short TTL + invalidation limits stale policy risk"
  trade_offs:
    - "Some implementation complexity for invalidation"
    - "Slight latency increase vs. 5-minute cache"
```

## RustFS Team Collaboration Examples

### Example 1: New Crate Design
```yaml
team_task:
  type: design
  description: "Design new crates/lifecycle crate"
  participants:
    architect:
      - "Define crate boundaries"
      - "Specify public API"
      - "Document dependencies"
    developer:
      - "Propose internal structure"
      - "Identify reusable components"
    security:
      - "Review access patterns"
      - "Ensure audit logging"
    tester:
      - "Define test strategy"
      - "Identify edge cases"
```

### Example 2: Performance Investigation
```yaml
team_task:
  type: investigation
  description: "Debug slow multipart uploads > 1GB"
  participants:
    developer:
      - "Profile code paths"
      - "Analyze memory allocation"
    tester:
      - "Create reproducible benchmark"
      - "Measure before/after metrics"
    architect:
      - "Review data flow"
      - "Suggest architectural changes"
```

### Example 3: Security Audit Response
```yaml
team_task:
  type: security_response
  description: "Address audit finding: weak HMAC in legacy code"
  participants:
    security:
      - "Assess vulnerability impact"
      - "Define remediation requirements"
    developer:
      - "Implement fix"
      - "Ensure backward compatibility"
    tester:
      - "Create regression tests"
      - "Verify fix effectiveness"
    reviewer:
      - "Review changes"
      - "Validate security requirements met"
```

## Quality Standards in TEAM Model

### Collective Responsibility
- All agents share ownership of quality
- Any agent can raise concerns
- Consensus required for major decisions

### Review Protocol
- Cross-review encouraged (developer reviews tester's test code)
- Security agent reviews all security-sensitive changes
- Architect reviews all API changes

### Documentation
- Design decisions documented with rationale
- All perspectives captured in ADRs
- Meeting notes stored for future reference
