---
name: "artifact-io"
description: "Core flow: Artifact upload/download (S3 + Redis) with admission control and cross-bucket copy"
---
# Artifact Upload/Download Flow

How proof inputs and outputs (`program`, `stdin`, `proof`, `groth16`/`plonk` circuits, `private-stdin`) move through the cluster: SDK uploads via gateway HTTP, workers and fulfiller download/upload via `ArtifactClient`, cross-bucket copies use `CompressedUpload::upload_raw_compressed` to avoid double-zstd.

## Entry Point

| Entry | Handler | Primary Entities |
|-------|---------|------------------|
| gRPC `ArtifactStore::create_artifact` | `ArtifactStoreImpl::create_artifact` (`bin/network-gateway/src/service/artifact_store.rs`) | Auth-signed `b"create_artifact"`; returns `artifact_uri` + `artifact_presigned_url` |
| HTTP `PUT /artifacts/{type_seg}/{id}` | `artifact_http::put_artifact` (`bin/network-gateway/src/artifact_http.rs`) | `DefaultBodyLimit::disable()`; calls `upload_raw_compressed` (body is already zstd-compressed) |
| HTTP `GET /artifacts/{type_seg}/{id}` | `artifact_http::get_artifact` | Calls `download_raw`; returns decompressed bytes |
| `ArtifactClient` trait | `crates/artifact/src/lib.rs` (re-export from `sp1_prover_types`) | `upload_raw`, `download_raw`, `exists`, `delete`, `delete_batch`, `add_ref`/`remove_ref` (Redis), `acquire_shard_permit` (Redis), `create_artifact`, `try_delete` |
| `CompressedUpload` trait | `crates/artifact/src/lib.rs` | `upload_raw_compressed(artifact, type, data)` — bypasses application-layer zstd |

## Primary Entities

`S3ArtifactClient` (`crates/artifact/src/s3.rs`), `RedisArtifactClient` (`crates/artifact/src/redis.rs`), `InMemoryArtifactClient` (re-export), `S3DownloadMode { AwsSDK, REST }`, `ShardPermit`, `AdmissionGuard`.

## State Transitions

### Redis backend (per artifact id, sharded by FNV-1a)

| Entity | Transition | Trigger | Terminal |
|--------|-----------|---------|----------|
| Artifact | (none) → Written-Small | single `SET <key> data EX 14400` | no |
| Artifact | (none) → Written-Chunked | `HSET <key>:chunks idx chunk` + `EXPIRE <key>:chunks 14400` (>32 MiB) | no |
| Artifact | (none) → Written-Persistent | `ArtifactType::Program` write (no `EX`, no `EXPIRE`) | no (until explicit delete) |
| Artifact | Written-* → Expired | Redis TTL fires after 14400 s (non-Program only) | yes |
| Artifact | Written-* → Deleted | `delete` / `delete_batch` / `try_delete` → `UNLINK` | yes |
| `refs:{id}` set | None → Tracked | `add_ref` SADD + EXPIRE 14400 s | no |
| `refs:{id}` set | Tracked → cleaned | `remove_ref` SREM + SCARD ≤ 0 → DEL + `try_delete` | yes |
| Per-node admission | None → Reserved | bytes > `ADMISSION_BYPASS_BYTES` and budget check passes | no |
| Per-node admission | Reserved → Released | `AdmissionGuard` drops | yes |
| `ShardPermit` | None → Held | `acquire_shard_permit` | no |
| `ShardPermit` | Held → Released | permit drop | yes |

**[Rule] Terminal Redis states**: Deleted, Expired (non-Program), Written-Persistent (Program until explicit delete).

### S3 backend (per id under `<prefix>/<id>` key)

| Entity | Transition | Trigger | Terminal |
|--------|-----------|---------|----------|
| Artifact | (none) → Uploaded | `par_upload_file` (single PutObject ≤32 MiB or multipart) | no |
| Artifact | Uploaded → Cross-Bucket-Copied | source `download_raw` decompresses (or raw fetch) → target `upload_raw_compressed` | no |
| Artifact | Uploaded → Deleted | `DeleteObject` via `delete` / `delete_batch` | yes |
| Artifact | Uploaded → Expired | S3 lifecycle rule on prefix | yes |

