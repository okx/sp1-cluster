---
name: "dependency"
description: "Upstream callers, inter-module deps, storage/middleware, external services"
---
# Dependency Map

## Upstream Callers

| Caller | Protocol | Entry Point |
|--------|----------|-------------|
| SP1 SDK clients (external) | gRPC over HTTP/2 (tonic, optional TLS) | `bin/network-gateway` `ProverNetwork` service: `request_proof`, `get_proof_request_status`, `get_proof_request_details`, `get_filtered_proof_requests`, `get_nonce`, `get_balance`, `get_program`, `create_program`; `ArtifactStore.create_artifact` (signed `b"create_artifact"`) |
| SP1 SDK clients (external) | HTTP PUT/GET | `bin/network-gateway/src/artifact_http.rs` `/artifacts/{type}/{id}` |
| SPN ProverNetwork (`rpc.mainnet.succinct.xyz`) | gRPC (tonic) | bin/bidder via `get_filtered_proof_requests` / `bid` / `get_nonce` / `get_owner`; bin/fulfiller via `get_filtered_proof_requests` / `fulfill_proof` / `fail_fulfillment` / `get_stdin_uri` / `get_nonce` / `get_owner` |
| Coordinator (`bin/coordinator`) | gRPC (tonic) | bin/api `ClusterService` via `ClusterServiceClient` on `COORDINATOR_CLUSTER_RPC` (default `http://127.0.0.1:50051`) |
| Fulfiller (`bin/fulfiller`) | gRPC (tonic) | bin/api `ClusterService` via `ClusterServiceClient` on `FULFILLER_CLUSTER_RPC` |
| Network Gateway (`bin/network-gateway`) | gRPC (tonic) | bin/api `ClusterService` via `ClusterServiceClient` on `GATEWAY_CLUSTER_RPC` |
| Worker nodes (`bin/node`) | gRPC streaming (tonic) | Coordinator `WorkerService`: `open` / `close` / `heartbeat` / `create_task` / `complete_task` / `fail_task` / `open_sub` / `update_sub` / `ack_sub` / `complete_proof` / `fail_proof` / `subscribe_task_messages` / `send_task_message` on `NODE_COORDINATOR_RPC` (default `http://[::1]:50051`) |
| External HTTP clients | HTTP | bin/api axum `/` + `/healthz` on `API_HTTP_ADDR` (default `127.0.0.1:3000`); each binary also exposes Prometheus metrics via `spn-metrics` |
| ECS Task metadata endpoint (external) | HTTP | bin/node fetches `ECS_CONTAINER_METADATA_URI_V4` to derive `worker_id` / cluster name |

## Inter-Module Dependencies

