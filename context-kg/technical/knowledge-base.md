---
name: "knowledge-base"
description: "Highest-authority rules — all Skills defer to this on conflicts"
---
# Knowledge Base

> This is the highest-weight file in the knowledge base. All Skills that support
> context-kg defer to this file when conflicts arise with general AI knowledge.

Entries below are sorted A→Z by source file path. Constraint words (`must`, `never`, `always`, `required`, `prohibited`) appear inline.

## Data Type Constraints

- [Rule] `bin/api/src/service.rs`: All `proof_request_*` mutations and reads must execute against `Arc<sqlx::PgPool>` via `sqlx::query` / `QueryBuilder`; no other store is permitted. — Reason: bin/api is the sole owner of Postgres connectivity.
- [Rule] `bin/coordinator/src/cluster.rs`: `ControllerInputMetadata { stdin_private }` must be JSON-serialized into `inputs[5]` of the controller task data when claiming a proof; `inputs[3]` is `cycle_limit` and `inputs[4]` is `proof_nonce` (currently empty). — Reason: the worker controller decodes inputs positionally.
- [Rule] `bin/network-gateway/src/ids.rs`: `mint_request_id` must produce exactly `REQUEST_ID_LEN = 32` bytes; typeid string is zero-padded. — Reason: SDK decodes via `B256::from_slice` which requires a 32-byte slice.
- [Rule] `bin/network-gateway/src/ids.rs`: `proof_id_from_request_id` must strip trailing zero bytes before returning the cluster `proof_id`. — Reason: zero-padded bytes would not match the typeid stored in Postgres.
- [Rule] `bin/network-gateway/src/ids.rs`: `program_artifact_id` must be deterministic `"program_" + hex(vk_hash)`. — Reason: warm-cache lookup via `ArtifactClient::exists` depends on stable ids across SDK calls.
- [Rule] `crates/artifact/src/redis.rs`: Multi-chunk Redis uploads must use `HSET` + `EXPIRE` on the hash key; `HSETEX` is prohibited because managed Redis (e.g. AWS ElastiCache) does not support it. — Reason: production reachability.
- [Rule] `crates/artifact/src/redis.rs`: `ArtifactType::Program` artifacts must never have a TTL set (no `SET ... EX`, no `EXPIRE`). — Reason: programs are deterministic-keyed and must persist for the warm-cache contract.
- [Rule] `crates/artifact/src/s3.rs`: S3 uploads must zstd-compress at level 3 before storage; downloads must `zstd::decode_all`. — Reason: persistent storage; level 3 chosen for size/CPU trade-off.
- [Rule] `crates/common/src/consts.rs`: Task RAM weight must always come from `task_weight(task_type)`; per-task tunables must be loaded from `WORKER_*_WEIGHT` env vars via `lazy_static!`. — Reason: hardcoded weights desync the scheduler from operator overrides.
- [Rule] `crates/common/src/proto.rs`: Proto enums/messages must be imported through `sp1_cluster_common::proto` (re-export of `sp1_prover_types::cluster` + `::worker`), never directly from `sp1_prover_types`. — Reason: keeps a single re-export point so cluster-wide enum mappings stay coherent.
- [Rule] `crates/worker/src/tasks/types.rs`: `CoreBatchData`, `DeferredBatchData`, `ReduceBatchData` must always be passed via artifact bodies; never via the gRPC `TaskData` envelope. — Reason: payloads exceed the 64 MiB gRPC ceiling.
- [Rule] `migrations/20260513000000_stdin_private.sql`: `proof_requests.stdin_private` must be `BOOLEAN NOT NULL DEFAULT FALSE`. — Reason: backwards compatibility for pre-existing rows.

## Naming Constraints

