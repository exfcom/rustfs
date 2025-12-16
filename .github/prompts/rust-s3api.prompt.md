---
domain: "S3 API Implementation"
language: "rust"
frameworks: ["axum", "aws-sdk", "serde"]
---

# S3 API Implementation Patterns for RustFS

## S3 API Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     S3 API Categories                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Bucket Operations          Object Operations                │
│  ├─ CreateBucket            ├─ GetObject                     │
│  ├─ DeleteBucket            ├─ PutObject                     │
│  ├─ ListBuckets             ├─ DeleteObject                  │
│  ├─ HeadBucket              ├─ HeadObject                    │
│  └─ GetBucketLocation       ├─ CopyObject                    │
│                             ├─ ListObjects (v1/v2)           │
│  Multipart Operations       └─ DeleteObjects (batch)         │
│  ├─ CreateMultipartUpload                                    │
│  ├─ UploadPart              Access Control                   │
│  ├─ CompleteMultipartUpload ├─ GetBucketAcl                  │
│  ├─ AbortMultipartUpload    ├─ PutBucketAcl                  │
│  └─ ListParts               ├─ GetObjectAcl                  │
│                             └─ PutObjectAcl                  │
└─────────────────────────────────────────────────────────────┘
```

## Request/Response Models

### S3 Error Response
```rust
use serde::Serialize;

#[derive(Debug, Serialize)]
#[serde(rename = "Error")]
pub struct S3Error {
    #[serde(rename = "Code")]
    pub code: String,
    #[serde(rename = "Message")]
    pub message: String,
    #[serde(rename = "Resource")]
    pub resource: String,
    #[serde(rename = "RequestId")]
    pub request_id: String,
}

impl S3Error {
    pub fn no_such_bucket(bucket: &str, request_id: &str) -> Self {
        Self {
            code: "NoSuchBucket".into(),
            message: format!("The specified bucket does not exist: {}", bucket),
            resource: bucket.into(),
            request_id: request_id.into(),
        }
    }
    
    pub fn no_such_key(key: &str, request_id: &str) -> Self {
        Self {
            code: "NoSuchKey".into(),
            message: "The specified key does not exist.".into(),
            resource: key.into(),
            request_id: request_id.into(),
        }
    }
    
    pub fn access_denied(resource: &str, request_id: &str) -> Self {
        Self {
            code: "AccessDenied".into(),
            message: "Access Denied".into(),
            resource: resource.into(),
            request_id: request_id.into(),
        }
    }
}
```

### ListObjects Response
```rust
#[derive(Debug, Serialize)]
#[serde(rename = "ListBucketResult")]
pub struct ListBucketResult {
    #[serde(rename = "Name")]
    pub name: String,
    #[serde(rename = "Prefix")]
    pub prefix: String,
    #[serde(rename = "MaxKeys")]
    pub max_keys: i32,
    #[serde(rename = "IsTruncated")]
    pub is_truncated: bool,
    #[serde(rename = "Contents", default)]
    pub contents: Vec<ObjectInfo>,
    #[serde(rename = "CommonPrefixes", default)]
    pub common_prefixes: Vec<CommonPrefix>,
    #[serde(rename = "NextContinuationToken", skip_serializing_if = "Option::is_none")]
    pub next_continuation_token: Option<String>,
}

#[derive(Debug, Serialize)]
pub struct ObjectInfo {
    #[serde(rename = "Key")]
    pub key: String,
    #[serde(rename = "LastModified")]
    pub last_modified: String, // ISO 8601
    #[serde(rename = "ETag")]
    pub etag: String,
    #[serde(rename = "Size")]
    pub size: u64,
    #[serde(rename = "StorageClass")]
    pub storage_class: String,
}
```

## Axum Handler Patterns

### Route Definition
```rust
use axum::{
    Router,
    routing::{get, put, delete, head},
    extract::{Path, Query, State},
};

pub fn s3_routes() -> Router<AppState> {
    Router::new()
        // Bucket operations
        .route("/", get(list_buckets))
        .route("/:bucket", put(create_bucket))
        .route("/:bucket", delete(delete_bucket))
        .route("/:bucket", head(head_bucket))
        .route("/:bucket", get(list_objects))
        // Object operations
        .route("/:bucket/*key", get(get_object))
        .route("/:bucket/*key", put(put_object))
        .route("/:bucket/*key", delete(delete_object))
        .route("/:bucket/*key", head(head_object))
}
```

### GetObject Handler
```rust
use axum::{
    extract::{Path, State, Query},
    http::{StatusCode, HeaderMap, header},
    response::IntoResponse,
    body::Body,
};
use tokio_util::io::ReaderStream;

