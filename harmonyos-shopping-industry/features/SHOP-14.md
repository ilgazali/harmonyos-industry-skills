---
id: SHOP-14
title: Self-sizing quick-entry menu - a paged horizontal List whose container height interpolates with the swipe
industry: 16_shopping
doc: huawei_industry_tree/16_shopping/docs/14_auto_height_list.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_height_list-0000002359342093
sample: huawei_industry_tree/16_shopping/downloads/AutoHeightList.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [componentUtils, hilog, window, List, ListItem, ListScroller, "List.lanes", scrollSnapAlign, ScrollSnapAlign, listDirection, onAreaChange, onWillScroll, onScrollIndex, "ListScroller.currentOffset", "ListScroller.scrollToIndex", Scroll, EdgeEffect, friction]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-16-0016, HW-16-0013, HW-16-0027]
status: verified-with-fixes
---

## When to use

Load this card when a **home-screen quick-entry grid has more entries than fit
on one row, and you want the extra rows to appear by swiping sideways rather
than by a "more" button or a taller block that is mostly empty**. Page one is
a single row of five icons with a sixth peeking in from the right; page two is
the rest of the catalogue as a full grid; and the container grows smoothly
from one-row height to grid height *while the finger is still moving*, pushing
the content below it down as it goes.

The transferable mechanism is small: a horizontal `List` of two `ListItem`s,
each holding a vertical `List` with `lanes()`, wrapped in a `Scroll` whose
height is a `@State` number. `onAreaChange` samples the natural one-row height
once, `onWillScroll` maps the horizontal offset into a height between that and
the computed maximum. Nothing is animated explicitly - the height *is* the
scroll position, so it tracks the drag, the fling, and the snap-back for free.

Use it for any two-state container whose size should follow a gesture rather
than a transition: a collapsing filter bar, a peeking detail pane, a category
strip that unfolds. **Check `HW-16-0016` before changing the number of
entries** - the row-count formula is off by one and only works for the shipped
data.

## Feature checklist

- A white rounded card under the search bar holds the quick-entry menu.
- Page one shows five icon+label entries across, with the sixth entry half
  visible at the right edge as a swipe affordance.
- Page two shows the remaining ten entries as a 5-wide, 2-row grid.
- Dragging left grows the card's height continuously from one row to two rows;
  dragging back shrinks it.
- The content below the card (the tab bar and the product list) moves down and
  up with the card, because the card is a normal Column child, not an overlay.
- Paging snaps: a release always lands on page one or page two, never between.
- A two-dot indicator under the menu tracks the current page; the second
  indicator is a pill rather than a dot.
- Tapping an indicator scrolls to that page.

## Architecture

One `entry` module. `AutoList` is the feature; everything else is the shop
page it sits in.

```
entry/src/main/ets
├── component
│   ├── AutoList.ets            the whole feature: paged menu + height interpolation
│   ├── BottomNavigation.ets    the bottom bar (positioned, not a Tabs bar)
│   ├── CustomTabBarView.ets    the scrollable category tab bar
│   ├── ProductView.ets         the product waterfall inside each TabContent
│   └── SearchView.ets          the search field
├── entryability/EntryAbility.ets    avoid areas -> AppStorage, full screen
├── entrybackupability/EntryBackupAbility.ets
├── model/ProductDataModel.ets  CatalogModel + CATALOGMODEL_DATA (15 entries), PRODUCTLIST
├── pages/Shop.ets              @Entry: SearchView, AutoList, Tabs, BottomNavigation
└── utils/Logger.ets
```

The documented tree matches the zip exactly.

**The design decision worth copying** is that the height is state, and the
scroll offset is its only input. There is no `animateTo`, no gesture
recogniser and no page-change callback driving the resize. `onWillScroll`
fires for every offset delta the list is about to apply - drag, fling and
snap alike - and the handler recomputes an absolute height from the absolute
offset:

```
   offsetX = 0                                offsetX = containerWidth
   height  = containerMinHeight               height  = containerMaxHeight
   ┌───────────────────────────┐              ┌───────────────────────────┐
   │ ① ② ③ ④ ⑤ ⑥⌐             │              │ ⑥ ⑦ ⑧ ⑨ ⑩                │
   └───────────────────────────┘              │ ⑪ ⑫ ⑬ ⑭ ⑮                │
                                              └───────────────────────────┘
```

Because it is a pure function of the offset, the animation is perfectly in
step with the finger and needs no separate reversal path: releasing mid-swipe
lets the snap animation run, `onWillScroll` keeps firing, and the height
follows it back. Copying this means resisting the temptation to drive the
height from `onScrollIndex` - that fires once per page and would give you a
jump, not a track.

