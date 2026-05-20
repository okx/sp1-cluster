---
name: "service-patterns"
description: "Recurring service patterns: retry, reconnect, admission, idempotency"
---
# Service Patterns

## gRPC Client Retry Wrapper

- [Convention] All cluster-internal RPC calls wrap the call in `backoff::future::retry(...).map_err(status_to_backoff_error)`. Choose between `retry::infinite()` (heartbeat, reports, must-land) and `retry::bounded(Duration)` (caller-actionable failure). See `crates/worker/src/client.rs mod retry` for the canonical split.
- [Convention] `backoff_retry` + `ExponentialBackoffBuilder` from `crates/common/src/util.rs` is the standard helper. Classify failures via `BackoffError` / `status_to_backoff_error`; transient codes include Internal/Unavailable/Unknown/Cancelled/DeadlineExceeded/ResourceExhausted/Aborted/DataLoss.

## tonic Endpoint Factory

- [Convention] Each binary that talks to external gRPC defines a small `grpc::configure_endpoint` with identical timeout/keepalive constants (10s call / 5s connect / 10s http2 keepalive / 10s http2 keepalive timeout / 30s tcp keepalive). Shared infra-level fact, NOT a shared function — `crates/fulfillment/src/grpc.rs` and `bin/bidder/src/grpc.rs` duplicate the constants intentionally; keep them in lockstep.

## Cluster Channel Helper

- [Convention] Cluster-internal channels go through `crates/common/src/client.rs::reconnect_with_backoff`: TLS gated on `https://` prefix, `keep_alive_while_idle=true`, http2 keepalive 15s, 60s call timeout, tcp keepalive 15s, initial 100 ms / max 4s backoff with no elapsed limit.

## Admission Control (Redis)

- [Convention] Bytes ≤ `ADMISSION_BYPASS_BYTES` (~2 MiB) skip admission entirely; load-bearing for tail-deadlock prevention.
- [Convention] Bytes > bypass acquire an `AdmissionGuard` (`#[must_use]`); must outlive the upload. Bind to `_guard`, never `_`.
- [Convention] Permit pool sized lazily from `INFO memory` × `INPUT_BUDGET_FRACTION` / `max_shard_bytes`, floored at `MIN_PERMITS_PER_NODE=4`.
- [Convention] `validate_config` at startup is mandatory for the Redis backend.

## Cross-Bucket Artifact Copy

- [Convention] Cross-bucket / cross-backend copies MUST use `CompressedUpload::upload_raw_compressed` on the target client to avoid re-zstd.
- [Convention] When you need decompressed form, call `download_raw` once and process; do not pass decompressed bytes back into a re-upload (use the raw compressed payload).

## Idempotency Patterns

- [Convention] Deterministic id for program artifacts: `program_artifact_id(vk_hash)` ensures multiple proofs over the same ELF share storage and skip re-upload within the Redis TTL window (4h).
- [Convention] Postgres `id` PK uniqueness is the only dedup on `proof_request_create`; sqlx errors all collapse to `Internal` — there is no `AlreadyExists` branch.
- [Convention] `proof_request_cancel` is implicitly idempotent for non-Pending rows (no-op `NotFound`).
- [Convention] Worker `tasks.contains_key((proof_id, task_id))` early-return dedupes coordinator retransmits of `NewTask`.

## Cache / Warm-Cache Patterns

- [Convention] `request_proof` hot path: `exists()` check first; only re-upload from `ProgramStore` on TTL miss. Survives Redis 4h TTL while avoiding redundant uploads per request.
- [Convention] `MarkerDeferredRecord` is a synthetic marker task type — handlers must not run real work; coordinator suppresses "worker was not working on task" warnings for it.

## Lock / Lifecycle Patterns

- [Convention] All coordinator state mutations acquire `state.write_owned()` instrumented with `tracing::debug_span!("acquire_write")`; reads use `state.read()` with `acquire`. Long mutations are spawned with `tokio::spawn(...).await.unwrap()?` so RPC cancellation cannot interrupt them.
- [Convention] Subscriber `Sub` entries must be dropped BEFORE acquiring `state.write_owned()` to avoid deadlock.

## Event / Channel Patterns

- [Convention] Coordinator emits `ProofResult` on `proofs_tx` (mpsc::Sender) when proofs terminate; `spawn_proof_status_task` drains the channel and writes the API.
- [Convention] Subscriber task messages live in per-task channels; closed channels persist `STALE_THRESHOLD=60s` so late subscribers can replay buffered messages. Resend cap is 3 retries every 2s.
