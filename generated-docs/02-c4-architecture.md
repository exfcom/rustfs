# C4 Architecture Diagrams

## Introduction

This document presents the RustFS architecture using the C4 model (Context, Containers, Components, and Code). The C4 model provides a hierarchical view of the system at different levels of abstraction.

## Level 1: System Context Diagram

The context diagram shows how RustFS fits into the overall IT environment and its interactions with external systems and users.

```plantuml
@startuml RustFS_System_Context
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Context.puml

title RustFS System Context Diagram

Person(client_user, "Application User", "Uses S3 client applications to store and retrieve objects")
Person(admin_user, "System Administrator", "Manages RustFS configuration, users, and policies")
Person(ops_user, "Operations Team", "Monitors system health and performance")

System(rustfs, "RustFS", "High-performance distributed object storage system with S3 compatibility")

System_Ext(s3_client, "S3 Client", "AWS CLI, SDKs, or third-party tools")
System_Ext(prometheus, "Prometheus", "Metrics collection and alerting")
System_Ext(jaeger, "Jaeger", "Distributed tracing backend")
System_Ext(external_kms, "External KMS", "AWS KMS, Vault, or other KMS providers")
System_Ext(ldap, "LDAP/AD", "Corporate identity provider")
System_Ext(notification_target, "Notification Targets", "Kafka, NATS, Webhooks, etc.")

Rel(client_user, s3_client, "Uses")
Rel(s3_client, rustfs, "S3 API requests", "HTTPS")
Rel(admin_user, rustfs, "Administers", "HTTPS/Admin API")
Rel(ops_user, prometheus, "Views metrics")
Rel(ops_user, jaeger, "Traces requests")

Rel(rustfs, prometheus, "Exports metrics", "HTTP")
Rel(rustfs, jaeger, "Sends traces", "gRPC")
Rel(rustfs, external_kms, "Key operations", "HTTPS")
Rel(rustfs, ldap, "Authentication", "LDAP/S")
Rel(rustfs, notification_target, "Event notifications", "Various protocols")

SHOW_LEGEND()
@enduml
```

### Key External Interactions

| External System | Purpose | Protocol |
|----------------|---------|----------|
| S3 Clients | Object storage operations | HTTPS (S3 API) |
| Prometheus | Metrics collection | HTTP (pull) |
| Jaeger | Distributed tracing | gRPC |
| External KMS | Encryption key management | HTTPS |
| LDAP/AD | User authentication | LDAP/S |
| Notification Targets | Event streaming | Kafka, NATS, HTTP |

## Level 2: Container Diagram

The container diagram shows the major runtime containers (applications, data stores, microservices) within RustFS.

```plantuml
@startuml RustFS_Container_Diagram
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Container.puml

title RustFS Container Diagram

Person(user, "User", "S3 client user")
Person(admin, "Admin", "System administrator")

System_Boundary(rustfs_system, "RustFS") {
    Container(s3_api, "S3 API Server", "Axum", "Handles S3-compatible REST API requests")
    Container(admin_api, "Admin API Server", "madmin", "Management and administration interface")
    Container(mcp_server, "MCP Server", "QUIC", "High-performance object operations")
    
    Container(iam_service, "IAM Service", "rustfs-iam", "Authentication and authorization")
    Container(kms_service, "KMS Service", "rustfs-kms", "Key management and encryption")
    Container(policy_engine, "Policy Engine", "rustfs-policy", "Policy evaluation")
    Container(notify_service, "Notification Service", "rustfs-notify", "Event notifications")
    
    Container(ecstore, "Storage Engine", "rustfs-ecstore", "Erasure coding storage")
    Container(metadata_store, "Metadata Store", "rustfs-filemeta", "Object metadata management")
    Container(lock_service, "Lock Service", "rustfs-lock", "Distributed locking")
    
    ContainerDb(storage_backend, "Storage Backend", "File System", "Physical storage for data shards")
    ContainerDb(config_store, "Configuration", "Files/Env", "System configuration")
}

System_Ext(prometheus, "Prometheus", "Metrics")
System_Ext(jaeger, "Jaeger", "Traces")
System_Ext(external_kms, "External KMS", "Key management")

Rel(user, s3_api, "S3 API", "HTTPS")
Rel(admin, admin_api, "Admin API", "HTTPS")
Rel(user, mcp_server, "QUIC operations", "QUIC")

Rel(s3_api, iam_service, "Authenticate/Authorize")
Rel(admin_api, iam_service, "Authenticate/Authorize")
Rel(s3_api, policy_engine, "Check policies")
Rel(iam_service, policy_engine, "Evaluate policies")

Rel(s3_api, ecstore, "Store/Retrieve objects")
Rel(mcp_server, ecstore, "Store/Retrieve objects")
Rel(ecstore, metadata_store, "Manage metadata")
Rel(ecstore, kms_service, "Encrypt/Decrypt")
Rel(ecstore, storage_backend, "Read/Write shards")
Rel(ecstore, lock_service, "Acquire locks")

Rel(s3_api, notify_service, "Publish events")
Rel(ecstore, notify_service, "Storage events")

Rel(kms_service, external_kms, "Key operations", "HTTPS")
Rel_Back(config_store, s3_api, "Load config")
Rel_Back(config_store, iam_service, "Load config")

Rel(s3_api, prometheus, "Metrics", "HTTP")
Rel(s3_api, jaeger, "Traces", "gRPC")
Rel(ecstore, prometheus, "Metrics", "HTTP")

SHOW_LEGEND()
@enduml
```

