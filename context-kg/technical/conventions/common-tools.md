---
name: "common-tools"
description: "Reusable helpers — duplication is prohibited; reuse via this catalog"
---
# Common Tools — Reusable Components

Every entry below carries a `[Reuse]` marker. Duplicating any of these locally is prohibited; reuse via the listed import path. Adding new local copies requires reviewing whether the existing helper already covers the case.

## crates/common

- [Reuse] `crates/common/src/util.rs::backoff_retry` — retry a tonic RPC with exponential backoff; classify via `BackoffError` (Internal/Unavailable/Unknown/Cancelled/DeadlineExceeded/ResourceExhausted/Aborted/DataLoss are transient). Pair with `backoff::ExponentialBackoffBuilder`; pass an idempotent operation closure returning `Result<T, E: BackoffError>`; uses `NoopNotify`.
- [Reuse] `crates/common/src/util.rs::status_to_backoff_error` — convert `tonic::Status` into `backoff::Error` for use inside `backoff::future::retry` blocks. Call `.map_err(status_to_backoff_error)` on the tonic call result; never on a final-failure status that should propagate.
- [Reuse] `crates/common/src/util.rs::BackoffError` trait — classify transient vs permanent failures uniformly across grpc clients. Existing impls cover `tonic::Status` and `backoff::Error<E>`. Implement for any custom error type.
- [Reuse] `crates/common/src/util.rs::NoopNotify` — suppress backoff retry notifications. Pass as the third arg of `backoff::future::retry_notify` when no logging is desired.
- [Reuse] `crates/common/src/util.rs::get_private_ip` — first private IPv4 via `if_addrs`. Used by workers self-registering to coordinator.
- [Reuse] `crates/common/src/client.rs::reconnect_with_backoff` — establish tonic `Channel` to coordinator/cluster with TLS + http2 keepalive + indefinite reconnect. Channels accept `https://` prefix to opt into `ClientTlsConfig::with_enabled_roots`; installs ring crypto provider; initial 100 ms / max 4 s interval / no elapsed limit.
- [Reuse] `crates/common/src/client.rs::ClusterServiceClient` — API-layer gRPC client for proof_request CRUD with bounded 10s retry budget. Construct via `ClusterServiceClient::new(addr)`; all methods retry with `backoff_retry` under 10s ceiling.
- [Reuse] `crates/common/src/logger.rs::init` — initialize OpenTelemetry + tracing_subscriber. Single `Once`-guarded entry; reads `OTLP_ENDPOINT` (default `http://localhost:4317`), `TRACING_ENABLED`, `LOGGING_ENABLED`, `TOKIO_BLOCKED_BUSY_MS`; pass `Resource` describing service name; silences `p3_*` noise and forces `sp1_*`/`slop_*`/`sp1_gpu_*` to debug.
- [Reuse] `crates/common/src/consts.rs::task_weight` — compute per-`TaskType` RAM weight in GB. Always source from this function; never hardcode. Per-task tunables: `WORKER_CONTROLLER_WEIGHT`, `WORKER_GROTH16_WRAP_WEIGHT`, `WORKER_PLONK_WRAP_WEIGHT`, `WORKER_EXECUTE_ONLY_WEIGHT`, `WORKER_CORE_EXECUTE_WEIGHT`.
- [Reuse] `crates/common/src/proto` (module) — re-export of `sp1_prover_types::cluster::*` + `::worker::*`. All cluster code imports proto types via this module, not `sp1_prover_types` directly.

## crates/artifact

- [Reuse] `crates/artifact/src/lib.rs::ArtifactClient + InMemoryArtifactClient` — shared artifact storage abstraction re-exported from `sp1_prover_types`. All cluster code depends on this trait; do not re-define.
- [Reuse] `crates/artifact/src/lib.rs::CompressedUpload` (trait) — upload pre-compressed (already zstd) bytes during cross-bucket copy paths, bypassing app-layer compression. Implement `upload_raw_compressed` alongside `ArtifactClient`; in-memory impl falls back to `upload_raw`.

## crates/worker

