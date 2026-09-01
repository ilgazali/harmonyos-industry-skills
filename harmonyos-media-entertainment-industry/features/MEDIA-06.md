---
id: MEDIA-06
title: Parallel tag filtering - four multi-select groups ANDed over a LazyForEach grid
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/06_hiararchicle_filtering.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/hiararchicle_filtering-0000002287162465
sample: huawei_industry_tree/13_media_entertainment/downloads/HierarchicalFiltering.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.BasicServicesKit", "@kit.PerformanceAnalysisKit"]
apis: [LazyForEach, IDataSource, DataChangeListener, notifyDataAdd, notifyDataDelete, Grid, GridItem, Flex, FlexWrap, LengthMetrics, List, ListItem, Navigation, NavDestination, NavPathStack, pushPathByName, getParamByName, "@Observed", "@ObjectLink", "@Provide", "@Consume", "@StorageProp", "resourceManager.getStringSync", "window.getLastWindow", setWindowSystemBarEnable, "UIContext.px2vp"]
permissions: []
min_api: 24
modules: [entry]
findings: [HW-13-0018, HW-13-0019, HW-13-0020, HW-13-0021, HW-13-0098]
status: verified-with-fixes
---

## When to use

Load this card when a catalogue needs **several independent filter bars at
once** - genre and resolution and year and region, each multi-select, all
applied together. It is the standard shape of a video library, a photo
archive, a font or asset store, a food-delivery category page.

The pattern is a matrix: filters are a `SelectionButtonType[][]` (one inner
array per group), the current selection is a parallel `[[], [], [], []]`, and
each item carries a `filterType: string[]` whose **positions line up with the
groups**. Filtering is then two nested loops: OR within a group, AND across
groups. Once you see the positional alignment, adding a fifth filter group is
one row in three arrays and nothing else changes.

The other half is the rendering side: results go through `LazyForEach` over a
custom `IDataSource` so a filter click can rebuild the visible set without
re-laying-out the whole grid. **Read `HW-13-0018` before copying that data
source** - its `clear()` is off by one and fires a delete notification for an
index that does not exist on every single filter click.

## Feature checklist

- A photos page with a title and four horizontal filter strips: 类型 (type),
  内容 (content), 年份 (year), 地区 (area).
- Every chip is independently selectable; tapping toggles it and recolours it
  blue.
- Selecting two chips in the same group widens the result set (OR).
- Selecting chips in different groups narrows it (AND).
- Deselecting every chip in a group makes that group stop constraining.
- Results render as a two-column grid of cards, each with an image, a title and
  its own tag chips.
- Tapping a card pushes a preview page carrying the item.
- The preview hides the system bars; tapping the image toggles them back with a
  header and a footer.

## Architecture

One `entry` module, two data files and two pages. The item list is a JSON
rawfile, not code.

```
entry/src/main/ets
├── data/ButtonData.ets        @Observed SelectionButtonType {index, str, isClicked}
├── data/PhotoData.ets         Photo, DataManager, BasicDataSource, PhotoDataSource
├── entryability/EntryAbility.ets   full screen + avoid areas -> AppStorage
├── entrybackupability/
├── pages/FilterPage.ets       @Entry: the four strips, filter()/update(), the Grid
└── pages/PreviewPage.ets      ItemPage NavDestination: the full-bleed preview
entry/src/main/resources/rawfile/photo_item_list.json    the catalogue
entry/src/main/resources/base/profile/route_map.json     FilterPage + PreviewPage builders
```

The documented tree matches the zip. What the document gets wrong is a type
name: its pseudo-code declares `update(isClicked: boolean, button: NewButtonType)`,
and `NewButtonType` exists nowhere in the project - the real type is
`SelectionButtonType` (`HW-13-0020`). A reader grepping `ButtonData.ets` for
the documented name finds nothing.

The environment block claims **API Version 24 Release**, and
`compatibleSdkVersion` in the zip is `6.1.1(24)`, so document and project at
least agree with each other - unusually for this industry, where three sibling
docs claim API 20 over samples built for 5.0.4/5.0.5 (`HW-13-0004`). Whether
an "API Version 24 Release" exists at all is the open question behind
`HW-13-0021`; every other doc in this category states API 20.

**The design decision worth copying** is the two data sources.
`originPhoto` holds the complete catalogue and is never mutated after
`aboutToAppear`; `showedPhoto` holds the rendered subset and is cleared and
refilled on every filter change. `filter()` therefore always reads from the
full set, never from what is currently on screen, which is what makes
deselecting a chip *widen* the results rather than being stuck inside a
previously narrowed list. It is the difference between a filter and a
progressively destructive search.