| From | To | Mechanism |
|------|----|-----------|
| `bin/api` → `sp1-cluster-common` (`proto::cluster_service_server`) | gRPC server (tonic) + tonic-web | bin/api exposes `ClusterService`; serializes `DbProofRequest` → proto `ProofRequest` via sqlx |
| `bin/coordinator` → `sp1-cluster-common` (`client::ClusterServiceClient`) | gRPC client | Coordinator pulls/updates proof requests in bin/api |
| `bin/coordinator` → `sp1-cluster-common` (`proto::worker_service_server`) | gRPC server (tonic + tonic-web `GrpcWebLayer`) | Hosts `WorkerService` streaming endpoints consumed by workers |
| `bin/node` → `sp1-cluster-worker::client::WorkerServiceClient` | gRPC client (tonic) | Streams tasks; uses `reconnect_with_backoff` from `sp1-cluster-common::client` |
| `bin/node` → `sp1-cluster-artifact` (`S3ArtifactClient` / `RedisArtifactClient`) | In-process | Selected via `NODE_ARTIFACT_STORE`; passed to SP1 worker builder |
| `bin/node` → `sp1-prover` (`cpu_worker_builder` / `gpu_worker_builder` via `sp1-gpu-prover`) | In-process | Wraps prover with cluster artifact + worker clients |
| `bin/fulfiller` → `sp1-cluster-fulfillment` (`config`, `grpc`, `run`, `network`, `Fulfiller`) | In-process | `run_fulfiller` wires `Fulfiller<A,N>` over chosen artifact backend + `ProverNetworkClient` |
| `bin/fulfiller` → `sp1-cluster-common` (`client::ClusterServiceClient`) | gRPC client | Reads/updates `ProofRequest` state in bin/api |
| `bin/fulfiller` → `sp1-cluster-artifact` | In-process | Selected via `FULFILLER_CLUSTER_ARTIFACT_STORE` |
| `bin/fulfiller` → `spn-network-types::prover_network_client::ProverNetworkClient` | gRPC client (tonic) | Pulls schedulable/cancelable requests; submits fulfill / fail_fulfillment; fetches private stdin URIs |
| `bin/fulfiller` → `spn-artifacts` | HTTPS download | `MainnetFulfiller::download_artifact` uses `Artifact::download_raw_from_uri_par` to pull network-provided program/stdin from network-side S3 region |
| `bin/bidder` → `spn-network-types::prover_network_client::ProverNetworkClient` | gRPC client (tonic) | Polls biddable requests + submits bids |
| `bin/bidder` → `spn-metrics`, `spn-utils` | In-process | Metrics server + logger |
| `bin/network-gateway` → `sp1-cluster-common` (`client::ClusterServiceClient` + proto) | gRPC client + status mapping | Bridges SDK requests to cluster `ProofRequest` CRUD; mapping table in `bin/network-gateway/src/status.rs` |
| `bin/network-gateway` → `sp1-cluster-artifact` (`ArtifactClient` + `CompressedUpload`) | In-process | HTTP artifact proxy uses `upload_raw_compressed` for PUT and `download_raw` for GET |
| `bin/network-gateway` → `sp1-sdk::network::proto::base` (`ProverNetwork` + `ArtifactStore`) | tonic-generated traits | Implements SDK-facing gRPC services; unsupported RPCs return `Status::unimplemented` |
| `bin/network-gateway` → `ProgramStore` (Filesystem or InMemory) | In-process trait | Durable program registry outside the ephemeral artifact lifecycle (ELF persists indefinitely until deleted) |
| `bin/network-gateway` → `auth::Auth` | In-process | EIP-191 signature recovery for `create_program` / `request_proof` / `create_artifact` |
| `sp1-cluster-artifact::s3` → `s3_sdk::S3SDKClient` / `s3_rest::S3RestClient` | In-process | Transport adapters selected via `S3DownloadMode` |
| `sp1-cluster-artifact::s3` → `sp1_prover_types::ArtifactClient` | Trait impl | Provides `upload_raw` / `download_raw` with zstd level 3 compression |
| `sp1-cluster-artifact::redis` → `sp1_prover_types::ArtifactClient` + `ShardPermit` | Trait impl | zstd level 0 + admission control; `validate_config` must succeed at startup |
| `sp1-cluster-artifact::redis` → `deadpool-redis::Pool` (one per shard) | Connection pool | `hash_string` FNV-1a routes id → shard |
| `crates/worker/src/client.rs` → `sp1_cluster_common::client::reconnect_with_backoff` | Helper | Builds tonic `Endpoint` with TLS + http2 keepalive + indefinite reconnect |
| `crates/worker/src/client.rs` → `sp1_prover::worker::WorkerClient` | Trait impl | Provides `submit_task`, `complete_task`, `subscriber`, `subscribe_task_messages`, `send_task_message`, `complete_proof` |
| `crates/fulfillment/src/grpc.rs` → `tonic::transport::Endpoint` | In-process | `configure_endpoint` sets 10s timeout, 5s connect, http2 keepalive 10s/10s, tcp keepalive 30s |
| `sp1-cluster-common` → tonic-generated stubs | gRPC client/server | `ClusterServiceClient`/`Server`, `WorkerServiceClient`/`Server` consumed across binaries |

## Storage and Middleware

