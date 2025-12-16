# Crate Architecture

## Overview

RustFS is organized as a Cargo workspace with 27 specialized crates. This modular architecture enables clear separation of concerns, independent testing, and reusability across the system.

## Workspace Structure

```
rustfs/
├── rustfs/           # Main binary (orchestration)
├── crates/          
│   ├── API Layer
│   │   ├── madmin/              # Admin API
│   │   └── mcp/                 # MCP server (QUIC)
│   ├── Core Services
│   │   ├── iam/                 # Identity and Access Management
│   │   ├── kms/                 # Key Management Service
│   │   ├── policy/              # Policy evaluation engine
│   │   ├── notify/              # Event notification system
│   │   └── appauth/             # Application authentication
│   ├── Storage Layer
│   │   ├── ecstore/             # Erasure coding storage
│   │   ├── filemeta/            # File metadata management
│   │   ├── lock/                # Distributed locking
│   │   ├── rio/                 # Rust I/O abstractions
│   │   └── targets/             # Storage target configurations
│   ├── Security
│   │   ├── crypto/              # Cryptographic operations
│   │   ├── signer/              # Request signing/verification
│   │   └── checksums/           # Data integrity checksums
│   ├── Protocols
│   │   ├── protos/              # Protocol buffer definitions
│   │   ├── s3select-api/        # S3 Select API
│   │   ├── s3select-query/      # S3 Select query engine
│   │   └── zip/                 # ZIP file handling
│   ├── Observability
│   │   ├── obs/                 # Observability (metrics, tracing)
│   │   └── audit/               # Audit logging
│   ├── Utilities
│   │   ├── common/              # Shared utilities
│   │   ├── config/              # Configuration management
│   │   ├── utils/               # General utilities
│   │   ├── ahm/                 # Async hash map
│   │   └── workers/             # Worker thread pools
│   └── Testing
│       └── e2e_test/            # End-to-end tests
```

## Crate Catalog

### API Layer

#### rustfs (Main Binary)
```toml
[package]
name = "rustfs"
version = "0.0.5"
edition = "2024"
```

**Purpose**: Main service binary that orchestrates all components

**Key Responsibilities**:
- HTTP server setup (Axum)
- Service initialization and lifecycle
- Configuration loading
- Request routing to handlers
- Graceful shutdown

**Key Dependencies**:
- `axum` - Web framework
- `tokio` - Async runtime
- `rustfs-iam` - Authentication
- `rustfs-ecstore` - Storage operations
- `rustfs-obs` - Observability

**Public API**:
```rust
// Main entry point
fn main() -> Result<()>;

// Server initialization
async fn start_server(config: Config) -> Result<()>;

// Handler registration
fn register_handlers(router: Router) -> Router;
```

#### rustfs-madmin
**Purpose**: Management and administration API

**Key Responsibilities**:
- User management
- Policy administration
- System configuration
- Health checks
- Diagnostic tools

**Public API**:
```rust
pub struct AdminService;

impl AdminService {
    pub fn new(config: AdminConfig) -> Self;
    pub async fn create_user(&self, user: User) -> Result<()>;
    pub async fn update_policy(&self, policy: Policy) -> Result<()>;
    pub async fn health_check(&self) -> HealthStatus;
}
```

#### rustfs-mcp
**Purpose**: High-performance MCP (MinIO Client Protocol) server using QUIC

**Key Responsibilities**:
- QUIC connection management
- Stream multiplexing
- Efficient bulk transfers
- Connection pooling

**Public API**:
```rust
pub struct McpServer;

impl McpServer {
    pub async fn new(config: McpConfig) -> Result<Self>;
    pub async fn handle_request(&self, req: McpRequest) -> Result<McpResponse>;
}
```

### Core Services

#### rustfs-iam
**Purpose**: Identity and Access Management

**Key Responsibilities**:
- User authentication
- Session management
- Role-based access control
- Token generation/validation
- LDAP/OIDC integration

