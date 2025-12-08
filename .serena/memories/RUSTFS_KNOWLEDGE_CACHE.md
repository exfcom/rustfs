# RustFS Knowledge Cache

> High-performance S3-compatible distributed object storage system written in Rust

---

## 1. Project Overview

### Identity
- **Name**: RustFS
- **Repository**: `/home/tuanna47/workspace/FSO/Proposal_Master/99_References/99_Ref_Opensources/rustfs`
- **License**: Apache 2.0
- **Language**: Rust 1.85, Edition 2024
- **Type**: Distributed Object Storage
- **S3 Compatibility**: Full S3-compatible APIs

### Key Features
- S3-compatible API (GET, PUT, DELETE, multipart, etc.)
- Erasure coding for data protection (Reed-Solomon SIMD)
- Server-side encryption (SSE-S3, SSE-KMS, SSE-C)
- Distributed locking system
- Auto-heal and background scanner
- Concurrent GetObject optimization with adaptive I/O
- Multi-tenant IAM system
- Event notifications (Webhook, MQTT)
- S3 Select support via DataFusion

---

## 2. Architecture

### Workspace Structure (27+ Crates)

```
rustfs/
├── rustfs/src/           # Main binary
│   ├── main.rs           # Entry point, service initialization
│   ├── storage/          # S3 operations implementation
│   │   ├── mod.rs        # Storage module exports
│   │   └── concurrency.rs # Concurrent GetObject optimization
│   ├── server/           # HTTP server, audit, events
│   ├── admin/            # Admin handlers (KMS, IAM, users, policies)
│   ├── auth.rs           # Authentication
│   ├── config/           # Configuration
│   └── error.rs          # Error types
├── crates/
│   ├── ecstore/          # Core: Erasure coding storage
│   ├── kms/              # Key Management Service
│   ├── iam/              # Identity & Access Management
│   ├── lock/             # Distributed locking
│   ├── ahm/              # Auto-heal manager & scanner
│   ├── rio/              # Rust I/O utilities
│   ├── notify/           # Event notifications
│   ├── audit/            # Audit logging
│   ├── crypto/           # Encryption/JWT
│   ├── policy/           # Policy management
│   ├── mcp/              # MCP server for AI tools
│   ├── workers/          # Worker pools
│   ├── common/           # Shared utilities
│   └── ... (14+ more)
├── docs/                 # Documentation
├── deploy/               # Deployment configs
└── scripts/              # Development scripts
```

### High-Level Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                     S3 API Request                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  axum HTTP Server + s3s Protocol Handler                    │
│  - Request parsing, routing, authentication                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  IAM + Policy Layer                                         │
│  - Credential validation, bucket/object ACLs                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Storage Operations (rustfs/src/storage/)                   │
│  - ConcurrencyManager for adaptive I/O                      │
│  - HotObjectCache for frequently accessed objects           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  ECStore (crates/ecstore/)                                  │
│  - Erasure coding (Reed-Solomon SIMD)                       │
│  - Disk management, bucket operations                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Lock Manager + KMS                                         │
│  - Object-level locking                                     │
│  - Transparent encryption/decryption                        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Physical Storage (Disks/Volumes)                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Key Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `axum` | 0.8.7 | HTTP framework for S3 API server |
| `tokio` | 1.48.0 | Async runtime |
| `s3s` | 0.12.0-rc.4 | S3 protocol implementation |
| `datafusion` | 51.0.0 | S3 Select query processing |
| `moka` | 0.12.11 | Lock-free object caching |
| `reed-solomon-simd` | 3.1.0 | Erasure coding |
| `tikv-jemallocator` | 0.6 | Memory allocator (Linux GNU) |
| `rustls` | 0.23.35 | TLS implementation |
| `hyper` | 1.8.1 | HTTP client/server |
| `sqlx` | 0.8.5 | Async SQL (PostgreSQL) |
| `serde` | 1.0 | Serialization |
| `tracing` | 0.1 | Logging/tracing |

---

## 4. Core Storage Layer (`crates/ecstore/`)

### ECStore (`store.rs`)

The central storage implementation managing erasure-coded data across disks.