### Container Descriptions

| Container | Technology | Responsibilities |
|-----------|------------|------------------|
| **S3 API Server** | Axum | S3 REST API, request routing, response formatting |
| **Admin API Server** | madmin | System administration, user management, configuration |
| **MCP Server** | QUIC | High-performance operations, streaming, multiplexing |
| **IAM Service** | rustfs-iam | User authentication, role management, session handling |
| **KMS Service** | rustfs-kms | Key generation, rotation, envelope encryption |
| **Policy Engine** | rustfs-policy | Policy parsing, evaluation, caching |
| **Notification Service** | rustfs-notify | Event generation, filtering, multi-target delivery |
| **Storage Engine** | rustfs-ecstore | Erasure coding, shard management, reconstruction |
| **Metadata Store** | rustfs-filemeta | Object metadata, versioning, indexing |
| **Lock Service** | rustfs-lock | Distributed coordination, lease management |

## Level 3: Component Diagram - S3 API Server

Detailed view of the S3 API Server container showing its internal components.

```plantuml
@startuml RustFS_S3_API_Components
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title RustFS S3 API Server - Component Diagram

Container_Boundary(s3_api, "S3 API Server") {
    Component(router, "API Router", "Axum Router", "Routes requests to handlers")
    Component(middleware, "Middleware Stack", "Tower", "Authentication, logging, metrics")
    
    Component(bucket_handler, "Bucket Handler", "Rust", "Bucket operations (create, delete, list)")
    Component(object_handler, "Object Handler", "Rust", "Object operations (get, put, delete)")
    Component(multipart_handler, "Multipart Handler", "Rust", "Multipart upload operations")
    Component(versioning_handler, "Versioning Handler", "Rust", "Object versioning")
    Component(lifecycle_handler, "Lifecycle Handler", "Rust", "Lifecycle management")
    
    Component(xml_processor, "XML Processor", "quick-xml", "XML request/response serialization")
    Component(signature_validator, "Signature Validator", "rustfs-signer", "SigV4 verification")
    Component(error_handler, "Error Handler", "Rust", "Error response formatting")
}

Component_Ext(iam_service, "IAM Service", "Authentication")
Component_Ext(policy_engine, "Policy Engine", "Authorization")
Component_Ext(ecstore, "Storage Engine", "Data operations")
Component_Ext(metadata_store, "Metadata Store", "Metadata operations")
Component_Ext(notify_service, "Notification Service", "Events")

Rel(router, middleware, "Processes through")
Rel(middleware, bucket_handler, "Routes bucket ops")
Rel(middleware, object_handler, "Routes object ops")
Rel(middleware, multipart_handler, "Routes multipart ops")
Rel(middleware, versioning_handler, "Routes versioning ops")
Rel(middleware, lifecycle_handler, "Routes lifecycle ops")

Rel(middleware, signature_validator, "Validates signature")
Rel(signature_validator, iam_service, "Verify credentials")

Rel(bucket_handler, policy_engine, "Check permissions")
Rel(object_handler, policy_engine, "Check permissions")

Rel(bucket_handler, metadata_store, "Bucket metadata")
Rel(object_handler, ecstore, "Store/retrieve objects")
Rel(object_handler, metadata_store, "Object metadata")
Rel(multipart_handler, ecstore, "Upload parts")
Rel(multipart_handler, metadata_store, "Part tracking")

Rel(bucket_handler, xml_processor, "Parse/generate XML")
Rel(object_handler, xml_processor, "Parse/generate XML")

Rel(object_handler, notify_service, "Object events")
Rel(bucket_handler, notify_service, "Bucket events")

Rel(error_handler, xml_processor, "Format error XML")

SHOW_LEGEND()
@enduml
```

