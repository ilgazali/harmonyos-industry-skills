# Performance

A triage checklist, not a tutorial. Source: HQ's performance optimisation
practice; the official guides for each mechanism are linked under
[See also](#see-also).
https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-performance-v1-0000001949813401

## Three areas, three bottlenecks

| Area | Bottleneck | Mechanism |
|---|---|---|
| ArkUI | main-thread frame budget: layout and render | lazy construction, node recycling, off-screen caching |
| ArkTS | main-thread CPU time: long computation | move off-thread or off the critical path |
| Web | network latency before first paint | do the work earlier |

**Measure first.** Use the DevEco profiler sessions (launch, frame, CPU,
allocations) to find which area is responsible. The checklist below lists
practices, not diagnoses.

## ArkUI

1. **No time-consuming work in lifecycle callbacks.** Move file, network and
   heavy computation out of `aboutToAppear` and `build`.
2. **Long lists: `LazyForEach` + `@Reusable` + `cachedCount`.** All three, in
   one pass. The source writes it with plus signs and it is one practice, not
   three alternatives:
   - `LazyForEach` over an `IDataSource` limits how many items are ever built
   - `@Reusable` avoids rebuilding items that scroll back in
   - `cachedCount` on the `List` / `Grid` / `WaterFlow` decides how many stay
     warm off-screen

   Drop any one and you reintroduce the cost the other two were avoiding.
   Give `LazyForEach` a stable key; do not build the key by serialising the item.
3. **Control visibility with state, not rebuilds.** Show and hide with a state
   variable instead of reconstructing the subtree.

Only 6 of 443 cards use `@Reusable`. If you are building a list, you are
already ahead of most samples in the corpus by adding it.

## ArkTS

Classify the heavy work:

- **I/O-bound** - make it async.
- **CPU-bound, short and independent** - `taskpool`.
- **CPU-bound, long-lived with its own context** - `worker`.

Remember that state management does not cross the thread boundary (see
[state-management.md](state-management.md)): compute off-thread, assign on the
main thread.

## Web

"Web" here means an embedded ArkWeb page, not the ArkUI declarative UI. These
three do nothing for a native screen.

A ladder of increasing commitment:

1. **Pre-connection** - `prepareForPageLoad()`, typically from the `Web`
   component's `onAppear`. Cheapest, helps every page. But it "only performs DNS
   resolution on URLs and establishes TCP connections, but does not obtain main
   resources and subresources" - it will not fix a slow first paint caused by
   large resources.
2. **Preloading** - fetches resources ahead of time.
3. **Pre-rendering** - creates an offline `Web` component in advance, activates
   it so the rendering engine renders in the background, and attaches it through
   a `NodeController` / `NodeContainer` pair only when needed. This holds a live
   rendering process for as long as the offline component exists, and needs the
   resource set known in advance.

## Re-measure

Every item here has a memory cost - `cachedCount`, pre-rendered offline
components, worker threads. Confirm the frame-time win is real before keeping
it.

## See also

- `documentation/harmonyos-guides/03_application-framework/arkts-component-reusable.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-component-reusable
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/03_application-framework/web-predictor.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-predictor
- `documentation/harmonyos-guides/03_application-framework/web-offline-mode.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/web-offline-mode
- `documentation/harmonyos-guides/14_performance-optimization/ide-profiler-introduction.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide-profiler-introduction
