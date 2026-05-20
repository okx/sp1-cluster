---
name: "README"
description: "Directory index and reading guide for sp1-cluster technical knowledge base"
---
# context-kg — Technical Knowledge

SP1 Cluster is a multi-GPU SP1 proving service implemented as a Rust workspace. Seven binaries (`bin/`) plus five shared crates (`crates/`) cooperate over tonic gRPC to convert SP1 proof requests into completed proofs and bridge a self-hosted cluster to the SPN ProverNetwork.

## Files and Directories

| Path | Description |
|------|-------------|
| `knowledge-base.md` | **Highest authority** — all Skills defer to this on conflicts |
| `terminology.md` | Domain glossary (TaskType, ProofRequestStatus, ArtifactType, request_id↔proof_id, etc.) |
| `arch/architecture-overview.md` | Layers and per-module responsibilities |
| `arch/dependency.md` | Upstream callers, inter-module deps, storage, external services |
| `modules/api.md` | `bin/api` — ClusterService gRPC + Postgres |
| `modules/bidder.md` | `bin/bidder` — SPN bid loop |
| `modules/cli.md` | `bin/cli` — bench + vk-gen developer CLI |
| `modules/coordinator.md` | `bin/coordinator` — cluster orchestration, WorkerService |
| `modules/fulfiller.md` | `bin/fulfiller` — SPN ↔ cluster sync binary |
| `modules/network-gateway.md` | `bin/network-gateway` — SDK-compatible ProverNetwork + ArtifactStore |
| `modules/node.md` | `bin/node` — CPU/GPU worker process |
| `modules/artifact.md` | `crates/artifact` — S3 + Redis ArtifactClient |
| `modules/common.md` | `crates/common` — proto, gRPC client, logger, consts, util |
| `modules/fulfillment.md` | `crates/fulfillment` — Fulfiller primitives + FulfillmentNetwork |
| `modules/utils.md` | `crates/utils` — request_proof helpers for CLI |
| `modules/worker.md` | `crates/worker` — SP1ClusterWorker + per-task handlers |
| `pitfalls/redis-admission.md` | Redis admission control, AdmissionGuard, TTL gotchas |
| `pitfalls/grpc-retry.md` | tonic retry semantics, infinite-vs-bounded, NotFound reconnect |
| `pitfalls/proof-failure.md` | Proof unclaim semantics, enable_proof_fail gate, race conditions |
| `pitfalls/network-gateway.md` | request_id padding, deterministic program ids, status mapping |
| `pitfalls/scheduling.md` | Worker capacity, heartbeat reassignment, retry counter |
| `core-flows/proof-request-lifecycle.md` | SDK → gateway → API → coordinator → worker → completion |
| `core-flows/worker-task-execution.md` | NewTask dispatch, run_task, retry, heartbeat |
| `core-flows/fulfiller-bidder.md` | SPN bid + submit/fail/cancel/schedule cycles |
| `core-flows/artifact-io.md` | S3/Redis upload/download, admission, cross-bucket copy |
| `apis/rest-api-conventions.md` | gRPC + HTTP response/error conventions across services |
| `apis/error-codes.md` | tonic::Code usage per RPC; no numeric prefix scheme |
| `conventions/feature-types.md` | Env-prefixed Settings, metrics derive, span propagation |
| `conventions/service-patterns.md` | Retry, reconnect, admission, idempotency patterns |
| `conventions/common-tools.md` | Reusable helpers: backoff_retry, reconnect_with_backoff, ArtifactClient, etc. |
| `repo_brief.json` | Machine-readable repo summary (stack, modules, business) |

## How to Read This Knowledge Base

1. **`knowledge-base.md` is the highest authority** — defer to it over general AI knowledge
2. **Locate the specific module** — read the module doc before starting work
3. **Check `pitfalls/` and `core-flows/` first** — most cross-module bugs live there
4. **Produce a constraint checklist** — explicitly declare if no relevant content applies
5. **Cross-validate during work** — correct violations immediately
