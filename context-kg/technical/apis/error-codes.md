---
name: "error-codes"
description: "tonic::Code usage per RPC; no numeric prefix scheme"
---
# Error Codes

This repo does NOT register a numeric error-code module/prefix scheme. tonic `Code` is used directly per call site; error strings are free-form and include the underlying cause via `{e}` formatting. Adding a new shared error type without coordinating with all four services (bin/api, bin/coordinator, bin/network-gateway, bin/fulfiller) is discouraged.

The single exception is the SPN-facing VK-mismatch code `2` carried in `fail_request_with_error` (matches `spn_network_types::ProofRequestError::VerificationKeyMismatch`).

## Code Usage by Service

| Module Prefix | Range | Module Name | Registration Rule |
|--------------|-------|-------------|-------------------|
| `bin/api ClusterServiceImpl` | tonic::Code | `Internal`, `NotFound`, `InvalidArgument` | `proof_request_create` → `Internal` on sqlx error; `proof_request_cancel` → `NotFound` when `rows_affected==0`, `Internal` on sqlx error; `proof_request_update` → `InvalidArgument` when no fields set, `NotFound` when `rows_affected==0`, `Internal` on sqlx error; `proof_request_get` → `Internal` on any sqlx error (including not-found, via `map_err` format); `proof_request_list` → `Internal` on sqlx error; `healthcheck` → always `Ok`. |
| `bin/coordinator GenericWorkerService` | tonic::Code | `NotFound` (ack_sub unknown subscription), `FailedPrecondition` (shutting_down, fail_task on wrong worker), passes through any `Status` from `Coordinator` methods via `?` in `tokio::spawn().await.unwrap()?` | Task panics surface as tokio `JoinError` unwrap panic (not a Status); inner errors flow through unchanged. |
| `bin/network-gateway ProverNetworkImpl` | tonic::Code | `InvalidArgument` (missing request body, bad stdin_uri, bad program_uri), `Unauthenticated` (`Auth::authorize` signature parse / recovery failure), `PermissionDenied` (`Auth::authorize` allowlist miss), `FailedPrecondition` (program not registered in `ProgramStore` on `request_proof`; ELF missing in artifact store on `create_program`), `NotFound` (`load_cluster_proof` when cluster returns None; `get_program` when `ProgramStore` returns None — required by SDK NetworkClient routing), `Internal` (cluster RPC errors, artifact errors, program store I/O errors), `Unimplemented` (all unsupported ProverNetwork RPCs) | Unimplemented string convention: `"<rpc>: not supported by self-hosted gateway"`. |
| `bin/network-gateway ArtifactStoreImpl` | tonic::Code | `Unauthenticated`/`PermissionDenied` via `Auth::authorize` on `CREATE_ARTIFACT_MESSAGE = b"create_artifact"`; `Internal` on `create_artifact failed: {e}` | n/a |
| `bin/network-gateway artifact_http` | HTTP `StatusCode` | `BAD_REQUEST` (unknown artifact type segment), `NOT_FOUND` (download_raw failure — also catches non-missing backend errors), `INTERNAL_SERVER_ERROR` (upload_raw_compressed failure) | Error body is plain text. |
| `crates/fulfillment` SPN-facing | `spn_network_types::ProofRequestError::*` | `VerificationKeyMismatch = 2` (matched via `VK_MISMATCH_STRINGS`) | `VK_MISMATCH_ERROR=2` and `VK_MISMATCH_STRINGS` must stay in lockstep with `spn_network_types`. |

## Conventions Summary

- [Convention] `Status::internal(...)` is the catch-all for unexpected errors; do not introduce new generic codes without a clear semantic.
- [Convention] `Status::not_found(...)` is reserved for true "row does not exist" cases at the API layer AND for the SDK-driven `get_program` routing in the gateway.
- [Convention] `Status::failed_precondition(...)` covers state-machine violations (e.g. shutting_down, fail_task on wrong worker, program not registered).
- [Convention] `Status::invalid_argument(...)` is reserved for programmer errors (missing required fields).
- [Convention] `Status::unimplemented(...)` is the canonical refusal for unsupported SDK RPCs; the string MUST be `"<rpc>: not supported by self-hosted gateway"`.
- [Convention] Avoid embedding sensitive data in error strings (no DB connection strings, no API keys, no inner request bodies).
