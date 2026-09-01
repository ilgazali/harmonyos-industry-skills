---
id: FOOD-03
title: Key-scenario index - the two food scenarios that sit on top of the app templates (store navigation, custom refresh)
industry: 17_food
doc: huawei_industry_tree/17_food/docs/03_practice-food-app-architecture-v1_1.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-architecture-v1_1-0000002237493428
sample: none
kits: []
apis: []
permissions: []
min_api: n/a
modules: []
findings: []
status: verified
---

## When to use

Load this card when you have the shape of a food app from `FOOD-01` or
`FOOD-02` and are looking for **the scenario-level pieces that plug into it**.
This document is an index page, nothing more: two links, no prose, no code, no
sample of its own. Its value is the editorial judgement behind it - out of
everything a food app does, Huawei singles out two scenarios as worth a
standalone sample.

The two are **store address and route navigation** (`FOOD-04`) and **custom
pull-to-refresh and pull-up load** (`FOOD-05`). Both are "last mile" features:
each one attaches to a screen the templates already build - the store detail
page and the recommendation feed - and each depends on a capability the
templates deliberately leave unfinished.

Use this card to decide which of the two you need, and to understand how they
join the templates. For the implementations themselves, go to `FOOD-04` and
`FOOD-05`, which are both backed by real zips.

## Feature checklist

What this page promises, and nothing else:

- A link to 商家地址路线导航 (store address route navigation) - static map
  thumbnail of a merchant location, tap-through to the POI in Petal Maps, and
  buttons for route planning and turn-by-turn navigation.
- A link to 下拉刷新和上拉加载效果自定义 (custom pull-to-refresh and pull-up
  load) - a `Refresh` with a custom builder over a `WaterFlow`, appending a page
  of items on `onReachEnd`.
- Both links resolve; both targets ship a downloadable sample.

The page carries no architecture narrative, no constraints section and no
permission list. Anything you need about environment or permissions is in the
two target documents.

## Architecture

**No sample project.** The document is two bullet lines under a heading; there
is nothing to lay out. What matters is where each linked scenario attaches:

```
FOOD-01 / FOOD-02  (the app template)
   │
   ├── store list / store detail  ─────────►  FOOD-04  商家地址路线导航
   │      FOOD-01 stops at a MapComponent          staticMap thumbnail
   │      with markers and a Petal Maps Want       petalMaps.openMapPoiDetail
   │                                               openMapNavi / openMapRoutePlan
   │
   └── recommendation feed  ──────────────►  FOOD-05  下拉刷新和上拉加载
          FOOD-02 renders a WaterFlow over          Refresh({builder}) + reload()
          a LazyDataSource, loaded once            onReachEnd + addLastItem()
```

**The design decision worth copying** is the split itself. Both scenarios were
pulled *out* of the templates rather than bolted on. `FOOD-01` ends its map
story at "show markers, hand off to Petal Maps with a `Want`"; `FOOD-02` ends
its feed story at "fill the data source once". Each scenario sample then owns a
single interaction end to end, in an `entry`-only project small enough to read
in one sitting. That is the right granularity for this kind of catalogue: the
template teaches structure, the scenario teaches one API surface, and neither
has to carry the other's complexity.

The cost is that the join is left to the reader. Neither scenario sample uses
the router, the HAR layering or the state model of the templates, so wiring one
in means re-hosting its component inside a feature HAR and replacing its
hardcoded coordinates or its counter-based data source with real data.

## Implementation steps

1. **Decide which scenario you actually need.** Route navigation matters when
   the user must physically reach a branch (dine-in, pickup); custom refresh
   matters when the feed is the product (recipes, recommendations).
2. **For navigation, start from `FOOD-04`.** Enable the map service for your app
   in AppGallery Connect, complete manual signing, and declare
   `ohos.permission.INTERNET` - the sample itself declares no permissions at all
   (`HW-17-0022`), so do not take its `module.json5` as the reference.
3. **Pass the store's own POI id into the map component,** rather than the
   hardcoded one the sample carries (`HW-17-0023`), and put a `catch` on every
   Map Kit call - "Map permission is not enabled" (`1002600004`) is the failure
   you will hit first (`HW-17-0024`).
4. **For the feed, start from `FOOD-05`.** Wrap the scrollable in
   `Refresh({ refreshing: $$this.refreshing, builder: this.refreshBuilder() })`
   and make `reload()` actually rebuild the data set - the sample's re-notifies
   an unchanged array, so pulling down visibly does nothing (`HW-17-0027`).
5. **Guard `onReachEnd` with an `isLoading` flag** so one bounce at the bottom
   appends one page, not three.
