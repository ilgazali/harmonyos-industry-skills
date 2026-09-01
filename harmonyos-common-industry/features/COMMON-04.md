---
id: COMMON-04
title: Performance optimisation practice - ArkUI, ArkTS and Web checklists for a responsive app
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/04_practice-common-app-performance-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-performance-v1-0000001949813401
sample: none (architecture practice document, no sample project)
kits: ["@kit.ArkUI", "@kit.ArkTS", "@kit.ArkWeb"]
apis: [LazyForEach, "@Reusable", cachedCount, taskpool, worker, "WebviewController.prepareForPageLoad", "WebviewController.prefetchPage", NodeController, NodeContainer]
permissions: []
min_api: n/a
modules: []
findings: []
status: verified
---

## When to use

Load this card when an application feels slow and you need the short list of
**what to check first**, split by the technology causing the problem: ArkUI
rendering, ArkTS execution, or an embedded Web page. It is a triage checklist,
not a tutorial - each item points at a full guide.

## Feature checklist

The document names three areas and, in each, the practices a high-performance app
must follow.

**High-performance ArkUI**

- No time-consuming operations inside custom-component lifecycle callbacks.
- Long lists built from `LazyForEach` **plus** component reuse **plus** list-item
  caching (`cachedCount`) - all three together, not one of them.
- State variables used deliberately, controlling element show/hide rather than
  rebuilding trees.

**High-performance ArkTS**

- Asynchronous concurrency used where it fits.
- Multi-threaded concurrency through **TaskPool** or **Worker** for work that must
  leave the UI thread.

**High-performance Web UI**

- Pre-parsing and pre-connection.
- Pre-download / preloading.
- Pre-rendering.

## Architecture

The three areas map onto three different bottlenecks and three different
mechanisms:

| Area | Bottleneck | Mechanism |
| --- | --- | --- |
| ArkUI | main-thread frame budget: layout and render of components | lazy construction (`LazyForEach`), node recycling (`@Reusable`), off-screen caching (`cachedCount`) |
| ArkTS | main-thread CPU time: long-running computation | move it off-thread (`taskpool`, `worker`) or off the critical path (async) |
| Web | network latency before first paint | do the work earlier: DNS/TCP ahead of time, resources ahead of time, whole page ahead of time |

The ArkUI trio is a pipeline, which is why the document lists it as one item:
`LazyForEach` limits how many items are ever constructed, `@Reusable` avoids
reconstructing the ones that scroll back in, and `cachedCount` decides how many
stay warm off-screen. Dropping any one of the three reintroduces the cost the
other two were avoiding.

The Web trio is a ladder of increasing commitment: `prepareForPageLoad()` is
domain-level only - the official guide is explicit that "This method only
performs DNS resolution on URLs and establishes TCP connections, but does not
obtain main resources and subresources". Preloading fetches resources.
Pre-rendering, per the offline-Web guide, goes further: an offline `Web`
component is created in advance and activated so that "the rendering engine
[can] initiate background rendering", and is attached to the view tree through a
`NodeController` / `NodeContainer` pair only when the page is actually needed.

## Implementation steps

1. **Measure before changing anything.** Use the DevEco profiler sessions
   (launch, frame, CPU, allocations) to find which of the three areas is
   responsible; the document lists practices, not diagnoses.
2. **ArkUI - clear the lifecycle callbacks.** Move any file, network or heavy
   computation out of `aboutToAppear` / `build` and into an async or off-thread
   path.
3. **ArkUI - fix long lists in one pass.** Replace `ForEach` with `LazyForEach`
   over an `IDataSource`, mark the list-item component `@Reusable`, and set
   `cachedCount` on the `List` / `Grid` / `WaterFlow`. Give `LazyForEach` a stable
   key; do not build the key by serialising the item.
4. **ArkUI - control visibility with state, not with rebuilds.** Use a state
   variable to show and hide elements instead of reconstructing the subtree.
5. **ArkTS - classify the heavy work.** I/O-bound work goes async. CPU-bound work
   goes to `taskpool` (short, independent tasks) or `worker` (long-lived
   thread with its own context).
6. **Web - add pre-connection first.** Call `prepareForPageLoad()` for the URL
   you are about to load, typically from the `Web` component `onAppear`
   callback. This is the cheapest of the three and helps every page.
7. **Web - add preloading for known-next pages**, then pre-rendering only for the
   pages that justify holding a live rendering process: create the offline `Web`
   component in advance, activate it, and attach it through `NodeController` when
   the user navigates.
8. **Re-measure.** Each of these changes has a memory cost (`cachedCount`,
   pre-rendered offline components, worker threads); confirm the frame-time win
   is real before keeping it.

## Verified snippets

Not applicable - the document contains no code and has no sample project. The
pre-connection example referenced above lives in
`documentation/harmonyos-guides/03_application-framework/web-predictor.md`, and
the pre-rendering example in
`documentation/harmonyos-guides/03_application-framework/web-offline-mode.md`.

## Permissions & config

Not applicable - no permissions and no `module.json5` entries are involved.
`cachedCount` is a component attribute, `taskpool`/`worker` need no declaration,
and the Web acceleration APIs need only the `ohos.permission.INTERNET`
declaration the Web page load itself already requires.

## Constraints

- **`prepareForPageLoad()` is domain-level only.** Per the official guide it
  "only performs DNS resolution on URLs and establishes TCP connections, but does
  not obtain main resources and subresources" - it will not fix a slow first
  paint caused by large resources.
- **Pre-rendering holds a live rendering process.** The offline-Web guide notes
  the component is used to "pre-start the rendering process and pre-render web
  pages"; that cost is paid for as long as the offline component exists, and the
  guide uses a boolean to stop rendering once pre-rendering completes.
- **Pre-rendering needs the resource set known in advance**: "For a web page to
  be pre-rendered successfully, identify the resources to be loaded beforehand."
- The document sets no API level, device or region restriction of its own.

## Pitfalls

None recorded. Each practice the document names was matched to an official
mechanism: component reuse to
`arkts-component-reusable.md`, pre-parsing/pre-connection and prefetch to
`web-predictor.md`, and pre-rendering to `web-offline-mode.md`.

Two readings to avoid:

- **The ArkUI list item is one practice, not three alternatives.** "长列表使用
  LazyForEach+组件复用+缓存列表项的能力" ("for long lists use LazyForEach +
  component reuse + list-item caching") is written with plus signs; applying only
  `LazyForEach` leaves the reconstruction cost in place.
- **"Web UI" here means an embedded ArkWeb page, not the ArkUI declarative UI.**
  All three Web practices are ArkWeb APIs and do nothing for a native ArkUI
  screen.

## References

- `documentation/harmonyos-guides/03_application-framework/web-predictor.md` -
  pre-parsing/pre-connection via `prepareForPageLoad()` and its explicit scope
  limitation; prefetching.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-predictor
- `documentation/harmonyos-guides/03_application-framework/web-offline-mode.md` -
  offline `Web` component, pre-starting the rendering process and pre-rendering,
  `NodeController` / `NodeContainer` attachment.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-offline-mode
- `documentation/harmonyos-guides/03_application-framework/arkts-component-reusable.md` -
  `@Reusable` component recycling.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-component-reusable
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` -
  `LazyForEach` and `IDataSource`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/14_performance-optimization/ide-profiler-introduction.md` and the
  `ide-insight-session-*` pages - the profiler sessions used to locate the
  bottleneck before applying any of the above.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-profiler-introduction
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-performance-v1-0000001949813401
