# Contribution #1: Close files, even when seeding, if they've been idle for some period of time

**Contribution Number:** 1  
**Student:** Jacob Lopez  
**Issue:** https://github.com/transmission/transmission/issues/7455 
**Status:** Phase II Complete

---

## Why I Chose This Issue

I chose this issue because the problem is concrete and well-defined. Transmission
never closes file handles on seeding torrents, even when no data has been
transferred for an extended period. This creates real problems for users whose
storage setups depend on idle file detection to manage cache space. The fix has
clear acceptance criteria, adding a configurable idle timeout that releases file
handles during seeding and reopens them when a peer requests data.

The scope feels right for a focused contribution. It touches the core file
management layer rather than requiring changes across the entire codebase, and
the maintainer has already labeled it "good first issue" and "pr welcome," which
signals it's actionable. I'm also interested in learning how a mature C++
application like Transmission manages system resources at this level.

---

## Understanding the Issue

### Problem Description

Transmission never closes file handles for seeding torrents based on idle time. Once a file is opened for a seeding torrent, the handle stays open indefinitely — it is only evicted when the 32-slot pool overflows (LRU order) or the torrent is removed entirely.

### Expected Behavior

File handles that haven't been accessed for a configurable period (e.g., 30 seconds) should be closed automatically. Handles should reopen transparently when a peer next requests data from that file.

### Current Behavior

`tr_open_files` caches up to 32 file descriptors and evicts only on overflow or torrent removal. There is no time-based eviction, so handles for seeding torrents accumulate and never release on idle.

### Affected Components

- `libtransmission/open-files.h` / `open-files.cc` — the file descriptor pool (`tr_open_files` and its `Val` struct)
- `libtransmission/session.cc` — the per-second `on_now_timer()` hook where the idle sweep belongs
- `libtransmission/session-settings.h` — where the new configurable threshold will live

---

## Reproduction Process

### Environment Setup

- **OS:** macOS (Darwin 24.6.0, Apple Silicon)
- **Build system:** CMake 4.3.2, Apple Clang (via Xcode toolchain)
- No issues — all third-party dependencies (googletest, fmt, dht) are bundled as git submodules, so no manual installs were needed.

```bash
git clone --recurse-submodules https://github.com/jlopez695/transmission transmission
cd transmission
git checkout fix-issue-7455
cmake -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build --target libtransmission-test -j8
```

### Steps to Reproduce

1. Build the unit tests using the commands in Environment Setup above.

2. Confirm that `tr_open_files::Val` — the struct representing a cached open file — has no idle timestamp:
   ```bash
   grep -n "last_use\|close_idle\|time_t" libtransmission/open-files.h
   ```
   Expected if the feature existed: lines showing a `last_use_` field and a `close_idle()` method.
   Actual result: no output — neither exists.

3. Confirm the LRU cache uses only an ordinal counter, not wall-clock time:
   ```bash
   grep -n "sequence_\|time_t\|tr_time" libtransmission/lru-cache.h
   ```
   Actual result: only `sequence_` (an incrementing counter) — no timestamp anywhere in the eviction logic.

4. Confirm no time-based eviction test exists:
   ```bash
   grep -rn "idle\|close_idle\|last_use" tests/libtransmission/open-files-test.cc
   ```
   Actual result: no output.

5. Run the existing open-files test suite:
   ```bash
   ./build/tests/libtransmission/libtransmission-test --gtest_filter="OpenFilesTest.*"
   ```
   Output:
   ```
   [==========] Running 10 tests from 1 test suite.
   [  PASSED  ] 10 tests.
   ```
   Observed result: all 10 tests pass and none cover time-based idle eviction — the feature does not exist.

### Reproduction Evidence

- **Commit showing reproduction:** https://github.com/jlopez695/transmission/tree/fix-issue-7455
- **Screenshots/logs:** The grep in step 2 returns no output. The test run in step 5 shows 10 passing tests with zero idle-eviction coverage.
- **My findings:** The root cause is `tr_open_files::Val` in `libtransmission/open-files.h` (lines 48–67). It holds only `fd_` (the file descriptor) and `writable_` — no `last_use_` timestamp. The LRU cache backing the pool tracks order via an ordinal `sequence_` counter, not wall-clock time. Without a timestamp on each cached entry, there is no way to evict handles based on idle time. The session's `on_now_timer()` fires every second and is the correct hook for a periodic idle-close sweep, but currently contains no such call.

---

## Solution Approach

### Analysis

The root cause is in `tr_open_files::Val` in `libtransmission/open-files.h` (lines 48–67). This struct represents a cached open file handle and stores only `fd_` (the file descriptor) and `writable_` (open mode). There is no `last_use_` timestamp. Without a timestamp on each cached entry, there is no way to measure how long a handle has been idle, making time-based eviction impossible at the pool level.

### Proposed Solution

Add a `time_t last_use_` field to `tr_open_files::Val` and stamp it on every cache hit and insertion in `get()`. Add a `close_idle()` method that uses the existing `pool_.erase_if()` mechanism to evict handles idle longer than a configurable threshold. Wire `close_idle()` into `on_now_timer()` in `session.cc` and expose the threshold as a new `open_file_idle_secs` setting via the C API and RPC/JSON API.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Transmission's file descriptor pool has no concept of idle time. Handles opened for seeding stay open until the pool overflows (32 slots) or the torrent is removed. The fix adds a timestamp to each cached handle and a periodic sweep that closes handles not accessed within a configurable threshold.

**Match:** The existing `close_torrent()` in `open-files.cc` uses `pool_.erase_if(lambda)` to evict all handles for a given torrent — the exact same mechanism `close_idle()` will use, with a time-based condition instead of a torrent ID match. The `queue_stalled_minutes` field in `session-settings.h` is the direct model for wiring a new configurable threshold through the quark/settings/RPC stack.

