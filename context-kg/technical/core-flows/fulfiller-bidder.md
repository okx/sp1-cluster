---
name: "fulfiller-bidder"
description: "Core flow: Fulfiller + Bidder SPN integration cycles"
---
# Fulfiller & Bidder Flow

bin/bidder polls SPN every `REFRESH_INTERVAL_SEC=3s` to bid on biddable requests; bin/fulfiller polls SPN + bin/api every `refresh_interval_sec` (default 3s) to schedule new work into the cluster, submit completed proofs back to SPN, and clean up failed/cancelled requests.

## Entry Point

| Entry | Handler | Primary Entities |
|-------|---------|------------------|
| Bidder main loop | `Bidder::run` (`bin/bidder/src/lib.rs`) | `ProverNetworkClient`, `PrivateKeySigner`, `BidRequestBody` |
| Fulfiller main loop | `Fulfiller::run` (`crates/fulfillment/src/lib.rs`) | `MainnetFulfiller`, `ClusterServiceClient`, `ArtifactClient` |

## Primary Entities

`BidRequestBody`, `FulfillProofRequestBody`, `GetStdinUriRequestBody`, `NetworkSigner` (local or `aws_kms`), `proof_requests` rows (cluster Pending/Completed/Failed/Cancelled), SPN `FulfillmentStatus` (Requested/Assigned/Fulfilled/Unfulfillable/Unexecutable).

## State Transitions

| Entity | Transition | Trigger | Terminal |
|--------|-----------|---------|----------|
| SPN request | `Requested` → bidder bids | bidder `bid` RPC | no |
| SPN request | `Requested → Assigned` (oracle-driven) | external | no |
| SPN request | `Assigned → Fulfilled` | fulfiller `fulfill_proof` RPC | yes |
| SPN request | `Assigned + Unexecutable → terminal failed` | fulfiller `fail_fulfillment` | yes |
| SPN request | `Unfulfillable` (network-side terminal) | external | yes |
| Cluster `proof_requests.handled` | `false → true` | fulfiller after `fulfill_proof` / `fail_fulfillment` | yes |
| Cluster `proof_requests.proof_status` | (none) → `Pending` | fulfiller `create_proof_request` in schedule | no |
| Cluster `proof_requests.proof_status` | `Pending → Cancelled` | fulfiller `cancel_proof_request` when SPN goes Unexecutable/Unfulfillable | yes |

## Normal Flow Steps

### Bidder cycle (every 3s)

| Step | Action | Notes |
|------|--------|-------|
| 1 | SPN biddable query: `GetFilteredProofRequests{version, status=Requested, minimum_deadline=now, not_bid_by=prover, limit=100}` | MUST run BEFORE the assigned query to avoid missing transitions |
| 2 | SPN assigned-to-us query: `status=Assigned, fulfiller=prover` | Sets initial `active_proofs` count |
| 3 | For each biddable: compute `request_duration = deadline - now()` | |
| 4 | If `aggressive_mode`: skip capacity/throughput checks; only enforce `min_deadline_secs` if set | Otherwise call `can_fulfill_proof(active_proofs, gas_limit, request_duration, mode)` |
| 5 | `can_fulfill_proof`: reject if mode disabled (`!groth16_enabled` or `!plonk_enabled`); compute `effective_throughput = throughput_mgas / max_concurrent_proofs`; `completion_time = (gas_limit / 1e6) / effective_throughput`; `total = completion_time + buffer_sec (+ mode-specific buffer)`; require `active_proofs < max_concurrent_proofs` AND `total <= deadline_secs` | |
| 6 | On accept: increment `active_proofs` (non-aggressive) and spawn `bid_request` | |
| 7 | `bid_request`: `GetNonceRequest{address=signer.address()}`; build `BidRequestBody{nonce, request_id, amount=bid_amount, domain, prover, variant=BidVariant}`; sign via `body.sign(&signer)`; send `BidRequest` | |

