---
id: COMMON-05
title: HarmonyOS porting analysis - technology reuse routes and APK-to-HarmonyOS data migration
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/05_practice-common-app-transplant-v1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-transplant-v1-0000001921616314
sample: none (architecture practice document, no sample project)
kits: ["@kit.ArkTS", "@kit.ArkWeb"]
apis: [ArkTS, NDK, ArkWeb]
permissions: []
min_api: 9
modules: []
findings: []
status: verified
---

## When to use

Load this card when an existing application - typically an Android APK, or a
HarmonyOS application built for API 9 and earlier - has to be brought onto
HarmonyOS NEXT, and you need to decide **what gets rewritten, what gets reused as
is, and what happens to the user data already on the device**.

The document splits that decision into two independent problems: technology
migration (the code) and data migration (the user data).

## Feature checklist

A porting effort following this practice must:

- Classify every part of the existing codebase into one of three reuse routes:
  ArkTS rewrite, C/C++ reuse, or Web reuse.
- Rewrite the UI and business layers in ArkTS.
- Reuse existing C/C++ code through the NDK rather than reimplementing it.
- Reuse existing Web assets through ArkWeb rather than reimplementing them.
- Identify which on-device data of the old APK application the new HarmonyOS
  application must still be able to read.
- Convert and relocate that data to the required location, because it is not
  reachable from the HarmonyOS application otherwise.

## Architecture

**Technology migration.** The document maps each technology area to its
development guide:

| Technology area | Route | Guide |
| --- | --- | --- |
| ArkTS development | rewrite | ArkTS development guide |
| C/C++ technology reuse | reuse via NDK | NDK development guide |
| Web technology reuse | reuse via ArkWeb | ArkWeb development guide |

The shape of the result is a HarmonyOS application whose UI and application
framework are ArkTS, with native libraries built through the NDK behind an ArkTS
binding layer, and Web content hosted in `Web` components. Only the first row is
a rewrite; the other two are reuse routes, which is the point of the table.

**Data migration.** The scenario is a version upgrade of the device itself:
a terminal moves from HarmonyOS 3.1 Release (API 9) and earlier - which the
document calls simply "HarmonyOS" - to HarmonyOS NEXT Developer Preview 1 and
later - "HarmonyOS NEXT". The APK applications that ran on the old system must be
replaced by HarmonyOS applications on the new one. Their data does not carry over
by itself:

> 原HarmonyOS中运行的APK应用，升级后需要切换为HarmonyOS NEXT中的HarmonyOS应用。APK
> 应用的部分数据需要转换并迁移到指定位置后，才能被HarmonyOS应用访问。
> ("APK applications that ran on the original HarmonyOS must be switched to
> HarmonyOS applications on HarmonyOS NEXT after the upgrade. Some of the APK
> application data must be converted and migrated to a designated location before
> the HarmonyOS application can access it.")

So the data flow is: old APK data on the device -> conversion step -> designated
location -> readable by the new HarmonyOS application. The conversion and the
designated location are defined by the application data migration guide, not by
this document.

## Implementation steps

1. **Inventory the existing codebase by technology.** Separate it into UI and
   business logic (rewrite), native C/C++ (reuse), and Web assets (reuse).
2. **Rewrite the UI and business layers in ArkTS**, following the ArkTS
   development guide. This is where the architecture decisions of COMMON-02
   (layering) and COMMON-03 (navigation) get applied.
3. **Bring the C/C++ across with the NDK.** Build the existing native code as a
   HarmonyOS native library and expose it to ArkTS through the NDK binding layer,
   rather than porting the algorithms to ArkTS.
4. **Host existing Web content in ArkWeb.** Where a feature is already a web
   page, embed it in a `Web` component instead of rebuilding it natively; see
   COMMON-04 for the pre-connection / preloading / pre-rendering practices that
   make this acceptable in performance terms, and the H5 scenario cards
   (COMMON-09, 15, 23, 28, 33, 34, 39, 46) for the concrete integration patterns.
5. **List the on-device data the old APK owned** and decide, per data set,
   whether the new application must still read it.
6. **Convert and relocate that data to the designated location** before the new
   application tries to read it - it is unreachable until then. Follow the
   application data migration guide for the exact conversion and target path.
7. **Verify on an upgraded device**, not only on a clean install; the data
   migration path only exists on devices that carried data across the upgrade.

## Verified snippets

Not applicable - the document contains no code and has no sample project. It is a
routing document: one figure, one table of guides, and one prose paragraph on
data migration.

## Permissions & config

Not applicable - the document specifies no permissions and no `module.json5`
configuration. The permissions the ported application needs are determined by the
features it ports, not by the porting practice.

## Constraints

- **Version boundary.** The data-migration path applies specifically to devices
  upgrading from HarmonyOS 3.1 Release / API 9 and earlier to HarmonyOS NEXT
  Developer Preview 1 and later. It is not a general import mechanism, and it does
  not apply to a fresh install on a NEXT device.
- **Only part of the data migrates.** The document says 部分数据 ("some of the
  data"), not all of it - do not plan on a complete carry-over.
- **Migration is a precondition, not a fallback.** The data "才能被HarmonyOS应用
  访问" ("can only then be accessed by the HarmonyOS application"): until the
  conversion has happened, the new application cannot read it at all.
- **APK applications do not run on HarmonyOS NEXT.** They must be switched to
  HarmonyOS applications; the porting effort is not optional for those devices.
- The document sets no device-type or region restriction.

## Pitfalls

None recorded. The document makes no code-level claim that could be contradicted
by a sample, and its three technology routes correspond to the three official
guides it links.

Points that are easy to misread:

- **"HarmonyOS" in this document means the pre-NEXT system.** The paragraph
  defines 简称HarmonyOS ("referred to as HarmonyOS") for 3.1 Release / API 9 and
  earlier, and 简称HarmonyOS NEXT for Developer Preview 1 and later. Elsewhere in
  this industry, "HarmonyOS" means the current system - the abbreviation is local
  to this paragraph.
- **The table is about reuse, not about equivalence.** C/C++ and Web code is
  reused, but it still needs an ArkTS host application around it; there is no
  route in which the ported application has no ArkTS.
- **Link liveness was not verified.** The four external guide links
  (`arkts-get-started`, `ndk-development-overview`, `arkweb`,
  `app-data-migration-overview`) were not fetched during this review; the first
  three have local mirrors under `documentation/harmonyos-guides/`, the fourth
  does not, which is a gap in the local mirror rather than evidence of a dead
  link.

## References

- `documentation/harmonyos-guides/01_getting-started/arkts-get-started.md` -
  ArkTS development route.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-get-started
- `documentation/harmonyos-guides/10_ndk-development/ndk-development-overview.md` -
  C/C++ reuse route.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ndk-development-overview
- `documentation/harmonyos-guides/03_application-framework/arkweb.md` - Web reuse
  route.
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkweb
- https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/app-data-migration-overview -
  application data migration guide referenced by the data-migration section; no
  local mirror in `documentation/`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-common-app-transplant-v1-0000001921616314
