---
created: 2026-05-27
updated: 2026-05-27
type: reference
status: accepted
tags: [adr, architecture, search, brinicle]
---
# ADR-001: brinicle as the search and autocomplete engine

## Status

Accepted

## Context

Membrinicle needs a vector search engine that can index tens of thousands of local files and web pages, run on a personal laptop with constrained RAM, persist its index to disk across restarts, and operate without an external embedding model or network calls.

Standard options (FAISS, hnswlib, usearch) are in-memory and require loading the full index on startup, which is prohibitive for large collections on low-RAM machines. They also require an external model to generate vectors.

## Decision

Use [brinicle](https://github.com/bicardinal/brinicle) (Apache 2.0) as the sole search backend.

brinicle provides three engines: `ItemSearchEngine`, `AutocompleteEngine`, and `VectorEngine`. This product uses the first two. brinicle is disk-first - the index lives on disk and is streamed during search, keeping RAM usage bounded regardless of index size. It includes an internal `LexicalEncoder` that converts text fields into search vectors without any external model.

`ItemSearchEngine` is initialized with `dim=96, alpha=0.0` for lexical-only mode. `AutocompleteEngine` is initialized with `dim=48` (the library default for this engine). Both engines are owned exclusively by `indexer.py` and accessed under a single `asyncio.Lock`.

## Alternatives considered

- **FAISS + sentence-transformers:** FAISS is in-memory, requires loading the full index on startup. sentence-transformers requires an external model download and CPU/GPU inference on every ingest and search. Rejected: too heavy for a local background daemon.
- **hnswlib:** In-memory only. Does not persist to disk natively. Rejected: requires a full reload on every restart.
- **SQLite FTS5:** Full-text search, not vector search. No semantic ranking capability even in hybrid mode. Rejected: weaker retrieval and no upgrade path to vector mode.

## Consequences

What gets easier: no embedding model dependency, bounded RAM, no network calls, simple lifecycle (`init` - `ingest` - `finalize`), native upsert deduplication.

What gets harder: all code that uses brinicle must go through `indexer.py` only. The C++ calls are synchronous and block the asyncio event loop during finalize - the single-lock strategy accepts this for v1. A change to `dim` requires a full index rebuild.
