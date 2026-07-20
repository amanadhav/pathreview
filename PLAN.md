# PLAN — Issue #159: structlog output not captured by pytest `caplog`

## Problem

The application logs through **structlog**, but structlog is not configured to feed
its events into Python's standard library `logging` system while tests run. pytest's
`caplog` fixture only sees records that pass through stdlib `logging`, so any test that
asserts on `caplog` fails — even though the code under test genuinely emits the expected
log event.

The canonical repro is:

```
pytest tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty -q
```

`processor.process([])` does emit the warning `"Empty chunks list provided to
BatchEmbeddingProcessor"` (it appears under pytest's "Captured stdout call"), yet the
assertion fails because `caplog.text == ''` and `caplog.records == []`. This breaks
caplog-based assertions suite-wide, not just in this one test.

## Root Cause

- `core/logging.py` defines `configure_logging()`, and that function *is* caplog-friendly
  in principle: it calls `structlog.configure(..., logger_factory=structlog.stdlib.LoggerFactory())`
  and `logging.basicConfig(...)`, which would route structlog events through stdlib logging.
- **However, `configure_logging()` is never invoked in the test path.** Its only caller is
  `scripts/seed_db.py`; it is not called from `api/main.py` and nothing in the pytest
  session triggers it. So during tests the app's logging configuration never applies.
- The module under test, `ingestion/embeddings/batch_processor.py`, binds a logger at
  **import time** with bare structlog: `logger = structlog.get_logger()`. There is no
  import of `core.logging` and therefore no configuration side effect.
- With nothing configuring structlog, it falls back to its **default, unconfigured**
  setup, which uses `PrintLoggerFactory`. That writes rendered log lines straight to
  `sys.stdout` via `print` and **bypasses stdlib `logging` entirely**.
- `caplog` works by attaching a handler to the stdlib logging system and collecting
  `LogRecord`s. Because structlog's default never produces stdlib `LogRecord`s, `caplog`
  captures nothing — hence empty `caplog.text`/`caplog.records` while the message is still
  visible on the console stream.
- `tests/conftest.py` currently contains only two string fixtures and has **no logging
  setup**: no `structlog.configure(...)`, no fixture routing structlog through stdlib
  logging, and no autouse hook to apply config before the import-time logger is first used.

## Proposed Fix

TBD — do not implement yet. Two open design questions to resolve first:

1. **Reuse `configure_logging()` vs. a dedicated test config in `conftest.py`.**
   The issue explicitly asks to "configure structlog in `tests/conftest.py`", which points
   toward a test-specific configuration. Reusing the app's `configure_logging()` from a
   fixture is another option. Decide which gives the most faithful, least surprising
   behavior for caplog without coupling tests to production/env-dependent rendering.

2. **`cache_logger_on_first_use` behavior given import-time logger binding.**
   `batch_processor.py` (and likely other modules) bind `logger = structlog.get_logger()`
   at import time. Configuration must be applied before the first log call, and we need to
   decide whether `cache_logger_on_first_use` should be `False` in tests to avoid a stale
   cached logger from the default configuration.

## Testing Plan

TBD.

## Infrastructure Note (separate from #159, left unfixed)

While bringing the stack up per `docs/SETUP.md`, the `vector-db` container
(`chromadb/chroma:0.4.22`) crashed on startup and exited (1):

```
AttributeError: `np.float_` was removed in the NumPy 2.0 release. Use `np.float64` instead.
```

Root cause (verified from the container's own startup logs): the image ships numpy
1.26.3, but its boot-time step "Rebuilding hnsw to ensure architecture compatibility"
runs an **unpinned** `pip install chroma-hnswlib numpy`, which upgrades numpy to 2.2.6
inside the container. chroma 0.4.22's `api/types.py` then executes `np.float_` (removed in
numpy 2.0) and the import fails. This is *not* the host venv's numpy pin — pinning numpy
in `pyproject.toml` would not help because the container reinstalls numpy from PyPI at
startup. The likely real fix is bumping the chroma server image (and matching the
`chromadb` client pin) to a 0.5.x line that no longer uses the removed numpy API.

This is unrelated to #159 and is **left unfixed**. Issue #159 is verified to need only
unit tests, which run fully against mocks (`tests/unit/test_batch_processor.py` uses
`Mock()` for both the embedding provider and vector db, and `tests/conftest.py` has no
service-dependent fixtures) — so no live vector-db is required to reproduce or fix it.
