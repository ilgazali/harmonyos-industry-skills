---
id: NEWS-08
title: Back to top on a long list - a floating button over scrollTo, plus the system status-bar double tap
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/08_move_to_top.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/move_to_top-0000002283118621
sample: huawei_industry_tree/11_news_reading/downloads/MoveToTop.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window]
permissions: []
min_api: 16
modules: [entry (entry)]
findings: [HW-11-0031, HW-11-0037, HW-11-0038, HW-11-0039]
status: verified
---

## When to use

Load this card when a feed is **long enough that scrolling back by hand is a
chore** - a news list, a comment thread, a product grid, a chat archive. The
feature is three ways to reach the top, and the reason to implement all three
is that they cover different users: a visible button for people who look for
one, the system status-bar double tap for people who already know it, and the
scroll itself for everyone else.

The mechanism is small: accumulate `onDidScroll` offsets into a number, show
the button once that number passes a threshold, and animate
`scroller.scrollTo({ yOffset: 0, animation: {...} })` on tap. The one line
that costs nothing and is most often missed is `.backToTop(true)` on the
scrollable - it opts the list into the platform's status-bar gesture.

It generalises to any scroll-position-derived affordance: a "jump to unread"
pill, a shrinking header, a scroll progress bar. The same accumulate-and-
threshold shape drives all of them, and the same caveat applies - see the
Constraints note on why an accumulated offset and the list's true position can
drift apart.

## Feature checklist

- A news feed of twelve items in a rounded white card over a tinted page
  background.
- Scrolling more than 100 vp down reveals a circular back-to-top button near
  the bottom right.
- Scrolling back above the threshold hides it again.
- Tapping the button animates the list to the top over 200 ms on a
  `FastOutSlowIn` curve.
- Double-tapping the system status bar also returns the list to the top.
- Four bottom tabs, of which only the first holds the feed; the other three
  show a 仅作展示 (display only) placeholder.
- The tab bar and the page respect the status-bar and navigation-indicator
  avoid areas, and follow them when they change.

## Architecture

One `entry` module, one page, three view components, two model classes. No
data layer - the list is synthesised in `aboutToAppear`.

```
entry/src/main/ets
├── common/Constants.ets              percentages, the 100 vp threshold, the 200 ms duration, tab indices
├── entryability/EntryAbility.ets     full-screen, avoid areas -> AppStorage, avoidAreaChange subscription
├── entrybackupability/EntryBackupAbility.ets
├── model/IconModel.ets               icon + description, for the tab bar
├── model/NewsInfoModel.ets           icon + title + description, for a feed row
├── pages/NewsMainPage.ets            @Entry, top padding from the status-bar height
└── views/
    ├── NewsInfoList.ets              the List, the offset accumulator, the floating button
    ├── PageBottomTabBuilder.ets      the four tabs and the tab-bar builder
    └── PageTopBuilder.ets            the page title
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the button lives in the same
`Stack` as the list, not in the page:

```typescript
Stack({ alignContent: Alignment.Top }) {
  Column() {
    List({ initialIndex: ..., scroller: this.scrollerController }) { ... }
      .onDidScroll(...)
  }
  if (this.iconIsShow) {
    Stack() { /* the two-layer button image */ }
      .position({ x: Constants.EIGHTY_FIVE_PERCENT, y: Constants.NINETY_TWO_PERCENT })
      .onClick(() => { this.scrollerController.scrollTo({ ... }) })
  }
}
```

The `Scroller`, the accumulated offset, the visibility flag and the button are
all fields of `NewsInfoList`. Nothing about back-to-top escapes into
`NewsMainPage` or the tab builder, so dropping `NewsInfoList()` into a
different tab - or a second one - carries the whole behaviour with it. Compare
with the alternative shape, where the page owns the `Scroller` and passes it
down: that works too, and immediately raises the question of which component
owns the threshold.

The `if (this.iconIsShow)` is worth noting as a choice against
`.visibility(...)`. The conditional destroys the button when it is not needed
rather than laying it out invisibly, which is right here because the button is
two `Image`s and has no state to preserve across appearances.

## Implementation steps

1. **Give the `List` a `Scroller`** (`scroller: this.scrollerController`).
   Without it there is nothing to call `scrollTo` on.
2. **Accumulate `onDidScroll` deltas into a `@State` number.** The callback
   receives the offset *for that event*, not the absolute position, so it must
   be summed.
3. **Threshold the accumulator with an explicit else branch** so the button
   hides again on the way back up. 100 vp is roughly one list row - far enough
   that the button does not flash on a nudge.
4. **Position the button with percentages**, not absolute vp, so it stays in
   the same place on any screen: `x: '85%'`, `y: '93%'` inside the full-size
   `Stack`.
5. **Animate `scrollTo`**, do not jump. `duration: 200` with
   `Curve.FastOutSlowIn` is the platform's own deceleration feel and keeps the
   user oriented.
6. **Add `.backToTop(true)` to the `List`** for the status-bar double tap. On
   API 18 and later this is already the default for vertical scrolling, but
   declaring it makes the intent explicit and covers API 16-17.
7. **Read the avoid areas through `@StorageProp` and convert with `px2vp` at
   the point of use.** The ability writes px; the page and the tab bar convert.
8. **Add the bottom inset into the tab bar height**, not as page padding -
   `barHeight(this.tabHeight + px2vp(bottomRectHeight))` keeps the bar's
   background under the navigation indicator while the labels stay above it.

## Verified snippets

All snippets are from `MoveToTop.zip`. Nothing here is corrected - the review
found no defects in this sample.

**The list, the accumulator and the button — `entry/src/main/ets/views/NewsInfoList.ets`** (as shipped)

```typescript
@Component
export struct NewsInfoList {
  @State newsInfoList: NewsInfoModel[] = [];
  @State iconIsShow: boolean = false;
  @State currentYOffset: number = 0;
  private scrollerController: Scroller = new Scroller();

