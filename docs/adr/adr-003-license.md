---
created: 2026-05-27
updated: 2026-05-27
type: reference
status: accepted
tags: [adr, architecture, license]
---
# ADR-003: PolyForm Noncommercial 1.0.0 license

## Status

Accepted

## Context

Membrinicle is a personal tool with commercial potential. The goal is to allow free individual use while retaining the ability to charge organizations and commercial deployers. Standard open-source licenses (MIT, Apache 2.0) permit unrestricted commercial use. A fully proprietary license prevents community contributions and trust. A middle path is needed.

The underlying brinicle engine is Apache 2.0. Any license applied to Membrinicle must be compatible with Apache 2.0, meaning it must permit downstream use of the Apache-licensed dependency without violating its terms.

## Decision

PolyForm Noncommercial 1.0.0 for the Membrinicle codebase. Apache 2.0 is compatible as an upstream dependency - Membrinicle may use brinicle under its terms while Membrinicle itself carries the more restrictive PolyForm Noncommercial license.

Individual and personal use is free and unrestricted. Commercial or organizational use requires written permission from the copyright holder, obtained via email to mohsen@ahmadi.pm. The process is documented in `COMMERCIAL.md`.

## Alternatives considered

- **MIT / Apache 2.0:** Permits unrestricted commercial use. Rejected: loses the ability to monetize organizational deployment.
- **GPL v3:** Copyleft, source-available, but permits commercial use. Rejected: copyleft requirements conflict with potential commercial licensing.
- **Proprietary / source-available with no license:** Maximally restrictive. Rejected: blocks community use and trust entirely.
- **BSL (Business Source License):** Converts to open source after a time period. More complex than needed for a personal project. Rejected.

## Consequences

What gets easier: individual users can use, share, and contribute freely. Revenue potential from organizations is preserved.

What gets harder: some developers will not contribute to noncommercially licensed projects. The license must be reviewed before any major distribution channel (app stores, package registries) to confirm compliance.
