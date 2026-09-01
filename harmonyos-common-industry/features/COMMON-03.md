---
id: COMMON-03
title: Navigation design practice - one UIAbility, many ArkUI pages, routed by Navigation rather than Router
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/03_practice-common-app-navigation-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-navigation-v1-0000001946960441
sample: none (architecture practice document, no sample project)
kits: ["@kit.ArkUI"]
apis: [UIAbility, Navigation, NavDestination, NavPathStack, Router]
permissions: []
min_api: n/a
modules: []
findings: []
status: verified
---

## When to use

Load this card when deciding **how an application moves between screens**: how
many UIAbility instances it has, how pages are declared, and which of the two
HarmonyOS routing mechanisms carries navigation. It is also the card to load when
a multi-module app suffers slow cold start because every module UI component is
loaded up front, or when feature modules have grown circular dependencies through
their route tables.

## Feature checklist

An application that follows this practice must:

- Use the Stage model with a **single UIAbility instance** - one UIAbility
  instance corresponds to one task in the task centre, so extra instances mean
  extra task-centre entries.
- Declare **multiple ArkUI pages** inside that one UIAbility rather than one
  ability per screen.
- Navigate between pages through the routing layer, not by starting abilities.
- Use **Navigation** as the routing mechanism. Router remains supported, but
  Navigation is the long-term evolution direction and the recommended choice.
- Register routes **dynamically**, so that loading a page does not force every UI
  component of every module to be constructed.
- Keep module route registration out of the modules themselves, so feature
  modules do not have to import each other.

## Architecture

Two routing mechanisms exist, and the document picks one:

| | Navigation | Router (@ohos.router) |
| --- | --- | --- |
| Page unit | content of a `NavDestination` component | a custom component decorated with `@Entry` |
| Position in the component tree | a container component mounted under a single page node; stackable and nestable | pages sit beneath the page-stack stage-management node |
| Recommended | yes - long-term evolution direction | supported, but not the direction |

The official introduction states the reason directly: compared with
`@ohos.router`, the Navigation framework "offer[s] enhanced multi-device
adaptation capabilities, more flexible page stack management, and richer
animation effects and lifecycle support. Therefore, to deliver a better user
experience, prioritize the **Navigation** component for implementing both page
navigation and intra-component navigation whenever possible."

Capabilities Navigation has and Router does not, per the official comparison
table: one-time development for multi-device deployment (Auto mode single/double
column adaptation), obtaining parameters of a specific page, obtaining the page
stack object, route interception via `setInterception`, removing specific routes
(`removeByIndexes` / `removeByName`), moving within the stack (`moveToTop` /
`moveIndexToTop`), shared-element animation with `geometryTransition`, and
immersive pages without extra window configuration.

Data flow in the recommended shape: `EntryAbility` loads one page; that page owns
a single `NavPathStack` and provides it to the whole tree; every feature module
exports `NavDestination`-rooted components; a route table maps a route-name
string to a builder for those components; navigation is
`pushPathByName(name, param)` and `pop()`. Because the mapping is by name and is
resolved dynamically, no feature module needs a compile-time reference to
another.

## Implementation steps

1. **Plan one UIAbility.** Keep the Stage-model application at a single UIAbility
   instance; add another only when you deliberately want a second task-centre
   entry.
2. **Split the UI into ArkUI pages inside that ability**, not into extra
   abilities.
3. **Choose Navigation.** Wrap the home content in a `Navigation` component and
   create exactly one `NavPathStack` for the application.
4. **Root every routed screen in a `NavDestination`.** Under Navigation a page is
   the content of a `NavDestination`, not an `@Entry`-decorated component.
5. **Register routes dynamically.** Bind a builder to `Navigation` that resolves
   a route name to a component at navigation time. This is the step that fixes
   both problems the document names: slow startup caused by loading many UI
   components at once, and circular dependencies between modules.
6. **Navigate by name and pass parameters as objects.** Navigation supports
   reading the parameters of a specific page and obtaining the stack object;
   Router does not.
7. **When you need route interception** (for example a confirm-before-leave
   dialog), use `setInterception` on the Navigation stack - Router has no
   equivalent and forces `showAlertBeforeBackPage` instead.

## Verified snippets

Not applicable - the document contains no code and has no sample project. Working
Navigation code verified against sample ZIPs is available in the scenario cards
of this industry, for example COMMON-50 (Navigation route continuation).

## Permissions & config

Not applicable - no permissions are involved. Under Navigation there is no router
table in `module.json5`; the route mapping lives in the page builder.

## Constraints

- **Router is not removed.** The document says Navigation is the long-term
  evolution and recommended option; it does not say Router is deprecated, and no
  deprecation statement exists in
  `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-router.md`.
  Existing Router code keeps working.
- **Router cannot do some things at all.** Per the official capability table:
  obtaining parameters of a specific page, obtaining a page stack object, route
  interception, `removeByIndexes` / `removeByName`, `moveToTop` /
  `moveIndexToTop`, `geometryTransition` shared-element animation, and multi-device
  single/double column adaptation are all "Not supported" on Router. If the
  product needs any of these, Navigation is not a preference but a requirement.
- **Router restricts parameter shape**: parameters are passed as an object, but
  "method variables not supported within the object". Navigation has no such
  restriction.
- **Immersive pages under Router require window configuration**; Navigation
  supports them directly.
- The document sets no API level, device or region restriction of its own.

## Pitfalls

None recorded. Every claim in the document - the single-UIAbility/one-task
correspondence, the existence of exactly two routing mechanisms, and Navigation
being the recommended and long-term choice - matches
`documentation/harmonyos-guides/03_application-framework/arkts-navigation-introduction.md`.

Two things worth not over-reading:

- **"Navigation is recommended" is not "Router is deprecated."** Do not report or
  refactor Router usage as deprecated API; the Router class reference carries no
  deprecation notice.
- **"Single UIAbility" is a design recommendation, not a platform limit.** The
  reason given is task-centre behaviour - one UIAbility instance maps to one task
  - not an inability to create more.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-introduction.md` -
  page definitions under each framework, the "prioritize the Navigation
  component" recommendation, the architecture advantages list, and the full
  Navigation-vs-Router capability comparison table.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-introduction
- `documentation/harmonyos-guides/03_application-framework/arkts-navigation-navigation.md` -
  the Navigation component itself.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-navigation-navigation
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-router.md` -
  Router class reference; checked for a deprecation statement, none present.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-router
- `documentation/harmonyos-guides/01_getting-started/hap-package.md` - the
  "one UIAbility + multiple pages" recommendation.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/hap-package
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-navigation-v1-0000001946960441
