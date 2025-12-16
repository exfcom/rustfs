# Sequence Diagrams

## Introduction

This document contains sequence diagrams for critical workflows in RustFS, illustrating the interactions between components during key operations.

## 1. Object Upload (PutObject)

```plantuml
@startuml PutObject_Sequence
title S3 PutObject - Complete Flow

actor Client
participant "S3 API\nHandler" as API
participant "Signature\nValidator" as SIG
participant "IAM\nService" as IAM
participant "Policy\nEngine" as POLICY
participant "KMS\nService" as KMS
participant "Write\nCoordinator" as WRITE
participant "Erasure\nCoder" as EC
participant "Shard\nDistributor" as DIST
participant "Storage\nBackend" as STORAGE
participant "Metadata\nStore" as META
participant "Notify\nService" as NOTIFY
participant "Audit\nService" as AUDIT

== Authentication ==
Client -> API: PUT /bucket/object\nAuthorization: AWS4-HMAC-SHA256...
activate API

API -> SIG: validate_signature(request)
activate SIG
SIG -> IAM: verify_access_key(access_key)
activate IAM
IAM --> SIG: User(id, roles, permissions)
deactivate IAM

SIG -> SIG: compute_expected_signature()
SIG -> SIG: compare_signatures()
SIG --> API: SignatureValid + User
deactivate SIG

== Authorization ==
API -> POLICY: check_permission(\n  user,\n  "s3:PutObject",\n  "arn:aws:s3:::bucket/object"\n)
activate POLICY
POLICY -> POLICY: load_user_policies()
POLICY -> POLICY: evaluate_statements()
POLICY -> POLICY: check_conditions()
POLICY --> API: Decision::Allow
deactivate POLICY

== Data Encryption ==
API -> API: read_request_body()
API -> WRITE: write_object(\n  bucket="bucket",\n  key="object",\n  data=body,\n  metadata=headers\n)
activate WRITE

alt Server-Side Encryption Enabled
    WRITE -> KMS: generate_data_key(master_key_id)
    activate KMS
    KMS -> KMS: generate_random_key()
    KMS -> KMS: encrypt_with_master_key()
    KMS --> WRITE: DataKey{\n  plaintext,\n  encrypted\n}
    deactivate KMS
    
    WRITE -> WRITE: encrypt_data(\n  data,\n  data_key.plaintext\n)
end

== Erasure Coding ==
WRITE -> EC: encode(\n  data,\n  data_shards=10,\n  parity_shards=4\n)
activate EC
EC -> EC: split_into_data_shards()
EC -> EC: calculate_parity_shards()
EC --> WRITE: shards[14]
deactivate EC

== Shard Distribution ==
WRITE -> DIST: distribute_shards(\n  shards,\n  storage_policy\n)
activate DIST

loop for each shard (14 shards)
    DIST -> DIST: select_storage_location()
    DIST -> STORAGE: write_shard(\n  shard_id,\n  shard_data,\n  path\n)
    activate STORAGE
    STORAGE -> STORAGE: write_to_disk()
    STORAGE -> STORAGE: calculate_checksum()
    STORAGE --> DIST: ShardLocation{\n  path,\n  checksum\n}
    deactivate STORAGE
end

DIST --> WRITE: shard_locations[]
deactivate DIST

== Metadata Storage ==
WRITE -> META: store_metadata(\n  bucket,\n  key,\n  ObjectMetadata{\n    size,\n    etag,\n    content_type,\n    encryption_info,\n    shard_locations\n  }\n)
activate META
META -> META: generate_etag()
META -> META: persist_to_store()
META --> WRITE: MetadataStored
deactivate META

WRITE --> API: WriteResult{\n  etag,\n  version_id\n}
deactivate WRITE

== Event Notification ==
API -> NOTIFY: publish_event(\n  Event::ObjectCreated,\n  bucket,\n  key,\n  metadata\n)
activate NOTIFY
NOTIFY -> NOTIFY: apply_filters()
NOTIFY -> NOTIFY: fan_out_to_targets()
NOTIFY --> API: EventPublished
deactivate NOTIFY

== Audit Logging ==
API -> AUDIT: log_operation(\n  operation="PutObject",\n  user,\n  resource,\n  result="Success"\n)
activate AUDIT
AUDIT -> AUDIT: format_audit_record()
AUDIT -> AUDIT: persist_log()
AUDIT --> API: LogRecorded
deactivate AUDIT

== Response ==
API --> Client: 200 OK\nETag: "abc123..."\nx-amz-version-id: "v1"
deactivate API

@enduml
```

