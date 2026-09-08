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
- [User research](#user-research)
- [Use cases](#use-cases)
  - [Use case 1](#use-case-1)
  - [Use case 2](#use-case-2)
- [[Potential Solution]](#potential-solution)
  - [How this solution would solve the use cases](#how-this-solution-would-solve-the-use-cases)
    - [Use case 1](#use-case-1-1)
    - [Use case 2](#use-case-2-1)
- [Detailed design discussion](#detailed-design-discussion)
  - [[Tricky design choice #1]](#tricky-design-choice-1)
  - [[Tricky design choice 2]](#tricky-design-choice-2)
- [Considered alternatives](#considered-alternatives)
  - [[Alternative 1]](#alternative-1)
  - [[Alternative 2]](#alternative-2)
- [Security and Privacy Considerations](#security-and-privacy-considerations)
- [Stakeholder Feedback / Opposition](#stakeholder-feedback--opposition)
- [References & acknowledgements](#references--acknowledgements)

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


## User research

[If any user research has been conducted to inform your design choices,
discuss the process and findings. User research should be more common than it is.]

## Use cases

This proposal is primarily driven by the performance requirements of the **Webium Product project**, which needs strict, programmatic control over main-thread responsiveness to deliver a stutter-free user experience. Webium's requirements highlight a broader challenge faced by complex, highly interactive web applications.


### Use case 1: Diagnosing micro-stutters in the Webium Product

In the Webium Product, even small tasks (e.g., 5-15ms) can cause noticeable micro-stutters if they occur during critical rendering or interaction phases. The project needs to enforce strict sub-50ms performance budgets and understand exactly which JavaScript function is blocking the thread at the moment the budget is breached. 

Current post-mortem tools only provide aggregated script attribution after a long frame finishes, which is insufficient for Webium's diagnostic needs. By capturing a precise, point-in-time stack trace exactly when a short threshold is exceeded, the Webium team (and developers of similar complex apps) can pinpoint and eliminate the exact source of main-thread contention in the wild.

### Use case 2

<!-- In your initial explainer, you shouldn't be attached or appear attached to any of the potential
solutions you describe below this. -->

## [Potential Solution]

We propose introducing a new `PerformanceObserver` entry type: `task-interrupt`. 

Developers can configure the observer with a custom `durationThreshold`. When a JavaScript task begins on the main thread, a background monitor starts a timer. If the task continues executing past the `durationThreshold`, the browser requests an immediate interrupt from the JavaScript engine (e.g., V8). 

At the next safe execution point, the engine captures the current execution stack trace. Once the task finally completes, a `PerformanceTaskInterruptTiming` entry is dispatched to the observer.

```webidl
[Exposed=Window]
interface PerformanceTaskInterruptTiming : PerformanceEntry {
    // entryType will be "task-interrupt"
    // startTime and duration are inherited from PerformanceEntry
    
    // The captured stack trace at the exact moment the threshold was crossed.
    readonly attribute FrozenArray<DOMString> stackTrace;
};
```

### How this solution would solve the use cases

Developers can define a strict budget (e.g., 10ms) and automatically log exact stack traces to their telemetry systems when the budget is violated.

```webidl
// 1. Create a PerformanceObserver to listen for task interrupts
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.warn(`Janky task detected. Total duration: ${entry.duration}ms`);
    
    // The API exposes the exact point-in-time stack trace captured when 
    // the threshold was crossed.
    if (entry.stackTrace) {
      console.warn(`Stack trace captured at threshold:\n${entry.stackTrace.join('\n')}`);
    }
  }
});

// 2. Start observing tasks with a custom threshold
observer.observe({ 
  type: 'task-interrupt', 
  durationThreshold: 10 
});
```

#### Use case 1

[Description of the end-user scenario]

```js
// Sample code demonstrating how to use these APIs to address that scenario.
```

#### Use case 2

[etc.]

## Detailed design discussion

### Mid-execution interrupts vs. Post-mortem aggregation

A core design choice was whether to collect script attribution data continuously during the task (aggregation) or to trigger a single interrupt. We chose the interrupt-driven model because it provides a precise "point-in-time" snapshot of what was blocking the thread exactly when the threshold was breached, resulting in a clearer signal for developers compared to aggregated execution times.

### Lock-free task observation

Because the browser must observe every single task on the main thread to measure its duration, introducing locks or thread-synchronization to communicate with a background watchdog thread would severely impact overall browser performance. This design relies on a lock-free, relaxed atomic write on the main thread that the background thread polls, ensuring zero overhead on the critical path.

## Considered alternatives

### Long Animation Frames (LoAF) API

The Long Animation Frames API is a fantastic tool for measuring responsiveness, but it is fundamentally unsuited for this specific use case for two reasons:

1. Frames vs. Tasks: LoAF is tied to a 50ms frame rendering threshold. We need to measure individual tasks with a configurable, much lower threshold to catch thread-hogging before a frame is even dropped.
2. Aggregated Attribution vs. Point-in-Time: LoAF uses a post-mortem, aggregated script attribution model (e.g., "Function X took 40ms total over the course of the frame"). Our design relies on triggering a mid-execution interrupt to capture a precise stack trace.

### Long Tasks API

The Long Tasks API measures tasks rather than frames, which aligns closer to our goal. However, its 50ms threshold is hardcoded into the spec. Extending it to accept a custom 5ms threshold and fundamentally changing its attribution model to use mid-execution stack traces would break the existing semantics of the API. Introducing a distinct task-interrupt entry type provides a cleaner separation of concerns.

## Security and Privacy Considerations

Exposing raw JavaScript stack traces to the web platform introduces significant privacy and security risks. Specifically, a malicious script could potentially use this API to read the call stacks of third-party scripts or cross-origin iframes, leaking sensitive execution data.

To mitigate this, the API must be strictly guarded. We propose the following security model:

1. Cross-Origin Isolation: The task-interrupt API should be restricted to secure contexts and gated behind strict Cross-Origin Isolation headers (Cross-Origin-Opener-Policy and Cross-Origin-Embedder-Policy), similar to how SharedArrayBuffer and high-resolution timers (performance.now()) are protected against side-channel attacks today.
2. Same-Origin Filtering: If Cross-Origin Isolation is deemed insufficient, the API will scrub or redact any frames in the stackTrace array that originate from cross-origin scripts unless appropriate CORS headers are present.

## Stakeholder Feedback / Opposition

- Chrome/WebUI : Positive (Driving initial incubation)
- Web Performance WG : Invited for discussion

## References & acknowledgements

Many thanks for valuable feedback and advice from:

- Fergal Daly
- Yoav Weiss