```rust
pub struct ECStore {
    disks: Vec<Arc<Disk>>,
    erasure: ErasureConfig,
    // ... additional fields
}
```

**Key Responsibilities:**
- Disk pool management
- Erasure set distribution
- Object PUT/GET/DELETE operations
- Health monitoring

### StorageAPI Trait (`store_api.rs`)

```rust
#[trait_variant::make(Send)]
pub trait StorageAPI: Send + Sync + 'static {
    type PutObjReader: PutObjReader;
    type GetObjReader: GetObjReader;
    
    async fn get_object_reader(&self, opts: &GetObjectOptions) -> Result<ObjectInfo, Error>;
    async fn put_object(&self, obj_reader: &mut Self::PutObjReader) -> Result<ObjectInfo, Error>;
    async fn delete_object(&self, opts: &DeleteObjectOptions) -> Result<ObjectInfo, Error>;
    async fn list_objects(&self, opts: &ListObjectOptions) -> Result<ListObjectsInfo, Error>;
    // ... additional methods
}
```

### Erasure Coding (`erasure_coding/`)

**Configuration:**
- Uses Reed-Solomon SIMD for high-performance encoding
- Default: 4 data shards + 2 parity shards (configurable)
- Automatically recovers from disk failures

```rust
pub struct Erasure {
    pub encoder: Arc<(ReedSolomonEncoder, ReedSolomonDecoder)>,
    pub shards_count: usize,
    pub data_shards_count: usize,
    pub parity_shards_count: usize,
    pub block_size: usize,
}
```

**Operations:**
- `encode()` - Split data into erasure-coded shards
- `decode()` - Reconstruct data from available shards
- Supports parallel encoding/decoding

---

## 5. Concurrency Optimization (`rustfs/src/storage/concurrency.rs`)

### Adaptive I/O Strategy

The system dynamically adjusts I/O parameters based on disk permit wait times:

```rust
pub enum IoLoadLevel {
    Low,      // <20ms wait → aggressive buffering
    Medium,   // 20-100ms wait → balanced
    High,     // 100-500ms wait → conservative
    Critical, // >500ms wait → minimal buffering
}

pub struct IoStrategy {
    pub read_ahead_size: usize,        // Prefetch size
    pub buffer_pool_size: usize,       // Buffer allocation
    pub max_concurrent_reads: usize,   // Parallel read limit
    pub chunk_size: usize,             // I/O chunk size
}
```

### Hot Object Cache

Uses `moka` for lock-free caching of frequently accessed objects:

```rust
pub struct HotObjectCache {
    cache: moka::sync::Cache<String, CachedGetObject>,
}

pub struct CachedGetObject {
    pub object_info: ObjectInfo,
    pub s3_output: s3s::dto::GetObjectOutput,
}
```

**Cache Strategy:**
- Cache key: `{bucket}/{object_key}`
- TTL-based eviction
- Size-limited cache
- Thread-safe access

### ConcurrencyManager

Central coordinator for concurrent operations:

```rust
pub struct ConcurrencyManager {
    active_requests: AtomicU64,
    io_strategy: RwLock<IoStrategy>,
    hot_cache: HotObjectCache,
    // ... metrics and config
}
```

**Features:**
- Request tracking via RAII guards (`GetObjectGuard`)
- Adaptive strategy adjustment
- Cache hit/miss metrics
- Load-based throttling

---

## 6. Key Management Service (`crates/kms/`)

### Encryption Modes

| Mode | Description |
|------|-------------|
| SSE-S3 | Server-managed keys (default) |
| SSE-KMS | Customer-managed master key |
| SSE-C | Customer-provided encryption key |

### Key Hierarchy (Three Layers)

```
┌─────────────────────────────────────┐
│  Master Key (MEK)                   │
│  - Stored in KMS backend            │
│  - Encrypts DEKs                    │
└─────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  Data Encryption Key (DEK)          │
│  - Per-bucket or per-object         │
│  - Encrypted by MEK                 │
└─────────────────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  Object Data                        │
│  - AES-256-GCM encrypted            │
└─────────────────────────────────────┘
```

### Backends