**Plan:** [Step-by-step implementation plan]
1. Add `time_t last_use_ = 0` to `Val` in `open-files.h`; update move constructor/operator to swap it; declare `void close_idle(time_t now, time_t threshold)`
2. In `open-files.cc`, stamp `last_use_ = tr_time()` in both `get()` overloads on cache-hit and cache-insertion; implement `close_idle()` with `pool_.erase_if([&](Key const&, Val const& v){ return now - v.last_use_ >= threshold; })`
3. In `session-settings.h`, add `size_t open_files_expire_secs = 30U` and its `Field<>` entry referencing `TR_KEY_open_file_idle_secs`
4. In `quark.h`, add `TR_KEY_open_file_idle_secs` enum value; in `quark.cc`, add `"open_file_idle_secs"sv` at the matching position
5. In `transmission.h`, declare `tr_sessionGetOpenFilesIdleSecs()` and `tr_sessionSetOpenFilesIdleSecs()` in the public C API
6. In `session.cc`, implement the getter/setter; add `open_files_.close_idle(tr_time(), settings_.open_files_expire_secs)` to `on_now_timer()`
7. In `rpcimpl.cc`, add `TR_KEY_open_file_idle_secs` getter/setter cases following the `queue_stalled_minutes` pattern
8. In `open-files-test.cc`, add `closeIdleClosesIdleFiles`: open a handle, call `close_idle()` past the threshold, verify `get()` returns nullopt; add negative case where threshold is not yet reached

**Implement:** https://github.com/jlopez695/transmission/tree/fix-issue-7455

**Review:** Run `./code_style.sh` before submitting (required by CONTRIBUTING.md). Confirm the new setting is reachable via both C API and RPC/JSON API (required by CONTRIBUTING.md line 71). Run the full libtransmission test suite and confirm all tests pass. Add a one-sentence `Notes:` line to the PR description for the release notes script.

**Evaluate:** New `OpenFilesTest.closeIdleClosesIdleFiles` test must pass. All 10 existing `OpenFilesTest.*` tests must continue to pass. Manual check: start `transmission-daemon` with a short idle threshold, add a seeding torrent, use `lsof -p <pid>` to confirm handles close after idle and reopen on the next request.

---

## Testing Strategy

### Unit Tests

Added to `tests/libtransmission/open-files-test.cc`:

- [x] `closeIdleClosesIdleFiles`: caches two files, refreshes only one, then sweeps past the threshold — confirms the idle file's handle is closed and the recently-used one is kept
- [x] `closeIdleKeepsRecentlyUsedFiles`: sweeps just before the threshold — confirms the handle stays cached
- [x] All 10 pre-existing `OpenFilesTest` cases still pass

### Integration Tests

- [x] `RpcTest.sessionGet`: confirms `open_file_idle_secs` is returned by the `session-get` RPC method (verifies the setting is reachable via the RPC/JSON API)
- [x] Full `libtransmission-test` suite: 573 passed, 0 failed (1 pre-existing unrelated skip)

### Manual Testing

Not yet run. Planned check: start `transmission-daemon` with `"open-file-idle-secs": 30` in `settings.json`, add a seeding torrent, and use `lsof -p <pid>` to confirm a handle is released after ~30s idle and re-opened on the next request.

---

## Implementation Notes

### Week 1 Progress (June 22, 2026)

**What I built:** the full `open_file_idle_secs` feature end-to-end — a configurable idle timeout that closes cached file handles (even while seeding) once they've gone unused past the threshold, reopening them transparently on the next request.

- Added a `time_t last_use_` timestamp to `tr_open_files::Val` and stamp it on every cache hit and insertion in `get()`.
- Added `tr_open_files::close_idle()`, which evicts idle handles via the cache's existing `pool_.erase_if()` (the same path `close_torrent()` uses).
- Wired `close_idle()` into `on_now_timer()` (runs once per second); it's a no-op when the setting is 0.
- Added the `open_file_idle_secs` setting (default 30s, 0 disables) and exposed it via both the C API and the RPC/JSON API.

**Challenges faced:**

- The C API setter didn't compile at first because `settings_` is private — the existing C setters get access through a `friend` list in `session.h`, so I added mine there.
- `RpcTest.sessionGet` failed after wiring, since it asserts the exact set of `session-get` keys; this confirmed the key is exposed, and I added it to the test's expected list.
- Couldn't run `./code_style.sh` locally (installed clang-format is v17; the repo's `.clang-format` needs v20), so I modeled every change on adjacent code and hand-checked the 128-column limit.

### Code Changes

- **Files modified:**
  - `libtransmission/open-files.{h,cc}` — `last_use_` field + `close_idle()`
  - `libtransmission/session.{cc,h}` — timer hook, C API getter/setter, accessor, friend declaration
  - `libtransmission/session-settings.h` — new setting + serializer field
  - `libtransmission/quark.{h,cc}` — `TR_KEY_open_file_idle_secs`
  - `libtransmission/transmission.h` — C API declarations
  - `libtransmission/rpcimpl.cc` — RPC getter/setter
  - `tests/libtransmission/open-files-test.cc` — 2 new tests
  - `tests/libtransmission/rpc-test.cc` — expected-key update
- **Key commits:** [`fix-issue-7455` branch](https://github.com/jlopez695/transmission/tree/fix-issue-7455)
- **Approach decisions:** reused the existing `pool_.erase_if()` eviction path rather than adding new machinery; kept the time concept in `tr_open_files` instead of the generic `tr_lru_cache`; used `0` to disable rather than adding a separate enabled flag.

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
