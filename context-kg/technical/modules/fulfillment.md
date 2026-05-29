---
name: "fulfillment"
description: "Module design for crates/fulfillment: Fulfiller primitives + FulfillmentNetwork trait"
---
# Fulfillment Crate (`sp1-cluster-fulfillment`)

## Responsibilities

- Define `Fulfiller<A, N>` (`A: ArtifactClient + CompressedUpload`, `N: FulfillmentNetwork`) and its run loop: `submit_requests → fail_requests → cancel_requests → schedule_requests → sleep(refresh_interval)`.
- Define `FulfillmentNetwork` trait abstracting SPN ProverNetwork access (`fetch_stdin_uri`, `should_download_proofs`, `submit_request`, `cancel_request`, `fail_request`, `get_owner`, etc.).
- Provide `configure_endpoint` (10s timeout, 5s connect, http2 keepalive 10s/10s, tcp 30s) for SPN-side gRPC.
- Provide metrics struct (`#[metrics(scope="fulfiller")]`) exposing per-RPC success and failure counters plus `cluster_unexecuted_requests` gauge.
- Encapsulate VK-mismatch heuristic: `VK_MISMATCH_ERROR = 2`, `VK_MISMATCH_STRINGS = ["InvalidPowWitness", "sp1 vk hash mismatch", "vk hash from syscall does not match vkey from input"]`.

## NOT Responsible For

- Task assignment, executing proofs, bidding.
- Owning persistent state — `bin/api` owns Postgres, network owns SPN state.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `Fulfiller<A,N>` | `name: Option<String>`, `cluster_client: ClusterServiceClient`, `cluster_artifact_client: Arc<A>`, `network: Arc<N>`, `copy_artifacts: bool`, `refresh_interval: Duration`, `request_probability: f64`, `addresses: Vec<Address>`, `disable_fulfillment: bool`, `download_concurrency: usize` | Loop driver |
| `FulfillerSettings` (re-export) | see `modules/fulfiller.md` | Loaded via `Environment::with_prefix("FULFILLER")` |
| `FulfillmentNetwork` (trait) | `init`, `get_owner`, `submit_request`, `cancel_request`, `fail_request`, `fail_request_with_error`, `get_schedulable_requests`, `get_cancelable_requests`, `fetch_stdin_uri`, `should_download_proofs` (default `true`) | Implemented by `MainnetFulfiller` (`bin/fulfiller`) and executor binary (overrides `should_download_proofs() = false`) |
| `NetworkSigner` | `local(PrivateKeySigner)` or `aws_kms(KmsClient, key_id)` | Used to sign `FulfillProofRequestBody`, `GetStdinUriRequestBody`, etc. |
| `ProofResultMetadata` | proof completion metadata | Forwarded into `extra_data` when scheduling/completing |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/fulfiller-bidder.md`.

## Module-Specific Pitfalls

- [Pitfall] When `Fulfiller.name` is set, every cluster query MUST filter by `scheduled_by=name`. Mixing named and unnamed fulfillers on the same coordinator silently drops work because unnamed fulfillers won't see tagged rows and vice versa.
- [Pitfall] VK-mismatch detection is string-based: matching any of `VK_MISMATCH_STRINGS` in the SPN error message triggers `fail_request_with_error(Some(2))`. Adding new mismatch wording requires extending the list to keep parity with `spn_network_types::ProofRequestError::VerificationKeyMismatch`.
- [Pitfall] `MainnetFulfiller::fail_request_with_error` swallows tonic `PermissionDenied | FailedPrecondition | NotFound | InvalidArgument` from `FailFulfillment` as warn — designed to tolerate races with SPN already terminalizing, but it also masks legitimate auth/argument bugs.
- [Pitfall] Cancel query covers both `(Assigned + Unexecutable)` AND pure `Unfulfillable` shapes and dedupes by `request_id` in a `HashSet` because status transitions race the two parallel queries. Don't collapse to a single query.
- [Pitfall] `copy_artifacts` is hardcoded `true` in `run_fulfiller`; there is no env switch despite the `Fulfiller::new` flag.
- [Pitfall] Private stdin requires `ArtifactType::PrivateStdin` + presigned URL from `fetch_stdin_uri`; cluster S3 key prefix `private-stdins/` must match SPN-side prefix.
- [Pitfall] `time_now` expects monotonic wall-clock — backward NTP jumps panic with "time went backwards".
- [Pitfall] `extract_artifact_name` splits on `/` and takes the last segment; trailing slashes are NOT handled. Used on `stdin_uri` (canonical S3 URL), not the presigned URL.
- [Pitfall] `request_probability < 1.0` filter is deterministic hash `sum(request_id bytes) % 100 < probability*100` — stable across fulfillers, meant for "shared coordinator, partitioned workers". Assumes all fulfillers configure the same probability.
- [Pitfall] `Fulfiller::run` has no inter-phase delay — a slow `submit_requests` postpones cancel/schedule by an unbounded amount within the same cycle.
- [Pitfall] `REQUEST_LIMIT=1000` per phase: fulfiller can starve if any single status exceeds 1000 requests at once.
- [Convention] `should_download_proofs()` default `true`; the executor binary overrides to `false`. Don't introduce other variants without coordinating with the executor.