#[derive(Debug, Deserialize)]
pub struct GetObjectQuery {
    #[serde(rename = "versionId")]
    pub version_id: Option<String>,
    #[serde(rename = "partNumber")]
    pub part_number: Option<i32>,
}

pub async fn get_object(
    State(state): State<AppState>,
    Path((bucket, key)): Path<(String, String)>,
    Query(query): Query<GetObjectQuery>,
    headers: HeaderMap,
) -> Result<impl IntoResponse, S3ErrorResponse> {
    // Validate bucket exists
    let bucket_meta = state.storage
        .get_bucket(&bucket)
        .await
        .map_err(|_| S3ErrorResponse::no_such_bucket(&bucket))?;
    
    // Check authorization
    let auth_context = extract_auth_context(&headers)?;
    state.policy
        .check_get_object(&auth_context, &bucket, &key)
        .await
        .map_err(|_| S3ErrorResponse::access_denied())?;
    
    // Get object
    let object = state.storage
        .get_object(&bucket, &key, query.version_id.as_deref())
        .await
        .map_err(|e| match e {
            StorageError::NotFound { .. } => S3ErrorResponse::no_such_key(&key),
            _ => S3ErrorResponse::internal_error(),
        })?;
    
    // Build response headers
    let mut response_headers = HeaderMap::new();
    response_headers.insert(header::CONTENT_TYPE, object.content_type.parse().unwrap());
    response_headers.insert(header::CONTENT_LENGTH, object.size.into());
    response_headers.insert(header::ETAG, object.etag.parse().unwrap());
    response_headers.insert(
        header::LAST_MODIFIED,
        object.last_modified.format("%a, %d %b %Y %H:%M:%S GMT").to_string().parse().unwrap()
    );
    
    // Stream response body
    let stream = ReaderStream::new(object.body);
    let body = Body::from_stream(stream);
    
    Ok((StatusCode::OK, response_headers, body))
}
```

### PutObject Handler
```rust
use axum::body::Bytes;

pub async fn put_object(
    State(state): State<AppState>,
    Path((bucket, key)): Path<(String, String)>,
    headers: HeaderMap,
    body: Bytes,
) -> Result<impl IntoResponse, S3ErrorResponse> {
    // Extract metadata from headers
    let content_type = headers
        .get(header::CONTENT_TYPE)
        .and_then(|v| v.to_str().ok())
        .unwrap_or("application/octet-stream");
    
    let content_md5 = headers
        .get("content-md5")
        .and_then(|v| v.to_str().ok());
    
    // Validate content MD5 if provided
    if let Some(md5) = content_md5 {
        let computed = compute_md5(&body);
        if computed != md5 {
            return Err(S3ErrorResponse::bad_digest());
        }
    }
    
    // Check authorization
    let auth_context = extract_auth_context(&headers)?;
    state.policy
        .check_put_object(&auth_context, &bucket, &key)
        .await
        .map_err(|_| S3ErrorResponse::access_denied())?;
    
    // Store object
    let result = state.storage
        .put_object(&bucket, &key, body, PutOptions {
            content_type: content_type.into(),
            user_metadata: extract_user_metadata(&headers),
            ..Default::default()
        })
        .await?;
    
    // Build response
    let mut response_headers = HeaderMap::new();
    response_headers.insert(header::ETAG, result.etag.parse().unwrap());
    
    Ok((StatusCode::OK, response_headers))
}
```

### Multipart Upload Handlers
```rust
// Initiate multipart upload
pub async fn create_multipart_upload(
    State(state): State<AppState>,
    Path((bucket, key)): Path<(String, String)>,
    headers: HeaderMap,
) -> Result<impl IntoResponse, S3ErrorResponse> {
    let upload = state.storage
        .create_multipart_upload(&bucket, &key, CreateMultipartOptions {
            content_type: extract_content_type(&headers),
            metadata: extract_user_metadata(&headers),
        })
        .await?;
    
    let response = InitiateMultipartUploadResult {
        bucket: bucket.clone(),
        key: key.clone(),
        upload_id: upload.upload_id,
    };
    
    Ok((StatusCode::OK, Xml(response)))
}

// Upload part
pub async fn upload_part(
    State(state): State<AppState>,
    Path((bucket, key)): Path<(String, String)>,
    Query(query): Query<UploadPartQuery>,
    body: Bytes,
) -> Result<impl IntoResponse, S3ErrorResponse> {
    let part = state.storage
        .upload_part(
            &bucket,
            &key,
            &query.upload_id,
            query.part_number,
            body,
        )
        .await?;
    
    let mut headers = HeaderMap::new();
    headers.insert(header::ETAG, part.etag.parse().unwrap());
    
    Ok((StatusCode::OK, headers))
}

