---
name: "cli"
description: "Module design for bin/cli: developer CLI for bench + vk-gen"
---
# CLI Module

## Responsibilities

- Clap-based command-line tool exposing `bench` and `vk-gen` subcommands.
- Loads `.env` via `dotenv` and the SP1 SDK logger at startup.
- Uses `crates/utils::request_proof` helpers to create a `ProofRequestCreateRequest` via `ClusterServiceClient`, upload program/stdin artifacts, poll `get_proof_request`, and download the completed proof artifact.
- Local-only — not part of the runtime data plane.

## NOT Responsible For

- Hosting any server.
- Direct cluster orchestration (delegates to `crates/utils`).
- Production deployment workflows (use `infra/charts/sp1-cluster` Helm chart).

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `commands::bench` | runs a benchmark proof | Local developer tool for end-to-end timing |
| `commands::vk_gen` | generates verifying-key map | Calls `process_sp1_generate_vk_*` via the cluster |
| Clap args | mirrors env defaults | Args take precedence over env |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/proof-request-lifecycle.md` for the request_proof path.

## Module-Specific Pitfalls

None identified.
