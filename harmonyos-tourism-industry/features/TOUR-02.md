---
id: TOUR-02
title: Tourism key-scenario catalog - the ten sample scenarios of the tourist park practice
industry: 09_tourism
doc: huawei_industry_tree/09_tourism/docs/02_practice-tourist-park-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-tourist-park-app-architecture-v1_1-0000002270258501
sample: NO ZIP
kits: []
apis: []
permissions: []
min_api: 24
modules: []
findings: []
status: verified
---

## When to use

This is the **index page** of the tourist park practice, not a capability. It
is the second of the three architecture-level documents in this industry:

| Document | Card | Role |
| --- | --- | --- |
| 旅游住宿应用案例 (tourism and accommodation app case) | `TOUR-01` | the architecture: layers, modules, park map |
| 关键场景示例 (key scenario examples) | `TOUR-02` (this card) | the catalog of the ten scenario samples |
| 行业常见问题 (industry FAQ) | `TOUR-13` | a redirect notice, no content |

Load it only to find **which scenario card to open**. Each entry below is a
standalone document with its own downloadable sample; the review of that
sample lives in the card named in the last column.

## Feature checklist

The document is a plain list of ten links and nothing else - no prose, no
code, no images. All ten targets are present in this corpus, and the sidebar
tree carries exactly these ten children under 关键场景示例, so the catalog is
complete.

| # | Scenario (document title) | Sample | Card |
| --- | --- | --- | --- |
| 1 | 气泡提醒开启定位 - bubble prompt to enable location | `LocationPermissionPrompt.zip` | `TOUR-03` |
| 2 | 选择日期范围 - pick a date range | `DateSelection.zip` | `TOUR-04` |
| 3 | animateTo实现地址左右交换动画 - swap origin and destination with animateTo | `AddressExchange.zip` | `TOUR-05` |
| 4 | 获取目的地位置及周边配套地图 - destination location and surrounding facilities map | `DestinationMap.zip` | `TOUR-06` |
| 5 | 地图指定位置名称标记 - name label marker at a map position | `MapMarker.zip` | `TOUR-07` |
| 6 | 酒店入住评价 - hotel stay review | `UserReview.zip` | `TOUR-08` |
| 7 | 景点语音讲解 - attraction audio guide | `景点语音讲解示例代码.zip` | `TOUR-09` |
| 8 | 酒店订单列表 - hotel order list | `TravelCheckinOrder.zip` | `TOUR-10` |
| 9 | 核对用户信息 - confirm traveller details | `CheckInformation.zip` | `TOUR-11` |
| 10 | 地图添加标记并收藏地址 - add a marker and save the address | `AddAndCollectMapMarker.zip` | `TOUR-12` |

## Architecture

No architecture of its own. The ten scenarios group into four clusters, which
is the useful way to read the catalog:

```
Map and location          TOUR-03  permission prompt (bubble, not a dialog)
                          TOUR-06  destination + surrounding POIs
                          TOUR-07  named position marker
                          TOUR-12  add marker, persist as a favourite
                          -> and TOUR-01 for the park map itself

Booking funnel            TOUR-04  date range picker (check-in / check-out)
                          TOUR-05  origin <-> destination swap animation
                          TOUR-11  traveller details confirmation
                          TOUR-10  order list

Post-stay                 TOUR-08  review with rating and images

On site                   TOUR-09  attraction audio guide
```

The booking funnel cluster is close to chronological: pick dates, set the
route, confirm who is travelling, then look the order up again afterwards.
Reading `TOUR-04`, `TOUR-05`, `TOUR-11` and `TOUR-10` in that order gives the
whole reservation flow.

## Implementation steps

Nothing to implement. Pick a cluster, open the card.

## Verified snippets

None - the document contains no code.

## Permissions & config

None declared by this document. Permissions belong to the individual scenario
cards; the location cluster (`TOUR-03`, `TOUR-06`, `TOUR-07`, `TOUR-12`)
carries `ohos.permission.LOCATION` and `ohos.permission.APPROXIMATELY_LOCATION`,
and everything that draws a Huawei map also needs `ohos.permission.INTERNET`
plus the AGC `client_id` metadata described in `TOUR-01`.

## Constraints

- The catalog is a navigation aid, not a compatibility statement. The API
  version and device support of each scenario are declared in its own
  document and recorded on its own card; they are not uniform across the ten.

## Pitfalls

None. The list is complete and every target resolves to a document and a
sample in this corpus.

## References

- `huawei_industry_tree/09_tourism/tree.json` - the sidebar tree this page mirrors
- `TOUR-01` - the architecture the ten scenarios plug into
- `TOUR-13` - the third architecture-level page, a redirect to the HarmonyOS FAQ
