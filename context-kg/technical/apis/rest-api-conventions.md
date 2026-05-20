---
name: "rest-api-conventions"
description: "gRPC + HTTP response/error conventions across sp1-cluster services"
---
# API Conventions

The cluster runs four gRPC services (tonic) plus two HTTP surfaces:

- bin/api `ClusterService` (`/healthz` via axum)
- bin/coordinator `WorkerService` (no extra HTTP)
- bin/network-gateway `ProverNetwork` + `ArtifactStore` (SDK-compatible) + HTTP `/artifacts/{type}/{id}` proxy

## Response Format

| Wrapper | Fields | Success | Error |
|---------|--------|---------|-------|
| `tonic::Response<T>` | T is a prost-generated proto struct (e.g. `ProofRequestGetResponse{proof_request}`, `ProofRequestListResponse{proof_requests: Vec<ProofRequest>}`, `CreateTaskResponse{task_id}`, `CreateProofResponse{task_id}`, `GetTaskStatusesResponse{statuses: Vec<TaskStatusMapEntry>}`, `pb::RequestProofResponse{tx_hash:[0u8;32], body: RequestProofResponseBody{request_id}}`, `pb::GetProofRequestStatusResponse{fulfillment_status, execution_status, request_tx_hash:[0u8;32], deadline, fulfill_tx_hash, proof_uri, public_values_hash, proof_public_uri}`, `pb::GetFilteredProofRequestsResponse{requests}`, `pb::GetNonceResponse{nonce}`, `pb::GetBalanceResponse{amount: String}`, `pb::GetProgramResponse{program}`, `pb::CreateProgramResponse{tx_hash:[0u8;32], body: CreateProgramResponseBody{}}`, `CreateArtifactResponse{artifact_uri, artifact_presigned_url}`) | `Ok(Response::new(T))` wrapping bare proto message; mutation RPCs return `Response<()>`; `healthcheck` returns `Response<()>` and HTTP `/healthz` returns literal `"OK"` body | `Err(tonic::Status::<code>(msg))`; SDK ProverNetwork unsupported RPCs return `Status::unimplemented("<rpc>: not supported by self-hosted gateway")` |
| Axum HTTP (`artifact_http` + `/healthz`) | `Bytes` body for GET artifact; `StatusCode` only for PUT; literal `"OK"` for `/healthz` and `/` | `Ok(StatusCode::OK)` on PUT; `Ok(impl IntoResponse)` raw bytes on GET | `(StatusCode, String)` tuple — `BAD_REQUEST` for unknown artifact type segment; `NOT_FOUND` for download failures; `INTERNAL_SERVER_ERROR` for upload failures |
| Streaming RPCs (`worker.open`, `open_sub`, `subscribe_task_messages`, `subscribe_proof_requests`) | `Response<UnboundedReceiverStream<Result<Msg, Status>>>` where `Msg` is `ServerMessage` / `ServerSubMessage` / `MessageStreamResponse` / `pb::ProofRequest`; `subscribe_proof_requests` is `Unimplemented` | Per-message `Ok(msg)` / `Err(Status)` on stream | Stream closed on error/drop; subscriber resends unacked messages up to 3 times every 2s |
| `tx_hash` convention | `RequestProofResponse.tx_hash`, `CreateProgramResponse.tx_hash`, `ProofRequest.tx_hash`, `GetProofRequestStatusResponse.request_tx_hash` | All hard-coded to `vec![0u8; 32]` (no on-chain settlement in self-hosted gateway) | n/a |

## Versioning

- No URL- or header-level versioning. gRPC service paths are proto-package-qualified (default tonic behaviour, no explicit `/v1/`). HTTP routes are unversioned (`/healthz`, `/`, `/artifacts/{type}/{id}`). No `Accept-Version`, `API-Version`, or similar header is read.
- `pb::ProofRequest.version` is set to `String::new()` in `build_sdk_proof_request` (cluster does not surface version). `pb::RequestProofRequestBody.version` is consumed from the SDK (e.g. `"v0"` in tests) but NOT persisted to the cluster `ProofRequestCreateRequest`.
- Coordinator logs `BUILD_VERSION` on startup ("coordinator (version {}) listening on {}") — build-time constant, not surfaced over the API.
- Bidder/Fulfiller send `version` to SPN (`FULFILLER_VERSION` / `BIDDER_VERSION`, e.g. `sp1-v6.0.0`) — for SPN-side matching, not cluster-side gating.
- Artifact URI shape `/artifacts/{type_seg}/{id}` (segment ∈ `unspecified|program|stdin|proof|groth16|plonk|private-stdin`) and `/programs/{vk_hash_hex}` for `get_program` response — path segment changes are breaking; `parse_artifact_type_segment` is a closed set.

