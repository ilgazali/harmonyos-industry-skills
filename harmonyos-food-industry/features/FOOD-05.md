---
id: FOOD-05
title: Custom pull-to-refresh and load-more - a Refresh builder over a WaterFlow feed
industry: 17_food
doc: huawei_industry_tree/17_food/docs/05_custom_refresh.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_refresh-0000002331948181
sample: huawei_industry_tree/17_food/downloads/CustomRefresh.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.CoreFileKit"]
apis: [Refresh, RefreshStatus, onRefreshing, onStateChange, onOffsetChange, "$$", WaterFlow, FlowItem, onReachEnd, nestedScroll, NestedScrollMode, edgeEffect, LazyForEach, IDataSource, DataChangeListener, Progress, ProgressStatus, LoadingProgress, Scroll, onDidScroll, Tabs, onContentWillChange, "UIContext.getMeasureUtils", "UIContext.px2vp", "window.getWindowAvoidArea"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-17-0026, HW-17-0027, HW-17-0029]
status: verified-with-fixes
---

## When to use

Load this card when a feed needs **its own refresh and load-more visuals**
rather than the system spinner - a branded ring, a progress arc that tracks
the pull, a footer that appears only while a page is in flight. The pattern
pairs `Refresh` with a `builder` for the top and `WaterFlow`'s `onReachEnd`
with a guarded flag for the bottom, over a `LazyForEach` fed by a hand-written
`IDataSource`.

The transferable part is the wiring, not the artwork. `Refresh` gives you two
callbacks - `onOffsetChange` (how far the user has pulled, in vp) and
`onStateChange` (which of the five `RefreshStatus` values you are in) - and
between them you can drive any indicator you like. Here the pull distance
feeds a ring `Progress` and the state decides whether the ring exists at all.
Swap the `Progress` for a Lottie animation or a logo and nothing else changes.

The layout is worth studying separately: an outer `Scroll` holds a collapsing
header, `Refresh` holds a `Tabs`, each tab holds a `WaterFlow`, and
`nestedScroll` decides which of the two scrollables consumes a gesture. That
four-deep nesting is what most real feeds look like, and it is the part that
is fiddly to get right.

## Feature checklist

- A two-column staggered card feed under a top tab bar (热门 / 最新 / 菜品) and
  a four-item bottom tab bar.
- A large page title that fades out as the feed scrolls up and back in as it
  scrolls down.
- Pulling down reveals a custom refresh area: a ring `Progress` whose value is
  the pull distance, plus a 正在刷新 (refreshing) label.
- The ring is drawn only while the state is neither `Inactive` nor `Done`, and
  switches from `PROGRESSING` to `LOADING` once the refresh actually starts.
- Releasing past the threshold triggers the refresh; after a simulated one
  second the feed reloads and the indicator retracts.
- Scrolling to the bottom shows a `LoadingProgress` footer and appends one
  page of six cards, exactly once per arrival.
- The three non-home bottom tabs are vetoed with a "demo only" toast.

## Architecture

One `entry` module, five ArkUI files. The data layer is one class with no
network in it.

```
entry/src/main/ets
├── common/Constants.ets            every literal in the sample, named
├── entryability/EntryAbility.ets   full-screen layout, avoid areas -> AppStorage
├── entrybackupability/
├── model
│  ├── TabModel.ets                 CategoryInfo/TabIcon + the two static tab lists
│  └── WaterFlowDataSource.ets      IDataSource: dataArray + listener fan-out
└── pages
   ├── MainPage.ets                 @Entry, 239 lines - the whole feature
   └── WaterFlowItem.ets            one feed card
```

The documented tree matches the zip.

`MainPage` is six `@Builder` methods (`refreshBuilder`, `footer`,
`getListView`, `headerBuilder`, `topTabBuilder`, `bottomTabBuilder`) and a
`build()` that reads as a layout outline. Seven pieces of `@State` carry the
feature: `topTabIndex`, `bottomTabIndex`, `titleOpacity`, `dataSource`,
`refreshing`, `refreshOffset`, `refreshState`, `isLoading`.

