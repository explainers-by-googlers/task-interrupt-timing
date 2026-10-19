# Explainer for the Task Interrupt Timing API

This proposal is an early design sketch by [Chrome Webium] to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.


## Proponents

- Eriko Kurimoto (@elkurin)

## Participate
- https://github.com/explainers-by-googlers/task-interrupt-timing/issues

## Table of Contents [if the explainer is longer than one printed page]

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
  - [Use case 1: Diagnosing micro-stutters in the Webium Product](#use-case-1-diagnosing-micro-stutters-in-the-webium-product)
  - [Use case 2: Real User Monitoring (RUM) and Telemetry](#use-case-2-real-user-monitoring-rum-and-telemetry)
- [Potential Solution](#potential-solution)
  - [Option A: Extending `PerformanceLongTaskTiming` (`longtask`)](#option-a-extending-performancelongtasktiming-longtask)
  - [Option B: Dedicated `task-interrupt` entry type](#option-b-dedicated-task-interrupt-entry-type)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
- [Detailed design discussion](#detailed-design-discussion)
  - [Mid-execution interrupts vs. Post-mortem aggregation](#mid-execution-interrupts-vs-post-mortem-aggregation)
  - [Handling multiple observers with different thresholds](#handling-multiple-observers-with-different-thresholds)
  - [Lock-free task observation and idle suspension](#lock-free-task-observation-and-idle-suspension)
- [Considered alternatives](#considered-alternatives)
  - [Long Animation Frames (LoAF) API](#long-animation-frames-loaf-api)
  - [JS Self-Profiling API (`Profiler`)](#js-self-profiling-api-profiler)
  - [Global threshold configuration](#global-threshold-configuration)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)
- [Appendix: Implementation Feasibility (Chromium PoC)](#appendix-implementation-feasibility-chromium-poc)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Modern web applications strive for smooth, responsive user interfaces. However, developers often struggle to identify and eliminate "jank"—micro-stutters caused by JavaScript tasks hogging the main thread. 

While tools exist to measure long frames or very long tasks (>50ms), developers lack a programmatic way to detect and diagnose shorter, highly-contended tasks (e.g., 5ms - 15ms) that consistently burn CPU cycles and delay responsiveness. Furthermore, existing APIs provide post-mortem, aggregated script attribution, which often fails to point developers to the exact line of code that was blocking the thread at the critical moment.

The **Task Interrupt Timing API** proposes a new `PerformanceObserver` entry type that allows developers to set a custom, low-duration threshold. When a task exceeds this threshold, the browser triggers a mid-execution interrupt to capture a precise, point-in-time stack trace of the offending script.


## Goals

* Provide a programmatic way to monitor individual JavaScript task execution times against custom, low-duration thresholds (e.g., 5ms or 10ms).
* Capture and expose precise, point-in-time JavaScript stack traces when a task exceeds the configured threshold.
* Ensure the detection and observation mechanisms have near-zero performance overhead on the main thread's critical path.


## Non-goals

* **Measuring total frame duration:** This API is explicitly designed to measure isolated JavaScript tasks, not the entire rendering pipeline or layout/paint times. (The Long Animation Frames API already solves this).
* **Replacing DevTools Profilers:** This API is for programmatic, lightweight monitoring in the wild, not for capturing full continuous CPU profiles.


## Use cases

This proposal is primarily driven by the performance requirements of the **Webium Product project**, which needs strict, programmatic control over main-thread responsiveness to deliver a stutter-free user experience. Webium's requirements highlight a broader challenge faced by complex, highly interactive web applications.


### Use case 1: Diagnosing micro-stutters in the Webium Product

In the Webium Product, even small tasks (e.g., 5-15ms) can cause noticeable micro-stutters if they occur during critical rendering or interaction phases. The project needs to enforce strict sub-50ms performance budgets and understand exactly which JavaScript function is blocking the thread at the moment the budget is breached. 

Current post-mortem tools only provide aggregated script attribution after a long frame finishes, which is insufficient for Webium's diagnostic needs. By capturing a precise, point-in-time stack trace exactly when a short threshold is exceeded, the Webium team (and developers of similar complex apps) can pinpoint and eliminate the exact source of main-thread contention in the wild.

### Use case 2: Real User Monitoring (RUM) and Telemetry

Performance analytics providers (e.g., third-party RUM scripts) currently struggle to report actionable main-thread contention data from actual users. While they can report that a user experienced a long frame, they cannot programmatically capture the offending JavaScript stack trace to send back to the server. This API would allow telemetry scripts to automatically capture and aggregate point-in-time stack traces for tasks that violate performance SLAs in production, giving site owners actionable data to fix jank.


<!-- In your initial explainer, you shouldn't be attached or appear attached to any of the potential
solutions you describe below this. -->

## Potential Solution

When a JavaScript task begins on the main thread, a background monitor tracks its elapsed execution time. If the task continues executing past the configured `durationThreshold`, the browser requests an immediate interrupt from the JavaScript engine. At the next safe execution point, the engine captures the current execution stack trace. Once the task completes, the entry is dispatched to the `PerformanceObserver`.

We are currently evaluating two shapes for exposing this via `PerformanceObserver`:

### Option A: Extending `PerformanceLongTaskTiming` (`longtask`)

Following feedback from the Web Performance Working Group to avoid introducing a separate entry type with overlapping task-measurement concepts, we can extend the existing `"longtask"` entry type with an opt-in `takeTracing` flag and support for custom `durationThreshold` (clamped to a minimum of 1ms, defaulting to 50ms):

```webidl
partial dictionary PerformanceObserverInit {
    DOMHighResTimeStamp durationThreshold;
    boolean takeTracing = false;
};

partial interface PerformanceLongTaskTiming {
    // The captured stack trace when the threshold was crossed,
    // populated only for observers that opted in via takeTracing: true.
    readonly attribute FrozenArray<DOMString> stackTrace;
};
```

* **Pros:** Avoids proliferating a new `PerformanceEntry` type for task duration measurement; backward-compatible for existing `"longtask"` observers (which continue using the 50ms threshold with the watchdog disabled).
* **Cons:** The existing Long Tasks API is largely superseded by Long Animation Frames (LoAF) for general responsiveness, and overloading `durationThreshold` to both filter entries and control mid-task interrupt timing can be surprising.

### Option B: Dedicated `task-interrupt` entry type

Alternatively, we can introduce a distinct `task-interrupt` entry type (`PerformanceTaskInterruptTiming`) specifically for threshold-triggered stack capture:

```webidl
[Exposed=Window]
interface PerformanceTaskInterruptTiming : PerformanceEntry {
    // entryType will be "task-interrupt"
    // startTime and duration are inherited from PerformanceEntry
    
    // The captured stack trace at the exact moment the threshold was crossed.
    readonly attribute FrozenArray<DOMString> stackTrace;
};
```

* **Pros:** Clean separation of concerns from legacy `longtask` semantics; makes it explicit that entries represent mid-task threshold interrupts rather than post-mortem long tasks.
* **Cons:** Introduces a new `PerformanceEntry` subclass and entry type name.

### How this solution would solve the use cases

Developers can define a strict budget (e.g., 10ms) and automatically log exact stack traces to their telemetry systems when the budget is violated:

```javascript
// 1. Create a PerformanceObserver to listen for janky tasks with stack traces
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn(`Janky task detected. Total duration: ${entry.duration}ms`);
    
    // The API exposes the exact point-in-time stack trace captured when 
    // the threshold was crossed.
    if (entry.stackTrace?.length) {
      console.warn(`Stack trace captured at threshold:\n${entry.stackTrace.join('\n')}`);
    }
  }
});

// 2. Start observing tasks with a custom threshold and stack trace capture
// (Under Option A: type: 'longtask', takeTracing: true; under Option B: type: 'task-interrupt')
observer.observe({ 
  type: 'longtask',
  durationThreshold: 10,
  takeTracing: true,
});
```

When a custom budget (e.g., 10ms) is violated, the `stackTrace` is synchronously captured by the browser engine and surfaced in the entry. The developer's script can then read this entry and beacon the exact function name and line number back to their telemetry backend, eliminating guesswork in diagnosing production jank.

## Detailed design discussion

### Mid-execution interrupts vs. Post-mortem aggregation

A core design choice was whether to collect script attribution data continuously during the task (aggregation) or to trigger a single interrupt. We chose the interrupt-driven model because it provides a precise "point-in-time" snapshot of what was blocking the thread exactly when the threshold was breached, resulting in a clearer signal for developers compared to aggregated execution times.

### Handling multiple observers with different thresholds

Normally, `PerformanceObserver` options (such as `durationThreshold` in Event Timing) act purely as consumer-side filters on entries without changing how the underlying entry is produced. With mid-execution stack capture, however, `durationThreshold` also determines *when* during task execution the JavaScript engine is interrupted to sample `stackTrace`.

If a page has two active observers with different thresholds (for example, while migrating from one RUM provider using `durationThreshold: 20` to another using `durationThreshold: 50`), there are two possible models:

1. **Single interrupt per task at the minimum threshold:**
   The browser triggers at most one interrupt per task at the minimum `durationThreshold` across all active tracing observers (`20ms`), while still filtering entry delivery per observer so the `50ms` observer only receives tasks whose total duration is at least `50ms`. This bounds engine interrupt overhead to at most once per task, though the `50ms` observer receives a stack trace sampled at `20ms`.
2. **Per-threshold interrupts:**
   The browser schedules interrupts at each distinct active threshold (e.g., interrupting at `20ms` to capture the stack for Observer A, and—if the same task is still running at `50ms`—interrupting again to capture the `50ms` stack for Observer B). Because only a small number of distinct observers typically exist on a page at once, this isolates observers from one another so one observer's threshold does not alter another observer's captured stack trace.

### Lock-free task observation and idle suspension

Because the browser must observe every task on the main thread to measure its duration, introducing locks or heavy thread synchronization on every task would impact overall browser performance.
* **Lock-free timestamps on task boundaries:** The main thread records task start and end state using lock-free atomic operations.
* **Zero polling when idle:** When no task is running on the main thread, the background watchdog thread sleeps on an OS wait primitive and consumes 0% CPU. To avoid synchronization overhead on back-to-back tasks, the main thread checks a lightweight atomic sleep flag and only signals the watchdog thread on the first task after an idle period.

## Considered alternatives

### Long Animation Frames (LoAF) API

The Long Animation Frames API is a fantastic tool for measuring responsiveness, but it is fundamentally unsuited for this specific use case for two reasons:

1. Frames vs. Tasks: LoAF is tied to a 50ms frame rendering threshold. We need to measure individual tasks with a configurable, much lower threshold to catch thread-hogging before a frame is even dropped.
2. Aggregated Attribution vs. Point-in-Time: LoAF uses a post-mortem, aggregated script attribution model (e.g., "Function X took 40ms total over the course of the frame"). Our design relies on triggering a mid-execution interrupt to capture a precise stack trace.

### JS Self-Profiling API (`Profiler`)

During review, we considered whether this capability could be exposed as an extension to the [JS Self-Profiling API](https://wicg.github.io/js-self-profiling/), for example:

```javascript
const profiler = new Profiler({ taskInterruptDurationThreshold: 50 });
profiler.addEventListener('sample', (event) => { ... });
```

While this would avoid overloading `PerformanceObserver.observe()` options, `Profiler` is not a natural fit for several reasons:

1. **Event-driven delivery vs. start/stop lifecycle:** `Profiler` is built around recording a bounded profile session and only delivers data when `profiler.stop()` is called. For continuous jank monitoring, a page would either have to periodically stop and restart `Profiler` instances to inspect samples, or we would need to bolt a new streaming event API onto `Profiler`—which `PerformanceObserver` already provides natively.
2. **One-shot threshold interrupt vs. continuous periodic sampling:** `Profiler` is designed around continuous periodic sampling (`sampleInterval`) and building a full profile tree (`resources`, `frames`, `stacks`, `samples`). Here, we specifically do *not* want continuous sampling or profile generation when tasks stay within budget; implementing this inside `Profiler` would require a completely separate mode and code path that shares very little with `Profiler`.
3. **Always-on monitoring without `Document-Policy: js-profiling`:** `Profiler` requires pages to serve a `Document-Policy: js-profiling` HTTP header because high-frequency continuous CPU profiling carries higher overhead and cross-origin timing risks. Because Task Interrupt Timing only fires a one-shot interrupt when a task exceeds a threshold and truncates non-CORS cross-origin stack frames, it is designed to be lightweight and usable without requiring document-level policy headers.

### Global threshold configuration

Another alternative to per-observer `durationThreshold` is a global configuration method such as `performance.setInterruptDuration(ms)` so that `PerformanceObserver` remains purely a passive consumer of entries.

However, a global setting has poor ergonomics when multiple independent scripts (e.g., first-party diagnostics and a third-party RUM provider, or two RUM providers during a migration) coexist on the same page, because whichever script calls `setInterruptDuration()` last silently overwrites the threshold for everyone else. Keeping `durationThreshold` scoped to each observer (paired with per-threshold interrupt capture when distinct thresholds are registered) avoids global state collisions.

## Security and Privacy Considerations

Exposing raw JavaScript stack traces to the web platform introduces significant privacy and security risks. Specifically, a malicious site could embed a cross-origin resource (such as a third-party script) and use this API to observe its internal execution flow. By reading the function names in the stack trace, the attacker could infer sensitive user state (e.g., inferring authentication status based on which code paths are executing).

To mitigate this cross-origin data leakage while keeping the API usable on general websites (which often rely on ads and analytics), we propose stack truncation at cross-origin boundaries.

Instead of strictly requiring Cross-Origin Isolation (COOP/COEP) for the entire page, the API will filter out any non-CORS third-party frames from the stack trace, keeping only the direct entry point called by the first-party script.

For example:

1. `myFirstPartyFunction()`
2. `thirdPartyEntry()` ⬅ Boundary kept (Actionable for the developer)
3. `[Filtered non-CORS frames]` ⬅ Internal state hidden (Privacy maintained)

This surgical truncation ensures developers know exactly which external call blocked the thread, while completely hiding the internal execution state of that opaque script.

## Stakeholder Feedback / Opposition

- Chrome/WebUI : Positive (Driving initial incubation)
- Web Performance WG : Invited for discussion

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Fergal Daly
- Michal Mocny
- Yoav Weiss

## Appendix: Implementation Feasibility (Chromium PoC)

Demonstrating that continuous task monitoring and mid-execution interrupts can be achieved with near-zero overhead is critical to the viability of this proposal.

While the exact implementation details are left to each browser engine, we built a Proof of Concept (PoC) in Chromium to demonstrate that this interrupt-driven architecture is practically feasible.

The PoC utilizes a lock-free atomic timestamp on the main thread paired with a dedicated background watchdog thread. Our performance evaluation on a heavy workload (50,000+ micro-tasks and multiple >120ms synchronous tasks) yielded the following results:

* **Main-Thread Critical Path (< 50 ns):** 
  By using relaxed atomic stores (`std::memory_order_relaxed`) and reusing existing task scheduler timestamps, the observation overhead per task can be reduced to under 50 nanoseconds.
* **Watchdog Thread Overhead (~0.4% Duty Cycle When Active, 0% When Idle):** 
  When tasks are actively running, a background thread polling every 5ms takes approximately 20 µs per poll (~0.4% CPU utilization). When no tasks are running on the main thread, the watchdog blocks on a waitable event and consumes 0% CPU.
* **Engine Interrupt Cost (< 6% Relative Overhead):** 
  Capturing the in-flight JavaScript engine stack trace takes a median of ~0.31 ms. Because this penalty is strictly deferred until a task has *already* violated the performance budget (e.g., 5ms), it represents less than 6% relative overhead on the janky task itself. Well-behaving tasks incur exactly 0 ms of capture overhead.

These results prove that capturing point-in-time stack traces via a watchdog thread is practically feasible and safe for continuous, in-the-wild production monitoring.