  build() {
    Stack({ alignContent: Alignment.Top }) {
      Column() {
        List({
          initialIndex: Constants.LIST_INITIAL_INDEX,
          scroller: this.scrollerController
        }) {
          ForEach(this.newsInfoList, (newsItem: NewsInfoModel, index: number) => {
            ListItem() {
              this.newsInfoItem(newsItem, index);
            }
            .alignSelf(ItemAlign.Start)
          })
        }
        .listDirection(Axis.Vertical)
        .scrollBar(BarState.Off)
        .friction(Constants.FRICTION)
        .edgeEffect(EdgeEffect.Spring)
        .backToTop(true)                    // 双击顶部状态栏返回顶部
        .divider({
          strokeWidth: Constants.DIVIDER_STROKE_WIDTH,
          color: $r('app.color.divider_color'),
          startMargin: $r('app.float.margin_14'),
          endMargin: $r('app.float.margin_14')
        })
        .onDidScroll((scrollOffset: number) => {
          this.currentYOffset += scrollOffset;
          if (this.currentYOffset > Constants.ICON_SHOW_Y_OFFSET) {
            this.iconIsShow = true;
          } else {
            this.iconIsShow = false;
          }
        })
      }
      .borderRadius({
        topLeft: $r('app.float.border_radius_16'),
        topRight: $r('app.float.border_radius_16')
      })
      .backgroundColor($r('app.color.background_color_white'))
      .height(Constants.FULL_PERCENT)

      if (this.iconIsShow) {
        Stack() {
          Image($r('app.media.back1'))
            .aspectRatio(Constants.ASPECT_RATIO)
            .width($r('app.float.width_32'))
          Image($r('app.media.back2'))
            .aspectRatio(Constants.ASPECT_RATIO)
            .width($r('app.float.width_20'))
        }
        .position({
          x: Constants.EIGHTY_FIVE_PERCENT,     // '85%'
          y: Constants.NINETY_TWO_PERCENT       // '93%'
        })
        .onClick(() => {
          this.scrollerController.scrollTo({
            xOffset: Constants.SCROLL_TO_X_OFFSET,
            yOffset: Constants.SCROLL_TO_Y_OFFSET,
            animation: {
              duration: Constants.SCROLL_TO_ANIMATION_DURATION,   // 200
              curve: Curve.FastOutSlowIn
            }
          })
        })
      }
    }
    .width(Constants.FULL_PERCENT)
    .height(Constants.FULL_PERCENT)
  }
}
```

**`onDidScroll` hands you a delta, and that is the whole reason
`currentYOffset` exists.** The callback fires after layout with the offset
consumed by *this* scroll event, so a single frame of a fast fling reports a
large number and a slow drag reports a stream of small ones. Summing them
gives a running position; comparing that sum against `ICON_SHOW_Y_OFFSET`
(100) is what makes the button appear about one row down. The `else` branch is
not optional - without it the flag latches on and the button never hides
again.

**Three list attributes shape the feel.** `scrollBar(BarState.Off)` removes the
scrollbar because the floating button is now the navigation affordance.
`edgeEffect(EdgeEffect.Spring)` gives the bounce that tells the user they have
reached the end - which matters more once the scrollbar is gone.
`friction(0.6)` slows the fling below the default so a flick does not
overshoot the twelve-item list entirely.

**The button is two stacked images, not one.** `back1` at 32 vp is the circular
plate; `back2` at 20 vp is the arrow glyph drawn over it. Both carry
`aspectRatio(1)` so a non-square asset cannot distort. Keeping them separate
means the plate can be re-tinted or the arrow swapped without re-exporting a
composite.

**The status-bar gesture — same file, one line** (as shipped)

```typescript
.backToTop(true) // 双击顶部状态栏返回顶部
```

**This is the cheapest of the three mechanisms and the one users discover
without being told.** It is a universal scrollable attribute available from
API 15. Note the default the reference records: before API 18 it is `false`
for every direction, and from API 18 it is `false` for horizontal scrolling
and `true` for vertical. Since this sample's `compatibleSdkVersion` is
`5.0.4(16)`, the explicit `true` is doing real work on the low end of its
range and is a harmless restatement on the high end. Setting it explicitly is
the right call whenever the supported range straddles 18.

`scrollTo` and `backToTop` reach the same place by different routes: the
button's `scrollTo` is your animation, and `backToTop` is the system's. They do
not conflict, and neither one needs to know about the other.

**Avoid areas out of the ability — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
windowStage.loadContent('pages/NewsMainPage', (err) => {
  if (err.code) { /* ... */ return; }

  let windowClass: window.Window = windowStage.getMainWindowSync();
  // 1. 设置窗口全屏
  windowClass.setWindowLayoutFullScreen(true).then(() => {
    hilog.info(DOMAIN, 'testTag', 'Succeeded in setting the window layout to full-screen mode.');
  }).catch((err: BusinessError) => {
    hilog.error(DOMAIN, 'testTag',
      'Failed to set the window layout to full-screen mode. Cause:' + JSON.stringify(err));
  });

  // 2. 获取布局避让遮挡的区域
  let type = window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR;
  let avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('bottomRectHeight', avoidArea.bottomRect.height);

  type = window.AvoidAreaType.TYPE_SYSTEM;
  avoidArea = windowClass.getWindowAvoidArea(type);
  AppStorage.setOrCreate('topRectHeight', avoidArea.topRect.height);

  // 3. 注册监听函数，动态获取避让区域数据
  windowClass.on('avoidAreaChange', (data) => {
    if (data.type === window.AvoidAreaType.TYPE_SYSTEM) {
      AppStorage.setOrCreate('topRectHeight', data.area.topRect.height);
    } else if (data.type === window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR) {
      AppStorage.setOrCreate('bottomRectHeight', data.area.bottomRect.height);
    }
  });
});
```