// Complete multipart upload
pub async fn complete_multipart_upload(
    State(state): State<AppState>,
    Path((bucket, key)): Path<(String, String)>,
    Query(query): Query<CompleteMultipartQuery>,
    Xml(body): Xml<CompleteMultipartUploadRequest>,
) -> Result<impl IntoResponse, S3ErrorResponse> {
    let result = state.storage
        .complete_multipart_upload(
            &bucket,
            &key,
            &query.upload_id,
            body.parts,
        )
        .await?;
    
    let response = CompleteMultipartUploadResult {
        location: format!("/{}/{}", bucket, key),
        bucket,
        key,
        etag: result.etag,
    };
    
    Ok((StatusCode::OK, Xml(response)))
}
```

## AWS Signature V4 Authentication

```rust
use hmac::{Hmac, Mac};
use sha2::Sha256;

pub struct SignatureV4Verifier {
    secret_key_lookup: Arc<dyn SecretKeyLookup>,
}

impl SignatureV4Verifier {
    pub async fn verify(&self, request: &Request) -> Result<AuthContext, AuthError> {
        // Extract authorization header
        let auth_header = request.headers()
            .get(header::AUTHORIZATION)
            .ok_or(AuthError::MissingAuth)?
            .to_str()
            .map_err(|_| AuthError::InvalidAuth)?;
        
        // Parse AWS4-HMAC-SHA256 Credential=..., SignedHeaders=..., Signature=...
        let parsed = parse_auth_header(auth_header)?;
        
        // Lookup secret key for access key
        let secret_key = self.secret_key_lookup
            .get_secret_key(&parsed.access_key)
            .await?;
        
        // Compute expected signature
        let string_to_sign = build_string_to_sign(request, &parsed)?;
        let signing_key = derive_signing_key(
            &secret_key,
            &parsed.date,
            &parsed.region,
            "s3",
        );
        let expected_signature = hex_encode(hmac_sha256(&signing_key, &string_to_sign));
        
        // Verify signature
        if expected_signature != parsed.signature {
            return Err(AuthError::SignatureMismatch);
        }
        
        Ok(AuthContext {
            access_key: parsed.access_key,
            // ... other context
        })
    }
}

fn derive_signing_key(secret: &str, date: &str, region: &str, service: &str) -> Vec<u8> {
    let k_date = hmac_sha256(format!("AWS4{}", secret).as_bytes(), date.as_bytes());
    let k_region = hmac_sha256(&k_date, region.as_bytes());
    let k_service = hmac_sha256(&k_region, service.as_bytes());
    hmac_sha256(&k_service, b"aws4_request")
}
```

## XML Serialization Helpers

```rust
use quick_xml::se::Serializer;
use serde::Serialize;

/// Custom XML response wrapper
pub struct Xml<T>(pub T);

impl<T: Serialize> IntoResponse for Xml<T> {
    fn into_response(self) -> Response {
        let mut buffer = String::new();
        let mut ser = Serializer::new(&mut buffer);
        ser.indent(' ', 2);
        
        match self.0.serialize(&mut ser) {
            Ok(_) => {
                let xml = format!(r#"<?xml version="1.0" encoding="UTF-8"?>{}"#, buffer);
                (
                    StatusCode::OK,
                    [(header::CONTENT_TYPE, "application/xml")],
                    xml,
                ).into_response()
            }
            Err(_) => StatusCode::INTERNAL_SERVER_ERROR.into_response(),
        }
    }
}
```

## S3 Error Codes Reference

| Code | HTTP Status | Description |
|------|-------------|-------------|
| AccessDenied | 403 | Access denied |
| BucketAlreadyExists | 409 | Bucket already exists |
| BucketNotEmpty | 409 | Bucket is not empty |
| EntityTooLarge | 400 | Entity too large |
| InvalidAccessKeyId | 403 | Invalid access key |
| InvalidBucketName | 400 | Invalid bucket name |
| InvalidDigest | 400 | Content-MD5 mismatch |
| InvalidPart | 400 | Invalid part |
| InvalidPartOrder | 400 | Parts not in order |
| NoSuchBucket | 404 | Bucket not found |
| NoSuchKey | 404 | Object not found |
| NoSuchUpload | 404 | Multipart upload not found |
| SignatureDoesNotMatch | 403 | Signature mismatch |
