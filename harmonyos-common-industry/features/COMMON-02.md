---
id: COMMON-02
title: Layered modular architecture - three tiers mapped onto HAP, HAR and HSP package types
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/02_practice-common-app-layered-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-layered-v1-0000001916033058
sample: none (architecture practice document, no sample project)
kits: []
apis: []
permissions: []
min_api: n/a
modules: [entry HAP, feature HAP, HAR, HSP]
findings: [HW-19-0001]
status: verified-with-fixes
---

## When to use

Load this card at the very start of a new HarmonyOS application, before any
module is created, when you must decide **how many packages the app ships as and
what kind each package is**. It is also the card to load when an existing app has
grown circular dependencies between feature modules, or when the same feature has
to behave differently on phone, tablet and PC.

The stated goal of the document is 一次开发，多端部署 ("develop once, deploy to
multiple devices"): one codebase whose layering lets each device get the right
subset of packages.

## Feature checklist

An application that follows this practice must:

- Separate code into three tiers: product-customisation, basic-feature, and
  common-capability.
- Compile the product-customisation tier into entry-type HAPs (one per device
  family when split, or one Default entry HAP when shared).
- Compile each basic feature as its own module, high-cohesion and low-coupling,
  choosing feature HAP when it must be deployable on its own and HAR/HSP when it
  must not.
- Compile the common-capability tier (UI components, utility classes) into HAR or
  HSP only - never into a HAP.
- Keep the dependency direction one-way: product-customisation depends on
  basic-feature depends on common-capability, with no reverse or sibling cycles.
- Make the split-vs-shared packaging decision per module, based on how much that
  module behaviour actually differs across devices.

## Architecture

Three tiers, as defined by the document:

| Tier | Chinese name | Contents | Package type |
| --- | --- | --- | --- |
| Product customisation | 产品定制层 | Per-device personalised business logic: UI, resources, configuration | entry-type HAP |
| Basic feature | 基础特性层 | Abstracted application features, each high-cohesion and low-coupling, flexibly deployable | feature HAP if it must be deployed independently; otherwise HAR or HSP |
| Common capability | 公共能力层 | Capabilities the basic-feature tier depends on: UI components, utility classes | HAR or HSP |

The document notes that developers may sub-divide further **inside** any of the
three tiers when the application is complex enough to warrant it - the three
tiers are a floor, not a ceiling.

Data and dependency flow: the entry HAP is the only module the system launches;
it imports feature modules, which import common modules. Because HAR content is
copied into each dependent at build time while HSP content is shared at runtime,
the HAR-vs-HSP choice is the size-versus-load-time trade-off inside the two lower
tiers, and it does not change the tier boundaries.

The document own summary of practice: **one entry HAP plus multiple HARs is the
common shape**, typical of single-device applications with no dynamic-deployment
requirement.

## Implementation steps

1. **Classify every planned feature into one of the three tiers.** Anything that
   is device-specific UI, resources or configuration belongs to the
   product-customisation tier. Anything reusable by more than one feature belongs
   to the common-capability tier. Everything else is a basic feature.
2. **Decide, per basic feature, whether it must be deployed on its own.** If yes
   it becomes a feature-type HAP; if no it becomes a HAR or an HSP. The official
   HAP guide confirms the constraint this decision lives under: "An application
   package can contain either only one entry HAP or one entry HAP plus one or more
   feature HAPs", and "each type of device supports zero or one entry HAP and
   zero, one, or more feature HAPs".
3. **Prefer a single entry HAP.** The official guide recommends a single entry
   HAP when the application only uses UIAbility, and multi-HAP only when
   ExtensionAbilities force it. Adopt the "one UIAbility + multiple pages" mode
   inside that HAP.
4. **Make the split-vs-shared call per module**, using the document three rules:
   - Applications with a desktop icon where tablet and PC differ little: use
     **Entry shared packaging** - one Default entry-type HAP distributed to
     multiple devices.
   - Modules whose behaviour differs substantially between devices: use **Entry
     split packaging** - a separate entry-type HAP planned per device type. (The
     document sentence here is truncated; see HW-19-0001.)
   - Layout structures that diverge strongly at the same breakpoint: split.
     Otherwise: share.
5. **Compile the common-capability tier as HAR or HSP and nothing else.** Putting
   shared utilities in a HAP makes them undeployable as a dependency.
6. **Verify the module graph is acyclic** before the first release; the layering
   only pays off if the dependency direction is enforced.

## Verified snippets

Not applicable - the document contains no code and has no sample project. The
package-type rules above were cross-checked against
`documentation/harmonyos-guides/01_getting-started/hap-package.md` rather than
against a ZIP.

## Permissions & config

Not applicable - no permissions are involved. The only configuration this
practice touches is the root `build-profile.json5` `modules` list and each
module `oh-package.json5` dependency block, neither of which the document
specifies.

## Constraints

- **One entry HAP per device type, at most.** The official HAP guide states that
  in an App Pack containing multiple HAPs, "each type of device supports zero or
  one entry HAP and zero, one, or more feature HAPs". Entry split packaging
  therefore means one entry HAP *per device type*, not several per device.
- **Multi-HAP has a real cost.** The official guide warns that in the multi-HAP
  scenario "multiple HAPs reference the same library file, which may cause
  repeated packaging". Choose feature HAPs only when independent deployment is
  genuinely required.
- **Single HAP is the recommended default.** "If your application only uses the
  UIAbility (that is, no ExtensionAbility is required), a single HAP (entry HAP)
  is recommended."
- The document sets no API level, device or region restriction of its own; it is
  a design practice, not an API.

## Pitfalls

- **The document sentence describing entry split packaging is incomplete, which
  is incorrect as written.** It says "Entry分包：针对不同单独规划一个Entry类型HAP
  包。" ("Entry split packaging: plan a separate entry-type HAP for different.")
  with no object after 针对不同. Read it as 针对不同**设备** ("for each device
  type"), which is the only reading consistent with the neighbouring sentence
  about distributing one Default entry HAP to multiple devices. (HW-19-0001)
- **Do not read 单Entry HAP包多HAR包 as a rule.** The document presents it as
  what is *common in practice* for single-device applications with no
  dynamic-deployment requirement - not as the prescribed shape for every app.
- **Putting common utilities in a feature HAP is a dead end.** The
  common-capability tier is explicitly restricted to HAR or HSP; a HAP cannot be
  consumed as a library dependency by another module.

## References

- `documentation/harmonyos-guides/01_getting-started/hap-package.md` - entry vs
  feature HAP definitions, the single-HAP recommendation, and the per-device HAP
  count constraint.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/hap-package
- `documentation/harmonyos-guides/01_getting-started/har-package.md` - HAR
  characteristics referenced by the basic-feature and common-capability tiers.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/har-package
- `documentation/harmonyos-guides/01_getting-started/in-app-hsp.md` - in-app HSP,
  the runtime-shared alternative to HAR.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/in-app-hsp
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-layered-v1-0000001916033058