### Fulfiller cycle (every `refresh_interval_sec`)

| Step | Action | Notes |
|------|--------|-------|
| 1 | `submit_requests`: cluster query `proof_status=Completed, handled=false, minimum_deadline=now, scheduled_by=name, limit=1000`; for each: if `should_download_proofs()` download proof bytes from cluster_artifact_client (`type=Proof`); validate `hex::decode(request.id)`; if `!disable_fulfillment` call `network.submit_request` → signs `FulfillProofRequestBody{nonce, request_id, proof, domain, variant=FulfillVariant}` and sends `FulfillProofRequest`; on success cluster `update_proof_request(handled=true)`; `try_delete` the proof artifact | |
| 2 | `fail_requests`: cluster query `proof_status=Failed, handled=false, scheduled_by=name, minimum_deadline=now`; call `MainnetFulfiller::fail_request` (delegates to `cancel_request` → `fail_request_with_error(error=None)`); cluster `update_proof_request(handled=true)` | |
| 3 | `cancel_requests(prover)`: SPN query `get_cancelable_requests` runs TWO parallel `GetFilteredProofRequests` per address — `(Assigned + Unexecutable)` and `(Unfulfillable)` — deduped via `HashSet<request_id>`. For each: SPN `fail_fulfillment` (unless `disable_fulfillment`); cluster `cancel_proof_request` | |
| 4 | `schedule_requests(prover)`: cluster query for all current requests (any status, `minimum_deadline=now`) → `HashSet<id>` of cluster requests; counts `Pending && !handled` for `cluster_unexecuted_requests` metric | |
| 5 | For each fulfiller address in parallel: `schedule_given_requests` → SPN `get_schedulable_requests` (Assigned, fulfiller=address, version match, `minimum_deadline=now`, limit=1000); reverse list (oldest-first → newest-first scheduling); filter out requests already in cluster; deterministic hash-mod-100 sampling against `request_probability` (skipped when probability == 1.0) | |
| 6 | For each survivor: `schedule_request` → extract artifact names from `program_uri`/`stdin_uri` (last path segment); cluster `create_artifact` for proof output; if `copy_artifacts`: `copy_artifact(program)` → `Program`; for stdin call `MainnetFulfiller::fetch_stdin_uri` — if `stdin_private`, signed `GetStdinUri` RPC to get presigned URL and use `ArtifactType::PrivateStdin`; else use `stdin_public_uri` and `ArtifactType::Stdin`. `copy_artifact` skips upload if artifact already exists; download zstd-compressed via `Artifact::download_raw_from_uri_par` (region `FULFILLER_NETWORK_S3_REGION` default us-east-2, concurrency `FULFILLER_S3_CONCURRENCY` default 32); re-upload via `upload_raw_compressed`. Cluster `create_proof_request` with `options_artifact_id = mode.to_string()`, `scheduled_by=name`, `stdin_private` | |

## Exception Branches

| Scenario | State Change | Compensation |
|----------|--------------|--------------|
| VK mismatch in `submit_request` (error text matches `VK_MISMATCH_STRINGS`) | `network.fail_request_with_error(error=Some(2))` (`ProofRequestError::VerificationKeyMismatch`) | Failure-to-fail logged, not retried |
| `fail_request_with_error` tonic `PermissionDenied`/`FailedPrecondition`/`NotFound`/`InvalidArgument` | Logs warn, swallows | SPN already terminalized; other codes propagate |
| Invalid hex `request_id` in `submit_request` | Marks `handled=true`, short-circuits | Never submitted on SPN, never retried |
| `disable_fulfillment=true` | Skips all SPN write RPCs | Still updates cluster state — dry-run |
| `request_probability < 1.0` deterministic skip | Hash partition `sum(request_id bytes) % 100 < probability*100` | Stable across fulfillers — meant for sharded coordinators |
| `network.init` failure | Bubbles up; exits `Fulfiller::run` | TODO: "Use backoff here" — currently a single attempt; needs process restart |
| AWS KMS init failure | `run_fulfiller` startup error | No fallback to local key |
| Bidder loop-level error | Increment `main_loop_errors`, log | No backoff, next 3s tick proceeds |
| Bidder mode disabled (`!groth16_enabled` or `!plonk_enabled`) | `can_fulfill_proof` rejects | Aggressive mode skips this check too |

