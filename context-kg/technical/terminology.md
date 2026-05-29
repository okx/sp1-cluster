---
name: "terminology"
description: "Domain term glossary for unified terminology across sp1-cluster"
---
# Domain Terminology

Rows sorted A→Z by term. Class names, methods and config keys are written verbatim in backticks.

| Term | Description | References |
|------|-------------|------------|
| `AdmissionGuard` | `#[must_use]` RAII guard returned by Redis admission control; must outlive the upload | `crates/artifact/src/redis.rs` |
| `AssignmentPolicy` | Trait parameterizing `Coordinator<P>`; supplies `ProofState`/`TaskState`/`WorkerState`/`ProofResultMetadata`, `enqueue_task`, `assign_tasks`, `post_task_*` hooks | `bin/coordinator/src/policy/mod.rs` |
| `ArtifactClient` | Backend-agnostic trait for upload/download/exists/delete + `acquire_shard_permit` (Redis) | `crates/artifact/src/lib.rs` (re-export from `sp1_prover_types`) |
| `ArtifactType` | Cluster-side artifact kinds: `UnspecifiedArtifactType`, `Program`, `Stdin`, `Proof`, `Groth16Circuit`, `PlonkCircuit`, `PrivateStdin` | `bin/network-gateway/src/ids.rs` |
| `Auth` | EIP-191 signature recovery for `create_program` / `request_proof` / `create_artifact`; modes `None` / `Verify` / `Allowlist` | `bin/network-gateway/src/auth.rs` |
| `BalancedPolicy` | Default `AssignmentPolicy`: per-proof GPU queues + global CPU queue; tiebreak `proof_created_at` then `created_at` | `bin/coordinator/src/policy/balanced.rs` |
| `Bidder` | SPN bidding loop polling every `REFRESH_INTERVAL_SEC = 3s` | `bin/bidder/src/lib.rs` |
| `ClusterProverComponents` | Type alias selecting GPU vs CPU SP1 prover components by `gpu` cargo feature | `crates/worker/src/lib.rs` |
| `ClusterService` | gRPC service for `proof_request_create / cancel / update / get / list / healthcheck` | `bin/api/src/service.rs` |
| `ClusterServiceClient` | gRPC client used by coordinator / fulfiller / network-gateway to talk to bin/api | `crates/common/src/client.rs` |
| `committed_value_digest` | Public-value digest threaded through deferred batches | `crates/worker/src/tasks/types.rs` |
| `CompressedUpload` | Trait extending `ArtifactClient` for passthrough of pre-compressed (zstd) bytes | `crates/artifact/src/lib.rs` |
| `configure_endpoint` | Helper building tonic `Endpoint` (10s timeout / 5s connect / http2 keepalive 10s / tcp 30s) | `crates/fulfillment/src/grpc.rs`, `bin/bidder/src/grpc.rs` |
| `ControllerInputMetadata` | JSON blob serialized into `inputs[5]` of Controller task, carrying `stdin_private` | `bin/coordinator/src/cluster.rs` |
| `Coordinator<P>` | In-memory cluster scheduler parameterized by `AssignmentPolicy` | `bin/coordinator/src/lib.rs` |
| `CoreBatchData` | Core batch payload: `shard_proofs`, `initial_reconstruct_challenger`, `is_complete`, `total_execution_shards`, `is_first_shard` | `crates/worker/src/tasks/types.rs` |
| `CoreExecute` | `TaskType` running the executor to produce execution traces feeding `ProveShard` | `crates/common/src/consts.rs` |
| `CreateProofRequest` | Coordinator input message: `proof_id`, `inputs[program, stdin, options, cycle_limit, proof_nonce, metadata]`, `outputs[proof]`, `requester`, `expires_at` | `bin/coordinator/src/cluster.rs` |
| `DeferredBatchData` | Deferred batch payload: `vks_and_proofs`, `vk_merkle_data`, digests, challenger, end states | `crates/worker/src/tasks/types.rs` |
| `DRAIN_TIMEOUT` | `30 min` graceful drain bound (under ECS `stopTimeout=3600s`) | `bin/node/src/main.rs` |
| `ExecutionFailureCause` | Enum stored in `proof_requests.execution_failure_cause` | `bin/api/src/service.rs` |
| `ExecutionResult` | Sub-message of `ProofRequest`: `status`, `failure_cause`, `cycles`, `gas`, `public_values_hash` | `bin/api/src/service.rs` |
| `ExecutionStatus` | Status enum stored in `proof_requests.execution_status` | `crates/common/src/proto.rs` |
| `ExecuteOnly` | `TaskType` running execution without proof generation | `crates/common/src/consts.rs` |
| `extra_data` | `TEXT` column for additional per-proof metadata (added 2025-11-05) | `migrations/20251105013312_extra_data.sql` |
| `fold layer` | `2^WORKER_FOLD_LAYER` shards per fold (default 20) | `crates/worker/src/lib.rs` |
| `Fulfiller` | Loop: `submit_requests → fail_requests → cancel_requests → schedule_requests → sleep` | `crates/fulfillment/src/lib.rs` |
| `FulfillmentNetwork` | Trait abstracting SPN ProverNetwork access (mainnet/testnet impls) | `crates/fulfillment/src/network.rs` |
| `FulfillmentStatus` | SPN-side status: `Requested`, `Assigned`, `Fulfilled`, `Unfulfillable`, `Unexecutable` (cluster maps `Cancelled` to `Unfulfillable`) | `bin/network-gateway/src/status.rs` |
| `Groth16Wrap` | `TaskType` finalizing proof with Groth16 SNARK; default weight 14 GB (`WORKER_GROTH16_WRAP_WEIGHT`) | `crates/worker/src/tasks/finalize.rs` |
| `idx_proof_requests_list` | Composite index `(handled, deadline, proof_status)` for list filter | `migrations/20250508020844_init.sql` |
| `MainnetFulfiller` | `FulfillmentNetwork` impl against `spn-network-types::ProverNetworkClient` | `bin/fulfiller/src/mainnet.rs` |
| `MAX_TASK_RETRIES` | `3` — retries before fatal | `bin/coordinator/src/lib.rs` |
| `MarkerDeferredRecord` | `TaskType` that runs no work; coordinator suppresses missing-task warnings for this variant | `crates/worker/src/lib.rs` |
| `MEMORY_BUDGET_FRACTION` | `0.80` — Redis admission upper bound vs `maxmemory` | `crates/artifact/src/redis.rs` |
| `INPUT_BUDGET_FRACTION` | `0.50` — Redis permit pool fraction (must stay below `MEMORY_BUDGET_FRACTION`) | `crates/artifact/src/redis.rs` |
| `NetworkSigner` | SPN signer (`local` private key or `aws_kms`) | `crates/fulfillment/src/run.rs` |
| `OkService<S>` | tower service wrapper exposing `/healthz` and `/` plus per-call tracing | `bin/coordinator/src/util.rs` |
| `PlonkWrap` | `TaskType` finalizing proof with Plonk SNARK; default weight 60 GB (`WORKER_PLONK_WRAP_WEIGHT`) | `crates/worker/src/tasks/finalize.rs` |
| `ProgramStore` | Durable program registry outside the artifact TTL window (`memory` or `fs` backend) | `bin/network-gateway/src/program_store.rs` |
| `program_artifact_id` | Deterministic `"program_" + hex(vk_hash)` cluster artifact id for ELFs | `bin/network-gateway/src/ids.rs` |
| `ProofId` | Strongly typed wrapper over the proof request id string | `crates/worker/src/lib.rs` |
| `ProofMode` | Final-proof mode chosen for wrap step (`Plonk`, `Groth16`) | `crates/worker/src/lib.rs` (re-export) |
| `ProofRequest` | Proto mirror of the `proof_requests` row | `crates/common/src/proto.rs` |
| `ProofRequestStatus` | `Pending` → `Cancelled` / `Completed` / `Failed` | `crates/common/src/proto.rs` |
| `ProofResult` | Coordinator-emitted message carrying `id`, `success`, `metadata`, `extra_data` on completion | `bin/coordinator/src/cluster.rs` |
| `ProveShard` | `TaskType` generating one shard's core STARK proof; weight 4 GB | `crates/worker/src/tasks/prove_shard.rs` |
| `proof_id` | Cluster-side typeid string (`req_…`) — `proof_id_from_request_id(request_id)` strips zero padding | `bin/network-gateway/src/ids.rs` |
| `proof_nonce` | Per-request nonce input to coordinator (currently empty `String::new()`) | `bin/coordinator/src/cluster.rs` |
| `proof_requests` | Postgres table backing every `ProofRequest` row | `migrations/20250508020844_init.sql` |
| `RecursionDeferred` | `TaskType` recursing over deferred-proof batches; consumes `DeferredBatchData` | `crates/worker/src/tasks/recursion.rs` |
| `RecursionReduce` | `TaskType` recursing the reduce tree; consumes `ReduceBatchData` | `crates/worker/src/tasks/recursion.rs` |
| `ReduceBatchData` | Reduce-tree payload: `vks_and_proofs`, `is_complete`, `total_execution_shards` | `crates/worker/src/tasks/types.rs` |
| `RedisArtifactClient` | `ArtifactClient` impl with sharded `deadpool-redis` pool + admission control | `crates/artifact/src/redis.rs` |
| `reconnect_with_backoff` | Helper building tonic `Channel` with TLS + http2 keepalive + indefinite reconnect | `crates/common/src/client.rs` |
| `request_id` | 32-byte SDK-facing id (V7 typeid zero-padded to satisfy `B256::from_slice`) | `bin/network-gateway/src/ids.rs` |
| `S3ArtifactClient` | `ArtifactClient` impl over `aws-sdk-s3` with zstd-level-3 compression, 32 MiB multipart | `crates/artifact/src/s3.rs` |
| `S3DownloadMode` | Toggle between `S3SDKClient` and `S3RestClient` download paths | `crates/artifact/src/s3.rs` |
| `scheduled_by` | Optional `TEXT` column / filter naming the fulfiller that scheduled the request (added 2026-03-27) | `migrations/20260327000000_scheduled_by.sql` |
| `SetupVkey` | `TaskType` setting up verifying-key artifacts; weight 1 GB | `crates/worker/src/tasks/setup.rs` |
| `ShardPermit` | Per-node semaphore-backed admission permit; closed semaphore yields `noop()` (fail-open) | `crates/artifact/src/redis.rs` |
| `ShrinkWrap` | `TaskType` compressing recursion proof to outer wrap proof; weight 4 GB | `crates/worker/src/tasks/shrink_wrap.rs` |
| `SP1ClusterWorker` | Wraps `SP1Worker`; dispatches `TaskType` to per-task handlers under `tasks/*` | `crates/worker/src/lib.rs` |
| `spawn_proof_claimer_task` | Coordinator background loop polling API every 500 ms for `Pending+!handled+deadline>=now` | `bin/coordinator/src/cluster.rs` |
| `spawn_proof_status_task` | Coordinator task forwarding `ProofResult` to API `proof_request_update` | `bin/coordinator/src/cluster.rs` |
| `stdin_artifact_id` | Artifact id of the program's stdin blob | `migrations/20250508020844_init.sql` |
| `stdin_private` | `BOOLEAN NOT NULL DEFAULT FALSE` — marks stdin as `ArtifactType::PrivateStdin` (added 2026-05-13) | `migrations/20260513000000_stdin_private.sql` |
| `STALE_THRESHOLD` | `60s` retention of closed task message channels for late subscribers | `bin/coordinator/src/lib.rs` |
| `task_weight` | Function returning per-`TaskType` RAM weight in GB | `crates/common/src/consts.rs` |
| `TaskError` | Worker task error: `Retryable`, `Fatal`, `Execution` (last is reported as `Succeeded` but still unclaims proof) | `crates/worker/src/error.rs` |
| `TaskMetadata` | Per-task metadata returned to coordinator (e.g. `gpu_ms`) | `crates/worker/src/lib.rs` |
| `TaskStatus` | Lifecycle of a single `WorkerTask`: `Pending`, `Running`, `Succeeded`, `FailedRetryable`, `FailedFatal` | `crates/common/src/proto.rs` |
| `TaskType` | Worker task discriminator (Controller, ProveShard, RecursionDeferred, RecursionReduce, ShrinkWrap, SetupVkey, MarkerDeferredRecord, PlonkWrap, Groth16Wrap, UtilVkeyMapChunk, UtilVkeyMapController, ExecuteOnly, CoreExecute) | `crates/common/src/proto.rs` |
| `TASK_TIMEOUT` | `6h` hard cap on a single task on the worker | `bin/node/src/main.rs` |
| `try_unclaim_proof` | Marks proof `Failed` and unclaims it on Fatal/Execution errors | `crates/worker/src/lib.rs` |
| `vk_hash` | Verifying-key hash used as deterministic key for program artifacts | `bin/network-gateway/src/ids.rs` |
| `vkey` | Short for "verifying key" (`SetupVkey`, `UtilVkeyMap*`, `StarkVerifyingKey`) | `crates/common/src/consts.rs` |
| `WorkerClient` | Worker-side gRPC client trait (open/heartbeat/complete_task/fail_task/complete_proof/subscriber) | `crates/worker/src/client.rs` |
| `WORKER_FOLD_LAYER` | Env override for fold layer (default 20) | `crates/worker/src/lib.rs` |
| `WORKER_HEARTBEAT_TIMEOUT` | `30s` — coordinator drops worker after this silence | `bin/coordinator/src/lib.rs` |
| `WORKER_POLLING_INTERVAL_MS` | Default 100 ms worker poll interval | `crates/worker/src/lib.rs` |
| `WorkerType` | `Cpu`, `Gpu`, `All` — set via `WORKER_TYPE` env | `crates/worker/src/client.rs` |