## Level 3: Component Diagram - Storage Engine

Detailed view of the Storage Engine (ecstore) showing its internal components.

```plantuml
@startuml RustFS_Storage_Engine_Components
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title RustFS Storage Engine (ecstore) - Component Diagram

Container_Boundary(ecstore, "Storage Engine") {
    Component(write_coordinator, "Write Coordinator", "Rust", "Coordinates write operations")
    Component(read_coordinator, "Read Coordinator", "Rust", "Coordinates read operations")
    
    Component(erasure_coder, "Erasure Coder", "Reed-Solomon", "Encodes/decodes data with parity")
    Component(shard_distributor, "Shard Distributor", "Rust", "Distributes shards across storage")
    Component(reconstruction_engine, "Reconstruction Engine", "Rust", "Reconstructs data from shards")
    
    Component(checksum_validator, "Checksum Validator", "rustfs-checksums", "Validates data integrity")
    Component(bitrot_scanner, "Bitrot Scanner", "Rust", "Continuous data validation")
    Component(compression, "Compression Engine", "Optional", "Data compression")
    
    Component(buffer_manager, "Buffer Manager", "Rust", "Manages I/O buffers")
    Component(io_scheduler, "I/O Scheduler", "rustfs-rio", "Schedules disk operations")
}

Component_Ext(kms_service, "KMS Service", "Encryption")
Component_Ext(metadata_store, "Metadata Store", "Metadata")
Component_Ext(lock_service, "Lock Service", "Coordination")
Component_Ext(storage_backend, "Storage Backend", "Physical storage")
Component_Ext(workers, "Worker Pool", "Background tasks")

Rel(write_coordinator, erasure_coder, "Encode data")
Rel(erasure_coder, shard_distributor, "Distribute shards")
Rel(shard_distributor, io_scheduler, "Schedule writes")
Rel(io_scheduler, storage_backend, "Write shards")

Rel(read_coordinator, shard_distributor, "Request shards")
Rel(shard_distributor, io_scheduler, "Schedule reads")
Rel(io_scheduler, storage_backend, "Read shards")
Rel(read_coordinator, reconstruction_engine, "Reconstruct data")
Rel(reconstruction_engine, erasure_coder, "Decode shards")

Rel(write_coordinator, checksum_validator, "Calculate checksums")
Rel(read_coordinator, checksum_validator, "Verify checksums")

Rel(write_coordinator, kms_service, "Encrypt data")
Rel(read_coordinator, kms_service, "Decrypt data")

Rel(write_coordinator, metadata_store, "Store metadata")
Rel(read_coordinator, metadata_store, "Retrieve metadata")

Rel(write_coordinator, lock_service, "Acquire write lock")
Rel(read_coordinator, lock_service, "Acquire read lock")

Rel(write_coordinator, buffer_manager, "Allocate buffers")
Rel(read_coordinator, buffer_manager, "Allocate buffers")

Rel(bitrot_scanner, workers, "Schedule scans")
Rel(bitrot_scanner, checksum_validator, "Validate data")
Rel(bitrot_scanner, reconstruction_engine, "Repair data")

Rel(write_coordinator, compression, "Compress data")
Rel(read_coordinator, compression, "Decompress data")

SHOW_LEGEND()
@enduml
```

## Level 3: Component Diagram - IAM Service

Detailed view of the IAM Service showing its internal components.