The nesting is the second half of the decision: an outer horizontal `List`
whose two `ListItem`s are each `width('100%')`, and inside each a **vertical**
`List` using `lanes()` as a grid. `lanes(5.5)` on page one is what produces
the half-visible sixth icon - a fractional lane count is a legitimate way to
size a peek without measuring anything.

## Implementation steps

1. **Build the pager as a horizontal `List`** with
   `listDirection(Axis.Horizontal)`, `scrollSnapAlign(ScrollSnapAlign.START)`,
   `edgeEffect(EdgeEffect.None)`, `friction(1)` and `lanes(1)`, over an array
   of `pageSize` placeholders.
2. **Make each page a vertical `List` with `lanes(n, gutter)`** rather than a
   `Grid`: page one `lanes(5.5, 10)` for the peek, page two `lanes(5, 10)`.
3. **Slice the catalogue by page** - `slice(0, 5)` and `slice(5, length)` -
   and derive both slice bounds from one shared page-size constant
   (`HW-16-0016`).
4. **Compute the row count of the last page in `aboutToAppear`** as
   `Math.ceil((data.length - PAGE_ITEM_COUNT) / LANES)`; the sample subtracts
   6 where the first page holds 5 (`HW-16-0016`).
5. **Sample the natural height once in `onAreaChange`,** guarded by
   `if (!this.containerWidth)` so later resizes caused by your own height
   writes do not re-enter the measurement.
6. **Derive max height as** `rowHeight * rows + gap * (rows - 1)`, using the
   same 14 vp that the page-two items carry as `margin({ bottom: 14 })`.
7. **Interpolate in `onWillScroll`** from `listScroller.currentOffset().xOffset`
   plus the incoming delta, clamped at zero, normalised by `containerWidth`.
8. **Give the wrapper `Scroll` the height** as
   `.height(this.containerHeight || 'auto')`, so the very first frame - before
   any measurement exists - lays out naturally instead of collapsing to 0.
9. **Track the page with `onScrollIndex`,** accepting the index only when
   `start === end`, i.e. when exactly one page is in view.
10. **Do not copy the doc's step-1 snippet** - its `onAreaChange` handler is
    cut before the closing brace of the `if` and does not parse
    (`HW-16-0013`).

## Verified snippets

All snippets are from `AutoHeightList.zip`. Corrected forms are marked.

**The nested pager — `entry/src/main/ets/component/AutoList.ets`** (as shipped)

```typescript
build() {
  Column() {
    Scroll() {
      List({ scroller: this.listScroller }) {
        ForEach(new Array(this.pageSize).fill(0), (item: number, index: number) => {
          ListItem() {
            List() {
              if (index === 0) {
                ForEach(CATALOGMODEL_DATA.slice(0, 5), (item: CatalogModel) => {
                  ListItem() {
                    Column() {
                      Image(item.icon).width(42).aspectRatio(1).objectFit(ImageFit.Contain);
                      Text(item.title).fontSize(12).margin({ top: 6 });
                    }
                    .alignItems(HorizontalAlign.Center);
                  }
                  .width('100%');
                });
              } else {
                ForEach(CATALOGMODEL_DATA.slice(5, CATALOGMODEL_DATA.length), (item: CatalogModel) => {
                  ListItem() {
                    Column() {
                      Image(item.icon).width(42).aspectRatio(1).objectFit(ImageFit.Contain);
                      Text(item.title).fontSize(12).margin({ top: 6 });
                    }
                    .margin({ bottom: 14 })                // the row gap the max-height formula uses
                    .alignItems(HorizontalAlign.Center);
                  }
                  .width('100%');
                });
              }
            }
            .lanes(index === 0 ? 5.5 : 5, 10)              // 5.5 lanes = the peeking sixth icon
            .padding({ left: 6, right: 6 })
            .edgeEffect(EdgeEffect.None)
            .scrollBar(BarState.Off);
          }
          .width('100%');                                  // each page is exactly one viewport wide
        });
      }
      .scrollBar(BarState.Off)
      .listDirection(Axis.Horizontal)
      .scrollSnapAlign(ScrollSnapAlign.START)
      .friction(1)
      .edgeEffect(EdgeEffect.None)
      .lanes(1);
    }
    .height(this.containerHeight || 'auto')
    .margin({ top: 15 });
  }
  .backgroundColor(Color.White)
  .width('91%')
  .borderRadius(16);
}
```