**The part worth avoiding** is `BasicDataSource`. It implements `IDataSource`
but its `totalCount()` returns a hardcoded `0` and its `getData()` reads a
`originDataArray` that nothing ever fills; both are overridden in
`PhotoDataSource`, so the base class is a listener registry wearing an
`IDataSource` costume. Anyone who extends `BasicDataSource` and forgets to
override both methods gets an empty grid with no error. Either make the base
abstract or move the array into it.

## Implementation steps

1. **Give every item a positional tag array.** In `photo_item_list.json` each
   entry's `filterType` is `[type, content, year, area]`, and a cell that
   belongs to several tags is one comma-joined string (`"运动,人物"`).
2. **Mirror that shape in the filter state.** `buttons: SelectionButtonType[][]`
   is one inner array per group, and each button carries its own group `index`,
   so a click knows which bucket to update without the handler being told.
3. **Keep the selection in a parallel array** `clickedButton: [[], [], [], []]`;
   an empty inner array means "this group does not constrain".
4. **Filter with OR inside a group, AND across groups** - break out of the
   group loop the moment one group fails, and `continue` the item loop.
5. **Clear before refiltering, and clear correctly** (`HW-13-0018`): loop from
   `length - 1`, never from `length`.
6. **Fix `removeAll` before reusing this data source** (`HW-13-0019`): a
   forward loop that splices must decrement its index, or iterate backwards.
7. **Render through `LazyForEach`** over the `PhotoDataSource` so only visible
   `GridItem`s are built - and give it a **stable** key, not one derived from
   `new Date()`.
8. **Wire the preview through `Navigation`**: `pushPathByName('ItemPage', item)`
   plus a `routerMap` builder, and read the parameter in `onWillAppear`.
9. **Use the real type name in your own docs** - `SelectionButtonType`, not
   `NewButtonType` (`HW-13-0020`).

## Verified snippets

All snippets are from `HierarchicalFiltering.zip`. Corrected forms are marked.

**The two-loop filter — `entry/src/main/ets/pages/FilterPage.ets`** (as shipped)

```typescript
//根据类型过滤显示列表
filter() {
  for (let i = 0; i < this.originPhoto.totalCount(); i++) {
    let canAdd: boolean = false;
    for (let j = 0; j < 4; j++) {                      // j is BOTH the group and the tag position
      canAdd = false;
      if (this.clickedButton[j].length === 0) {
        canAdd = true;                                 // an empty group does not constrain
      } else {
        for (let k of this.clickedButton[j]) {
          let l = this.originPhoto.getData(i).filterType[j].toString();
          if (l?.indexOf(this.context.resourceManager.getStringSync(k.str.id)) !== -1) {
            canAdd = true;                             // OR within the group
          }
        }
      }
      if (!canAdd) {
        break;                                         // AND across groups
      }
    }
    if (!canAdd) {
      continue;
    }
    this.showedPhoto.pushData(this.originPhoto.getData(i));
  }
}

update(isClicked: boolean, button: SelectionButtonType) {
  this.showedPhoto.clear();
  if (isClicked) {
    this.clickedButton[button.index].push(button);
  } else {
    this.clickedButton[button.index].splice(this.clickedButton[button.index].indexOf(button), 1);
  }
  this.filter();
}
```

**The single index `j` is the whole trick.** It selects the filter group *and*
the position in the item's `filterType` array at the same time, which is why
neither the buttons nor the items need to name their category anywhere. Add a
fifth group and you add a fifth row to `buttons`, a fifth `[]` to
`clickedButton`, a fifth cell to every item, and change `j < 4` - no lookup
table, no string keys.

The `canAdd = false` reset at the top of the group loop is what makes the AND
work: each group must re-earn the item. Resetting *outside* the loop instead -
a common simplification - would let a single matching group carry an item past
all the others.

Two properties are worth knowing before you ship this. The comparison is
`indexOf(...) !== -1` on a comma-joined string, so a tag that is a substring of
another tag in the same group matches both - here `1080` and `480` are safe but
`高清` inside `超高清` would not be. And `getStringSync` resolves a resource
inside the innermost of three loops; for a catalogue of any size, resolve the
selected labels once into a `string[]` before entering the item loop.

**The data source — `entry/src/main/ets/data/PhotoData.ets`** (corrected, see `HW-13-0018`, `HW-13-0019`)

