---
name: "utils"
description: "Module design for crates/utils: request_proof helpers for CLI"
---
# Utils Crate (`sp1-cluster-utils`)

## Responsibilities

- Provide high-level developer-facing helpers (currently consumed by `bin/cli`) that wrap the full request-proof flow:
  - Construct `ProofRequestCreateRequest` via `ClusterServiceClient`.
  - Upload program / stdin artifacts via the chosen `ArtifactClient`.
  - Poll `get_proof_request` until terminal.
  - Download the completed proof artifact.
- Re-export common types (e.g. `ArtifactClient`, `ClusterServiceClient`) for CLI use.

## NOT Responsible For

- Server-side flows in API/coordinator/node/fulfiller/bidder/gateway.
- Bidder or fulfiller business logic.

## Core Entities

| Entity | Key Fields | Description |
|--------|-----------|-------------|
| `request_proof` helpers | program + stdin bytes, mode, deadline | End-to-end developer entry point |

## Dependencies

- Require to reference `arch/dependency.md` for full dependency details.

## Relevant Flows

- Require to reference `core-flows/proof-request-lifecycle.md` for the request_proof path.

## Module-Specific Pitfalls

None identified.
