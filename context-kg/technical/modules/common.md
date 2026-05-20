---
name: "common"
description: "Module design for crates/common: proto, gRPC client, logger, consts, util"
---
# Common Crate (`sp1-cluster-common`)

## Responsibilities

- Aggregate the cross-binary shared modules: `client`, `consts`, `logger`, `proto`, `util`.
- Re-export proto enums/messages from `sp1_prover_types::cluster` and `sp1_prover_types::worker` so all binaries import via `sp1_cluster_common::proto`.
- Provide `ClusterServiceClient` wrapping tonic-generated stubs with a bounded 10s retry budget for every method.
- Provide `reconnect_with_backoff` building tonic `Channel` with TLS gated on `https://` prefix, `keep_alive_while_idle=true`, http2 keepalive 15s, 60s call timeout, tcp keepalive 15s.
- Provide `logger::init` (OTLP + tracing_subscriber, Once-guarded; silences `p3_*` noise; forces `sp1_*`/`slop_*`/`sp1_gpu_*` to debug).
- Provide `task_weight(task_type)` and `lazy_static!` env-tunable per-task weights (`WORKER_CONTROLLER_WEIGHT`, `WORKER_GROTH16_WRAP_WEIGHT`, `WORKER_PLONK_WRAP_WEIGHT`, `WORKER_EXECUTE_ONLY_WEIGHT`, `WORKER_CORE_EXECUTE_WEIGHT`).
- Provide `backoff_retry`, `status_to_backoff_error`, `BackoffError`, `NoopNotify`, `get_private_ip` utility helpers.

## NOT Responsible For

- Business logic — pure shared infrastructure.
- Coordinator/scheduler-specific types — keep this crate free of bin/coordinator imports.
- Artifact storage.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `ClusterServiceClient` | tonic Channel + bounded retry helpers | Used by coordinator, fulfiller, network-gateway |
| `reconnect_with_backoff` | initial 100 ms, max 4 s, no elapsed limit | Idempotent rustls ring install |
| `task_weight(task_type)` | maps `TaskType` → GB | Source-of-truth for scheduler/limiter |
| `logger::init` | `OTLP_ENDPOINT` (default `http://localhost:4317`), `TRACING_ENABLED`, `LOGGING_ENABLED`, `TOKIO_BLOCKED_BUSY_MS` | Once-guarded |
| `backoff_retry`, `status_to_backoff_error`, `BackoffError`, `NoopNotify` | tonic retry abstractions | Maps Internal/Unavailable/Unknown/Cancelled/DeadlineExceeded/ResourceExhausted/Aborted/DataLoss to transient |
| `get_private_ip` | first private IPv4 via `if_addrs` | Used by workers self-registering |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- N/A — shared infrastructure consumed by all flows.

## Module-Specific Pitfalls

- [Pitfall] `logger::init` is `Once`-guarded; calling it twice is a no-op but you must still pass the correct `Resource` (service.name etc.) on the first call.
- [Pitfall] Proto enums/messages MUST be imported through `sp1_cluster_common::proto`, not `sp1_prover_types` directly — keeps re-exports coherent.
- [Pitfall] `ClusterServiceClient`'s bounded 10s retry is tuned for the cluster's expected RTT; raising it can hide upstream outages.
- [Convention] Task weights are env-tunable via `WORKER_*_WEIGHT`; never hardcode weights in callers.
