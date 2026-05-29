---
name: "architecture-overview"
description: "Layer definitions, allowed/prohibited call directions, service responsibilities"
---
# Architecture Overview

sp1-cluster is a Rust workspace of seven binaries and five shared crates. The runtime data plane is six gRPC services (tonic): bin/api hosts `ClusterService`, bin/coordinator hosts `WorkerService`, bin/network-gateway hosts SDK-facing `ProverNetwork` + `ArtifactStore`, bin/node consumes `WorkerService`, bin/fulfiller and bin/bidder act as outbound SPN clients.

## Layer Definitions

| Layer | Responsibilities | Allowed Calls | Prohibited Calls |
|-------|-----------------|---------------|------------------|
| API (`bin/api`) | Hosts `ClusterService` gRPC + `/healthz` HTTP, owns the Postgres `proof_requests` table via `sqlx::PgPool`, runs migrations on `API_AUTO_MIGRATE=true`. | Postgres (sqlx), `tonic_web` for browser clients | Must never call coordinator, worker, or SPN; must not perform cluster orchestration. |
| Coordinator (`bin/coordinator`, `bin/coordinator/src/policy`) | Owns in-memory `Coordinator<P>` (proofs, tasks, workers, subscribers, task channels); hosts `WorkerService` gRPC server; polls bin/api via `ClusterServiceClient`; dispatches tasks to workers via per-worker mpsc channels; runs heartbeat + periodic cleanup. | bin/api (`ClusterServiceClient`), worker mpsc channels, OpenTelemetry/Prometheus | Must not touch Postgres directly, must not speak to SPN, must not execute proving work. |
| Worker / Node (`bin/node`, `crates/worker`) | Connects to coordinator over gRPC streaming via `WorkerServiceClient`; downloads artifacts via `ArtifactClient`; runs `SP1ClusterWorker::run_task` to dispatch `TaskType` to per-task handlers. | Coordinator gRPC, S3/Redis artifact backends, SP1 prover internals | Must not write to API DB, must not bid/fulfill on SPN, must not assign tasks to other workers. |
| Fulfiller (`bin/fulfiller`, `crates/fulfillment`) | Polls bin/api via `ClusterServiceClient`, talks to SPN via `ProverNetworkClient`, copies artifacts cluster↔network via `CompressedUpload::upload_raw_compressed`, signs via local key or AWS KMS. | bin/api (`ClusterServiceClient`), SPN `ProverNetworkClient`, S3/Redis | Must not call coordinator/worker RPC, must not bid, must not produce proofs. |
| Bidder (`bin/bidder`) | Polls SPN `ProverNetworkClient` every `REFRESH_INTERVAL_SEC=3s`, evaluates `can_fulfill_proof` against throughput + concurrency + per-mode buffers, signs `BidRequestBody` with `PrivateKeySigner`. | SPN `ProverNetworkClient`, spn-metrics | Must not touch cluster API/coordinator/workers/artifacts, must not fulfill proofs. |
| Network Gateway (`bin/network-gateway`) | Terminates SP1 SDK's `ProverNetwork` + `ArtifactStore` gRPC contract, proxies to bin/api via `ClusterServiceClient`, exposes `/artifacts/{type}/{id}` HTTP proxy backed by `ArtifactClient`, signs auth on `request_proof` / `create_program` / `create_artifact`. | bin/api (`ClusterServiceClient`), `ArtifactClient` (S3/Redis), local `ProgramStore` (fs/memory), in-memory nonce map | Must not run proving, must not bid, must not bypass `Auth`, must not call coordinator/worker RPC. |
| CLI (`bin/cli`) | Developer tooling for benchmarks and vkey generation via `crates/utils::request_proof`. | bin/api (`ClusterServiceClient`), SP1 SDK | Not part of runtime data plane. |
| Shared Crates (`crates/{artifact, common, fulfillment, utils, worker}`) | `artifact`: `ArtifactClient` + `CompressedUpload` backends; `common`: proto re-exports + `ClusterServiceClient` + logger + consts + util; `fulfillment`: `Fulfiller` + `FulfillmentNetwork`; `utils`: high-level `request_proof` helpers; `worker`: `SP1ClusterWorker` + task implementations. | Workspace internal | Must not depend on binaries; `common` must not depend on policy/coordinator types; `artifact` must not embed cluster API/coordinator clients. |

## Service Responsibilities

