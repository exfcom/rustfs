---
model: "PIPELINE"
topology: "Ring"
project: "RustFS"
language: "Rust"
---

# PIPELINE Ring Orchestration for RustFS

## Orchestration Model

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│  PLANNER   │───►│ ARCHITECT  │───►│ DEVELOPER  │───►│  REVIEWER  │───►│   TESTER   │
│            │    │            │    │            │    │            │    │            │
│ - Define   │    │ - Design   │    │ - Implement│    │ - Review   │    │ - Test     │
│ - Spec     │    │ - Diagram  │    │ - TDD      │    │ - Quality  │    │ - Validate │
└────────────┘    └────────────┘    └────────────┘    └────────────┘    └────────────┘
     │                                                                        │
     └────────────────────────────────────────────────────────────────────────┘
                              Feedback Loop (if issues found)
```

## When to Use PIPELINE Model

### Best For
- **Well-defined features**: Clear requirements, predictable workflow
- **CI/CD integration**: Automated quality gates
- **Standard development**: Following established patterns
- **Batch processing**: Multiple similar tasks

### Not Ideal For
- Exploratory tasks (use TEAM)
- Urgent hotfixes (use HIVE with fast-track)
- Complex cross-cutting concerns (use HIVE or TEAM)

## Pipeline Stages

### Stage 1: PLANNER
**Input**: User requirement or feature request
**Output**: Structured specification

```yaml
stage: planner
input:
  requirement: "Add bucket replication support"
outputs:
  - specification:
      feature_id: FEAT-042
      title: "S3 Cross-Region Bucket Replication"
      description: |
        Implement automatic replication of objects between buckets
        to support disaster recovery and data locality requirements.
      functional_requirements:
        - "Configure replication rules per bucket"
        - "Support prefix-based filtering"
        - "Track replication status per object"
      non_functional_requirements:
        - "Replication lag < 15 minutes for 99% of objects"
        - "No impact on write latency > 10%"
      acceptance_criteria:
        - "Replicated objects match source exactly"
        - "Replication metrics exposed via Prometheus"
      out_of_scope:
        - "Cross-account replication"
        - "Bi-directional sync"
pass_to: architect
```

### Stage 2: ARCHITECT
**Input**: Specification from Planner
**Output**: Technical design

```yaml
stage: architect
input:
  specification: "FEAT-042 from Planner"
outputs:
  - design:
      affected_crates:
        - crates/ecstore: "Core replication logic"
        - crates/workers: "Background replication tasks"
        - crates/notify: "Replication event notifications"
      new_components:
        - ReplicationManager: "Coordinates replication workflows"
        - ReplicationWorker: "Processes replication queue"
      api_changes:
        - endpoint: "PUT /?replication"
          description: "Configure bucket replication"
        - endpoint: "GET /?replication"
          description: "Get replication configuration"
      data_model:
        - ReplicationRule:
            fields: [id, status, priority, filter, destination]
        - ReplicationStatus:
            fields: [object_key, status, last_attempt, error]
      sequence_diagram: "docs/architecture/replication-flow.puml"
pass_to: developer
```

### Stage 3: DEVELOPER
**Input**: Technical design from Architect
**Output**: Implementation with tests

```yaml
stage: developer
input:
  design: "Technical design from Architect"
process:
  1_red:
    - "Write failing tests for ReplicationManager"
    - "Write failing tests for ReplicationWorker"
    - "Write failing tests for API endpoints"
  2_green:
    - "Implement ReplicationManager"
    - "Implement ReplicationWorker"
    - "Implement API handlers"
  3_refactor:
    - "Extract common patterns"
    - "Optimize hot paths"
    - "Add documentation"
outputs:
  - implementation:
      files_created:
        - crates/ecstore/src/replication/mod.rs
        - crates/ecstore/src/replication/manager.rs
        - crates/ecstore/src/replication/worker.rs
        - crates/workers/src/replication_task.rs
      files_modified:
        - crates/ecstore/src/lib.rs
        - rustfs/src/handlers/bucket.rs
      tests:
        - crates/ecstore/tests/replication_test.rs
        - crates/e2e_test/src/replication_e2e.rs
      coverage: 84%
  - quality_check:
      fmt_check: pass
      clippy_check: pass
      tests_pass: true
pass_to: reviewer
```

### Stage 4: REVIEWER
**Input**: Implementation from Developer
**Output**: Review verdict

```yaml
stage: reviewer
input:
  implementation: "Code from Developer"
review_checklist:
  functional:
    - requirement_coverage: "All requirements addressed"
    - edge_cases: "Checked"
  code_quality:
    - naming: "Clear and consistent"
    - complexity: "Acceptable (max cyclomatic: 8)"
    - duplication: "None detected"
  architecture:
    - pattern_adherence: "Follows established patterns"
    - dependency_direction: "Correct"
  security:
    - input_validation: "Present"
    - authorization: "Verified"
  performance:
    - algorithm_efficiency: "O(n) where expected"
    - resource_cleanup: "Verified"
outputs:
  - review_result:
      verdict: approved_with_comments
      comments:
        - file: "crates/ecstore/src/replication/manager.rs"
          line: 145
          type: suggestion
          comment: "Consider extracting retry logic to shared utility"
        - file: "crates/ecstore/src/replication/worker.rs"
          line: 78
          type: nitpick
          comment: "Typo in error message"
      blocking_issues: []