6. **Re-host, do not copy wholesale.** Both samples are single-`entry` projects.
   Inside a template, the map component belongs in the map feature HAR and the
   refresh wrapper around the existing `WaterFlow` in the feed component; the
   data source in particular should be the template's own.

## Verified snippets

This page ships no sample, so nothing here is compile-verified. The two snippets
below are the ones the **linked** documents lead with - shown so you can tell at
a glance which scenario you are after. The verified, zip-backed forms are in
`FOOD-04` and `FOOD-05`.

**Route navigation** (from `huawei_industry_tree/17_food/docs/04_map_navigation.md` — no sample shipped with this page; not compile-verified)

```typescript
let params: petalMaps.NaviParams = {
  destinationPosition: {
    // 经纬度参数 (latitude / longitude)
  },
  // vehicleType:0，选择导航出行方式 (travel mode)
};
await petalMaps.openMapNavi(this.context.getHostContext(), params);
```

`petalMaps` starts the installed Petal Maps app rather than drawing a route
in-app, so the whole feature is a `Want`-shaped handoff: you supply a
destination and a travel mode and the system app takes over. That is why the
companion piece is `staticMap.getMapImage` - a picture of the destination is
enough for the calling screen, and no `MapComponent` needs to be instantiated
just to show where a restaurant is. Contrast with `FOOD-01`, which does host a
live `MapComponent` because the user is choosing between branches on it.

**Custom refresh** (from `huawei_industry_tree/17_food/docs/05_custom_refresh.md` — no sample shipped with this page; not compile-verified)

```typescript
Refresh({ refreshing: $$this.refreshing, builder: this.refreshBuilder() }) {
}
.onRefreshing(() => {
  setTimeout(() => {
    this.dataSource.reload();
    this.refreshing = false;
  }, Constants.DELAY_TIME);
});
```

Two things carry this design. `$$this.refreshing` is a two-way binding:
`Refresh` sets it to `true` when the user pulls far enough, and the handler must
set it back to `false` when the work is done - forgetting that leaves the
spinner up forever. And `builder` replaces the default indicator entirely, which
is the only reason this scenario needs a sample at all; the plain `Refresh` is a
one-liner.

The load-more half is not a `Refresh` feature: it is `onReachEnd` on the
`WaterFlow`, guarded by an `isLoading` flag, appending items through the same
`IDataSource` the list already renders.

## Constraints

- Documentation-only page. Verification level here is lower than for the
  zip-backed cards: nothing on it was compiled or run, because there is nothing
  to compile.
- The page states no environment requirements of its own. Both targets require
  API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or later and
  DevEco Studio 6.0.0 Release or later - a lower baseline than `FOOD-02`'s API
  24, so a project built on the recipe template cannot assume the scenario
  samples were validated against it.
- The index is not exhaustive and is not stable: it lists two scenarios today
  where sibling industries list five or more, and scenario pages are added and
  renumbered without notice.

## Pitfalls

- No defects were found in this document itself. Its two links resolve and match
  the documents crawled as `04_map_navigation.md` and `05_custom_refresh.md`.
- The defects that matter are in the pages it points at, and they are recorded
  on those cards: `HW-17-0022` (the navigation sample declares no permissions at
  all), `HW-17-0023` (a hardcoded POI id in a component advertised as reusable),
  `HW-17-0024` (no error handling on any Map Kit call), `HW-17-0025` (a
  `UIContext` passed as `@Prop`), `HW-17-0026` (unclamped scroll-driven opacity)
  and `HW-17-0027` (pull-to-refresh that refreshes nothing). Read `FOOD-04` and
  `FOOD-05` before lifting either sample.

## References

- `huawei_industry_tree/17_food/docs/03_practice-food-app-architecture-v1_1.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/practice-food-app-architecture-v1_1-0000002237493428
- `huawei_industry_tree/17_food/docs/04_map_navigation.md` - 商家地址路线导航
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/map_navigation-0000002248154336
- `huawei_industry_tree/17_food/docs/05_custom_refresh.md` - 下拉刷新和上拉加载效果自定义
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_refresh-0000002331948181
- `documentation/harmonyos-references/02_application-framework/ts-container-refresh.md` - `Refresh`, `refreshing`, custom `builder`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-refresh
- `documentation/harmonyos-references/02_application-framework/ts-container-waterflow.md` - `WaterFlow` and `onReachEnd`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-waterflow
- `FOOD-04` - the verified store-navigation implementation
- `FOOD-05` - the verified custom-refresh implementation
- `FOOD-01`, `FOOD-02` - the templates these two scenarios attach to