**Key Types**:
```rust
pub struct User {
    pub id: UserId,
    pub username: String,
    pub roles: Vec<Role>,
    pub access_key: AccessKey,
    pub secret_key: SecretKey,
}

pub struct Role {
    pub id: RoleId,
    pub name: String,
    pub policies: Vec<PolicyArn>,
}

pub trait Authenticator {
    async fn authenticate(&self, credentials: Credentials) -> Result<User>;
    async fn create_session(&self, user: &User) -> Result<Session>;
    async fn validate_token(&self, token: &str) -> Result<Claims>;
}
```

**Public API**:
```rust
pub struct IamService;

impl IamService {
    pub fn new(config: IamConfig) -> Self;
    pub async fn authenticate(&self, creds: Credentials) -> Result<User>;
    pub async fn authorize(&self, user: &User, action: &str, resource: &str) -> Result<bool>;
    pub async fn create_user(&self, user: CreateUserRequest) -> Result<User>;
}
```

#### rustfs-kms
**Purpose**: Key Management Service

**Key Responsibilities**:
- Encryption key generation
- Key rotation and lifecycle
- Envelope encryption
- External KMS integration (AWS KMS, Vault)
- Hardware Security Module (HSM) support

**Key Types**:
```rust
pub struct MasterKey {
    pub id: KeyId,
    pub version: u32,
    pub algorithm: EncryptionAlgorithm,
    pub created_at: DateTime<Utc>,
}

pub struct DataKey {
    pub plaintext: Vec<u8>,
    pub encrypted: Vec<u8>,
    pub master_key_id: KeyId,
}

pub trait KeyManagementService {
    async fn generate_data_key(&self, key_id: &KeyId) -> Result<DataKey>;
    async fn encrypt(&self, plaintext: &[u8], key_id: &KeyId) -> Result<Vec<u8>>;
    async fn decrypt(&self, ciphertext: &[u8], key_id: &KeyId) -> Result<Vec<u8>>;
    async fn rotate_master_key(&self, key_id: &KeyId) -> Result<MasterKey>;
}
```

**Public API**:
```rust
pub struct KmsService;

impl KmsService {
    pub async fn new(config: KmsConfig) -> Result<Self>;
    pub async fn generate_data_key(&self) -> Result<DataKey>;
    pub async fn encrypt(&self, data: &[u8], key_id: &str) -> Result<Vec<u8>>;
    pub async fn decrypt(&self, data: &[u8]) -> Result<Vec<u8>>;
}
```

#### rustfs-policy
**Purpose**: Policy evaluation engine

**Key Responsibilities**:
- Policy parsing (JSON)
- Condition evaluation
- Permission checks
- Policy caching

**Key Types**:
```rust
pub struct Policy {
    pub version: String,
    pub statements: Vec<Statement>,
}

pub struct Statement {
    pub effect: Effect,
    pub actions: Vec<String>,
    pub resources: Vec<String>,
    pub conditions: Option<Conditions>,
}

pub enum Effect {
    Allow,
    Deny,
}

pub trait PolicyEvaluator {
    fn evaluate(&self, request: &AuthRequest) -> Result<Decision>;
}
```

**Public API**:
```rust
pub struct PolicyEngine;

impl PolicyEngine {
    pub fn new() -> Self;
    pub fn load_policy(&mut self, policy: Policy) -> Result<()>;
    pub fn evaluate(&self, user: &User, action: &str, resource: &str) -> Result<Decision>;
}
```

#### rustfs-notify
**Purpose**: Event notification system

**Key Responsibilities**:
- Event generation
- Multi-target fan-out (Kafka, NATS, HTTP)
- Event filtering
- Delivery guarantees
- Retry logic

**Key Types**:
```rust
pub struct Event {
    pub event_type: EventType,
    pub timestamp: DateTime<Utc>,
    pub bucket: String,
    pub object_key: String,
    pub metadata: HashMap<String, String>,
}

pub enum EventType {
    ObjectCreated,
    ObjectDeleted,
    ObjectRestored,
    BucketCreated,
    BucketDeleted,
}

pub trait NotificationTarget {
    async fn send(&self, event: &Event) -> Result<()>;
}
```

**Public API**:
```rust
pub struct NotificationService;

impl NotificationService {
    pub fn new(config: NotifyConfig) -> Self;
    pub async fn publish(&self, event: Event) -> Result<()>;
    pub fn add_target(&mut self, target: Box<dyn NotificationTarget>);
}
```