**[Rule] Terminal S3 states**: Deleted, Expired (per-prefix lifecycle), Uploaded-Persistent (Groth16/PlonkCircuit at bucket root).

## Normal Flow Steps

| Step | Action |
|------|--------|
| 1 | SDK obtains a writable URI: gRPC `ArtifactStore::create_artifact` validates signature (`Auth::authorize` on `b"create_artifact"`), backend mints id via `client.create_artifact()`, response carries `artifact_uri = artifact_uri(public_http_url, type, id)` |
| 2 | SDK bincode-serializes payload then zstd-compresses (level 3) and HTTP-PUTs the bytes to `/artifacts/{type_seg}/{id}` |
| 3 | Gateway `put_artifact`: parses type segment, calls `client.upload_raw_compressed(&id, type, body.to_vec())` — body is treated as pre-compressed, no second zstd pass |
| 4a | S3 path: `par_upload_file` — `data.len() <= 32 MiB` → single `PutObject`; else `CreateMultipartUpload` → parallel `UploadPart` (32 MiB chunks) → `CompleteMultipartUpload`. Key via `get_s3_key_from_id(type, id)` (`programs/`, `stdins/`, `proofs/`, `private-stdins/`; circuits use `<id>-{groth16|plonk}.tar.gz` at bucket root) |
| 4b | Redis path: `upload_to_transport` — first `check_admission(key, data.len())` (skipped if ≤ `ADMISSION_BYPASS_BYTES`) acquires per-node `decide` Mutex, reads `INFO memory` for `(used, maxmemory)`, computes `budget = maxmemory * MEMORY_BUDGET_FRACTION (0.80)`, ensures `used + in_flight + incoming ≤ budget`; if not, releases lock, sleeps `ADMISSION_POLL_INTERVAL=250ms`, retries up to `ADMISSION_MAX_WAIT=120s`; on admit, `in_flight.fetch_add(bytes)` and returns `AdmissionGuard`. Then `backoff_retry` wraps `par_upload_file` with per-attempt `TRANSFER_TIMEOUT=60s`. `len <= 32 MiB` → `SET <id> data EX 14400` (no EX for Program); else chunked `HSET <id>:chunks idx chunk` + `EXPIRE` (HSETEX unavailable on managed Redis). Guard releases bytes on drop |
| 5 | Download: client `GET /artifacts/{type_seg}/{id}` → gateway `download_raw` |
| 6a | S3: `par_download_file` → HEAD via `S3DownloadMode` to get size, then single GET or parallel ranged GETs (`Range: bytes=start-end`); each wrapped in `backoff::future::retry(BACKOFF)` (100 ms initial, 120 s max elapsed, 7 max attempts via SDK retry config); chunks copied into output `Vec<u8>` at exact offsets; `zstd::decode_all` |
| 6b | Redis: `par_download_file` → `HLEN <id>:chunks`; if 0 → `GET <id>`; else parallel `HGET <id>:chunks idx` JoinSet, reassemble in index order; 60 s per-attempt `tokio::time::timeout` + `backoff_retry`; `zstd::decode_all` |
| 7 | Cross-bucket / cross-backend copy: caller downloads compressed bytes once (or uses `download_raw` only when decompressed form is needed), then writes via `CompressedUpload::upload_raw_compressed` on the target client — avoids double-zstd |

### Reference-counted lifetime (Redis only)

- Producer `add_ref(artifact, task_key)`: SADD into `refs:{id}` then EXPIRE 14 400 s
- Consumer `remove_ref`: SREM, SCARD; on count ≤ 0, DEL the ref set and call `try_delete` to UNLINK both `<id>` and `<id>:chunks`; returns true

### Shard permit lifetime (`ProveShardGate`)

- `acquire_shard_permit` → `node_semaphore(idx).acquire_owned()` (lazily sized on first call); closed semaphore → `ShardPermit::noop()` (fail-open)
- Tests confirm: gate drop aborts pending release tasks and reclaims permits; only `Succeeded` releases (not `FailedRetryable`)

## Exception Branches

