---
name: "network-gateway"
description: "Module design for bin/network-gateway: SDK-compatible ProverNetwork + ArtifactStore"
---
# Network Gateway Module

## Responsibilities

- Terminate SP1 SDK's `ProverNetwork` + `ArtifactStore` gRPC contract (`sp1_sdk::network::proto::base`) and proxy to bin/api via `ClusterServiceClient` + `ArtifactClient`.
- Host HTTP artifact proxy at `/artifacts/{type_seg}/{id}` (PUT uses `upload_raw_compressed`, GET uses `download_raw`); `DefaultBodyLimit::disable()` because SP1 ELFs are hundreds of MB.
- Raise tonic `max_decoding_message_size` and `max_encoding_message_size` to `64 MiB` on both servers — SP1 vkeys exceed the 4 MiB default.
- Implement auth via `Auth::authorize`: modes `None` / `Verify` / `Allowlist`; recovers EIP-191 signer address from request body for `request_proof`, `create_program`, `create_artifact`; read-only RPCs are unauthenticated.
- Maintain a durable `ProgramStore` (`memory` | `fs` backed by `GATEWAY_PROGRAM_STORE_DIR`) keyed by `vk_hash`; warm-cache the ELF in the cluster artifact store under `program_artifact_id(vk_hash)`.
- Map cluster proto enums ↔ SDK proto enums via `bin/network-gateway/src/status.rs` (`fulfillment_from_cluster`, `execution_from_cluster`, `cluster_fulfillment_filter`, `cluster_execution_filter`).

## NOT Responsible For

- Proving — no task execution, no coordinator/worker calls.
- Bidding or fulfillment — gateway never speaks to SPN.
- Persistent nonce tracking — per-address counters are in-memory only.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `ProverNetworkImpl` | `cluster_client: ClusterServiceClient`, `artifact_client: Arc<dyn ArtifactClient + CompressedUpload>`, `program_store: Arc<dyn ProgramStore>`, `auth: Arc<Auth>`, `nonces: DashMap<Vec<u8>, AtomicU64>`, `domain: B256`, `public_http_url: String` | Implements SDK's `ProverNetwork` trait |
| `ArtifactStoreImpl` | `artifact_client: Arc<dyn ArtifactClient + CompressedUpload>`, `auth: Arc<Auth>`, `public_http_url: String` | Implements SDK's `ArtifactStore` trait |
| `Auth` | `AuthMode { None, Verify, Allowlist(HashSet<Address>) }` | EIP-191 recovery; `None` mode recovers `Address::ZERO` |
| `FilesystemProgramStore` | `dir: PathBuf` | Atomic writes via `<file>.tmp` + `fs::rename`; format: 4-byte LE length prefix + vk + ELF |
| `InMemoryProgramStore` | `DashMap<B256, (Vec<u8>, Vec<u8>)>` | Volatile; loses programs on restart |
| `mint_request_id`, `proof_id_from_request_id`, `program_artifact_id`, `artifact_uri`, `parse_artifact_type_segment`, `artifact_type_segment` | helpers in `ids.rs` | ID minting + round-trip for SDK ↔ cluster |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/proof-request-lifecycle.md` and `core-flows/artifact-io.md`.

## Module-Specific Pitfalls

- [Pitfall] `mint_request_id` MUST produce exactly `REQUEST_ID_LEN = 32` bytes; SDK decodes via `B256::from_slice` which panics on length mismatch. Typeid string is asserted ≤ 30 bytes before zero-padding.
- [Pitfall] `proof_id_from_request_id` MUST strip trailing zero bytes (`rposition` over non-zero bytes); zeros leaking into the cluster `proof_id` will not match the typeid stored in Postgres.
- [Pitfall] `get_program` MUST return `Status::not_found` for unknown vk_hash, never `Ok` with empty fields — SDK's `NetworkClient::get_program` only routes to `create_program` on `NotFound`.
- [Pitfall] `request_proof` hot path: `client.exists(&program_artifact_id, Program)` first; only on TTL miss reloads from `ProgramStore` and re-uploads under the deterministic id. If `ProgramStore` lost the ELF, returns `FailedPrecondition`. Don't reorder.
- [Pitfall] `create_program` cleans up the orphan SDK-uploaded ephemeral artifact via `try_delete` AFTER persisting to durable store + warm-keyed cluster id. Skipping the cleanup wastes storage.
- [Pitfall] Unsupported SDK RPCs MUST return `Status::unimplemented("<rpc>: not supported by self-hosted gateway")` — fabricating values would break SDK retry/escalation flows.
- [Pitfall] `get_filtered_proof_requests` silently drops `vk_hash` / `requester` / `fulfiller` / `from` / `to` / `version` / `mode` filters — callers relying on them get unfiltered results.
- [Pitfall] `status::cluster_*_filter` maps SDK `Assigned`/`UnspecifiedFulfillmentStatus` to empty filter vectors (intentional no-op); cluster has no `Assigned` analogue.
- [Pitfall] `next_nonce` uses `DashMap<address, AtomicU64>` with `Relaxed` ordering and is NOT persisted; gateway restart resets all nonces. Don't rely on them for replay protection across restarts.
- [Pitfall] `artifact_type_segment` ↔ `parse_artifact_type_segment` MUST be a bijection; network-only ArtifactTypes (`Transaction`, `State`, `Export`) fold to `UnspecifiedArtifactType` on cluster projection.
- [Pitfall] `request_proof` body's `nonce` field is unread by the gateway — there is NO server-side dedup by `(requester, nonce, vk_hash, stdin)`. Each call mints a new cluster `ProofRequest`.
- [Pitfall] `Auth` recovers `Address::ZERO` in `None` mode regardless of signature; don't treat zero address as malicious in that mode.
- [Pitfall] `CreateArtifactResponse` returns the same URL for both `artifact_uri` and `artifact_presigned_url` — there is no real presigning today.
- [Pitfall] Tonic graceful shutdown 2s sleep before signaling stop may cut long-running RPCs.
- [Convention] `Status::unimplemented` is the canonical refusal for unsupported SDK RPCs; error strings include the RPC name.