```plantuml
@startuml RustFS_IAM_Components
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title RustFS IAM Service - Component Diagram

Container_Boundary(iam, "IAM Service") {
    Component(auth_manager, "Authentication Manager", "Rust", "Authenticates users")
    Component(user_store, "User Store", "Rust", "User account management")
    Component(role_manager, "Role Manager", "Rust", "Role and permission management")
    Component(session_manager, "Session Manager", "Rust", "Session and token management")
    Component(credential_validator, "Credential Validator", "Rust", "Validates credentials")
    
    Component(ldap_connector, "LDAP Connector", "ldap3", "LDAP/AD integration")
    Component(oidc_connector, "OIDC Connector", "openid", "OpenID Connect integration")
    Component(token_generator, "Token Generator", "jsonwebtoken", "JWT generation/validation")
}

Component_Ext(policy_engine, "Policy Engine", "Policy evaluation")
Component_Ext(audit_service, "Audit Service", "Audit logging")
Component_Ext(cache, "Cache", "Session cache")
Component_Ext(ldap_server, "LDAP Server", "External directory")

Rel(auth_manager, credential_validator, "Validate credentials")
Rel(credential_validator, user_store, "Lookup user")
Rel(credential_validator, ldap_connector, "LDAP auth")
Rel(credential_validator, oidc_connector, "OIDC auth")

Rel(ldap_connector, ldap_server, "Authenticate", "LDAP/S")

Rel(auth_manager, session_manager, "Create session")
Rel(session_manager, token_generator, "Generate token")
Rel(session_manager, cache, "Store session")

Rel(auth_manager, role_manager, "Get user roles")
Rel(role_manager, policy_engine, "Get role policies")

Rel(auth_manager, audit_service, "Log auth events")
Rel(user_store, audit_service, "Log user changes")

SHOW_LEGEND()
@enduml
```

## Level 4: Code Diagram - Object Upload Flow

Detailed code-level view of the object upload workflow.

```plantuml
@startuml RustFS_Object_Upload_Code
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Component.puml

title RustFS Object Upload - Code Level Flow

participant "Client" as client
participant "ObjectHandler" as handler
participant "SignatureValidator" as sig
participant "IAMService" as iam
participant "PolicyEngine" as policy
participant "KMSService" as kms
participant "WriteCoordinator" as write_coord
participant "ErasureCoder" as ec
participant "ShardDistributor" as dist
participant "StorageBackend" as storage
participant "MetadataStore" as meta
participant "NotifyService" as notify

client -> handler: PUT /bucket/key\n(request body)
activate handler

handler -> sig: validate_signature(request)
activate sig
sig -> iam: verify_credentials(access_key)
activate iam
iam --> sig: User
deactivate iam
sig --> handler: SignatureValid
deactivate sig

handler -> policy: check_permission(user, "s3:PutObject", bucket, key)
activate policy
policy --> handler: Allow
deactivate policy

handler -> write_coord: write_object(bucket, key, data, metadata)
activate write_coord

alt Encryption enabled
    write_coord -> kms: encrypt(data, key_id)
    activate kms
    kms --> write_coord: encrypted_data, data_key
    deactivate kms
end

write_coord -> ec: encode(data, parity_shards, data_shards)
activate ec
ec --> write_coord: shards[]
deactivate ec

write_coord -> dist: distribute_shards(shards, storage_policy)
activate dist

loop for each shard
    dist -> storage: write_shard(shard_id, shard_data, location)
    activate storage
    storage --> dist: success
    deactivate storage
end

dist --> write_coord: shard_locations[]
deactivate dist

write_coord -> meta: store_metadata(bucket, key, metadata, shard_locations)
activate meta
meta --> write_coord: success
deactivate meta

write_coord --> handler: write_result
deactivate write_coord

handler -> notify: publish_event("ObjectCreated", bucket, key)
activate notify
notify --> handler: acknowledged
deactivate notify

handler --> client: 200 OK\nETag: "..."
deactivate handler

@enduml
```

## Crate Dependency Graph

Visual representation of dependencies between RustFS crates.

