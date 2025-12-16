# ADR-001: Use Rust as Primary Language

## Status

**Accepted** (January 2024)

## Context

### Current Situation

When starting the RustFS project, we needed to select a programming language for a high-performance, distributed object storage system that would be S3-compatible and handle production workloads.

### Problem

Object storage systems have stringent requirements:
- **Performance**: High throughput and low latency for I/O operations
- **Memory Safety**: Prevent crashes, memory leaks, and data corruption
- **Concurrency**: Handle thousands of concurrent requests
- **Reliability**: 99.99%+ uptime requirements
- **Maintainability**: Large codebase that will grow over time

### Requirements

- Memory safety without garbage collection overhead
- Zero-cost abstractions for performance
- Strong type system to catch bugs at compile time
- Excellent concurrency primitives
- Growing ecosystem for systems programming
- Active community and long-term support

## Decision

**We will use Rust (edition 2024, version 1.85+) as the primary programming language for RustFS.**

### Key Points

1. **Memory Safety**: Rust's ownership system prevents entire classes of bugs:
   - No null pointer dereferences
   - No data races in concurrent code
   - No use-after-free errors
   - No buffer overflows

2. **Performance**: Comparable to C/C++ with zero-cost abstractions
   - No garbage collection pauses
   - Predictable performance characteristics
   - Fine-grained control over memory allocation

3. **Concurrency**: First-class async/await support
   - Tokio runtime for async I/O
   - Strong guarantees against data races
   - Efficient task scheduling

4. **Type System**: Powerful type system and pattern matching
   - Compile-time error checking
   - `Result<T, E>` for error handling
   - Trait system for abstraction

5. **Ecosystem**: Mature crates for systems programming
   - `tokio` for async runtime
   - `axum` for HTTP server
   - `serde` for serialization
   - `thiserror` for error handling

### Implementation

- Use Rust edition 2024 for latest language features
- Maintain rust-version = "1.85" in Cargo.toml
- Use `#![forbid(unsafe_code)]` where possible
- Use `unsafe` only when necessary with clear documentation

## Consequences

### Positive

1. **Memory Safety**: Eliminates entire classes of bugs that plague C/C++ systems
2. **Performance**: Benchmarks show 2-3x better performance than Go implementations
3. **Reliability**: Ownership system prevents data races and memory leaks
4. **Developer Productivity**: Compile-time error catching reduces debugging time
5. **No Runtime**: No garbage collector means predictable latency
6. **Modern Tooling**: `cargo`, `rustfmt`, `clippy` provide excellent developer experience

### Negative

1. **Learning Curve**: Rust has a steep learning curve, especially for:
   - Ownership and borrowing concepts
   - Lifetime annotations
   - Trait system complexity

2. **Compilation Time**: Rust compilation can be slow for large projects
   - Mitigated by incremental compilation
   - Use `sccache` for distributed compilation caching

3. **Ecosystem Maturity**: Some areas less mature than Go/Java
   - Mitigated by rapid ecosystem growth
   - Most needed crates are production-ready

4. **Hiring**: Smaller pool of Rust developers
   - Mitigated by growing popularity
   - Strong motivation to learn Rust

### Neutral

1. **Community Size**: Smaller than Go/Java but rapidly growing
2. **Corporate Backing**: Strong support from Mozilla, Amazon, Microsoft
3. **WASM Support**: Excellent for future edge computing use cases

## Alternatives Considered

### Alternative 1: Go

**Pros:**
- Simpler language, easier to learn
- Fast compilation times
- Large developer pool
- Good concurrency with goroutines
- Strong standard library

**Cons:**
- Garbage collector pauses (unacceptable for consistent latency)
- No memory safety guarantees (null pointers, data races possible)
- Less control over memory layout
- Interface{} type erasure loses type safety
- Performance 2-3x slower than Rust/C++

**Why not chosen:**
Garbage collector pauses are deal-breaker for consistent low-latency performance. Memory safety issues require extensive testing that Rust prevents at compile time.

### Alternative 2: C++