**The design decision worth copying** is that `refreshOffset` and
`refreshState` are mirrored into `@State` from the two `Refresh` callbacks,
and the builder reads only that state. The builder never asks `Refresh`
anything; it is a pure function of three numbers. That is what makes the
indicator swappable - and it is also why `Refresh`'s documented caveat about
`builder` (the custom component is destroyed and re-created during refresh)
does no damage here: there is no animation state inside the builder to lose.

`WaterFlowDataSource` is the standard `IDataSource` boilerplate - a
`dataArray`, a `listeners` array, and one `notifyXxx` per listener callback.
Copy it as the shape; do not copy its `reload()` (`HW-17-0027`).

## Implementation steps

1. **Wrap the scrollable in `Refresh({ refreshing: $$this.refreshing, builder:
   this.refreshBuilder() })`.** The `$$` is required: `Refresh` writes
   `refreshing` back to `true` when the gesture crosses the threshold, and you
   write it back to `false` when the work finishes.
2. **Mirror both callbacks into state:** `onOffsetChange` → `refreshOffset`,
   `onStateChange` → `refreshState`. The offset arrives in vp.
3. **Gate the indicator on the state,** not on `refreshing`. `Inactive` and
   `Done` are the two states where nothing should be drawn; the three in
   between (`Drag`, `OverDrag`, `Refresh`) are the ones with a visible pull.
4. **Do the work in `onRefreshing` and clear `refreshing` yourself.** Nothing
   clears it for you; the sample simulates a request with a 1 s `setTimeout`.
5. **Make the reload actually reload.** Rebuild the backing array before
   notifying, otherwise the pull is a one-second animation over an unchanged
   list (`HW-17-0027`).
6. **Implement `IDataSource` and drive it with `LazyForEach`** so only visible
   cards are built. Notify with `notifyDataAdd(index)` on append, not
   `notifyDataReload`, so appending does not rebuild the whole feed.
7. **Append in `onReachEnd` behind an `isLoading` guard,** set before the
   async work and cleared after it. Without the guard the callback fires again
   on every settle frame while the list is at the bottom.
8. **Put the footer in its own `FlowItem`** after the `LazyForEach` and toggle
   `Visibility.Hidden`, not `Visibility.None`, so the reserved space does not
   make the list jump when loading starts.
9. **Set `nestedScroll` on the inner `WaterFlow`:** `PARENT_FIRST` forward so
   the outer `Scroll` collapses the header before the feed moves,
   `SELF_FIRST` backward so the feed returns to its own top before the header
   comes back.
10. **Clamp any scroll-driven visual** to its valid range (`HW-17-0026`).

## Verified snippets

All snippets are from `CustomRefresh.zip`. Corrected forms are marked.

**The custom refresh area — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@State refreshing: boolean = false;
@State refreshOffset: number = 0;
@State refreshState: RefreshStatus = RefreshStatus.Inactive;

