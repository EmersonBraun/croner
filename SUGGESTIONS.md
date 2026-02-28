# Croner Fork - Contribution Suggestions

> Upstream repo: [Hexagon/croner](https://github.com/Hexagon/croner)
> Fork: [EmersonBraun/croner](https://github.com/EmersonBraun/croner)
> Analysis date: 2026-02-28
> Upstream version: 10.0.1

---

## Open Upstream Issues (as of 2026-02-28)

A total of **1 open issue** was found on the upstream repository.

---

### Issue #348 - `date-based jobs are not invoked`

- **URL:** https://github.com/Hexagon/croner/issues/348
- **Label:** bug
- **Created:** 2026-02-12
- **Comments:** 7
- **Reporter:** smiccoli
- **Assigned to:** Hexagon, Copilot bot

**Description:**

When scheduling jobs using a one-time date (the `once` pattern, i.e. passing a `Date` or ISO 8601 string as the first argument instead of a cron expression), jobs intermittently fail to fire. The issue does not reproduce consistently — it sometimes surfaces after 2-3 job cycles, other times after 10+ cycles, making it extremely hard to isolate.

In the reporter's setup, each date-based job is responsible for scheduling the next one upon completion (a chained scheduling pattern). After an indeterminate number of cycles, a job simply never executes: no logs are printed, no errors are thrown, and `previousRun()` remains `null` while `nextRun()` also returns `null`.

**Reproduction dump at `2026-02-12T03:00:00.000Z`:**

```json
{
  "jobs": [
    { "name": "periodic-sync", "pattern": "*/10 * * * *", "next": "2026-02-12T06:40:00.000", "previous": "2026-02-12T06:30:00.010" },
    { "name": "event-a", "once": "2026-02-12T01:36:00.000Z", "next": null, "previous": "2026-02-12T05:36:00.000" },
    { "name": "event-b", "once": "2026-02-12T08:35:00.000Z", "next": "2026-02-12T12:35:00.000", "previous": null },
    { "name": "event-c", "once": "2026-02-12T01:45:00.000Z", "next": null, "previous": null }
  ]
}
```

`event-c` had a scheduled time of `01:45 UTC` but the dump at `03:00 UTC` shows it never fired.

**Root cause analysis:**

The croner scheduler caps `setTimeout` delay at `maxDelay = 30 * 1000` ms (30 seconds) to work around the 32-bit signed integer overflow in JS engines and to handle system suspend/resume. When a job is scheduled far in the future, croner re-arms a short timer repeatedly until the target time arrives.

A likely race condition exists in this re-arming loop when multiple date-based jobs are scheduled close together in time. If the internal timer state is corrupted or a re-arm is missed (e.g., due to GC pause, event loop starvation, or a subtle state mutation bug in the chained scheduling scenario), the job can silently drop.

**Implementation plan:**

| Step | Action | Difficulty |
|------|--------|------------|
| 1 | Write a deterministic regression test using fake timers (e.g. `@sinonjs/fake-timers` or Deno's `FakeTime`) that schedules 5+ chained date-based jobs and verifies all fire | Medium |
| 2 | Add a heartbeat/watchdog log inside the internal `setTimeout` re-arm loop in `croner.ts` to expose missed re-arms | Easy |
| 3 | Audit the `CronState` object for shared mutable references that could cause cross-job interference when many jobs exist in `scheduledJobs` | Medium |
| 4 | Add a `job.getOnce()` public accessor (currently referenced in the issue dump but may not be publicly documented) | Easy |
| 5 | Consider adding a `job.isStale()` utility that returns `true` if a one-time job's target time has passed but `previousRun()` is still `null` | Easy |

**Estimated difficulty:** Medium

---

## General Contribution Suggestions

The following items are not tied to any single open issue but represent meaningful improvements to the library that would make strong pull requests.

---

### 1. Better Timezone Handling

**Current state:** Croner supports `timezone` and `utcOffset` options, but the two cannot be combined. Timezone support relies on the host runtime's `Intl` API, which has known inconsistencies across Node versions, Bun, and browsers.

**Suggested improvements:**
- Add validation that the supplied timezone string is a valid IANA identifier at options-parse time, with a clear error message listing a few examples.
- Expose a `Cron.getSupportedTimezones()` static helper that returns the runtime's available IANA timezones.
- Add a `timezone` getter on the job instance so it can be inspected after construction.
- Document edge cases where `Intl.DateTimeFormat` behavior differs between runtimes (Node 18 vs Node 20 vs Bun 1.x).

**Difficulty:** Easy to Medium
**Files to touch:** `src/options.ts`, `src/date.ts`, `README.md`

---

### 2. DST-Aware Scheduling

**Current state:** When a cron job is scheduled using wall-clock times (e.g., `0 2 * * *` — run at 2:00 AM), Daylight Saving Time transitions cause the job to either run twice (when clocks fall back) or skip entirely (when clocks spring forward). This is a well-known problem with cron systems.

**Suggested improvements:**
- Add a `dstHandling` option with values:
  - `"skip"` (default, current behavior): if the scheduled time does not exist during DST transition, skip that run.
  - `"retry"`: if the time is skipped, retry at the next valid time after the transition.
  - `"run-twice"`: during fall-back, allow the job to run twice as it would in a traditional Unix cron.
- Add a dedicated test suite covering DST transitions for both UTC+N and UTC-N timezones, using fixed timestamps around known DST transition dates.

**Difficulty:** Hard
**Files to touch:** `src/date.ts`, `src/croner.ts`, `src/options.ts`, `test/`

---

### 3. Distributed Lock Support

**Current state:** Croner is an in-memory scheduler with no built-in mechanism to prevent the same job from executing simultaneously across multiple Node.js processes or replicas (e.g., in a horizontally scaled environment).

**Suggested improvements:**
- Define a `DistributedLock` interface that consumers can implement:
  ```typescript
  interface DistributedLock {
    acquire(jobName: string, ttlMs: number): Promise<boolean>;
    release(jobName: string): Promise<void>;
  }
  ```
- Add a `lock` option to `CronOptions` that accepts a `DistributedLock` implementation.
- Before each job execution, croner would call `lock.acquire()`. If it returns `false`, treat the run as blocked (similar to how `protect` works for in-process overrun protection).
- Ship a reference implementation backed by a simple in-memory Map (for testing) with notes on how to wire in Redis, Postgres advisory locks, or similar.

**Difficulty:** Medium
**Files to touch:** `src/options.ts`, `src/croner.ts`, new `src/lock.ts`

---

### 4. Better TypeScript Types

**Current state:** The `CronOptions` interface and `Cron` class are typed, but several areas could be more precise.

**Suggested improvements:**
- The generic `<T = undefined>` context type is present but the ergonomics are awkward — the callback type should propagate `T` more explicitly.
- Add a `CronJob<T>` type alias that is easier to reference externally without importing internal types.
- The `catch` option accepts `boolean | CatchCallbackFn` — consider splitting into two separate options for clarity (`catch: boolean` and `onError: CatchCallbackFn`), or at minimum add overloaded type signatures.
- Export all public-facing types (`CronOptions`, `CronDate`, `CronPattern`, `CronMode`) from the top-level entry point so consumers do not need to import from sub-paths.
- Add strict return types to all public methods (`nextRun(): Date | null`, `previousRun(): Date | null`, etc.) and ensure they appear correctly in generated `.d.ts` files.

**Difficulty:** Easy to Medium
**Files to touch:** `src/options.ts`, `src/croner.ts`, `src/pattern.ts`, `build/`

---

### 5. Performance Improvements

**Current state:** The `scheduledJobs` global array is a flat array. Every job lookup by name requires a linear scan. With many jobs this is O(n).

**Suggested improvements:**
- Replace the `scheduledJobs` array with a `Map<string, Cron>` for O(1) named lookups. Maintain a parallel array for ordered iteration if needed.
- Profile the `nextRun()` computation for patterns with large year ranges — the date iteration loop in `date.ts` could be expensive for patterns far in the future.
- Add a benchmark suite (using `@std/assert` timers or a simple `performance.now()` harness) covering: 1000 concurrent jobs, next-run computation for 100 runs, and pattern matching against 10,000 dates.
- Memoize `CronPattern` parsing so that re-creating a `Cron` instance with the same pattern string does not re-parse it.

**Difficulty:** Easy (Map refactor) to Medium (benchmarks, memoization)
**Files to touch:** `src/croner.ts`, `src/pattern.ts`, new `bench/`

---

### 6. Bun Compatibility Improvements

**Current state:** Croner officially supports Bun >= 1.0.0, but Bun's `setTimeout`/`setInterval` implementation has historically had subtle differences from Node's (e.g., timer drift under high load, `unref()` behavior differences, and `Bun.sleep` being preferred for precision timing).

**Suggested improvements:**
- Add a Bun-specific CI job in `.github/workflows/` that runs the full test suite under the latest stable Bun release.
- Audit `unrefTimer()` in `src/utils.ts` — Bun's timer objects may not expose `.unref()` in all versions; add a runtime guard.
- Document any known behavior differences in a `COMPATIBILITY.md` file or in the existing docs.
- Test the `maxDelay` (30s cap) behavior specifically in Bun to verify that the re-arm loop does not drift more than an acceptable threshold (e.g., ±100ms) over a 10-minute window.
- Investigate whether Bun's `Bun.sleep()` or `Bun.serve()` event loop interaction causes any starvation with many simultaneous cron jobs.

**Difficulty:** Easy (CI + docs) to Medium (timer audit)
**Files to touch:** `src/utils.ts`, `.github/workflows/`, new `COMPATIBILITY.md`

---

## Prioritized Contribution Roadmap

| Priority | Item | Type | Estimated Difficulty |
|----------|------|------|----------------------|
| 1 | Fix issue #348 (date-based jobs silently not firing) | Bug fix | Medium |
| 2 | Better TypeScript exports and type ergonomics | Enhancement | Easy |
| 3 | Bun compatibility CI + unref audit | Compatibility | Easy |
| 4 | DST-aware scheduling option | Feature | Hard |
| 5 | Distributed lock interface | Feature | Medium |
| 6 | Performance: Map-based job registry + benchmarks | Enhancement | Medium |
| 7 | Timezone validation and helper utilities | Enhancement | Easy |

---

## Notes on Contributing

- The project is written in **TypeScript** and uses **Deno** as the primary development runtime; the npm package is built via `deno task build:npm`.
- Tests live in `test/` and use `@cross/test` and `@std/assert` from JSR.
- Before submitting a PR, run: `deno task pre-commit` (fmt check + lint + type check).
- The project has zero runtime dependencies — any contribution must maintain this constraint.
- Check `AGENTS.md` in the repo root for any AI-specific contribution guidelines.
