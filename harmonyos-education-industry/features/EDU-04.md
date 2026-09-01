---
id: EDU-04
title: Two-axis timetable - frozen header and rail synchronised to a scrolling grid
industry: 04_education
doc: huawei_industry_tree/04_education/docs/04_horizental_and_vertical_scrolling_list.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/horizental_and_vertical_scrolling_list-0000002296361950
sample: huawei_industry_tree/04_education/downloads/HorizentalAndVerticalScrollingList.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [Scroll, Scroller, scrollBy, scrollTo, onScrollFrameBegin, ScrollDirection, List, listDirection, "EdgeEffect.None", ForEach, Tabs, TabContent, onContentWillChange, barModifier, CommonModifier, position, zIndex, "@StorageProp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-04-0019, HW-04-0020, HW-04-0021, HW-04-0022, HW-04-0023, HW-04-0024, HW-04-0025, HW-04-0026, HW-04-0155]
status: verified-with-fixes
---

## When to use

Load this card when you need a **spreadsheet-shaped view**: a grid that scrolls
in both axes with a **frozen header row and a frozen left rail** that follow it.
ArkUI has no such component - `Grid` scrolls one axis, and a `List` of `List`s
loses the header.

The construction is five `Scroller`s and one rule: **`onScrollFrameBegin` reads
the frame's pending offset and pushes it to the peers with `scrollBy`.**
`scrollBy` does not itself fire `onScrollFrameBegin`, so the peers move without
echoing back - that asymmetry is what makes the whole thing terminate.

The education case is a weekly timetable: periods and times down the left,
weekdays across the top, courses in the middle. The same shape fits an exam
schedule, a seat map, a gradebook, a room-booking grid.

## Feature checklist

- Left rail: period number (50 vp) and class time (60 vp), scrolls vertically
  only.
- Top header: seven weekday columns, 120 vp each, scrolls horizontally only.
- Centre grid: 8 periods x 7 days of course cells, scrolls in both axes.
- Dragging the grid horizontally moves the weekday header with it; dragging it
  vertically moves both left rails.
- Dragging the header horizontally moves the grid; dragging either rail
  vertically moves the grid and the other rail.
- No bounce at any edge - the edges are hard stops.
- On open, the table scrolls to today's column.

## Architecture

One `entry` module, one page. There is no view model - the timetable is a
`Course[][]` literal.

```
entry/src/main/ets
├── datamodel/Course.ets                 the Course class and COURSE_MODEL
├── entryability/EntryAbility.ets        full screen, avoid areas -> AppStorage
├── entrybackupability/EntryBackupAbility.ets
└── pages/CourseTablePage.ets            @Entry - the whole timetable
```

The document's tree names the `entrybackupability` directory `entryability`
(`HW-04-0024`).

**Five scrollers, and which one drives which:**

| Scroller | Axis | On its own scroll it pushes to |
| --- | --- | --- |
| `weekdaysScroller` | horizontal | `horizontalScroller` |
| `horizontalScroller` | horizontal | `weekdaysScroller` |
| `classScroller` (period rail) | vertical | `verticalScroller`, `timeScroller` |
| `timeScroller` (time rail) | vertical | `verticalScroller`, `classScroller` |
| `verticalScroller` (grid) | vertical | `classScroller`, `timeScroller` |

The grid is a `Scroll(horizontalScroller)` **wrapping** a
`Scroll(verticalScroller)`: the outer one owns the horizontal axis, the inner
one the vertical, so one nested pair covers both directions and each still has a
single-axis `onScrollFrameBegin`.

**The layout is two absolutely positioned Rows inside a Column** - the header
`Row` at `.position({ x: 0, y: 0 })` with `zIndex(10)` and a shadow, the body
`Row` at `.position({ x: 0, y: 42 })`. That is what keeps the header above the
grid while the grid slides underneath. The document's sketch shows a single flat
`Row` instead (`HW-04-0025`).

## Implementation steps

1. **Declare one `Scroller` per independently draggable region.** Five here; do
   not try to share one.
2. **Nest the grid's two axes**: `Scroll(horizontalScroller)` outside,
   `Scroll(verticalScroller)` inside, each with an explicit
   `.scrollable(ScrollDirection.Horizontal | Vertical)`.
3. **Give the inner grid an explicit pixel width** -
   `.width(this.classificationNames.length * 120)` - so the horizontal `Scroll`
   has something wider than the viewport to scroll.
4. **In every `onScrollFrameBegin`, push the offset to the peers and return it
   unchanged:**
   ```typescript
   .onScrollFrameBegin((offset: number) => {
     this.peerScroller.scrollBy(0, offset);
     return { offsetRemain: offset };
   })
   ```
   Returning `offsetRemain: offset` means "scroll the full requested amount";
   returning less is how you would rubber-band or clamp.
5. **Set `.edgeEffect(EdgeEffect.None)` on every `List` inside these
   `Scroll`s.** The Scroll reference makes this a precondition of the pattern,
   not a nicety - the document omits it (`HW-04-0023`).
6. **Turn the scroll bars off** (`.scrollBar(BarState.Off)`) on all five; three
   visible bars for one logical surface is noise.
7. **Pin the header** with `.position({ x: 0, y: 0 })`, `zIndex(10)` and a
   shadow, and offset the body `Row` by the header height (42).
8. **Scroll to today in `onAppear`**, on *both* horizontal scrollers, and map
   `Date.getDay()` to a Monday-first index first - the sample's guard misses
   Sunday (`HW-04-0022`).
9. **Key every `ForEach` from the data.** The sample keys from `new Date()` and
   a global counter, which defeats reuse entirely (`HW-04-0020`).

## Verified snippets

All snippets are from `HorizentalAndVerticalScrollingList.zip`. Corrected forms are marked.

**The five scrollers — `entry/src/main/ets/pages/CourseTablePage.ets`** (as shipped)

```typescript
weekDay: number = new Date().getDay();
classScroller = new Scroller();       // period numbers, left rail
timeScroller = new Scroller();        // class times, second rail
weekdaysScroller = new Scroller();    // the frozen header
horizontalScroller = new Scroller();  // grid, x axis
verticalScroller = new Scroller();    // grid, y axis
```

**The header, synchronised to the grid — same file** (corrected, see `HW-04-0021`)

```typescript
Scroll(this.weekdaysScroller) {
  List() {
    ForEach(this.classificationNames, (item: Resource) => {
      ListItem() {
        Row() {
          Column() {
            Text(item).fontSize(14).textAlign(TextAlign.Center).width('100%')
          }
          .width(120).height(42).backgroundColor(Color.White)
        }
      }
    }, (item: Resource, index: number) => index.toString())   // FIX: sample keys on new Date()
  }
  .listDirection(Axis.Horizontal)
  .edgeEffect(EdgeEffect.None)          // REQUIRED for onScrollFrameBegin + scrollBy
}
.scrollable(ScrollDirection.Horizontal)
.scrollBar(BarState.Off)
.height(42)
.onScrollFrameBegin((offset: number) => {
  this.horizontalScroller.scrollBy(offset, 0);   // drag the header -> the grid follows
  return { offsetRemain: offset };
})
```

**`EdgeEffect.None` is load-bearing.** The Scroll reference is explicit: with
`onScrollFrameBegin` and `scrollBy` used for nested scrolling, a `List` inside
must set it, "otherwise, swiping the List triggers its edge bounce animation,
which results in failed nested scrolling". The header and the grid drift apart
at every edge without it. The document's three-step write-up never mentions it.

**The grid, both axes — same file** (as shipped)

```typescript
Scroll(this.horizontalScroller) {
  Scroll(this.verticalScroller) {
    Column() {
      ForEach(this.arr, (_temp: number) => {
        Row() {
          ForEach(this.data[this.arr[_temp]], (item: Course) => {
            this.itemBuilder(item.name, 100, 120, item.backColor, Color.Black);
          }, () => getResourceID().toString())        // see HW-04-0020
        }
      }, (_temp: number) => _temp + new Date().toString())   // see HW-04-0020
    }
  }
  .scrollBar(BarState.Off)
  .scrollable(ScrollDirection.Vertical)
  .height('100%')
  .width(this.classificationNames.length * 120)     // 7 * 120 - wider than the viewport
  .onScrollFrameBegin((offset: number) => {
    this.classScroller.scrollBy(0, offset);          // grid moves -> both rails follow
    this.timeScroller.scrollBy(0, offset);
    return { offsetRemain: offset };
  })
}
.scrollBar(BarState.Off)
.scrollable(ScrollDirection.Horizontal)
.onScrollFrameBegin((offset: number) => {
  this.weekdaysScroller.scrollBy(offset, 0);         // grid moves -> header follows
  return { offsetRemain: offset };
})
```

**Why this does not loop.** Every scroller pushes to its peers, and the peers
push back - on paper that is mutual recursion. It terminates because of one
sentence in the Scroll reference: `onScrollFrameBegin` fires for user
interaction, inertia and `fling`, and **"This event is not triggered when ... a
scroll control API other than fling is called."** `scrollBy` is such an API, so
a programmatic move is silent. Only the finger starts a cascade, and the cascade
is one level deep.

That also tells you what **not** to substitute: `fling` would re-enter, and
`onDidScroll`/`onScroll` fire for programmatic moves too and would loop.

**The left rails — same file** (as shipped)

```typescript
Scroll(this.classScroller) { /* List of period numbers, width 50 */ }
  .position({ x: 0, y: 0 })
  .width(50)
  .scrollBar(BarState.Off)
  .onScrollFrameBegin((offset: number) => {
    this.verticalScroller.scrollBy(0, offset);
    this.timeScroller.scrollBy(0, offset);
    return { offsetRemain: offset };
  })

Scroll(this.timeScroller) { /* List of class times, width 60 */ }
  .position({ x: 50, y: 0 })            // sits immediately right of the period rail
  .width(60)
  .scrollBar(BarState.Off)
  .onScrollFrameBegin((offset: number) => {
    this.verticalScroller.scrollBy(0, offset);
    this.classScroller.scrollBy(0, offset);
    return { offsetRemain: offset };
  })
```

Each rail pushes to the grid **and** to the other rail - with three participants
on the vertical axis, every one of them must name the other two, or dragging one
rail leaves the third behind.

**Open on today — same file** (corrected, see `HW-04-0022`)

```typescript
// FIX: Date.getDay() is 0 for Sunday; the columns are Monday-first
private todayColumn(): number {
  return (new Date().getDay() + 6) % 7;      // Mon -> 0 ... Sun -> 6
}

Column() { /* header Row + body Row */ }
  .height('88%')
  .onAppear(() => {
    const col = this.todayColumn();
    if (col > 0) {
      this.weekdaysScroller.scrollTo({ xOffset: col * 120, yOffset: 0 });
      this.horizontalScroller.scrollTo({ xOffset: col * 120, yOffset: 0 });
    }
  })
```

**Both horizontal scrollers must be moved.** `scrollTo`, like `scrollBy`, does
not fire `onScrollFrameBegin`, so moving only the grid would leave the header
behind. The same property that prevents the loop obliges you to drive the pair
by hand here.

## Permissions & config

None. `entry/src/main/module.json5` declares no `requestPermissions` block; the
feature is pure UI over local data.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The column width 120, the row height 100 and the header height 42 are
  inlined** in several places each (cell builder defaults, grid width, header
  height, scroll-to arithmetic). Changing the column width means editing all of
  them.
- The grid is a fixed `Course[][]` of 8 periods x 7 days built from a
  `COURSE_MODEL` literal. There is no data source, no week switching (the title
  第8周 is a `Text`), and the 主页 / 消息 tabs are placeholders reading
  功能待在实际使用场景中实现 (to be implemented in a real scenario).
- Because the cells are plain `ForEach` inside a `Scroll` and not `LazyForEach`,
  all 56 are built up front. That is fine at this size and would not be at a
  term's worth of columns.
- The two tabs are unreachable in any case until `HW-04-0019` is fixed.

## Pitfalls

- **`HW-04-0019` — `onContentWillChange` returns `false`,** which vetoes every
  tab switch. Two of the three tabs are dead; only the bar highlight moves.
  Remove the callback.
- **`HW-04-0020` — three key generators are not persistent.** Two build the key
  from `new Date().toString()`, the third from a module-level counter that
  increments on every call. A key that changes every pass tells ArkUI the
  element was replaced, so all 56 cells and the header are destroyed and rebuilt
  on any state change.
- **`HW-04-0021` — the weekday key generator is typed `(item: string)` for a
  `Resource[]`,** so every key stringifies to the same `[object Object]` prefix
  and only the timestamp separates them.
- **`HW-04-0022` — Sunday never scrolls to today.** `Date.getDay()` returns 0
  for Sunday, the guard is `> 1`, and the columns are Monday-first. Convert to a
  Monday-first index before using it.
- **`HW-04-0023` — the document omits `EdgeEffect.None`,** which the Scroll
  reference names as a precondition for `onScrollFrameBegin` + `scrollBy`
  nesting. The code has it; a reader following only the three documented steps
  will not.
- **`HW-04-0024` — the project tree lists `entryability` twice,** the second
  holding `EntryBackupAbility.ets`. The real directory is `entrybackupability`,
  which is what `module.json5` points at.
- **`HW-04-0025` — the layout sketch is a flat `Row`,** losing the two
  absolutely positioned rows and the 42 vp header offset that are what freeze
  the header.
- **`HW-04-0026` — `barModifier` carries an empty `alignRules({})`,** an
  attribute that only applies inside a `RelativeContainer`. The whole modifier
  is a no-op.
- **Do not reach for `onScroll` or `onDidScroll` to drive the peers.** They fire
  for programmatic scrolls too, and the mutual pushes become an infinite loop.
  `onScrollFrameBegin` is the correct hook precisely because `scrollBy` is
  invisible to it.

## References

- `documentation/harmonyos-references/02_application-framework/ts-container-scroll.md` - `onScrollFrameBegin`, what does and does not trigger it, `scrollBy`, `scrollTo`, and the `EdgeEffect.None` requirement
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-scroll
- `documentation/harmonyos-references/02_application-framework/ts-container-list.md` - `listDirection`, `edgeEffect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-list
- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-foreach.md` - the "unique and persistent key" rule and the default key generator
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-foreach
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `onContentWillChange` and `OnTabsContentWillChangeCallback`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `position`, and `alignRules` being `RelativeContainer`-only
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-show-hide.md` - `onAppear`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-show-hide
- `EDU-03` - the same `alignRules`-outside-a-`RelativeContainer` mistake in this industry