```rust
pub enum KmsBackend {
    Local(LocalKms),      // File-based key storage
    Vault(VaultKms),      // HashiCorp Vault integration
}
```

### ObjectEncryptionService

Provides transparent encryption/decryption:

```rust
pub struct ObjectEncryptionService {
    kms: Arc<Kms>,
    default_encryption: EncryptionType,
}

impl ObjectEncryptionService {
    pub async fn encrypt_object(&self, data: &[u8], opts: &EncryptOptions) -> Result<Vec<u8>>;
    pub async fn decrypt_object(&self, data: &[u8], opts: &DecryptOptions) -> Result<Vec<u8>>;
}
```

---

## 7. Identity & Access Management (`crates/iam/`)

### System Structure

```rust
pub struct IamSys {
    store: Arc<dyn ObjectStore>,
    cache: IamCache,
    // ... additional fields
}

// Global singleton
pub static IAM_SYS: OnceLock<Arc<IamSys>> = OnceLock::new();
```

### Features
- User/group management
- Service accounts
- Policy attachment
- Credential rotation
- Access key management

### Key Types
- `IamSys` - Main IAM system
- `IamCache` - In-memory cache for policies/users
- `ObjectStore` - Persistence layer

---

## 8. Distributed Locking (`crates/lock/`)

### Lock Managers

```rust
pub trait LockManager: Send + Sync + 'static {
    async fn lock(&self, resource: &str, mode: LockMode) -> Result<LockGuard>;
    async fn unlock(&self, guard: LockGuard) -> Result<()>;
}

pub enum LockMode {
    Read,   // Shared lock
    Write,  // Exclusive lock
}
```

### Implementations

| Manager | Use Case |
|---------|----------|
| `FastObjectLockManager` | High-performance distributed locking |
| `DisabledLockManager` | No-op for single-node deployments |
| `GlobalLockManager` | Runtime selection based on config |

### FastLock System

```rust
pub struct FastObjectLockManager {
    locks: DashMap<String, LockState>,
    // ... additional fields
}
```

**Features:**
- Lock-free concurrent access
- Deadlock detection
- Timeout-based acquisition
- Automatic cleanup

---

## 9. Auto-Heal & Scanner (`crates/ahm/`)

### Components

```rust
pub struct HealManager {
    scanner: Scanner,
    processor: HealChannelProcessor,
    // ... additional fields
}

pub struct Scanner {
    // Background data scanning
}

pub struct HealChannelProcessor {
    // Async heal request processing
}
```

### Scanner Operations
- **Background scanning**: Continuous data integrity verification
- **Bitrot detection**: Identifies corrupted data blocks
- **Heal triggering**: Initiates repair when corruption detected

### Heal Process
1. Detect missing/corrupted shard
2. Read available shards
3. Reconstruct via erasure decoding
4. Write repaired shard
5. Verify integrity

### Environment Control

```bash
RUSTFS_ENABLE_SCANNER=true   # Enable background scanning
RUSTFS_ENABLE_HEAL=true      # Enable auto-heal
```

---

## 10. I/O Utilities (`crates/rio/`)

### Reader Chain Pattern

```rust
// Composable readers for streaming I/O
pub use hash_reader::HashReader;      // MD5/SHA256 on-the-fly
pub use etag_reader::EtagReader;      // S3 ETag calculation
pub use compress_reader::CompressReader;
pub use encrypt_reader::EncryptReader;
pub use decrypt_reader::DecryptReader;
pub use limit_reader::LimitReader;
```

### Usage Pattern

```rust
// Chain readers for complex I/O pipelines
let reader = LimitReader::new(
    EncryptReader::new(
        HashReader::new(source_reader),
        encryption_key
    ),
    max_size
);
```

---

## 11. Event Notification (`crates/notify/`)

### Targets

```rust
pub enum NotificationTarget {
    Webhook(WebhookConfig),
    Mqtt(MqttConfig),
}
```

### Event Types
- `s3:ObjectCreated:*`
- `s3:ObjectRemoved:*`
- `s3:ObjectRestore:*`
- Bucket lifecycle events

### Features
- Event persistence
- Retry on failure
- Filtering by prefix/suffix
- Batch delivery

---