### Storage Layer

#### rustfs-ecstore
**Purpose**: Erasure coding storage engine

**Key Responsibilities**:
- Reed-Solomon erasure coding
- Data shard distribution
- Shard reconstruction
- Bitrot protection
- Checksumming

**Key Types**:
```rust
pub struct StorageConfig {
    pub data_shards: usize,
    pub parity_shards: usize,
    pub shard_size: usize,
}

pub struct ObjectInfo {
    pub key: String,
    pub size: u64,
    pub etag: String,
    pub shards: Vec<ShardLocation>,
    pub metadata: ObjectMetadata,
}

pub trait StorageEngine {
    async fn put_object(&self, key: &str, data: Bytes, meta: ObjectMetadata) -> Result<ObjectInfo>;
    async fn get_object(&self, key: &str) -> Result<(Bytes, ObjectMetadata)>;
    async fn delete_object(&self, key: &str) -> Result<()>;
    async fn list_objects(&self, prefix: &str) -> Result<Vec<ObjectInfo>>;
}
```

**Public API**:
```rust
pub struct EcStore;

impl EcStore {
    pub async fn new(config: StorageConfig) -> Result<Self>;
    pub async fn write_object(&self, key: &str, data: impl AsyncRead) -> Result<ObjectInfo>;
    pub async fn read_object(&self, key: &str) -> Result<impl AsyncRead>;
    pub async fn heal_shard(&self, shard_id: &ShardId) -> Result<()>;
}
```

#### rustfs-filemeta
**Purpose**: Object metadata management

**Key Responsibilities**:
- Metadata storage and retrieval
- Versioning support
- Indexing
- Metadata caching

**Key Types**:
```rust
pub struct ObjectMetadata {
    pub content_type: String,
    pub content_length: u64,
    pub etag: String,
    pub last_modified: DateTime<Utc>,
    pub version_id: Option<String>,
    pub user_metadata: HashMap<String, String>,
}

pub trait MetadataStore {
    async fn put_metadata(&self, bucket: &str, key: &str, meta: ObjectMetadata) -> Result<()>;
    async fn get_metadata(&self, bucket: &str, key: &str) -> Result<ObjectMetadata>;
    async fn delete_metadata(&self, bucket: &str, key: &str) -> Result<()>;
    async fn list_versions(&self, bucket: &str, key: &str) -> Result<Vec<ObjectMetadata>>;
}
```

#### rustfs-lock
**Purpose**: Distributed locking

**Key Responsibilities**:
- Lock acquisition and release
- Lease management
- Deadlock prevention
- Lock timeouts

**Key Types**:
```rust
pub struct Lock {
    pub id: LockId,
    pub resource: String,
    pub owner: String,
    pub expires_at: DateTime<Utc>,
}

pub trait DistributedLock {
    async fn acquire(&self, resource: &str, timeout: Duration) -> Result<Lock>;
    async fn release(&self, lock: Lock) -> Result<()>;
    async fn renew(&self, lock: &Lock) -> Result<()>;
}
```

### Security

#### rustfs-crypto
**Purpose**: Cryptographic operations

**Key Responsibilities**:
- Encryption/decryption (AES-256-GCM)
- Hashing (SHA-256, Blake3)
- Random number generation
- Key derivation

**Public API**:
```rust
pub fn encrypt(plaintext: &[u8], key: &[u8]) -> Result<Vec<u8>>;
pub fn decrypt(ciphertext: &[u8], key: &[u8]) -> Result<Vec<u8>>;
pub fn hash_sha256(data: &[u8]) -> [u8; 32];
pub fn generate_random_bytes(len: usize) -> Vec<u8>;
```

#### rustfs-signer
**Purpose**: Request signing and verification

**Key Responsibilities**:
- AWS Signature Version 4
- Request signing
- Signature validation

**Public API**:
```rust
pub fn sign_request(request: &HttpRequest, credentials: &Credentials) -> String;
pub fn verify_signature(request: &HttpRequest, signature: &str) -> Result<bool>;
```

### Observability

