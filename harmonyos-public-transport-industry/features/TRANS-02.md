---
id: TRANS-02
title: Key scenario index for the public transport industry
industry: 06_public_transport
doc: huawei_industry_tree/06_public_transport/docs/02_financial-insurance-v1-6_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/financial-insurance-v1-6_1-0000002293996400
sample: none
kits: []
apis: []
permissions: []
min_api: 24
modules: []
findings: [HW-06-0017]
status: verified-with-fixes
---

## When to use

Do not load this card on its own. It is the industry's scenario index: it
records which feature-level practices Huawei publishes for public transport, so
a router or an agent picking a card knows the full set. Load `TRANS-01` for the
architecture, then the card for the screen being built.

## Feature checklist

Huawei publishes six feature-level practices for this industry:

| Card | Scenario | Sample |
|---|---|---|
| `TRANS-03` | Tap a card to open the ride code (点击卡片跳转乘车码) | `QRcardDemo.zip` |
| `TRANS-04` | Real-time bus arrivals (实时公交服务) | `实时公交服务示例代码.zip` |
| `TRANS-05` | Ride history lookup (乘车记录查询) | `TicketDemo.zip` |
| `TRANS-06` | Add the metro map to the home screen (地铁线路图添加桌面) | `AddShortcutToDesktop.zip` |
| `TRANS-07` | Boarding and alighting reminders (公交上下车提醒) | `BusOnOffNotification.zip` |
| `TRANS-08` | Map rotation lock (地图旋转锁定) | `MapRotationLock.zip` |

Everything else in a transit app - the four-tab shell, city resolution, route
planning, ride-code generation, the card wallet - is covered by the architecture
card `TRANS-01` and its sample project.

## Architecture

Not applicable. The source document is a six-entry link list with no technical
content of its own.

## Implementation steps

None. Follow `TRANS-01` first, then the relevant feature card.

## Verified snippets

None - the source document contains no code.

## Permissions & config

None.

## Constraints

The published feature set skews towards presentation and shortcuts. Fare
payment, NFC transit cards, account top-up, offline ride codes and inter-city
federation have no HQ practice document in this industry, even though
`TRANS-01`'s architecture explicitly plans for per-city ride-code modules.

## Pitfalls

- **`HW-06-0017` — this page is published under the finance and insurance URL
  family.** Its address is `financial-insurance-v1-6_1`, while seventeen of the
  nineteen industries name themselves in the slug
  (`practice-bus-app-architecture-v1_1` would be the matching form here). The
  content is correct and all six links resolve; only the identity is wrong. If
  you are scripting against these documents, do not assume the slug identifies
  the industry.

All six links resolve to the documents they name, and each link's text matches
the `title` frontmatter of its target exactly.

## References

- `huawei_industry_tree/06_public_transport/docs/03_qrcard_demo.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/qrcard_demo-0000002328156469
- `huawei_industry_tree/06_public_transport/docs/04_real_time_bus.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/real_time_bus-0000002300272510
- `huawei_industry_tree/06_public_transport/docs/05_ride_records.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/ride_records-0000002312400232
- `huawei_industry_tree/06_public_transport/docs/06_add_shortcut_to_desktop.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/add_shortcut_to_desktop-0000002368913597
- `huawei_industry_tree/06_public_transport/docs/07_bus_on_off_notification.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/bus_on_off_notification-0000002394404617
- `huawei_industry_tree/06_public_transport/docs/08_map_rotation_lock.md`
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_rotation_lock-0000002396727869