## 12. Audit Logging (`crates/audit/`)

### Structure

```rust
pub struct AuditLogger {
    targets: Vec<AuditTarget>,
    // ... additional fields
}
```

### Features
- Multi-target fan-out (file, syslog, HTTP)
- Hot reload configuration
- Structured logging (JSON)
- Request/response correlation

---

## 13. MCP Server (`crates/mcp/`)

### Purpose
Model Context Protocol server for AI tool integration.

```rust
pub struct RustfsMcpServer {
    s3_client: S3Client,
    // ... additional fields
}
```

### Exposed Operations
- Bucket operations (list, create, delete)
- Object operations (get, put, delete, list)
- Metadata queries

---

## 14. Build System

### Makefile Targets

```makefile
# Code Quality
make fmt              # Format code
make fmt-check        # Check formatting
make clippy           # Run linter
make check            # Compile check
make test             # Run tests
make pre-commit       # All quality checks

# Native Builds
make build            # Production build (via build-rustfs.sh)
make build-dev        # Development build
make build-musl       # Static musl build (x86_64)
make build-gnu        # Dynamic glibc build (x86_64)
make build-musl-arm64 # Static musl build (ARM64)
make build-gnu-arm64  # Dynamic glibc build (ARM64)

# Docker Builds
make docker-buildx       # Multi-arch production image
make docker-buildx-push  # Build and push to registry
make docker-dev          # Development image
make docker-dev-local    # Local development image
```

### Cargo Commands

```bash
# Build
cargo build --release -p rustfs

# Test
cargo test --workspace --exclude e2e_test
cargo test --package e2e_test  # E2E tests (requires running server)

# Check
cargo clippy --workspace --all-targets
cargo fmt --all --check
```

### Build Scripts
- `build-rustfs.sh` - Cross-platform release builds
- `docker-buildx.sh` - Multi-architecture Docker builds

---

## 15. Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `RUSTFS_ENABLE_SCANNER` | `true` | Enable background data scanner |
| `RUSTFS_ENABLE_HEAL` | `true` | Enable auto-heal functionality |
| `RUSTFS_ENABLE_LOCKS` | `true` | Enable distributed lock system |
| `RUSTFS_ROOT_USER` | - | Root user access key |
| `RUSTFS_ROOT_PASSWORD` | - | Root user secret key |
| `RUSTFS_VOLUMES` | - | Comma-separated disk paths |
| `RUSTFS_CONFIG` | - | Config file path |
| `RUSTFS_ADDRESS` | `0.0.0.0:9000` | API listen address |
| `RUSTFS_CONSOLE_ADDRESS` | `0.0.0.0:9001` | Console listen address |

---

## 16. Code Style & Best Practices

### Safety Rules (from `AGENTS.md`)

```rust
// ❌ NEVER use unwrap/expect outside tests
let value = map.get("key").unwrap();  // BAD

// ✅ Always handle errors properly
let value = map.get("key").ok_or(Error::KeyNotFound)?;  // GOOD

// ❌ No unsafe code
unsafe { /* anything */ }  // PROHIBITED

// ✅ Use safe abstractions
```

### Workspace Settings

```toml
# Cargo.toml [workspace.lints.rust]
unsafe_code = "deny"
```

### Testing Guidelines

```rust
// Unit tests in same file
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_feature() {
        // Test implementation
    }
}

// Integration tests in tests/ directory
// E2E tests in e2e_test crate
```

### Pre-commit Checklist
1. `cargo fmt --all`
2. `cargo clippy --workspace --all-targets`
3. `cargo test --workspace`
4. No compiler warnings

---

## 17. API Quick Reference

### S3 Operations (via `crates/ecstore/src/store_api.rs`)