## Flow-Specific Pitfalls

- [Pitfall] Bidder ordering: biddable query must precede assigned query — otherwise a proof transitioning Requested→Assigned between the two calls is missed by both.
- [Pitfall] `active_proofs` is only incremented locally in non-aggressive mode; aggressive mode disables capacity tracking entirely.
- [Pitfall] `effective_throughput = throughput_mgas / max_concurrent_proofs` is an average — the bidder rejects big proofs even when the cluster is idle. Both knobs must be tuned together.
- [Pitfall] Default per-mode buffers: base 30s, Groth16 +30s, Plonk +80s. Plonk total safety floor = `completion_time + 110s`.
- [Pitfall] `REQUEST_LIMIT=1000` fulfiller vs `100` bidder — fulfiller can starve if more than 1000 requests are in any single status; bidder caps at 100 biddable + 100 assigned per tick.
- [Pitfall] `time_now` used as `minimum_deadline` everywhere — expired proofs disappear from every query including `handled=false`. A late-finishing cluster proof past deadline never gets its `FulfillProof` sent and never gets `handled=true`, but SPN may already be Unfulfillable. Intentional but leaks tasks on the cluster side past deadline.
- [Pitfall] `fail_request` on cluster Failed status uses `error=None`; network sees generic cancel rather than typed error. Only VK-mismatch path carries error code 2.
- [Pitfall] `MainnetFulfiller::fail_request_with_error` swallowing 4xx tonic codes masks legitimate auth/argument bugs alongside the intended race tolerance.
- [Pitfall] `Unfulfillable` query in `get_cancelable_requests` does NOT filter by `execution_status` — includes any Unfulfillable. Dedup with `Assigned+Unexecutable` query relies on `seen.insert(request_id)`.
- [Pitfall] `copy_artifacts` is hardcoded `true` in `run_fulfiller`; the `Fulfiller::new` flag has no env switch.
- [Pitfall] Private stdin: `stdin_public_uri` is empty when `stdin_private=true`; cluster artifact key is derived from SPN-internal URI (not the presigned URL) so keys match across sides.
- [Pitfall] `extract_artifact_name` splits on `/` and takes last segment — query strings would corrupt the name; only safe for the canonical S3 URL in `stdin_uri`.
- [Pitfall] Reverse-then-filter-then-sample schedule order means newest first; `request_probability < 1.0` partitioning is stable across fulfillers — assumes all configure the same probability.
- [Pitfall] Per-fulfiller-address parallel scheduling shares the same `cluster_requests` HashSet snapshot; two addresses can both try to schedule the same SPN request only if both have it Assigned (rare; SPN assigns to one) but cluster `create_proof_request` enforces uniqueness by `proof_id`.
- [Pitfall] `name`/`scheduled_by` filtering on submit/fail/list queries means fulfillers without a `name` set only see untagged cluster requests; mixing named and unnamed fulfillers on the same coordinator silently drops work.
- [Pitfall] `download_artifact` uses `FULFILLER_NETWORK_S3_REGION` (default `us-east-2`); region mismatch with the network bucket silently fails the artifact copy.
- [Pitfall] `Fulfiller::run` has no inter-phase delay — a slow `submit_requests` postpones cancel/schedule by an unbounded amount within the same cycle.
- [Pitfall] Bidder signs synchronously via `PrivateKeySigner` — no AWS KMS path. Fulfiller has KMS. Operators wanting KMS for one but not the other must split keys.
