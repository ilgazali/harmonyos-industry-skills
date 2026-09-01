---
id: SPORT-02
title: Sports and health key-scenario catalog - the twelve sample scenarios
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/02_practice-sports-health-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-sports-health-app-architecture-v1_1-0000002232088872
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

This is the **index page** of the sports and health practice. It is one of
three architecture-level documents in this industry:

| Document | Card | Role |
| --- | --- | --- |
| 运动健康应用案例 (sports and health app case) | `SPORT-01` | the architecture: layers, HARs, BLE, pedometer |
| 关键场景示例 (key scenario examples) | `SPORT-02` (this card) | the catalog of the twelve scenario samples |
| 行业常见问题 (industry FAQ) | `SPORT-15` | a redirect notice, no content |

Load it only to find **which scenario card to open**. Twelve entries makes
this the largest scenario set of any industry reviewed so far.

## Feature checklist

The document is a plain list of twelve links and nothing else - no prose, no
code, no images. All twelve targets are present in this corpus, and the
sidebar tree carries exactly these twelve children under 关键场景示例, so the
catalog is complete.

| # | Scenario (document title) | Sample | Card |
| --- | --- | --- | --- |
| 1 | 运动开始、结束交互动画 - start and stop interaction animation | `IndoorRun.zip` | `SPORT-03` |
| 2 | 添加运动计划日历提醒 - add a workout plan to the calendar | `CalendarScheduleEvents.zip` | `SPORT-04` |
| 3 | 周期数据图表绘制 - periodic data charts | `PeriodChart.zip` | `SPORT-05` |
| 4 | 比赛计分器 - match scorer | `MatchScorer.zip` | `SPORT-06` |
| 5 | 健身记录三环效果 - the three activity rings | `SportThreeRing.zip` | `SPORT-07` |
| 6 | 最大化显示完整路线 - fit a whole route on screen | `MaxDisplayOfRoutes.zip` | `SPORT-08` |
| 7 | 户外活动组团信息发布 - publish an outdoor group activity | `OutdoorSports.zip` | `SPORT-09` |
| 8 | 基于Polyline实现比赛晋级图 - tournament bracket with Polyline | `KnockoutMatchList.zip` | `SPORT-10` |
| 9 | 实时定位绘制运动轨迹 - draw a live GPS track | `MotionTrajectory.zip` | `SPORT-11` |
| 10 | 运动321倒计时动效 - the 3-2-1 countdown animation | `CountdownToRun.zip` | `SPORT-12` |
| 11 | 组合手势调整运动看板卡片顺序 - reorder dashboard cards by gesture | `CustomExerciseDashboard.zip` | `SPORT-13` |
| 12 | scanBarcode实现扫描药品条形码获取信息 - scan a medicine barcode | `ScanToAddMedication.zip` | `SPORT-14` |

## Architecture

No architecture of its own. The twelve scenarios group into four clusters,
which is the useful way to read the catalog:

```
Maps and location      SPORT-08  fit a whole recorded route into the viewport
                       SPORT-11  draw the track live as the user moves

Charts drawn by hand   SPORT-05  periodic data over a time axis
                       SPORT-07  the three concentric activity rings
                       SPORT-10  a knockout bracket built from Polyline

Motion and animation   SPORT-03  the start and stop transition of a workout
                       SPORT-12  the 3-2-1 countdown before it begins
                       SPORT-13  reordering dashboard cards with a combined gesture

System integration     SPORT-04  writing a workout plan into the system calendar
                       SPORT-06  the scorer - state and interaction only
                       SPORT-09  publishing a group activity
                       SPORT-14  scanning a medicine barcode
```

The two map scenarios pair naturally: `SPORT-11` records the track and
`SPORT-08` frames it afterwards. The three chart scenarios are the visual
vocabulary a fitness app needs and none of them uses a chart library - all
three are drawn from primitives.

## Implementation steps

Nothing to implement. Pick a cluster, open the card.

## Verified snippets

None - the document contains no code.

## Permissions & config

None declared by this document. Permissions belong to the individual scenario
cards; expect `ohos.permission.LOCATION` and
`ohos.permission.APPROXIMATELY_LOCATION` in the map cluster
(`SPORT-08`, `SPORT-11`), calendar permissions in `SPORT-04`, and
`ohos.permission.CAMERA` in `SPORT-14`.

## Constraints

- The catalog is a navigation aid, not a compatibility statement. Each
  scenario declares its own API version, SDK and DevEco Studio floor in its
  own document; they are not guaranteed uniform across the twelve.
- The catalog covers scenarios only. The two capabilities that define this
  industry at the app level - BLE accessory pairing and the pedometer - are in
  `SPORT-01`, not here.

## Pitfalls

None. The list is complete and every target resolves to a document and a
sample in this corpus.

## References

- `huawei_industry_tree/03_sports_health/tree.json` - the sidebar tree this page mirrors
- `SPORT-01` - the architecture the twelve scenarios plug into
- `SPORT-15` - the third architecture-level page, a redirect to the HarmonyOS FAQ
