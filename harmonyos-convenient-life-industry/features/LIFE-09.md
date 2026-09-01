---
id: LIFE-09
title: Infinite icon carousel - a timer-driven Scroll with front-remove/back-append rotation, beside a Swiper of Grid pages
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/09_easylife_loopscroll.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/easylife_loopscroll-0000002284598853
sample: huawei_industry_tree/02_convenient_life/downloads/LoopScroll.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit"]
apis: [Scroll, Scroller, "scroller.scrollTo", "scroller.currentOffset", onScrollFrameBegin, onScrollStart, onScrollStop, ScrollDirection, friction, Swiper, SwiperController, Grid, GridItem, columnsTemplate, LazyForEach, IDataSource, DataChangeListener, pixelRound, PixelRoundCalcPolicy, setInterval, clearInterval, expandSafeArea, "@Prop", "@Builder", "ConfigurationConstant.ColorMode"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0061, HW-02-0062, HW-02-0063, HW-02-0064, HW-02-0065, HW-02-0066, HW-02-0067]
status: verified-with-fixes
---

## When to use

Load this card when you need a **strip of items that scrolls forever in one
direction** and still accepts a finger - a marquee of category icons, a ticker,
a promo rail. `Swiper` gives you page-at-a-time looping; this gives you
continuous, sub-pixel, constant-velocity motion with no seam.

The mechanism is worth knowing even if you never ship it: a `Scroll` holding
twice as many items as fit on screen, a 10 ms timer that advances the offset by
a fixed distance per elapsed millisecond, and - when the offset passes half the
data - a **rotation of the data source** (remove from the front, append to the
back) paired with an equal subtraction from the offset. The user sees an
unbroken strip; the scroller never actually travels far.

The same document also covers the ordinary case beside it: a `Swiper` of `Grid`
pages for manual paging. That half is four lines and needs no tricks.

Take the carousel when the motion must be continuous. Take `Swiper` with
`autoPlay(true)` when page-at-a-time is acceptable - it is a fraction of the
code and has none of the failure modes below.

## Feature checklist

- A two-page `Swiper` of 4x2 icon `Grid`s, paged by finger.
- Below it, a horizontal strip of eight icons that scrolls continuously right to
  left at a constant speed.
- The strip has no visible start or end - it wraps seamlessly.
- Touching the strip stops the animation; releasing it resumes.
- A manual fling wraps too: dragging past either end jumps the content by one
  full data width without a visible seam.
- Motion is sub-pixel smooth - items are not snapped to whole pixels.

## Architecture

One `entry` module, three views, two constant files.

```
entry/src/main/ets
├── constants
│   ├── CommonConstants.ets      ITEM_WIDTH 100, ITEM_NUM 8, ITEM_NUM_HALF 4,
│   │                            TIMING 10, UNIT_DISPLACEMENT 0.5, SCROLL_FRICTION 0.5
│   └── ListDataConstants.ets    GRID_COL_LIST / SWIPER_IMAGE_LIST (2 pages), SCROLL_* (8 icons)
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── pages/HomePage.ets           @Entry - title, Swiper, banner, TimeScrollView
└── view
    ├── GridItemComponent.ets    icon + label, root is GridItem()   (HW-02-0065)
    ├── MenuItemComponent.ets    one Swiper page: a Grid of eight GridItemComponents
    └── TimeScrollView.ets       THE CARD: IDataSource, the timer, the wrap arithmetic
```

The documented tree matches the zip exactly.

**Three numbers define the motion.**

```
ITEM_WIDTH       100    one icon cell, vp
ITEM_NUM           8    distinct icons; the data source holds 2 x 8 = 16
ITEM_NUM_HALF      4    how many items to rotate per cycle
TIMING            10    timer period, ms
UNIT_DISPLACEMENT 0.5   vp advanced per TIMING ms  ->  50 vp/s
```

**The rotation is the whole idea.** The strip only ever scrolls through half the
data before the source is rotated and the offset is wound back by exactly the
same distance:

```
rollOffset grows ...
  when rollOffset > ITEM_WIDTH * ITEM_NUM_HALF (400 vp):
      move ITEM_NUM_HALF items from the head to the tail
      rollOffset -= ITEM_WIDTH * ITEM_NUM_HALF
  scroller.scrollTo({ xOffset: rollOffset, animation: false })
```

