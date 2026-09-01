---
name: harmonyos-utilities-industry
description: Use when building a HarmonyOS utility or tool app - printing, device info widgets, network panels, compasses and gauges, ringtone setting, floating tool balls, immersive keyboards, or web-page capture.
---

# HarmonyOS Utilities and Tools Industry

Huawei HQ's verified feature catalog for this vertical: 47 features drawn
from `15_utilities`, backed by 110 evidence-based findings against HQ's own
published documentation. Feature IDs are prefixed `UTIL-`.

## Before anything else

Read [references/base/](references/base/). It carries the module layout,
routing, state, performance and permission rules Huawei HQ applies across every
industry, and this skill is self-contained: nothing else needs installing.

The rules in one paragraph each:

1. **Three tiers, one direction.** Product customisation depends on basic
   features, which depend on common capabilities. Never the reverse, never a
   cycle between siblings.
2. **One entry HAP, everything else a HAR.** Feature HAPs only for genuine
   independent deployment; HSP only when HAR duplication actually hurts.
3. **One UIAbility, many pages, routed by `Navigation`.** Not one ability per
   screen, and not `@ohos.router` for new code. A HAR cannot declare `pages`, so
   in a multi-module app Navigation is the only way into a feature's screens.
4. **State management is V1 in this corpus.** Read `@State` / `@Link` /
   `@Provide` / `@Observed` fluently; the official advice for new apps is V2.
   State cannot be touched from a Worker or TaskPool.
5. **Long lists are `LazyForEach` + `@Reusable` + `cachedCount`.** All three.
   Measure with the profiler before optimising anything.
6. **Declare only permissions your code exercises**, each with `reason` and
   `usedScene`, once per app. Check the grant result before proceeding.

Everything below is what is specific to utilities and tools applications.

## How to use this skill

1. Open [references/feature-catalog.md](references/feature-catalog.md) and map
   the request onto the catalog. Name the feature IDs you matched.
2. Read `features/<ID>.md` in full before writing any code.
3. Follow the module layout from
   [references/base/layered-architecture.md](references/base/layered-architecture.md).
4. Copy from the card's **Verified snippets**. Those come from the sample
   project's compiled source, not from the published documentation, which is
   abridged and in places wrong.
5. Check the card's **Pitfalls** and
   [references/pitfalls.md](references/pitfalls.md) before you finish.
   43 of 47 cards here correct at least one defect in HQ's
   published document; a naive copy reproduces it.
6. Declare every permission the card lists in `module.json5`, with `reason` and
   `usedScene`, and nothing more.

## Files

| File | What it holds |
|---|---|
| [references/feature-catalog.md](references/feature-catalog.md) | all 47 features in one table - start here |
| [references/api-map.md](references/api-map.md) | feature to kit / permission / API level, plus reverse indexes |
| [references/pitfalls.md](references/pitfalls.md) | the 110 findings, systemic ones first |
| `features/UTIL-*.md` | one card per feature: checklist, architecture, verified code, pitfalls |
| [references/base/](references/base/) | the cross-industry rules: layered architecture, module types, navigation, state, performance, permissions, plus a three-tier project skeleton |
| [EVIDENCE.md](EVIDENCE.md) | each feature's source document and sample project |

## Industry conventions

- The largest catalog in the package at 47 features; treat it as the grab-bag for anything not obviously another vertical.
- Canvas instruments - radar, azimuth dial, speed gauge - share one drawing pattern.
- Home-screen widgets and the floating tool ball both live outside the page lifecycle; their pitfalls differ from the rest.
