# System Overview

## Introduction

RustFS is a high-performance, distributed object storage system built in Rust, designed to provide S3-compatible storage with enhanced performance, security, and reliability. This document provides a comprehensive overview of the system architecture, design principles, and key components.

## System Vision

RustFS aims to deliver next-generation object storage that combines:
- **Performance**: Leveraging Rust's zero-cost abstractions and memory safety
- **Compatibility**: 100% S3 API compatibility for seamless migration
- **Security**: Zero-trust architecture with comprehensive IAM and KMS
- **Scalability**: Horizontal scaling with distributed architecture
- **Reliability**: Erasure coding for data durability and fault tolerance

## Architecture Principles

### 1. Performance First

**Rationale**: Object storage is I/O intensive and requires maximum throughput with minimal latency.

**Implementation**:
- Rust's zero-cost abstractions eliminate runtime overhead
- Memory safety prevents crashes and data corruption
- Custom memory allocators (Jemalloc/MiMalloc) optimize allocation patterns
- Async I/O with Tokio runtime maximizes concurrency
- Zero-copy operations where possible

### 2. Modular Design

**Rationale**: Large systems require clear separation of concerns for maintainability and testability.

**Implementation**:
- 27 specialized crates with well-defined boundaries
- Minimal inter-crate dependencies
- Clear public API surfaces
- Domain-driven design principles
- Independent testing and versioning

### 3. Security by Default

**Rationale**: Storage systems handle sensitive data and require robust security.

**Implementation**:
- Zero-trust architecture (default deny)
- Fine-grained IAM policies
- Envelope encryption with KMS
- Audit logging for all operations
- TLS/HTTPS for all communications

### 4. Distributed Architecture

**Rationale**: Modern workloads require horizontal scalability and fault tolerance.

**Implementation**:
- Stateless service design
- Distributed locking for coordination
- Erasure coding for data redundancy
- Health checks and automatic recovery
- No single point of failure

### 5. Observability Built-in

**Rationale**: Production systems require comprehensive monitoring and debugging capabilities.

**Implementation**:
- Prometheus metrics for all components
- Distributed tracing with Jaeger
- Structured logging with correlation IDs
- Health check endpoints
- Performance profiling support

## System Components

### API Layer

#### S3 API (Axum)
- **Purpose**: S3-compatible REST API
- **Technology**: Axum web framework
- **Features**: 
  - Standard S3 operations (GET, PUT, DELETE, LIST)
  - Multipart upload support
  - Versioning and lifecycle management
  - Server-side encryption
  - Access control

#### Admin API (madmin)
- **Purpose**: Management and administration interface
- **Features**:
  - System configuration
  - User and policy management
  - Health and metrics endpoints
  - Diagnostic tools

#### MCP Server
- **Purpose**: High-performance QUIC-based operations
- **Features**:
  - Low-latency object operations
  - Efficient bulk transfers
  - Connection pooling
  - Stream multiplexing

### Core Services Layer

#### Identity and Access Management (IAM)
- **Crate**: `rustfs-iam`
- **Responsibilities**:
  - User authentication
  - Role-based access control
  - Policy evaluation
  - Token management
  - Session handling

#### Key Management Service (KMS)
- **Crate**: `rustfs-kms`
- **Responsibilities**:
  - Encryption key generation
  - Key rotation and lifecycle
  - Envelope encryption
  - External KMS integration
  - Hardware security module support

#### Policy Engine
- **Crate**: `rustfs-policy`
- **Responsibilities**:
  - Policy parsing and validation
  - Access decision evaluation
  - Condition evaluation
  - Policy caching

#### Notification System
- **Crate**: `rustfs-notify`
- **Responsibilities**:
  - Event generation
  - Multi-target fan-out
  - Filter and routing
  - Delivery guarantees

### Storage Layer

#### Erasure Coding Store (ecstore)
- **Crate**: `rustfs-ecstore`
- **Responsibilities**:
  - Erasure coding algorithm
  - Data shard distribution
  - Parity calculation
  - Reconstruction on read
  - Bitrot protection

#### File Metadata
- **Crate**: `rustfs-filemeta`
- **Responsibilities**:
  - Object metadata storage
  - Version tracking
  - Indexing and search
  - Metadata caching

#### Distributed Lock
- **Crate**: `rustfs-lock`
- **Responsibilities**:
  - Distributed coordination
  - Lease management
  - Deadlock prevention
  - Lock timeouts

### Cross-Cutting Concerns

#### Observability (obs)
- Metrics collection and export
- Distributed tracing
- Structured logging
- Performance profiling

#### Cryptography (crypto)
- Encryption primitives
- Hash functions
- Signature verification
- Random number generation

#### Configuration (config)
- Configuration loading
- Environment variables
- Hot reload support
- Validation

## Data Flow

### Write Path

