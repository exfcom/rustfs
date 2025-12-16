---
domain: "Async Rust Programming"
language: "rust"
frameworks: ["tokio", "async-trait", "futures"]
---

# Async Rust Patterns for RustFS

## Runtime Configuration

```rust
// Main entry point with Tokio runtime
#[tokio::main]
async fn main() -> Result<()> {
    // Initialize observability
    init_tracing();
    
    // Run the application
    run_server().await
}

// Custom runtime configuration
fn build_runtime() -> tokio::runtime::Runtime {
    tokio::runtime::Builder::new_multi_thread()
        .worker_threads(num_cpus::get())
        .enable_all()
        .build()
        .expect("Failed to build runtime")
}
```

## Async Trait Pattern

```rust
use async_trait::async_trait;

/// Async storage interface
#[async_trait]
pub trait AsyncStorage: Send + Sync {
    async fn read(&self, path: &str) -> Result<Vec<u8>>;
    async fn write(&self, path: &str, data: &[u8]) -> Result<()>;
    async fn delete(&self, path: &str) -> Result<()>;
    async fn exists(&self, path: &str) -> Result<bool>;
}

/// Implementation
pub struct FileStorage {
    root: PathBuf,
}

#[async_trait]
impl AsyncStorage for FileStorage {
    async fn read(&self, path: &str) -> Result<Vec<u8>> {
        let full_path = self.root.join(path);
        tokio::fs::read(full_path)
            .await
            .map_err(Into::into)
    }
    
    async fn write(&self, path: &str, data: &[u8]) -> Result<()> {
        let full_path = self.root.join(path);
        tokio::fs::write(full_path, data)
            .await
            .map_err(Into::into)
    }
    
    async fn delete(&self, path: &str) -> Result<()> {
        let full_path = self.root.join(path);
        tokio::fs::remove_file(full_path)
            .await
            .map_err(Into::into)
    }
    
    async fn exists(&self, path: &str) -> Result<bool> {
        let full_path = self.root.join(path);
        Ok(tokio::fs::try_exists(full_path).await?)
    }
}
```

## Concurrency Patterns

### Shared State with RwLock
```rust
use tokio::sync::RwLock;
use std::sync::Arc;

pub struct CacheService {
    cache: Arc<RwLock<HashMap<String, CachedItem>>>,
}

impl CacheService {
    pub fn new() -> Self {
        Self {
            cache: Arc::new(RwLock::new(HashMap::new())),
        }
    }
    
    pub async fn get(&self, key: &str) -> Option<CachedItem> {
        let cache = self.cache.read().await;
        cache.get(key).cloned()
    }
    
    pub async fn set(&self, key: String, value: CachedItem) {
        let mut cache = self.cache.write().await;
        cache.insert(key, value);
    }
    
    pub async fn invalidate(&self, key: &str) {
        let mut cache = self.cache.write().await;
        cache.remove(key);
    }
}
```

### Channel Communication
```rust
use tokio::sync::mpsc;

pub struct EventProcessor {
    sender: mpsc::Sender<Event>,
}

impl EventProcessor {
    pub fn new(buffer_size: usize) -> (Self, mpsc::Receiver<Event>) {
        let (sender, receiver) = mpsc::channel(buffer_size);
        (Self { sender }, receiver)
    }
    
    pub async fn send(&self, event: Event) -> Result<()> {
        self.sender
            .send(event)
            .await
            .map_err(|_| Error::ChannelClosed)
    }
}

// Consumer task
async fn process_events(mut receiver: mpsc::Receiver<Event>) {
    while let Some(event) = receiver.recv().await {
        if let Err(e) = handle_event(event).await {
            tracing::error!(error = %e, "Failed to process event");
        }
    }
}
```

### Semaphore for Rate Limiting
```rust
use tokio::sync::Semaphore;

pub struct RateLimiter {
    semaphore: Arc<Semaphore>,
}

impl RateLimiter {
    pub fn new(max_concurrent: usize) -> Self {
        Self {
            semaphore: Arc::new(Semaphore::new(max_concurrent)),
        }
    }
    
    pub async fn acquire(&self) -> Result<SemaphorePermit<'_>> {
        self.semaphore
            .acquire()
            .await
            .map_err(|_| Error::RateLimitExceeded)
    }
}

// Usage
async fn process_request(limiter: &RateLimiter, request: Request) -> Result<Response> {
    let _permit = limiter.acquire().await?;
    // Request is rate-limited while permit is held
    handle_request(request).await
}
```

## Parallel Processing

### Join Multiple Futures
```rust
use tokio::try_join;

async fn fetch_all_metadata(keys: &[String]) -> Result<Vec<Metadata>> {
    // Process in parallel with join
    let (meta1, meta2, meta3) = try_join!(
        fetch_metadata(&keys[0]),
        fetch_metadata(&keys[1]),
        fetch_metadata(&keys[2]),
    )?;
    
    Ok(vec![meta1, meta2, meta3])
}
```

