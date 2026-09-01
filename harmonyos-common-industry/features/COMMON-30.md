---
id: COMMON-30
title: Custom TabBar switch animation - a sliding underline that tracks both the tab list scroll and the content swipe
industry: 19_common_technical_solutions
doc: huawei_industry_tree/19_common_technical_solutions/docs/30_tab_toggle_animation.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_toggle_animation-0000002332006785
sample: huawei_industry_tree/19_common_technical_solutions/downloads/TabbarDemo.zip
kits: ["@kit.ArkUI"]
apis: [Stack, List, ListItem, ListScroller, "Scroller.scrollToIndex", "List.onWillScroll", "List.listDirection", "List.fadingEdge", "List.friction", Tabs, TabsController, "TabsController.changeIndex", TabContent, "Tabs.barHeight", "Tabs.animationDuration", "Tabs.onChange", "Tabs.onAnimationStart", "Tabs.onAnimationEnd", "Tabs.onGestureSwipe", TabsAnimationEvent, "component .onAreaChange", "component .safeAreaPadding", "UIContext.animateTo", LengthMetrics, "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-19-0092, HW-19-0093, HW-19-0094]
status: verified-with-fixes
---

## When to use

Load this card when the built-in `Tabs` bar is not enough and you need a **custom
scrollable tab strip with an underline that slides continuously** - following the
finger while the content is dragged, animating to the target when a tab is
tapped, and staying aligned when the tab strip itself is scrolled.

The hard part is not the animation; it is keeping three independent motions in
agreement: the horizontal `List` that holds the tabs, the `Tabs` content swipe,
and the underline's own position.

## Feature checklist

The application must:

- Build the tab strip as a horizontally scrolling `List`, separate from the
  `Tabs` component.
- Hide the built-in tab bar (`barHeight(0)`) and drive `Tabs` through a
  `TabsController`.
- Overlay the underline on the strip in a `Stack`, positioned by a state-driven
  left margin and width.
- Measure each tab's position and width with `onAreaChange` and cache them.
- Update the underline frame-by-frame while the tab strip scrolls
  (`List.onWillScroll`).
- Update it frame-by-frame while the content is dragged (`Tabs.onGestureSwipe`).
- Animate it to the target when a switch animation starts
  (`Tabs.onAnimationStart`) and snap it when the animation ends.
- Scroll the tab strip so the newly selected tab is visible (`Tabs.onChange` ->
  `Scroller.scrollToIndex`), clamped to a valid index (HW-19-0093).
- Keep all position measurements in one coordinate space (HW-19-0094).

## Architecture

Single-module project (`entry` HAP):

| File | Responsibility |
| --- | --- |
| `entryability/EntryAbility.ets` | window setup; publishes `topRectHeight` / `bottomRectHeight` already converted with `px2vp` |
| `model/FoodInfoModel.ets` | `TabBarItem` and `TAB_BAR_INFO` - the tab names and their content |
| `components/ContentFlowCom.ets` | the content of one tab |
| `components/CardInfoCom.ets` | one content card |
| `utils/Logger.ets` | `hilog` wrapper |
| `pages/TabToggle.ets` | the whole mechanism |

**Component tree.**

```
Column
├─ Row (page title)
├─ Stack(TopStart)                  <- the tab strip and the underline share this origin
│   ├─ Row
│   │   ├─ List(scroller)           <- horizontal, one ListItem per tab
│   │   └─ moreTab()                <- a decorative "more" affordance
│   └─ Column                       <- the underline, positioned by margin.left
└─ Tabs(controller, barHeight(0))   <- content only; the bar is the List above
```

**State.**

| Field | Role |
| --- | --- |
| `curIndex` | the selected tab; drives the tab text colour |
| `indicatorLeftMargin`, `indicatorWidth` | the underline's geometry |
| `tabPosWidthMap: Map<number, Record<string, number>>` | per-tab measured `left` and `width`, filled by `onAreaChange` |
| `isExpanding` | whether the strip has scrolled far enough to fade its edge and hide the "more" affordance |
| `controller: TabsController`, `scroller: ListScroller` | the two imperative handles |

**Four update paths for the underline**, and each answers a different question:

| Trigger | What happened | What the underline does |
| --- | --- | --- |
| `List.onWillScroll(xOffset)` | the user scrolled the tab strip | translate by `-xOffset`, frame by frame |
| `Tabs.onGestureSwipe(index)` | the user is dragging the content | jump to that tab's cached geometry |
| `Tabs.onAnimationStart(index, targetIndex)` | a switch animation began | `animateTo` the target geometry over `animationDuration` |
| `Tabs.onAnimationEnd(index)` | the animation finished | snap to the final geometry with duration 0 |

The `onWillScroll` path is why the underline lives in the `Stack` rather than
inside the `List`: it must be positioned in the strip's coordinate space but not
scroll with it.

**Measurement** happens in `onAreaChange` on each tab's `Text`. The first
measurement of the selected tab seeds `indicatorLeftMargin` / `indicatorWidth`;
every measurement caches `left` and `width` into `tabPosWidthMap` for
`getTextInfo(index)` to look up later.

## Implementation steps

1. **Split the bar from the content.** A horizontal `List` for the tabs, `Tabs`
   with `barHeight(0)` for the content, joined by a `TabsController`.
2. **Wrap the strip and the underline in a `Stack({ alignContent:
   Alignment.TopStart })`** so the underline's `margin.left` is measured from the
   strip's left edge.
3. **Measure every tab** in `onAreaChange`, caching `left` and `width` per index -
   in the *same* coordinate space the margin uses (HW-19-0094).
4. **Seed the underline** from the selected tab's first measurement, guarded so
   it only runs while the geometry is still zero.
5. **Follow the strip scroll**: `onWillScroll((xOffset) => { this.indicatorLeftMargin -= xOffset; })`.
6. **Follow the content drag**: `onGestureSwipe((index) => { ... })`, taking the
   selected index from the callback argument (HW-19-0092).
7. **Animate on switch**: `onAnimationStart` -> `animateTo(duration, left,
   width)`; `onAnimationEnd` -> the same with duration 0.
8. **Keep the selected tab visible**: `onChange` -> `scrollToIndex`, clamped
   (HW-19-0093).
9. **Tap to switch**: each tab's `onClick` calls
   `this.controller.changeIndex(index)`, which drives the `Tabs` and therefore
   all of the callbacks above.

## Verified snippets

All snippets below come from the sample project, not from the document.

### Measuring the tabs

`TabbarDemo.zip#TabbarDemo/entry/src/main/ets/pages/TabToggle.ets`

```ts
// 自定义tab页签
@Builder
tabBarBuilder(tabName: ResourceStr, index: number) {
  Row() {
    Text(tabName)
      .fontSize($r('app.integer.tab_font_size'))
      .fontColor(index === this.curIndex ? $r('app.color.tab_selected_color') : $r('app.color.tab_unselected_color'))
      .id(`${index}`)
      .onAreaChange((oldValue, newValue) => {
        let width = Number.parseFloat(newValue.width.toString());
        let globalX = Number.parseFloat(newValue.globalPosition.x!.toString());
        // 初始位置：底部指示器偏移量计算
        if (this.curIndex === index && (this.indicatorLeftMargin === 0 || this.indicatorWidth === 0)) {
          if (newValue.position.x) {
            let positionX = Number.parseFloat(newValue.position.x.toString());
            this.indicatorLeftMargin = Number.isNaN(positionX) ? 0 : positionX;
          }
          let width = Number.parseFloat(newValue.width.toString());
          this.indicatorWidth = Number.isNaN(width) ? 0 : width;
        }
        this.tabPosWidthMap.set(index, { 'left': globalX, 'width': width });   // FIX (HW-19-0094)
      });
  }
  .justifyContent(FlexAlign.Center)
  .width($r('app.integer.tab_width'))
  .height($r('app.integer.tab_height'))
  .onClick(() => {
    this.controller.changeIndex(index);
  });
}
```

### The strip and the underline in one Stack

`TabbarDemo.zip#TabbarDemo/entry/src/main/ets/pages/TabToggle.ets`

```ts
Stack({ alignContent: Alignment.TopStart }) {
  // 页签
  Row({ space: 8 }) {
    List({ space: 16, initialIndex: 0, scroller: this.scroller }) {
      ForEach(TAB_BAR_INFO, (item: TabBarItem, index: number) => {
        ListItem() {
          this.tabBarBuilder(item.name, index);
        };
      }, (item: TabBarItem, index) => JSON.stringify(item) + '_' + index);
    }
    .listDirection(Axis.Horizontal)
    .height($r('app.integer.tab_height'))
    .layoutWeight(1)
    .friction(0.6)
    .alignListItem(ListItemAlign.Start)
    .scrollBar(BarState.Off)
    .fadingEdge(this.isExpanding ? true : false, { fadingEdgeLength: LengthMetrics.vp(80) })
    .onWillScroll((xOffset: number) => {
      this.indicatorLeftMargin -= xOffset;
      if (this.curIndex >= 2) {
        this.isExpanding = true;
      }
    });

    this.moreTab();
  }.width('100%');

  // 下划线
  Column()
    .width(this.indicatorWidth)
    .height($r('app.integer.tab_indicator_height'))
    .borderRadius($r('app.integer.tab_indicator_border_radius'))
    .margin({ left: this.indicatorLeftMargin, top: $r('app.integer.tab_indicator_offset') })
    .backgroundColor('#0A59F7');
}
.width('100%')
.height($r('app.integer.tabbar_height'));
```

### The four Tabs callbacks

`TabbarDemo.zip#TabbarDemo/entry/src/main/ets/pages/TabToggle.ets`

```ts
Tabs({ barPosition: BarPosition.Start, controller: this.controller }) {
  ForEach(TAB_BAR_INFO, (item: TabBarItem) => {
    TabContent() {
      Scroll() {
        ContentFlowCom({ foods: item.contents });
      }.scrollBar(BarState.Off);
    }.backgroundColor($r('app.color.page_background_color'));
  }, (item: string, index) => item + '_' + index);
}
.width('100%')
.layoutWeight(1)
.barHeight(0)
.animationDuration(this.animationDuration)
.onChange((index: number) => {
  Logger.info('tab change');
  this.curIndex = index;
  this.scroller.scrollToIndex(index - 1, true);      // FIX (HW-19-0093): clamp to >= 0
})
.onAnimationStart((index: number, targetIndex: number) => {
  // 切换动画开始时触发该回调,下划线跟着页面一起滑动
  this.curIndex = targetIndex;
  let targetIndexInfo = this.getTextInfo(targetIndex);
  this.startAnimateTo(this.animationDuration, targetIndexInfo.left, targetIndexInfo.width);
})
.onAnimationEnd((index: number) => {
  // 切换动画结束时触发该回调,下划线动画停止。
  let currentIndicatorInfo = this.getTextInfo(index);
  this.startAnimateTo(0, currentIndicatorInfo.left, currentIndicatorInfo.width);
})
.onGestureSwipe((index: number) => {
  // 在页面跟手滑动过程中,逐帧触发该回调。
  let currentIndicatorInfo = this.getTextInfo(index);
  this.curIndex = currentIndicatorInfo.index;        // FIX (HW-19-0092): use index
  this.indicatorLeftMargin = currentIndicatorInfo.left;
  this.indicatorWidth = currentIndicatorInfo.width;
});
```

### The animation helper and the geometry lookup

`TabbarDemo.zip#TabbarDemo/entry/src/main/ets/pages/TabToggle.ets`

```ts
private startAnimateTo(duration: number, leftMargin: number, width: number) {
  this.getUIContext().animateTo({
    duration: duration, // 动画时长
    curve: Curve.Linear, // 动画曲线
    iterations: 1, // 播放次数
    playMode: PlayMode.Normal, // 动画模式
    onFinish: () => {
      Logger.info('play end');
    }
  }, () => {
    this.indicatorLeftMargin = leftMargin;
    this.indicatorWidth = width;
  });
}

private getTextInfo(keyId: number): Record<string, number> {
  let posWidth = this.tabPosWidthMap.get(keyId);
  if (posWidth) {
    return posWidth;
  }
  return { 'left': 0, 'width': 0 };
}
```

Note that `getTextInfo` returns exactly two keys - which is what makes
`currentIndicatorInfo.index` in `onGestureSwipe` undefined (HW-19-0092).

## Permissions & config

**No permissions are required** and none are declared - the feature is entirely
layout and animation.

`TabbarDemo.zip#TabbarDemo/entry/src/main/module.json5` declares the usual single
`EntryAbility` with the home skill and `"deviceTypes": ["phone", "tablet",
"2in1"]`. There is no `requestPermissions` block and no `routerMap`.

The page consumes `topRectHeight` through `@StorageProp` and applies it with
`.safeAreaPadding({ top: this.topRectHeight })`; the ability already converted it
with `px2vp` before publishing, so no conversion happens in the page - unlike
several other samples in this industry which publish raw px.

## Constraints

- **API level.** API Version 20 Release or later, HarmonyOS 6.0.0 Release SDK or
  later, DevEco Studio 6.0.0 Release or later.
- **`scrollToIndex` ignores out-of-range values.** "If the value set is a
  negative value or greater than the maximum index of the items in the container,
  the value is deemed abnormal, and no scrolling will be performed" - silently.
- **A `Scroller` cannot be shared** between scrollable components; this project
  has one `ListScroller` for the strip and lets `Tabs` manage its own content
  scrolling.
- **`onAreaChange` fires on layout, not on demand.** `tabPosWidthMap` is empty
  until each tab has been laid out at least once, which is why `getTextInfo` has
  a zero fallback - and why the underline is seeded from the first measurement of
  the selected tab rather than in `aboutToAppear`.
- **`onWillScroll` gives a delta, not a position.** The underline accumulates it,
  so any missed or duplicated call drifts; the `onAnimationEnd` snap is what
  re-anchors it.
- **`barHeight(0)` is what hides the built-in bar** while keeping `Tabs`'
  gestures and callbacks; removing it would show two bars.
- **Devices.** `phone`, `tablet`, `2in1`.

## Pitfalls

- **`currentIndicatorInfo.index` does not exist, which is incorrect.**
  `getTextInfo` returns only `left` and `width`, so `curIndex` becomes
  `undefined` on every swipe frame - no tab renders as selected during the drag,
  and the `curIndex >= 2` guard in `onWillScroll` stops working. Use the
  callback's `index`. The document's snippet has the same line. (HW-19-0092)
- **`scrollToIndex(index - 1, true)` passes `-1` for the first tab, which is
  incorrect.** The reference treats a negative index as abnormal and performs no
  scrolling, so returning to the first tab leaves the strip where it was. Clamp
  with `Math.max(index - 1, 0)`. (HW-19-0093)
- **The underline offset mixes `position.x` and `globalPosition.x`.** It works
  only because the strip sits at the window's left edge; any ancestor inset,
  split layout or RTL mirror breaks it. Use one coordinate space. (HW-19-0094)
- **`isExpanding` is one-way.** Once the strip has been scrolled with
  `curIndex >= 2` it is set to `true` and never set back, so the fading edge and
  the hidden "more" affordance persist for the rest of the session.
- **The two `ForEach` key generators differ.** The strip uses
  `JSON.stringify(item) + '_' + index` and the content uses `item + '_' + index`
  with the item typed as `string` while `TAB_BAR_INFO` holds `TabBarItem`
  objects - so the content key is `'[object Object]_0'`, `'[object Object]_1'`,
  unique only because of the index suffix. Key on a stable field instead.
- **`indicatorWidth` starts at 0**, so the underline is invisible until the
  selected tab has been measured. Do not add an entry animation that assumes a
  non-zero starting width.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` -
  `Scroller.scrollToIndex`, its parameters and the negative-index rule.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-tabs -
  `Tabs`, `TabsController.changeIndex`, `barHeight`, `animationDuration`,
  `onChange`, `onAnimationStart`, `onAnimationEnd`, `onGestureSwipe` and
  `TabsAnimationEvent`.
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` -
  `List`, `listDirection`, `scroller`, `space`, `fadingEdge`, `friction` and
  `onWillScroll`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-stack -
  `Stack` and `alignContent`, which fixes the underline's coordinate origin.
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` -
  `UIContext.animateTo`.
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-universal-component-area-change-event -
  `onAreaChange`, `position` vs `globalPosition`.
- https://developer.huawei.com/consumer/cn/doc/architecture-guides/tab_toggle_animation-0000002332006785
