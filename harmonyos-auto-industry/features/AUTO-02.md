---
id: AUTO-02
title: Key scenario index for the automotive industry
industry: 01_auto
doc: huawei_industry_tree/01_auto/docs/02_practice-auto-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-auto-app-architecture-v1_1-0000002347244997
sample: none
kits: []
apis: []
permissions: []
min_api: 20
modules: []
findings: []
status: verified
---

## When to use

Do not load this card on its own. It is the industry's scenario index: it
records which feature-level practices Huawei publishes for automotive apps, so a
router or an agent picking a card knows the full set. Load `AUTO-01` for the
architecture, then the card for the screen being built.

## Feature checklist

Huawei publishes five feature-level practices for this industry:

| Card | Scenario | Sample |
|---|---|---|
| `AUTO-03` | Bottom sheet pinned in view (底部弹窗固定显示) | `GlobalComponents.zip` |
| `AUTO-04` | Nearby petrol stations with Map Kit (基于MapKit实现附近加油站查询) | `NearbyGasStation.zip` |
| `AUTO-05` | One-tap dialling (一键拨号) | `CallPhone.zip` |
| `AUTO-06` | Dashboard customisation (仪表盘自定义) | `CustomizeDashboard.zip` |
| `AUTO-07` | Smart autofill of owner details (车主信息智能填充) | `CarMaintenance.zip` |

Everything else in an automotive app - the five-tab shell, the news feed, the
model catalogue, the accessory store, QR charging and the charging-station map -
is covered by the architecture card `AUTO-01` and its sample project.

## Architecture

Not applicable. The source document is a five-entry link list with no technical
content of its own.

## Implementation steps

None. Follow `AUTO-01` first, then the relevant feature card.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

The published feature set is small relative to a real automotive app. Vehicle
control, remote status, digital key, trip history and OTA updates have no HQ
practice document in this industry; build them on the `AUTO-01` architecture and
the common technical solutions skill.

## Pitfalls

No defects found in this document. All five links resolve to the documents they
name, and each link's text matches the `title` frontmatter of its target
exactly.

## References

- `huawei_industry_tree/01_auto/docs/03_global_components.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/global_components-0000002313328742
- `huawei_industry_tree/01_auto/docs/04_nearby_gas_station.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/nearby_gas_station-0000002342199206
- `huawei_industry_tree/01_auto/docs/05_call_phone.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/call_phone-0000002342413398
- `huawei_industry_tree/01_auto/docs/06_customize_dashboard.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/customize_dashboard-0000002369250730
- `huawei_industry_tree/01_auto/docs/07_car_maintenance.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/car_maintenance-0000002404899097