| Scenario | State Change | Compensation |
|----------|--------------|--------------|
| S3 HEAD `content_length` missing | Error "failed to get content size" | Propagates to caller |
| S3 ranged GET error | `backoff::Error::permanent(anyhow!(e))` | Retries handled by AWS SDK retry policy, not `BACKOFF` |
| S3 multipart `upload_part().await.unwrap()` panic | Spawned task panic propagates as `panic!("artifact download thread panicked")` | Process crashes |
| S3 `head_object` 404 in `exists()` | Returns `Ok(false)` | Caller decides |
| `validate_config` finds shard with `maxmemory==0` | Returns error refusing startup | Operator must `CONFIG SET maxmemory <N>gb` |
| `INFO memory` timeout (5 s) | Error "INFO memory timed out" | Refuses startup |
| `check_admission` parse miss / unlimited | Returns no-op guard (fail-open) | |
| `check_admission` Err on `query_memory` | Warn log "admission INFO memory failed; fail-open"; no-op guard | Backpressure silently disabled |
| Wait exceeds `ADMISSION_MAX_WAIT=120s` | Error "Redis shard {idx} admission wait > ..." | Periodic warn every 10s while blocked |
| Per-attempt upload timeout | `backoff::Error::transient` | After retries exhaust: "Upload operation timed out ..." or "Upload failed for artifact {id}: {err}" |
| `acquire_shard_permit` on closed semaphore | Warn "semaphore closed, releasing unbounded permit"; returns `noop()` | Fail-open |
| Gateway unknown `type_seg` | `400 BAD_REQUEST` "unknown artifact type: {seg}" | |
| Gateway upload failure | `500 INTERNAL_SERVER_ERROR` "internal error: {e}" | |
| Gateway download failure | `404 NOT_FOUND` "download failed: {e}" | Conflates missing artifact with backend errors |

## Flow-Specific Pitfalls

- [Pitfall] Cross-bucket re-uploads must use `CompressedUpload::upload_raw_compressed`. Calling `upload_raw` for already-zstd bytes double-compresses, wasting CPU and breaking the SDK's symmetric pipeline.
- [Pitfall] `AdmissionGuard` is `#[must_use]`. Binding to `_` drops it immediately and reintroduces the TOCTOU race. Use `_guard`.
- [Pitfall] Writes ≤ `ADMISSION_BYPASS_BYTES` (~2 MiB) skip admission entirely — load-bearing for tail-deadlock prevention so outputs can land while inputs are pinned.
- [Pitfall] The permit-pool / admission ceiling gap is intentional: `INPUT_BUDGET_FRACTION = 0.50 < MEMORY_BUDGET_FRACTION = 0.80`. Closing the gap deadlocks the pipeline.
- [Pitfall] `validate_config` must be called at startup. Skipping it lets the lazy permit-sizing path silently floor at 4 permits per node when `maxmemory=0`.
- [Pitfall] Redis admission fails open on `INFO memory` errors — sick Redis silently disables backpressure. Watch the warn log.
- [Pitfall] `ArtifactType::Program` is special-cased to skip TTL entirely on Redis. Adding a new persistent type requires extending the `matches!` check in both `par_upload_file` branches; S3 handles persistence per-prefix via lifecycle policies.
- [Pitfall] S3 multipart upload's `.unwrap()` on `upload_part().await` and `tx.send()` panics the spawned task on per-part failures — in effect crashes the process rather than returning a retryable Err.
- [Pitfall] Redis download retry has `max_elapsed_time = 1 s` on the `BACKOFF` — at most one timeout shot despite 60 s per-attempt. Increase carefully if expanding retries.
- [Pitfall] `Groth16Circuit` / `PlonkCircuit` artifacts live at bucket root, not under a prefix. S3 lifecycle rules keyed on prefix won't catch them.
- [Pitfall] Gateway returns `404 NOT_FOUND` for ANY backend download error (transport blip, zstd decode failure, true missing), not only "not found". Logs are the only disambiguation.
- [Pitfall] `refs:{id}` set sharding key MUST always be the artifact id, not a derived label, or refs and artifacts diverge across nodes.
- [Pitfall] `ShardPermit::noop()` on closed semaphore is intentional fail-open — semaphore poisoning silently disables OOM protection.
- [Pitfall] `DefaultBodyLimit::disable()` is required for artifact PUT because SP1 ELFs/stdins are hundreds of MB. Don't re-introduce a body-limit layer; backpressure must come from the artifact store.
- [Pitfall] `CreateArtifactResponse.artifact_presigned_url` is currently the same URL as `artifact_uri` — there is no real presigning today. SDK clients distinguishing the two will silently get plain URLs.
