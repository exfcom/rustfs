# RustFS Project Overview

## Project Purpose
RustFS is a high-performance, distributed object storage system built in Rust. It serves as a Rust alternative to MinIO with the following key features:
- **Full S3 Compatibility**: Seamless integration with existing S3-compatible applications and tools
- **High Performance**: Built with Rust to ensure maximum speed and resource efficiency
- **Distributed Architecture**: Scalable and fault-tolerant design suitable for large-scale deployments
- **Data Lake Support**: Optimized for high-throughput big data and AI workloads
- **Open Source**: Licensed under Apache 2.0, avoiding AGPL restrictions
- **Memory Safety**: Rust's built-in memory safety guarantees

## Feature Status
- ✅ Available: S3 Core Features, Upload/Download, Versioning, Logging, Event Notifications, K8s Helm Charts, Bitrot Protection, Single Node Mode
- ⚠️ Partial Support: Bucket Replication
- 🚧 Under Testing: Lifecycle Management, Distributed Mode, OPA (Open Policy Agent)

## Technology Stack
- **Programming Language**: Rust (edition 2024, rust-version 1.85)
- **License**: Apache 2.0
- **Async Runtime**: Tokio, async-trait
- **Web Framework**: Axum
- **Memory Allocators**: 
  - Linux x86_64 gnu: Jemalloc
  - Other platforms: MiMalloc
- **Build Tools**: Cargo, Make
- **Containerization**: Docker, Docker Compose
- **Testing**: cargo-nextest (preferred), cargo test

## Performance Characteristics
- Optimized for concurrent operations
- Zero-copy data transfer paths
- Efficient erasure coding implementation
- Memory-safe by design (no GC pauses)
