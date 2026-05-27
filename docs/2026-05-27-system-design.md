---
created: 2026-05-27
updated: 2026-05-27
type: project
status: seedling
tags: [membrinicle, design, architecture, brinicle, personal-memory, local-first, mcp]
---
# Membrinicle System Design

See also: [[CLAUDE]]

---

## Problem

Every AI session starts from zero. You re-explain context, re-reference documents you have already read, and re-surface knowledge you have already encountered. The AI has no memory of you between sessions, no access to your files, and no awareness of what you have been working on. All the context that would make AI genuinely useful - notes, research, reading history - sits scattered across your filesystem and browser, disconnected from the tools that could use it.

---

## What it is

A local-first personal memory engine that runs silently in the background on your computer. It indexes the files you own and the web pages you read, stores everything on your machine using brinicle (a disk-first C++ HNSW search engine with bounded RAM usage), and serves your accumulated knowledge to any AI tool via MCP and REST API. Nothing leaves your machine.

Underlying engine: [brinicle](https://github.com/bicardinal/brinicle) - Apache 2.0, pip-installable, C++ core with Python bindings.

---

## License

**LICENSE file:** PolyForm Noncommercial 1.0.0

Individual and personal use is free and unrestricted. Commercial use, organizational deployment, or any use intended to generate revenue is prohibited without prior written permission from the copyright holder.

**COMMERCIAL.md file:** Organizations wishing to deploy Membrinicle must request permission by email to mohsen@ahmadi.pm. A simple email confirmation from the copyright holder is sufficient for non-revenue organizational use. Revenue-generating use requires a separate written agreement.

This license is compatible with brinicle's Apache 2.0 license.

---

## What brinicle provides

Three engines. `ItemSearchEngine` and `AutocompleteEngine` are Python wrappers in `brinicle/item_search.py` and `brinicle/autocomplete_search.py`. Both wrap `VectorEngine` (the C++ core in `brinicle/_brinicle`). All three share the same lifecycle: `init(mode)` - one or more `ingest()` calls - `finalize()`.

**ItemSearchEngine** is the primary engine for this product. Constructor:

```python
ItemSearchEngine(
    index_path,
    dim=96,
    vector_dim=0,        # 0 = lexical-only; no external embedding needed
    alpha=0.0,           # 0.0 = full lexical weight; MUST be set explicitly for v1
    M=16,
    ef_construction=200,
    ef_search=64,
)
```

`alpha=0.0` is mandatory for v1. The constructor default is `alpha=0.95`, which scales all lexical field weights (title, category, subcategory, attributes) down to approximately 5% of their base values - correct only when external semantic vectors dominate. With `vector_dim=0` and no vectors at ingest, `alpha=0.95` silently degrades retrieval quality. Lexical-only mode requires `alpha=0.0` explicitly.

`ingest(external_id, title, category=None, subcategory=None, attributes=None, vector=None)` accepts a structured record. `finalize(optimize=False)` - the `optimize=True` flag triggers graph optimization inline, equivalent to calling `optimize_graph()` after finalize.

`search(query, k=10) -> list[str]` returns a list of `external_id` strings only. `search_with_distance(query, k=10) -> list[tuple[str, float]]` returns (external_id, distance) tuples.

`delete_items(external_ids: list[str])` removes records by id. `needs_rebuild() -> bool` returns True when the deletion ratio has crossed the threshold that degrades graph quality. `optimize_graph()` compacts the graph in place. `rebuild_compact()` performs a full rebuild with compact parameters - more aggressive than `optimize_graph()`, appropriate after large bulk deletions.

`has_index: bool` property - True if a persisted index exists at `index_path`. Use this at startup to distinguish a cold start (no existing index, full scan required) from a warm reload (index present, scan only files newer than `last_scan_time`).

`close()` releases file handles. Always call on daemon shutdown.

Native upsert: same `external_id` on a second `ingest()` call replaces the previous record. No manual delete before re-indexing.

**AutocompleteEngine** provides query suggestion. Constructor:

```python
AutocompleteEngine(
    index_path,
    dim=48,   # 48 is the library default for autocomplete; correct for this engine
    M=16,
    ef_construction=200,
    ef_search=64,
)
```

`dim=48` is the AutocompleteEngine default. It is independent of ItemSearchEngine's `dim=96`. The two engines have separate index files and separate dimensions. Do not force `dim=96` on AutocompleteEngine.

`ingest(external_id: str, text: str)` - `external_id` is what `search()` returns. `text` is what gets vectorized for matching. To make autocomplete return readable title strings: set `external_id=item.title` and `text=item.title`. `search()` then returns title strings, not file paths.

`search(query, k=10) -> list[str]` returns the `external_id` values of the top-k matches. Same `delete_items()`, `needs_rebuild()`, `optimize_graph()`, `rebuild_compact()`, `has_index`, and `close()` methods as ItemSearchEngine.

**VectorEngine** (C++ core) is not used directly. Both high-level engines delegate to it internally.

Dependencies included with brinicle: `numpy`, `tokenizers`, `orjson`. Optional server deps declared by brinicle: `fastapi`, `uvicorn[standard]`, `pydantic`. Python 3.10+ required by brinicle; this product targets Python 3.11.

---

## Phase 0 - Verification before writing any code

These eight checks must pass before any Phase 1 code is written. Each produces a recorded result in `docs/phase0-results.md`. No assumption made in this document proceeds to code until confirmed against the real library.

**V0 - Python 3.11 installation**

```bash
python3.11 -m pip install brinicle
python3.11 -c "import brinicle; print(brinicle.ItemSearchEngine.__module__)"
```

Expected: no error, the module path prints. brinicle's build toolchain targets Python 3.14. The `pyproject.toml` declares `requires-python = ">=3.10"`, so a prebuilt wheel for 3.11 should exist on PyPI. If the install fails with "no matching distribution," source compilation is required - and brinicle's `build.sh` hardcodes `python3.14`, so you cannot build from source on 3.11 without modifying the build script. Record whether the prebuilt wheel installs cleanly. If it does not, switch the target to Python 3.14 and update this document and CLAUDE.md before any other Phase 0 test runs.

**V1 - search() return format**

```python
import brinicle

engine = brinicle.ItemSearchEngine("test_indexes/v1", dim=96, alpha=0.0)
engine.init("build")
engine.ingest(
    external_id="test-1",
    title="Q2 planning meeting",
    category="document",
    subcategory="md",
    attributes={}
)
engine.finalize()

results = engine.search("planning meeting", k=5)
print(type(results), results)

results_d = engine.search_with_distance("planning meeting", k=5)
print(type(results_d), results_d)
```

Expected: `results` is a list of strings. `results_d` is a list of (str, float) tuples. "test-1" appears in both. If the format differs, update the web UI display logic before Phase 7.

**V2 - Full file path as external_id**

```python
long_id = "/Users/testuser/Library/Application Support/deeply/nested/path/to/document-with-long-name.pdf"
engine.init("upsert")
engine.ingest(
    external_id=long_id,
    title="Long path test",
    category="document",
    subcategory="pdf",
    attributes={}
)
engine.finalize()

results = engine.search("long path", k=5)
assert long_id in results, "Full path not returned as external_id"
```

Expected: the full path is returned verbatim. If brinicle imposes a length limit, the fallback is `sha256(path)` as external_id with a `path_index.json` file mapping hashes back to paths. Record the limit if found.

**V3 - Optional vector parameter**

```python
import numpy as np

engine.init("upsert")
engine.ingest(
    external_id="vector-test",
    title="Vector test",
    category="document",
    subcategory="md",
    attributes={},
    vector=np.random.randn(96).astype(np.float32)
)
engine.finalize()
```

Expected: no error. If this raises a TypeError, the hybrid mode upgrade path is not available and must be redesigned.

**V4 - Concurrent search during upsert finalize**

```python
import threading
import brinicle

# Step 1: build an initial searchable index
engine = brinicle.ItemSearchEngine("test_indexes/v4", dim=96, alpha=0.0)
engine.init("build")
for i in range(20):
    engine.ingest(
        external_id=f"item-{i}",
        title=f"Document number {i}",
        category="document",
        subcategory="txt",
        attributes={}
    )
engine.finalize()

# Step 2: start an upsert, then search concurrently mid-ingest
engine.init("upsert")
for i in range(20, 1020):
    engine.ingest(
        external_id=f"item-{i}",
        title=f"Document number {i}",
        category="document",
        subcategory="txt",
        attributes={}
    )

search_results = []
search_errors = []

def search_during_ingest():
    try:
        r = engine.search("document", k=5)
        search_results.extend(r)
    except Exception as e:
        search_errors.append(str(e))

t = threading.Thread(target=search_during_ingest)
t.start()
engine.finalize()
t.join()

print("results during ingest:", search_results)
print("errors during ingest:", search_errors)
```

Record whether search blocked until finalize, raised an exception, or returned results (potentially stale). All three outcomes are handled by the single-lock strategy. This test determines whether that lock can ever be relaxed in a future version.

**V5 - Index persistence across process restart**

```python
import brinicle

# Session 1: build and finalize
engine = brinicle.ItemSearchEngine("test_indexes/v5", dim=96, alpha=0.0)
engine.init("build")
engine.ingest(external_id="persist-test", title="Persistence check document", category="document", subcategory="md", attributes={})
engine.finalize()
print("has_index after build:", engine.has_index)
engine.close()
del engine

# Session 2: reload and search without re-ingesting
engine2 = brinicle.ItemSearchEngine("test_indexes/v5", dim=96, alpha=0.0)
print("has_index on reload:", engine2.has_index)
results = engine2.search("persistence check", k=5)
assert "persist-test" in results, "Index not persisted across process restart"
print("Persistence confirmed:", results)
engine2.close()
```

Expected: `has_index` is True in both sessions. The record ingested in Session 1 is immediately queryable in Session 2 without calling `init()` or `ingest()`.

**Fallback if `has_index` raises `AttributeError`:** replace the property check with a direct filesystem check. brinicle persists the index as files under `index_path/`. Check `Path(index_path).exists() and any(Path(index_path).iterdir())` as a substitute. Update the startup sequence in this document and in `indexer.py` to use this check.

**Fallback if `close()` raises `AttributeError`:** remove the `close()` call from the shutdown sequence. brinicle is disk-first - all data is already on disk after `finalize()`. Skipping `close()` means file handles are released by the OS on process exit rather than explicitly. No data loss.

**Fallback if persistence itself fails** (Session 2 search returns empty): the startup scan recovery mechanism already handles this. On every restart, files newer than `last_scan_time` are re-indexed. Set `last_scan_time = 0` to force a full re-index on every startup. This is expensive but correct. If V5 fails, this fallback must be adopted as the permanent architecture - document it in `docs/phase0-results.md` before Phase 1.

**V6 - Delete, rebuild detection, and graph optimization**

```python
import brinicle

engine = brinicle.ItemSearchEngine("test_indexes/v6", dim=96, alpha=0.0)
engine.init("build")
for i in range(10):
    engine.ingest(
        external_id=f"del-test-{i}",
        title=f"Delete test document {i}",
        category="document",
        subcategory="txt",
        attributes={}
    )
engine.finalize()

# Delete several records
engine.delete_items(["del-test-0", "del-test-1", "del-test-2", "del-test-3"])

# Confirm deleted records are gone
results = engine.search("delete test", k=10)
for i in range(4):
    assert f"del-test-{i}" not in results, f"del-test-{i} still in results after delete"
assert "del-test-4" in results, "del-test-4 should still be present"

# Check rebuild detection and optimization
print("needs_rebuild after 4 deletes of 10:", engine.needs_rebuild())
engine.optimize_graph()
print("optimize_graph completed without error")
engine.rebuild_compact()
print("rebuild_compact completed without error")
engine.close()

# Repeat for AutocompleteEngine
auto = brinicle.AutocompleteEngine("test_indexes/v6_auto", dim=48)
auto.init("build")
for i in range(10):
    auto.ingest(external_id=f"suggestion-{i}", text=f"planning document number {i}")
auto.finalize()

auto.delete_items(["suggestion-0", "suggestion-1"])
results_a = auto.search("planning", k=10)
assert "suggestion-0" not in results_a
assert "suggestion-2" in results_a
print("AutocompleteEngine delete confirmed")
print("needs_rebuild:", auto.needs_rebuild())
auto.optimize_graph()
auto.close()
```

This test gates Phase 3 (DELETE and MOVED event handling).

**Fallback if `delete_items()` raises `AttributeError`:** real-time deletion from the index is not possible. Replace the delete strategy with tombstoning: on DELETE/MOVED events, store the removed `external_id` in a `Set[str]` in `indexer.py` (the tombstone set). Filter tombstoned ids out of every `search()` result before returning them. On every daemon restart, rebuild the index from the current filesystem state (all files in `watched_folders`), which naturally excludes deleted files. The tombstone set is in-memory only and does not persist - a restart clears it and the rebuild makes it unnecessary.

**Fallback if `needs_rebuild()` raises `AttributeError`:** replace the check with an internal counter. `indexer.py` tracks `_delete_count: int`. After every batch that contains deletes, increment by the number of deleted records. Call `optimize_graph()` when `_delete_count` exceeds a threshold (e.g., 100). Reset the counter after each call.

**Fallback if `optimize_graph()` raises `AttributeError`:** skip the compaction step entirely. The HNSW graph quality degrades slightly with accumulated deletions but remains functional. Log a warning after every 1000 cumulative deletes recommending a daemon restart (which triggers a full rebuild via the startup scan).

**V7 - AutocompleteEngine ingest signature and search return format**

```python
import brinicle

auto = brinicle.AutocompleteEngine("test_indexes/v7", dim=48)
auto.init("build")

# Ingest with external_id=title, text=title
# search() must return title strings, not paths
auto.ingest(external_id="Q2 planning meeting notes", text="Q2 planning meeting notes")
auto.ingest(external_id="Quarterly report draft", text="Quarterly report draft")
auto.ingest(external_id="Meeting agenda template", text="Meeting agenda template")
auto.finalize()

results = auto.search("planning", k=5)
print("Autocomplete results for 'planning':", results)
# Expected: ["Q2 planning meeting notes"] or similar - readable title strings

results2 = auto.search("meet", k=5)
print("Autocomplete results for 'meet':", results2)
# Expected: titles containing "meet" ranked by match quality

# Confirm upsert mode works (not just build)
auto.init("upsert")
auto.ingest(external_id="Budget review notes", text="Budget review notes")
auto.finalize()

results3 = auto.search("budget", k=5)
assert "Budget review notes" in results3, "upsert mode must work on AutocompleteEngine"
auto.close()
```

This test confirms: (1) `ingest(external_id, text)` is the correct positional/keyword form, (2) `search()` returns the strings set as `external_id`, (3) upsert mode is supported.

**Fallback if keyword form raises `TypeError`** (pybind11 positional-only parameters): switch all `auto_engine.ingest(external_id=..., text=...)` calls to positional form: `auto_engine.ingest(item.title, item.title)`. Update the Autocomplete record section and background_indexer code in this document.

**Fallback if upsert mode raises an error:** replace incremental updates with full rebuilds on the AutocompleteEngine only. On every background_indexer batch, call `auto_engine.init("build")`, re-ingest all known titles from `indexer.py`'s internal title set, then `auto_engine.finalize()`. This is more expensive but AutocompleteEngine at `dim=48` is small. Document the title-tracking requirement in `indexer.py` before Phase 1.

---

## Architecture

```
Filesystem (user-selected folders)
        |
        | watchdog (CREATE / MODIFY / DELETE / MOVED)
        v
+----------------------------------------------------------+
|                     Python Daemon                        |
|                                                          |
|  Extractor                                               |
|  .txt .md  -> plain read                                 |
|  .pdf      -> pymupdf                                    |
|  .docx     -> python-docx                                |
|         |                                                |
|         v                                                |
|  asyncio.Queue  (ingest_queue)                           |
|  Items: IngestItem | MoveItem | DeleteItem               |
|         |                                                |
|         v                                                |
|  Background Indexer Task                                 |
|    drains queue in batches                               |
|    acquires asyncio.Lock                                 |
|    processes IngestItem, MoveItem, DeleteItem            |
|    item_engine.init("upsert")                            |
|    for each item: item_engine.ingest(...)                |
|    item_engine.finalize()                                |
|    auto_engine.init("upsert")                            |
|    for each item: auto_engine.ingest(title, title)       |
|    auto_engine.finalize()                                |
|    releases asyncio.Lock                                 |
|         |                                                |
|         v                                                |
|  brinicle.ItemSearchEngine   dim=96  alpha=0.0           |
|    path: ~/...localmemory/indexes/items                  |
|                                                          |
|  brinicle.AutocompleteEngine  dim=48                     |
|    path: ~/...localmemory/indexes/autocomplete           |
|                                                          |
|  config.json                                             |
|    watched_folders, port, last_scan_time, api_token      |
|                                                          |
|  FastAPI  localhost:7474                                 |
|    POST /v1/ingest      [requires Bearer token]          |
|    GET  /v1/setup       [localhost only, time-limited]   |
|    POST /v1/search      [localhost only]                 |
|    GET  /v1/autocomplete [localhost only]                |
|    GET  /               web UI                           |
|    /mcp                 MCP protocol [localhost only]    |
+----------------------------------------------------------+
        ^
        | HTTP POST with Bearer token
        |
Chrome Extension (TypeScript, Manifest V3)
@mozilla/readability extracts page text
Excluded domains checked against chrome.storage.local
Posts {url, title, content, domain, timestamp}
```

All persisted data lives in one directory:

```
~/Library/Application Support/localmemory/
  config.json
  indexes/
    items/
    autocomplete/
```

No database. No secondary metadata store.

---

## Data model

### External ID design

`search()` returns `external_id` strings only. The id must be self-describing so results are meaningful without any lookup. The full absolute path or full URL is the external_id.

- File: `external_id = "/Users/mohsen/Documents/notes/q2-planning.md"`
- Webpage: `external_id = "https://example.com/article/vector-search"`

Upsert deduplication is native: same path or URL on second ingest replaces the old record automatically. No manual comparison, no manual delete before re-index.

Fallback if V2 reveals a length limit: `sha256(path)` as external_id, plus a `path_index.json` file in the localmemory directory mapping hashes back to original paths. This file is only written if the fallback is needed.

### Document record

```python
item_engine.ingest(
    external_id = "/absolute/path/to/document.pdf",
    title       = "extracted title or filename stem",
    category    = "document",
    subcategory = "pdf",
    attributes  = {
        "indexed_at": "2026-05-27T10:00:00Z",
        "source":     "file_watcher",
    }
)
```

### Webpage record

```python
item_engine.ingest(
    external_id = "https://example.com/article",
    title       = "page title from readability extraction",
    category    = "webpage",
    subcategory = "example.com",
    attributes  = {
        "indexed_at": "2026-05-27T10:00:00Z",
        "source":     "chrome_extension",
    }
)
```

### Autocomplete record

Every ItemSearchEngine ingest is mirrored to AutocompleteEngine in the same batch. The `external_id` for autocomplete is set to the title string, not the file path. `search()` then returns readable title strings as suggestions, not paths.

```python
auto_engine.ingest(
    external_id = item.title,   # search() returns this - must be human-readable
    text        = item.title    # what gets vectorized for matching
)
```

If two documents share the same title, they produce the same `external_id`. In upsert mode, the second write replaces the first. For autocomplete this is acceptable - only one suggestion per distinct title is needed.

### Search results

`search()` returns `["/path/to/doc.pdf", "https://example.com/page", ...]`.

`search_with_distance()` returns `[("/path/to/doc.pdf", 0.12), ("https://...", 0.19), ...]`.

Distance is the HNSW approximate distance (lower = more similar). The web UI derives the display label from the external_id: the filename stem for file paths, the domain plus URL path for webpages.

`autocomplete search()` returns `["Q2 planning meeting notes", "Meeting agenda template", ...]` - title strings because `external_id=title` at ingest time.

### File event handling

| Event | Action |
|---|---|
| CREATE | Push `IngestItem(path)` to queue |
| MODIFY | Push `IngestItem(path)` to queue - native upsert replaces old record |
| DELETE | Push `DeleteItem(path)` to queue |
| MOVED | Push `MoveItem(src_path, dest_path)` to queue - handled atomically under one lock acquisition |

`MoveItem` is processed as: delete `src_path` from both engines, then ingest `dest_path` to both engines, within a single `async with lock` block. There is no window between delete and ingest where neither path exists - both operations are under one lock.

---

## Concurrency model

### The single lock rule

All calls to both brinicle engines go through a single `asyncio.Lock`: `init()`, `ingest()`, `finalize()`, `search()`, `delete_items()`, `optimize_graph()`. No brinicle method is called outside this lock, from any module.

### Background indexer task

The watcher and the HTTP ingest endpoint push items to `asyncio.Queue`. The background task drains the queue:

```python
async def background_indexer(queue, item_engine, auto_engine, lock, notifier):
    while True:
        first = await queue.get()
        batch = [first]
        while True:
            try:
                batch.append(queue.get_nowait())
            except asyncio.QueueEmpty:
                break

        notifier.set_indexing(len(batch))

        async with lock:
            ingests = [i for i in batch if isinstance(i, IngestItem)]
            deletes = [i for i in batch if isinstance(i, DeleteItem)]
            moves   = [i for i in batch if isinstance(i, MoveItem)]

            if deletes or moves:
                del_ids = [i.path for i in deletes] + [i.src_path for i in moves]
                item_engine.delete_items(del_ids)
                # autocomplete is indexed with external_id=title, not path
                # cannot delete by path - stale title suggestions remain until restart
                if item_engine.needs_rebuild():
                    item_engine.optimize_graph()

            all_ingests = ingests + [IngestItem.from_move(m) for m in moves]
            if all_ingests:
                item_engine.init("upsert")
                for item in all_ingests:
                    item_engine.ingest(**item.fields)
                item_engine.finalize()

                auto_engine.init("upsert")
                for item in all_ingests:
                    auto_engine.ingest(external_id=item.title, text=item.title)
                auto_engine.finalize()

        notifier.set_ready()
        for _ in batch:
            queue.task_done()
```

Autocomplete entries for deleted files remain as stale suggestions until the title appears in no future ingest. They cause no search result contamination - only ItemSearchEngine drives main results. A full autocomplete rebuild on startup is the v2 solution. This is recorded in Known Limitations.

### Search under lock

```python
async with lock:
    results = item_engine.search(query, k=limit)
```

A search waits if a batch is being processed. For typical batch sizes of 1-5 items, the wait is sub-millisecond.

### Known limitation: asyncio event loop blocking

brinicle's C++ calls are synchronous. When `ingest()` or `finalize()` runs inside `async with lock`, the asyncio event loop is frozen for the duration. HTTP requests queue at the OS level and are processed after the lock releases.

For normal operation (small batches, file changes one at a time) this is invisible. During the startup scan when a large folder is indexed for the first time, the event loop may freeze for several seconds. This is accepted for v1. The migration path is `asyncio.get_event_loop().run_in_executor()` with a `threading.Lock`.

### Error handling

If `finalize()` raises an exception: log the error with the batch contents, release the lock, discard the batch, and continue. The items remain queryable in their pre-batch state. On next startup, the startup scan re-indexes any files whose mtime is newer than `last_scan_time`, recovering lost ingests automatically. No crash, no data loss beyond the failed batch.

### Daemon shutdown

When the daemon receives SIGTERM or SIGINT:

1. Signal `background_indexer` to stop accepting new items
2. Drain the queue (allow the current batch to complete)
3. Call `item_engine.close()` and `auto_engine.close()` to release file handles
4. Write `last_scan_time` to `config.json`
5. Exit

`close()` is non-destructive. The index files remain on disk. Next startup reads them via `has_index`.

---

## Authentication

### Token generation

On first daemon launch, a 32-byte cryptographically random token is generated (`secrets.token_hex(32)`), stored in `config.json` as `api_token`, and never shown to the user. On subsequent launches the existing token is loaded.

### Setup endpoint

`GET /v1/setup` returns the token as JSON, but only if:

1. The request comes from `127.0.0.1`
2. The request arrives within 120 seconds of daemon startup

After that window, `/v1/setup` returns 403. This gives the Chrome extension time to retrieve the token on install without exposing it permanently.

The Chrome extension calls `/v1/setup` once in `background.ts` immediately after install. The token is stored in `chrome.storage.local`. All subsequent requests include `Authorization: Bearer <token>`.

### Protected endpoints

| Endpoint | Auth required |
|---|---|
| `POST /v1/ingest` | Bearer token |
| `GET /v1/setup` | None (localhost + time window) |
| `POST /v1/search` | Localhost IP check only |
| `GET /v1/autocomplete` | Localhost IP check only |
| `GET /` | None |
| `/mcp` | Localhost IP check only |

Write endpoints require the token to prevent any process or browser tab from injecting data into the index. Read endpoints require only a localhost source check - external access is blocked at the network layer, not by token.

---

## Indexing state notifications

The `notifier` object passed to `background_indexer` updates the rumps menu bar title in real time.

| State | Menu bar title |
|---|---|
| Idle | `Membrinicle - Ready` |
| Indexing batch | `Membrinicle - Indexing...` |
| Startup scan in progress | `Membrinicle - Scanning folders...` |
| Index rebuild required | `Membrinicle - Rebuild needed` |

The Ready state does not show a document count. brinicle does not expose a document count through its API. `indexer.py` maintains an internal counter (`_doc_count: int`) that increments on new ingests and decrements on deletes. A new ingest for an already-tracked `external_id` is an update, not an addition - `indexer.py` tracks known ids in a `set[str]` to distinguish them. This counter is approximate (lost on crash, rebuilt on next startup scan). It is not shown in v1. If document count display is added later, it comes from this internal counter, not from brinicle.

**Rebuild notification:** If a future version changes `dim` or the index is corrupted, the daemon detects this on startup (wrong dim in existing index or `has_index` is False after expected warm reload) and sets the menu bar to `Membrinicle - Rebuild needed`. The user clicks the menu bar item and sees: "Your search index needs to be rebuilt. This will re-index all your files and may take a few minutes. Proceed?" On confirmation, `indexes/` is deleted and the startup scan runs in full.

**New folder notification:** When a new watched folder is added via the first-launch wizard or settings, the menu bar immediately shows `Membrinicle - Scanning folders...` while the startup scan processes that folder.

---

## Component specifications

### Daemon modules

| Module | Responsibility |
|---|---|
| `server.py` | FastAPI app, all route definitions, startup sequence orchestration, SIGTERM/SIGINT shutdown handler; uvicorn must be launched with `workers=1` - brinicle index files have no inter-process locking and two worker processes will corrupt the index silently |
| `watcher.py` | watchdog observer; handles CREATE, MODIFY, DELETE, MOVED; pushes IngestItem, DeleteItem, MoveItem to queue |
| `extractor.py` | accepts a file path, returns plain text; dispatches by extension (.pdf, .docx, .md, .txt) |
| `indexer.py` | owns both brinicle engines (with `alpha=0.0` and correct dims), the asyncio.Lock, the ingest_queue, the notifier reference, and the internal `_doc_count` counter; exposes async `put(item)`, async `search(query, k)`, async `close()` |
| `notifier.py` | owns the rumps menu bar app; provides `set_indexing()`, `set_ready()`, `set_rebuild_needed()`; runs rumps event loop in a separate thread |
| `mcp.py` | MCP protocol handler on /mcp; tools: `search(query, limit)` and `get_context(query)` |
| `config.py` | reads/writes config.json; fields: `watched_folders`, `port`, `last_scan_time`, `api_token` |

`metadata.py` does not exist. There is no database.

### Chrome extension modules

| File | Responsibility |
|---|---|
| `content.ts` | injected on every page; checks `chrome.storage.local` for excluded domains before proceeding; uses @mozilla/readability to extract clean text; posts to `/v1/ingest` with Bearer token |
| `background.ts` | service worker; calls `/v1/setup` once on install to retrieve and store the token; imports last 30 days of history via `chrome.history.search({text: "", startTime: Date.now() - 30*24*60*60*1000, maxResults: 10000})`; manages retry queue when daemon is unreachable |
| `popup.ts` | shows connected/disconnected status; reads and writes `chrome.storage.local` exclusion list; adding a domain to the exclusion list is stored locally in the extension, not sent to the daemon |
| `chrome_api.ts` | typed wrappers for all `chrome.*` API calls |

### Startup sequence

On every daemon launch, `server.py` runs these steps in order:

1. Load or create `config.json`
2. Create `~/Library/Application Support/localmemory/indexes/` if it does not exist
3. Initialise `ItemSearchEngine(dim=96, alpha=0.0)` and `AutocompleteEngine(dim=48)` - brinicle reads the existing index from disk if `has_index` is True (warm reload); if `has_index` is False, a cold start is recorded and a full scan is required regardless of `last_scan_time`
4. Start `background_indexer` asyncio task
5. Start `notifier` (rumps menu bar) in a daemon thread
6. Start `watcher.py` observers for all paths in `watched_folders`
7. Start FastAPI server via uvicorn with `workers=1` hard-coded - never parameterise this value. brinicle's Dockerfile carries an explicit `# workers must be one` comment. Two processes sharing the same index files produce silent corruption.
8. Run startup scan: query file system for all files in `watched_folders` with `mtime > last_scan_time` (or all files if cold start); push each as an `IngestItem` to the queue; update `last_scan_time` to now

If `watched_folders` is empty (first run before wizard), steps 6 and 8 do nothing. The daemon is still functional for web page ingests from the Chrome extension.

---

## Technology stack

| Layer | Choice | Reason |
|---|---|---|
| Search engine | brinicle ItemSearchEngine dim=96 alpha=0.0 | Disk-first HNSW, proven benchmarks, Apache 2.0, no external model; alpha=0.0 required for lexical-only mode |
| Autocomplete | brinicle AutocompleteEngine dim=48 | Ships with brinicle, same lifecycle; dim=48 is the library default for this engine |
| Language | Python 3.11 | Required by brinicle bindings; minimum supported version is 3.10 |
| HTTP server | FastAPI + uvicorn | Declared optional deps in brinicle's own pyproject.toml |
| File watching | watchdog | Handles CREATE, MODIFY, DELETE, MOVED on all platforms |
| PDF extraction | pymupdf | Fast, pure Python, no system dependency |
| DOCX extraction | python-docx | Standard, well-maintained |
| Settings | config.json via stdlib json | Single file, no database |
| Menu bar | rumps | Python-native macOS status bar; runs in a daemon thread |
| Packaging | PyInstaller | Single .app bundle, no Python required on target machine |
| Chrome extension | TypeScript, Manifest V3 | Required for Chrome extensions |
| Content extraction | @mozilla/readability | Same library as Firefox Reader View |
| Token generation | stdlib secrets | Cryptographically secure, no dependency |
| License | PolyForm Noncommercial 1.0.0 | Restricts commercial use; compatible with brinicle Apache 2.0 |

SQLite is not in this stack.

---

## Coding standards

### Python

- Python 3.11, typed throughout with `from __future__ import annotations`
- `TypedDict` for all structured data shapes; no `Any`
- All brinicle engine access through `indexer.py` only - no other module imports brinicle; enforced in CI by: `grep -r "import brinicle" src/daemon/ | grep -v "indexer.py" && exit 1 || exit 0`
- All brinicle calls inside `async with lock` - no exceptions to this rule
- No raw JSON reads/writes outside `config.py`
- `ruff` for linting, `black` for formatting, both enforced in CI
- Tests use real brinicle engines against a temp directory - brinicle is never mocked

### TypeScript

- `strict: true` in tsconfig
- No `any` - `unknown` with type guards at all message boundaries
- ESLint with `@typescript-eslint/recommended`
- All `chrome.*` API calls wrapped in typed helpers in `chrome_api.ts`

### Repository layout

```
/
  LICENSE
  COMMERCIAL.md
  README.md
  CLAUDE.md
  pyproject.toml
  src/
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
  data/
  tests/
    test_phase0.py
    test_indexer.py
    test_extractor.py
    test_server.py
  docs/
    adr/
    2026-05-27-system-design.md
```

---

## Implementation phases

### Phase 0 - Verify brinicle behavior

Write and run `tests/test_phase0.py` containing V0 through V7 above. Record all outputs in `docs/phase0-results.md`. Update this design document where any result differs from the expected behavior. For each test that fails, implement its documented fallback before moving to the next test.

This phase produces no product code. It produces eight confirmed facts. No Phase 1 code is written until `docs/phase0-results.md` exists and all eight tests pass or their fallbacks are documented.

### Phase 1 - Search core

Build `indexer.py`: ItemSearchEngine at `dim=96, alpha=0.0` and AutocompleteEngine at `dim=48`, asyncio.Lock, ingest_queue, background_indexer task, notifier interface. Write tests confirming lexical retrieval.

```python
await idx.put(IngestItem(
    external_id="/notes/q2-planning.md",
    title="Q2 planning meeting notes",
    category="document",
    subcategory="md",
    attributes={"indexed_at": "2026-05-27T10:00:00Z", "source": "file_watcher"}
))
await idx.queue.join()

results = await idx.search("planning meeting notes", k=5)
assert "/notes/q2-planning.md" in results
```

The query shares tokens with the title. This confirms brinicle's LexicalEncoder encodes and retrieves correctly with `alpha=0.0`. Semantic cross-vocabulary retrieval is a hybrid mode feature - not tested here.

Also confirm upsert: ingest the same external_id twice with different titles. Search must return the updated title's record only - no duplicate.

### Phase 2 - Text extraction

Build `extractor.py`. Test all four supported formats. Confirm no HTML remnants, no page number artifacts, no empty output from valid files.

```python
text = extract("/docs/report.pdf")
assert len(text) > 100
assert "<" not in text
assert "quarterly revenue" in text
```

### Phase 3 - File watcher and config

Build `watcher.py` and `config.py`. Test all four watchdog event types:

- **CREATE:** file in watched folder appears in search within 3 seconds
- **MODIFY:** updated content replaces old content; no duplicate record
- **DELETE:** file removed from search; `optimize_graph()` called if `needs_rebuild()` is true
- **MOVED:** old path removed, new path indexed, no window where neither path is searchable

Test startup scan: set `last_scan_time` to epoch zero. Restart daemon. Confirm all files in watched folder are indexed. Confirm `last_scan_time` in config.json is updated after scan. Confirm `has_index` is True after scan completes.

### Phase 4 - HTTP server and Chrome extension

Build `server.py`. Implement `/v1/ingest`, `/v1/search`, `/v1/setup`. Build the Chrome extension with token retrieval on install and Bearer token on all ingest requests.

Test token flow:
1. Start daemon
2. Wait for setup window to open
3. Confirm `GET /v1/setup` returns token
4. Wait 121 seconds
5. Confirm `GET /v1/setup` returns 403

Test ingest flow: send a page from Chrome extension, search via curl, confirm it appears.

### Phase 5 - Autocomplete

AutocompleteEngine is already fed by the background_indexer from Phase 1 (with `external_id=title, text=title`). Build `GET /v1/autocomplete?q=` in `server.py`. Confirm suggestions from previously indexed titles appear and are readable strings, not paths.

### Phase 6 - MCP server

Build `mcp.py`. Add to Claude Desktop config:

```json
{
  "mcpServers": {
    "membrinicle": { "url": "http://localhost:7474/mcp" }
  }
}
```

Ask Claude a question whose answer exists in an indexed file. Confirm Claude calls `search` and cites the document in its response.

### Phase 7 - Web UI

Build `web/index.html` and `web/app.js`. Embed via `importlib.resources`. Serve at `GET /`. Features:

- Search input with autocomplete from `/v1/autocomplete`
- Result cards: display label derived from external_id, category badge, relevance score from `search_with_distance()`
- Settings tab: watched folder list with add and remove controls. Exclusion list is managed via the extension popup, not the web UI.

### Phase 8 - macOS app and installer

Build `notifier.py` with the rumps status bar. Run the startup wizard on first launch (no default folders - user selects). Register LaunchAgent. Build with PyInstaller. Sign and notarize.

Test on a fresh macOS user account: no Python, no brinicle. Double-click .dmg. Complete wizard. Add one test file to the selected folder. Search at localhost:7474. Confirm result appears within 30 seconds.

---

## Deliverables per phase

| Phase | Confirmed state |
|---|---|
| 0 | Eight brinicle behaviors confirmed in writing; fallbacks documented for any failures |
| 1 | Search core working; upsert deduplication verified; queue and lock confirmed |
| 2 | Text extraction correct for all formats |
| 3 | All four file events handled; startup scan working; MOVED is atomic |
| 4 | Chrome extension sends pages with auth; HTTP API live |
| 5 | Autocomplete working; returns title strings not paths |
| 6 | Claude Desktop queries local index mid-conversation |
| 7 | Any user can search without the terminal; folder settings manageable in UI |
| 8 | Signed installable macOS app, zero dependencies required |

---

## Known limitations (v1)

- **Asyncio blocking:** brinicle C++ calls block the event loop. Startup scans on large folders may cause multi-second unresponsiveness. Mitigation: `run_in_executor` in a future version.
- **Autocomplete stale entries:** deleted files leave stale title suggestions in the autocomplete index until the daemon restarts. They do not appear as main search results. Fix: rebuild autocomplete from ItemSearchEngine results on every startup (v2).
- **Chrome history gap during downtime:** pages visited while the daemon is off are not indexed. The extension captures pages only in real time. A catch-up import on daemon restart is a v2 feature.
- **No mobile support:** the daemon and extension are macOS and Chrome only. Other platforms are not in scope for v1.
- **No title in main search results:** `search()` returns external_ids only. Display labels are derived from the path or URL, not from the stored title. The stored title improves search quality via LexicalEncoder but is not surfaced in results.
- **dim change requires full rebuild:** if hybrid mode is added and a larger dim is chosen for ItemSearchEngine, all users must rebuild their index. The rebuild wizard is included in the notification system. AutocompleteEngine dim is independent and can change without affecting the main index.

---

## Competitive position

| Feature | Membrinicle | TraceMind | LLMnesia | ContextBolt |
|---|---|---|---|---|
| Browsing history | Yes | Yes | No | No |
| Local files | Yes | No | No | No |
| AI conversations (v2) | Planned | No | Yes | No |
| MCP endpoint | Yes | No | No | Yes (bookmarks only) |
| REST API | Yes | No | No | No |
| Disk-based HNSW (brinicle) | Yes | No - WASM k-d tree | No | No |
| No external database | Yes | N/A | N/A | N/A |
| Noncommercial open source | Yes | No | No | No |
