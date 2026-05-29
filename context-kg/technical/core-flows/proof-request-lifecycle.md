---
name: "proof-request-lifecycle"
description: "Core flow: ProofRequest lifecycle from SDK submission through completion"
---
# Proof Request Lifecycle Flow

End-to-end path of a proof request: SDK/CLI submits → bin/network-gateway or bin/cli writes through `ClusterServiceClient` → bin/api inserts Pending row → bin/coordinator's claimer polls and dispatches Controller task → bin/node worker runs proof pipeline → completion flows back to bin/api as terminal status.

## Entry Point

| Entry | Handler | Primary Entities |
|-------|---------|------------------|
| gRPC `ProverNetwork/RequestProof` | `ProverNetworkImpl::request_proof` (`bin/network-gateway/src/service/prover_network.rs`) | `RequestProofRequest`, `RequestProofResponse{request_id:[u8;32]}`, `vk_hash`, `stdin_uri` |
| gRPC `ProverNetwork/CreateProgram` | `ProverNetworkImpl::create_program` (`bin/network-gateway/src/service/prover_network.rs`) | `vk_hash`, ELF bytes, `ProgramStore` (durable) + cluster artifact store (warm copy under `program_artifact_id`) |
| gRPC `ProverNetwork/GetProofRequestStatus` | `ProverNetworkImpl::get_proof_request_status` | `request_id` → `proof_id`, `FulfillmentStatus`, `ExecutionStatus` |
| gRPC `ClusterService/ProofRequestCreate` | `ClusterServiceImpl::proof_request_create` (`bin/api/src/service.rs`) | INSERT row with `proof_status=Pending`, `stdin_private` flag |
| gRPC `ClusterService/ProofRequestCancel` | `ClusterServiceImpl::proof_request_cancel` | `Pending → Cancelled` (only if currently Pending) |
| Coordinator polling task | `cluster::spawn_proof_claimer_task` (`bin/coordinator/src/cluster.rs`) | Every 500 ms polls `ProofRequestListRequest(status=Pending, handled=false, minimum_deadline=now)` |
| Coordinator completion notifier | `cluster::spawn_proof_status_task` | Receives `ProofResult`, calls `proof_request_update(status=Completed | Failed)` |
| Worker controller task | `SP1ClusterWorker::process_sp1_controller` (`crates/worker/src/tasks/controller/mod.rs`) | Runs full proof pipeline then `complete_proof(status=Completed)` |

## Primary Entities

`ProofRequest` row in Postgres `proof_requests`; in-memory `Proof`/`Task`/`Worker` in `Coordinator`; `ControllerInputMetadata { stdin_private }` JSON in `inputs[5]`; `request_id` (32 bytes) ↔ `proof_id` (typeid string) via `mint_request_id` / `proof_id_from_request_id`; deterministic `program_artifact_id = "program_" + hex(vk_hash)`.

## State Transitions

| Entity | Transition | Trigger | Terminal |
|--------|-----------|---------|----------|
| `ProofRequest` | (none) → `Pending` | `proof_request_create` INSERT | no |
| `ProofRequest` | `Pending` → `Cancelled` | `proof_request_cancel` SQL `UPDATE ... WHERE proof_status=Pending` (rejected otherwise) | yes |
| `ProofRequest` (in-memory at coordinator) | (unclaimed) → claimed/`task_map=Pending` | `spawn_proof_claimer_task` calls `coordinator.create_proof` | no |
| `ProofRequest` | `Pending` (claimed) → `Completed` | Controller finishes; worker `complete_proof` → `ProofResult{success=true}` → `spawn_proof_status_task` writes `Completed` | yes |
| `ProofRequest` | `Pending` (claimed) → `Failed` | Controller fatal OR retries exhausted OR `try_unclaim_proof` on Fatal/Execution | yes |
| `Task` (Controller) | `Pending` → `Running` → `Succeeded` | normal path | task-level yes |
| `Task` (Controller) | `Running` → `FailedRetryable` (retries < `MAX_TASK_RETRIES=3`) → re-enqueued | `TaskError::Retryable` | no |
| `Task` (Controller) | `Running` → `FailedFatal` | `TaskError::Fatal` OR retries == 3 on retryable; triggers `manual_proof_fail` since Controller is in `enable_proof_fail` | yes (triggers proof Failed) |

**[Rule] Terminal states must never be reversed**: `ProofRequestStatus::{Cancelled, Completed, Failed}` are terminal in Postgres; coordinator must not overwrite a terminal row.

## Normal Flow Steps