| Module | Responsibility | NOT Responsible For |
|--------|----------------|---------------------|
| `bin/api/src/service.rs` (`ClusterServiceImpl`) | gRPC ClusterService: `proof_request_{create, cancel, update, get, list}` against Postgres; converts `DbProofRequest` → proto `ProofRequest`. | Task assignment, SPN, coordinator RPC. |
| `bin/api/src/main.rs` | Loads `.env`, opens `PgPool`, optionally runs sqlx migrations, starts tonic gRPC (with `tonic_web`) on `API_GRPC_ADDR` and axum HTTP `/healthz` on `API_HTTP_ADDR`. | Task assignment, SPN client, artifact store. |
| `bin/coordinator/src/lib.rs` (`Coordinator<P>`) | In-memory cluster state, `create_proof / create_task / complete_task / fail_task / fail_proof` flows, per-worker channels, heartbeat handling, subscriber redelivery, metrics; depends on `P: AssignmentPolicy`. | Persistence, executing proofs, SPN. |
| `bin/coordinator/src/server.rs` (`GenericWorkerService`) | gRPC WorkerService surface (open/close/heartbeat/create_task/complete_task/fail_task/create_proof/cancel_proof/open_sub/update_sub/ack_sub/subscribe_task_messages); jemalloc; tonic-web; spawn proof status, claimer, heartbeat, periodic tasks. | Direct state mutation — delegates to `Coordinator`. |
| `bin/coordinator/src/cluster.rs` | Glue between `Coordinator` and bin/api: `spawn_proof_status_task` forwards `ProofResult` into `ProofRequestUpdate`; `spawn_proof_claimer_task` polls API for `Pending+!handled` and calls `Coordinator::create_proof`; fails proofs no longer in DB. | SPN, executing tasks. |
| `bin/coordinator/src/policy/{balanced, default}.rs` | `AssignmentPolicy` impls — `BalancedPolicy` (per-proof GPU fairness, default); `DefaultPolicy` (FIFO). | gRPC, storage. |
| `bin/node/src/main.rs` | Builds CPU or GPU worker, selects `S3` or `Redis` artifact backend via `NODE_ARTIFACT_STORE`, connects to coordinator, dispatches `NewTask` to `SP1ClusterWorker::run_task`, drains on shutdown (`DRAIN_TIMEOUT=30min`), aborts overlong tasks (`TASK_TIMEOUT=6h`), optional Pyroscope profiling. | bin/api or SPN calls, task assignment, bidding. |
| `crates/worker/src/lib.rs` (`SP1ClusterWorker`) | Wraps `SP1Worker`; dispatches `TaskType` → per-task handler under `tasks/`; OpenTelemetry tracing per task. | gRPC server, worker registration. |
| `bin/fulfiller/src/main.rs` + `crates/fulfillment/src/lib.rs` (`Fulfiller`) | Loop `submit_requests → fail_requests → cancel_requests → schedule_requests`; talks to bin/api + SPN; copies artifacts via `CompressedUpload`; deterministic `request_probability` filter; tags work with `name`/`scheduled_by`. | Task assignment, executing proofs, bidding. |
| `bin/fulfiller/src/mainnet.rs` (`MainnetFulfiller`) | `FulfillmentNetwork` impl against `spn_network_types::prover_network_client::ProverNetworkClient`; signs `FulfillProofRequestBody`; maps cancel/fail to `fail_request_with_error`. | Cluster artifacts. |
| `bin/bidder/src/main.rs` + `lib.rs` (`Bidder`) | Polls SPN every 3s; evaluates capacity + per-mode toggles + `aggressive_mode`; signs `BidRequestBody` with `PrivateKeySigner`; emits `spn_metrics`. | Cluster API/coordinator/workers/artifacts. |
| `bin/network-gateway/src/lib.rs` | Builds `Auth`, `ProgramStore` (fs/memory), wires `ProverNetworkImpl` + `ArtifactStoreImpl` over `ClusterServiceClient` + `ArtifactClient`; serves tonic with `tonic_web` + HTTP artifact proxy; gRPC max message size 64 MiB. | Proving, bidding, SPN calls. |
| `bin/cli/src/main.rs` | Clap CLI exposing `bench` + `vk-gen`; loads `.env` and SP1 SDK logger. | Server-side responsibilities. |
| `crates/utils/src/lib.rs` | High-level `request_proof` helpers for CLI: creates `ProofRequestCreateRequest` via `ClusterServiceClient`, uploads artifacts, polls `get_proof_request`, downloads completed proof. | Internal use by API/coordinator/node/fulfiller/bidder/gateway. |
| `crates/artifact/src/lib.rs` | Re-exports `ArtifactClient`/`ArtifactId`/`ArtifactType`/`InMemoryArtifactClient`; defines `CompressedUpload` trait. | Cluster proto, coordinator state. |
| `crates/common/src/lib.rs` | Aggregates `client` (`ClusterServiceClient`), `consts` (`task_weight`, `CONTROLLER_WEIGHT`, …), `logger` (OTLP init), `proto` (cluster + worker re-exports), `util` (`backoff_retry`, `get_private_ip`). | Business logic. |