@Builder
refreshBuilder() {
  Stack({ alignContent: Alignment.Bottom }) {
    // 当刷新状态处于下拉中或刷新中状态时Progress组件才存在
    if (this.refreshState !== RefreshStatus.Inactive && this.refreshState !== RefreshStatus.Done) {
      Row() {
        Progress({ value: this.refreshOffset, total: Constants.TOTAL, type: ProgressType.Ring })
          .width(Constants.PROGRESS_SIZE)
          .height(Constants.PROGRESS_SIZE)
          .style({ status: this.refreshing ? ProgressStatus.LOADING : ProgressStatus.PROGRESSING })
          .margin(Constants.TEN);
        Text($r('app.string.refreshing'))          // 正在刷新
          .fontSize(Constants.FONT_SIZE_NORMAL)
          .fontColor(Constants.FONT_COLOR_GREY);
      }
      .alignItems(VerticalAlign.Center);
    }
  }
  .clip(true)
  .height(Constants.FULL_HEIGHT)
  .width(Constants.FULL_WIDTH);
}
```

**Three details carry the design.** `value: this.refreshOffset` with
`total: 64` turns the pull distance directly into an arc - 64 vp is the
default `refreshOffset` threshold, so the ring completes exactly when the
gesture becomes a refresh, and the user gets a progress bar toward the
trigger point rather than a spinner that means nothing. `style({ status })`
is the handover: `PROGRESSING` while the arc is being drawn by the finger,
`LOADING` once `refreshing` is true and the ring should spin on its own. And
the `if` on `refreshState` is what keeps the row out of the tree in `Inactive`
and `Done` - retracting past a still-mounted indicator looks like a glitch.

`Stack({ alignContent: Alignment.Bottom })` with `height('100%')` and
`clip(true)` pins the row to the bottom edge of the refresh area, which itself
grows with the pull. So the indicator appears to be dragged down from behind
the header rather than fading in place.

Note the API caveat: since API 12 the reference recommends `refreshingContent`
over `builder`, "to avoid animation interruptions caused by the destruction
and re-creation of the custom component during the refreshing process". This
builder holds no internal animation, so it is safe here - but a builder
running its own `animateTo` would stutter.

**The refresh handler — same file** (as shipped)

```typescript
Refresh({ refreshing: $$this.refreshing, builder: this.refreshBuilder() }) {
  Tabs() {
    // ... one TabContent per top tab, each holding this.getListView()
  }
}
.width(Constants.FULL_WIDTH)
.height(Constants.FULL_HEIGHT)
.backgroundColor(Constants.BACKGROUND_COLOR)
.onOffsetChange((offset: number) => {
  this.refreshOffset = offset;
})
.onStateChange((state: RefreshStatus) => {
  this.refreshState = state;
})
.onRefreshing(() => {
  setTimeout(() => {
    this.dataSource.reload();
    this.refreshing = false;      // nothing else clears it
  }, Constants.DELAY_TIME);
});
```

**`$$this.refreshing` is a two-way binding and both directions are used.**
`Refresh` sets it to `true` when the released pull exceeds the threshold -
that is what fires `onRefreshing` - and only the app can set it back to
`false`. Forget the second assignment and the indicator never retracts. The
`setTimeout` stands in for the request; in a real app the same line goes in
the promise's `finally`.

`onStateChange` and `onOffsetChange` are read-only observers - they do not
influence the gesture, so mirroring them into `@State` costs one extra render
per frame of the pull and buys a fully custom indicator.

**The data source — `entry/src/main/ets/model/WaterFlowDataSource.ets`**
(corrected, see `HW-17-0027`)

```typescript
export class WaterFlowDataSource implements IDataSource {
  private dataArray: number[] = [];
  private listeners: DataChangeListener[] = [];

  constructor() {
    for (let i = 0; i < Constants.ONCE_COUNT; i++) {
      this.dataArray.push(i);
    }
  }

  public getData(index: number): number {
    return this.dataArray[index];
  }

  public totalCount(): number {
    return this.dataArray.length;
  }

  registerDataChangeListener(listener: DataChangeListener): void {
    if (this.listeners.indexOf(listener) < 0) {
      this.listeners.push(listener);
    }
  }

  // 在数据尾部增加一个元素
  public addLastItem(): void {
    this.dataArray.splice(this.dataArray.length, 0, this.dataArray.length);
    this.notifyDataAdd(this.dataArray.length - 1);      // targeted: only the new index rebuilds
  }

