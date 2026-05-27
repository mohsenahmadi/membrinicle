# Membrinicle

A local-first personal memory engine. It indexes the files you own and the web pages you read, stores everything on your machine using [brinicle](https://github.com/bicardinal/brinicle) (a disk-first C++ HNSW search engine), and serves your accumulated knowledge to any AI tool via MCP and REST API.

Nothing leaves your machine.

## The problem it solves

Every AI session starts from zero. You re-explain context, re-reference documents you have already read, and re-surface knowledge you have already encountered. Membrinicle closes that gap by making your own knowledge continuously available to any AI tool, privately.

## Status

Phase 0 - design complete, verification in progress. See `docs/2026-05-27-system-design.md` for the full architecture and build plan.

## License

PolyForm Noncommercial 1.0.0 - free for personal use. See `LICENSE` and `COMMERCIAL.md`.