Four items leave the left edge and reappear at the right; the offset drops by
their combined width at the same instant. Both changes land in one timer tick,
so no frame ever shows the jump.

**Manual scrolling wraps in `onScrollFrameBegin` instead.** That hook runs
before each scroll frame is applied and returns how far the scroller is actually
allowed to move, so it can add or subtract a whole data width and hand back the
adjusted remainder:

```
newOffset = currentOffset + offset
  if newOffset < totalWidth * 0.5   ->  newOffset += totalWidth     (dragged left past the seam)
  if newOffset > totalWidth * 1.5   ->  newOffset -= totalWidth     (dragged right past the seam)
return { offsetRemain: newOffset - currentOffset }
```

Keeping the resting position between 0.5 and 1.5 data widths means there is
always a full data width of content on both sides of the viewport.

**The timer and the gesture take turns.** `onScrollStart` kills the interval,
`onScrollStop` restarts it. That handshake assumes the two always alternate,
which is where `HW-02-0062` lives; and the interval handle is cleared from the
wrong field on teardown, which is `HW-02-0061`.

## Implementation steps

1. **Size the data source at twice the visible run.** With `ITEM_NUM` distinct
   icons, push `2 * ITEM_NUM` entries so there is always a full screen ahead of
   the viewport. Build **distinct objects** for the second copy
   (`HW-02-0064`).
2. **Implement `IDataSource`** with the standard listener boilerplate, plus
   `deleteData(0)` and `pushData(x)` - those two are the rotation.
3. **Give every entry a stable unique id** and key `LazyForEach` on it
   (`HW-02-0063`). The index is not usable here: the rotation renumbers
   everything each cycle.
4. **Do not reuse a `GridItem`-rooted component in the strip** (`HW-02-0065`).
   `GridItem` may only be a child of `Grid`.
5. **Drive the offset from elapsed time, not from tick count**:
   `rollOffset += UNIT_DISPLACEMENT * (now - last) / TIMING`. A dropped frame
   then produces a bigger step rather than a slowdown.
6. **Rotate before you scroll,** inside the same tick, and subtract exactly
   `ITEM_WIDTH * ITEM_NUM_HALF` from the offset when you do.
7. **Scroll with `animation: false`.** The timer is the animation; letting
   `scrollTo` animate as well produces two competing interpolations.
8. **Set `pixelRound` to `NO_FORCE_ROUND`** on the cells so a 0.5 vp step is not
   snapped to whole pixels.
9. **Make `startAutoRoll` idempotent** - clear any live handle first
   (`HW-02-0062`) - and **clear the right field in `aboutToDisappear`**
   (`HW-02-0061`).
10. **Wrap manual scrolling in `onScrollFrameBegin`** and return
    `{ offsetRemain }`, not the raw offset.

## Verified snippets

All snippets are from `LoopScroll.zip`. Corrected forms are marked.

**The rotating data source - `LoopScroll.zip#entry/src/main/ets/view/TimeScrollView.ets:79`** (as shipped)

```typescript
class MyDataSource extends BasicDataSource {
  private dataArray: DataSource[] = [];

  public totalCount(): number {
    return this.dataArray.length;
  }

  public getData(index: number): DataSource {
    return this.dataArray[index % this.dataArray.length];   // never out of range
  }

  public pushData(data: DataSource): void {
    this.dataArray.push(data);
    this.notifyDataAdd(this.dataArray.length - 1);
  }

  public deleteData(index: number): void {
    this.dataArray.splice(index, 1);
    this.notifyDataDelete(index);
  }
}
```

**`notifyDataAdd` / `notifyDataDelete` rather than `notifyDataReload`.** The
rotation happens up to a hundred times a second; a reload would rebuild every
visible cell, while the add/delete notifications let `LazyForEach` move one item
from the head to the tail and leave the rest alone. That is the difference
between a smooth strip and a stuttering one.

The `% this.dataArray.length` in `getData` is a guard against the framework
asking for an index that the rotation has just invalidated.

**Seeding - same file, line 153** (corrected, see `HW-02-0064`)

