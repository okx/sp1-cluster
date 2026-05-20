---
name: "node"
description: "Module design for bin/node: CPU/GPU worker process"
---
# Node Module

## Responsibilities

- Bootstrap a CPU or GPU SP1 worker via `cpu_worker_builder` / `cuda_worker_builder` (selected by `gpu` cargo feature).
- Select artifact backend via `NODE_ARTIFACT_STORE` (`s3` default | `redis`); for Redis, call `validate_config()` before connecting.
- For CPU workers, run `try_install_circuit_artifacts("groth16")` + `try_install_circuit_artifacts("plonk")` in parallel before calling `WorkerServiceClient::open` so first proofs don't race the heartbeat path.
- Connect to coordinator at `NODE_COORDINATOR_RPC` via `WorkerServiceClient::new`; open the streaming worker channel with `max_weight = WORKER_MAX_WEIGHT_OVERRIDE` or `ceil(total_ram_gb / 16) * 16` from `sys_info`.
- Main loop dispatches `ServerMessage::NewTask` → `SP1ClusterWorker::run_task` (`tokio::spawn`); handles `ServerMessage::CancelTask` (`handle.abort()`); sends 5s heartbeats with `(active_task_proof_ids, active_task_ids, current_weight)`.
- Drain on shutdown (`DRAIN_TIMEOUT = 30 min`, under ECS `stopTimeout`); abort any task running > `TASK_TIMEOUT = 6h`.
- Optional Pyroscope profiling via `PYROSCOPE_USER` / `_PASSWORD` / `_URL` / `_SAMPLE_RATE`.

## NOT Responsible For

- Talking to bin/api or SPN — only coordinator + artifact store.
- Assigning tasks to other workers.
- Bidding or fulfillment.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `tasks: DashMap<(ProofId, TaskId), (TaskData, JoinHandle<TaskStatus>, Instant)>` | per-task tuple of data, spawned future handle, start time | Idempotency dedupe — repeated `NewTask` for the same key is silently skipped. |
| `worker_id` | ECS task ARN last segment, or `unknown_<32-hex-chars>` if not on ECS | Stable identity across coordinator reconnects. |
| `shutdown_tx` / `shutdown_rx` (`watch`) | broadcast of shutdown signal | Drives graceful drain. |
| Ctrl+C handler (CAS on `was_first_signal`) | atomic flag | Second signal forces `std::process::exit(1)`. |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/worker-task-execution.md` and `core-flows/artifact-io.md`.

## Module-Specific Pitfalls

- [Pitfall] `PROVER_CORE_CACHE_SIZE` / `PROVER_COMPRESS_CACHE_SIZE` are unconditionally set to `1` at startup, overriding any operator setting. Don't expect external configuration to take effect.
- [Pitfall] `WORKER_MAX_WEIGHT_OVERRIDE` above the physical RAM rounds emits `tracing::error!("...risk of OOM")` but still applies. The override is intentional but dangerous.
- [Pitfall] Heartbeat reconnect is triggered specifically on `tonic::Code::NotFound` (the worker was purged by `cleanup_dead_workers` after 30s silence). Other error codes infinite-retry inside `WorkerServiceClient::heartbeat`. If no `ServerHeartbeat` from coordinator > 10s, reconnect anyway.
- [Pitfall] `DRAIN_TIMEOUT = 30 min` exit path does NOT send `fail_task` for in-flight tasks; coordinator must rely on its own 30s heartbeat timeout to reassign. Stay below ECS `stopTimeout` (3600s).
- [Pitfall] Tasks running > `TASK_TIMEOUT = 6h` are `abort()`-ed and reported as `fail_task(retryable=true)` — they will retry up to `MAX_TASK_RETRIES=3`. Worst case proof Failure takes up to 24h via this path.
- [Pitfall] CAS in ctrl+c handler is required because two near-simultaneous signals could both enter the first-signal branch and skip the force-exit path.
- [Pitfall] `tasks.contains_key((proof_id, task_id))` early-return is the only dedupe against `NewTask` retransmits (e.g. after heartbeat reconciliation). If a handler already finished and removed the key, a late retransmit would re-run the task.
- [Pitfall] Panicked task detection uses `JoinHandle::is_finished()` during heartbeat tick; the handler removes the entry on its own success path so a stranded entry implies panic. Sent as `fail_task(retryable=false)` → `FailedFatal`.
- [Convention] First call in `main` after `dotenv()` is `rustls::crypto::ring::default_provider().install_default()` — required before any TLS gRPC connection.
