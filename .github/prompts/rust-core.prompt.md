---
domain: "Rust Core Development"
language: "rust"
version: "edition 2024, rust-version 1.85"
frameworks: ["thiserror", "serde", "tracing"]
---

# Rust Core Development Patterns for RustFS

## Language Configuration

```toml
# Cargo.toml workspace settings
[workspace.package]
edition = "2024"
rust-version = "1.85"

[workspace.lints.rust]
unsafe_code = "deny"

[workspace.lints.clippy]
all = "warn"
```

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Functions | snake_case | `get_object`, `validate_bucket` |
| Types/Traits | PascalCase | `BucketPolicy`, `StorageService` |
| Constants | SCREAMING_SNAKE | `MAX_OBJECT_SIZE`, `DEFAULT_TIMEOUT` |
| Modules | snake_case | `bucket_metadata`, `policy_engine` |
| Lifetimes | lowercase | `'a`, `'ctx` |

## Error Handling Pattern

```rust
use thiserror::Error;

/// Domain-specific error type
#[derive(Error, Debug)]
pub enum StorageError {
    #[error("Bucket not found: {bucket}")]
    BucketNotFound { bucket: String },
    
    #[error("Object not found: {bucket}/{key}")]
    ObjectNotFound { bucket: String, key: String },
    
    #[error("Access denied to {resource}")]
    AccessDenied { resource: String },
    
    #[error("Invalid input: {message}")]
    InvalidInput { message: String },
    
    #[error("Internal error: {0}")]
    Internal(#[from] anyhow::Error),
    
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}

/// Convenient Result alias
pub type Result<T> = std::result::Result<T, StorageError>;
```

## Type Design Patterns

### Newtype Pattern
```rust
/// Strongly-typed bucket name
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct BucketName(String);

impl BucketName {
    pub fn new(name: impl Into<String>) -> Result<Self> {
        let name = name.into();
        validate_bucket_name(&name)?;
        Ok(Self(name))
    }
    
    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl TryFrom<String> for BucketName {
    type Error = StorageError;
    
    fn try_from(value: String) -> Result<Self> {
        Self::new(value)
    }
}
```

### Builder Pattern
```rust
#[derive(Default)]
pub struct ObjectBuilder {
    bucket: Option<BucketName>,
    key: Option<String>,
    content_type: Option<String>,
    metadata: HashMap<String, String>,
}

impl ObjectBuilder {
    pub fn new() -> Self {
        Self::default()
    }
    
    pub fn bucket(mut self, bucket: BucketName) -> Self {
        self.bucket = Some(bucket);
        self
    }
    
    pub fn key(mut self, key: impl Into<String>) -> Self {
        self.key = Some(key.into());
        self
    }
    
    pub fn content_type(mut self, ct: impl Into<String>) -> Self {
        self.content_type = Some(ct.into());
        self
    }
    
    pub fn metadata(mut self, key: impl Into<String>, value: impl Into<String>) -> Self {
        self.metadata.insert(key.into(), value.into());
        self
    }
    
    pub fn build(self) -> Result<Object> {
        Ok(Object {
            bucket: self.bucket.ok_or(StorageError::InvalidInput {
                message: "bucket is required".into()
            })?,
            key: self.key.ok_or(StorageError::InvalidInput {
                message: "key is required".into()
            })?,
            content_type: self.content_type,
            metadata: self.metadata,
        })
    }
}
```

## Trait Design

### Service Trait
```rust
use async_trait::async_trait;

/// Core storage service trait
#[async_trait]
pub trait StorageService: Send + Sync + 'static {
    /// Get object data
    async fn get_object(&self, bucket: &BucketName, key: &str) -> Result<ObjectData>;
    
    /// Put object
    async fn put_object(&self, bucket: &BucketName, key: &str, data: Bytes) -> Result<PutResult>;
    
    /// Delete object
    async fn delete_object(&self, bucket: &BucketName, key: &str) -> Result<()>;
    
    /// List objects with pagination
    async fn list_objects(&self, bucket: &BucketName, options: ListOptions) -> Result<ListResult>;
}
```

### Extension Trait
```rust
/// Extension methods for Result
pub trait ResultExt<T, E> {
    /// Log error and convert to internal error
    fn log_err(self, context: &str) -> Result<T>;
}

impl<T, E: std::error::Error> ResultExt<T, E> for std::result::Result<T, E> {
    fn log_err(self, context: &str) -> Result<T> {
        self.map_err(|e| {
            tracing::error!(error = %e, context = %context, "Operation failed");
            StorageError::Internal(anyhow::anyhow!("{}: {}", context, e))
        })
    }
}
```

## Serialization Patterns

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct ObjectMetadata {
    pub key: String,
    pub size: u64,
    pub last_modified: DateTime<Utc>,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub content_type: Option<String>,
    #[serde(default)]
    pub user_metadata: HashMap<String, String>,
}

// Custom serialization
impl ObjectMetadata {
    pub fn to_xml(&self) -> String {
        // S3-compatible XML format
        format!(
            "<Contents><Key>{}</Key><Size>{}</Size></Contents>",
            self.key, self.size
        )
    }
}
```

## Logging with Tracing

```rust
use tracing::{debug, error, info, instrument, warn, Span};

#[instrument(skip(self, data), fields(bucket = %bucket, key = %key, size = data.len()))]
pub async fn put_object(
    &self,
    bucket: &BucketName,
    key: &str,
    data: Bytes,
) -> Result<PutResult> {
    info!("Starting object upload");
    
    // Add dynamic field
    Span::current().record("content_type", &tracing::field::display(&content_type));
    
    match self.storage.write(bucket, key, &data).await {
        Ok(result) => {
            info!(etag = %result.etag, "Object uploaded successfully");
            Ok(result)
        }
        Err(e) => {
            error!(error = %e, "Failed to upload object");
            Err(e.into())
        }
    }
}
```

## Unit Test Pattern

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn bucket_name_validates_correctly() {
        // Valid names
        assert!(BucketName::new("valid-bucket").is_ok());
        assert!(BucketName::new("bucket123").is_ok());
        
        // Invalid names
        assert!(BucketName::new("Invalid").is_err()); // uppercase
        assert!(BucketName::new("ab").is_err());      // too short
        assert!(BucketName::new("-invalid").is_err()); // starts with hyphen
    }

    #[test]
    fn object_builder_requires_bucket_and_key() {
        let result = ObjectBuilder::new().build();
        assert!(result.is_err());
        
        let result = ObjectBuilder::new()
            .bucket(BucketName::new("test").unwrap())
            .key("test-key")
            .build();
        assert!(result.is_ok());
    }
}
```

## Common Imports Template

```rust
// Standard library
use std::collections::HashMap;
use std::sync::Arc;

// Error handling
use thiserror::Error;
use anyhow::Context;

// Async
use tokio::sync::{Mutex, RwLock};
use async_trait::async_trait;

// Serialization
use serde::{Deserialize, Serialize};
use bytes::Bytes;

// Logging
use tracing::{debug, error, info, instrument, warn};

// Time
use chrono::{DateTime, Utc};
```