```typescript
aboutToAppear(): void {
  // FIX: the sample runs this loop twice with the same `i` range, so the second
  //      pass re-assigns and re-pushes the FIRST eight objects. Build 2 x ITEM_NUM
  //      distinct entries instead:
  for (let i = 0; i < CommonConstants.ITEM_NUM * 2; i++) {
    const item = new DataSource();
    item.col = ListDataConstants.SCROLL_COL_LIST[i % CommonConstants.ITEM_NUM];
    item.image = ListDataConstants.SCROLL_IMAGE_LIST[i % CommonConstants.ITEM_NUM];
    item.id = `icon-${i}`;                    // FIX: LazyForEach needs a unique key
    this.dataSource.push(item);
    this.data.pushData(item);
  }
  this.startAutoRoll();
}

aboutToDisappear(): void {
  clearInterval(this.intervalNum);            // FIX: the sample clears intervalID, never assigned
}
```

**Two copies, not two references.** The shipped second loop allocates eight new
`DataSource`s and then writes to `this.dataSource[i]` - indices 0-7 - so the new
objects are never configured and the old ones are pushed a second time. The
strip therefore contains eight object identities in sixteen slots, which makes
the `LazyForEach` key collision unavoidable no matter what the key generator
returns.

**The timer - same file, line 128** (corrected, see `HW-02-0062`)

```typescript
startAutoRoll() {
  if (this.intervalNum !== CommonConstants.INIT_ZERO) {      // FIX: idempotent
    clearInterval(this.intervalNum);
    this.intervalNum = CommonConstants.INIT_ZERO;
  }
  this.last = new Date().getTime();
  this.intervalNum = setInterval(() => {
    if (this.rollOffset > (this.itemWidth * CommonConstants.ITEM_NUM_HALF)) {
      for (let i = 0; i < CommonConstants.ITEM_NUM_HALF; i++) {
        // 前减后加: take from the head, give to the tail
        let tempData = this.data.getData(0);
        this.data.deleteData(0);
        this.data.pushData(tempData);
      }
      // wind the offset back by exactly what the rotation moved
      this.rollOffset -= this.itemWidth * CommonConstants.ITEM_NUM_HALF;
    }
    let curr = new Date().getTime();
    this.rollOffset += CommonConstants.UNIT_DISPLACEMENT * (curr - this.last) / CommonConstants.TIMING;
    this.scroller.scrollTo({ xOffset: this.rollOffset, yOffset: 0, animation: false });
    this.last = curr;
  }, CommonConstants.TIMING);
}
```

**The rotation and the offset correction must be the same distance.** Four items
of 100 vp move to the tail, and 400 vp comes off `rollOffset`. Get either number
wrong and the strip jumps by the difference once per cycle.

**Elapsed time, not tick count.** `UNIT_DISPLACEMENT * (curr - last) / TIMING`
means the strip covers the same ground per second whether the timer fires on
schedule or late. A plain `rollOffset += UNIT_DISPLACEMENT` would slow down
under load.

**`animation: false` is required.** The timer already produces one step per
10 ms; asking `scrollTo` to animate between steps stacks a second interpolation
on top and the motion judders.

`getData(0)` before `deleteData(0)` - the value has to be captured before the
splice, because `deleteData` shifts everything down.

**Manual scrolling and the wrap - same file, line 196** (as shipped)

```typescript
.friction(CommonConstants.SCROLL_FRICTION)        // 0.5 - a long, slow fling
.onScrollStart(() => {
  clearInterval(this.intervalNum);
})
.onScrollStop(() => {
  this.startAutoRoll();
})
.onScrollFrameBegin((offset: number) => {
  let currOffset = this.scroller.currentOffset().xOffset;
  let newOffset = currOffset + offset;
  let totalWidth = this.itemWidth * CommonConstants.ITEM_NUM;    // 800
  if (newOffset < totalWidth * 0.5) {            // dragged past the left seam
    newOffset += totalWidth;
  } else if (newOffset > totalWidth * 1.5) {     // dragged past the right seam
    newOffset -= totalWidth;
  }
  this.rollOffset = newOffset;                   // hand the timer the corrected position
  return { offsetRemain: newOffset - currOffset };
});
```

**`onScrollFrameBegin` is the only hook that can rewrite a scroll before it
happens.** It receives the offset the scroller intends to apply and returns
`offsetRemain` - what it is actually allowed to apply. Returning
`newOffset - currOffset` after wrapping means the teleport is applied as part of
the same frame, so the user never sees the content at the seam.

**Writing `this.rollOffset` here is what keeps the two mechanisms in sync.**
When the finger lets go, `onScrollStop` restarts the timer, and the timer
continues from wherever the gesture left the strip rather than from its own
stale accumulator.