| Step | Module | Action | Notes |
|------|--------|--------|-------|
| 1 | `bin/network-gateway/src/service/prover_network.rs` | SDK calls `CreateProgram` (once per vk_hash): pull ELF from gateway artifact store, persist to `ProgramStore` (durable), warm-copy to cluster artifact store under `program_artifact_id`, delete orphan SDK-uploaded artifact | Survives Redis TTL (4h) via re-upload from `ProgramStore` |
| 2 | `bin/network-gateway/src/service/prover_network.rs` | SDK calls `RequestProof`: `Auth::authorize` → derive `program_artifact_id` from vk_hash → `exists()` check → on TTL miss re-upload from `ProgramStore` → mint 32-byte `request_id` (typeid + zero pad) → `create_artifact` for proof output | `request_id` length is exactly `REQUEST_ID_LEN=32` |
| 3 | `bin/network-gateway/src/service/prover_network.rs` | Gateway calls `ClusterServiceClient.create_proof_request(proof_id=proof_id_from_request_id(request_id), program_artifact_id, stdin_artifact_id, options_artifact_id=mode.to_string(), proof_artifact_id, deadline, cycle_limit, gas_limit, requester, stdin_private)` | `proof_id` is typeid string with zero pad stripped |
| 4 | `bin/api/src/service.rs` | API INSERT INTO `proof_requests` with `proof_status=Pending`, `handled=false`, `stdin_private` | DB of record |
| 5 | `bin/coordinator/src/cluster.rs` | `spawn_proof_claimer_task` polls every 500 ms for `status=Pending, handled=false, minimum_deadline=now, limit=1000`; skips rows already in `task_map` | Expired requests silently filtered out |
| 6 | `bin/coordinator/src/cluster.rs` | Build `inputs=[program_artifact_id, stdin_artifact_id, options_artifact_id, cycle_limit.to_string(), proof_nonce="", metadata_json]`, `outputs=[proof_artifact_id]`; serialize `ControllerInputMetadata{stdin_private}` as JSON | `options_artifact_id` holds `ProofMode` integer; missing required fields cause `continue` with error log (dead row) |
| 7 | `bin/coordinator/src/lib.rs Coordinator.create_proof` | Insert Proof into in-memory state, create Controller TaskData (or ExecuteOnly if `execute_only_mode`), assign worker with `CONTROLLER_WEIGHT` | `task_map.insert(proof.id, Pending)` marks claimed locally |
| 8 | `bin/coordinator/src/server.rs WorkerService` | Sends `ServerMessage::NewTask(WorkerTask{task_id, data, metadata})` over worker's open stream | `metadata` embeds OpenTelemetry parent context |
| 9 | `crates/worker/src/lib.rs SP1ClusterWorker::run_task` | Matches `TaskType::Controller` → `process_sp1_controller` | Returns `TaskMetadata` or `TaskError` |
| 10 | `crates/worker/src/tasks/controller/mod.rs` | Validate inputs non-empty, build `RawTaskRequest`, call `worker.controller().run(...)` which orchestrates `ProveShard / CoreExecute / RecursionDeferred / RecursionReduce / ShrinkWrap / PlonkWrap / Groth16Wrap` via SP1's internal scheduler | `inputs[3]=cycle_limit`, `inputs[4]=proof_nonce`, `inputs[5]=stdin_private` metadata |
| 11 | `crates/worker/src/tasks/controller/mod.rs` | On success: `worker_client.complete_proof(proof_id, Some(task_id), ProofRequestStatus::Completed, "")` | Routes to coordinator's CompleteProof RPC |
| 12 | `bin/coordinator/src/server.rs WorkerService.complete_proof` → `Coordinator.complete_proof` | Emit `ProofResult{success=true}` on `proofs_tx` | Drained by `spawn_proof_status_task` |
| 13 | `bin/coordinator/src/cluster.rs spawn_proof_status_task` | Call `ClusterService.proof_request_update(proof_status=Completed, metadata=json, extra_data)` | DB row terminal |
| 14 | `bin/network-gateway/src/service/prover_network.rs` | SDK polls `GetProofRequestStatus` → `load_cluster_proof` maps cluster `ProofRequestStatus` to SDK `FulfillmentStatus`/`ExecutionStatus`; returns `proof_uri` only when Fulfilled | `proof_uri` derived from `proof_artifact_id` |

## Exception Branches