- [Reuse] `crates/worker/src/limiter.rs::get_max_weight` — worker capacity from `sys_info::mem_info`, rounded up to nearest 16 GB. Respects `WORKER_MAX_WEIGHT_OVERRIDE` (logs error if it exceeds physical RAM). Use this — do not call `sys_info` directly elsewhere.
- [Reuse] `crates/worker/src/utils.rs::create_http_client` — `reqwest::Client` tuned for cluster-internal HTTP (`pool_max_idle_per_host=0`, 240s idle timeout). Use for ECS metadata + similar one-shot http calls; do not construct ad-hoc reqwest clients.
- [Reuse] `crates/worker/src/utils.rs::{current_context, task_metadata, current_task_metadata, record_current, with_parent}` — OpenTelemetry trace-context propagation across worker → coordinator → worker hops. Inject context via `task_metadata` for outbound RPC; extract by deserializing `TaskData.metadata` JSON.
- [Reuse] `crates/worker/src/utils.rs::get_global_gpu_span` — shared GPU `info_span!` re-created every 120s for grouping GPU-related work. Use as parent for GPU lock spans so traces aggregate.
- [Reuse] `crates/worker/src/utils.rs::DeferGuard` — RAII-style cleanup with `FnOnce(T)` closure on drop. Prefer over manual `Drop` impls for ad-hoc cleanups.
- [Reuse] `crates/worker/src/utils.rs::conditional_future` — await `Option<Future>` inside `tokio::select!` arms. Returns `Option<T>`; use to make `select!` arms conditional.
- [Reuse] `crates/worker/src/utils.rs::get_tree_layer_size` — compute reduction-tree layer widths with odd-leaf bookkeeping. Reuse for recursion/reduce planning instead of recomputing tree shape.
- [Reuse] `crates/worker/src/utils.rs::chunk_vec` — chunk owned `Vec<T>` by draining (preserves order, frees memory). Prefer over `slice::chunks` when you need owned chunks.
- [Reuse] `crates/worker/src/utils.rs::ECSTaskInfo + get_ecs_task_info` — read `ECS_CONTAINER_METADATA_URI_V4` to discover cluster/task ARN. Use when self-reporting AWS placement; fallible — returns `Result`.
- [Reuse] `crates/worker/src/utils.rs::print_accelerator_info` — one-line GPU/SIMD/SP1_PROFILE banner at boot. Call from `main` to log accelerator capabilities.
- [Reuse] `crates/worker/src/config.rs::cluster_opts / cluster_worker_config` — canonical `SP1CoreOpts` and `SP1WorkerConfig` for cluster execution (custom `sharding_threshold`, `global_dependencies_opt=true`). Always start from these — do not construct `SP1WorkerConfig::new()` directly.
- [Reuse] `crates/worker/src/error.rs::TaskError` — use `sp1_prover::worker::TaskError` (re-export); do not introduce a parallel error enum.
- [Reuse] `crates/worker/src/metrics.rs::WorkerMetrics` — per-binary Prometheus metrics struct derived via `spn_metrics::Metrics`. Scope `worker`; use `#[metrics(scope=...)]` derive; `describe_*!` macros for label-only emissions.

## crates/fulfillment

- [Reuse] `crates/fulfillment/src/grpc.rs::configure_endpoint` — open every fulfillment-side gRPC endpoint with 10s timeout / 5s connect / http2 keepalive 10s/10s / tcp 30s. Mirror constants in `bin/bidder/src/grpc.rs`.
- [Reuse] `crates/fulfillment/src/network.rs::FulfillmentNetwork + NetworkRequest` traits — abstraction over the SP1 prover-network gRPC for the fulfillment loop. Implement per-environment (mainnet/testnet); `fetch_stdin_uri` must honor `stdin_private`; `should_download_proofs()` default `true` (executor binary overrides to `false`).
- [Reuse] `crates/fulfillment/src/config.rs::FulfillerSettings` — load via `FulfillerSettings::new("FULFILLER")`. Declare comma-separated env lists via `with_list_parse_key`. Use `spn_utils::deserialize_domain` for B256 domain and `spn_utils::LogFormat` for log format.

## bin/coordinator

- [Reuse] `bin/coordinator/src/util.rs::OkService<S>` — `tower::Service` wrapping with `/healthz` and `/` liveness endpoints plus per-call tracing span. Wrap any tonic-web/grpc service to expose health probes without writing a handler.
- [Reuse] `bin/coordinator/src/util.rs::spawn_heartbeat_task / spawn_coordinator_periodic_task` — spawn the coordinator's periodic background loops (5s heartbeat; `COORDINATOR_PERIODIC_INTERVAL` cleanup). Use these spawners — do not inline the cleanup loop.
- [Reuse] `bin/coordinator/src/latency.rs::track_latency!() + LATENCY_TRACKER` — low-overhead latency profiling for coordinator hot paths. Gated by `ENABLE_LATENCY_DEBUG` env var; warns at >100 ms; coordinator-private.

## bin/network-gateway

- [Reuse] `bin/network-gateway/src/status.rs::fulfillment_from_cluster / execution_from_cluster / cluster_fulfillment_filter / cluster_execution_filter` — single source of truth for cluster↔SDK enum mapping. Exhaustive match by design; update tests when adding variants.
- [Reuse] `bin/network-gateway/src/ids.rs` helpers — `proto_to_cluster_type`, `artifact_type_segment`, `parse_artifact_type_segment`, `artifact_uri`, `artifact_id_from_uri`, `program_artifact_id`, `mint_request_id`, `proof_id_from_request_id`. URI/id minting and round-tripping between SDK `request_ids` (32-byte) and cluster `proof_ids` (typeid string). `mint_request_id` zero-pads to 32 bytes (SDK B256 requirement); `program_artifact_id` is deterministic per `vk_hash`; segments must round-trip.
