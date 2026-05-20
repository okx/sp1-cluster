---
name: "worker-task-execution"
description: "Core flow: Worker task execution from coordinator dispatch to terminal status"
---
# Worker Task Execution Flow

Coordinator schedules `NewTask` → bin/node receives over its open gRPC stream → `SP1ClusterWorker::run_task` dispatches by `TaskType` → completion / failure reported back via `complete_task` / `fail_task` → coordinator updates per-task state via `AssignmentPolicy`.

## Entry Point

| Entry | Handler | Primary Entities |
|-------|---------|------------------|
| Worker registration | `bin/node/src/main.rs::run_worker` → `WorkerServiceClient::open(OpenRequest{ worker_id, worker_type, max_weight })` | Streaming `mpsc::UnboundedReceiver<ServerMessage>` |
| Coordinator open | `bin/coordinator/src/server.rs::WorkerService::open` → `coordinator.add_worker` | Inserts in-memory `Worker` |
| Task creation | `WorkerService::create_proof` (Controller/ExecuteOnly) OR `WorkerServiceClient::submit_task` (per-task) | `CreateTaskRequest{TaskData{...}}` with `task_weight(task_type)` |
| Main worker loop | `tokio::select!` over `channel.recv()`, `heartbeat_ticker.tick()` (5s), `shutdown_rx.changed()` | `server_message::Message::NewTask` / `CancelTask` / `ServerHeartbeat` |

## Primary Entities

`Worker` (in-memory at coordinator), `Task` (state + retries + worker assignment), `TaskData` (gRPC envelope), `ServerMessage` (NewTask/CancelTask/ServerHeartbeat), `TaskStatus` (proto enum), `TaskError` (worker-side error enum).

## State Transitions

| Entity | Transition | Trigger | Terminal |
|--------|-----------|---------|----------|
| `Task` | (none) → `Pending` | `create_task_internal` (`bin/coordinator/src/lib.rs:449`) | no |
| `Task` | `Pending` → `Running` | `BalancedPolicy::assign_task` sets `mut_task.status = Running` and calls `send_task` | no |
| `Task` | `Running` → `Succeeded` | `Coordinator::complete_task` | yes |
| `Task` | `Running` → `FailedRetryable` (transient) | `fail_task` when `retries < MAX_TASK_RETRIES`; re-enqueued and re-assigned via `Running` | no |
| `Task` | `Running` → `FailedFatal` | `fail_task` when `!retryable` OR `retries == MAX_TASK_RETRIES`; also via subscriber `UnknownTask` synthetic | yes |
| `Worker` | `closed=false` → `closed=true` | `close_worker` RPC OR shutdown signal | yes |
| `Worker` | live → removed | `cleanup_dead_workers` after `WORKER_HEARTBEAT_TIMEOUT=30s` silence | yes |

**[Rule] Terminal states must never be reversed**: `Task::status ∈ {Succeeded, FailedFatal}` are terminal. `proof.active_tasks` is decremented only on first transition into a terminal state.

## Normal Flow Steps