The window `[0.5w, 1.5w]` is chosen so a wrap always lands a full data width
away from both edges - there is content to show in either direction no matter
which way the fling was going.

**The cell - same file, line 173** (corrected, see `HW-02-0065`)

```typescript
Scroll(this.scroller) {
  Row() {
    LazyForEach(this.data, (item: DataSource) => {
      Column() {
        ScrollItemComponent({ itemName: item.col, itemImage: item.image });
        // FIX: the sample uses GridItemComponent, whose root is GridItem() -
        //      "This component can be used only as a child of Grid."
      }
      .height(CommonConstants.FULL_PERCENT)
      .width(this.itemWidth)
      .pixelRound({ start: PixelRoundCalcPolicy.NO_FORCE_ROUND,
                    end: PixelRoundCalcPolicy.NO_FORCE_ROUND });
    }, (item: DataSource) => item.id);       // FIX: sample returns the object itself
  }
  .height(CommonConstants.SCROLL_HEIGHT);
}
.scrollable(ScrollDirection.Horizontal)
.scrollBar(BarState.Off)
```

**`pixelRound(NO_FORCE_ROUND)` is the reason the motion looks smooth.** The
timer advances the offset by 0.5 vp per tick; with the default rounding the cell
edges would snap to whole pixels and the strip would tick rather than glide.
Applying it on `start`/`end` (the horizontal edges) is exactly the axis that
moves.

**`friction(0.5)`** halves the deceleration of a fling, so a manual flick coasts
for a long time - which suits a marquee, and gives `onScrollFrameBegin` plenty of
frames to wrap in.

**The manual-paging half - `LoopScroll.zip#entry/src/main/ets/pages/HomePage.ets:26`** (as shipped)

```typescript
@Builder
moduleView() {
  Swiper(this.swiperController) {
    ForEach(ListDataConstants.GRID_COL_LIST, (item: Resource[], index) => {
      MenuItemComponent({ nameList: item, imageList: ListDataConstants.SWIPER_IMAGE_LIST[index] });
    }, (item: string[], index: number) => index + JSON.stringify(item));
  }
  .align(Alignment.Center)
  .width(CommonConstants.FULL_PERCENT)
  .height(CommonConstants.SWIPER_HEIGHT);
}
```

and one page - `LoopScroll.zip#entry/src/main/ets/view/MenuItemComponent.ets:24`:

```typescript
Grid() {
  ForEach(this.nameList, (item: Resource, index: number) => {
    GridItemComponent({ itemName: item, itemImage: this.imageList[index] });
  }, (item: string, index: number) => index + JSON.stringify(item));
}
.columnsTemplate(CommonConstants.GRID_COLUMNS_TEMPLATE)
.height(Math.ceil(this.nameList.length / CommonConstants.HEIGHT_PER_ROW) * CommonConstants.HEIGHT_GRID);
```

**One `Swiper` child per page.** `Swiper` pages by direct child, so a page of
eight icons is one `MenuItemComponent` containing a `Grid`, not eight children.

**The `Grid` height is computed, not fixed** - `ceil(count / perRow) * rowHeight`
- so a page with a different number of icons still sizes itself. This is the
part of the sample most worth copying verbatim.

The document's snippet shows this `Swiper` with a bare `MenuItemComponent()` and
no `ForEach` and no props (`HW-02-0066`), which renders one empty page.

## Permissions & config

None. `LoopScroll.zip#entry/src/main/module.json5` declares no
`requestPermissions` block.

```json5
{
  "module": {
    "name": "entry",
    "type": "entry",
    "mainElement": "EntryAbility",
    "deviceTypes": ["phone", "tablet", "2in1"],
    "deliveryWithInstall": true,
    "installationFree": false,
    "pages": "$profile:main_pages",
    "abilities": [{
      "name": "EntryAbility",
      "srcEntry": "./ets/entryability/EntryAbility.ets",
      "exported": true,
      "skills": [{ "entities": ["entity.system.home"], "actions": ["action.system.home"] }]
    }],
    "extensionAbilities": [{
      "name": "EntryBackupAbility",
      "srcEntry": "./ets/entrybackupability/EntryBackupAbility.ets",
      "type": "backup",
      "exported": false,
      "metadata": [{ "name": "ohos.extension.backup", "resource": "$profile:backup_config" }]
    }]
  }
}
```