pass_to: tester
# If rejected, returns to developer with feedback
```

### Stage 5: TESTER
**Input**: Reviewed code from Reviewer
**Output**: Test results and quality metrics

```yaml
stage: tester
input:
  reviewed_code: "Approved code from Reviewer"
test_execution:
  unit_tests:
    run: "cargo test -p ecstore --features replication"
    result: pass
    coverage: 84%
  integration_tests:
    run: "cargo test -p e2e_test replication"
    result: pass
  load_tests:
    run: "k6 run tests/load/replication.js"
    results:
      throughput: "450 replications/sec"
      latency_p95: "230ms"
      error_rate: "0.02%"
  api_tests:
    run: "cargo test -p e2e_test replication_api"
    result: pass
outputs:
  - test_report:
      overall_status: pass
      metrics:
        unit_coverage: 84%
        integration_coverage: 72%
        performance_baseline: established
      recommendations:
        - "Add chaos testing for network failures"
        - "Consider property-based tests for rule matching"
pass_to: done
# If tests fail, returns to developer
```

## Pipeline Flow Control

### Normal Flow
```
Planner ──► Architect ──► Developer ──► Reviewer ──► Tester ──► Done
```

### Feedback Loops
```
Reviewer rejected:
Developer ◄── Reviewer
             │
             └── "Issues found, returning for fixes"

Tester failed:
Developer ◄────────────────────────── Tester
                                      │
                                      └── "Test failures, need fixes"
```

### Skip Conditions
```yaml
skip_rules:
  - stage: architect
    skip_if: "Minor bug fix with no design impact"
  - stage: planner
    skip_if: "Technical debt cleanup, spec already exists"
```

## Handoff Protocol

### Standard Handoff
```yaml
handoff:
  from: developer
  to: reviewer
  artifact:
    type: pull_request
    reference: "PR #234"
    summary: "Implement bucket replication (FEAT-042)"
  checklist_completed:
    - "All tests pass"
    - "Code formatted"
    - "Clippy clean"
    - "Documentation updated"
  notes: "Ready for review. Key logic in manager.rs"
```

### Rejection Handoff
```yaml
handoff:
  from: reviewer
  to: developer
  type: rejection
  reason: "Critical issues found"
  issues:
    - severity: high
      file: "manager.rs"
      description: "Race condition in concurrent replication"
      suggested_fix: "Use Arc<Mutex<>> for shared state"
    - severity: medium
      file: "worker.rs"
      description: "Missing error propagation"
  expected_turnaround: "1-2 hours"
```

## CI/CD Integration

### Pipeline Triggers
```yaml
triggers:
  - event: pull_request_opened
    stages: [developer, reviewer, tester]
  - event: pull_request_updated
    stages: [developer, reviewer, tester]
  - event: merge_to_main
    stages: [tester]  # Full regression
```

### Automated Gates
```yaml
automated_gates:
  post_developer:
    - cargo fmt --all --check
    - cargo clippy --all-targets -- -D warnings
    - cargo test --workspace --exclude e2e_test
  post_reviewer:
    - security_scan: snyk
    - dependency_audit: cargo-audit
  post_tester:
    - coverage_threshold: 80%
    - performance_regression: 10%
```

## RustFS Pipeline Examples

### Example 1: Bug Fix Pipeline
```yaml
pipeline: bug_fix
stages:
  planner:
    skip: true  # Bug already documented
  architect:
    skip: true  # No design impact
  developer:
    tasks:
      - "Write failing test reproducing bug"
      - "Fix bug"
      - "Verify fix"
  reviewer:
    focus: "Verify fix correctness, no regressions"
  tester:
    focus: "Regression testing, edge cases"
```

### Example 2: New Feature Pipeline
```yaml
pipeline: new_feature
stages:
  planner:
    output: "Full specification with BDD scenarios"
  architect:
    output: "HLD, component diagrams, API spec"
  developer:
    output: "Implementation following TDD"
  reviewer:
    output: "Thorough review with security check"
  tester:
    output: "Full test suite including load tests"
```

### Example 3: Performance Optimization Pipeline
```yaml
pipeline: performance
stages:
  planner:
    focus: "Define performance targets"
  architect:
    focus: "Analyze bottlenecks, propose solutions"
  developer:
    focus: "Implement optimizations with benchmarks"
  reviewer:
    focus: "Verify no functionality regression"
  tester:
    focus: "Benchmark before/after, validate targets"
```

## Metrics and Monitoring

### Pipeline Metrics
```yaml
metrics:
  cycle_time:
    target: "< 4 hours for standard features"
    current: "3.5 hours average"
  rejection_rate:
    target: "< 20%"
    current: "15%"
  first_pass_rate:
    target: "> 80%"
    current: "85%"
```

### Stage Metrics
```yaml
stage_metrics:
  developer:
    avg_time: "2 hours"
    rework_rate: "12%"
  reviewer:
    avg_time: "45 minutes"
    issues_found_rate: "0.8 per PR"
  tester:
    avg_time: "30 minutes"
    test_failure_rate: "5%"
```
