---
name: "bidder"
description: "Module design for bin/bidder: SPN bid loop"
---
# Bidder Module

## Responsibilities

- Boot via `dotenv` + `rustls::crypto::ring::default_provider().install_default()`; load `Settings::new()` (env prefix `BIDDER`).
- Build `ProverNetworkClient` to `BIDDER_RPC_GRPC_ADDR` via local `configure_endpoint` (mirrors `crates/fulfillment::grpc::configure_endpoint`: 10s timeout, 5s connect, http2 keepalive 10s/10s, tcp 30s).
- Build `PrivateKeySigner::from_str(BIDDER_SP1_PRIVATE_KEY)`; resolve prover address once via `GetOwnerRequest{address=signer.address()}`.
- Start `spn-metrics` server on `BIDDER_METRICS_ADDR` (default `0.0.0.0:9061`).
- Run `Bidder::run` loop every `REFRESH_INTERVAL_SEC = 3s` calling `bid_requests(prover)` — exits on ctrl_c or task error.

## NOT Responsible For

- Fulfillment (separate binary `bin/fulfiller`).
- Cluster API/coordinator/workers/artifacts.
- Producing proofs.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `Bidder` | `client: ProverNetworkClient`, `signer: PrivateKeySigner`, `domain: B256`, `version: String`, `bid_amount: u64`, `throughput_mgas: u64`, `max_concurrent_proofs: usize`, `buffer_sec`, `groth16_buffer_sec`, `plonk_buffer_sec`, `groth16_enabled`, `plonk_enabled`, `aggressive_mode`, `min_deadline_secs` | Single struct holding all bid policy state |
| `Settings` (bidder) | env-prefixed `BIDDER_*` | Loaded via `Config::builder().add_source(Environment::with_prefix("BIDDER"))` |
| `BidderMetrics` | `biddable_requests` gauge, `requests_bid` counter, `request_bid_failures` counter, `main_loop_errors` counter | `#[metrics(scope="bidder")]` |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/fulfiller-bidder.md`.

## Module-Specific Pitfalls

- [Pitfall] `bid_requests` MUST query biddable proofs (`status=Requested, not_bid_by=prover`) BEFORE assigned-to-us proofs (`status=Assigned, fulfiller=prover`). Reversing the order misses any proof that transitions Requested→Assigned between the two calls.
- [Pitfall] `aggressive_mode == true` bypasses `can_fulfill_proof` capacity/throughput checks; only `min_deadline_secs` (if set) still applies. Reserve for permissioned environments — uncapped throughput can drain accounts.
- [Pitfall] `active_proofs` is incremented locally only in non-aggressive mode; in aggressive mode capacity tracking is meaningless.
- [Pitfall] `effective_throughput = throughput_mgas / max_concurrent_proofs` is an *average* — the bidder rejects large proofs even when the cluster is idle. Tuning both knobs is required to bid.
- [Pitfall] Default buffers: base `buffer_sec=30`, `groth16_buffer_sec=30`, `plonk_buffer_sec=80`. Plonk total safety floor is `completion_time + 110s`.
- [Pitfall] `request hashing for probability` (in fulfillment) uses additive byte hash mod 100 — fine for sampling but never use for security decisions.
- [Pitfall] `REQUEST_LIMIT=100` (vs fulfiller's 1000) — bidder caps at 100 biddable + 100 assigned per tick.
- [Pitfall] `bid_requests` signs `BidRequestBody{nonce, request_id, amount=bid_amount, domain, prover, variant=BidVariant}` synchronously via `PrivateKeySigner` — there is no AWS KMS path. Operators wanting KMS for fulfillment but local for bidding must split keys.
- [Convention] Loop-level errors increment `main_loop_errors` and log — no backoff, next 3s tick proceeds.