```typescript
export class PhotoDataSource extends BasicDataSource {
  private dataArr: Photo[] = [];

  public totalCount(): number {
    return this.dataArr.length;
  }

  public getData(index: number): Photo {
    return this.dataArr[index];
  }

  public pushData(data: Photo): void {
    this.dataArr.push(data);
    this.notifyDataAdd(this.dataArr.length - 1);
  }

  public removeAll(data: Photo): void {
    for (let i = 0; i < this.dataArr.length; i++) {
      if (this.dataArr[i].title === data.title) {
        this.dataArr.splice(i, 1);
        this.notifyDataDelete(i);
        i--;                                  // FIX: shipped code skips the shifted-in element
      }
    }
  }

  public clear(): void {
    for (let i = this.dataArr.length - 1; i >= 0; i--) {   // FIX: shipped code starts at length
      this.dataArr.splice(i, 1);
      this.notifyDataDelete(i);
    }
  }
}
```

**`clear()` is the one that actually fires.** `update()` calls it on every chip
tap, and the shipped loop starts at `this.dataArr.length`: the first iteration
splices at one past the end (a no-op) and then calls
`notifyDataDelete(length)` for an index that never existed. `LazyForEach`
maintains its own index bookkeeping from exactly these notifications, so it is
handed a delete for a row it does not have, on every interaction. Starting at
`length - 1` is the whole fix.

`removeAll()` is the classic mutate-while-iterating bug and is dead code in
this sample - nothing calls it. It is still worth fixing before the file is
copied into the next project, because `BasicDataSource`/`PhotoDataSource` is
exactly the kind of file that gets copied: two consecutive items with the same
title, and the second survives the removal.

Note also what `clear()` does *not* do: it emits one `notifyDataDelete` per
row rather than a single `notifyDataReload()`. For a full rebuild the reload
notification is both cheaper and harder to get wrong.

**The filter strip — `entry/src/main/ets/pages/FilterPage.ets`** (as shipped)

```typescript
@Builder
multiSelect(title: Resource, selection: SelectionButtonType[]) {
  Flex({ justifyContent: FlexAlign.Center }) {
    Text(title).fontSize(16).width(52).height(20);
    List({ space: 20, initialIndex: 0, scroller: this.scroller }) {
      ForEach(selection, (item: SelectionButtonType) => {
        ListItem() {
          SelectionButton({ nButton: item })
            .backgroundColor('#F1F3F5')
            .borderRadius(16)
            .onClick(() => {
              item.isClicked = !item.isClicked;      // @Observed -> the chip recolours
              this.update(item.isClicked, item);     // ...and the grid refilters
            });
        };
      }, (item: SelectionButtonType) => this.context.resourceManager.getStringSync(item.str.id) + new Date());
    }
    .width('85%')
    .listDirection(Axis.Horizontal)                  // a horizontal List, not a Row
    .scrollBar(BarState.Off)
    .friction(0.9);
  }
  .height('6%')
  .width('100%');
}
```

A horizontal `List` rather than a `Row` inside a `Scroll` is the right choice
for a chip strip whose content can exceed the width: it scrolls natively,
recycles its items, and `friction(0.9)` gives the short flick a strip this size
wants. `scrollBar(BarState.Off)` is not cosmetic here - a bar under a 6%-high
row would eat most of it.

`SelectionButtonType` is `@Observed` and `SelectionButton` takes it as `@Prop`,
so mutating `isClicked` on the original object both repaints the chip and is
visible to `update()`. The `@Builder` is invoked four times with a different
group, which is why the four strips are four lines in `build()`.

The key generator is the weak point: `getStringSync(...) + new Date()` produces
a **different key on every evaluation**, so the framework can never match an
existing chip to a new one. The same anti-pattern appears on the grid's
`LazyForEach` key (`item.title + new Date().toString()`), where it defeats the
entire point of using `LazyForEach`. Key on something stable - the resource id,
or the title.

**Grid, LazyForEach and the preview jump — same file** (as shipped)

```typescript
Navigation(this.pageInfos) {
  Column() {
    // ... the four multiSelect strips ...
    Scroll(this.scroller) {
      Grid() {
        LazyForEach(this.showedPhoto, (item: Photo) => {
          GridItem() {
            ItemView({ photo: item })
              .onClick(() => {
                this.pageInfos.pushPathByName('ItemPage', item);
              });
          };
        }, (item: Photo) => item.title + new Date().toString());
      }
      .columnsGap(16)
      .rowsGap(15);
    }
    .scrollBar(BarState.Off);
  }
}
.navDestination(this.routerMap)
.padding({ top: this.getUIContext().px2vp(this.topRectHeight) })
.hideToolBar(true)
.hideTitleBar(true);
```

`Navigation` here is doing route work, not chrome work - the title bar and
tool bar are both hidden and the page draws its own header. The value is
`pushPathByName('ItemPage', item)`, which carries the whole `Photo` object to
the destination; `ItemPage` reads it back in `onWillAppear` via
`getParamByName('ItemPage')` and never needs a shared store or an id lookup.

