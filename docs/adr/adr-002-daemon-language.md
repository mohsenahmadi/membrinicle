---
created: 2026-05-27
updated: 2026-05-27
type: reference
status: accepted
tags: [adr, architecture, python]
---
# ADR-002: Python 3.11 as the daemon language

## Status

Accepted

## Context

The daemon must integrate directly with brinicle, which is distributed as a pip package with a Python C extension binding (`brinicle/_brinicle*.so`). The binding requires a compatible Python runtime. The HTTP server, file watcher, and macOS menu bar each have mature Python libraries available. The daemon runs as a single background process with no concurrency requirements beyond what asyncio provides.

## Decision

Python 3.11 as the daemon language. brinicle's `pyproject.toml` declares `requires-python = ">=3.10"` and ships prebuilt wheels for 3.11 on PyPI. FastAPI, uvicorn, watchdog, pymupdf, python-docx, and rumps all support Python 3.11.

Note: brinicle's own build toolchain (`build.sh`, Dockerfile) targets Python 3.14. Phase 0 verification (V0) must confirm that the prebuilt 3.11 wheel installs cleanly before any product code is written. If V0 fails, this decision is revised to Python 3.14.

All Python code uses `from __future__ import annotations`, full type annotations, and `TypedDict` for structured data shapes. No `Any`. Pydantic is used for all HTTP request and response validation at the FastAPI layer.

## Alternatives considered

- **Python 3.14:** brinicle's own target. Would guarantee build compatibility from source. Rejected for now because 3.14 is not yet released as stable and most deployment tooling (PyInstaller, rumps) targets 3.11. Revisit if V0 fails.
- **Rust:** No brinicle Python bindings exist for Rust. Would require reimplementing the engine integration from scratch. Rejected.
- **Go:** Same as Rust. No brinicle bindings. Rejected.

## Consequences

What gets easier: direct brinicle integration with no FFI layer, mature ecosystem for every required library, straightforward PyInstaller packaging.

What gets harder: the GIL means brinicle's synchronous C++ calls block the asyncio event loop during ingest and finalize. The single-lock strategy accepts this for v1. If startup scan performance proves unacceptable, the migration path is `run_in_executor` with a `threading.Lock`.
