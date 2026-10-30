# Explainer for Jankiness Measurement API (Extending LoAF Script Attribution)

This proposal is an early design sketch by [elkurin@chromium.org] to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.


## Proponents

- Eriko Kurimoto (@elkurin)
- Fergal Daly (@fergald)

## Participate
- https://github.com/explainers-by-googlers/task-interrupt-timing/issues
- https://github.com/WICG/proposals/issues/312

## Table of Contents

<!-- Update this table of contents by running `npx doctoc README.md` -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Introduction](#introduction)
- [Goals](#goals)
- [Non-goals](#non-goals)
- [Use cases](#use-cases)
  - [Use case 1: Diagnosing micro-stutters in the Webium Product](#use-case-1-diagnosing-micro-stutters-in-the-webium-product)
  - [Use case 2: Real User Monitoring (RUM) and Wrapper-Resilient Attribution](#use-case-2-real-user-monitoring-rum-and-wrapper-resilient-attribution)
- [Proposed Solution: Extending Long Animation Frames (LoAF)](#proposed-solution-extending-long-animation-frames-loaf)
  - [WebIDL](#webidl)
  - [Why Extending LoAF `PerformanceScriptTiming` Fits Best](#why-extending-loaf-performancescripttiming-fits-best)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
- [Detailed design discussion](#detailed-design-discussion)
  - [Combining entry-point attribution with mid-execution interrupts](#combining-entry-point-attribution-with-mid-execution-interrupts)
  - [OS timer resolution clamping (10ms–15ms), battery life, and side channels](#os-timer-resolution-clamping-10ms15ms-battery-life-and-side-channels)
  - [Lazy stack symbolization and slow-device guardrails](#lazy-stack-symbolization-and-slow-device-guardrails)
  - [Handling multiple observers with different thresholds](#handling-multiple-observers-with-different-thresholds)
  - [Lock-free task observation and zero-CPU idle suspension](#lock-free-task-observation-and-zero-cpu-idle-suspension)
- [Considered alternatives](#considered-alternatives)
  - [Extending legacy `PerformanceLongTaskTiming` (`longtask`)](#extending-legacy-performancelongtasktiming-longtask)
  - [Dedicated `task-interrupt` entry type](#dedicated-task-interrupt-entry-type)
  - [JS Self-Profiling API (`Profiler`)](#js-self-profiling-api-profiler)
  - [Global threshold configuration](#global-threshold-configuration)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)
- [Appendix: Implementation Feasibility (Chromium Prototype)](#appendix-implementation-feasibility-chromium-prototype)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Modern web applications strive for smooth, responsive user interfaces. However, developers often struggle to identify and eliminate "jank"—stutters caused by JavaScript tasks hogging the main thread.

While the **Long Animation Frames (LoAF) API** (`long-animation-frame`) provides valuable visibility into long frames (>50ms) and attributes scripts by their outer entry point (`invoker`, `sourceURL`, `sourceFunctionName`), two key diagnostic gaps remain:

1. **Leaf-level execution visibility (Wrapper / RUM blindspot):** Entry-point attribution only identifies the outer callback or event listener that started a script. When applications use framework schedulers, state batchers, or RUM/telemetry error wrappers (e.g., `Function.prototype.apply` wrappers), `PerformanceScriptTiming` attributes the entire duration to the outer wrapper rather than the inner leaf function that was actually burning CPU cycles when the budget was breached.
2. **Sub-50ms smoothness budgets:** Developers enforcing 60Hz/120Hz frame budgets or diagnosing main-thread contention prior to a full 50ms frame drop lack a way to lower the observation threshold (e.g., down to 10ms–15ms).

Following feedback from the W3C Web Performance Working Group at TPAC 2026, this proposal extends **Long Animation Frames (`long-animation-frame`)** with an opt-in `takeTracing` flag, configurable `durationThreshold`, and a `stackTrace` attribute on `PerformanceScriptTiming`. When a script exceeds the configured threshold, the browser triggers a lightweight mid-execution interrupt to capture a point-in-time call stack of the offending script.


## Goals

* Extend `PerformanceScriptTiming` (within `long-animation-frame`) to expose the point-in-time JavaScript call stack (`stackTrace`) captured when a script exceeds its execution threshold.
* Allow `PerformanceObserver` instances observing `"long-animation-frame"` to configure a custom `durationThreshold` (subject to a UA minimum clamp for OS timer/power constraints) and opt into stack capture via `takeTracing: true`.
* Ensure the detection and observation mechanisms have zero main-thread overhead on healthy tasks and minimal one-time overhead when a threshold breach occurs.


## Non-goals

* **Replacing DevTools Profilers:** This API is for programmatic, lightweight threshold-breach monitoring in the wild, not for capturing full continuous CPU flamegraphs across healthy execution.
* **Reviving legacy `PerformanceLongTaskTiming`:** Rather than adding new capabilities to legacy `"longtask"` entries, this proposal builds on the modern Long Animation Frames (`long-animation-frame`) standard.


## Use cases

This proposal is driven by both the performance requirements of the **Webium Product project** (Chrome's WebUI architecture) and real-world **Real User Monitoring (RUM)** diagnostic needs across complex web applications.


### Use case 1: Diagnosing micro-stutters in the Webium Product

In the Webium Product, even moderate synchronous scripts (e.g., 10ms–20ms) can cause noticeable stutters if they occur during critical rendering or interaction phases. The project needs to enforce strict sub-50ms budgets and understand exactly which inner JavaScript function is blocking the thread at the moment the budget is breached.

Entry-point attribution alone (`TimerHandler:setTimeout` or an outer Lit/Mojo dispatcher) is insufficient when hundreds of different handlers funnel through a shared dispatcher. By capturing a point-in-time `stackTrace` on `PerformanceScriptTiming` when the threshold is exceeded, engineers can pinpoint and eliminate the exact source of main-thread contention.

### Use case 2: Real User Monitoring (RUM) and Wrapper-Resilient Attribution

In production web applications (particularly Single-Page Applications and Soft Navigations), RUM providers and site owners frequently see long scripts in LoAF whose `sourceURL` and `sourceFunctionName` point to a generic third-party wrapper, router hook, or telemetry SDK wrapper rather than the application logic underneath. Furthermore, engineers currently have to manually reproduce and marry RUM long-task waterfalls with throttled local CPU profiles to guess which inner function caused P90+ delays in the wild.

Exposing `stackTrace` directly on `PerformanceScriptTiming` allows RUM scripts to see both the **initiator entry point** and the **inner first-party leaf call stack**, even when outer entry points are wrapped.


## Proposed Solution: Extending Long Animation Frames (LoAF)

When a `"long-animation-frame"` observer opts in via `takeTracing: true` (and optionally specifies a `durationThreshold`), a background watchdog tracks main-thread execution duration. If execution continues past `durationThreshold`, the browser requests a one-shot interrupt from the JavaScript engine (`v8::Isolate::RequestInterrupt`). At the next V8 safe point, the engine captures the current call stack, truncates any non-CORS cross-origin frames, and attaches the resulting `stackTrace` to the active `PerformanceScriptTiming` entry.

### WebIDL

```webidl
partial dictionary PerformanceObserverInit {
    DOMHighResTimeStamp durationThreshold;
    boolean takeTracing = false;
};

partial interface PerformanceScriptTiming {
    // The point-in-time call stack captured when the threshold was crossed,
    // ordered top-down from the interrupted leaf frame to the outer entry point.
    // Populated when observed with takeTracing: true (empty array otherwise).
    readonly attribute FrozenArray<DOMString> stackTrace;
};
```

### Why Extending LoAF `PerformanceScriptTiming` Fits Best

1. **Unified Entry-Point + Leaf Call Stack Attribution:** Developers get LoAF's existing entry-point metadata (`invoker`, `invokerType`, `sourceURL`, `sourceFunctionName`, `sourceCharPosition`, `forcedStyleAndLayoutDuration`) alongside the mid-execution `stackTrace` on the exact same `PerformanceScriptTiming` object.
2. **Solves the Wrapper Attribution Problem:** When an entry point is wrapped by a framework or error-monitoring library, `sourceFunctionName` reflects the outer wrapper, while `stackTrace[0]` reveals the inner leaf function where execution was actually spending time when the threshold was crossed.
3. **Natural Correlation with Rendering & Input (`EventTiming`):** Because LoAF already groups scripts with rendering phases (`renderStart`, `styleAndLayoutStart`) and overlaps with `EventTiming` (INP) interactions, attaching `stackTrace` to `PerformanceScriptTiming` immediately enriches slow-interaction and slow-frame diagnostics without requiring a separate timeline entry type.

### How this solution would solve the use cases

Developers can observe `"long-animation-frame"` with `takeTracing: true` and inspect `script.stackTrace` on each long script in the frame:

```javascript
const observer = new PerformanceObserver((list) => {
  for (const frame of list.getEntries()) {
    for (const script of frame.scripts) {
      if (script.stackTrace?.length) {
        console.warn(
          `Janky script (${script.invoker}, ${script.duration}ms) in frame (${frame.duration}ms):\n` +
          `  Entry point: ${script.sourceFunctionName} (${script.sourceURL}:${script.sourceCharPosition})\n` +
          `  Stack trace at threshold:\n    ${script.stackTrace.join('\n    ')}`
        );
      }
    }
  }
});

observer.observe({
  type: 'long-animation-frame',
  durationThreshold: 15,
  takeTracing: true,
});
```


## Detailed design discussion

### Combining entry-point attribution with mid-execution interrupts

LoAF's existing `PerformanceScriptTiming` records post-mortem entry-point attribution ("Callback X ran for 65ms total"), whereas a threshold interrupt captures a point-in-time snapshot of the call stack at the moment the budget is breached. Combining both on `PerformanceScriptTiming` avoids forcing developers to choose between knowing *what initiated the work* (`invoker` / `sourceURL`) and *what inner function was hogging the CPU* (`stackTrace`).

### OS timer resolution clamping (10ms–15ms), battery life, and side channels

On Windows, the default OS timer tick interval is `15.625ms` (64 Hz). Supporting sub-10ms watchdog wakeups on Windows would require switching the OS into a high-resolution timer mode (`timeBeginPeriod`), which prevents deep CPU sleep states and degrades battery life. Furthermore, while browsers already enter high-resolution timer mode under specific conditions (such as when running on AC power or playing media), dynamically varying the minimum allowed `durationThreshold` based on whether high-resolution timers are currently active would create a privacy side channel leaking power/timer state.

To avoid battery regressions and side-channel leaks:
* **Production minimum clamp (`10ms–15ms`):** User agents clamp `durationThreshold` to a fixed minimum (e.g., `10ms` or `15ms`) in normal production browsing, which still captures the vast majority of actionable jank—especially when aggregated across real user sessions.
* **Developer mode / local debugging override:** For local lab profiling or controlled internal environments (such as Webium development builds), browsers may offer a local developer flag or Permissions Policy override to allow lower thresholds without affecting general web battery life.

### Lazy stack symbolization and slow-device guardrails

In our initial Chromium prototype, capturing and eagerly symbolizing a V8 stack trace at the interrupt safe point takes a median of `~0.31ms` (`0.00ms` for healthy scripts below the threshold). At a `50ms` threshold this is `~0.6%`, and at a `10ms–15ms` threshold it is `~2–3%`.

To minimize overhead further—especially on slower devices:
1. **Lazy stack symbolization:** During the V8 safe-point interrupt (`OnInterrupt`), the engine only needs to capture lightweight raw stack frame references (script ID, function handle, bytecode offset / line+column), deferring string formatting and URL origin parsing until the `PerformanceScriptTiming` entry is finalized or read. (If a script is garbage-collected before lazy symbolization occurs, affected frames degrade gracefully.)
2. **Slow-device guardrails (breach-storm rate limiting):** On low-end devices experiencing continuous back-to-back threshold breaches, the user agent can apply a token-bucket cooldown so excess breaches continue reporting LoAF timing metrics while temporarily skipping V8 stack capture interrupts.

### Handling multiple observers with different thresholds

When multiple `"long-animation-frame"` observers are registered on the same page with different `durationThreshold` values (e.g., Observer A at `15ms` and Observer B at `50ms`), each observer receives notifications according to its own configured threshold, while the page-level watchdog schedules interrupts at the active thresholds (or samples at the minimum active threshold) so observers do not clobber one another's configuration.

### Lock-free task observation and zero-CPU idle suspension

Because the browser tracks every task/script on the main thread, heavy thread synchronization on every task boundary would regress performance:
* **Fast-path main-thread bookkeeping:** Task start and end state is updated with minimal main-thread overhead.
* **Zero polling when idle (`CondVar::Wait` / `TimedWait`):** Rather than polling continuously, the background watchdog thread parks on a condition variable (`cv.Wait()`) when the main thread is idle (consuming `0%` CPU) and sleeps for the exact remaining budget (`cv.TimedWait(threshold - elapsed)`) when a task is running, waking the JS engine at most once per breached threshold.


## Considered alternatives

### Extending legacy `PerformanceLongTaskTiming` (`longtask`)

We initially prototyped adding `takeTracing` and `stackTrace` to `PerformanceLongTaskTiming` (`type: "longtask"`):

```webidl
partial interface PerformanceLongTaskTiming {
    readonly attribute FrozenArray<DOMString> stackTrace;
};
```

However, as discussed at TPAC 2026, `PerformanceLongTaskTiming` is a legacy API that has been superseded by Long Animation Frames (LoAF) and lacks script-level attribution (`invoker`, `sourceURL`, `forcedStyleAndLayoutDuration`). Extending LoAF's `PerformanceScriptTiming` instead provides a much richer, modern standard foundation.

### Dedicated `task-interrupt` entry type

We also considered a standalone `PerformanceTaskInterruptTiming` (`entryType: "task-interrupt"`) interface. The Web Performance Working Group preferred integrating stack traces into LoAF rather than introducing a separate overlapping entry type on the Performance Timeline.

### JS Self-Profiling API (`Profiler`)

We also evaluated whether threshold-triggered stack capture should be part of the [JS Self-Profiling API](https://wicg.github.io/js-self-profiling/). While Chromium could potentially share internal stack-sampling infrastructure between LoAF and `Profiler` (especially for sub-5ms profiling use cases), relying solely on `Profiler` has significant drawbacks:
1. **Cross-browser adoption:** Other browser vendors (such as Mozilla) have indicated they do not plan to implement the JS Self-Profiling API, whereas LoAF is the standards-track path for main-thread responsiveness.
2. **Lifecycle & policy header:** `Profiler` requires an explicit start/stop buffer lifecycle and a `Document-Policy: js-profiling` response header, whereas LoAF streams via `PerformanceObserver` and protects cross-origin scripts via stack frame truncation without requiring document-level headers.

### Global threshold configuration

A global setter such as `performance.setInterruptDuration(ms)` was rejected because multiple independent scripts on a page (e.g., first-party telemetry and a third-party RUM provider) would overwrite each other's global threshold. Scoping `durationThreshold` to `PerformanceObserver.observe()` avoids global state collisions.


## Security and Privacy Considerations

Exposing raw JavaScript stack traces to the web platform introduces cross-origin privacy and security risks if a page embeds a cross-origin script and inspects its internal call stack to infer user state.

To mitigate cross-origin leakage without requiring full Cross-Origin Isolation (`COOP`/`COEP`), the API enforces **cross-origin stack truncation** (consistent with CORS / same-origin attribution rules):

* All frames that are same-origin with the document (or loaded via CORS) are included in `stackTrace`.
* When a same-origin frame calls into a non-CORS cross-origin script, only the first boundary entry frame is retained (so developers can see which external call was invoked), and all internal contiguous cross-origin frames are omitted:

1. `myFirstPartyFunction (https://app.example/main.js:42:5)`
2. `thirdPartyEntry (https://cdn.other.example/sdk.js:10:1)` ⬅ Boundary kept
3. `[Internal non-CORS cross-origin frames omitted]` ⬅ Internal state hidden


## Stakeholder Feedback / Opposition

- **Chrome / WebUI (Webium):** Positive (Driving initial incubation and prototype)
- **W3C Web Performance WG (TPAC 2026):** Discussed on Oct 29, 2026; consensus to prototype extending **Long Animation Frames (LoAF)** rather than legacy `longtask` or a standalone entry type, with a `10ms–15ms` OS timer clamp and exploration of lazy stack symbolization.

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Bas Schouten (Mozilla)
- Barry Pollard (Google)
- Fergal Daly (Google)
- Matt Kubej (Shopify)
- Michal Mocny (Google)
- Nazım Can Altınova (Mozilla)
- Nic Jansma (Cloudflare)
- Noam Helfman (Microsoft)
- Noam Rosenthal (Google)
- Yoav Weiss (Shopify)

## Appendix: Implementation Feasibility (Chromium Prototype)

We built and verified an end-to-end Chromium prototype extending both `TaskInterruptMonitor` and LoAF's `PerformanceScriptTiming.stackTrace` ([CL 8683720](https://chromium-review.googlesource.com/c/chromium/src/+/8683720), stacked on [CL 8350623](https://chromium-review.googlesource.com/c/chromium/src/+/8350623)):

* **Healthy Tasks (`< durationThreshold` — `0.00 ms` Overhead):**
  Tasks that complete within the threshold trigger zero V8 interrupts and incur `0.00 ms` (`0%`) main-thread overhead.
* **Watchdog Thread (`0%` CPU When Idle):**
  The watchdog thread blocks on a condition variable (`cv.Wait()`) when no main-thread task is running (`0%` CPU) and uses `cv.TimedWait()` for the remaining threshold duration when a task starts.
* **Breach Capture Cost (`~0.31 ms` Eager, Further Reducible via Lazy Symbolization):**
  When a script breaches `durationThreshold`, capturing and eagerly formatting the V8 stack trace at the safe point takes a median of `~0.31 ms` (`P90: 0.42 ms`), which `AnimationFrameTimingMonitor` attaches directly to `ScriptTimingInfo` / `PerformanceScriptTiming.stackTrace`. Deferring string symbolization out of the safe point is expected to reduce the in-band interrupt pause even further.
