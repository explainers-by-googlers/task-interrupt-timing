# Explainer for Jankiness Measurement API (Extending LoAF Script Attribution)

This proposal is an early design sketch by [elkurin@chromium.org] to describe the problem below and solicit
feedback on the proposed solution. It has not been approved to ship in Chrome.


## Proponents

- Eriko Kurimoto (@elkurin)
- Fergal Daly (@fergald)

## Participate
- https://github.com/w3c/long-animation-frames/issues/47
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
  - [Predetermined script interrupt threshold, battery life, and side channels](#predetermined-script-interrupt-threshold-battery-life-and-side-channels)
  - [Lazy stack symbolization and slow-device guardrails](#lazy-stack-symbolization-and-slow-device-guardrails)
  - [Single capture per script across observers](#single-capture-per-script-across-observers)
  - [Lock-free task observation and zero-CPU idle suspension](#lock-free-task-observation-and-zero-cpu-idle-suspension)
- [Considered alternatives](#considered-alternatives)
  - [Extending legacy `PerformanceLongTaskTiming` (`longtask`)](#extending-legacy-performancelongtasktiming-longtask)
  - [Dedicated `task-interrupt` entry type](#dedicated-task-interrupt-entry-type)
  - [JS Self-Profiling API (`Profiler`)](#js-self-profiling-api-profiler)
  - [Coupling script interrupt threshold to observer `durationThreshold`](#coupling-script-interrupt-threshold-to-observer-durationthreshold)
- [Future Work: Configurable Sub-50ms `durationThreshold` for LoAF](#future-work-configurable-sub-50ms-durationthreshold-for-loaf)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)
- [Appendix: Implementation Feasibility (Chromium Prototype)](#appendix-implementation-feasibility-chromium-prototype)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Introduction

Modern web applications strive for smooth, responsive user interfaces. However, developers often struggle to identify and eliminate "jank"—stutters caused by JavaScript tasks hogging the main thread.

While the **Long Animation Frames (LoAF) API** (`long-animation-frame`) provides valuable visibility into long frames ($\ge 50\text{ms}$) and attributes scripts by their outer entry point (`invoker`, `sourceURL`, `sourceFunctionName`), a key diagnostic gap remains:

* **Leaf-level execution visibility (Wrapper / RUM blindspot):** Entry-point attribution only identifies the outer callback or event listener that started a script. When applications use framework schedulers, state batchers, or RUM/telemetry error wrappers (e.g., `Function.prototype.apply` wrappers), `PerformanceScriptTiming` attributes the entire duration to the outer wrapper rather than the inner leaf function that was actually burning CPU cycles when the budget was breached.

Following feedback from the W3C Web Performance Working Group at TPAC 2026, this proposal extends **Long Animation Frames (`long-animation-frame`)** with an opt-in `takeTracing` flag on `PerformanceObserverInit` and a `stackTrace` attribute on `PerformanceScriptTiming`. When a script exceeds a predetermined execution threshold (e.g., `7ms`), the browser triggers a lightweight mid-execution interrupt to capture a point-in-time call stack of the offending script.


## Goals

* Extend `PerformanceScriptTiming` (within `long-animation-frame`) to expose the point-in-time JavaScript call stack (`stackTrace`) captured when a script exceeds a predetermined execution threshold.
* Allow `PerformanceObserver` instances observing `"long-animation-frame"` to opt into stack capture via `takeTracing: true`.
* Ensure the detection and observation mechanisms have zero main-thread overhead on healthy tasks and minimal one-time overhead when a threshold breach occurs.


## Non-goals

* **Replacing DevTools Profilers:** This API is for programmatic, lightweight threshold-breach monitoring in the wild, not for capturing full continuous CPU flamegraphs across healthy execution.
* **Reviving legacy `PerformanceLongTaskTiming`:** Rather than adding new capabilities to legacy `"longtask"` entries, this proposal builds on the modern Long Animation Frames (`long-animation-frame`) standard.
* **Changing LoAF's overall frame `durationThreshold` in this feature:** Allowing `"long-animation-frame"` observers to configure a sub-50ms frame `durationThreshold` is an orthogonal capability and is tracked separately from `takeTracing` and `PerformanceScriptTiming.stackTrace`.


## Use cases

This proposal is driven by both the performance requirements of the **Webium Product project** (Chrome's WebUI architecture) and real-world **Real User Monitoring (RUM)** diagnostic needs across complex web applications.


### Use case 1: Diagnosing micro-stutters in the Webium Product

In the Webium Product, synchronous scripts (e.g., `10ms–20ms` scripts contributing to a long animation frame) can cause noticeable stutters during critical rendering or interaction phases. Engineers need to understand exactly which inner JavaScript function is blocking the thread at the moment a script exceeds its budget.

Entry-point attribution alone (`TimerHandler:setTimeout` or an outer Lit/Mojo dispatcher) is insufficient when hundreds of different handlers funnel through a shared dispatcher. By capturing a point-in-time `stackTrace` on `PerformanceScriptTiming` when the script execution threshold is exceeded, engineers can pinpoint and eliminate the exact source of main-thread contention.

### Use case 2: Real User Monitoring (RUM) and Wrapper-Resilient Attribution

In production web applications (particularly Single-Page Applications and Soft Navigations), RUM providers and site owners frequently see long scripts in LoAF whose `sourceURL` and `sourceFunctionName` point to a generic third-party wrapper, router hook, or telemetry SDK wrapper rather than the application logic underneath. Furthermore, engineers currently have to manually reproduce and marry RUM long-task waterfalls with throttled local CPU profiles to guess which inner function caused P90+ delays in the wild.

Exposing `stackTrace` directly on `PerformanceScriptTiming` allows RUM scripts to see both the **initiator entry point** and the **inner first-party leaf call stack**, even when outer entry points are wrapped.


## Proposed Solution: Extending Long Animation Frames (LoAF)

When a `"long-animation-frame"` observer opts in via `takeTracing: true`, a background watchdog tracks main-thread script execution duration. If a script continues past a single predetermined script interrupt threshold (e.g., `7ms` in the Chromium prototype, slightly above LoAF's `5ms` long-script threshold and common scheduler yield quantums), the browser requests a one-shot interrupt from the JavaScript engine. At the next safe point, the engine captures the current call stack, applies LoAF's opaque-script filtering rules, and attaches the resulting `stackTrace` to the active `PerformanceScriptTiming` entry.

### WebIDL

```webidl
partial dictionary PerformanceObserverInit {
    boolean takeTracing = false;
};

dictionary PerformanceStackFrame {
    DOMString functionName = "";
    DOMString sourceURL = "";
    long long sourceLine = -1;
    long long sourceColumn = -1;
};

partial interface PerformanceScriptTiming {
    // The point-in-time call stack captured when the threshold was crossed,
    // ordered top-down from the interrupted leaf frame to the outer entry point.
    // Populated when observed with takeTracing: true (empty array otherwise).
    readonly attribute FrozenArray<PerformanceStackFrame> stackTrace;
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
        const formattedFrames = script.stackTrace
          .map(f => `${f.functionName || '<anonymous>'} (${f.sourceURL}:${f.sourceLine}:${f.sourceColumn})`)
          .join('\n    ');
        console.warn(
          `Janky script (${script.invoker}, ${script.duration}ms) in frame (${frame.duration}ms):\n` +
          `  Entry point: ${script.sourceFunctionName} (${script.sourceURL}:${script.sourceCharPosition})\n` +
          `  Stack trace at threshold:\n    ${formattedFrames}`
        );
      }
    }
  }
});

observer.observe({
  type: 'long-animation-frame',
  takeTracing: true,
});
```


## Detailed design discussion

### Combining entry-point attribution with mid-execution interrupts

LoAF's existing `PerformanceScriptTiming` records post-mortem entry-point attribution ("Callback X ran for 65ms total"), whereas a threshold interrupt captures a point-in-time snapshot of the call stack at the moment the budget is breached. Combining both on `PerformanceScriptTiming` avoids forcing developers to choose between knowing *what initiated the work* (`invoker` / `sourceURL`) and *what inner function was hogging the CPU* (`stackTrace`).

### Predetermined script interrupt threshold, battery life, and side channels

LoAF already records scripts $\ge 5\text{ms}$, and many userland schedulers treat `5ms` as their standard yield quantum. Using a single predetermined script interrupt threshold slightly above that boundary (e.g., `6ms–10ms`; `7ms` in the Chromium prototype) avoids interrupting tasks that are just about to yield while ensuring that any long script inside a LoAF frame (even a `15ms` script inside a `50ms` frame) receives a stack trace.

On Windows, the default OS system tick interval is `15.625ms` (64 Hz), while modern Windows 10 (1803+) provides [`CREATE_WAITABLE_TIMER_HIGH_RESOLUTION`](https://learn.microsoft.com/en-us/windows/win32/api/synchapi/nf-synchapi-createwaitabletimerexw) for short waitable timer delays without globally raising the system tick via `timeBeginPeriod`. Keeping the script interrupt interval UA-defined (rather than exposing arbitrary sub-millisecond timer intervals to script) prevents battery regressions and high-frequency sampling side channels.

### Lazy stack symbolization and slow-device guardrails

To minimize in-band pause time when an interrupt fires—especially on slower devices:
1. **Lazy stack symbolization:** During the JS engine safe-point interrupt, the browser only records lightweight frame references (or `(script_id, bytecode_offset)` pairs) without URL normalization, opaque-script filtering, or string formatting, deferring symbolization and formatting until after the script finishes. (If a script is garbage-collected before lazy symbolization occurs, affected frames degrade gracefully.)
2. **Slow-device guardrails (breach-storm rate limiting):** On low-end devices experiencing continuous back-to-back threshold breaches, the user agent can apply a token-bucket cooldown so excess breaches continue reporting LoAF timing metrics while temporarily skipping stack capture interrupts.

### Single capture per script across observers

Because the script interrupt fires at a **single predetermined script duration** whenever tracing is active, the browser captures **at most one stack trace per script** and shares it across all `takeTracing: true` observers. This ensures both that shorter scripts (`10ms–20ms`) inside a `50ms` frame still get stack traces and that multiple observers cannot be used to trigger repeated interrupts within a single script execution.

### Lock-free task observation and zero-CPU idle suspension

Because the browser tracks every task/script on the main thread, heavy thread synchronization on every task boundary would regress performance:
* **Fast-path main-thread bookkeeping:** Script start and end state is updated with minimal main-thread overhead.
* **Zero polling when idle (`CondVar::Wait` / `TimedWait`):** Rather than polling continuously, the background watchdog thread parks on a condition variable (`cv.Wait()`) when the main thread is idle (consuming `0%` CPU) and sleeps for the exact remaining budget (`cv.TimedWait(threshold - elapsed)`) when a script is running, waking the JS engine at most once per breached script threshold.


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

### Coupling script interrupt threshold to observer `durationThreshold`

We initially considered using `PerformanceObserverInit.durationThreshold` both to lower LoAF's `50ms` frame threshold and to set the script interrupt timer. However, a `50ms` LoAF frame frequently consists of multiple shorter scripts (e.g., three `18ms` scripts), so tying the script interrupt timer to `50ms` would miss all three scripts under the default threshold, while allowing multiple observers with staggered thresholds (`10, 11, 12ms`) could be abused as a sampling profiler. Using a single predetermined per-script interrupt threshold (`7ms`) and keeping configurable LoAF frame `durationThreshold` as a separate, orthogonal feature avoids both issues.


## Future Work: Configurable Sub-50ms `durationThreshold` for LoAF

While `takeTracing: true` with a predetermined per-script interrupt threshold (`7ms`) already captures call stacks for $\ge 7\text{ms}$ scripts inside standard $\ge 50\text{ms}$ Long Animation Frames, LoAF currently discards animation frames whose *total* duration is under `50ms`.

As a separate follow-up feature, we plan to explore honoring `PerformanceObserverInit.durationThreshold` on `"long-animation-frame"` observers (subject to a UA minimum clamp such as `10ms–15ms` for OS timer and battery constraints, with a local developer override for lab profiling):

```javascript
observer.observe({
  type: 'long-animation-frame',
  durationThreshold: 15, // Future follow-up: observe >= 15ms animation frames
  takeTracing: true,
});
```

This companion capability is valuable both independently (for developers enforcing 60Hz/120Hz frame smoothness budgets below `50ms` using standard LoAF attribution) and together with `takeTracing` (so an isolated `15ms–30ms` janky script is reported even when the overall animation frame completes in under `50ms`).


## Security and Privacy Considerations

Exposing JavaScript stack traces must preserve Long Animation Frames' existing [opaque script mitigations](https://w3c.github.io/long-animation-frames/#loaf-opaque-scripts-sec) ([exposed source URL](https://w3c.github.io/long-animation-frames/#script-exposed-source-url) and [apply source location](https://w3c.github.io/long-animation-frames/#apply-source-location)) for classic scripts whose **muted errors** flag is true (`no-cors` scripts and code dynamically evaluated inside them):

* **Keyed on `muted errors` & pre-redirect URL:** Frame opacity is determined by the script's **muted errors** flag (not URL origin comparison), and frame URLs use the script's pre-redirect [exposed source URL](https://w3c.github.io/long-animation-frames/#script-exposed-source-url) (ignoring `//# sourceURL=` comments on muted scripts and collapsing `data:` URLs to `"data:"`).
* **No capture for muted script entries or opaque innermost frames:** `stackTrace` is `[]` when the `PerformanceScriptTiming` entry itself is a muted classic script (preserving `executionStart = startTime`) or when the innermost frame at the interrupt has `muted errors` set.
* **Opaque boundary truncation & single capture per script:** When non-opaque code is called through an opaque frame lower on the stack, `stackTrace` stops at that boundary frame and exposes only its pre-redirect `sourceURL` (with `functionName = ""` and `sourceLine = sourceColumn = -1`), and at most one stack is captured per script across all observers:

1. `{ functionName: "myFirstPartyCallback", sourceURL: "https://app.example/main.js", sourceLine: 42, sourceColumn: 5 }` ⬅ Non-opaque frame kept in full
2. `{ functionName: "", sourceURL: "https://cdn.other.example/sdk.js", sourceLine: -1, sourceColumn: -1 }` ⬅ Opaque boundary keeps only exposed `sourceURL`
3. `[Caller frames below opaque boundary omitted]`


## Stakeholder Feedback / Opposition

- **Chrome / WebUI (Webium):** Positive (Driving initial incubation and prototype)
- **W3C Web Performance WG (TPAC 2026):** Discussed on Oct 29, 2026; consensus to prototype extending **Long Animation Frames (LoAF)** rather than legacy `longtask` or a standalone entry type, using a predetermined script interrupt threshold (`6ms–10ms`) and lazy stack symbolization.

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

We built and verified an end-to-end Chromium prototype extending `TaskInterruptMonitor` and LoAF's `PerformanceScriptTiming.stackTrace` on `"long-animation-frame"` observers ([CL 8683720](https://chromium-review.googlesource.com/c/chromium/src/+/8683720), stacked on [CL 8350623](https://chromium-review.googlesource.com/c/chromium/src/+/8350623)):

* **Healthy Scripts (`< 7ms` — `0.00 ms` Overhead):**
  Scripts that complete within the `7ms` interrupt threshold trigger zero V8 interrupts and incur `0.00 ms` (`0%`) main-thread overhead.
* **Watchdog Thread (`0%` CPU When Idle):**
  The watchdog thread blocks on a condition variable (`cv.Wait()`) when no main-thread script is running (`0%` CPU) and uses `cv.TimedWait()` for the remaining threshold duration when a script starts.
* **V8 Safe-Point Capture Cost (`55 µs–93 µs` with Lazy Symbolization — 52%–64% Reduction):**
  Deferring opaque-frame truncation, URL filtering, and `StringBuilder` string formatting out of `TaskInterruptMonitor::OnInterrupt()` until script/task completion cuts the in-band V8 safe-point pause by **52%–64%** across call stack depths in Release builds (`is_debug=false`, `dcheck_always_on=false`):

| Call Stack Depth | Eager Safe-Point `P50 (median)` (95% CI) | Lazy Safe-Point `P50 (median)` (95% CI) | Safe-Point `P50 (median)` Reduction | Eager `P90` $\to$ Lazy `P90` | Deferred Post-Script Formatting `P50 (median)` |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **5 frames** | `114.0 µs` (`[110.0, 119.5]`) | **`55.0 µs`** (`[54.0, 58.0]`) | **`-59.0 µs` (`-51.8%`)** | `156.2 µs` $\to$ `82.1 µs` (`-47.4%`) | `53.0 µs` (`[52.0, 54.0]`) |
| **15 frames** | `154.5 µs` (`[152.0, 161.0]`) | **`64.0 µs`** (`[62.0, 69.0]`) | **`-90.5 µs` (`-58.6%`)** | `238.3 µs` $\to$ `107.0 µs` (`-55.1%`) | `78.0 µs` (`[77.0, 78.5]`) |
| **40 frames** | `258.0 µs` (`[252.0, 263.5]`) | **`93.0 µs`** (`[90.0, 96.0]`) | **`-165.0 µs` (`-64.0%`)** | `353.1 µs` $\to$ `147.0 µs` (`-58.4%`) | `131.0 µs` (`[129.0, 134.0]`) |

*(Values above report thread CPU duration `thread_dur` from 160–200 Perfetto `TaskInterruptCapture` / `TaskInterruptFormat` slices per variant across 5 interleaved Release A/B browser runs per depth with mixed same-origin and cross-origin script URLs; wall-clock `dur` reductions are `-50.2%`, `-58.3%`, and `-63.4%` respectively.)*
