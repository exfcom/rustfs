# Design Patterns in RustFS

## Overview

This document catalogs the design patterns used throughout the RustFS codebase, providing examples and explaining the rationale for each pattern.

## Table of Contents

1. [Architectural Patterns](#architectural-patterns)
2. [Structural Patterns](#structural-patterns)
3. [Behavioral Patterns](#behavioral-patterns)
4. [Concurrency Patterns](#concurrency-patterns)
5. [Error Handling Patterns](#error-handling-patterns)
6. [Rust-Specific Patterns](#rust-specific-patterns)

## Architectural Patterns

### 1. Layered Architecture

**Description**: System organized in layers with clear dependencies flowing downward.

**Implementation in RustFS**:
```
API Layer (S3 API, Admin API, MCP)
    ↓
Core Services Layer (IAM, KMS, Policy, Notify)
    ↓
Storage Layer (ecstore, filemeta, lock)
    ↓
Infrastructure Layer (rio, crypto, obs)
```

**Benefits**:
- Clear separation of concerns
- Each layer can be tested independently
- Easy to understand and maintain

**Example**:
```rust
// API Layer depends on Core Services
pub struct S3ApiHandler {
    iam_service: Arc<IamService>,
    storage_engine: Arc<EcStore>,
    kms_service: Arc<KmsService>,
}

// Core Services depend on Storage Layer
pub struct EcStore {
    metadata_store: Arc<MetadataStore>,
    lock_service: Arc<LockService>,
    io_layer: Arc<RioLayer>,
}
```

### 2. Microservices / Modular Monolith

**Description**: System decomposed into independently deployable services (crates).

**Implementation**: 27 specialized crates with well-defined boundaries

**Benefits**:
- Independent development and testing
- Clear ownership
- Reusability

**Example**:
```rust
// Each crate has a clear public API
// crates/iam/src/lib.rs
pub struct IamService;

impl IamService {
    pub async fn authenticate(&self, creds: Credentials) -> Result<User>;
    pub async fn authorize(&self, user: &User, action: &str) -> Result<bool>;
}
```

### 3. Event-Driven Architecture

**Description**: Components communicate through events rather than direct calls.

**Implementation**: `notify` crate provides event bus

**Benefits**:
- Loose coupling
- Easy to add new event consumers
- Asynchronous processing

**Example**:
```rust
// Event definition
pub enum StorageEvent {
    ObjectCreated { bucket: String, key: String, metadata: Metadata },
    ObjectDeleted { bucket: String, key: String },
    BucketCreated { bucket: String },
}

// Event publisher
pub struct NotificationService {
    targets: Vec<Box<dyn NotificationTarget>>,
}

impl NotificationService {
    pub async fn publish(&self, event: StorageEvent) {
        for target in &self.targets {
            target.send(&event).await;
        }
    }
}

// Event subscriber
impl NotificationTarget for KafkaTarget {
    async fn send(&self, event: &StorageEvent) -> Result<()> {
        // Serialize and send to Kafka
    }
}
```

## Structural Patterns

### 1. Repository Pattern

**Description**: Abstraction layer for data access.

**Implementation**: Metadata and IAM storage use repository pattern

**Benefits**:
- Separates business logic from data access
- Easy to swap implementations (in-memory, file-based, database)
- Testable with mock repositories

**Example**:
```rust
// Repository trait
pub trait MetadataRepository: Send + Sync {
    async fn get(&self, bucket: &str, key: &str) -> Result<Option<Metadata>>;
    async fn put(&self, bucket: &str, key: &str, metadata: Metadata) -> Result<()>;
    async fn delete(&self, bucket: &str, key: &str) -> Result<()>;
    async fn list(&self, bucket: &str, prefix: &str) -> Result<Vec<Metadata>>;
}

// Implementation
pub struct FileMetadataRepository {
    base_path: PathBuf,
}

impl MetadataRepository for FileMetadataRepository {
    async fn get(&self, bucket: &str, key: &str) -> Result<Option<Metadata>> {
        let path = self.base_path.join(bucket).join(key).with_extension("meta");
        if !path.exists() {
            return Ok(None);
        }
        let data = tokio::fs::read(&path).await?;
        let metadata = serde_json::from_slice(&data)?;
        Ok(Some(metadata))
    }
    // ... other methods
}
```

### 2. Builder Pattern

**Description**: Construct complex objects step by step.

**Implementation**: Configuration builders, request builders

**Benefits**:
- Fluent API
- Optional parameters
- Immutable objects

**Example**:
```rust
pub struct StorageConfig {
    data_shards: usize,
    parity_shards: usize,
    shard_size: usize,
    encryption_enabled: bool,
    compression_enabled: bool,
}

pub struct StorageConfigBuilder {
    data_shards: usize,
    parity_shards: usize,
    shard_size: usize,
    encryption_enabled: bool,
    compression_enabled: bool,
}

impl StorageConfigBuilder {
    pub fn new() -> Self {
        Self {
            data_shards: 10,
            parity_shards: 4,
            shard_size: 64 * 1024 * 1024,
            encryption_enabled: true,
            compression_enabled: false,
        }
    }
    
    pub fn data_shards(mut self, n: usize) -> Self {
        self.data_shards = n;
        self
    }
    
    pub fn parity_shards(mut self, n: usize) -> Self {
        self.parity_shards = n;
        self
    }
    
    pub fn encryption_enabled(mut self, enabled: bool) -> Self {
        self.encryption_enabled = enabled;
        self
    }
    
    pub fn build(self) -> Result<StorageConfig> {
        if self.data_shards == 0 {
            return Err(Error::InvalidConfig("data_shards must be > 0"));
        }
        Ok(StorageConfig {
            data_shards: self.data_shards,
            parity_shards: self.parity_shards,
            shard_size: self.shard_size,
            encryption_enabled: self.encryption_enabled,
            compression_enabled: self.compression_enabled,
        })
    }
}

// Usage
let config = StorageConfigBuilder::new()
    .data_shards(16)
    .parity_shards(2)
    .encryption_enabled(true)
    .build()?;
```

### 3. Facade Pattern

**Description**: Simplified interface to complex subsystem.

**Implementation**: `EcStore` provides facade to storage subsystem

**Benefits**:
- Simplified API for clients
- Hides complexity
- Reduces coupling

**Example**:
```rust
// Complex subsystem
pub struct WriteCoordinator { /* ... */ }
pub struct ReadCoordinator { /* ... */ }
pub struct ErasureCoder { /* ... */ }
pub struct ShardDistributor { /* ... */ }

// Facade
pub struct EcStore {
    write_coordinator: Arc<WriteCoordinator>,
    read_coordinator: Arc<ReadCoordinator>,
    erasure_coder: Arc<ErasureCoder>,
    shard_distributor: Arc<ShardDistributor>,
}

impl EcStore {
    // Simple public API
    pub async fn put_object(&self, key: &str, data: Bytes) -> Result<ObjectInfo> {
        // Coordinates all subsystems internally
        let shards = self.erasure_coder.encode(data).await?;
        let locations = self.shard_distributor.distribute(shards).await?;
        self.write_coordinator.finalize(locations).await?;
        Ok(ObjectInfo { /* ... */ })
    }
    
    pub async fn get_object(&self, key: &str) -> Result<Bytes> {
        // Coordinates all subsystems internally
        let locations = self.read_coordinator.locate_shards(key).await?;
        let shards = self.shard_distributor.collect(locations).await?;
        let data = self.erasure_coder.decode(shards).await?;
        Ok(data)
    }
}
```

### 4. Strategy Pattern

**Description**: Select algorithm at runtime.

**Implementation**: KMS providers, storage backends, notification targets

**Benefits**:
- Pluggable algorithms
- Runtime selection
- Easy to add new strategies

**Example**:
```rust
// Strategy trait
pub trait KmsProvider: Send + Sync {
    async fn generate_data_key(&self) -> Result<DataKey>;
    async fn encrypt(&self, data: &[u8], key_id: &str) -> Result<Vec<u8>>;
    async fn decrypt(&self, data: &[u8]) -> Result<Vec<u8>>;
}

// Concrete strategies
pub struct LocalKmsProvider { /* ... */ }
pub struct AwsKmsProvider { /* ... */ }
pub struct VaultKmsProvider { /* ... */ }

impl KmsProvider for LocalKmsProvider {
    async fn generate_data_key(&self) -> Result<DataKey> {
        // Local implementation
    }
}

impl KmsProvider for AwsKmsProvider {
    async fn generate_data_key(&self) -> Result<DataKey> {
        // AWS KMS API call
    }
}

// Context that uses strategy
pub struct KmsService {
    provider: Box<dyn KmsProvider>,
}

impl KmsService {
    pub fn new(provider_type: KmsProviderType) -> Self {
        let provider: Box<dyn KmsProvider> = match provider_type {
            KmsProviderType::Local => Box::new(LocalKmsProvider::new()),
            KmsProviderType::Aws => Box::new(AwsKmsProvider::new()),
            KmsProviderType::Vault => Box::new(VaultKmsProvider::new()),
        };
        Self { provider }
    }
    
    pub async fn encrypt(&self, data: &[u8]) -> Result<Vec<u8>> {
        self.provider.encrypt(data, "default").await
    }
}
```

## Behavioral Patterns

### 1. Observer Pattern

**Description**: Objects notify observers of state changes.

**Implementation**: Notification system, event subscriptions

**Benefits**:
- Loose coupling
- Dynamic subscription
- Multiple observers

**Example**:
```rust
// Subject
pub struct ObjectStore {
    observers: Vec<Box<dyn StorageObserver>>,
}

// Observer trait
pub trait StorageObserver: Send + Sync {
    async fn on_object_created(&self, bucket: &str, key: &str);
    async fn on_object_deleted(&self, bucket: &str, key: &str);
}

impl ObjectStore {
    pub fn add_observer(&mut self, observer: Box<dyn StorageObserver>) {
        self.observers.push(observer);
    }
    
    pub async fn put_object(&self, bucket: &str, key: &str, data: Bytes) -> Result<()> {
        // ... store object ...
        
        // Notify observers
        for observer in &self.observers {
            observer.on_object_created(bucket, key).await;
        }
        Ok(())
    }
}

// Concrete observer
pub struct AuditLogger;

impl StorageObserver for AuditLogger {
    async fn on_object_created(&self, bucket: &str, key: &str) {
        info!("Object created: {}/{}", bucket, key);
    }
    
    async fn on_object_deleted(&self, bucket: &str, key: &str) {
        info!("Object deleted: {}/{}", bucket, key);
    }
}
```

### 2. Chain of Responsibility

**Description**: Pass request along chain of handlers.

**Implementation**: Middleware stack, policy evaluation

**Benefits**:
- Flexible handler composition
- Easy to add/remove handlers
- Single responsibility

**Example**:
```rust
// Handler trait
pub trait Middleware: Send + Sync {
    async fn handle(&self, req: Request, next: Next<'_>) -> Result<Response>;
}

// Concrete middleware
pub struct AuthMiddleware {
    iam_service: Arc<IamService>,
}

impl Middleware for AuthMiddleware {
    async fn handle(&self, mut req: Request, next: Next<'_>) -> Result<Response> {
        // Extract credentials
        let auth_header = req.headers().get("Authorization")
            .ok_or(Error::Unauthorized)?;
        
        // Authenticate
        let user = self.iam_service.authenticate(auth_header).await?;
        
        // Add user to request extensions
        req.extensions_mut().insert(user);
        
        // Continue chain
        next.run(req).await
    }
}

pub struct LoggingMiddleware;

impl Middleware for LoggingMiddleware {
    async fn handle(&self, req: Request, next: Next<'_>) -> Result<Response> {
        let start = Instant::now();
        info!("Request: {} {}", req.method(), req.uri());
        
        let response = next.run(req).await;
        
        let duration = start.elapsed();
        info!("Response: {:?} in {:?}", response.status(), duration);
        
        response
    }
}

// Chain construction
let app = Router::new()
    .route("/bucket/:bucket/object/:key", get(get_object))
    .layer(LoggingMiddleware)
    .layer(AuthMiddleware::new(iam_service));
```

## Concurrency Patterns

### 1. Actor Pattern

**Description**: Isolated actors communicate via message passing.

**Implementation**: Background workers, task processors

**Benefits**:
- No shared state
- Message-based communication
- Easy to reason about concurrency

**Example**:
```rust
use tokio::sync::mpsc;

// Actor message types
pub enum WorkerMessage {
    ProcessJob(Job),
    Shutdown,
}

// Actor
pub struct Worker {
    receiver: mpsc::Receiver<WorkerMessage>,
}

impl Worker {
    pub fn spawn() -> mpsc::Sender<WorkerMessage> {
        let (tx, rx) = mpsc::channel(100);
        
        let worker = Worker { receiver: rx };
        
        tokio::spawn(async move {
            worker.run().await;
        });
        
        tx
    }
    
    async fn run(mut self) {
        while let Some(msg) = self.receiver.recv().await {
            match msg {
                WorkerMessage::ProcessJob(job) => {
                    self.process_job(job).await;
                }
                WorkerMessage::Shutdown => {
                    info!("Worker shutting down");
                    break;
                }
            }
        }
    }
    
    async fn process_job(&self, job: Job) {
        // Process job in isolation
    }
}

// Usage
let worker = Worker::spawn();
worker.send(WorkerMessage::ProcessJob(job)).await?;
```

### 2. Reader-Writer Lock Pattern

**Description**: Multiple readers or single writer access.

**Implementation**: Metadata cache, configuration

**Benefits**:
- Concurrent reads
- Exclusive writes
- Better than Mutex for read-heavy workloads

**Example**:
```rust
use tokio::sync::RwLock;

pub struct MetadataCache {
    cache: Arc<RwLock<HashMap<String, Metadata>>>,
}

impl MetadataCache {
    pub async fn get(&self, key: &str) -> Option<Metadata> {
        let cache = self.cache.read().await;  // Multiple concurrent readers
        cache.get(key).cloned()
    }
    
    pub async fn put(&self, key: String, metadata: Metadata) {
        let mut cache = self.cache.write().await;  // Exclusive writer
        cache.insert(key, metadata);
    }
}
```

### 3. Pipeline Pattern

**Description**: Data flows through stages of processing.

**Implementation**: Object upload/download pipelines

**Benefits**:
- Parallel processing
- Clear stages
- Backpressure handling

**Example**:
```rust
use tokio::sync::mpsc;

// Pipeline stages
pub async fn read_stage(path: &Path) -> Result<mpsc::Receiver<Bytes>> {
    let (tx, rx) = mpsc::channel(10);
    let path = path.to_owned();
    
    tokio::spawn(async move {
        let mut file = tokio::fs::File::open(&path).await.unwrap();
        let mut buffer = vec![0u8; 64 * 1024];
        
        loop {
            let n = file.read(&mut buffer).await.unwrap();
            if n == 0 { break; }
            tx.send(Bytes::copy_from_slice(&buffer[..n])).await.unwrap();
        }
    });
    
    Ok(rx)
}

pub async fn encrypt_stage(
    mut input: mpsc::Receiver<Bytes>,
    key: &[u8],
) -> mpsc::Receiver<Bytes> {
    let (tx, rx) = mpsc::channel(10);
    let key = key.to_owned();
    
    tokio::spawn(async move {
        while let Some(chunk) = input.recv().await {
            let encrypted = encrypt(&chunk, &key);
            tx.send(encrypted).await.unwrap();
        }
    });
    
    rx
}

pub async fn encode_stage(
    mut input: mpsc::Receiver<Bytes>,
) -> mpsc::Receiver<Vec<Bytes>> {
    let (tx, rx) = mpsc::channel(10);
    
    tokio::spawn(async move {
        while let Some(chunk) = input.recv().await {
            let shards = erasure_encode(chunk);
            tx.send(shards).await.unwrap();
        }
    });
    
    rx
}

// Pipeline assembly
pub async fn upload_pipeline(path: &Path, key: &[u8]) -> Result<()> {
    let read_rx = read_stage(path).await?;
    let encrypt_rx = encrypt_stage(read_rx, key).await;
    let encode_rx = encode_stage(encrypt_rx).await;
    
    // Final stage writes shards
    while let Some(shards) = encode_rx.recv().await {
        write_shards(shards).await?;
    }
    
    Ok(())
}
```

## Error Handling Patterns

### 1. Result Type Pattern

**Description**: Explicit error handling with `Result<T, E>`.

**Implementation**: All fallible operations return `Result`

**Benefits**:
- Compile-time error checking
- No exceptions
- Clear error paths

**Example**:
```rust
use thiserror::Error;

#[derive(Debug, Error)]
pub enum StorageError {
    #[error("Object not found: {key}")]
    NotFound { key: String },
    
    #[error("Access denied: {reason}")]
    AccessDenied { reason: String },
    
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
    
    #[error("Insufficient shards: need {needed}, have {available}")]
    InsufficientShards { needed: usize, available: usize },
}

pub type Result<T> = std::result::Result<T, StorageError>;

// Usage
pub async fn get_object(key: &str) -> Result<Bytes> {
    let metadata = get_metadata(key).await?;  // ? operator propagates errors
    let shards = collect_shards(&metadata).await?;
    reconstruct_object(shards).await
}
```

### 2. Error Context Pattern

**Description**: Add context to errors as they propagate.

**Implementation**: Using `anyhow` or custom context

**Benefits**:
- Rich error messages
- Stack of context
- Better debugging

**Example**:
```rust
use anyhow::{Context, Result};

pub async fn read_config(path: &Path) -> Result<Config> {
    let content = tokio::fs::read_to_string(path).await
        .context(format!("Failed to read config file: {:?}", path))?;
    
    let config: Config = toml::from_str(&content)
        .context("Failed to parse TOML config")?;
    
    validate_config(&config)
        .context("Config validation failed")?;
    
    Ok(config)
}
```

## Rust-Specific Patterns

### 1. Newtype Pattern

**Description**: Wrap primitive types for type safety.

**Implementation**: BucketName, ObjectKey, UserId

**Benefits**:
- Type safety
- Domain semantics
- API clarity

**Example**:
```rust
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct BucketName(String);

impl BucketName {
    pub fn new(name: String) -> Result<Self> {
        if name.is_empty() || name.len() > 63 {
            return Err(Error::InvalidBucketName);
        }
        if !name.chars().all(|c| c.is_ascii_alphanumeric() || c == '-') {
            return Err(Error::InvalidBucketName);
        }
        Ok(Self(name))
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

// Usage prevents mixing up string types
pub async fn get_object(bucket: BucketName, key: ObjectKey) -> Result<Bytes> {
    // Type system ensures correct arguments
}
```

### 2. Extension Trait Pattern

**Description**: Add methods to external types.

**Implementation**: Extending Axum types, standard library types

**Benefits**:
- Add functionality to foreign types
- Organize related methods
- Trait coherence

**Example**:
```rust
// Extend Axum's Request type
pub trait RequestExt {
    fn user(&self) -> Option<&User>;
    fn bucket_name(&self) -> Result<BucketName>;
}

impl RequestExt for Request {
    fn user(&self) -> Option<&User> {
        self.extensions().get::<User>()
    }
    
    fn bucket_name(&self) -> Result<BucketName> {
        let path = self.uri().path();
        let bucket_str = path.split('/').nth(1)
            .ok_or(Error::InvalidRequest)?;
        BucketName::new(bucket_str.to_owned())
    }
}

// Usage
async fn handler(req: Request) -> Result<Response> {
    let user = req.user().ok_or(Error::Unauthorized)?;
    let bucket = req.bucket_name()?;
    // ...
}
```

### 3. Builder with Phantom Types

**Description**: Statically enforce builder states.

**Implementation**: Configuration builders with type states

**Benefits**:
- Compile-time validation
- Impossible to create invalid objects
- Clear API

**Example**:
```rust
// Type states
pub struct NoDataShards;
pub struct HasDataShards;

pub struct ConfigBuilder<State> {
    data_shards: Option<usize>,
    parity_shards: Option<usize>,
    _state: PhantomData<State>,
}

impl ConfigBuilder<NoDataShards> {
    pub fn new() -> Self {
        Self {
            data_shards: None,
            parity_shards: None,
            _state: PhantomData,
        }
    }
    
    pub fn data_shards(self, n: usize) -> ConfigBuilder<HasDataShards> {
        ConfigBuilder {
            data_shards: Some(n),
            parity_shards: self.parity_shards,
            _state: PhantomData,
        }
    }
}

impl ConfigBuilder<HasDataShards> {
    pub fn parity_shards(mut self, n: usize) -> Self {
        self.parity_shards = Some(n);
        self
    }
    
    // build() only available after data_shards is set
    pub fn build(self) -> Config {
        Config {
            data_shards: self.data_shards.unwrap(),
            parity_shards: self.parity_shards.unwrap_or(4),
        }
    }
}

// Usage - won't compile without data_shards
let config = ConfigBuilder::new()
    .data_shards(10)  // Required
    .parity_shards(4)
    .build();
```

## Anti-Patterns to Avoid

### 1. God Object
**Problem**: Single object doing too much  
**Solution**: Break into smaller, focused components

### 2. Anemic Domain Model
**Problem**: Data structures with no behavior  
**Solution**: Add methods to types that manipulate their data

### 3. Primitive Obsession
**Problem**: Using strings/ints for domain concepts  
**Solution**: Use newtype pattern for domain types

### 4. Uncontrolled Unwrap
**Problem**: `.unwrap()` everywhere causing panics  
**Solution**: Proper error handling with `Result`

### 5. Mutex Overuse
**Problem**: Mutex where RwLock or channels would be better  
**Solution**: Choose appropriate concurrency primitive

## Summary

RustFS employs a rich set of design patterns to achieve:
- **Modularity**: Clear boundaries and responsibilities
- **Safety**: Type system prevents bugs
- **Performance**: Zero-cost abstractions
- **Maintainability**: Well-known patterns
- **Testability**: Dependency injection and interfaces

These patterns work together to create a robust, performant, and maintainable distributed storage system.

---

**Next**: See [ADRs](09-adr/README.md) for architectural decisions.