consumed by the page and the tab bar:

```typescript
// NewsMainPage.ets
@StorageProp('topRectHeight') topRectHeight: number = 0;
// top数值与状态栏区域高度保持一致
.padding({ top: this.getUIContext().px2vp(this.topRectHeight) })

// PageBottomTabBuilder.ets
@StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
.barHeight(this.tabHeight + this.getUIContext().px2vp(this.bottomRectHeight))
.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM])
```

**This is the correct form of a pattern several samples in this industry get
wrong.** `setWindowLayoutFullScreen` has both a `then` and a `catch`, so a
failure is logged rather than swallowed as an unhandled rejection. The initial
values are read once *and* an `avoidAreaChange` subscription keeps them
current, so rotating the device or folding the screen updates the padding
instead of leaving it stale. And `@StorageProp` with a typed default of `0`
means a missing key degrades to no padding rather than `undefined`.

The one thing missing is symmetry: nothing calls `windowClass.off('avoidAreaChange')`
in `onWindowStageDestroy`. For a single-ability app whose window dies with the
process this is inert, but it is the habit that leaks in a multi-window app.

**Adding the inset to `barHeight` rather than padding the page** is the detail
worth copying. The tab bar's background then extends under the navigation
indicator - so the indicator sits on the bar's colour, not on the page's - and
the per-tab `Row().height(px2vp(this.bottomRectHeight))` spacer inside
`tabBuilder` keeps the icons and labels above it. `expandSafeArea` on the
`Tabs` is what permits the bar to draw into that region at all.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`module.json5` declares `deviceTypes: ["phone", "tablet", "2in1"]` and an
`EntryBackupAbility` extension. `AppScope/app.json5` declares no
`configuration` profile, so the app follows the system font scale.