| Step | Module | Action | Notes |
|------|--------|--------|-------|
| 1 | `bin/node/src/main.rs::run_worker` | Build `WorkerServiceClient`, call `.open()` → coordinator's `WorkerService::open` spawns `coordinator.add_worker(worker_id, worker_type, max_weight, tx)` | `max_weight = WORKER_MAX_WEIGHT_OVERRIDE` or `ceil(total_ram_gb/16)*16` |
| 2 | `bin/coordinator/src/lib.rs Coordinator::add_worker` | Insert `Worker { weight:0, active_tasks:HashSet::new, last_heartbeat:now, closed:false }`; call `assign_tasks` so backlog dispatches | Bootstraps the worker |
| 3 | `crates/worker/src/client.rs::submit_task` OR `WorkerService::create_proof` | Translate `RawTaskRequest` → `CreateTaskRequest{TaskData{task_id="task".create_type_id::<V7>(), data, task_type, weight=task_weight(task_type)}}` | `Coordinator::create_task_internal` inserts `Task{ status:Pending, retries:0, subscribers:HashSet::new, worker:None }` |
| 4 | `bin/coordinator/src/policy/balanced.rs` | `enqueue_task` routes to `cpu_queue: BinaryHeap<Reverse<QueuedTask>>` or per-proof `gpu_queues[proof_id]`; `assign_tasks` selects task and worker | Worker selection: `!closed`, matching `worker_type` (or `All`), lowest `next_free_time`, `weight + task_weight <= max_weight` |
| 5 | `bin/coordinator/src/lib.rs send_task` | `worker.weight += task_weight`; `worker.next_free_time = max(now, prev) + estimate_duration(task_type)`; `worker.active_tasks.insert((proof_id, task_id))`; task `worker=Some(id), status=Running` | For GPU: also bump `proof_gpu_weights` |
| 6 | `bin/coordinator/src/lib.rs send_task` | Send `ServerMessage::NewTask(WorkerTask{task_id, data, metadata})` over worker's mpsc tx | metadata embeds OTel parent context |
| 7 | `bin/node/src/main.rs` worker loop | Receive `NewTask`; if `closed` skip; if key already in `tasks` DashMap skip (retransmit dedupe); `tokio::spawn` to call `worker.run_task(&task)` | DashMap entry: `(TaskData, JoinHandle, Instant)` |
| 8 | `crates/worker/src/lib.rs run_task` | Parse `metadata` JSON, set up OTel spans (outer `Prove {proof_id}` only for empty metadata + Controller; else extract via propagator), dispatch by `TaskType` to `tasks/*` handler | Handler runs proof work, returns `Result<TaskMetadata, TaskError>` |
| 9 | `crates/worker/src/lib.rs run_task` | On success serialize `TaskMetadata` JSON → `worker_client.complete_task(CompleteTaskRequest{...})` (infinite retry). On `Retryable`/`Fatal`/`Execution` → `worker_client.fail_task(FailTaskRequest{retryable=...})`. Remove DashMap entry | `Execution` reports as `Succeeded` but unclaims proof |
| 10 | `bin/coordinator/src/lib.rs complete_task` | `task.status=Succeeded`; decrement `proof.active_tasks` (only first time); notify subscribers `TaskResult{Succeeded}`; close task channel via `close_task_channel`; remove from `worker.active_tasks`; decrement `worker.weight`; `post_worker_empty` if idle; `P::post_task_update_state`; `assign_tasks` | If `proof.active_tasks==0` proof is removed |
| 11 | Heartbeat reconciliation | Every 5s worker sends `(active_task_proof_ids, active_task_ids, current_weight)`; coordinator `handle_heartbeat` updates `worker.last_heartbeat`, sets `worker.weight=current_weight`, send `CancelTask` for worker-only tasks, re-`send_task` for coordinator-only tasks, `assign_tasks` | Heartbeat clobbers `worker.weight` (incremental bookkeeping can drift between ticks) |

## Exception Branches

| Scenario | State Change | Compensation |
|----------|--------------|--------------|
| `TaskError::Retryable` AND `task.retries < MAX_TASK_RETRIES` | `task.retries += 1`; status `FailedRetryable`; re-`enqueue_task` and re-assign | Bounded retry up to 3 |
| `TaskError::Retryable` AND `task.retries == MAX_TASK_RETRIES` | Status `FailedFatal`; `proof.active_tasks -= 1` | `manual_proof_fail` if `enable_proof_fail(task_type)` |
| `TaskError::Fatal` | Worker `try_unclaim_proof(complete_proof(status=Failed))`; coordinator `fail_task(retryable=false) → FailedFatal` | DB Failed via two paths; manual_proof_fail if Controller/Wrap/Execute task |
| `TaskError::Execution` | Same `try_unclaim_proof(Failed)`; worker reports `TaskStatus::Succeeded` to coordinator | Coordinator runs success path; DB Failed via direct unclaim |
| Panicked task (DashMap entry remains; `JoinHandle::is_finished()==true`) | Removed, awaited, sent `fail_task(retryable=false)` → `FailedFatal` | Process continues |
| Task timeout > `TASK_TIMEOUT=6h` | `handle.abort()` then `fail_task(retryable=true)` | Up to 3 retries |
| Heartbeat reconnect: `tonic::Code::NotFound` from coordinator | Worker reconnects via `worker_client.open()` | Other codes infinite-retry; >10s without `ServerHeartbeat` triggers reconnect |
| Worker dies (heartbeat silence > 30s) | `cleanup_dead_workers` calls `remove_worker_internal` → re-enqueue active_tasks WITHOUT bumping `task.retries` | Flaky workers can replay same task indefinitely |
| Shutdown / drain after DRAIN_TIMEOUT=30 min | Loop breaks; in-flight tasks left running | No `fail_task` sent; coordinator relies on 30s heartbeat timeout |
| Server stream closed (`channel.recv()` returns None) | Worker logs, reconnects via `open()`; reset `last_heartbeat` | 1s sleep if reconnect fails |
| Coordinator shutting_down | `create_proof` returns `failed_precondition`; `fail_proof_internal` early-returns | New coordinator picks up via claimer |

