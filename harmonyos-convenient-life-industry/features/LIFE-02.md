---
id: LIFE-02
title: Convenient-life key-scenario catalog - the 28 scenario documents and their feature cards
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/02_practice-convenient-life-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1_1-0000002270511797
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

Load this card when you know **what** a convenient-life app has to do but not
**which** feature card covers it. This document is the index page of the
industry: 关键场景示例 ("key scenario examples"), a flat list of 28 links, one per
scenario document. It contains no prose, no architecture, and no code.

Use `LIFE-01` for the application shell (module split, router, immersive layout,
login gate). Use this card only to route to the right scenario card.

## Feature checklist

Nothing to implement. This is a navigation document. Its one job is to be a
complete and correct map of the industry's scenario documents, and it is:
all 28 link titles match the `title:` frontmatter of their target documents
exactly, and the 28 links cover documents 03 through 30 with nothing missing and
nothing extra.

## Architecture

The industry's 31 documents split into three architecture/overview documents and
28 scenario documents:

```
01  practice-convenient-life-app-architecture-v1     LIFE-01   the app shell + sample framework code
02  practice-convenient-life-app-architecture-v1_1   LIFE-02   THIS CARD - the scenario index
03..30                                               LIFE-03..LIFE-30   one scenario each
31  practice-convenient-life-app-architecture-v1_2   LIFE-31   FAQ, redirected to HarmonyOS FAQ
```

Each of the 28 scenario documents ships its own sample project zip, except
`12_perpetual_calendar.md`, which has none.

## Implementation steps

Not applicable. To pick a scenario card, match your requirement against the
catalog below, then load that card.

## Verified snippets

None. The document contains no code.

## Permissions & config

None.

## Constraints

The catalog is the full set of scenarios Huawei publishes for this industry.
Anything a convenient-life app needs beyond these 28 (payments, account systems,
messaging, background tasks) is not covered here - look in
`19_common_technical_solutions` or another industry.

## Pitfalls

No findings were recorded for this document. Two things are worth knowing
anyway:

- **This index is not a dependency graph.** The order of the 28 links is
  publication order, not a build order; the scenarios are independent of each
  other and of `LIFE-01`.
- **Two documents in the industry are not in this index** - `LIFE-01` (the app
  architecture and framework code) and `LIFE-31` (the FAQ redirect). Do not
  treat the 28 entries as the complete industry.

## Catalog

| # | Card | Chinese title | English gloss | Sample zip |
| --- | --- | --- | --- | --- |
| 03 | LIFE-03 | 车牌号编辑 | Licence-plate keyboard | VehicleKeyboard.zip |
| 04 | LIFE-04 | 电影选座 | Cinema seat selection | CanvasCinema.zip |
| 05 | LIFE-05 | 级联菜单选择 | Cascading menu selection | CascadingMenuSelection.zip |
| 06 | LIFE-06 | 待办事项便贴 | To-do sticky notes | StickyNote.zip |
| 07 | LIFE-07 | 查看账单收支明细 | Collapsible bill detail list | CollapseList.zip |
| 08 | LIFE-08 | 新建日程管理 | Creating a calendar event | Schedule.zip |
| 09 | LIFE-09 | 功能卡片自动循环播放 | Auto-looping feature cards | LoopScroll.zip |
| 10 | LIFE-10 | 密码保险箱自定义 | Custom password vault | AssetVerification.zip |
| 11 | LIFE-11 | 双指捏合缩放卡片 | Pinch-to-zoom card | CardPinchScale.zip |
| 12 | LIFE-12 | 日历常用功能及黄历查询 | Calendar and almanac lookup | none |
| 13 | LIFE-13 | 三日视图日历及日程管理 | Three-day calendar view | ThreeDayViewCalendar.zip |
| 14 | LIFE-14 | 地图页面半模态交互 | Half-modal sheet over a map | MapBindSheet.zip |
| 15 | LIFE-15 | 证件类型选择 | ID-document type picker | IDType4UserRegistration.zip |
| 16 | LIFE-16 | 地图起始点自定义路线绘制 | Custom route drawing on a map | MapDirectionDemo.zip |
| 17 | LIFE-17 | 双列表联动切换 | Two linked lists | DualListLinkage.zip |
| 18 | LIFE-18 | 日历周视图、月视图切换 | Week/month calendar switching | CalendarSwiper.zip |
| 19 | LIFE-19 | 快递地址文本识别 | Parcel address text parsing | AddressExtraction.zip |
| 20 | LIFE-20 | 地图旋转与移动 | Map rotation and panning | ConfirmDirectionInMap.zip |
| 21 | LIFE-21 | 快递地址图片识别 | Parcel address image OCR | CourierAddressIdentification.zip |
| 22 | LIFE-22 | 身份证扫描识别 | ID card scanning | ScanIdCard.zip |
| 23 | LIFE-23 | 卡证信息识别 | Card/certificate recognition | 卡证信息识别示例代码.zip |
| 24 | LIFE-24 | 上门服务预约提醒 | Home-service appointment reminder | AppointmentServiceRemind.zip |
| 25 | LIFE-25 | 通勤路径规划 | Commute route planning | CommutingCalculation.zip |
| 26 | LIFE-26 | 获取附近网点位置并一键导航 | Nearby outlets with one-tap navigation | ListOfNearbyOutlets.zip |
| 27 | LIFE-27 | 房源图片预览 | Property photo preview | demo_HouseView.zip |
| 28 | LIFE-28 | H5页面获取通讯录电话号码 | H5 page reading a contact number | H5RechargePlateform.zip |
| 29 | LIFE-29 | 在地图上设置和显示指定位置的覆盖范围 | Coverage area on a map | SetCoverageArea.zip |
| 30 | LIFE-30 | taskpool批量图片加载 | Batch image loading with taskpool | ThumbnailTaskpool.zip |

## References

- `huawei_industry_tree/02_convenient_life/docs/02_practice-convenient-life-app-architecture-v1_1.md` - the index document itself
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-convenient-life-app-architecture-v1_1-0000002270511797
- `huawei_industry_tree/02_convenient_life/docs/` - the 28 scenario documents the index links to
- `LIFE-01` - the application shell these scenarios drop into