| Component | Type | Usage |
|-----------|------|-------|
| Postgres (`sqlx::PgPool`) | Persistent | `proof_requests` CRUD; migrations from `../../migrations` when `API_AUTO_MIGRATE=true`. `bin/api` is the sole owner; all other services go through bin/api gRPC. |
| AWS S3 (`aws-sdk-s3` + `aws-config`) | Persistent (zstd level 3) | `S3ArtifactClient` (`crates/artifact/src/s3.rs`) — 32 MiB multipart, parallel range downloads via `S3DownloadMode::AwsSDK` or `REST`. Object keys: `programs/`, `stdins/`, `proofs/`, `private-stdins/`; circuits at bucket root (`<id>-{groth16\|plonk}.tar.gz`). |
| Redis (multi-node, `deadpool-redis`) | Ephemeral (zstd level 0) | `RedisArtifactClient` (`crates/artifact/src/redis.rs`); 4h TTL (`ARTIFACT_TIMEOUT_SECONDS=14400`) except `ArtifactType::Program`; chunked at 32 MiB via `<key>:chunks` hash; `refs:{id}` set for refcount; FNV-1a sharding. |
| Local filesystem (`FilesystemProgramStore`) | Persistent (atomic) | bin/network-gateway program registry under `GATEWAY_PROGRAM_STORE_DIR`; writes via `.tmp` + `fs::rename`; `memory` backend loses programs on restart. |
| In-memory `DashMap<address, AtomicU64>` | Non-durable | Per-address nonce counter in `ProverNetworkImpl::next_nonce`; reset on gateway restart. |

## External Services

| Service | SDK/Client | Purpose |
|---------|-----------|---------|
| SPN ProverNetwork | `spn-network-types::prover_network_client::ProverNetworkClient` | Mainnet bidding (bidder) + fulfillment (fulfiller) RPCs over tonic |
| SPN `spn-artifacts` | `Artifact::download_raw_from_uri_par` | Pulls SPN-provided program/stdin from network-side S3 region (`FULFILLER_NETWORK_S3_REGION`, default `us-east-2`) |
| AWS S3 | `aws-sdk-s3` + `aws-config` | Cluster artifact storage (`FULFILLER_CLUSTER_S3_*`, `NODE_S3_*`, `GATEWAY_S3_*`) |
| AWS KMS | `NetworkSigner::aws_kms` | Fulfiller signing when `FULFILLER_USE_AWS_KMS=true` |
| rustls + ring crypto provider | `rustls::crypto::ring::default_provider` | TLS for all outbound tonic gRPC channels |
| Pyroscope | `pyroscope` crate + `pyroscope_pprofrs` | bin/node continuous profiling (optional, configured by `PYROSCOPE_*` env vars) |
| OpenTelemetry collector | `opentelemetry_sdk` | Tracing + metrics via `sp1_cluster_common::logger::init` |
| SP1 SDK install | `try_install_circuit_artifacts("groth16"\|"plonk")` | CPU workers download circuit artifacts before opening the worker stream |

## Prohibited Patterns

- [Dependency] `bin/api` owns Postgres connectivity — every other binary must reach `proof_requests` via `ClusterServiceClient`, never via direct `PgPool`.
- [Dependency] `bin/coordinator` must never speak to SPN — fulfiller and bidder are the only SPN-facing binaries.
- [Dependency] Crates must not depend on binaries — `crates/*` are reusable libraries; binaries depend on crates, not the other way around.
- [Dependency] `crates/common` must never depend on policy/coordinator-specific types — keep it free of bin/coordinator imports so cross-binary clients stay lean.
- [Dependency] `crates/artifact` must not embed cluster API or coordinator clients — it is an artifact-only backend abstraction.
- [Rule] `bin/network-gateway` must never call coordinator or worker RPC — gateway is a one-way SDK→cluster bridge through `ClusterServiceClient` + `ArtifactClient` only.
- [Rule] Cross-bucket artifact moves must use `CompressedUpload::upload_raw_compressed`; calling `ArtifactClient::upload_raw` for already-zstd bytes is prohibited (double-compression).
- [Rule] No new `tonic::transport::Endpoint` may be constructed inline; cluster-internal uses must route through `crates/common/src/client.rs::reconnect_with_backoff`; SPN-facing must use `crates/fulfillment/src/grpc.rs::configure_endpoint` (or the identical copy in `bin/bidder/src/grpc.rs`).