## 2. Object Download (GetObject)

```plantuml
@startuml GetObject_Sequence
title S3 GetObject - Complete Flow

actor Client
participant "S3 API\nHandler" as API
participant "Signature\nValidator" as SIG
participant "IAM\nService" as IAM
participant "Policy\nEngine" as POLICY
participant "Metadata\nStore" as META
participant "Read\nCoordinator" as READ
participant "Shard\nCollector" as COLLECTOR
participant "Storage\nBackend" as STORAGE
participant "Reconstruction\nEngine" as RECON
participant "Erasure\nCoder" as EC
participant "KMS\nService" as KMS
participant "Audit\nService" as AUDIT

== Authentication & Authorization ==
Client -> API: GET /bucket/object
activate API

API -> SIG: validate_signature(request)
activate SIG
SIG -> IAM: verify_access_key()
IAM --> SIG: User
SIG --> API: SignatureValid + User
deactivate SIG

API -> POLICY: check_permission(\n  user,\n  "s3:GetObject",\n  resource\n)
POLICY --> API: Decision::Allow

== Metadata Lookup ==
API -> META: get_metadata(bucket, key)
activate META
META -> META: query_store()
META --> API: ObjectMetadata{\n  size,\n  etag,\n  encryption_info,\n  shard_locations[]\n}
deactivate META

== Shard Collection ==
API -> READ: read_object(bucket, key, metadata)
activate READ

READ -> COLLECTOR: collect_shards(shard_locations)
activate COLLECTOR

par Parallel Shard Reads
    loop for each shard location
        COLLECTOR -> STORAGE: read_shard(shard_id, location)
        activate STORAGE
        STORAGE -> STORAGE: read_from_disk()
        STORAGE -> STORAGE: verify_checksum()
        STORAGE --> COLLECTOR: shard_data
        deactivate STORAGE
    end
end

COLLECTOR -> COLLECTOR: validate_shard_count()
COLLECTOR --> READ: shards[]
deactivate COLLECTOR

== Data Reconstruction ==
alt Some shards missing/corrupted
    READ -> RECON: reconstruct_missing_shards(\n  available_shards,\n  missing_indices\n)
    activate RECON
    RECON -> EC: decode_shards(\n  shards,\n  data_shards=10,\n  parity_shards=4\n)
    EC --> RECON: reconstructed_shards[]
    RECON --> READ: complete_shards[]
    deactivate RECON
else All shards available
    READ -> EC: decode_data_shards(shards)
    EC --> READ: original_data
end

== Decryption ==
alt Object Encrypted
    READ -> KMS: decrypt_data_key(\n  encrypted_data_key\n)
    activate KMS
    KMS -> KMS: decrypt_with_master_key()
    KMS --> READ: plaintext_data_key
    deactivate KMS
    
    READ -> READ: decrypt_data(\n  encrypted_data,\n  plaintext_data_key\n)
end

READ --> API: object_data
deactivate READ

== Audit Logging ==
API -> AUDIT: log_operation(\n  "GetObject",\n  user,\n  resource,\n  "Success"\n)

== Response ==
API --> Client: 200 OK\nContent-Length: ...\nETag: ...\n\n[object data]
deactivate API

@enduml
```