#### rustfs-obs
**Purpose**: Observability infrastructure

**Key Responsibilities**:
- Prometheus metrics export
- Distributed tracing (OpenTelemetry)
- Structured logging
- Performance profiling

**Public API**:
```rust
pub fn init_metrics() -> PrometheusRegistry;
pub fn init_tracing() -> Result<()>;
pub fn record_metric(name: &str, value: f64, labels: &[(&str, &str)]);
```

#### rustfs-audit
**Purpose**: Audit logging

**Key Responsibilities**:
- Audit event generation
- Structured audit logs
- Multi-target audit delivery

### Utilities

#### rustfs-common
**Purpose**: Shared types and utilities

**Contents**:
- Common error types
- Result type aliases
- Shared constants
- Utility functions

#### rustfs-config
**Purpose**: Configuration management

**Key Responsibilities**:
- Configuration loading (file, env)
- Validation
- Hot reload support

#### rustfs-ahm (Async Hash Map)
**Purpose**: Concurrent hash map for async contexts

**Public API**:
```rust
pub struct AsyncHashMap<K, V>;

impl<K, V> AsyncHashMap<K, V> {
    pub async fn insert(&self, key: K, value: V) -> Option<V>;
    pub async fn get(&self, key: &K) -> Option<V>;
    pub async fn remove(&self, key: &K) -> Option<V>;
}
```

#### rustfs-workers
**Purpose**: Worker thread pool management

**Public API**:
```rust
pub struct WorkerPool;

impl WorkerPool {
    pub fn new(size: usize) -> Self;
    pub async fn spawn<F>(&self, task: F) -> JoinHandle<F::Output>
    where
        F: Future + Send + 'static,
        F::Output: Send + 'static;
}
```

## Crate Dependency Matrix

| Crate | Depends On |
|-------|------------|
| rustfs | iam, kms, policy, ecstore, notify, obs, config |
| madmin | iam, config, obs |
| mcp | ecstore, iam, obs |
| iam | policy, crypto, audit, common |
| kms | crypto, audit, common |
| policy | common, ahm |
| notify | common, workers |
| appauth | iam, policy |
| ecstore | filemeta, lock, kms, crypto, checksums, rio, targets, workers, common |
| filemeta | lock, common, ahm |
| lock | common |
| rio | common |
| crypto | - (no internal deps) |
| signer | crypto, common |
| checksums | crypto, common |
| obs | common |
| audit | notify, common |
| common | - (no internal deps) |
| config | common |
| utils | common |
| ahm | - (tokio only) |
| workers | common |

## Design Patterns

### Repository Pattern
Crates like `filemeta` and `iam` use repository pattern for data access abstraction.

### Strategy Pattern
`kms` uses strategy pattern for different KMS backends (local, AWS, Vault).

### Observer Pattern
`notify` implements observer pattern for event notifications.

### Factory Pattern
Most services use factory methods for initialization.

## Testing Strategy

### Unit Tests
Each crate contains unit tests co-located with source code:
```
crates/iam/src/
├── lib.rs
├── authenticator.rs
└── tests/              # Unit tests
    └── authenticator_test.rs
```

### Integration Tests
Integration tests in `tests/` directory:
```
crates/iam/tests/
├── integration_test.rs
└── ldap_integration_test.rs
```

### End-to-End Tests
Full system tests in `crates/e2e_test/`:
```
crates/e2e_test/src/
├── s3_operations_test.rs
├── multipart_test.rs
└── iam_test.rs
```

## Best Practices

### Crate Interface Design
1. Keep public API minimal and well-documented
2. Use semantic versioning
3. Avoid breaking changes
4. Provide migration guides

### Error Handling
1. Use `thiserror` for error types
2. Bubble errors with `Result`
3. Provide context in error messages
4. No `unwrap()` in production code

### Async Patterns
1. Use Tokio primitives (RwLock, Mutex)
2. Spawn blocking for CPU work
3. Use channels for communication
4. Implement proper cancellation

### Documentation
1. Document all public items
2. Provide examples
3. Maintain README per crate
4. Update docs with changes

---

**Next**: See [Storage Layer Design](04-storage-layer.md) for detailed storage architecture.
