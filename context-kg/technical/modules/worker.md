---
name: "worker"
description: "Module design for crates/worker: SP1ClusterWorker + per-task handlers"
---
# Worker Crate (`sp1-cluster-worker`)

## Responsibilities

- Wraps `SP1Worker` in `SP1ClusterWorker`; dispatches `WorkerTask.task_type` via `run_task` to per-task handlers in `src/tasks/`.
- Provides `WorkerServiceClient` (gRPC wrapper over coordinator) with explicit per-call retry budget: `retry::infinite()` for must-land calls (heartbeat, complete_task, fail_task, ack_sub) and `retry::bounded(Duration)` for caller-actionable calls (open_sub initial).
- Provides worker-side helpers: `get_max_weight` (16 GB-rounded), `create_http_client` (pool_max_idle_per_host=0, 240s idle timeout), OpenTelemetry context propagation via `task_metadata`/`with_parent`, GPU `get_global_gpu_span` (rotates every 120s).
- Provides metrics struct `WorkerMetrics` (`#[derive(Metrics)] #[metrics(scope="worker")]`) and `initialize_metrics` bootstrap (broadcast shutdown channel).

## NOT Responsible For

- Hosting a gRPC server — `crates/worker` is a client of coordinator.
- Worker registration logic outside `WorkerServiceClient::open` itself.
- Artifact storage backends — those are in `crates/artifact`.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `SP1ClusterWorker<C, A, W>` | `worker: SP1Worker<C>`, `artifact_client: A`, `worker_client: Option<W>` | Generic over `ClusterProverComponents`, `ArtifactClient`, `WorkerClient` |
| `ClusterProverComponents` (type alias) | `sp1_gpu_prover::SP1CudaProverComponents` (gpu feature) or `sp1_prover::CpuSP1ProverComponents` | Compile-time backend selection |
| `WorkerServiceClient` | inner gRPC channel + retry helpers | All calls go through `retry::infinite()` or `retry::bounded(Duration)` |
| `CoreBatchData`, `DeferredBatchData`, `ReduceBatchData` | batch payloads passed via artifact bodies | NOT in gRPC `TaskData` — too large; loaded from artifacts inside the handler |
| `TaskError` (re-export from `sp1_prover::worker`) | `Retryable`, `Fatal`, `Execution` | `Fatal` and `Execution` both call `try_unclaim_proof` → `ProofRequestStatus::Failed` |
| `WorkerMetrics` | counters/gauges/histograms | Includes `task_failures` with `retryable` label, GPU busy time, etc. |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/worker-task-execution.md`.

## Module-Specific Pitfalls

- [Pitfall] `TaskError::Execution` returns `TaskStatus::Succeeded` to coordinator but ALSO calls `try_unclaim_proof` to fail the proof. Two paths can race to update the DB (complete_proof channel + direct `complete_proof(Failed)`); coordinator's normal success path treats the task as Succeeded.
- [Pitfall] `try_unclaim_proof` is invoked on both `Fatal` and `Execution` errors before coordinator's `enable_proof_fail` gate. Non-`enable_proof_fail` task types (ProveShard, RecursionDeferred, etc.) with Fatal/Execution errors still drag the whole proof to Failed.
- [Pitfall] Each `WorkerServiceClient` call site must explicitly pick `retry::infinite()` or `retry::bounded(Duration)` — there is no shared default. Failure semantics cannot be inherited silently.
- [Pitfall] Subscriber `update_sub` sends a random 30-task subset of active task ids on each heartbeat (`choose_multiple(rng, 30)`). Raising this without coordinator-side rate limiting can DoS the server.
- [Pitfall] Subscriber `ack_sub` must run before processing each message; `open_sub` and `ack_sub` are explicitly flagged not cancel-safe — wrapping them in `tokio::time::timeout` or `tokio::select!` can corrupt subscriber state.
- [Pitfall] `WORKER_TYPE` env var is decoded via `WorkerType::from_str_name`; invalid values panic.
- [Pitfall] `MarkerDeferredRecord` and `UnspecifiedTaskType` in `run_task` log error then return `TaskMetadata::default()` (degenerate Ok). Do not assume these task types do real work.
- [Pitfall] `get_global_gpu_span` returns a fresh span every 2 minutes — don't cache it across long awaits.
- [Pitfall] `cluster_worker_config` derives `SP1WorkerConfig` from `cluster_opts()` with adjusted `sharding_threshold` and `global_dependencies_opt=true`. Constructing `SP1WorkerConfig::new()` directly bypasses these critical cluster settings.
- [Pitfall] `create_http_client` returns a `reqwest::Client` with `pool_max_idle_per_host(0)` + `pool_idle_timeout(240s)` — ad-hoc reqwest clients leak connections in long-running workers.
- [Warning] `shrink_wrap` task returns `TaskMetadata::default()` with a TODO to return real metadata — callers see `gpu_ms: None` despite GPU work occurring.
- [Warning] `execute_only` ExecutionError mapping leaves `InvalidMemoryAccessUntrustedProgram`, `EndInUnconstrained`, `UnexpectedExitCode`, `InstructionNotFound`, `InvalidShardingState` falling through to `UnspecifiedExecutionFailureCause`.
- [Warning] Older `create_tasks` / `get_task_statuses` batch APIs are commented out in `client.rs`. Do not resurrect without coordinator-side changes; `submit_task` is the single-task replacement.