## 3. User Authentication Flow

```plantuml
@startuml Authentication_Flow
title User Authentication and Authorization Flow

actor User
participant "S3 API" as API
participant "Signature\nValidator" as SIG
participant "IAM\nService" as IAM
participant "LDAP\nConnector" as LDAP
participant "Session\nManager" as SESSION
participant "Token\nGenerator" as TOKEN
participant "Policy\nEngine" as POLICY
participant "Cache" as CACHE

== Signature-Based Authentication ==
User -> API: Request with\nAWS4-HMAC-SHA256 Signature
activate API

API -> SIG: validate_signature(request)
activate SIG

SIG -> SIG: parse_authorization_header()
SIG -> SIG: extract_access_key()

SIG -> CACHE: get_credentials(access_key)
activate CACHE

alt Cache Hit
    CACHE --> SIG: cached_credentials
else Cache Miss
    CACHE --> SIG: None
    deactivate CACHE
    
    SIG -> IAM: get_user_credentials(access_key)
    activate IAM
    
    alt Local User
        IAM -> IAM: lookup_local_user()
        IAM --> SIG: Credentials{\n  access_key,\n  secret_key\n}
    else LDAP User
        IAM -> LDAP: authenticate(\n  username,\n  password\n)
        activate LDAP
        LDAP -> LDAP: bind_to_ldap()
        LDAP -> LDAP: search_user()
        LDAP --> IAM: LdapUser
        deactivate LDAP
        
        IAM -> IAM: map_ldap_to_local()
        IAM --> SIG: Credentials
    end
    deactivate IAM
    
    SIG -> CACHE: set_credentials(\n  access_key,\n  credentials,\n  ttl=300s\n)
end

SIG -> SIG: compute_signature(\n  request,\n  secret_key\n)
SIG -> SIG: compare_signatures()

alt Signature Valid
    SIG -> IAM: get_user_details(access_key)
    activate IAM
    IAM -> IAM: load_user_roles()
    IAM -> IAM: load_user_attributes()
    IAM --> SIG: User{\n  id,\n  username,\n  roles[]\n}
    deactivate IAM
    
    SIG --> API: AuthResult{\n  valid=true,\n  user\n}
else Signature Invalid
    SIG --> API: AuthResult{\n  valid=false,\n  error="InvalidSignature"\n}
    API --> User: 403 Forbidden\nSignatureDoesNotMatch
    deactivate API
end
deactivate SIG

== Session Creation ==
API -> SESSION: create_session(user)
activate SESSION

SESSION -> TOKEN: generate_token(user)
activate TOKEN
TOKEN -> TOKEN: create_jwt_claims()
TOKEN -> TOKEN: sign_jwt()
TOKEN --> SESSION: jwt_token
deactivate TOKEN

SESSION -> CACHE: store_session(\n  session_id,\n  session_data,\n  ttl=3600s\n)
SESSION --> API: Session{\n  id,\n  token,\n  expires_at\n}
deactivate SESSION

== Authorization Check ==
API -> POLICY: check_permission(\n  user,\n  action,\n  resource\n)
activate POLICY

POLICY -> POLICY: load_user_policies()

loop for each policy
    POLICY -> POLICY: evaluate_statement()
    POLICY -> POLICY: check_conditions()
end

POLICY -> POLICY: apply_deny_precedence()

alt Allowed
    POLICY --> API: Decision::Allow
    API -> API: process_request()
    API --> User: 200 OK\n[response data]
else Denied
    POLICY --> API: Decision::Deny
    deactivate POLICY
    API --> User: 403 Forbidden\nAccessDenied
end

deactivate API

@enduml
```

## 4. Multipart Upload Flow

