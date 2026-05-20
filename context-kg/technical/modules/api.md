---
name: "api"
description: "Module design for bin/api: ClusterService gRPC backed by Postgres"
---
# API Module

## Responsibilities

- Implements `ClusterService` gRPC: `proof_request_create`, `proof_request_cancel`, `proof_request_update`, `proof_request_get`, `proof_request_list`, `healthcheck`.
- Owns the Postgres `proof_requests` table via `sqlx::PgPool`; optionally runs sqlx migrations on `API_AUTO_MIGRATE=true`.
- Serves gRPC on `API_GRPC_ADDR` (with `tonic_web::enable`) and HTTP `/healthz` on `API_HTTP_ADDR` via axum.

## NOT Responsible For

- Task assignment, worker orchestration, or scheduling — that is the coordinator's responsibility.
- Communicating with SPN (`spn-network-types`) — fulfiller and bidder do that.
- Hosting artifacts — `crates/artifact` + S3/Redis backends are independent.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `proof_requests` (Postgres table) | `id TEXT PRIMARY KEY`, `proof_status INT`, `requester BYTEA`, `execution_status INT`, `execution_failure_cause INT`, `execution_cycles BIGINT`, `execution_gas BIGINT`, `execution_public_values_hash BYTEA`, `stdin_artifact_id TEXT`, `program_artifact_id TEXT`, `proof_artifact_id TEXT`, `options_artifact_id TEXT`, `cycle_limit BIGINT`, `gas_limit BIGINT`, `deadline BIGINT`, `handled BOOLEAN`, `metadata TEXT`, `created_at TIMESTAMPTZ`, `updated_at TIMESTAMPTZ`, `extra_data TEXT`, `scheduled_by TEXT`, `stdin_private BOOLEAN NOT NULL DEFAULT FALSE` | Single canonical row for each proof request |
| `DbProofRequest` (in `service.rs`) | sqlx-derived struct mirroring `proof_requests` | Converted to proto `ProofRequest` via `DbProofRequest::into_proto` |
| `ClusterServiceImpl` | `db_pool: Arc<PgPool>` | Sole collaborator — no coordinator, worker, or SPN clients |
| `idx_proof_requests_list` | composite `(handled, deadline, proof_status)` | Backs the coordinator/fulfiller list filter |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/proof-request-lifecycle.md` for the create/claim/complete path.

## Module-Specific Pitfalls

- [Pitfall] `proof_request_create` returns `Status::internal("Failed to create proof request")` for any sqlx error — including unique-key violations. There is no `AlreadyExists` branch; callers must not rely on a typed error code for duplicate ids.
- [Pitfall] `proof_request_cancel` is idempotent only over `Pending`: `WHERE proof_status = Pending`. A row already in `Cancelled`/`Completed`/`Failed` returns `NotFound` (rows_affected == 0); a row already claimed by the coordinator stays running until coordinator-side cleanup runs.
- [Pitfall] `proof_request_get` folds row-not-found into `Status::internal` via `map_err` formatting — it does not return `NotFound`. Callers should not depend on the code; check the error message.
- [Pitfall] `proof_request_update` returns `Status::invalid_argument` when no fields are set; failing to provide at least one mutable field is a programmer error, not a transient one.
- [Pitfall] `API_AUTO_MIGRATE=true` runs `sqlx::migrate!("../../migrations").run(&pool)` at boot — in shared multi-instance deployments, only one instance should have this set or migrations race.
- [Convention] List pagination is offset-based: `limit` defaults to 10, capped to 1000; no total_count or cursor token. Callers iterate by incrementing offset.
