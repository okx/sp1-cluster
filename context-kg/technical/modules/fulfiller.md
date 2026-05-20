---
name: "fulfiller"
description: "Module design for bin/fulfiller: SPN ↔ cluster sync binary"
---
# Fulfiller Module

## Responsibilities

- Boot via `dotenv` + `rustls::crypto::ring::default_provider().install_default()`; load `FulfillerSettings::new("FULFILLER")`.
- Dispatch artifact backend on `FULFILLER_CLUSTER_ARTIFACT_STORE` (`s3` default | `redis`); for Redis call `validate_config()` before run.
- Build `ProverNetworkClient` to `FULFILLER_RPC_GRPC_ADDR` (default mainnet) via `configure_endpoint`; wrap in `MainnetFulfiller`.
- Build cluster `ClusterServiceClient` to `FULFILLER_CLUSTER_RPC` via `reconnect_with_backoff`.
- Build signer: `NetworkSigner::aws_kms` when `FULFILLER_USE_AWS_KMS=true`, else `NetworkSigner::local`.
- Delegate to `crates/fulfillment::run_fulfiller` which spawns `Fulfiller::run` (Arc-wrapped) with `copy_artifacts = true`.

## NOT Responsible For

- Bidding (separate binary `bin/bidder`).
- Executing proofs.
- Coordinator orchestration.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `MainnetFulfiller` | `client: ProverNetworkClient`, `signer-state`, `s3_concurrency` | `FulfillmentNetwork` impl against SPN mainnet. |
| `FulfillerSettings` | `cluster_rpc`, `cluster_artifact_store`, `cluster_s3_{region,bucket}`, `cluster_redis_nodes`, `rpc_grpc_addr`, `sp1_private_key`, `use_aws_kms`, `version`, `domain`, `name`, `refresh_interval_sec`, `request_probability`, `disable_fulfillment`, `addresses`, `log_format`, `download_concurrency` | Loaded via `config` crate, `FULFILLER_` env prefix |
| `LogFormat` (from `spn_utils`) | `Pretty` / `Json` | Configured via `FULFILLER_LOG_FORMAT` |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/fulfiller-bidder.md`.

## Module-Specific Pitfalls

- [Pitfall] `rustls::crypto::ring::default_provider().install_default()` is called once in `main` and will panic on duplicate install if other components also install — keep it the single install site.
- [Pitfall] Artifact backend dispatch panics on unknown `FULFILLER_CLUSTER_ARTIFACT_STORE`; s3 requires `FULFILLER_CLUSTER_S3_REGION` + `FULFILLER_CLUSTER_S3_BUCKET`; redis requires `FULFILLER_CLUSTER_REDIS_NODES`.
- [Pitfall] s3 backend defaults `download_concurrency=32` and uses `S3DownloadMode::AwsSDK`; redis backend defaults pool size 16 and requires `validate_config()` before run.
- [Pitfall] AWS KMS init failure aborts startup with no fallback to local key.
- [Pitfall] `network.init(&signer)` is currently a single attempt with no backoff (TODO in code) — a transient network blip at startup forces a process restart.
- [Convention] Fulfiller binary owns only startup wiring; all loop logic lives in `crates/fulfillment::Fulfiller`.