```plantuml
@startuml Multipart_Upload
title S3 Multipart Upload Flow

actor Client
participant "S3 API" as API
participant "Multipart\nHandler" as MPH
participant "Metadata\nStore" as META
participant "Temp\nStorage" as TEMP
participant "Write\nCoordinator" as WRITE
participant "Storage\nEngine" as STORAGE

== Initiate Multipart Upload ==
Client -> API: POST /bucket/object?uploads
activate API

API -> MPH: initiate_multipart_upload(\n  bucket,\n  key,\n  metadata\n)
activate MPH

MPH -> MPH: generate_upload_id()
MPH -> META: store_upload_metadata(\n  upload_id,\n  bucket,\n  key,\n  metadata\n)
activate META
META --> MPH: Stored
deactivate META

MPH --> API: UploadId
API --> Client: 200 OK\nUploadId: "abc123..."
deactivate MPH
deactivate API

== Upload Parts (concurrent) ==
par Part 1
    Client -> API: PUT /bucket/object?\n  uploadId=abc123&partNumber=1
    activate API
    API -> MPH: upload_part(upload_id, 1, data)
    activate MPH
    MPH -> TEMP: store_part(upload_id, 1, data)
    activate TEMP
    TEMP -> TEMP: calculate_etag()
    TEMP --> MPH: PartETag
    deactivate TEMP
    MPH -> META: record_part(upload_id, 1, etag)
    MPH --> API: PartETag
    deactivate MPH
    API --> Client: 200 OK\nETag: "part1-etag"
    deactivate API
and Part 2
    Client -> API: PUT /bucket/object?\n  uploadId=abc123&partNumber=2
    activate API
    API -> MPH: upload_part(upload_id, 2, data)
    activate MPH
    MPH -> TEMP: store_part(upload_id, 2, data)
    activate TEMP
    TEMP --> MPH: PartETag
    deactivate TEMP
    MPH -> META: record_part(upload_id, 2, etag)
    MPH --> API: PartETag
    deactivate MPH
    API --> Client: 200 OK\nETag: "part2-etag"
    deactivate API
and Part N
    Client -> API: PUT /bucket/object?\n  uploadId=abc123&partNumber=N
    activate API
    API -> MPH: upload_part(upload_id, N, data)
    activate MPH
    MPH -> TEMP: store_part(upload_id, N, data)
    activate TEMP
    TEMP --> MPH: PartETag
    deactivate TEMP
    MPH -> META: record_part(upload_id, N, etag)
    MPH --> API: PartETag
    deactivate MPH
    API --> Client: 200 OK\nETag: "partN-etag"
    deactivate API
end

== Complete Multipart Upload ==
Client -> API: POST /bucket/object?\n  uploadId=abc123\n<CompleteMultipartUpload>
activate API

API -> MPH: complete_multipart_upload(\n  upload_id,\n  parts[]\n)
activate MPH

MPH -> META: get_upload_parts(upload_id)
activate META
META --> MPH: parts[]
deactivate META

MPH -> MPH: validate_parts_sequence()
MPH -> MPH: validate_part_etags()

loop for each part
    MPH -> TEMP: read_part(upload_id, part_num)
    activate TEMP
    TEMP --> MPH: part_data
    deactivate TEMP
end

MPH -> MPH: concatenate_parts()

MPH -> WRITE: write_object(\n  bucket,\n  key,\n  concatenated_data,\n  metadata\n)
activate WRITE
WRITE -> STORAGE: store_with_erasure_coding()
WRITE --> MPH: ObjectInfo
deactivate WRITE

MPH -> TEMP: cleanup_parts(upload_id)
MPH -> META: delete_upload_metadata(upload_id)

MPH --> API: CompleteResult{\n  location,\n  bucket,\n  key,\n  etag\n}
deactivate MPH

API --> Client: 200 OK\n<CompleteMultipartUploadResult>\n  <ETag>final-etag</ETag>\n</CompleteMultipartUploadResult>
deactivate API

@enduml
```

## 5. Key Management Service Flow

