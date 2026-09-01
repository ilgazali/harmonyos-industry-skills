---
id: SPORT-15
title: Sports and health industry FAQ - a redirect to the HarmonyOS FAQ
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/15_practice-sports-health-app-architecture-v1_2.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1_2-0000002307975325
sample: NO ZIP
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

Do not. This page holds no content.

It is the third and last of the architecture-level documents in the sports and
health practice, and its entire body is a migration notice:

> 行业常见问题内容已迁移至HarmonyOS FAQ，请点击此处前往。
> (The industry FAQ content has been migrated to the HarmonyOS FAQ; click here
> to go there.)

The link points at `harmonyos-faqs/faq-phone`, outside this corpus.

Load `SPORT-01` for the architecture, `SPORT-02` for the scenario catalog.

## Feature checklist

Nothing to implement. The page is a stub by design, not by omission - the
content moved rather than being lost, and the notice says where.

## Architecture

None. Eleven lines, no code, no images, no sample.

Its place in the industry:

| Document | Card | State |
| --- | --- | --- |
| 运动健康应用案例 | `SPORT-01` | the architecture and the framework code |
| 关键场景示例 | `SPORT-02` | the catalog of the twelve scenario samples |
| 行业常见问题 | `SPORT-15` (this card) | migrated away; a redirect only |

The same three-document shape, and the same empty third page, appears in
`09_tourism` (`TOUR-13`).

## Implementation steps

None.

## Verified snippets

None - the document contains no code.

## Permissions & config

None.

## Constraints

- The target of the redirect is the general HarmonyOS phone FAQ, not a
  sports-and-health-specific page, so nothing industry-specific survives here.
  Anything a reader needed from the old 行业常见问题 has to be found by search
  in the general FAQ.
- The kits this industry actually leans on each have their own troubleshooting
  page in the corpus, and those are the pages to reach for instead:
  `documentation/harmonyos-guides/04_system/bluetooth-faq.md` for the BLE
  pairing in `SPORT-01`, `documentation/harmonyos-guides/05_media/scan-faq.md`
  for the barcode scanner in `SPORT-14`, and
  `documentation/harmonyos-guides/07_application-services/map-faq.md` through
  `map-faq-5.md` for the two map samples, `SPORT-08` and `SPORT-11`.

## Pitfalls

None in the document itself. The corpus-level gap is worth stating: because
this page is empty, **there is no sports-and-health-specific troubleshooting
anywhere in this industry** - and the three failure families its samples ship
with are exactly what an industry FAQ would have caught.

- **Subscriptions that are opened and never closed.** Six of the fifty-three
  findings, and the industry's only blocker among them: the pedometer that
  re-subscribes from inside its own callback (`HW-03-0001`), the BLE
  unsubscribe aimed at a freshly created client rather than the connected one
  (`HW-03-0003`), the scan stopped only from a lifecycle callback that never
  runs (`HW-03-0004`), the map listener and trace overlay with no teardown
  (`HW-03-0035`), and the continuous location subscription that outlives its
  page (`HW-03-0042`, `HW-03-0043`). Every one of these is a battery drain on
  a device class - wearable companions - where battery is the product.
- **Map Kit modules shipped without their network configuration.** Both map
  samples declare no `INTERNET` permission, and `MaxDisplayOfRoutes`
  additionally has no `client_id` metadata at all (`HW-03-0032`,
  `HW-03-0044`), while the framework code nests its `client_id` inside the
  ability instead of the module (`HW-03-0006`). A map that renders blank is
  the single most common Map Kit support question, and
  `documentation/harmonyos-guides/07_application-services/map-faq-1.md`
  answers it.
- **Permission declarations written without a real reason.** Three permissions
  sharing `$string:EntryAbility_desc` (`HW-03-0007`), both calendar
  permissions requested before the user has expressed any intent
  (`HW-03-0022`), a permission scoped to an ability the module does not
  contain (`HW-03-0037`), and location justified by a generic string
  (`HW-03-0044`). These are review-gate failures, not runtime ones, so they
  surface late.

A fourth, smaller family is worth a line because it is trivially preventable:
**the documented project trees do not match the zips**. Four findings across
four documents (`HW-03-0009`, `HW-03-0028`, `HW-03-0041`, `HW-03-0053`) - a
misspelled directory, a renamed page, an omitted page, a model file described
as something else, and a test-asset path the reader cannot follow.

## References

- `huawei_industry_tree/03_sports_health/docs/15_practice-sports-health-app-architecture-v1_2.md` - the page itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1_2-0000002307975325
- `documentation/harmonyos-guides/04_system/bluetooth-faq.md` - BLE troubleshooting, for `SPORT-01`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/bluetooth-faq
- `documentation/harmonyos-guides/05_media/scan-faq.md` - scan troubleshooting, for `SPORT-14`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/scan-faq
- `documentation/harmonyos-guides/07_application-services/map-faq.md`, `map-faq-1.md` - map not displayed, for `SPORT-08` and `SPORT-11`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq
- `SPORT-01` - the architecture document this page belongs to
- `SPORT-02` - the scenario catalog
- `TOUR-13` - the identical redirect page in `09_tourism`
- `documentation/harmonyos-guides/07_application-services/map-faq-1.md` - the map FAQ the answer points at
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/map-faq-1