```
Client Request
    ↓
S3 API Handler (Axum)
    ↓
IAM Authentication
    ↓
Policy Authorization
    ↓
KMS Encryption (optional)
    ↓
Erasure Coding (ecstore)
    ↓
Shard Distribution
    ↓
Storage Backend
    ↓
Metadata Update (filemeta)
    ↓
Event Notification (notify)
    ↓
Response to Client
```

### Read Path

```
Client Request
    ↓
S3 API Handler (Axum)
    ↓
IAM Authentication
    ↓
Policy Authorization
    ↓
Metadata Lookup (filemeta)
    ↓
Shard Collection (ecstore)
    ↓
Data Reconstruction
    ↓
KMS Decryption (if encrypted)
    ↓
Response to Client
```

## Deployment Models

### Single Node Mode
- All components on one server
- Suitable for development and small deployments
- Full feature parity with distributed mode
- Easy to deploy and manage

### Distributed Mode
- Multiple nodes for horizontal scaling
- Shared storage or distributed storage
- Load balancing across nodes
- High availability and fault tolerance

### Kubernetes Deployment
- Helm charts provided
- StatefulSet for persistent storage
- Service discovery
- Horizontal pod autoscaling
- Resource limits and requests

## Technology Stack

### Core Technologies
- **Language**: Rust (Edition 2024, version 1.85)
- **Async Runtime**: Tokio
- **Web Framework**: Axum
- **Serialization**: Serde, quick-xml
- **Testing**: Cargo test, criterion (benchmarks)

### Memory Management
- **Linux x86_64**: Jemalloc
- **Other platforms**: MiMalloc
- Custom allocator selection for optimal performance

### Networking
- **HTTP/HTTPS**: Hyper + Axum
- **QUIC**: Custom MCP implementation
- **TLS**: rustls

### Storage
- **File System**: rio abstraction layer
- **Erasure Coding**: Custom implementation
- **Metadata**: Embedded database (consideration)

### Observability
- **Metrics**: Prometheus + custom exporters
- **Tracing**: OpenTelemetry + Jaeger
- **Logging**: tracing + tracing-subscriber

## Performance Characteristics

### Throughput
- **Small Objects (<1MB)**: 10,000+ ops/sec per node
- **Large Objects (>100MB)**: Line-rate throughput
- **Concurrent Requests**: Scales with CPU cores

### Latency
- **Metadata Operations**: < 10ms p99
- **Small Object GET**: < 50ms p99
- **Small Object PUT**: < 100ms p99

### Scalability
- **Horizontal**: Linear scaling up to tested limits
- **Vertical**: Efficient CPU and memory utilization
- **Storage**: Petabyte-scale capability

### Resource Usage
- **Memory**: ~100MB baseline + working set
- **CPU**: Efficient utilization of all cores
- **I/O**: Optimized for NVMe and HDD

## Quality Attributes

### Reliability
- **Data Durability**: 11 nines (99.999999999%)
- **Availability**: 99.99% with proper deployment
- **MTTR**: < 5 minutes with automation
- **Bitrot Protection**: Continuous verification

### Security
- **Authentication**: Multiple methods (IAM, LDAP, OIDC)
- **Authorization**: Fine-grained policies
- **Encryption**: At-rest and in-transit
- **Audit**: Complete operation history

### Maintainability
- **Code Quality**: Clippy, rustfmt enforced
- **Test Coverage**: > 80% target
- **Documentation**: Inline and external
- **Modularity**: Clear crate boundaries

### Operability
- **Deployment**: Docker, Kubernetes, bare metal
- **Monitoring**: Comprehensive metrics
- **Debugging**: Distributed tracing
- **Configuration**: Environment variables, files

## Design Trade-offs

### Performance vs. Flexibility
**Decision**: Prioritize performance with configurable flexibility
**Rationale**: Storage workloads are performance-critical, but different use cases require different configurations.

### Consistency vs. Availability
**Decision**: Configurable consistency levels
**Rationale**: Different workloads have different requirements; allow users to choose based on their needs.

### Simplicity vs. Features
**Decision**: Core features first, advanced features incrementally
**Rationale**: Maintain system simplicity while adding well-tested features over time.

### Memory Safety vs. Performance
**Decision**: Memory safety without compromise
**Rationale**: Rust provides both; any performance concerns addressed through profiling and optimization.

## Future Architecture Direction

### Short Term (3-6 months)
- Complete distributed mode testing
- Enhanced lifecycle management
- Improved replication
- Performance optimizations

### Medium Term (6-12 months)
- Multi-tenancy support
- Advanced query capabilities (S3 Select)
- Tiered storage
- Global namespace

### Long Term (1-2 years)
- Geo-replication
- Machine learning integration
- Advanced analytics
- Cloud-native optimizations

## Conclusion

RustFS architecture is designed for high-performance, secure, and scalable object storage. The modular design allows for independent development and testing, while the distributed architecture enables horizontal scaling. The use of Rust provides memory safety and performance, making RustFS a compelling choice for modern data workloads.

---

**Next Steps**: Review [C4 Architecture Diagrams](02-c4-architecture.md) for detailed component views.