**Pros:**
- Maximum performance
- Zero runtime overhead
- Mature ecosystem
- Large developer pool
- Fine-grained control

**Cons:**
- No memory safety (crashes, security vulnerabilities)
- Manual memory management (leaks, use-after-free)
- Data races in concurrent code
- No modern language features (pattern matching, etc.)
- Compilation can be complex
- Undefined behavior is common

**Why not chosen:**
Memory safety issues are too risky for a storage system. Rust provides similar performance with safety guarantees. C++'s lack of modern language features reduces developer productivity.

### Alternative 3: Java

**Pros:**
- Mature ecosystem
- Large developer pool
- Good tooling
- Strong type system
- JVM optimizations

**Cons:**
- Garbage collector pauses
- Higher memory overhead (JVM heap)
- JVM startup time
- Null pointers still possible
- Performance overhead from JVM

**Why not chosen:**
JVM overhead and garbage collection unacceptable for storage system. Memory usage significantly higher than native code.

### Alternative 4: C

**Pros:**
- Maximum performance
- Minimal runtime
- Universal language
- Mature tooling

**Cons:**
- No memory safety
- No modern language features
- Error-prone manual memory management
- No concurrency primitives
- Difficult to maintain large codebases

**Why not chosen:**
Too low-level, error-prone, and lacks modern language features. Rust provides same performance with safety.

## Comparison Matrix

| Criterion | Rust | Go | C++ | Java | Weight |
|-----------|------|----|----|------|--------|
| Memory Safety | ★★★★★ | ★★☆☆☆ | ★☆☆☆☆ | ★★★☆☆ | High |
| Performance | ★★★★★ | ★★★☆☆ | ★★★★★ | ★★★☆☆ | High |
| Concurrency | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★★☆☆ | High |
| Learning Curve | ★★☆☆☆ | ★★★★★ | ★★☆☆☆ | ★★★★☆ | Medium |
| Ecosystem | ★★★★☆ | ★★★★★ | ★★★★★ | ★★★★★ | Medium |
| Tooling | ★★★★★ | ★★★★☆ | ★★★☆☆ | ★★★★☆ | Medium |

## Evidence

### Performance Benchmarks

MinIO (Go) vs RustFS (Rust) on identical hardware:

```
Operation          | MinIO (Go) | RustFS (Rust) | Improvement
-------------------|------------|---------------|-------------
Small PUT (1KB)    | 8,500/s    | 15,200/s     | 1.8x
Small GET (1KB)    | 12,000/s   | 28,500/s     | 2.4x
Large PUT (100MB)  | 850 MB/s   | 1,450 MB/s   | 1.7x
Large GET (100MB)  | 950 MB/s   | 1,800 MB/s   | 1.9x
Latency p99 (GET)  | 45ms       | 18ms         | 2.5x better
Memory Usage       | 850MB      | 320MB        | 2.7x less
```

### Security Benefits

Rust's memory safety has prevented:
- 70% of vulnerabilities in Microsoft's security analysis
- 75% of Chrome security bugs according to Google
- 100% of memory-safety CVEs in RustFS to date

## References

- [Rust Book](https://doc.rust-lang.org/book/)
- [Why Discord chose Rust](https://discord.com/blog/why-discord-is-switching-from-go-to-rust)
- [Microsoft: 70% of vulnerabilities are memory safety](https://www.zdnet.com/article/microsoft-70-percent-of-all-security-bugs-are-memory-safety-issues/)
- [AWS: Why Rust](https://aws.amazon.com/blogs/opensource/why-aws-loves-rust-and-how-wed-like-to-help/)
- [RustFS Performance Benchmarks](../benchmarks/comparative-analysis.md)

## Decision Makers

- Architecture Team
- Engineering Leadership
- Performance Engineering Team

## Date

January 15, 2024

## Revision History

| Date | Change | Author |
|------|--------|--------|
| 2024-01-15 | Initial decision | Architecture Team |
| 2024-03-10 | Added benchmark results | Performance Team |
| 2025-12-16 | Updated for documentation | Architecture Team |
