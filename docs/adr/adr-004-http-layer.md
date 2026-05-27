---
created: 2026-05-27
updated: 2026-05-27
type: reference
status: accepted
tags: [adr, architecture, fastapi, http]
---
# ADR-004: FastAPI and uvicorn as the HTTP layer

## Status

Accepted

## Context

The daemon needs an HTTP server for three purposes: receiving ingest requests from the Chrome extension, serving search and autocomplete results to local clients, and hosting the MCP endpoint for AI tool integration. The server must handle concurrent HTTP requests while the asyncio event loop also runs the background indexer task. A single-process, single-worker model is required because brinicle index files have no inter-process locking.

## Decision

FastAPI with uvicorn as the HTTP layer. Both are declared as optional server dependencies in brinicle's own `pyproject.toml` (`fastapi`, `uvicorn[standard]`, `pydantic`). brinicle's reference implementation uses the same stack. This means no additional dependency risk - the stack is already validated by the library we are building on.

All request and response bodies use Pydantic models. `uvicorn` is always launched with `workers=1`. This constraint is hard-coded in `server.py` and enforced as a rule in CLAUDE.md. Two worker processes sharing the same brinicle index path will corrupt the index silently.

## Alternatives considered

- **Flask:** Synchronous by default, requires additional work to integrate with asyncio. No native Pydantic integration. Rejected.
- **aiohttp:** Async, but less ergonomic for route definition and no native Pydantic. Rejected.
- **Starlette (raw):** FastAPI is a thin layer over Starlette and adds Pydantic integration and automatic OpenAPI docs at no cost. No reason to use Starlette directly.
- **Custom TCP server:** No HTTP client compatibility, no MCP protocol support. Rejected.

## Consequences

What gets easier: Pydantic models provide free request validation, automatic OpenAPI documentation, and alignment with the engineering standard's boundary-validation requirement. brinicle's own reference server uses this stack - any brinicle-specific behavior is already tested against it.

What gets harder: the single-worker constraint means the server cannot scale horizontally. For a local daemon serving one user on localhost, this is not a limitation. It becomes a limitation if Membrinicle is ever deployed as a multi-user service - at that point this ADR must be superseded.
