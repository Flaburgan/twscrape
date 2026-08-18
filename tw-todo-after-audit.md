# twscrape fork — audit TODO

Context for the executing model:

- This repo is a fork of `vladkens/twscrape` (remote `upstream`). Local `main` sits on
  upstream `v0.20.0` (`9745b02`) with **zero local diff**: everything the fork carried was
  merged upstream. Keep diffs small where possible to ease future rebases; tasks marked
  **BREAKING** intentionally diverge (this fork's only consumer is `alexandros` in the parent
  repo `open-web`).
- Completed tasks have been removed from this file; the original numbering is kept, so task
  numbers are not contiguous. Gone: Task 1 (`GqlFeaturesOutdatedError` instead of `exit(1)`,
  merged as #324), Task 5.1 (NetworkError retry cap, merged reworked as #325 — see Task 5),
  Task 13 (relogin skips cookie accounts; `add_cookie` is a validating upsert), Task 15
  (cookie values no longer leak into exceptions), and the username-parameterization part of
  Task 2.
- Relevant upstream v0.20.0 state for the remaining tasks: retry counters live on the `Ctx`
  object keyed by a `FailKind` enum (`TRANSPORT`/`LOADSHED`/`UNKNOWN`; 3 per kind, 4 total per
  account), and `ConnectError` is folded into the `TRANSPORT` path — never re-raised (60-s
  cooldown + rotation instead). XClId failures are typed (`XClIdAccountError` → 15-min lock +
  rotate, `XClIdParseError` → abort) and retried only in `Ctx.req`.
- Tooling: `uv` (not Docker). Verify with `make check` (ruff + ty) and `make test` (pytest) from
  the repo root. Python ≥ 3.10.
- Consumer integration to keep in mind: `../alexandros/src/scraper.py` uses
  `API(pool, raise_when_no_account=True, wait_timeout=...)`, catches `NoAccountError`, regex-parses
  its message to recover the queue name, and re-scans `api.pool.get_all()` to compute a retry
  time. Task 3 is designed to remove the need for that. Since 2026-08-11 it also probes
  `https://x.com/robots.txt` on `NoAccountError` (`_platform_reachable`) to tell a network
  outage apart from rate limits — needed because v0.20.0 no longer re-raises `ConnectError`.
- Do the tasks in order inside each priority band; they are scoped to not rework each other.

---

## P0 — Correctness bugs

### Task 2: Parameterize SQL in `AccountsPool` (injection + double-quote identifier bug) — remaining parts

**File:** `twscrape/accounts_pool.py`

Username `IN` lists were parameterized upstream in v0.20.0 (`_usernames_where`). Still built
with f-strings:

- `_get_and_lock` interpolates `'{condition}'` (single quotes) when given a username — breaks/injects
  on `'` (~line 283).
- `lock_until`, `unlock`, `get_for_queue`, `next_available_at`, `stats` interpolate the `queue`
  name into JSON paths and column aliases. Queue names are internal (GQL op names) so risk is low,
  but nothing enforces that.

Instructions:
1. `_get_and_lock`: replace the "space means subquery" heuristic (fragile) with two explicit code
   paths or an explicit flag. Bind the username with `:username` in the non-subquery path.
2. Queue names — two options:
   - **Option A — validate at the boundary**: add
     `def _check_queue(queue: str)` raising `ValueError` unless
     `re.fullmatch(r"[A-Za-z0-9_]+", queue)`, call it in every method that receives `queue`; keep
     f-string interpolation for JSON paths. Pros: tiny diff, JSON-path SQL stays readable, easy
     upstream rebase. Recommended.
   - **Option B — bind the path**: use `json_set(locks, '$."' || :queue || '"', ...)` etc.
     Pros: no validation needed. Cons: harder to read, every query rewritten, and the
     `stats()` column aliases still can't be bound (needs Option A treatment anyway).
3. Unify the JSON-path quoting style: `get_for_queue`/`lock_until`/`unlock`/`stats` use `'$.{q}'`
   while `next_available_at` uses `'$."{q}"'`. Pick the quoted form `'$."{q}"'` everywhere
   (works for all key names).
4. Tests (`tests/test_pool.py`): add a case with a username containing a quote, and one literally
   named `email` (plus a second account whose email equals it) — the username must bind, not
   resolve as an identifier.

### Task 3: **BREAKING** — machine-readable availability (`next_available_at` + `NoAccountError`) and race fix

**Files:** `twscrape/accounts_pool.py`, `twscrape/__init__.py`, `tests/test_pool.py`

This is the continuation of the bounded-wait feature (merged upstream as `c0e81bf`, #317) and
the reason for the parent-repo branch `get-next-available-account-time-from-x`. Today:

- `next_available_at()` returns a *human string* — `"now"` or `"%H:%M:%S"` in **local** time
  (mixes UTC and local, unusable programmatically). Alexandros re-implements the whole scan
  (`_pool_retry_at`) and regex-parses the queue name out of the `NoAccountError` message.
- Race bug in `get_for_queue_or_wait`: `no_active = not nat` conflates "no active accounts"
  with "active accounts exist but none currently holds a lock for this queue" — which is exactly
  what happens when an account frees up between the failed `get_for_queue` call and the
  `next_available_at` query. In that window the wait budget is ignored and it raises/returns
  immediately.
- The poll sleeps a fixed `wait_interval` even when the remaining budget or the known unlock time
  is smaller.
- ~~The `wait_timeout`/`wait_interval` feature has zero tests.~~ Resolved: the merged #317 added
  tests (`tests/test_pool.py::test_get_for_queue_or_wait_*`) covering wait-then-acquire,
  `wait_timeout=0`, no-active-accounts, and config plumbing. Instruction 6 below now only adds
  the cases not covered there (structured `NoAccountError` fields, tz-aware `next_available_at`).

Instructions:
1. Change `next_available_at(queue)` to return `datetime | None` (timezone-aware UTC):
   the earliest lock expiry among **active** accounts for that queue; return the expiry as-is even
   if already past; `None` when no active account holds a lock for the queue. Format for humans
   only at the log site in `get_for_queue_or_wait`.
2. Enrich `NoAccountError`:
   ```python
   class NoAccountError(Exception):
       def __init__(self, msg, *, queue: str,
                    reason: Literal["no_active_accounts", "all_locked"],
                    next_available_at: datetime | None): ...
   ```
   Keep the current message text (`f"No account available for queue {queue}"`) so alexandros'
   regex keeps working until it migrates.
3. Fix the race: determine "no active accounts" explicitly with
   `SELECT COUNT(*) FROM accounts WHERE active = true` instead of inferring from
   `next_available_at`. Two options:
   - **Option A — two queries** (count + next_available_at). Pros: trivial, readable. Cons: one
     extra round-trip per poll iteration (poll runs every ~5s — negligible). Recommended.
   - **Option B — one combined query** returning `(active_count, min_lock)`. Pros: single
     round-trip, atomic snapshot. Cons: more SQL for a cold path.
   Additionally: when active accounts exist and `nat` is `None` (the freed-account race), do NOT
   give up — loop again immediately (the next `get_for_queue` will claim the freed account).
4. Sleep no longer than needed:
   `min(self._wait_interval, remaining_budget, seconds_until(nat) or wait_interval)`, floored at
   ~0.1s.
5. Update `twscrape/__init__.py` exports if the exception signature changes.
6. Tests to add in `tests/test_pool.py` (pool fixture is a temp DB):
   - `wait_timeout=0.2, wait_interval=0.05`, one active account locked for the queue → returns
     `None` (or raises with `reason="all_locked"` and a datetime when `raise_when_no_account`)
     after ≥ timeout.
   - Same setup but the lock expires after ~0.1s → returns the account (the whole point of
     `7f472f7`).
   - No active accounts + `wait_timeout` set → immediate raise/None with
     `reason="no_active_accounts"`, no waiting.
   - `next_available_at` returns a tz-aware datetime equal to the stored lock.
7. Note in the task result that `alexandros` can then delete `_NO_ACCOUNT_QUEUE_RE` and
   `_pool_retry_at` and read `exc.reason` / `exc.next_available_at` instead (separate repo —
   do not change it in this task). Its `_platform_reachable` probe can also become better
   targeted (probe only when `reason == "all_locked"`).

### Task 4: `db.py` — version check on every query + wrong float comparison

**File:** `twscrape/db.py`

- `DB.__aenter__` runs `check_version()` for **every** `execute/fetchone/fetchall`, and
  `check_version` opens a fresh in-memory aiosqlite connection each time. Every pool operation
  therefore opens two SQLite connections.
- `check_version` compares versions as floats: `float("3.9") > float("3.24")`, so SQLite 3.9
  (older than 3.24) passes the check. Wrong for any single-digit minor.

Instructions:
1. Compare as tuples:
   `tuple(map(int, ver.split(".")[:2])) < tuple(map(int, MIN_SQLITE_VERSION.split(".")))`.
   Keep a guard for exotic version strings (current `ValueError → pass`).
2. Run the check once per process: module-level `_version_checked` flag set inside
   `check_version`.
3. Optional simplification: read the version from the sync `sqlite3.sqlite_version` (same
   underlying library as aiosqlite) instead of opening an in-memory async connection. Keep
   `get_sqlite_version()` working — the CLI `version` command uses it.
4. Test: extract the comparison into a pure function and unit-test it (3.9 vs 3.24 must fail,
   3.35 must pass).

### Task 5: `QueueClient.req` retry policy hardening — remaining parts

**File:** `twscrape/queue_client.py` (plus `twscrape/http.py`, `tests/test_http.py`, `tests/test_queue_client.py`)

5.1 (NetworkError cap) was merged upstream as #325; XClId failures were typed in v0.19.2.
Still open:

- `CurlClient.request` retries NetworkError 4× internally (`_CURL_MAX_RETRIES`) while httpx only
  has connect-level transport retries — nested inside the `Ctx.fails` retry logic and asymmetric
  between backends; retry responsibility should live in one layer.
- `except Exception` path: counters are per-`Ctx` now (each account gets 3 tries before the
  15-min lock), but a systemic error (e.g. a parser-breaking API change) can still serially lock
  the entire pool, 3 failures at a time.
- `Ctx.req` treats **any** 404 as a stale `x-client-transaction-id`, retries 3× and then raises
  `AbortReqError` — a genuine 404 (deleted resource) aborts the whole query and surfaces as `None`
  with no signal. Also the message has a typo: "Faield" (~line 110).
- `QueueClient._close_ctx` sets `self.req_count = 0` — attribute never declared in
  `__init__` and never read (the real counter is `ctx.req_count`). Dead leftover.
- Cloudflare/HTML block (`_check_rep`, the `text/html` branch, ~line 225) raises a bare
  `AbortReqError`, which `req()` swallows into `None` — indistinguishable from "no data" for
  callers. Concrete consumer impact: a block during a user lookup makes `api.user_by_login`
  return `None`, which alexandros reports as `social_account_not_found` — a **permanent** result
  in Perikles (entity parked for manual review) for what was a transient block.

Instructions (single task — all in the same two functions):
1. Remove the internal retry loop from `CurlClient.request` (keep the error mapping in `_wrap`)
   so both backends have identical semantics and retries happen only in `QueueClient`. Update the
   curl retry tests in `tests/test_http.py`.
2. Unknown `Exception`: after 3 distinct accounts have been locked consecutively by unknown
   errors within one `req()` call —
   - **Option A — raise the last exception**. Pros: fail fast, pool preserved, caller sees the
     real error. Recommended.
   - **Option B — keep looping** (status quo). Pros: a later account might work. Cons: can lock
     the whole pool on systemic errors.
3. `Ctx.req` 404 handling — options:
   - **Option A — after the 3 clid retries, return the last 404 response** and let `_check_rep`
     deal with it; special-case 404 there to log + `return` (yield-nothing) rather than the
     final `raise_for_status` branch that would lock the account 15 min. Pros: deleted
     tweets/users stop aborting whole scrape jobs. Recommended.
   - **Option B — keep aborting** but fix the typo and log the URL at warning level. Pros:
     minimal change. Cons: keeps conflating two unrelated conditions.
4. Fix "Faield" → "Failed" in both `AbortReqError` messages; delete the dead
   `self.req_count = 0` assignment.
5. Tests: unknown errors stop after 3 accounts (Option A); 404 returns response / does not abort
   (Option A).
6. Make Cloudflare/HTML blocks loud: add `class BlockedError(AbortReqError)` in
   `queue_client.py` carrying the source (`cloudflare`/`html`) and status code; raise it from
   the `text/html` branch of `_check_rep` and re-raise it explicitly in `req()` above the
   `AbortReqError` handler (same pattern as `GqlFeaturesOutdatedError`). Do NOT lock or
   deactivate the account — blocks are usually IP-level, rotating accounts on the same IP won't
   help, and `__aexit__` already unlocks on propagation. Export from `twscrape/__init__.py`.
   Test: mocked 403 HTML response with a `cf-ray` header → raises `BlockedError`; account
   neither deactivated nor left locked. Consumer note: alexandros should then map it by type to
   `platform_unavailable` (transient, auto-retried) in `_map_exception` — separate repo, do not
   change it in this task.

---

## P1 — Design / API decisions

### Task 6: **BREAKING** — make telemetry opt-in for this fork

**Files:** `twscrape/telemetry.py`, `tests/test_telemetry.py` (Option B also: `twscrape/queue_client.py`, `twscrape/cli.py`, `pyproject.toml`)

Upstream added opt-out anonymous telemetry (PostHog, hashed machine id, one `capture()` per GQL
request taking a `threading.Lock` in the async hot path). In library mode `flush()` is never
called, so events accumulate in memory for the process lifetime and are never sent — pure
overhead. A self-hosted scraping service should not phone home by default.

Options:
- **Option A — flip the default**: `_is_disabled()` returns True unless `TWS_TELEMETRY=1`.
  Pros: ~3-line diff, trivial upstream rebase. Cons: dead code still shipped, `py-machineid`
  still a dependency. Recommended.
- **Option B — delete the module**: remove `telemetry.py`, its call sites (queue_client, cli),
  the `py-machineid` dependency, and `tests/test_telemetry.py`. Pros: zero overhead, no lock in
  the hot path. Cons: guaranteed merge conflicts on every upstream rebase.
- **Option C — keep as upstream**. Cons: default phone-home from production scraper.

Implement A unless the operator says otherwise. Update `tests/test_telemetry.py` fixtures to set
`TWS_TELEMETRY=1`.

### Task 7: `limit` semantics are a lie — decide and implement

**Files:** `twscrape/models.py` (`_parse_items`), `twscrape/api.py`, `tests/test_pagination.py`

`_parse_items` receives `limit` but the check is dead code (`pass` where `break` was, upstream
issue #26). Pagination stops via `_is_end` counting **raw timeline entries** (incl. filtered-out
ones), not parsed items. Net effect: `search(q, limit=100)` can yield more or fewer than 100
tweets. Every public method plumbs the misleading parameter through.

Options:
- **Option A — enforce the cap at the API layer**: add an async helper in `api.py`
  (`async def _limited(gen, limit)`) that stops after `limit` yields; wrap every non-raw public
  method's `async for` with it; drop the dead `limit` handling from `_parse_items` (keep the
  parameter accepted-but-ignored in the exported `parse_*` helpers with a deprecation note, to
  soften the break). Pagination keeps using `_is_end` as a fetch heuristic. Pros: `limit`
  finally means "at most N items"; one place; removes dead code. **BREAKING** for callers relying
  on getting more than `limit`. Recommended.
- **Option B — document only**: keep signatures, add docstrings saying limit is an approximate
  pagination hint. Pros: no breakage. Cons: perpetuates the confusion.

Tests: assert exact item count with a limit smaller than one page, and one spanning multiple
pages.

### Task 8: Account selection strategy (`_order_by`)

**File:** `twscrape/accounts_pool.py` (`_order_by`, used by `get_for_queue`)

`ORDER BY username LIMIT 1` always claims the alphabetically-first free account, concentrating
requests (and ban risk / rate-limit burn) on the same accounts; later accounts idle until the
first ones are locked.

Options:
- **Option A — least-recently-used**: `ORDER BY last_used IS NOT NULL, last_used ASC` (never-used
  accounts first). Pros: even wear across the pool, deterministic, uses an existing column that is
  already bumped on every lock/unlock. Recommended.
- **Option B — `RANDOM()`** (upstream's commented-out alternative). Pros: one-word change.
  Cons: uneven short-term distribution.
- **Option C — keep `username`**. Pros: matches upstream. Cons: the concentration problem.

If A: check `tests/test_pool.py` for tests assuming username order and update them deliberately.

### Task 9: Small API-layer cleanups (single mechanical pass)

**File:** `twscrape/api.py`

1. `_is_end(self, rep, q, res, cur, cnt, lim)` — `q` is unused; drop it and update the call site.
2. Inconsistent parser input: some wrappers pass `rep.json()` (`search`, `tweet_replies`,
   `followers`, ...), others pass `rep` (`user_media`, `list_timeline`, `community_tweets`,
   `community_members`, `community_moderators`, `trends`). Works because `_parse_items` sniffs the
   type — but pick one form: pass `rep` everywhere (`Response` caches `.json()`).
3. `trends_raw`: local variable `map` shadows the builtin — rename to `timeline_ids`.
4. `search_trend` duplicates `search` except `querySource` — keep the public method but implement
   it as a one-line delegation to `self.search(q, limit, kv={"querySource": "trend_click", **(kv or {})})`.

No behavior change; `make check && make test` must stay green.

---

## P2 — Minor fixes (each a tiny standalone change)

### Task 10: Harden `get_or` against non-dict traversal

**File:** `twscrape/utils.py`
`get_or(obj, "a.b")` raises `TypeError` when `obj["a"]` is a list/str/None (`"b" not in obj`).
Guard: `if not isinstance(obj, dict) or part not in obj: return default_value`. Add a test in
`tests/test_utils.py` (e.g. `get_or({"a": [1]}, "a.b") is None`).

### Task 11: `accounts_info` renders missing error as the string "None"

**File:** `twscrape/accounts_pool.py`
`"error_msg": str(x.error_msg)[0:60]` → `"None"` when there is no error. Use
`str(x.error_msg)[:60] if x.error_msg else None`. The AccountInfo TypedDict already declares
`str | None`.

### Task 12: `load_from_file` / `guess_delim` error messages

**File:** `twscrape/accounts_pool.py`
`guess_delim` crashes with an unpacking `ValueError` if `line_format` lacks `username` (or
contains it twice). Validate first (`"username" not in line_format` → clear `ValueError`) and use
`split("username", 1)`. Add a test.

### Task 14: IMAP session hygiene

**Files:** `twscrape/login.py`, `twscrape/imap.py`
- The IMAP connection created in `login()` (email_first) or lazily in `login_confirm_email_code`
  is never logged out on success — ensure `ctx.imap.logout()` runs in a `finally` in `login()`
  (guard with try/except).
- `imap_get_email_code`'s `except` block calls `imap.select`/`imap.close`, which themselves raise
  if the connection died — wrap them so the original error propagates.
- `_wait_email_code` uses `strptime` on the `Date` header and raises for nonstandard headers,
  aborting the poll — use `email.utils.parsedate_to_datetime` (lenient, already used in
  models.py) and skip messages whose date can't be parsed.

### Task 16: Curl backend ignores UA `seed`

**File:** `twscrape/http.py`
`make_client(..., seed=...)` forwards `seed` only to `HttpxClient`; `CurlClient` drops it, so
per-account browser-family stability (seeded in `Account.make_client`) is lost on the curl
backend. Pass `seed` into `CurlClient.__init__` and use `_resolve_browser(hint, seed=seed)` for
family selection. Low impact; do it for parameter symmetry.

### Task 17: Library import side effect on loguru

**File:** `twscrape/logger.py`
`logger.remove(); logger.add(sys.stderr, filter=_filter)` runs at import time and wipes the host
application's loguru sinks (alexandros configures its own logging). Standard loguru-for-libraries
pattern:
- **Option A**: drop the global `logger.remove()`/`logger.add()`; scope the level filter to
  records with `r["name"].startswith("twscrape")`; keep `set_log_level` working. Recommended —
  but verify alexandros still receives twscrape warnings afterwards (its pool dashboard relies
  on them).
- **Option B — keep as-is** (upstream behavior). Cons: host logging config silently destroyed
  depending on import order.

### Task 18: Fork docs (do last)

**Files:** `readme.md`, `changelog.md`
- Add a short "Fork changes" section at the top of `readme.md` listing whatever Tasks 3/6/7/8
  changed (`next_available_at` returning a datetime, structured `NoAccountError`, telemetry
  default, `limit` semantics, account selection order).
- Add a `## fork (unreleased)` entry in `changelog.md` listing the same.