The layout constants are string percentages and `$r('app.float....')`
resources rather than inline numbers, which is why the same component behaves
on all three declared device types without a breakpoint anywhere.

## Constraints

- API Version 16 Release or later; HarmonyOS 5.0.4 Release SDK or later;
  DevEco Studio 5.0.4 Release or later. `compatibleSdkVersion` is `5.0.4(16)`.
- **`backToTop` needs API 15**, and its default flipped to `true` for vertical
  scrolling in API 18. Declaring it explicitly is what keeps behaviour uniform
  across the sample's supported range.
- **`onDidScroll` replaced `onScroll`**, which is deprecated from API 12 for
  `List`, `Grid` and `WaterFlow`. Do not mix them.
- **The accumulator is not the list's position.** `currentYOffset` only ever
  sees deltas from user scrolling. `scrollTo` moves the list without producing
  matching negative deltas, and `backToTop` moves it entirely outside this
  component's knowledge - so after either one the accumulator can still hold a
  value above the threshold while the list is at the top. In this sample the
  button is off-screen at the top of a twelve-item list so it does not show,
  but on a taller layout, or if you drive other UI from the same number, read
  `this.scrollerController.currentOffset().yOffset` instead of summing.
- **The feed is synthesised, not fetched.** `aboutToAppear` builds twelve
  `NewsInfoModel`s from six images and six titles, reusing `image4`/
  `news_title4` at index 6 and wrapping the rest with `i - 6`. There is no
  network, no paging and no refresh.
- **`ForEach` without a key generator.** Fine for a fixed twelve-item array
  built once; add one before the list becomes dynamic.
- Three of the four bottom tabs render the same 仅作展示 placeholder.
  `Tabs` has `scrollable(false)`, so tabs change by tap only - deliberate,
  since a horizontal swipe on a feed should not switch sections.

## Pitfalls

No defects were found in this document or sample during review. The frontmatter
`findings` list is empty and the status is `verified`.

Two things reviewed and cleared, worth naming so they are not re-investigated:

- The `avoidAreaChange` subscription is never released. Unlike the equivalent
  defect in other industries' samples, here the window lives for the process
  and the app has a single ability, so nothing leaks in practice - it remains
  the wrong habit to copy into a multi-window app.
- `PageBottomTabBuilder` sets both `Tabs({ index: this.currentIndex })` and
  `onChange` writing `currentIndex`, which looks circular. It is the standard
  two-way binding for a controlled `Tabs`, and with `scrollable(false)` the
  only writer is the tab builder's own `onClick`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-scrollable-common.md` - `onDidScroll` (API 12+), `backToTop` (API 15+) and its per-version defaults, the `onScroll` deprecation
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scrollable-common
- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroller.scrollTo`, `ScrollOptions`, `ScrollAnimationOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `initialIndex`, `divider`, `friction`, `edgeEffect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-image.md` - `aspectRatio` on the two stacked button layers
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-image
- `NEWS-07` - the same avoid-area boilerplate on a reader instead of a list
- `huawei_industry_tree/11_news_reading/docs/08_move_to_top.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/move_to_top-0000002283118621
