# su26-ai-301-contribution2


# Open Source Contribution Log

CodePath AI301: My contribution journey from issue selection to merged pull request. This is a living document. I update it at the end of every phase.

| Field | Details |
|-------|---------|
| **Name** | Lenin Goud Athikam |
| **GitHub** | [@leninathikam](https://github.com/leninathikam) |
| **Status** | Phase II Complete (PR open, awaiting maintainer review) |
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

**Steps I took to set up the project locally and reproduce the problem.**

#### Environment Setup

- **OS:** Windows 11
- **Python:** 3.14.3 (local); CI uses 3.10 and 3.12 on Ubuntu
- **Package:** smolagents 1.27.0.dev0 (cloned from `main`)

**Setup steps:**

1. Cloned upstream repo:
   ```bash
   git clone https://github.com/huggingface/smolagents.git
   cd smolagents
   git checkout -b fix/timeout-deadlock-2464
   ```
2. Installed the package and test dependencies:
   ```bash
   pip install -e .
   pip install pytest numpy pandas ruff
   ```
   Note: `pip install -e ".[dev]"` failed on Windows because the test extra pulls in `mlx[cpu]`, which is macOS-only. For local work, installing the base package plus pytest and ruff was enough.

3. Read the bug location at `src/smolagents/local_python_executor.py` (the `timeout()` decorator around lines 285–320).

4. Ran the reproduction pattern from the issue:
   ```python
   import time
   from smolagents.local_python_executor import timeout, ExecutionTimeoutError

   @timeout(1)
   def hangs_forever():
       time.sleep(30)
       return "done"

   hangs_forever()  # expected: ExecutionTimeoutError; old code: process freezes
   ```

**What I observed:**

- With the original `with ThreadPoolExecutor(...)` implementation, a call that outlives the timeout could block the process on executor shutdown instead of returning quickly
- The issue reporter’s proposed fix (`shutdown(wait=False)` on the timeout path) matched the behavior I expected
- Existing tests like `test_timeout_decorator_raises_error_when_exceeded` used `time.sleep(2)` with a 1-second timeout. That case eventually completes because the worker thread wakes up, so it did not catch an infinite hang

**Status:** Bug understood and reproduction path confirmed. Fix direction was clear before coding.

### Solution Approach

**My planned fix. Which files are involved? What will I change and why?**

#### Plan

1. Replace the `with ThreadPoolExecutor(...)` context manager with explicit executor lifecycle control
2. On `FuturesTimeoutError`, call `executor.shutdown(wait=False)` so a stuck worker does not block the caller
3. On success (and non-timeout paths), call `executor.shutdown(wait=True)` so resources are cleaned up normally
4. Add a regression test where the wrapped function hangs in a loop, and assert the test finishes in under ~3 seconds with `ExecutionTimeoutError`
5. Run `ruff check` / `ruff format --check` and `pytest tests/test_local_python_executor.py::TestTimeout` before opening the PR

#### Key changes (as implemented)

**`src/smolagents/local_python_executor.py`:**

- Create the executor manually instead of using `with`
- Track `timed_out` when `FuturesTimeoutError` is raised
- Use a `finally` block: `executor.shutdown(wait=not timed_out)` so timeout uses non-blocking shutdown and all other paths wait for the worker

**`tests/test_local_python_executor.py`:**

- Add `test_timeout_decorator_does_not_deadlock_on_hanging_call`
- Use a `threading.Event` in the hanging loop so the background worker can stop after the assertion (addresses Copilot feedback on test hygiene)

#### Files to modify

| File | Change |
|------|--------|
| `src/smolagents/local_python_executor.py` | Fix `timeout()` shutdown behavior |
| `tests/test_local_python_executor.py` | Add regression test for hanging calls |

#### CONTRIBUTING.md checklist (reviewed before coding)

- Read [CONTRIBUTING.md](https://github.com/huggingface/smolagents/blob/main/CONTRIBUTING.md): run `make quality` and `make test` before submitting
- CI runs `ruff check`, `ruff format --check`, and `pytest ./tests/` on Ubuntu
- PR should link the issue with `Fixes #2464`

---

<!-- Phase III and IV to be added as work continues -->