| Scenario | State Change | Compensation |
|----------|--------------|--------------|
| SDK cancels before any worker claims it | DB `Pending → Cancelled` | If coordinator was already claiming, next claimer poll's `retain` block calls `fail_proof(notify_sender=false)` to local-fail the running proof |
| Deadline already in the past at claim time | Excluded from poll response (filtered by `minimum_deadline=now`) | Row stays Pending in DB but never claimed; if previously claimed, claimer's `retain` block fails it (DB NOT updated to Failed in this branch) |
| Required field missing (`options_artifact_id`, `cycle_limit`, `proof_artifact_id`) | Claimer logs error and `continue`s | Row stays Pending forever — dead row, no retry |
| `coordinator.create_proof` fails | Claimer logs error and does not insert into `task_map` | Next 500 ms poll re-attempts |
| Worker controller returns `TaskError::Retryable` | `task.retries += 1`, re-enqueue as `FailedRetryable` until `retries == MAX_TASK_RETRIES` | Bounded retry |
| Retries == `MAX_TASK_RETRIES` on retryable | `enable_proof_fail(Controller)=true` → `manual_proof_fail` → `fail_proof_internal` cancels assigned tasks, emits `ProofResult{success=false}` → DB `Failed` | Terminal |
| Worker controller returns `TaskError::Fatal` | `try_unclaim_proof` calls `complete_proof(status=Failed)` directly + returns `TaskStatus::FailedFatal` to coordinator | DB Failed via direct unclaim AND `ProofResult` channel |
| Worker controller returns `TaskError::Execution` | Same `try_unclaim_proof` (DB → Failed) but worker reports `TaskStatus::Succeeded` to coordinator | Coordinator success path treats task as Succeeded |
| Coordinator shutting down | `fail_proof_internal` early-returns Ok | In-flight proofs NOT marked Failed during shutdown; new coordinator rediscovers via claimer |
| Worker heartbeat timeout > 30 s | `cleanup_dead_workers` re-enqueues that worker's tasks via `remove_worker_internal` | Retry counter NOT reset |
| API row in DB Pending but coordinator never sees it (`seen_set`) | Claimer's `task_map.retain` detects "Running proof no longer in cluster DB list" → `coordinator.fail_proof(notify_sender=false)` | Local cleanup only; API not notified |

## Flow-Specific Pitfalls

- [Pitfall] `get_program` returning `Ok(None)` instead of `NotFound`: SDK's `NetworkClient::get_program` treats any Ok as "already registered" → future `request_proof` fails with `FailedPrecondition`. Must return `Status::not_found`.
- [Pitfall] `request_id` length deviation: SDK decodes via `B256::from_slice` which requires exactly 32 bytes. `proof_id_from_request_id` must strip trailing zeros before any DB lookup.
- [Pitfall] `program_artifact_id` is deterministic — hot path skips upload via `exists()`. If Redis TTL expires between proofs, `request_proof` re-uploads from `ProgramStore`; if `ProgramStore` lost the ELF, gateway returns `FailedPrecondition`. Don't rely on cluster artifact alone.
- [Pitfall] `stdin_private` flag must be propagated through THREE places: DB column, `ControllerInputMetadata` JSON in `inputs[5]`, AND SDK's stdin URI scheme (`private-stdin` vs `stdin`). Missing any one branches the worker to the wrong artifact type.
- [Pitfall] `proof_request_cancel` only transitions `Pending → Cancelled`. A claimed request must rely on coordinator-side `fail_proof`, which is currently only triggered by claimer detecting the row disappeared.
- [Pitfall] Claimer uses `minimum_deadline=now` — a request that becomes expired after being claimed will be removed from the poll result and force-failed (`notify_sender=false`). DB row is NOT updated to Failed by the claimer in that branch; final status depends on the worker also calling `try_unclaim_proof`.
- [Pitfall] `TaskError::Execution` returns `TaskStatus::Succeeded` but ALSO calls `try_unclaim_proof` which writes Failed directly. Two paths can race to update the DB.
- [Pitfall] Coordinator's `task_map.insert` happens AFTER `create_proof` succeeds. If the coordinator restarts between DB-status=Pending and `create_proof` returning, the proof appears unclaimed on next poll and is re-claimed.
- [Pitfall] Controller is in `enable_proof_fail`; ProveShard, RecursionReduce, etc. are NOT. A non-retryable failure of a non-`enable_proof_fail` task only marks itself `FailedFatal`, leaving the proof potentially hung until the controller observes the failure.
- [Pitfall] `spawn_proof_status_task` logs an error if `*status != Pending` but still overwrites; metadata serialization failure falls back to `"null"` silently.
