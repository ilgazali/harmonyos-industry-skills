---
name: harmonyos-common-industry
description: Use when building any HarmonyOS app and the need is cross-cutting rather than industry-specific - theming and dark mode, login gates, cache clearing, launch guides, payment, authentication, localisation, or embedded H5.
---

# HarmonyOS Common Technical Solutions Industry

Huawei HQ's verified feature catalog for this vertical: 52 features drawn
from `19_common_technical_solutions`, backed by 183 evidence-based findings against HQ's own
published documentation. Feature IDs are prefixed `COMMON-`.

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

Everything below is what is specific to common technical solutions applications.

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
   48 of 52 cards here correct at least one defect in HQ's
   published document; a naive copy reproduces it.
6. Declare every permission the card lists in `module.json5`, with `reason` and
   `usedScene`, and nothing more.

## Files

| File | What it holds |
|---|---|
| [references/feature-catalog.md](references/feature-catalog.md) | all 52 features in one table - start here |
| [references/api-map.md](references/api-map.md) | feature to kit / permission / API level, plus reverse indexes |
| [references/pitfalls.md](references/pitfalls.md) | the 183 findings, systemic ones first |
| `features/COMMON-*.md` | one card per feature: checklist, architecture, verified code, pitfalls |
| [references/base/](references/base/) | the cross-industry rules: layered architecture, module types, navigation, state, performance, permissions, plus a three-tier project skeleton |
| [EVIDENCE.md](EVIDENCE.md) | each feature's source document and sample project |

## Industry conventions

- This is the cross-industry catalog. Check it before assuming a need is specific to your vertical.
- Its first five cards are the architecture practices that `harmonyos-hq-base` is distilled from; read the base skill instead unless you need the full source.
- Theme switching, dark mode and edition switching interact - read all three before implementing one.