- [Rule] `bin/network-gateway/src/ids.rs`: Artifact URI segments must round-trip exactly through `artifact_type_segment` ↔ `parse_artifact_type_segment` (unspecified|program|stdin|proof|groth16|plonk|private-stdin); network-only ArtifactTypes (Transaction/State/Export) must fold to `UnspecifiedArtifactType`. — Reason: HTTP path is stable wire surface.
- [Rule] `bin/network-gateway/src/ids.rs`: Minted proof_id strings must use the TypeID prefix `"req_"`. — Reason: SDK and tests assert the prefix.
- [Rule] `crates/common/src/consts.rs`: Worker capacity env vars must keep the `WORKER_*_WEIGHT` naming (`WORKER_CONTROLLER_WEIGHT`, `WORKER_GROTH16_WRAP_WEIGHT`, `WORKER_PLONK_WRAP_WEIGHT`, `WORKER_EXECUTE_ONLY_WEIGHT`, `WORKER_CORE_EXECUTE_WEIGHT`). — Reason: operators rely on the prefix.
- [Rule] `crates/worker/src/tasks/mod.rs`: New task handlers must live under `crates/worker/src/tasks/<name>.rs` as sibling modules and be wired into `SP1ClusterWorker::run_task` via a `TaskType::<Variant>` match arm. — Reason: dispatch is exhaustive by `TaskType`.

## Dependency Constraints

- [Rule] `bin/api/src/main.rs`: bin/api must connect to Postgres via `API_DATABASE_URL` and serve gRPC on `API_GRPC_ADDR` + HTTP `/healthz` on `API_HTTP_ADDR`; auto-migrate only when `API_AUTO_MIGRATE=true`. — Reason: operations contract for the canonical API binary.
- [Rule] `bin/api/src/service.rs`: API service must never call coordinator, worker, or SPN network; only collaborator is `Arc<PgPool>`. — Reason: layering contract that keeps API stateless about cluster operations.
- [Rule] `bin/coordinator/src/cluster.rs`: Coordinator must reach the API only through `ClusterServiceClient` at `COORDINATOR_CLUSTER_RPC` (default `http://127.0.0.1:50051`); no direct Postgres access. — Reason: bin/api owns the database.
- [Rule] `bin/coordinator/src/lib.rs`: `Coordinator` state must be guarded by `state.write_owned()` / `state.read()` and instrumented with `tracing::debug_span!("acquire_write"/"acquire")`; when `shutting_down`, new proof creation and proof failure must refuse. — Reason: lock visibility + clean shutdown.
- [Rule] `bin/coordinator/src/server.rs`: gRPC handlers must `tokio::spawn` long mutations and `await.unwrap()?` so RPC cancellation cannot tear down the coordinator. — Reason: prevent partial state mutations on client disconnect.
- [Rule] `bin/coordinator/src/server.rs`: `WorkerServiceServer` must set `tcp_keepalive(20s)`, `http2_keepalive_interval(15s)`, `http2_keepalive_timeout(35s)`, and enable `GrpcWebLayer`. — Reason: keep idle worker streams alive across NAT/LB.
- [Rule] `bin/fulfiller/src/main.rs`: Fulfiller must install `rustls::crypto::ring::default_provider` before opening TLS gRPC; artifact backend must be one of `s3` | `redis`; Redis path must call `validate_config()` before running. — Reason: startup safety.
- [Rule] `bin/network-gateway/src/lib.rs`: tonic `ArtifactStoreServer` and `ProverNetworkServer` must raise both encoding and decoding max message size to `64 MiB`. — Reason: SP1 vkeys + `create_program` payloads exceed tonic's 4 MiB default.
- [Rule] `bin/network-gateway/src/lib.rs`: Artifact bodies must be served via the HTTP proxy (`/artifacts/{type}/{id}`), not gRPC. — Reason: gRPC message ceilings; HTTP backpressure comes from the artifact store, not Axum.
- [Rule] `bin/network-gateway/src/lib.rs`: Network Gateway must reach the cluster only through `ClusterServiceClient` at `cfg.cluster_rpc`; must never speak to SPN directly. — Reason: gateway is a one-way SDK→cluster bridge.
- [Rule] `bin/node/src/main.rs`: CPU workers must complete `try_install_circuit_artifacts("groth16")` + `try_install_circuit_artifacts("plonk")` before calling `WorkerServiceClient::open`. — Reason: avoid first-use latency that races into the heartbeat path.
- [Rule] `bin/node/src/main.rs`: tasks must be aborted after `TASK_TIMEOUT = 6h`; graceful shutdown must never exceed `DRAIN_TIMEOUT = 30min` to stay under ECS `stopTimeout`. — Reason: liveness vs ECS contract.
- [Rule] `crates/artifact/src/lib.rs`: Any artifact backend supporting cross-bucket copy must implement `CompressedUpload` alongside `ArtifactClient`. — Reason: cross-bucket bytes are already zstd-compressed; re-compressing wastes CPU and produces incompatible payloads.
- [Rule] `crates/common/src/client.rs`: All outbound cluster RPCs must go through `reconnect_with_backoff` (TLS gated on `https://` prefix, `keep_alive_while_idle=true`, http2 keepalive 15s, 60s call timeout, tcp keepalive 15s). — Reason: standard cluster transport contract.
- [Rule] `crates/common/src/client.rs`: `rustls::crypto::ring::default_provider().install_default()` must be called before opening any TLS gRPC channel (helper is idempotent). — Reason: rustls panics on duplicate provider.
- [Rule] `crates/fulfillment/src/grpc.rs`: SPN-side gRPC endpoints must be built via `configure_endpoint` (timeout 10s, connect_timeout 5s, http2 keepalive 10s/10s, tcp keepalive 30s); `bin/bidder/src/grpc.rs` must mirror these constants exactly. — Reason: divergence silently changes connection behaviour across binaries.
- [Rule] `crates/worker/src/client.rs`: Each call site must pick `retry::infinite()` (must-land calls: heartbeat, complete_task, fail_task, ack_sub) or `retry::bounded(Duration)` (caller-actionable calls: open_sub initial); there is no implicit default. — Reason: retry semantics cannot be inherited silently.
- [Rule] `Cargo.toml`: Workspace SPN deps must come from `git: succinctlabs/network@main`; SP1 deps are pinned to `=6.2.0` and patched to `succinctlabs/sp1 rev 085a68a`; `vergen` is pinned to `=9.0.6`. — Reason: avoid `vergen-lib` version conflict; SPN+SP1 ABI stability.