`main_pages.json` is `{ "src": ["pages/HomePage"] }`. Root
`build-profile.json5` targets `6.0.0(20)`.

`EntryAbility.onCreate` pins light mode; it does **not** touch the window layout
or avoid areas at all. The page instead uses
`.expandSafeArea([SafeAreaType.SYSTEM], [SafeAreaEdge.BOTTOM])`
(`HomePage.ets:65`), which is the declarative equivalent and needs no
`AppStorage` round trip - the cleanest safe-area handling of any sample in this
industry.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 50-52).
- The cell width is a constant (100 vp) and the wrap arithmetic depends on it;
  variable-width items would need the running width tracked instead.
- A 10 ms interval is a hundred callbacks a second, each doing a `scrollTo` and,
  once per cycle, four `splice`s with listener notifications. It is not free -
  and there is no pause when the app goes to the background, because
  `EntryAbility.onBackground` only logs.
- `ITEM_NUM_HALF` (4) must divide the visible run: the rotation only stays
  invisible while at least four cells are off-screen to the right.
- The strip only scrolls in one direction under the timer. Reversing needs the
  rotation reversed too (tail to head) and the offset wound the other way.
- The `Swiper` pages come from a two-element constant array; there is no data
  layer.

## Pitfalls

- **`HW-02-0061` - `aboutToDisappear` clears `intervalID`, which is never
  assigned.** `setInterval` returns into `intervalNum`. The timer outlives the
  component and keeps calling `scrollTo` on a dead scroller a hundred times a
  second, and its handle is unreachable. This is the one defect that makes the
  sample unfit to copy as-is.
- **`HW-02-0062` - `startAutoRoll` does not clear a running timer** before
  starting a new one, and it is called from both `aboutToAppear` and
  `onScrollStop`. Any unmatched `onScrollStop` doubles the carousel speed
  permanently.
- **`HW-02-0063` - the `LazyForEach` key generator is `(item: string) => item`**
  on a `DataSource` object, so all sixteen entries share one key. The guidance:
  "Duplicate key values will cause rendering issues." Give the class an id.
- **`HW-02-0064` - the second seeding loop re-indexes the first eight
  elements,** pushing the same objects twice and leaving eight allocations
  unused. Use `i + ITEM_NUM`, or one loop over `2 * ITEM_NUM` with
  `i % ITEM_NUM` for the resources.
- **`HW-02-0065` - `GridItemComponent`'s root is `GridItem()`** and the strip
  uses it inside `Scroll > Row > Column`. The reference: "This component can be
  used only as a child of `Grid`." Split the component.
- **`HW-02-0066` - the document's snippet shows a `Swiper` with a bare
  `MenuItemComponent()`** - no `ForEach`, no props - and replaces both
  load-bearing bodies with comments.
- **`HW-02-0067` - `nextNum` is maintained inside the 10 ms timer and never
  read.**
- **Do not use `notifyDataReload` for the rotation.** At this frequency it
  rebuilds every visible cell; `notifyDataAdd`/`notifyDataDelete` move one.
- **Do not let `scrollTo` animate.** The timer is the animation.
- **Do not key on the index.** The rotation renumbers every entry each cycle.
- **Do not omit `pixelRound(NO_FORCE_ROUND)`.** A 0.5 vp step snapped to whole
  pixels is a visible stutter.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `Scroll`, `Scroller.scrollTo`/`currentOffset`, `onScrollFrameBegin` and its `offsetRemain` return, `onScrollStart`/`onScrollStop`, `friction`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `IDataSource`, `DataChangeListener`, the key-uniqueness rule and the duplicate-key failure case
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-references/02_application-framework/ts-container-griditem.md` - "This component can be used only as a child of Grid"
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-griditem
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `columnsTemplate`, `rowsGap`/`columnsGap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-container-swiper.md` - `Swiper`, `SwiperController`, page-per-child
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-swiper
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-pixel-round.md` - `pixelRound`, `PixelRoundCalcPolicy.NO_FORCE_ROUND`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-pixel-round
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-expand-safe-area.md` - `expandSafeArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-expand-safe-area
- `LIFE-06` - the same industry's other timer-and-gesture card, with the teardown done correctly
- `LIFE-01` - the industry shell this page would sit in
