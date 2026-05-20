---
name: "feature-types"
description: "Recurring infrastructure feature patterns: settings, metrics, span propagation"
---
# Feature Types (Recurring Infra Patterns)

## Env-Prefixed Settings

- [Convention] Every binary defines a `Settings` (or `FulfillerSettings`) `Deserialize` struct loaded via `Config::builder().add_source(Environment::with_prefix("COORDINATOR" | "BIDDER" | "FULFILLER" | "GATEWAY" | "API" | "WORKER" | "NODE"))`.
- [Convention] Supports `#[serde(default = "...")]` for optional knobs and `with_list_parse_key(...).with_list_separator(",")` for comma-separated lists (addresses, redis nodes).
- [Convention] `spn_utils::deserialize_domain` for `B256` domain; `spn_utils::LogFormat` for log format.
- [Convention] Per-prefix defaults: API default addr `127.0.0.1:3000` (HTTP) / `127.0.0.1:50051` (gRPC); coordinator default `127.0.0.1:50051`; bidder defaults `buffer_sec=30`, `groth16_buffer_sec=30`, `plonk_buffer_sec=80`, `groth16_enabled=true`, `plonk_enabled=true`.

## Metrics Derive

- [Convention] `spn_metrics::Metrics` derive on per-binary struct: `#[derive(Metrics, Clone)] #[metrics(scope = "...")]` with scope `"worker"` / `"coordinator"` / `"fulfiller"` / `"bidder"`.
- [Convention] Cardinal labels emitted via `counter!` / `gauge!` / `histogram!` with `describe_*!` macros.
- [Convention] `initialize_metrics`: `MetricServerConfig::new(addr, VersionInfo, service_name).with_ready_signal(oneshot)`; spawn server, await `ready_rx`, then describe metrics. Returns `(Arc<Metrics>, Option<JoinHandle>, broadcast::Sender<()> shutdown)`.

## VersionInfo

- [Convention] Standard `VersionInfo` boilerplate at top of `initialize_metrics`: `env!("CARGO_PKG_VERSION")` + `VERGEN_BUILD_TIMESTAMP` + `VERGEN_GIT_SHA` + `VERGEN_CARGO_FEATURES` + `VERGEN_CARGO_TARGET_TRIPLE` + debug/release. Coordinator uses `option_env!` for `CARGO_*`/`TARGET` while worker uses `env!()` — keep the asymmetry intentional (worker has `vergen-git2` build.rs; coordinator uses its own chrono-based stamp).

## dotenv at Boot

- [Convention] Every binary calls `dotenv()` first in `main` and swallows the missing-file error (`if let Err(e) = dotenv() { eprintln!(...) }`).

## rustls ring Provider

- [Convention] `rustls::crypto::ring::default_provider().install_default()` must be installed once per process before any TLS gRPC connection; idempotent install in `reconnect_with_backoff` plus explicit calls in `main.rs` of fulfiller/bidder/network-gateway/node.

## OpenTelemetry Span Propagation

- [Convention] Producers inject opentelemetry trace context via `task_metadata(&context)` serialized as JSON in `TaskData.metadata`.
- [Convention] Consumers extract context via `with_parent` to set parent span.
- [Convention] `crates/worker/src/utils.rs::get_global_gpu_span` returns a shared GPU `info_span!` re-created every 120s for grouping GPU-related work; treat the span as ephemeral (don't cache across long awaits).

## Pagination

- [Convention] Both cluster `ProofRequestListRequest` and SDK-facing `GetFilteredProofRequestsRequest` use offset-based pagination. No cursors. Cluster: `limit` default 10, capped at 1000. SDK: `limit` clamped to 1..=100, offset computed from `(page-1)*limit`.

## Async Helpers

- [Convention] `tokio::select!` over `channel.recv()` + ticker + `shutdown_rx.changed()` is the standard worker loop shape.
- [Convention] `crates/worker/src/utils.rs::conditional_future` awaits `Option<Future>` inside `tokio::select!` arms.
- [Convention] `crates/worker/src/utils.rs::DeferGuard` is preferred for ad-hoc RAII cleanup over a manual `Drop` impl.