## Flow-Specific Pitfalls

- [Pitfall] `TaskError::Execution` returns `Succeeded` to coordinator but unclaims the proof. The `complete_task` RPC may run BEFORE the `fail_proof` RPC (both infinite retry; success path doesn't await `try_unclaim_proof` first). Outcome can vary.
- [Pitfall] `try_unclaim_proof` runs for both Fatal AND Execution before coordinator's `enable_proof_fail` gate — non-`enable_proof_fail` task types (ProveShard, RecursionDeferred, etc.) with Fatal/Execution still drag the whole proof to Failed.
- [Pitfall] `MAX_TASK_RETRIES = 3` increments only on retryable. Non-retryable failure → immediate FailedFatal regardless of count.
- [Pitfall] Heartbeat-driven reassignment doesn't bump `retries`. A flaky worker dying after 30s silence → its tasks re-enqueue without retry-count growth — can replay indefinitely.
- [Pitfall] `TASK_TIMEOUT=6h` calls `fail_task(retryable=true)`. Worst case: 3 × 6h before a truly stuck task fails the proof.
- [Pitfall] `DRAIN_TIMEOUT` exit path emits no `fail_task`. Coordinator only finds out via the 30s heartbeat timeout; abandoned tasks then re-enqueue. ECS `stopTimeout=3600s` is the assumed backstop.
- [Pitfall] `tasks.contains_key` early-return in NewTask handler is the only dedupe for retransmits. If a handler already finished and removed the key, a late retransmit could re-run the task.
- [Pitfall] `worker.weight` is clobbered authoritatively by heartbeat (`current_weight`). Between heartbeats, incremental coordinator bookkeeping can drift — brief windows of incorrect capacity decisions are possible.
- [Pitfall] `Task.status` carries a misleading TODO comment ("this is unused/incorrect?") — actually read in multiple places (`balanced.rs`, `add_subscriptions`, `fail_proof_internal`).
- [Pitfall] `estimate_duration` returns fixed ms (Controller=200, PlonkWrap=8000, ProveShard=2000, …). Real durations vary 10×+, so `next_free_time` ordering can pile work on a single worker.
- [Pitfall] `proof_gpu_weights` is decremented in `post_task_update_state` and cleared in `post_worker_empty`/`on_proof_deleted`. A coordinator crash between `worker.weight -= w` and `post_task_update_state` can leave a proof's GPU weight inflated and de-prioritized.
- [Pitfall] `complete_task` on already-`Succeeded` task: subscriber notify and `active_tasks` decrement are short-circuited, but `worker.weight` and `active_tasks` removal still happen.
- [Pitfall] `subscribe_task_messages` `drive_message_stream` reconnects with `start_offset=forwarded` to avoid duplicate payloads; closed channels swept every minute (`STALE_THRESHOLD=60s`).
- [Pitfall] Heartbeat reconnect trigger is specifically `tonic::Code::NotFound`. Coordinator must return NotFound (not InvalidArgument or similar) when a worker re-heartbeats after being purged — the code is the only signal.
- [Pitfall] `CoreBatchData` / `DeferredBatchData` / `ReduceBatchData` are NOT part of gRPC `TaskData` — too large; worker reads them from artifacts after `NewTask`.
