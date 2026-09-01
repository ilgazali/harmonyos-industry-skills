# State management

Source: the official state management overview, linked under
[See also](#see-also), plus what the 443 verified feature cards in this package
actually use.

## V1 or V2

ArkUI ships two versions. The official position:

> For new applications, you are advised to adopt the V2 version directly. For
> applications already using V1, there is no need to switch to V2 immediately if
> the features and performance of V1 already meet requirements.

**The HQ corpus is overwhelmingly V1.** Across 443 feature cards:

| V1 | cards | V2 | cards |
|---|---:|---|---:|
| `AppStorage` | 271 | `@Local` | 24 |
| `@State` | 242 | `@Trace` | 20 |
| `@Provide` | 93 | `@ObservedV2` | 19 |
| `@Link` | 82 | `@ComponentV2` | 16 |
| `@Consume` | 81 | `@Param` | 5 |
| `@Watch` | 70 | `@Event` | 1 |
| `@Prop` | 66 | `@Once` | 0 |
| `@Observed` | 63 | | |
| `@ObjectLink` | 29 | | |

So: **read V1 fluently, because every industry card you will open is written in
it.** For new code the official advice is V2, but do not mix the two sets inside
one component - a `@ComponentV2` struct accepts only V2 decorators.

## V1 decorators

Component-level, by data flow:

| Decorator | Direction | Use |
|---|---|---|
| `@State` | owns the state | source of truth inside a component |
| `@Prop` | one-way, parent to child | child may mutate its copy; parent does not see it |
| `@Link` | two-way with parent | child mutation propagates up |
| `@Provide` / `@Consume` | two-way, across levels | skips intermediate components; bind by name or alias |
| `@Observed` + `@ObjectLink` | two-way, nested | required for nested objects and arrays |
| `@Watch` | callback | fires when the decorated state changes |

Application-level:

| | Two-way | One-way |
|---|---|---|
| `AppStorage` | `@StorageLink` | `@StorageProp` |
| `LocalStorage` | `@LocalStorageLink` | `@LocalStorageProp` |

`AppStorage` is app-global and in-memory. `PersistentStorage` persists selected
`AppStorage` properties across restarts - only 4 cards use it, so most samples
lose their state on exit by design.

## V2 decorators

`@ObservedV2` + `@Trace` for deep observation on a class; `@ComponentV2` for the
struct, which then admits `@Local` (internal, not externally initialisable),
`@Param` (input), `@Once` (initialise-only, pairs with `@Param`), `@Event`
(output), `@Monitor` (deep change observation), `@Provider` / `@Consumer`
(cross-level two-way), `@Computed` (memoised derived value), and the `!!`
two-way binding sugar.

What V2 fixes, per the guide: V1 state cannot exist independently of the UI;
V1 detects only top-level property changes, not deep ones; V1 re-renders
redundantly when nested properties or array elements change.

## The constraint that catches people

> State management can be used only in the UI main thread and cannot be used in
> child threads, Worker, or TaskPool.

This bites exactly where [performance.md](performance.md) sends you: you move
heavy work to a TaskPool, then cannot touch `@State` from inside it. Return the
result to the main thread and assign there.

## Rules of thumb from the corpus

- Nested object or array that must trigger updates: `@Observed` on the class
  **and** `@ObjectLink` on the field. `@State` alone observes only the top
  level - this is the single most common state bug in the samples.
- Passing data more than one level down: `@Provide` / `@Consume` rather than
  threading `@Prop` through every intermediate component.
- Cross-page or cross-UIAbility state: `AppStorage`, not a module-level
  singleton.
- Show and hide with a state variable rather than rebuilding the subtree.
- Never mutate a `@State` object's inner field and expect a re-render without
  `@Observed`.

## See also

- `documentation/harmonyos-guides/03_application-framework/arkts-state-management-overview.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-state-management-overview
- `documentation/harmonyos-guides/03_application-framework/arkts-v1-component-state-management.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-v1-component-state-management
- `documentation/harmonyos-guides/03_application-framework/arkts-state-management-v1-v2-migration-guide.md`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-state-management-v1-v2-migration-guide