### Concurrent Stream Processing
```rust
use futures::stream::{self, StreamExt};

async fn process_objects(objects: Vec<ObjectKey>) -> Result<Vec<ProcessedObject>> {
    let results: Vec<Result<ProcessedObject>> = stream::iter(objects)
        .map(|key| async move {
            process_single_object(key).await
        })
        .buffer_unordered(10) // Process 10 at a time
        .collect()
        .await;
    
    results.into_iter().collect()
}
```

### Spawn Blocking for CPU Work
```rust
pub async fn compute_checksum(data: Bytes) -> Result<String> {
    tokio::task::spawn_blocking(move || {
        // CPU-intensive work runs on blocking thread pool
        let mut hasher = Sha256::new();
        hasher.update(&data);
        Ok(format!("{:x}", hasher.finalize()))
    })
    .await
    .map_err(|e| Error::Internal(e.into()))?
}
```

## Graceful Shutdown

```rust
use tokio::signal;
use tokio_util::sync::CancellationToken;

pub struct Server {
    cancel_token: CancellationToken,
}

impl Server {
    pub fn new() -> Self {
        Self {
            cancel_token: CancellationToken::new(),
        }
    }
    
    pub async fn run(&self) -> Result<()> {
        let shutdown_signal = async {
            signal::ctrl_c().await.expect("Failed to listen for Ctrl+C");
            tracing::info!("Shutdown signal received");
        };
        
        tokio::select! {
            result = self.serve() => result,
            _ = shutdown_signal => {
                self.cancel_token.cancel();
                self.graceful_shutdown().await
            }
        }
    }
    
    async fn graceful_shutdown(&self) -> Result<()> {
        tracing::info!("Starting graceful shutdown");
        
        // Wait for in-flight requests with timeout
        tokio::time::timeout(
            Duration::from_secs(30),
            self.wait_for_connections(),
        )
        .await
        .ok();
        
        tracing::info!("Shutdown complete");
        Ok(())
    }
}
```

## Timeout Patterns

```rust
use tokio::time::{timeout, Duration};

pub async fn fetch_with_timeout<T, F, Fut>(
    operation: F,
    timeout_duration: Duration,
) -> Result<T>
where
    F: FnOnce() -> Fut,
    Fut: Future<Output = Result<T>>,
{
    timeout(timeout_duration, operation())
        .await
        .map_err(|_| Error::Timeout)?
}

// Usage
let result = fetch_with_timeout(
    || async { storage.get_object(bucket, key).await },
    Duration::from_secs(30),
).await?;
```

## Select Pattern

```rust
use tokio::select;

async fn process_with_cancellation(
    cancel: CancellationToken,
    data: Data,
) -> Result<Output> {
    select! {
        result = process_data(data) => result,
        _ = cancel.cancelled() => {
            tracing::warn!("Operation cancelled");
            Err(Error::Cancelled)
        }
    }
}
```

## Async Test Pattern

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tokio::test;

    #[tokio::test]
    async fn test_concurrent_operations() {
        let service = TestService::new();
        
        // Run operations concurrently
        let (r1, r2, r3) = tokio::join!(
            service.operation_a(),
            service.operation_b(),
            service.operation_c(),
        );
        
        assert!(r1.is_ok());
        assert!(r2.is_ok());
        assert!(r3.is_ok());
    }

    #[tokio::test]
    async fn test_timeout_behavior() {
        let service = SlowService::new();
        
        let result = tokio::time::timeout(
            Duration::from_millis(100),
            service.slow_operation(),
        ).await;
        
        assert!(result.is_err()); // Should timeout
    }
}
```

## Anti-Patterns to Avoid

### ❌ Blocking in Async Context
```rust
// BAD - blocks the async runtime
async fn bad_read() -> Result<String> {
    std::fs::read_to_string("file.txt") // Blocking!
}

// GOOD - use async I/O
async fn good_read() -> Result<String> {
    tokio::fs::read_to_string("file.txt").await
}
```

### ❌ Holding Lock Across Await
```rust
// BAD - holds lock across await point
async fn bad_update(state: Arc<Mutex<State>>) {
    let mut guard = state.lock().await;
    async_operation().await; // Lock held during await!
    guard.value = new_value;
}

// GOOD - minimize lock scope
async fn good_update(state: Arc<Mutex<State>>) {
    let new_value = async_operation().await;
    let mut guard = state.lock().await;
    guard.value = new_value;
    // Lock released immediately
}
```

### ❌ Spawning Without Tracking
```rust
// BAD - fire and forget
async fn bad_spawn() {
    tokio::spawn(async { 
        important_work().await; // Error ignored!
    });
}

// GOOD - track spawned tasks
async fn good_spawn() -> Result<()> {
    let handle = tokio::spawn(async { 
        important_work().await
    });
    handle.await??
}
```