## Synonyms

| Term A | Term B | Context |
|--------|--------|---------|
| `proof_id` | `request_id` | SDK-side 32-byte zero-padded id; cluster-side typeid string after stripping zeros |
| `proof_id` | `id` (in `proof_requests`) | DB primary key column and gRPC field name for the same identifier |
| `stdin_private` | `PrivateStdin` | Boolean column corresponds to `ArtifactType::PrivateStdin` |
| `vkey` | verifying key | Short form used in `TaskType::SetupVkey`, `UtilVkeyMap*`, `StarkVerifyingKey` |
| `vk_hash` | `program_artifact_id` payload | Cluster artifact id is `"program_" + hex(vk_hash)` |
| `ProveShard` | core shard proof | Produces per-shard core proofs feeding `CoreBatchData.shard_proofs` |
| `RecursionDeferred` | deferred batch | Consumes `DeferredBatchData` payload |
| `RecursionReduce` | reduce batch | Consumes `ReduceBatchData` payload |
| `ShrinkWrap` | shrink + wrap | Single `TaskType` covers shrink-then-wrap step |
| `PlonkWrap` | finalize Plonk | Dispatches `process_sp1_finalize` with `ProofMode::Plonk` |
| `Groth16Wrap` | finalize Groth16 | Dispatches `process_sp1_finalize` with `ProofMode::Groth16` |
| `ProofRequestStatus::Failed` | unclaimed proof | `try_unclaim_proof` transitions to `Failed` |
| `ExecutionResult.cycles` / `.gas` | `execution_cycles` / `execution_gas` | Proto `u64` fields stored as DB `BIGINT` columns |