```rust
// Object Operations
get_object_reader(&GetObjectOptions) -> Result<ObjectInfo>
put_object(&mut PutObjReader) -> Result<ObjectInfo>
delete_object(&DeleteObjectOptions) -> Result<ObjectInfo>
list_objects(&ListObjectOptions) -> Result<ListObjectsInfo>
copy_object(&CopyObjectOptions) -> Result<ObjectInfo>
head_object(&HeadObjectOptions) -> Result<ObjectInfo>

// Multipart Upload
new_multipart_upload(&CreateMultipartOptions) -> Result<InitiateMultipartUploadResult>
put_object_part(&PutObjectPartOptions, reader) -> Result<PartInfo>
complete_multipart_upload(&CompleteMultipartOptions) -> Result<ObjectInfo>
abort_multipart_upload(&AbortMultipartOptions) -> Result<()>

// Bucket Operations
list_buckets() -> Result<Vec<BucketInfo>>
get_bucket_info(&GetBucketInfoOptions) -> Result<BucketInfo>
make_bucket(&MakeBucketOptions) -> Result<()>
delete_bucket(&DeleteBucketOptions) -> Result<()>
```

### Concurrency Manager (`rustfs/src/storage/concurrency.rs`)

```rust
// Request tracking
acquire_get_request() -> GetObjectGuard
get_active_requests() -> u64
get_current_strategy() -> IoStrategy

// Caching
cache_get_object(key: &str, object: CachedGetObject)
get_cached_object(key: &str) -> Option<CachedGetObject>
```

### KMS (`crates/kms/`)

```rust
// Key operations
create_key(&CreateKeyOptions) -> Result<KeyMetadata>
get_key(key_id: &str) -> Result<KeyMetadata>
delete_key(key_id: &str) -> Result<()>

// Encryption
encrypt(&EncryptRequest) -> Result<EncryptResponse>
decrypt(&DecryptRequest) -> Result<DecryptResponse>
generate_data_key(&GenerateDataKeyRequest) -> Result<GenerateDataKeyResponse>
```

### Lock Manager (`crates/lock/`)

```rust
// Locking
lock(resource: &str, mode: LockMode) -> Result<LockGuard>
try_lock(resource: &str, mode: LockMode, timeout: Duration) -> Result<LockGuard>
unlock(guard: LockGuard) -> Result<()>
```

---

## 18. Extension Points

### Adding New Storage Backend
1. Implement `StorageAPI` trait in `crates/ecstore/src/store_api.rs`
2. Register in storage factory
3. Add configuration options

### Adding New KMS Backend
1. Implement backend in `crates/kms/src/`
2. Add to `KmsBackend` enum
3. Update configuration parsing

### Adding New Notification Target
1. Implement target in `crates/notify/src/`
2. Add to `NotificationTarget` enum
3. Update notification dispatcher

### Adding New Admin Handler
1. Create handler in `rustfs/src/admin/`
2. Register route in admin router
3. Add authentication/authorization

---

## 19. Common Patterns

### Deferred Cleanup (`crates/common/`)

```rust
use common::defer;

fn operation() -> Result<()> {
    let resource = acquire_resource()?;
    defer!(release_resource(resource));
    
    // Work with resource
    // Automatically released on scope exit
    Ok(())
}
```

### Global Service Access

```rust
use iam::IAM_SYS;

async fn check_permission() -> Result<()> {
    let iam = IAM_SYS.get().ok_or(Error::IamNotInitialized)?;
    iam.check_access(/* ... */).await
}
```

### Error Handling

```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum StorageError {
    #[error("Object not found: {0}")]
    NotFound(String),
    
    #[error("Permission denied")]
    PermissionDenied,
    
    #[error("Internal error: {0}")]
    Internal(#[from] anyhow::Error),
}
```

---

## 20. Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Build fails on Linux | Missing jemalloc | Install `libjemalloc-dev` |
| Tests fail | Missing env vars | Set `RUSTFS_ROOT_USER/PASSWORD` |
| E2E tests fail | Server not running | Start server before E2E tests |
| Lock timeout | High contention | Increase lock timeout or optimize |
| Heal not working | Scanner disabled | Set `RUSTFS_ENABLE_SCANNER=true` |

### Debug Logging

```bash
RUST_LOG=debug cargo run -p rustfs
RUST_LOG=rustfs=debug,ecstore=trace cargo run -p rustfs
```

---

## Version History

| Version | Date | Description |
|---------|------|-------------|
| 1.0 | 2025-01-XX | Initial knowledge cache |

---

*Generated from RustFS codebase analysis. For latest information, refer to source code and official documentation.*