## Security Constraints

- [Rule] `bin/network-gateway/src/auth.rs`: `Auth::authorize` must recover the signer address from EIP-191 signatures over the canonical message body; `AuthMode::None` recovers `Address::ZERO` and is intentionally permissive — do not interpret zero address as malicious in that mode. — Reason: explicit dev-mode opt-out.
- [Rule] `bin/network-gateway/src/service/artifact_store.rs`: `create_artifact` must require an EIP-191 signature over the constant message `b"create_artifact"`; the constant must not change without coordinated SDK changes. — Reason: SDK signs this exact bytestring.
- [Rule] `bin/network-gateway/src/service/prover_network.rs`: `get_program` must return `Status::not_found` for an unregistered vk_hash, never `Ok` with empty fields. — Reason: SDK's `NetworkClient::get_program` routes to `create_program` only on `NotFound`.
- [Rule] `bin/network-gateway/src/service/prover_network.rs`: Unsupported SDK RPCs must return `Status::unimplemented("<rpc>: not supported by self-hosted gateway")`; never fabricate values. — Reason: false success would break SDK retry/escalation.
- [Rule] `crates/artifact/src/redis.rs`: `RedisArtifactClient::validate_config` must be called before serving traffic; refuses to start when any Redis shard has `maxmemory == 0`. — Reason: admission gating depends on a non-zero memory budget.
- [Rule] `crates/artifact/src/redis.rs`: `AdmissionGuard` is `#[must_use]` and must be bound to a named scope (`_guard`, never `_`); the guard must outlive the upload. — Reason: dropping the guard early reintroduces a TOCTOU race.
- [Rule] `crates/artifact/src/redis.rs`: Compile-time `assert!(INPUT_BUDGET_FRACTION < MEMORY_BUDGET_FRACTION)` must hold (currently 0.50 < 0.80). — Reason: non-permitted writes need headroom below the admission ceiling to avoid pipeline deadlock.
- [Rule] `crates/fulfillment/src/lib.rs`: `VK_MISMATCH_ERROR = 2` and the matching string list must stay in sync with `spn_network_types::ProofRequestError::VerificationKeyMismatch`. — Reason: the network expects this exact error code on VK mismatch.
- [Rule] `crates/fulfillment/src/lib.rs`: When `name` is set, every cluster query must filter by `scheduled_by=name`. — Reason: multiple fulfillers sharing a cluster API otherwise steal each other's work.
- [Rule] `crates/fulfillment/src/network.rs`: Private stdin (`request.stdin_private() == true`) must be fetched via signed `FulfillmentNetwork::fetch_stdin_uri`; never trust `stdin_public_uri` in the private branch. — Reason: presigned URL is the only legitimate access path.