```plantuml
@startuml KMS_Flow
title Key Management Service - Encryption/Decryption Flow

participant "S3 API" as API
participant "Write\nCoordinator" as WRITE
participant "KMS\nService" as KMS
participant "Master Key\nStore" as MASTER
participant "External\nKMS" as EXTERNAL
participant "HSM" as HSM

== Data Encryption (Envelope Encryption) ==
API -> WRITE: write_encrypted_object(data)
activate WRITE

WRITE -> KMS: generate_data_key(master_key_id)
activate KMS

KMS -> MASTER: get_master_key(master_key_id)
activate MASTER

alt Local Master Key
    MASTER -> MASTER: retrieve_from_local_store()
    MASTER --> KMS: MasterKey
else External KMS
    MASTER -> EXTERNAL: get_master_key(key_id)
    activate EXTERNAL
    EXTERNAL --> MASTER: MasterKey
    deactivate EXTERNAL
else HSM
    MASTER -> HSM: get_master_key(key_id)
    activate HSM
    HSM --> MASTER: MasterKey
    deactivate HSM
end
deactivate MASTER

KMS -> KMS: generate_random_data_key()
note right: 256-bit AES key

alt Master Key in HSM
    KMS -> HSM: encrypt_data_key(\n  plaintext_data_key,\n  master_key_id\n)
    HSM --> KMS: encrypted_data_key
else Master Key Local/External
    KMS -> KMS: encrypt_data_key(\n  plaintext_data_key,\n  master_key\n)
end

KMS --> WRITE: DataKey{\n  plaintext: [32 bytes],\n  encrypted: [encrypted],\n  master_key_id\n}
deactivate KMS

WRITE -> WRITE: encrypt_data(\n  object_data,\n  plaintext_data_key\n)
note right: AES-256-GCM

WRITE -> WRITE: store_encrypted_data_and_key()
note right: Store encrypted data +\nencrypted data key

WRITE --> API: Success
deactivate WRITE

== Data Decryption ==
API -> WRITE: read_encrypted_object(key)
activate WRITE

WRITE -> WRITE: retrieve_encrypted_data_and_key()

WRITE -> KMS: decrypt_data_key(\n  encrypted_data_key,\n  master_key_id\n)
activate KMS

KMS -> MASTER: get_master_key(master_key_id)
activate MASTER
MASTER --> KMS: MasterKey
deactivate MASTER

alt Master Key in HSM
    KMS -> HSM: decrypt_data_key(\n  encrypted_data_key,\n  master_key_id\n)
    HSM --> KMS: plaintext_data_key
else Master Key Local/External
    KMS -> KMS: decrypt_data_key(\n  encrypted_data_key,\n  master_key\n)
end

KMS --> WRITE: plaintext_data_key
deactivate KMS

WRITE -> WRITE: decrypt_data(\n  encrypted_data,\n  plaintext_data_key\n)

WRITE -> WRITE: zero_memory(plaintext_data_key)
note right: Secure memory cleanup

WRITE --> API: plaintext_data
deactivate WRITE

== Master Key Rotation ==
... Time passes ...

KMS -> KMS: schedule_key_rotation()
activate KMS

KMS -> MASTER: create_new_master_key_version(\n  master_key_id\n)
activate MASTER
MASTER -> MASTER: generate_new_key()
MASTER -> MASTER: update_key_metadata()
MASTER --> KMS: NewMasterKey{version: 2}
deactivate MASTER

KMS -> KMS: mark_old_key_for_re_encryption()

note over KMS: Background process re-encrypts\nall data keys with new master key version

KMS -> KMS: schedule_old_key_deletion(\n  grace_period=30_days\n)

deactivate KMS

@enduml
```

## 6. Distributed Lock Acquisition

