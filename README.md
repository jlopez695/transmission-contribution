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

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

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

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

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

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

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
