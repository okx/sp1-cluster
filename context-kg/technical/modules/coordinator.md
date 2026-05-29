---
name: "coordinator"
description: "Module design for bin/coordinator: in-memory scheduler + WorkerService gRPC"
---
# Coordinator Module

## Responsibilities

- Owns `Coordinator<P: AssignmentPolicy>` in-memory state: proofs, tasks, workers, subscribers, task message channels.
- Hosts `WorkerService` gRPC (tonic + tonic-web `GrpcWebLayer`) on `config.addr` with `tcp_keepalive(20s)`, `http2_keepalive_interval(15s)`, `http2_keepalive_timeout(35s)`.
- Spawns four background tasks: `spawn_proof_status_task` (forwards `ProofResult` → bin/api), `spawn_proof_claimer_task` (polls bin/api every 500 ms for `Pending+!handled+deadline>=now`), heartbeat checker (worker timeout `30s`), periodic cleanup (stale task channels at `STALE_THRESHOLD=60s`).
- Delegates queue + assignment to `AssignmentPolicy` impl — `BalancedPolicy` (default, used by `bin/coordinator/src/main.rs::run_coordinator_server::<BalancedPolicy>`) or `DefaultPolicy`.

## NOT Responsible For

- Persisting state — all in-memory; rebuilds from bin/api after restart via the claimer.
- Executing proofs — those run in worker nodes.
- Calling SPN — fulfiller and bidder cover that surface.
- Direct Postgres access.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `Coordinator<P>` | `state: Arc<RwLock<CoordinatorState>>`, `proofs_tx: mpsc::Sender<ProofResult>` | Single struct exposing all state methods (`create_proof`, `create_task_internal`, `complete_task`, `fail_task`, `fail_proof_internal`, etc.). |
| `Proof` (in-memory) | `id`, `tasks: HashMap<TaskId, Task>`, `active_tasks: usize`, `requester`, `created_at`, `proof_state: P::ProofState` | Removed when `active_tasks == 0`. |
| `Task` (in-memory) | `task_id`, `data: TaskData`, `status: TaskStatus`, `retries: u32`, `subscribers: HashSet<SubscriberId>`, `worker: Option<WorkerId>` | Lifecycle covered in `core-flows/worker-task-execution.md`. |
| `Worker` (in-memory) | `id`, `worker_type: WorkerType`, `max_weight: u32`, `weight: u32`, `next_free_time: Instant`, `active_tasks: HashSet<(ProofId, TaskId)>`, `last_heartbeat: Instant`, `closed: bool`, `channel: mpsc::UnboundedSender<ServerMessage>` | Heartbeat clobbers `weight` to authoritative `current_weight` each tick. |
| `AssignmentPolicy` (trait) | `enqueue_task`, `assign_tasks`, `post_task_update_state`, `post_worker_empty`, `on_proof_deleted`, queue length getters | Parameterizes `Coordinator<P>`. |
| `BalancedPolicy` | `cpu_queue: BinaryHeap<Reverse<QueuedTask>>`, `gpu_queues: HashMap<ProofId, BinaryHeap<...>>`, `proof_gpu_weights: HashMap<ProofId, u32>` | Selects GPU task from proof with lowest `proof_gpu_weights` (tiebreak `proof_created_at`). |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/proof-request-lifecycle.md` and `core-flows/worker-task-execution.md`.

## Module-Specific Pitfalls

- [Pitfall] Subscriber state must be released before acquiring `state.write_owned()`. The comment in `add_subscriptions` is explicit: "Sub cannot be acquired while acquiring state" — reversing the order deadlocks the coordinator.
- [Pitfall] `complete_task` on an already-`Succeeded` task must short-circuit notify and `active_tasks` decrement; duplicate completion (e.g. on retry) still updates `worker.weight` and removes from `active_tasks`. Do not also decrement `active_tasks` again.
- [Pitfall] `MarkerDeferredRecord` is a synthetic task type — `complete_task` suppresses the "worker was not working on task" warning for it. When adding new marker-style task types, mirror the suppression.
- [Pitfall] `enable_proof_fail(task_type)` covers `Controller`, `Groth16Wrap`, `PlonkWrap`, `ExecuteOnly`, `CoreExecute`. Adding a new top-level task type requires deciding explicitly whether to include it; omitting it leaves the proof in a hung state when the new task fatals.
- [Pitfall] `fail_proof_internal` re-inserts the proof on validation failure (task_id given but not found) so cross-instance fail_proof calls can retry. Maintain this rollback when refactoring.
- [Pitfall] `BalancedPolicy::get_queued_task` panics with `proof not found` if the proof has been removed mid-queue. Always queue tasks before removing proofs.
- [Pitfall] Subscriber redelivery cap is 3 retries every 2s (`spawn_subscriber_channel_task`); messages dropped beyond that are silently lost. Increasing the cap requires consumer changes.
- [Pitfall] Task message channels live `STALE_THRESHOLD = 60s` after close so late subscribers can replay. Manual cleanup of channels races the periodic sweeper.
- [Pitfall] Heartbeat-driven reassignment (worker dies after 30s of silence via `cleanup_dead_workers`) does NOT bump `task.retries`. Flaky workers can replay the same task indefinitely.
- [Pitfall] `spawn_proof_claimer_task` filters `minimum_deadline=now` — claimed proofs whose deadline has since passed disappear from the poll feed and trigger `fail_proof(notify_sender=false)` in the claimer's `retain` block, but the API row is NOT updated to `Failed` in that branch. Status reconciliation relies on the worker's `try_unclaim_proof`.
- [Pitfall] tonic graceful shutdown sleeps 2s post-signal then forces stop — long-running RPCs can be cut mid-flight.
- [Warning] `Task.status` carries a TODO comment ("this is unused/incorrect?") but is read in multiple sites (`balanced.rs`, `add_subscriptions`, `fail_proof_internal`). Treat the TODO as misleading.
- [Convention] All state mutations go through `state.write_owned()` instrumented with `tracing::debug_span!("acquire_write")`; reads through `state.read()` with `acquire`. Long mutations are spawned with `tokio::spawn(...).await.unwrap()?` so RPC cancellation cannot interrupt them.
