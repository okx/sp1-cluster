---
name: "artifact"
description: "Module design for crates/artifact: ArtifactClient + CompressedUpload backends"
---
# Artifact Crate (`sp1-cluster-artifact`)

## Responsibilities

- Re-export `ArtifactClient`, `ArtifactId`, `ArtifactType`, `InMemoryArtifactClient` from `sp1_prover_types`.
- Define `CompressedUpload` trait extending `ArtifactClient` for passthrough of pre-compressed (zstd) bytes — required for cross-bucket / gateway re-upload paths that would otherwise double-zstd.
- Provide `S3ArtifactClient` (zstd level 3, 32 MiB multipart, `S3DownloadMode::AwsSDK` or `REST`) and `RedisArtifactClient` (zstd level 0, deadpool-redis multi-node sharding via FNV-1a, 4h TTL except `ArtifactType::Program`, admission control with `#[must_use] AdmissionGuard`).
- Provide admission control: per-node `Semaphore` (`ShardPermit`) + admission ledger gated by `MEMORY_BUDGET_FRACTION = 0.80`; permits sized from `INFO memory` × `INPUT_BUDGET_FRACTION = 0.50` / `max_shard_bytes`, floored at `MIN_PERMITS_PER_NODE = 4`; bypass for writes ≤ `ADMISSION_BYPASS_BYTES` (~ 2 MiB).

## NOT Responsible For

- Knowing about cluster proto or coordinator state.
- Hosting any gRPC service.
- Reference-counted lifetime in S3 backend (only Redis has `refs:{id}`).

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `S3ArtifactClient` | `aws-sdk-s3 Client`, `bucket`, `region`, `concurrency: Arc<Semaphore>`, `download_mode: S3DownloadMode` | Persistent backend with `RetryConfig::standard` (7 attempts), 120s op timeout, 20s `StalledStreamProtection`, 300s identity-cache buffer |
| `RedisArtifactClient` | per-shard `deadpool-redis::Pool[]`, `node_semaphores[]`, `admission_states[]`, `max_shard_bytes` | Ephemeral; refs via `refs:{id}` set; admission via `decide` Mutex + `in_flight: AtomicU64`; per-attempt `TRANSFER_TIMEOUT = 60s` |
| `ShardPermit` | per-node semaphore token; closed semaphore yields `noop()` | Acquired via `acquire_shard_permit`; held until artifact released |
| `AdmissionGuard` | `#[must_use]` RAII guard | Drops decrement `in_flight`; binding to `_` reintroduces TOCTOU race |
| `CompressedUpload` (trait) | `upload_raw_compressed(artifact, type, data)` | Implemented for `S3ArtifactClient`, `RedisArtifactClient`, `InMemoryArtifactClient` (delegates to `upload_raw`) |
| `S3DownloadMode` | `AwsSDK` / `REST` | Selects `S3SDKClient` or `S3RestClient` transport |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/artifact-io.md`.

## Module-Specific Pitfalls

- [Pitfall] `AdmissionGuard` is `#[must_use]`; binding to `_` drops it immediately and breaks the admission invariant. Inside `upload_to_transport` the comment explicitly mandates `_guard`.
- [Pitfall] `RedisArtifactClient::validate_config()` MUST be called at startup; refuses to start when any shard has `maxmemory == 0`. Without it, the permit pool floors at `MIN_PERMITS_PER_NODE = 4` and caps cluster throughput.
- [Pitfall] The static `assert!(INPUT_BUDGET_FRACTION < MEMORY_BUDGET_FRACTION)` is load-bearing: closing the gap deadlocks the pipeline when inputs saturate because unpermitted outputs also block on admission.
- [Pitfall] Redis admission FAILS OPEN on `INFO memory` errors — a sick Redis silently disables backpressure. Watch the warn log "admission INFO memory failed; fail-open".
- [Pitfall] `ArtifactType::Program` is special-cased to skip TTL on Redis (no `EX` on `SET`, no `EXPIRE` on hash). Adding a new persistent type requires extending both `par_upload_file` branches.
- [Pitfall] HSETEX is NOT supported on managed Redis (e.g. AWS ElastiCache) — multi-chunk uploads must use `HSET` + `EXPIRE` on the hash key.
- [Pitfall] zstd level differs by transport: Redis uses level 0 (ephemeral, 4h TTL); S3 uses level 3 (persistent). Don't unify.
- [Pitfall] `CompressedUpload::upload_raw_compressed` skips compression — only invoke from cross-bucket paths where bytes are already zstd-compressed.
- [Pitfall] S3 multipart upload calls `.unwrap()` on `upload_part().await` and `tx.send()`; failures panic the spawned task (in effect crashes the process). Download path explicitly `panic!("artifact download thread panicked")`.
- [Pitfall] Redis download's `BACKOFF` has `max_elapsed_time = 1s` — at most ~1 second of retry budget despite a 60s per-attempt timeout. Increase carefully if expanding retries.
- [Pitfall] S3 `Groth16Circuit` / `PlonkCircuit` artifacts live at bucket root (`<id>-{groth16|plonk}.tar.gz`), not under a prefix. S3 lifecycle rules keyed on prefix won't catch them.
- [Pitfall] Network Gateway's `GET /artifacts` returns `404 NOT_FOUND` for ANY backend download error (transport blip, zstd decode failure, true missing), not only missing artifacts. Logs (`error!`) are the only place to disambiguate.
- [Pitfall] `ShardPermit::noop()` on closed semaphore is intentional fail-open — semaphore poisoning silently disables OOM protection.
- [Pitfall] Reference-counted lifetime via `refs:{id}` set is sharded by FNV-1a of artifact id. Sharding key in `get_redis_connection(id)` must always be the artifact id, not a derived label.
- [Convention] Bytes ≤ `ADMISSION_BYPASS_BYTES` skip admission entirely; load-bearing for tail-deadlock prevention so consumers can free input artifacts.