  // 重新加载数据
  public reload(): void {
    this.dataArray = [];                                // FIX: sample only notifies
    for (let i = 0; i < Constants.ONCE_COUNT; i++) {    // FIX: refetch or reset the first page
      this.dataArray.push(i);
    }
    this.notifyDataReload();
  }
}
```

**The asymmetry between `addLastItem` and `reload` is the lesson.** Appending
uses `notifyDataAdd(index)`, which tells `LazyForEach` that exactly one index
appeared and leaves every existing card mounted - that is why load-more does
not flicker. `notifyDataReload` is the blunt one: it re-queries everything.
The sample calls the blunt one over an unchanged array, so a pull-to-refresh
costs a full rebuild and produces an identical list. After a few load-more
rounds the "refreshed" feed is still 30 cards long, which is the give-away.

`registerDataChangeListener` guarding with `indexOf(listener) < 0` matters
more than it looks: with the data source shared across three `TabContent`s,
each tab's `LazyForEach` registers its own listener, and a tab revisit must
not register the same one twice.

**Load-more and the footer — `entry/src/main/ets/pages/MainPage.ets`** (as shipped)

```typescript
@Builder
private getListView() {
  WaterFlow() {
    LazyForEach(this.dataSource, (item: number) => {
      FlowItem() {
        WaterFlowItem({ index: item });
      }
      .width(Constants.FULL_WIDTH)
      .margin({ bottom: Constants.FOUR });
    }, (item: string) => item);
    FlowItem() {
      this.footer();                       // the loading row, always present
    };
  }
  .onReachEnd(() => {
    // 滚动到列表底部时触发加载
    if (!this.isLoading) {
      this.isLoading = true;
      setTimeout(() => {
        for (let i = 0; i < Constants.ONCE_COUNT; i++) {
          this.dataSource.addLastItem();
        }
        this.isLoading = false;
      }, Constants.DELAY_TIME);
    }
  })
  .columnsTemplate('1fr 1fr')
  .edgeEffect(EdgeEffect.Spring, { alwaysEnabled: true })
  .nestedScroll({
    scrollForward: NestedScrollMode.PARENT_FIRST,
    scrollBackward: NestedScrollMode.SELF_FIRST
  });
}
```

**`onReachEnd` fires repeatedly, so the guard is the feature.** It is raised
every time the content end is reached, including on each spring-back frame of
`EdgeEffect.Spring`; without `isLoading` a single overscroll appends several
pages. Set the flag synchronously before the async work, clear it in the same
callback that finishes.

**`nestedScroll` is what makes the four-deep layout behave.** `PARENT_FIRST`
on the forward (upward-content) direction hands the gesture to the outer
`Scroll` first, so the big title collapses before the cards move;
`SELF_FIRST` backward means a downward flick returns the feed to its own top
before the header reappears. Reverse the two and the header fights the feed
for every gesture.

The footer is a plain `FlowItem` appended after the `LazyForEach` and toggled
with `.visibility(this.isLoading ? Visibility.Visible : Visibility.Hidden)` -
`Hidden` keeps its 64 vp of height reserved, so arriving at the bottom does
not shift the last row of cards.

**The collapsing title — same file** (corrected, see `HW-17-0026`)

```typescript
Scroll() {
  Column() {
    this.headerBuilder();            // Text($r('app.string.header_title')).opacity(this.titleOpacity)
    Refresh({ refreshing: $$this.refreshing, builder: this.refreshBuilder() }) {
      // ...
    }
  }
  .width(Constants.FULL_WIDTH);
}
.scrollBar(BarState.Off)
.onDidScroll((yOffset: number) => {
  // FIX: sample is `this.titleOpacity -= yOffset / Constants.TWENTY;` with no clamp
  this.titleOpacity =
    Math.min(1, Math.max(0, this.titleOpacity - yOffset / Constants.TWENTY));
});
```

**`onDidScroll` gives you a delta, not a position, and deltas accumulate.**
The shipped line subtracts every frame's delta from an unbounded number, so
after a long scroll `titleOpacity` is not 0 but something like -40. Scrolling
back up then has to "repay" that overshoot before the title becomes visible
again - the user returns to the top of the feed and the header is still
blank. Clamping each update, as above, fixes it. Deriving the value from
`Scroll.currentOffset()` instead of accumulating deltas is the sturdier
version: it cannot drift at all.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions`; all
content is bundled resources. It registers one `EntryBackupAbility`
(`type: "backup"`) with a `$profile:backup_config`, which is the DevEco
template default and unrelated to the feature.