## Idempotency

- `program_artifact_id(vk_hash) = "program_" + hex(vk_hash)` is deterministic and content-addressed. Used by both `request_proof` (warm-cache lookup via `exists` then conditional upload) and `create_program` (re-upload under the deterministic id). Scope: per gateway → cluster artifact store; allows multiple proofs against the same program to skip re-upload while the ELF is warm (Redis TTL = 4h). On miss, `request_proof` re-uploads from `ProgramStore`.
- `ProgramStore::put(vk_hash, vk, elf)` is durable per-gateway program registry — repeated `create_program` for the same `vk_hash` overwrites in place; orphan SDK-uploaded artifact is deleted via `client.try_delete` after persistence.
- `get_nonce` is a per-address monotonic counter in `ProverNetworkImpl::next_nonce` (`DashMap<Vec<u8>, AtomicU64>` with `fetch_add(Ordering::Relaxed)`, starting at 0). Process-local — lost on gateway restart. Intended to satisfy SDK nonce flow only; NOT validated against incoming `request_proof`/`create_program` nonces (the body `nonce` field is unread).
- `mint_request_id()` mints a fresh V7 typeid `"req_..."` string, zero-pads to exactly 32 bytes (SDK B256 requirement); `proof_id_from_request_id` strips trailing zeros to recover typeid for cluster `proof_id`. There is NO server-side dedup keyed on `(requester, nonce, vk_hash, stdin)`; each call creates a new cluster `ProofRequest`.
- `proof_request_create` relies on Postgres `id` PK uniqueness; sqlx errors (including unique violation) all collapse to `Status::internal("Failed to create proof request")` — no special `AlreadyExists` branch.
- `proof_request_cancel` only transitions `Pending → Cancelled` (`WHERE proof_status = Pending`); a no-op (`NotFound`) for non-pending states — implicit idempotency.
- `ArtifactStoreImpl::create_artifact` mints a fresh id every call; no client-supplied idempotency key. `artifact_uri == artifact_presigned_url` and PUT is content-addressed by id.

## Pagination

| Request | Response | Rule |
|---------|----------|------|
| `cluster_pb::ProofRequestListRequest` | `cluster_pb::ProofRequestListResponse{proof_requests: Vec<ProofRequest>}` | `limit: Option<u32>` default 10, capped to `min(1000)`; `offset: Option<u32>` default 0; no `total_count`, no `next_page_token`, no cursor — offset-based only. Filters: `proof_status` (Vec, IN), `execution_status` (Vec, IN), `minimum_deadline` (>=), `handled` (=), `scheduled_by` (=). |
| `pb::GetFilteredProofRequestsRequest` (SDK→gateway) | `pb::GetFilteredProofRequestsResponse{requests: Vec<ProofRequest>}` | Uses page/limit (1-based page): `limit` clamped to `1..=100`; `offset = (page-1)*limit` when both are `Some` and `page>0`, else `None`. `fulfillment_status`/`execution_status` singletons mapped through `cluster_fulfillment_filter`/`cluster_execution_filter` (SDK `Assigned` → empty filter = matches nothing); `minimum_deadline` forwarded; `vk_hash`/`requester`/`fulfiller`/`from`/`to`/`version`/`mode` filters are silently dropped. |
| `ProofRequestListRequest.handled` / `scheduled_by` | same response | Optional refine filters used by claimer/status loops in coordinator (not paginated separately). |

## Conventions Summary

- [Convention] Use `tonic::Response<T>` for unary RPCs; never wrap responses in custom envelope structs (the cluster does not have a `code/msg/data` style envelope).
- [Convention] Error responses use `tonic::Code` directly; error strings are free-form and include the underlying cause via `{e}` formatting. There is no numeric error-code module/prefix or registration registry — see `apis/error-codes.md`.
- [Convention] Mutation RPCs that don't return data still return `Response<()>` (e.g. `proof_request_update`, `proof_request_cancel`).
- [Convention] Unsupported SDK RPCs return `Status::unimplemented("<rpc>: not supported by self-hosted gateway")` (consistent string).
- [Convention] `[Reuse] tonic::Response<T>` wrapping is the only acceptable success envelope for gRPC; never introduce a parallel envelope type.
