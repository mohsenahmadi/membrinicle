# CLAUDE.md

## What this repository is

Membrinicle is a local-first personal memory engine. It indexes files and web pages, stores everything on the local machine using brinicle (a disk-first C++ HNSW search engine), and serves accumulated knowledge to AI tools via MCP and REST API.

Remote: git@github.com:mohsenahmadi/membrinicle.git

---

## Before any session

1. Read `docs/2026-05-27-system-design.md` for the full architecture and build plan.
2. Check current phase and open work before writing any code.
3. Never install dependencies or write code outside a feature branch.
4. All brinicle engine access in the daemon goes through `indexer.py` only.
5. All brinicle calls must go through a single `asyncio.Lock`.
6. `uvicorn` must always be launched with `workers=1`. brinicle index files have no inter-process locking. Two worker processes sharing the same index path will corrupt it silently.
7. Log decisions and changes at the end of each session in `docs/YYYY-MM-DD-session.md`.

---

## Engineering standard

Every rule below is binding. The test for each decision: can this be debugged at 2am with no context beyond what is written here?

### Naming

- Full words only. No abbreviations. `requestCorrelationId` not `corrId`. `isIndexReady` not `ready`.
- Name for intent, not type. `fetchResultsByQuery` not `getResults`.
- Booleans as assertions. `isIndexReady`, `hasPermission`, `canWrite`. Never `ready`, `permission`, `write`.
- Constants in SCREAMING_SNAKE_CASE. `MAX_BATCH_SIZE`, `SETUP_WINDOW_SECONDS`.
- No magic literals. Every value with meaning is a named constant.

### Python

- Python 3.11, typed throughout with `from __future__ import annotations`.
- `TypedDict` for all structured data shapes. No `Any` anywhere.
- Guard clauses first - validate and return at the top, happy path at the bottom.
- No boolean parameters that change behavior. That is two functions pretending to be one.
- Functions stay under 30 lines. If longer, extract.
- Functions that mutate state are named to say so: `updateWatchedFolders`, not `processConfig`.
- Pydantic models for all HTTP request and response bodies. Do not trust data that crosses an HTTP boundary without validation.
- Typed error classes for all domain failures. Never raise bare `Exception` or strings.
- Structured JSON logs on every error:

```python
logger.error("Failed to finalize index batch", extra={
    "action": "finalize_batch",
    "correlation_id": batch_id,
    "batch_size": len(batch),
    "error": str(error),
})
raise
```

- Never swallow exceptions. Log with context, then re-raise.
- No `TODO` or `FIXME` in code. Use the issue tracker.
- No commented-out code. Git history has it.
- Comments explain WHY only, one line maximum. If deleting the comment would not confuse a competent engineer, delete it.

### TypeScript

- `strict: true` in tsconfig. No exceptions.
- No `any`. `unknown` with type guards at every message boundary.
- Explicit return types on all exported functions.
- Zod for all external input validation at Chrome extension message boundaries.
- All `chrome.*` API calls wrapped in typed helpers in `chrome_api.ts`.
- No boolean parameters that change behavior.

### Testing

- Every test follows AAA: Arrange, Act, Assert.
- Never mock brinicle engines - use real engines against a `tmp_path` fixture.
- Never mock internal classes. Mock only true external boundaries.
- Flaky tests do not merge.

### Git

Conventional Commits with scope, always:

```
feat(indexer): add delete_items support with optimize_graph
fix(watcher): handle MOVED events atomically under lock
refactor(extractor): extract pdf parsing to dedicated function
test(phase0): add V6 delete and rebuild verification
docs(adr): record brinicle engine selection decision
chore(deps): add pymupdf and python-docx to pyproject
```

One logical change per commit. If the message needs "and", split the commit. No "WIP", "misc", or "fixes" messages.

### Pull requests

- Soft limit: 400 lines. Hard limit: 600 lines. Over 400 requires justification in the PR description.
- One concern per PR. Mixed concerns are returned without review.
- All CI checks must pass before requesting review.
- Merge and delete the feature branch within 24 hours of approval.

### Architecture decisions

Write an ADR in `docs/adr/` for every significant architectural choice: new dependency, infrastructure component, or decision with trade-offs that will matter later. Link the ADR from the PR that introduces the change. See existing ADRs in `docs/adr/` for the required format.

### Observability

- Structured JSON logs only in daemon production paths. No bare `print()`.
- Every HTTP request receives a `X-Correlation-ID` header (generated in middleware if absent). Pass it through all log entries for that request.
- Every catch block logs `action`, `correlation_id`, and `error` at minimum.
- No dead exports, no orphan modules. If a function has no consumer, delete it.

### Security

- No secrets or credentials in code or version control.
- Pydantic validation on all HTTP request bodies at the FastAPI layer.
- Rate-limit all endpoints that accept external input.
- `pip-audit` runs in CI. Vulnerable dependencies are treated as P1 bugs.

---

## CI checks (required to pass before merge)

```bash
ruff check daemon/ tests/
black --check daemon/ tests/
pytest tests/
grep -r "import brinicle" daemon/ | grep -v "indexer.py" && exit 1 || exit 0
```

---

## Repository layout

```
/
  LICENSE
  COMMERCIAL.md
  README.md
  CLAUDE.md
  pyproject.toml
  daemon/
    server.py
    watcher.py
    extractor.py
    indexer.py
    notifier.py
    mcp.py
    config.py
  web/
    index.html
    app.js
    style.css
  extension/
    src/
      content.ts
      background.ts
      popup.ts
      chrome_api.ts
    manifest.json
    tsconfig.json
  installer/
    build_dmg.sh
    LaunchAgent.plist.template
  tests/
    test_phase0.py
    test_indexer.py
    test_extractor.py
    test_server.py
  docs/
    adr/
    2026-05-27-system-design.md
```