`EntryAbility` reads the avoid areas once in `onWindowStageCreate`:

```typescript
const WIN = await windowStage.getMainWindow();
WIN.setWindowLayoutFullScreen(true);
AppStorage.setOrCreate('topHeight', WIN.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height);
AppStorage.setOrCreate('bottomHeight',
  WIN.getWindowAvoidArea(window.AvoidAreaType.TYPE_NAVIGATION_INDICATOR).bottomRect.height);
```

This is the one-shot form. `FOOD-04` uses the `avoidAreaChange` subscription
instead, which is the better half of the two - a one-shot read does not follow
a rotation or a 2in1 window resize. Note also that `MainPage` reads these back
with `AppStorage.get<number>('topHeight')`, which is `number | undefined`, and
feeds it straight to `px2vp`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` is `6.0.0(20)`.
- `deviceTypes` is `phone`, `tablet`, `2in1`, but the feed is fixed at
  `columnsTemplate('1fr 1fr')` and `width('90%')` - two columns on a tablet is
  not what a tablet wants. Make the template breakpoint-driven if the app is
  adaptive.
- **There is no network.** The data source is an array of integers and the
  cards index six bundled images and four bundled strings modulo their index,
  so card 7 is card 1 again. Both "requests" are `setTimeout(1000)`.
- The list only grows. There is no page count, no end-of-feed state and no
  error path - `onReachEnd` will keep appending forever.
- The `LazyForEach` key generator is declared `(item: string) => item` over a
  `number[]`. It works because the values are unique integers, but the type is
  wrong and it breaks the moment the item type becomes an object.
- The three non-home bottom tabs are deliberately vetoed:
  `onContentWillChange` returns `false` for any `comingIndex > 0` after
  showing a 仅用于功能演示 toast. That is an intentional demo guard here, not
  the accidental veto seen in other samples.
- Since API 12, `refreshingContent` supersedes `builder` for the refresh area.

## Pitfalls

- **`HW-17-0026`** (B/low, confirmed): `onDidScroll` subtracts unclamped
  deltas from `titleOpacity`, driving it far below 0 on a long scroll, so the
  title stays invisible long after the user has scrolled back to the top.
  Fix: clamp to `[0, 1]` on every update, or compute the opacity from
  `Scroll.currentOffset()` instead of accumulating.
- **`HW-17-0027`** (B/low, probable): `reload()` only calls
  `notifyDataReload()` without touching `dataArray`, so a pull-to-refresh
  re-renders the same - possibly already grown - list and visibly does
  nothing, while the document claims 通过reload()实现刷新 (refresh is
  implemented via `reload()`). Fix: rebuild `dataArray` with the first
  `ONCE_COUNT` items (or refetch) before notifying.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-refresh.md` - `RefreshOptions`, `builder` vs `refreshingContent`, `RefreshStatus`, `onStateChange`, `onOffsetChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-refresh
- `documentation/harmonyos-references/02_application-framework/ts-container-waterflow.md` - `WaterFlow`, `FlowItem`, `onReachEnd`, `columnsTemplate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-waterflow
- `documentation/harmonyos-references/02_application-framework/ts-rendering-control-lazyforeach.md` - `IDataSource`, `DataChangeListener`, the key generator
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - when to notify add vs reload
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/12_coding-and-debugging/ide_hp-arkui-no-stringify-lazyforeach-key.md` - key-generator pitfalls
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/ide_hp-arkui-no-stringify-lazyforeach-key
- `FOOD-04` - the avoid-area boilerplate in its subscription form
- `FOOD-02` - the same waterfall feed shape inside a full recipe app
