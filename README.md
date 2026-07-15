# su26-ai-301-contribution2


# Open Source Contribution Log

CodePath AI301: My contribution journey from issue selection to merged pull request. This is a living document. I update it at the end of every phase.

| Field | Details |
|-------|---------|
| **Name** | Lenin Goud Athikam |
| **GitHub** | [@leninathikam](https://github.com/leninathikam) |
| **Status** | Phase III Complete (PR open, awaiting maintainer review) |
| **Project** | [huggingface/smolagents](https://github.com/huggingface/smolagents), lightweight Python library for building AI agents |
| **Issue** | [#2464](https://github.com/huggingface/smolagents/issues/2464): `local_python_executor.timeout()` deadlocks when the wrapped call hangs forever |
| **PR** | [#2465](https://github.com/huggingface/smolagents/pull/2465): Fix: avoid deadlock in local_python_executor timeout decorator |

---

## Phase I: Issue Selection

### Issue

[huggingface/smolagents #2464](https://github.com/huggingface/smolagents/issues/2464): BUG: `local_python_executor.timeout()` deadlocks when the wrapped call hangs forever

### Project

[smolagents](https://github.com/huggingface/smolagents) is a Hugging Face library for building AI agents that can run code, call tools, and coordinate multi-step workflows. The `local_python_executor` module runs Python code in a sandboxed environment and uses a `timeout()` decorator to cap how long each execution step can run. Contributing follows the standard open source flow: fork the repo, fix the bug with tests, run `make quality` and `make test`, and open a PR against `main`.

### Why I Chose This Issue

The `timeout()` decorator is supposed to stop runaway agent code after a set number of seconds. Instead, when a wrapped function hangs (for example a long sleep, a `while True` loop, or a stuck network call inside a tool), the process freezes forever. No exception is raised and no traceback appears. That is a real bug for anyone running agents in production or on restricted compute nodes.

I picked this issue because:

- The scope was small and clear: fix one decorator in `local_python_executor.py` and add a regression test
- The reporter included a minimal reproduction script and a proposed fix, so the problem was easy to understand
- It sits at the intersection of **agents** and **systems behavior** (threading, timeouts), which I wanted to learn
- No one had claimed the issue or opened a PR when I started
- "Done" meant the caller gets `ExecutionTimeoutError` and can continue, instead of the whole process blocking on `executor.shutdown(wait=True)`

### Community Engagement

- Commented on [#2464](https://github.com/huggingface/smolagents/issues/2464#issuecomment-4878110181) to say I was working on it and shared my plan (regression test, `shutdown(wait=False)` on timeout, keep success path unchanged)
- Forked the project: [https://github.com/leninathikam/smolagents](https://github.com/leninathikam/smolagents)
- Opened PR [#2465](https://github.com/huggingface/smolagents/pull/2465) (open, awaiting maintainer review and CI workflow approval)
- Addressed GitHub Copilot review feedback in a follow-up commit on the same PR

---

## Phase II: Reproduce & Plan

### Understanding the Issue

**In my own words, what is actually being asked for?**

**Problem statement:** The `timeout()` decorator in `smolagents.local_python_executor` is meant to cap how long sandboxed agent code can run. When the wrapped function hangs longer than `timeout_seconds`, the caller should get `ExecutionTimeoutError` and move on. Instead, the whole Python process can freeze with no exception and no traceback.

**What was wrong:** The decorator used a `with ThreadPoolExecutor(max_workers=1) as executor:` block. When `future.result(timeout=timeout_seconds)` timed out, Python still exited the context manager, which calls `executor.shutdown(wait=True)`. If the worker thread was stuck in a long sleep, an infinite loop, or a blocked network call, that thread never finishes. `shutdown(wait=True)` then blocks forever waiting for it. Python cannot force-kill a running thread, so the main process hangs silently.

**Root cause in the old code:**

- `ThreadPoolExecutor` context manager always shuts down with `wait=True` on exit
- On timeout, the worker thread is still alive, so shutdown blocks indefinitely
- The `ExecutionTimeoutError` path was never reached by the caller because shutdown happened before control returned

**Success criteria:**

- A hanging call raises `ExecutionTimeoutError` within about `timeout_seconds`
- The main process continues (no freeze at shutdown)
- Successful calls still behave the same
- Existing timeout tests in `tests/test_local_python_executor.py::TestTimeout` still pass
- Only two files change: `local_python_executor.py` and the test file

### Reproduction Process

**Setup path:** I did **not** use a dev container. I followed install and test instructions from [CONTRIBUTING.md](https://github.com/huggingface/smolagents/blob/main/CONTRIBUTING.md) and inspected CI config in `.github/workflows/quality.yml` and `.github/workflows/tests.yml` to see that PRs must pass `ruff check`, `ruff format --check`, and `pytest ./tests/` on Ubuntu with Python 3.10 and 3.12.

#### Environment Setup

| Item | Detail |
|------|--------|
| **OS** | Windows 11 |
| **Python** | 3.14.3 (local); CI uses 3.10 and 3.12 on Ubuntu |
| **Package** | smolagents 1.27.0.dev0 (cloned from `main`) |
| **Fork** | [https://github.com/leninathikam/smolagents](https://github.com/leninathikam/smolagents) |
| **Branch** | `fix/timeout-deadlock-2464` ΓÇö [https://github.com/leninathikam/smolagents/tree/fix/timeout-deadlock-2464](https://github.com/leninathikam/smolagents/tree/fix/timeout-deadlock-2464) |

**Challenge encountered and fix:**

- **Error:** `pip install -e ".[dev]"` failed with `ERROR: No matching distribution found for mlx[cpu]`
- **Cause:** The `test` extra in `pyproject.toml` lists `mlx[cpu]`, which is macOS-only and not available on Windows
- **Fix:** Installed the base package plus manual test deps: `pip install -e .` then `pip install pytest numpy pandas ruff`

**Setup steps:**

1. Clone upstream and create a branch named after the issue:
   ```bash
   git clone https://github.com/huggingface/smolagents.git
   cd smolagents
   git checkout -b fix/timeout-deadlock-2464
   ```
2. Install dependencies (see challenge above if `".[dev]"` fails on Windows).
3. Point `origin` at your fork and push the branch when ready:
   ```bash
   git remote rename origin upstream
   git remote add origin https://github.com/leninathikam/smolagents.git
   ```

#### Reproduction Steps

Steps another person could follow without extra context:

1. Open `src/smolagents/local_python_executor.py` and locate the `timeout()` decorator (around lines 285ΓÇô320).
2. Create a small Python script (or REPL) with the reproduction from issue #2464:
   ```python
   import time
   from smolagents.local_python_executor import timeout, ExecutionTimeoutError

   @timeout(1)
   def hangs_forever():
       time.sleep(30)
       return "done"

   hangs_forever()
   ```
3. Run the script against the **original** code (with `with ThreadPoolExecutor(...)`).
4. Observe whether the process returns an exception or freezes.
5. Optionally run existing tests: `pytest tests/test_local_python_executor.py::TestTimeout -v` and note that `test_timeout_decorator_raises_error_when_exceeded` uses `sleep(2)` with a 1s timeout, which eventually completes because the worker thread wakes up.

#### Expected vs Actual Behavior

| | Behavior |
|---|----------|
| **Expected** | After 1 second, `hangs_forever()` raises `ExecutionTimeoutError` with message `Code execution exceeded the maximum execution time of 1 seconds`. The main script continues and can run more code. |
| **Actual (buggy code)** | `future.result(timeout=1)` times out internally, but exiting the `with ThreadPoolExecutor(...)` block calls `executor.shutdown(wait=True)`. The worker thread is still in `time.sleep(30)` (or an infinite loop), so shutdown blocks forever. No exception reaches the caller. The process appears alive with near-zero CPU and no traceback. |

**Status:** Bug reproduced and understood. Fix direction confirmed before implementation.

### Solution Approach (UMPIRE)

#### Understand

Restate the problem: The `timeout()` decorator must stop long-running agent code and raise `ExecutionTimeoutError`. The bug is not that `future.result(timeout=...)` fails to time out. The bug is that **executor shutdown after timeout blocks forever** when the worker thread never exits.

**Root cause (not just symptom):**

- Symptom: process freezes on hung calls
- Root cause: `ThreadPoolExecutor` context manager calls `shutdown(wait=True)` on exit; Python cannot kill a stuck worker thread, so `wait=True` deadlocks when the wrapped call hangs indefinitely

**Files and functions involved:**

| Path | Role |
|------|------|
| `src/smolagents/local_python_executor.py` | `timeout()` decorator (~lines 285ΓÇô325), `ExecutionTimeoutError` class |
| `tests/test_local_python_executor.py` | `TestTimeout` class ΓÇö existing timeout tests |

#### Match

Analogous pattern already in the codebase:

- **`test_timeout_works_in_thread`** (`tests/test_local_python_executor.py`, ~line 2238): Documents that the threading-based `timeout()` replaced signal-based timeouts because signals only work on the main thread. My fix must preserve timeout behavior when called from worker threads, not only the main thread.
- **`timeout()` docstring** (lines 298ΓÇô301): Already warns that timed-out threads keep running in the background. That matches using `shutdown(wait=False)` on timeout: we accept a leaked background thread rather than blocking the caller.
- **Existing test `test_timeout_decorator_raises_error_when_exceeded`**: Uses `sleep(2)` with 1s timeout. This passes on buggy code because the worker eventually finishes. My new test must use a **non-terminating** hang to catch the shutdown deadlock.

#### Plan

1. **Modify** `src/smolagents/local_python_executor.py`:
   - Remove `with ThreadPoolExecutor(...)`
   - Use explicit `executor.shutdown(wait=False)` on timeout, `shutdown(wait=True)` otherwise (via `finally` + `timed_out` flag)
2. **Add** `test_timeout_decorator_does_not_deadlock_on_hanging_call` in `tests/test_local_python_executor.py`:
   - Hanging loop with `threading.Event` so the worker can stop after the assertion
   - Assert elapsed time under 3 seconds
3. **Run** `ruff check` / `ruff format --check` on `examples`, `src`, `tests`
4. **Run** `pytest tests/test_local_python_executor.py::TestTimeout`
5. **Open PR** with `Fixes #2464`

**Edge cases considered proactively:**

| Case | Expected behavior after fix |
|------|---------------------------|
| Call completes within timeout | Returns normally; executor shuts down with `wait=True` |
| Call exceeds timeout (`sleep` longer than limit) | Raises `ExecutionTimeoutError`; does not block on shutdown |
| Call hangs forever (`while True` loop) | Raises `ExecutionTimeoutError` within ~timeout seconds; test finishes in under 3s |
| Call from non-main thread | Still works (must not regress `test_timeout_works_in_thread`) |
| Non-timeout exception in worker | Executor still shut down via `finally` with `wait=True` |

#### Review

Self-review against project guidelines before coding:

- [ ] Read [CONTRIBUTING.md](https://github.com/huggingface/smolagents/blob/main/CONTRIBUTING.md): `make quality` and `make test` expected before PR
- [ ] CI (`.github/workflows/quality.yml`, `tests.yml`): `ruff` + full `pytest ./tests/` on Ubuntu
- [ ] PR links issue: `Fixes #2464`
- [ ] Diff limited to decorator fix + one regression test
- [ ] No unrelated refactors

#### Evaluate

How I will verify the fix works:

| Check | Method |
|-------|--------|
| Regression test | `test_timeout_decorator_does_not_deadlock_on_hanging_call` passes in under 3s |
| Existing behavior | All 11 tests in `TestTimeout` pass |
| Lint/format | `ruff check examples src tests` and `ruff format --check examples src tests` pass |
| Manual repro | Issue #2464 script raises `ExecutionTimeoutError` instead of freezing |
| CI | GitHub Actions on PR #2465 (after maintainer approves workflows) |

*Implement: see Phase III.*

---

## Phase III: Implement

### What I Built

I implemented the fix planned in Phase II on branch [`fix/timeout-deadlock-2464`](https://github.com/leninathikam/smolagents/tree/fix/timeout-deadlock-2464) and opened [PR #2465](https://github.com/huggingface/smolagents/pull/2465) against `huggingface/smolagents` (`Fixes #2464`).

**Net change:** +34 / -9 across 2 files.

| File | Change |
|------|--------|
| `src/smolagents/local_python_executor.py` | Stop using `with ThreadPoolExecutor(...)` so timeout can call `shutdown(wait=False)` instead of blocking on `shutdown(wait=True)` |
| `tests/test_local_python_executor.py` | Add `test_timeout_decorator_does_not_deadlock_on_hanging_call` regression test |

### Code Changes

**Before (deadlock path):**

```python
with ThreadPoolExecutor(max_workers=1) as executor:
    future = executor.submit(func, *args, **kwargs)
    try:
        result = future.result(timeout=timeout_seconds)
        return result
    except FuturesTimeoutError:
        raise ExecutionTimeoutError(...)
# context manager exit always calls shutdown(wait=True) -> hang if worker never finishes
```

**After (non-blocking timeout path):**

```python
executor = ThreadPoolExecutor(max_workers=1)
future = executor.submit(func, *args, **kwargs)
timed_out = False
try:
    return future.result(timeout=timeout_seconds)
except FuturesTimeoutError:
    timed_out = True
    raise ExecutionTimeoutError(
        f"Code execution exceeded the maximum execution time of {timeout_seconds} seconds"
    )
finally:
    # Do not wait for the stuck worker thread on timeout: shutdown(wait=True) would
    # deadlock when the wrapped call hangs indefinitely.
    executor.shutdown(wait=not timed_out)
```

**Regression test idea:** Use a `threading.Event`-gated loop so the worker can exit after the assertion. Assert `ExecutionTimeoutError` and that elapsed time is under 3 seconds. That catches the real deadlock case (`while True`-style hang), which the older `sleep(2)` test could miss because the worker eventually finishes and `shutdown(wait=True)` unblocks.

### Local Verification

| Check | Result |
|-------|--------|
| `pytest tests/test_local_python_executor.py::TestTimeout` | Passed (including the new hanging-call test) |
| `ruff check` / `ruff format --check` on `examples`, `src`, `tests` | Passed |
| Manual repro from issue #2464 (`@timeout(1)` + long sleep / hang) | Raises `ExecutionTimeoutError`; process continues |
| Existing timeout behavior (success path, thread-safe timeout) | Unchanged; no regressions in `TestTimeout` |

### Review Feedback Addressed

GitHub Copilot reviewed the first commit and left two comments. I fixed both in a follow-up commit on the same PR.

| Feedback | Fix |
|----------|-----|
| Success / non-timeout paths could leak the executor because `shutdown(wait=True)` sat in an `else` after an early `return` | Restructured with `timed_out` + `finally` so every exit path shuts down; timeout uses `wait=False`, everything else uses `wait=True` |
| Hanging test loop had no stop condition, so the worker could live for the rest of the suite | Added `stop_event` and set it in a `finally` after the timeout assertion |

### Commits

| Commit | Message |
|--------|---------|
| [`3ec20ca`](https://github.com/leninathikam/smolagents/commit/3ec20ca7bac5615f5d714bb29fdc24c7dcae02a4) | `fix: avoid deadlock in local_python_executor timeout decorator` |
| [`4886c2a`](https://github.com/leninathikam/smolagents/commit/4886c2ae80c82a48ec3b990fae66a53c7b9bfb81) | `address review: use finally for executor shutdown and stop hanging test thread` |

### Challenges / Decisions

- **Windows `.[dev]` install:** Same Phase II issue (`mlx[cpu]` is macOS-only). Kept using base install + manual test deps so I could run `pytest` and `ruff` locally.
- **Do not kill the worker thread:** Python cannot force-stop a stuck thread. The intentional tradeoff (already noted in the decorator docstring) is a leaked background thread on timeout rather than freezing the caller. That matches `shutdown(wait=False)` on the timeout path.
- **Keep the diff small:** Only the decorator and one regression test changed. No drive-by refactors, so review stays focused on the deadlock fix.
- **CI on first-time contributors:** PR workflows still need maintainer approval before GitHub Actions runs on #2465. Local `TestTimeout` + ruff were the verification until CI is approved.

### Phase III Status

Implementation and local testing are done. PR [#2465](https://github.com/huggingface/smolagents/pull/2465) is open and waiting on maintainer review (and CI workflow approval). Phase IV will cover maintainer feedback, any follow-up commits, and merge outcome.

---

<!-- Phase IV to be added as work continues -->