**Four attributes on the outer list carry the paging.**
`listDirection(Axis.Horizontal)` plus `lanes(1)` makes it a single-track
horizontal strip; `ListItem().width('100%')` sizes each page to the container,
so a page is exactly one swipe; `scrollSnapAlign(ScrollSnapAlign.START)` snaps
each page's leading edge to the viewport start, which is what guarantees the
gesture always resolves to a whole page; and `edgeEffect(EdgeEffect.None)`
removes the bounce that would otherwise push `xOffset` negative and drive the
height formula outside its range. `friction(1)` keeps a fling from carrying
far past a page boundary before the snap takes over.

The inner lists are vertical `List`s in disguise. `lanes(count, gutter)`
distributes items across `count` columns with a `gutter` between them - the
grid you would otherwise build with `Grid`/`GridItem`, but with `List`'s
scroll semantics, and crucially accepting a **fractional** count. `5.5` means
five full icons plus half of the sixth: the peek that tells the user there is
more to the right, with no measurement, no hardcoded width and no separate
affordance view.

`.height(this.containerHeight || 'auto')` deserves the `||`. `containerHeight`
starts at 0, and the first layout pass is the one that produces the natural
height for `onAreaChange` to sample - a literal `0` would collapse the card
and there would be nothing to measure.

**Measuring once and interpolating — same file** (corrected, see `HW-16-0016`)

```typescript
private listScroller: ListScroller = new ListScroller();
@State containerHeight: number = 0;
@State containerWidth: number = 0;
@State containerMaxHeight: number = 0;
@State containerMinHeight: number = 0;
@State pageSize: number = 2;
@State lastPageLines: number = 1;

aboutToAppear(): void {
  // 计算最后一页的条目数 — FIX: the shipped line subtracts 6, but page one holds 5
  this.lastPageLines = Math.ceil((CATALOGMODEL_DATA.length - 5) / 5);
}

// on the outer List:
.onAreaChange((_, n) => {
  // 监听容器区域变化事件
  if (!this.containerWidth) {                       // sample once; our own height writes re-enter this
    this.containerWidth = Number(n.width);
    this.containerHeight = Number(n.height);
    // 总高度 = (单行高度 * 行数) + (行间距 * (行数-1))
    this.containerMaxHeight =
      (this.containerHeight) * this.lastPageLines + 14 * (this.lastPageLines - 1) + 2;
    this.containerMinHeight = this.containerHeight;
  }
})
.onWillScroll((x) => {
  this.updateContainerHeight(x);
})
.onScrollIndex((start, end) => {
  if (start === end) {
    this.currentIndex = start;
  }
})

updateContainerHeight(x: number) {
  // 当前滚动位置 + 新增偏移量 - 容器宽度*(分页数-2)
  let offsetX = this.listScroller.currentOffset().xOffset + x - this.containerWidth * (this.pageSize - 2);
  if (offsetX > 0) {
    let h = offsetX / this.containerWidth * (this.containerMaxHeight - this.containerMinHeight);
    this.containerHeight = this.containerMinHeight + h;
  } else {
    this.containerHeight = this.containerMinHeight;
  }
}
```

**`onAreaChange` is a measurement hook, and it must be idempotent.** Writing
`containerHeight` changes the area, which fires `onAreaChange` again; without
the `if (!this.containerWidth)` guard the "one row" baseline would be
overwritten by the interpolated height on the very first drag and the menu
would never return to one row. Guarding on `containerWidth` rather than
`containerHeight` is deliberate: the width is the value nothing else writes.

**The offset arithmetic is a normalised interpolation, written awkwardly.**
`currentOffset().xOffset` is the offset already applied; `x` is the delta
`onWillScroll` is about to apply, so the sum is the offset *after* this frame -
using it is what keeps the height a frame ahead of the visible scroll instead
of one behind. The `- containerWidth * (pageSize - 2)` term evaluates to `0`
for the shipped `pageSize` of 2; it is the general form of "start growing only
once the last page begins to enter", and it is the line to rewrite if you ever
add a third page, since with `pageSize = 3` it would delay the growth by a
whole page. Dividing by `containerWidth` turns the remaining offset into a
0..1 fraction, which scales the min-to-max span. The `else` branch is not
decoration: it clamps the height at the minimum for every offset in page one,
including the negative offsets a bounce would produce.

The trailing `+ 2` in the max-height formula is a fudge for the residual
sub-pixel gap; the document's version of the same snippet omits it, along with
the closing brace of the `if` (`HW-16-0013`).