```plantuml
@startuml RustFS_Crate_Dependencies
!theme plain
skinparam componentStyle rectangle

package "API Layer" {
    [rustfs] as main
    [madmin]
    [mcp]
}

package "Core Services" {
    [iam]
    [kms]
    [policy]
    [notify]
    [appauth]
}

package "Storage Layer" {
    [ecstore]
    [filemeta]
    [lock]
    [rio]
    [targets]
}

package "Security" {
    [crypto]
    [signer]
    [checksums]
}

package "Utilities" {
    [common]
    [config]
    [utils]
    [ahm]
    [workers]
}

package "Protocols" {
    [protos]
    [s3select-api]
    [s3select-query]
    [zip]
}

package "Observability" {
    [obs]
    [audit]
}

' API Layer dependencies
main --> iam
main --> kms
main --> policy
main --> ecstore
main --> notify
main --> obs
main --> config

madmin --> iam
madmin --> config
madmin --> obs

mcp --> ecstore
mcp --> iam
mcp --> obs

' Core Services dependencies
iam --> policy
iam --> crypto
iam --> audit
iam --> common

kms --> crypto
kms --> audit
kms --> common

policy --> common
policy --> ahm

notify --> common
notify --> workers

appauth --> iam
appauth --> policy

' Storage Layer dependencies
ecstore --> filemeta
ecstore --> lock
ecstore --> kms
ecstore --> crypto
ecstore --> checksums
ecstore --> rio
ecstore --> targets
ecstore --> workers
ecstore --> common

filemeta --> lock
filemeta --> common
filemeta --> ahm

lock --> common

rio --> common

' Security dependencies
signer --> crypto
signer --> common

checksums --> crypto
checksums --> common

' Observability dependencies
obs --> common

audit --> notify
audit --> common

' Protocol dependencies
s3select-query --> s3select-api
s3select-query --> common

zip --> common

@enduml
```

## Deployment View

Physical deployment architecture for different deployment scenarios.

```plantuml
@startuml RustFS_Deployment
!include https://raw.githubusercontent.com/plantuml-stdlib/C4-PlantUML/master/C4_Deployment.puml

title RustFS Deployment Diagram - Kubernetes

Deployment_Node(k8s_cluster, "Kubernetes Cluster", "Cloud/On-premises") {
    Deployment_Node(namespace, "rustfs-namespace", "Kubernetes Namespace") {
        
        Deployment_Node(ingress, "Ingress Controller", "Nginx/Traefik") {
            Container(ingress_ctrl, "Load Balancer", "L7", "Routes traffic to services")
        }
        
        Deployment_Node(api_pods, "API Pod", "StatefulSet", "Replicas: 3+") {
            Container(rustfs_api, "RustFS API", "rustfs binary", "S3 API + Admin API")
            Container(mcp_server, "MCP Server", "rustfs-mcp", "QUIC operations")
        }
        
        Deployment_Node(storage_pods, "Storage Pod", "StatefulSet", "Replicas: N") {
            Container(storage_engine, "Storage Engine", "ecstore", "Erasure coding storage")
            ContainerDb(persistent_volume, "Persistent Volume", "PVC", "Object storage")
        }
        
        Deployment_Node(service_pods, "Service Pod", "Deployment", "Replicas: 2+") {
            Container(iam_svc, "IAM Service", "rustfs-iam", "Authentication")
            Container(kms_svc, "KMS Service", "rustfs-kms", "Key management")
            Container(notify_svc, "Notify Service", "rustfs-notify", "Events")
        }
    }
    
    Deployment_Node(monitoring, "Monitoring Stack", "Separate namespace") {
        Container(prometheus, "Prometheus", "Metrics collection")
        Container(grafana, "Grafana", "Visualization")
        Container(jaeger, "Jaeger", "Tracing")
    }
}

Deployment_Node(external, "External Services") {
    System_Ext(external_kms, "External KMS", "AWS KMS / Vault")
    System_Ext(ldap, "LDAP/AD", "Corporate directory")
}

Rel(ingress_ctrl, rustfs_api, "Routes requests", "HTTPS")
Rel(rustfs_api, storage_engine, "Store/retrieve", "Internal")
Rel(rustfs_api, iam_svc, "Authenticate", "Internal")
Rel(rustfs_api, kms_svc, "Encrypt/decrypt", "Internal")

Rel(storage_engine, persistent_volume, "Read/write shards")

Rel(rustfs_api, prometheus, "Export metrics", "HTTP")
Rel(storage_engine, prometheus, "Export metrics", "HTTP")
Rel(rustfs_api, jaeger, "Send traces", "gRPC")

Rel(kms_svc, external_kms, "Key operations", "HTTPS")
Rel(iam_svc, ldap, "Authentication", "LDAP/S")

SHOW_LEGEND()
@enduml
```

## Summary

This C4 architecture documentation provides multiple views of the RustFS system:

1. **Context**: External interactions and stakeholders
2. **Containers**: Runtime components and their relationships
3. **Components**: Internal structure of key containers
4. **Code**: Detailed workflows and interactions

Each level provides increasing detail while maintaining clear boundaries and responsibilities.

---

**Next**: See [Crate Architecture](03-crate-architecture.md) for detailed crate organization.
