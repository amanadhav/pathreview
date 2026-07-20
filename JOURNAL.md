# Module 3 Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/159

**Issue title:** structlog output is not captured by pytest caplog — log assertions fail suite-wide

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The backend does its logging through `structlog`, but during test runs structlog
is never wired into Python's standard-library `logging` system. pytest's `caplog`
fixture only sees records that flow through stdlib `logging`, so any test that
asserts on `caplog` comes up empty and fails — even when the code under test
genuinely emits the expected log line (the message shows up in captured stdout,
but `caplog.text` and `caplog.records` stay empty). The canonical failure is
`test_empty_chunks_list_returns_empty` in `tests/unit/test_batch_processor.py`,
but the gap affects caplog-based assertions across the whole suite. The root cause
is that `core/logging.py::configure_logging()` (which would route structlog through
stdlib logging) is never invoked in the test path, so structlog falls back to its
default `PrintLoggerFactory` that prints straight to stdout and bypasses `logging`.
A successful fix configures structlog in `tests/conftest.py` so its events
propagate into stdlib logging, making `caplog` capture them and the existing
assertions pass — without coupling the tests to production/env-dependent rendering.
This affects the test harness (`tests/conftest.py`) and the logging setup in
`core/logging.py` / `ingestion/embeddings/batch_processor.py`.

**Branch name:** test/159-structlog-caplog-propagation

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

---

### "Is this right for me?" — scope reasoning

- **Tier fit:** Tier 1, labeled `good for first-time contributors`. Appropriate for a
  first contribution to a large multi-module codebase.
- **Scope is well-bounded:** The fix is concentrated in test configuration
  (`tests/conftest.py`) plus understanding the existing `core/logging.py` setup. It
  does not require touching application/business logic or the RAG/agent pipelines.
- **Reproducible:** There is a single, deterministic repro command
  (`pytest tests/unit/test_batch_processor.py::TestBatchEmbeddingProcessor::test_empty_chunks_list_returns_empty -q`)
  and a clear pass/fail signal.
- **No external service dependency:** The failing unit test runs entirely against
  mocks (mocked embedding provider and vector db), so a live vector-db / Docker
  stack is not required to reproduce or verify the fix. (Note: the `chromadb/chroma:0.4.22`
  container crashes on startup due to a NumPy 2.0 incompatibility, but that is
  unrelated to #159 and out of scope here.)
- **Clear definition of done:** caplog-based assertions capture structlog events and
  the previously failing test passes, with no regressions elsewhere.