**The page indicator — same file** (as shipped)

```typescript
Row({ space: 5 }) {
  ForEach(new Array(this.pageSize).fill(0), (_: number, index: number) => {
    Circle()
      .width(5).height(5)
      .fill(this.currentIndex === index ? '#0A59F7' : '#660a59f7')
      .visibility(index === 0 ? Visibility.Visible : Visibility.None)
      .onClick(() => {
        this.listScroller.scrollToIndex(index, true);
        this.currentIndex = index;
      });
    Image($r('app.media.slide_fill'))
      .borderRadius(5)
      .width(15).height(5)
      .fillColor(this.currentIndex === index ? '#0A59F7' : '#660a59f7')
      .visibility(index === 1 ? Visibility.Visible : Visibility.None)
      .onClick(() => {
        this.listScroller.scrollToIndex(index, true);
        this.currentIndex = index;
      });
  });
}
```

`scrollToIndex(index, true)` reuses the same `ListScroller` the pager was
built with, so a tap animates through the same path a swipe would take - and
therefore fires `onWillScroll` and resizes the card exactly the same way. That
is the payoff of driving the height from the scroller rather than from the tap
handler: the indicator needed no resize code of its own.

Note the shape of this builder, though: it emits **both** a `Circle` and an
`Image` for every page and hides one of each with `Visibility.None`, keyed on
literal indices 0 and 1. It is a two-page indicator wearing a `ForEach`
costume; a third page would render nothing. Either give `CatalogModel` pages a
real indicator component or drop the loop.

## Permissions & config

**None.** The sample declares no `requestPermissions`. Layout values in
`Shop.ets` come from `app.integer`/`app.string` resources, but `AutoList.ets`
itself uses numeric literals throughout (42, 12, 6, 14, 10, 5.5) - including
the `14` that the max-height formula must agree with. If the row gap ever
moves to a resource, the formula has to read the same resource.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- The design assumes exactly two pages. `pageSize` is a `@State` set to 2, the
  indicator hardcodes indices 0 and 1, and the `- containerWidth * (pageSize - 2)`
  term is only a no-op at 2.
- The height interpolation assumes every page-two row is the same height as
  the page-one row that was measured. Longer labels wrapping to two lines on
  page two break the maximum.
- `onWillScroll` runs on every scroll frame and writes `@State`, so the card
  re-lays out per frame. That is acceptable for a two-row menu; do not extend
  the same technique to a container holding a long list.
- `CATALOGMODEL_DATA` has 15 entries. That is the one size at which the
  shipped off-by-one is invisible (`HW-16-0016`).

## Pitfalls

- **`HW-16-0016`** (B/low, confirmed): `aboutToAppear` computes
  `Math.ceil((CATALOGMODEL_DATA.length - 6) / 5)` although the first page
  renders `slice(0, 5)` - five items - so the last page's row count is
  computed from the wrong remainder. With the shipped 15 entries both formulas
  give 2 and the bug is invisible; at 16 entries the second page really has 3
  rows but `containerMaxHeight` is sized for 2, clipping the last row when
  expanded. Fix: subtract 5, and derive both 5s from one shared
  `PAGE_ITEM_COUNT` constant.
- **`HW-16-0013`** (E/medium, confirmed, systematic): the document's step-1
  snippet cuts the `onAreaChange` handler off before the closing brace of its
  `if`, so the published excerpt does not parse; it also drops the `+ 2` the
  shipped formula carries. This is one instance of a corpus-wide abridgement
  defect affecting roughly thirty shopping documents - take snippets from the
  zip, not from the page.
- **`onAreaChange` re-entrancy.** The handler is only correct because it is
  guarded; remove the `if (!this.containerWidth)` and the one-row baseline is
  destroyed on the first drag.
- **A negative `xOffset` must be clamped.** The `else` branch and
  `EdgeEffect.None` together are what stop an over-scroll from producing a
  height below the minimum. Adding `EdgeEffect.Spring` back re-opens that hole.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `List`, `lanes`, `listDirection`, `scrollSnapAlign`, `ListScroller.currentOffset`, `onWillScroll`, `onScrollIndex`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-references/02_application-framework/ts-universal-component-area-change-event.md` - `onAreaChange` and when it fires
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-component-area-change-event
- `huawei_industry_tree/16_shopping/docs/14_auto_height_list.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/auto_height_list-0000002359342093
- `SHOP-12` - the systematic snippet-abridgement finding `HW-16-0013` is anchored there
