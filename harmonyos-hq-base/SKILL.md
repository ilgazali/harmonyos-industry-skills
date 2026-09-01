---
name: harmonyos-hq-base
description: Use when building or reviewing a HarmonyOS (ArkTS/ArkUI) application without a matching industry skill - module layout, HAP/HAR/HSP choice, Navigation routing, state management, performance triage and permission handling as Huawei HQ applies them across every industry.
---

# HarmonyOS HQ Base

What is common to every HarmonyOS application, distilled from Huawei HQ's
published industry best practices and verified against 399 of their sample
projects and the official guides.

**This folder is the source, not something you normally install.** Its
`references/` are copied into every `harmonyos-*-industry` skill as
`references/base/` when the package is built, so an industry skill is a single
self-contained folder. Install this one only when you are building a HarmonyOS
app that no industry skill covers.

Edit here and re-run `review/scripts/build_skill_refs.py`; the 19 copies are
generated and must not be edited directly.

## References

Read the one that matches the decision in front of you. Do not read all six.

| File | Load it when |
|---|---|
| [references/layered-architecture.md](references/layered-architecture.md) | starting a project, or untangling module dependencies |
| [references/module-types.md](references/module-types.md) | deciding HAP vs HAR vs HSP for a module |
| [references/navigation.md](references/navigation.md) | wiring screens together, or choosing Navigation vs Router |
| [references/state-management.md](references/state-management.md) | choosing decorators, or a UI does not update |
| [references/performance.md](references/performance.md) | the app is slow and you need the short list |
| [references/permissions.md](references/permissions.md) | declaring or requesting any permission |
| [references/project-skeleton/](references/project-skeleton/) | you need the files, not the prose |

## The short version

1. **Three tiers, one direction.** Product customisation depends on basic
   features, which depend on common capabilities. Never the reverse, never a
   cycle between siblings.
2. **One entry HAP, everything else a HAR.** Add feature HAPs only for genuine
   independent deployment; reach for HSP only when HAR duplication actually
   hurts.
3. **One UIAbility, many pages, routed by `Navigation`.** Not one ability per
   screen, and not `@ohos.router` for new code. A HAR cannot declare `pages`, so
   in a multi-module app Navigation is the only way to reach a feature's screens.
4. **State management is V1 in this corpus.** Read `@State` / `@Link` /
   `@Provide` / `@Observed` fluently; the official advice for new apps is V2.
   State cannot be touched from a Worker or TaskPool.
5. **Long lists are `LazyForEach` + `@Reusable` + `cachedCount`.** All three.
   Measure with the profiler before optimising anything.
6. **Declare only permissions your code exercises**, each with `reason` and
   `usedScene`, once per app. Check the grant result before proceeding.

## Working with an industry skill

1. Load the industry skill for the app being built. It already contains this
   content under `references/base/`.
2. Open its `references/feature-catalog.md` and match the request onto feature
   IDs. Name the IDs you matched.
3. Read `features/<ID>.md` in full before writing code.
4. Copy from the card's **Verified snippets**, which come from the sample
   project, never from the published documentation.
5. Check the card's **Pitfalls** and the industry's
   `references/pitfalls.md` before finishing. These are confirmed defects in
   HQ's published documents; a naive copy reproduces them.
6. Declare every permission the card lists, and nothing more.

## Provenance

Built from a review of 443 HQ industry documents and their sample projects,
which produced 1665 evidence-backed findings. Every claim here traces to either
an official guide under `documentation/`, a card in
`harmonyos-common-industry`, or a finding ID in `review/bugs/`. Where HQ's own
documentation is wrong, these files say so and give the correction.
