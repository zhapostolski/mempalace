# Local deviations from upstream mempalace

**Read this before any upstream merge/rebase.** These are intentional and load-bearing. Losing one
of them has already caused a production outage.

Base: **v3.6.0** (rebased 2026-08-04). Safety net from that upgrade:
tag `pre-v3.6-upgrade-20260804`, branch `pre-v3.6-backup`.

---

## 1. stderr suppression when `MEMPALACE_LOG_FILE` is set — **CRITICAL**

**File:** `mempalace/mcp_server.py`, `_init_logging()`

**Deviation:** in the standalone path we attach **only** the file handler when
`MEMPALACE_LOG_FILE` is set. Upstream unconditionally appends a `StreamHandler(sys.stderr)`.

**Why:** `mcp-proxy` / `mcp-go` does **not drain the subprocess stderr pipe**. Linux pipe buffers are
64KB; once full, the next write blocks forever and **mempalace deadlocks**. The gateway keeps the
route registered, so `aios status` still reports `ok` while every call returns `broken pipe`.

**This is the probable root cause of the 2026-08-04 outage**, where mempalace was dead for hours and
the memory fan-out failed silently.

**On merge:** keep upstream's `_logging_configured` idempotency guard and the `#1860`
host-owned-root-logger handling. Only the standalone `else:` branch deviates. If upstream ever fixes
the drain behaviour (or mcp-go does), this can be dropped.

**Guarded by:** `tests/test_mcp_server.py::TestColdStartDiagnostics::test_eager_warmup_query_failure_logs_and_persists_to_log_file`
— adapted to assert the diagnostic lands in the **file** and that stderr stays **quiet**. If a future
merge reverts that test to assert stderr, the deviation has been lost.

---

## 2. No ChromaDB pre-warm in the session-start hook

**File:** `hooks/mempal_session_start_hook.sh`

**Deviation:** the ChromaDB pre-warm block is removed.

**Why:** this palace runs the **pgvector** backend. Instantiating `ChromaBackend` recreated
`chroma.sqlite3` and migration flag-files inside the palace directory, tripping a
"multiple backend artifacts" mismatch. There is no chroma to warm.

---

## 3. Opt-in recency-aware ranking

**Files:** `mempalace/searcher.py`, `tests/test_searcher_recency.py`

**Deviation:** `_hybrid_rank()` takes extra `recency_weight` and `half_life_days` parameters and
blends an exponential-decay recency factor over `filed_at`.

**On merge:** this is **additive** to upstream's `metric` parameter — the function body references
`metric`, `recency_weight` **and** `half_life_days`. Keeping only one side of a conflict here will
break the function. Merge both.

Defaults come from `MEMPALACE_RECENCY_WEIGHT` / `MEMPALACE_RECENCY_HALF_LIFE_DAYS`;
`recency_weight=0` reproduces upstream behaviour exactly.

---

## 4. Defensive `None` metadata in the `authored_at` tie-break

**File:** `mempalace/searcher.py`

**This is an upstream bug fix, not a preference** — offer it upstream.

Upstream `#1890` used `pair[1].get("metadata", {}).get("authored_at")`. A `dict.get` default only
applies when the key is **absent**; when the key exists holding `None`, it returns `None` and the
chained `.get` raises `AttributeError`. Changed to `(pair[1].get("metadata") or {})`, matching the
defensive form already used elsewhere in the same module.

---

## Upstream hooks were deliberately taken over local ones (2026-08-04)

`hooks/mempal_save_hook.sh` and `hooks/mempal_precompact_hook.sh` conflicted during the v3.6.0
rebase. Upstream's security-hardened implementations (sanitizer, `#1231` review) were taken over the
local one-line delegations, because hooks are **disabled on this install** (`auto_save=False`) and
upstream's version is strictly better if they are ever enabled.