```plantuml
@startuml Distributed_Lock
title Distributed Lock Acquisition and Release

participant "Client\nService" as CLIENT
participant "Lock\nService" as LOCK
participant "Lock\nStore" as STORE
participant "Lock\nMonitor" as MONITOR

== Lock Acquisition ==
CLIENT -> LOCK: acquire_lock(\n  resource="bucket/object",\n  timeout=5s\n)
activate CLIENT
activate LOCK

LOCK -> LOCK: generate_lock_id()
LOCK -> LOCK: calculate_expiry(\n  timeout + safety_margin\n)

LOCK -> STORE: try_create_lock(\n  resource,\n  lock_id,\n  owner=client_id,\n  expires_at\n)
activate STORE

alt Lock Available
    STORE -> STORE: atomic_create(\n  key=resource,\n  value=lock_data\n)
    STORE --> LOCK: LockCreated
    
    LOCK -> MONITOR: register_lock(\n  lock_id,\n  callback\n)
    activate MONITOR
    MONITOR -> MONITOR: start_lease_renewal_timer()
    MONITOR --> LOCK: Registered
    deactivate MONITOR
    
    LOCK --> CLIENT: Lock{\n  id,\n  resource,\n  expires_at\n}
else Lock Already Exists
    STORE --> LOCK: LockExists{\n  owner,\n  expires_at\n}
    
    alt Lock Expired
        LOCK -> STORE: delete_expired_lock(resource)
        STORE --> LOCK: Deleted
        LOCK -> STORE: try_create_lock(...)
        STORE --> LOCK: LockCreated
        LOCK --> CLIENT: Lock{...}
    else Lock Still Valid
        LOCK -> LOCK: wait_or_retry()
        
        alt Timeout Exceeded
            LOCK --> CLIENT: Error::Timeout
        else Retry
            LOCK -> STORE: try_create_lock(...)
            note right: Retry with backoff
        end
    end
end
deactivate STORE

== Lock Usage ==
CLIENT -> CLIENT: perform_protected_operation()

== Lease Renewal ==
loop Every lease_interval (e.g., 1s)
    MONITOR -> MONITOR: check_lock_expiry()
    
    alt Lock expiring soon
        MONITOR -> LOCK: renew_lock(lock_id)
        activate LOCK
        LOCK -> STORE: update_expiry(\n  lock_id,\n  new_expires_at\n)
        activate STORE
        STORE -> STORE: atomic_update()
        STORE --> LOCK: Updated
        deactivate STORE
        LOCK --> MONITOR: Renewed
        deactivate LOCK
    end
end

== Lock Release ==
CLIENT -> LOCK: release_lock(lock)
LOCK -> MONITOR: unregister_lock(lock_id)
activate MONITOR
MONITOR -> MONITOR: stop_lease_renewal()
MONITOR --> LOCK: Unregistered
deactivate MONITOR

LOCK -> STORE: delete_lock(\n  resource,\n  lock_id,\n  owner\n)
activate STORE

alt Lock Still Owned by Client
    STORE -> STORE: atomic_delete(\n  if owner matches\n)
    STORE --> LOCK: Deleted
else Lock Stolen/Expired
    STORE --> LOCK: NotFound
end
deactivate STORE

LOCK --> CLIENT: Released
deactivate LOCK
deactivate CLIENT

== Deadlock Prevention ==
note over LOCK: Lock acquisition timeout\nprevents indefinite waiting

note over MONITOR: Automatic lease expiry\nensures locks don't persist\nif holder crashes

@enduml
```

## Summary

These sequence diagrams illustrate the key workflows in RustFS:

1. **Object Upload**: Complete flow from authentication to storage
2. **Object Download**: Shard collection, reconstruction, and decryption
3. **Authentication**: Signature validation and session management
4. **Multipart Upload**: Concurrent part uploads and assembly
5. **Key Management**: Envelope encryption and key rotation
6. **Distributed Locking**: Coordination and deadlock prevention

Each diagram shows the interaction between components and the flow of control through the system.

---

**Next**: See [Data Flow Diagrams](07-data-flow.md) for data movement patterns.
