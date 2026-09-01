---
id: COMMON-01
title: Key scenario catalogue - the entry point of the HarmonyOS common technical solutions library
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/01_practice-common-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-architecture-v1_1-0000002317481234
sample: none (index document, no sample project)
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: []
status: verified
---

## When to use

Load this card when you need to find **which common technical solution document
covers a given product requirement**. It is the index page of the
`architecture-guides` "common technical solutions" branch: it does not describe a
feature of its own, it enumerates the 50 sibling documents that do.

Use it as a routing table: read the requirement, pick the matching row below,
then load the corresponding `COMMON-xx` card.

## Feature checklist

This document is a catalogue, so the "checklist" is what it must provide:

- One entry per solution in the branch, each linking to its own guide page.
- Coverage of the five architecture-level topics (layering, navigation,
  performance, porting, Huawei Pay) before the 45 concrete scenario solutions.
- Ordering that matches the site sidebar, so the reader can navigate between the
  index and the sidebar without ambiguity.

Verified against `huawei_industry_tree/19_common_technical_solutions/tree.json`:
the index lists exactly the 50 child nodes of 关键场景示例 ("Key scenario
examples"), in sidebar order. 行业常见问题 ("Industry FAQ", COMMON-52) is a
sibling of this page rather than a child, which is why it is absent from the list
- that is correct, not an omission.

## Architecture

The branch is organised in two tiers:

| Tier | Documents | Nature |
| --- | --- | --- |
| Architecture practice | COMMON-02 layering, COMMON-03 navigation, COMMON-04 performance, COMMON-05 porting, COMMON-06 Huawei Pay | Prose guidance, no downloadable sample |
| Scenario solutions | COMMON-07 through COMMON-51 | Each has 方案概述 / 实现思路 ("solution overview" / "implementation approach") plus a downloadable sample project |

The index links to both tiers uniformly; the presence or absence of a sample
project is not signalled on the index page, only on the individual pages.

Routing table by requirement area:

- App shell and theming: COMMON-08 feature guide page, COMMON-10 splash guide
  swiper, COMMON-11 edition switch, COMMON-12 custom theme, COMMON-16 TabBar
  blur, COMMON-30 tab toggle animation, COMMON-42 dark mode, COMMON-41 page
  grayscale, COMMON-40 in-app language switch.
- Layout and form factor: COMMON-31 two-column page, COMMON-32 foldable split
  mode, COMMON-36 foldable web layout switch, COMMON-49 PC system tray.
- ArkWeb / H5 integration: COMMON-09 gesture-back interception, COMMON-15 H5
  file upload, COMMON-23 H5 scan, COMMON-28 H5 press-to-save, COMMON-33 H5 custom
  font, COMMON-34 H5 url-scheme app launch, COMMON-39 H5 privacy mode, COMMON-46
  H5 scroll-to-top.
- Identity and compliance: COMMON-13 user authentication, COMMON-17 silent
  login, COMMON-27 blur-until-login, COMMON-29 user agreement and privacy policy,
  COMMON-37 text-order human verification, COMMON-38 slide-puzzle verification.
- Data, files and connectivity: COMMON-07 cache clean, COMMON-14 picture upload,
  COMMON-21 download management, COMMON-19 BLE scan and connect, COMMON-48 Wi-Fi
  connect, COMMON-44 VPN long-running task, COMMON-47 UTF-8 to GB2312, COMMON-51
  persistent file permissions.
- Observability and system events: COMMON-18 load-time instrumentation,
  COMMON-22 page tracing points, COMMON-20 screenshot listener, COMMON-43 custom
  common events, COMMON-25 accessibility, COMMON-26 splash ads, COMMON-50
  Navigation continuation.
- Motion: COMMON-24 shared-element transition, COMMON-45 connection animation,
  COMMON-35 title/content linkage.

## Implementation steps

There is no implementation. The steps for using the index are:

1. Identify the requirement area from the routing table above.
2. Open the corresponding document under
   `huawei_industry_tree/19_common_technical_solutions/docs/`.
3. If that document has a sample project, verify the code against the ZIP under
   `huawei_industry_tree/19_common_technical_solutions/downloads/` rather than
   against the document snippet - across this branch the two differ often enough
   that the ZIP must be treated as the source of truth.
4. Read the architecture-practice documents (COMMON-02 to COMMON-05) before the
   scenario documents when starting a new application; they set the module
   layout and routing choices every scenario document assumes.

## Verified snippets

Not applicable - this document contains no code and has no sample project.

## Permissions & config

Not applicable - no permissions and no `module.json5` are involved.

## Constraints

- The index carries no API level, device or region statement; every such
  constraint lives on the individual solution pages.
- Every link on the page is an external `developer.huawei.com` URL. Link
  liveness was not verified (no network access during this review), so no
  dead-link finding is recorded either way.
- The page has no 方案概述 / 实现思路 sections that the scenario documents carry.
  This is expected for an index node and is not treated as a missing mandatory
  section.

## Pitfalls

None recorded. The document's list was checked entry-by-entry against
`tree.json` and against
`huawei_industry_tree/19_common_technical_solutions/docs/README.md`; all three
agree on the same 50 entries in the same order.

## References

- `huawei_industry_tree/19_common_technical_solutions/tree.json` - sidebar
  structure used to confirm the index is complete and correctly scoped.
- `huawei_industry_tree/19_common_technical_solutions/docs/README.md` - crawled
  document index, cross-checked against the page's own list.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-architecture-v1_1-0000002317481234