`topRectHeight` arrives from `EntryAbility` in **px** (the ability writes
`avoidArea.topRect.height` raw), so the page converts with `px2vp` at the point
of use. Compare `MEDIA-08`, whose ability converts before writing and whose
pages therefore use the value directly - both are valid, but a project must
pick one and the units must be checked at every read.

## Permissions & config

**None.** The sample declares no `requestPermissions`; the catalogue is a
bundled rawfile and the images are bundled media.

`deviceTypes` is `["phone", "tablet", "2in1", "car"]` - the widest set of the
four samples in this group. Note that the grid's card width is a hardcoded
`172` vp and the strips are percentage-based, so the layout will not actually
adapt to a car head unit or a wide 2in1 window without work.

`strictMode` is on in `build-profile.json5`:

```json5
"buildOption": {
  "strictMode": {
    "caseSensitiveCheck": true,
    "useNormalizedOHMUrl": true
  }
}
```

`caseSensitiveCheck: true` is why the systematic tree-naming defect
(`HW-13-0003`) matters in this industry: with this flag a file referenced under
the wrong case does not resolve at build time.

Routing is declarative, via `"routerMap": "$profile:route_map"` and
`route_map.json` naming `FilterPageBuilder` and `ItemPageBuilder` - which is
why both pages export a `@Builder` function alongside the struct.

## Constraints

- The document states API Version 24 Release and DevEco Studio 6.1.1 Release;
  the zip declares `compatibleSdkVersion` and `targetSdkVersion` `6.1.1(24)`.
  See `HW-13-0021` on whether that release designation is real - every sibling
  doc in this category states API 20.
- Tag matching is substring-based over a comma-joined string. Tags that are
  substrings of other tags in the same group will over-match.
- `getStringSync` is called inside the innermost filter loop and again in the
  chip key generator; both are per-frame resource lookups.
- The `LazyForEach` and `ForEach` key generators embed `new Date()`, so keys are
  never stable and no view is ever reused. On a large catalogue this removes
  the benefit of `LazyForEach` entirely.
- `BasicDataSource.totalCount()` returns `0` and `getData()` reads an array
  that is never populated; the class is only usable via `PhotoDataSource`.
- The preview page toggles the system bars with `window.getLastWindow` in both
  `aboutToAppear` and `aboutToDisappear` and on every image tap; there is no
  guard against the callbacks landing out of order.

## Pitfalls

- **`HW-13-0018`** (B/medium, confirmed): `PhotoDataSource.clear()` loops from
  `this.dataArr.length`, so the first pass splices nothing and calls
  `notifyDataDelete(length)` - an out-of-range delete notification sent to
  `LazyForEach` on every filter click, which can corrupt its index bookkeeping.
  Fix: start at `this.dataArr.length - 1` (or emit one `notifyDataReload()`).
- **`HW-13-0019`** (B/low, confirmed): `removeAll` splices inside a forward loop
  without decrementing `i`, so the element shifted into position `i` is never
  examined and consecutive matches are skipped. Dead code today, but it is the
  canonical bug in a data source written to be reused. Fix: `i--` after each
  splice, or iterate backwards.
- **`HW-13-0020`** (D/low, confirmed): the document's pseudo-code declares
  `update(isClicked: boolean, button: NewButtonType)`; no such type exists in
  the sample. The real one is `SelectionButtonType`. Fix: correct the document.
- **`HW-13-0021`** (E/low, probable): the 环境准备 block claims "API Version 24
  Release", an API level the sibling docs in this category do not recognise -
  they all state API 20 / HarmonyOS 6.0. Fix: state the real level.

## References

- `documentation/harmonyos-guides/03_application-framework/arkts-rendering-control-lazyforeach.md` - `IDataSource`, `DataChangeListener`, the notification contract and key requirements
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-rendering-control-lazyforeach
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-create-grid.md` - `Grid`/`GridItem`, `columnsGap`/`rowsGap`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-create-grid
- `documentation/harmonyos-guides/03_application-framework/arkts-layout-development-flex-layout.md` - `Flex`, `FlexWrap`, `LengthMetrics` spacing
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-layout-development-flex-layout
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navigation.md` - `Navigation`, `NavPathStack.pushPathByName`, `navDestination`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navigation
- `documentation/harmonyos-guides/03_application-framework/arkts-set-navigation-routing.md` - the `routerMap` profile and named routes
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-set-navigation-routing
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getLastWindow`, `setWindowSystemBarEnable`, `getWindowAvoidArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `MEDIA-08` - the other avoid-area convention (px converted in the ability, not the page)
