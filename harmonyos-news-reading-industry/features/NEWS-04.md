---
id: NEWS-04
title: Channel subscription editor - a bindSheet half-modal with a drag-sortable Grid
industry: 11_news_reading
doc: huawei_industry_tree/11_news_reading/docs/04_channel_selection.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/channel_selection-0000002270325497
sample: huawei_industry_tree/11_news_reading/downloads/ChannelSelection.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.PerformanceAnalysisKit"]
apis: [hilog, window, bindSheet, Grid, GridItem, editMode, supportAnimation, onItemDragStart, onItemDrop, ItemDragInfo, "@Link", "@Watch", "@StorageProp", Tabs, TabContent, TabsController, Scroller, "UIContext.getMeasureUtils", measureTextSize, onContentWillChange, safeAreaPadding]
permissions: []
min_api: 20
modules: [entry (entry)]
findings: [HW-11-0006, HW-11-0031]
status: verified
---

## When to use

Load this card when the user must be able to **reorder and curate a tab strip
themselves** - the "my channels" editor that sits behind the `+` at the right
end of a scrolling tab bar. The pattern: a half-modal sheet holding two grids,
one for what is subscribed (drag-sortable, removable) and one for everything
else (tap to add), with a single `Set` of ids keeping the two in step.

The transferable machinery is `Grid` in edit mode. `editMode(true)` plus
`supportAnimation(true)` turns every `GridItem` into a drag source with the
reflow animation supplied by the framework; you only supply the proxy that
follows the finger (`onItemDragStart`) and the array move (`onItemDrop`). The
same three lines cover a home-screen icon grid, a dashboard-card arranger, a
playlist reorder - anything where the model is an array and the gesture is
"put this one there".

The sheet is worth copying too. The trigger (`+`) lives inside the tab-bar
component, the sheet is bound by the page that owns the channel array, and the
two are joined by one `@Link` boolean. Editing the array inside the sheet
re-renders the tab bar behind it live, with no save step and no copy.

**Read `HW-11-0006` before copying from the document.** The document's own
snippet opens the sheet by writing a variable that does not exist anywhere in
the sample.

## Feature checklist

- A red header bar, a horizontally scrolling channel tab strip, and a `+` icon
  pinned at the right end of the strip.
- Tapping `+` raises a half-modal sheet over the page.
- The sheet shows 我的订阅 (my subscription) on top and 精选频道 (featured
  channels) below; the second grid never shows a channel already in the first.
- An 编辑 / 保存 (edit / save) text button toggles edit mode for both grids at
  once.
- In edit mode each subscribed chip carries a close cross and each candidate
  chip a plus.
- In edit mode a long press picks a subscribed chip up; dropping it on another
  position reorders the strip, with the remaining chips animating into place.
- Removing the last remaining channel is refused with a toast.
- The tab strip behind the sheet reflects additions, removals and reordering
  immediately.
- Selecting a tab scrolls the strip so the selected tab is visible and moves
  the blue indicator line, sized to the tab's measured text width.
- The three non-home bottom tabs raise a "demo only" toast instead of
  switching.

## Architecture

One `entry` module. No network, no persistence, no ability code beyond window
setup.

```
entry/src/main/ets
├── common/CommonConstants.ets       chip/tab geometry + the two theme colours
├── components
│   ├── ChannelSelection.ets         the candidate grid (精选频道), add on tap
│   ├── HeaderToolBar.ets            title + Search + scan icon, top avoid area
│   ├── MySubscription.ets           the subscribed grid, edit mode + drag sort
│   ├── NewsContent.ets              the per-tab article list (static data)
│   └── TabsWithBarIcon.ets          scrolling tab strip + the `+` that opens the sheet
├── entryability/EntryAbility.ets    full screen, avoid areas -> AppStorage
├── model
│   ├── ChannelData.ets              ChannelData, DEFAULT_CHANNEL (4), CHANNEL_DATA (13)
│   └── NewsData.ets                 static article payload
├── pages
│   ├── ChannelEditSheet.ets         the sheet body: the two grids + shared state
│   └── HomePage.ets                 @Entry, bottom Tabs, owns mySubscriptionList
└── utils/ArrayUtil.ets              moveElement / deleteElement / addElement
```

The documented 工程目录 matches the zip exactly, file for file.

**The design decision worth copying** is where the three pieces of shared state
live. `HomePage` owns `mySubscriptionList` as `@State`; it passes it down to
the tab strip as a `@Prop` (read-only, it only renders tabs) and into the sheet
as a `@Link` (read-write, it is the editor). `ChannelEditSheet` then adds one
piece of state of its own - `selectedChannelIdSet` - and hands both grids
`@Link`s to it. So there is exactly one array and exactly one membership set,
and the "add" grid filters itself with `!this.selectedChannelIdSet.has(item.id)`
rather than diffing two arrays on every render.

The trigger is deliberately not co-located with the sheet: the `+` is part of
the tab strip, so `TabsWithBarIcon` declares `@Link canShowEditSheet` and
`HomePage` binds the sheet to the same boolean under its own name
(`canShowSheet`). The child announces intent; the parent, which owns the data
the sheet edits, owns the presentation.

**One structural choice worth avoiding**: `mySubscriptionList` is initialised
to `DEFAULT_CHANNEL`, a module-level `const` array, and every edit
(`push`, `splice`) mutates that array in place. The `const` protects the
binding, not the contents. There is therefore no "cancel" and no way to restore
defaults for the life of the process. Copy the array (`[...DEFAULT_CHANNEL]`)
when seeding editable state.

## Implementation steps

1. **Model a channel as `{ id, name }`** and keep two lists: the full catalogue
   and the user's subscriptions. Seed the subscription list from a *copy* of
   the defaults.
2. **Own the subscription array in the page**, pass it to the tab strip as
   `@Prop` and into the sheet as `@Link`.
3. **Bind the sheet on the child, from the parent**:
   `TabsWithBarIcon({...}).bindSheet($$this.canShowSheet, this.channelEditSheetBuilder(), {})`.
   `$$` is required - it makes the binding two-way so a swipe-down dismissal
   writes `false` back.
4. **Raise the sheet from the `+` icon** by setting the `@Link`
   `canShowEditSheet`, not a locally invented name (`HW-11-0006`).
5. **Derive membership from a `Set<number>` of ids**, populated once in the
   sheet's `aboutToAppear`, and mutated alongside the array on every add and
   remove.
6. **Give the subscribed `Grid` an explicit height** recomputed by
   `@Watch` on the array, because a `Grid` inside a scrolling `Column` has no
   intrinsic height.
7. **Turn on `editMode(this.isEditMode)` and `supportAnimation(true)`** on the
   subscribed grid only. The candidate grid is never draggable.
8. **Return the drag proxy from `onItemDragStart`** - a `@Builder` that renders
   the same chip shape, so the item under the finger looks like the item that
   left the grid.
9. **Reorder in `onItemDrop` only when `isSuccess`**, with a splice-out /
   splice-in helper. A drop outside the grid reports `isSuccess === false` and
   must be ignored.
10. **Refuse the last removal** with a toast rather than allowing an empty tab
    strip.

## Verified snippets

All snippets are from `ChannelSelection.zip`.

**Sheet wiring — `entry/src/main/ets/pages/HomePage.ets`** (as shipped; the
document's version of this is wrong, see `HW-11-0006`)

```typescript
@Entry
@Component
struct HomePage {
  @StorageProp('bottomRectHeight') bottomRectHeight: number = 0;
  @State canShowSheet: boolean = false;
  @State mySubscriptionList: ChannelData[] = DEFAULT_CHANNEL;

  @Builder
  channelEditSheetBuilder() {
    ChannelEditSheet({ mySubscriptionList: this.mySubscriptionList });
  }

  build() {
    Column() {
      HeaderToolBar();
      Tabs() {
        TabContent() {
          TabsWithBarIcon({ tabArray: this.mySubscriptionList, canShowEditSheet: this.canShowSheet })
            .bindSheet($$this.canShowSheet, this.channelEditSheetBuilder(), {});
        }
        .tabBar(this.tabBuilder($r('app.string.tab_home'), 0, $r('app.media.ic_public_home')));
        // ... three more TabContent()s, all placeholders
      }
      .scrollable(false)
      .barPosition(BarPosition.End)
      .onContentWillChange((currentIndex: number, comingIndex: number) => {
        if (comingIndex > 0) {
          this.getUIContext().getPromptAction().showToast({ message: $r('app.string.toast_demo') });
          return false;                       // deliberate: only the home tab exists
        }
        return true;
      });
    }
  }
}
```

**Three bindings carry the design.** `tabArray` is a `@Prop` into a component
that only renders, while the same array goes into the sheet builder as a
`@Link`, so one mutation updates both surfaces. `$$this.canShowSheet` is the
two-way form - with a plain `this.canShowSheet` the sheet would open once and
never reset the flag when the user swipes it away. And the sheet is attached
to `TabsWithBarIcon`, not to the `+` inside it: `bindSheet` needs to hang off a
node the *page* renders, while the button that sets the flag lives two levels
down.

The `+` itself is four lines in the tab strip, and it writes the `@Link`:

```typescript
// entry/src/main/ets/components/TabsWithBarIcon.ets
@Link canShowEditSheet: boolean;

Image($r('sys.media.ohos_ic_public_add'))
  .size({ width: TAB_BAR_HEIGHT, height: TAB_BAR_HEIGHT })
  .draggable(false)
  .onClick(() => {
    this.canShowEditSheet = true;          // the document writes `this.showEditSheet` here
  });
```

`draggable(false)` matters on every icon in this sample: system media resources
are draggable by default, and a long press on the `+` would otherwise start an
image drag instead of opening the sheet.

**The drag-sortable grid — `entry/src/main/ets/components/MySubscription.ets`** (as shipped)

```typescript
@Link @Watch('updateGridHeight') mySubscriptionList: ChannelData[];
@Link selectedChannelIdSet: Set<number>;
@Link isEditMode: boolean;
@State gridHeight: number = 50;

updateGridHeight() {
  const size = this.mySubscriptionList.length;
  this.gridHeight =
    Math.ceil(size / CHANNEL_GRID_COUNT_PER_ROW) * (CHANNEL_GRID_HEIGHT + CHANNEL_GRID_ROW_GAP);
}

@Builder
gridDragBuilder(name: string) {
  Text(name)
    .width(CHANNEL_GRID_WIDTH)
    .height(CHANNEL_GRID_HEIGHT)
    .textAlign(TextAlign.Center)
    .borderRadius(CHANNEL_GRID_HEIGHT / 2)
    .backgroundColor(THEME_BACKGROUND_COLOR_GRAY);
}

@Builder
gridBuilder() {
  Grid() {
    ForEach(this.mySubscriptionList, (item: ChannelData, index: number) => {
      GridItem() {
        Row({ space: 4 }) {
          if (this.isEditMode) {
            Image($r('sys.media.ohos_ic_public_close')).width(18).height(18).draggable(false);
          }
          Text(item.name).fontSize(CHANNEL_GRID_FONT_SIZE);
        }
        .onClick(() => {
          if (this.isEditMode) {
            if (this.mySubscriptionList.length > 1) {
              deleteElement(this.mySubscriptionList, index);
              this.selectedChannelIdSet.delete(item.id);
            } else {
              this.getUIContext().getPromptAction()
                .showToast({ message: $r('app.string.toast_remove_channel') });
            }
          }
        });
      };
    }, (item: ChannelData) => String(item.id));
  }
  .height(this.gridHeight)
  .columnsTemplate('1fr 1fr 1fr')
  .editMode(this.isEditMode)
  .supportAnimation(true)
  .onItemDragStart((event: ItemDragInfo, itemIndex: number) => {
    return this.gridDragBuilder(this.mySubscriptionList[itemIndex].name);
  })
  .onItemDrop((event: ItemDragInfo, itemIndex: number, insertIndex: number, isSuccess: boolean) => {
    if (isSuccess) {
      moveElement(this.mySubscriptionList, itemIndex, insertIndex);
    }
  });
}
```

**`editMode` is the switch that makes the grid a drag source at all** - without
it a long press does nothing, and `supportAnimation(true)` is what animates the
untouched chips out of the way; drop either and the feature degrades silently
rather than failing. `onItemDragStart` must *return* a builder: what it returns
is what the user drags, and returning the same chip geometry is why the drag
looks like a lift rather than a substitution.

`isSuccess` is the guard people forget. A release outside the grid still fires
`onItemDrop`, with `insertIndex` pointing at the last valid slot; reordering
unconditionally silently moves a chip the user meant to leave alone.

The height computation exists because `Grid` claims no intrinsic height inside
a `Column`. `@Watch` on the `@Link` recomputes it whenever the sheet adds or
removes a channel - note that the formula adds a trailing `rowsGap`, so the
grid is 12 vp taller than its content.

**The candidate grid filters, it does not diff — `entry/src/main/ets/components/ChannelSelection.ets`** (as shipped)

```typescript
@Prop channelData: ChannelData[];             // the full catalogue, 13 entries
@Link myChannelList: ChannelData[];
@Link selectedChannelIdSet: Set<number>;

Grid() {
  ForEach(this.channelData, (item: ChannelData) => {
    if (!this.selectedChannelIdSet.has(item.id)) {      // the whole "already added" rule
      GridItem() {
        Row({ space: 4 }) {
          if (this.isEditMode) {
            Image($r('sys.media.ohos_ic_public_add')).width(16).height(16).draggable(false);
          }
          Text(item.name);
        }
        .onClick(() => {
          if (this.isEditMode) {
            addElement(this.myChannelList, item);
            this.selectedChannelIdSet.add(item.id);
          }
        });
      };
    }
  }, (item: ChannelData) => JSON.stringify(item.id));
}
```

**The `Set` is the reason both grids stay consistent for free.** Membership is
an id lookup inside the `ForEach` body, so adding a channel makes it vanish
from the candidate grid and appear in the subscription grid in the same frame,
from one `add` call plus one `push`. The alternative - keeping two arrays and
removing from one when adding to the other - is where duplicate-channel bugs
come from.

The set is seeded once, in the sheet:

```typescript
// entry/src/main/ets/pages/ChannelEditSheet.ets
aboutToAppear(): void {
  this.initSelectedChannelData();
}

initSelectedChannelData() {
  this.mySubscriptionList.forEach((item: ChannelData) => {
    this.selectedChannelIdSet.add(item.id);
  });
}
```

**The array move — `entry/src/main/ets/utils/ArrayUtil.ets`** (as shipped)

```typescript
export function moveElement<T>(arr: T[], fromIndex: number, toIndex: number) {
  if (fromIndex === toIndex) {
    return;
  }
  const element = arr[fromIndex];
  arr.splice(fromIndex, 1);
  arr.splice(toIndex, 0, element);
}
```

Remove first, then insert at the *original* target index. That is correct
because `onItemDrop` reports `insertIndex` in the coordinates of the grid the
user sees, which is the post-removal layout the framework has already animated
into place. Adjusting the index for the removal - the reflex fix - moves the
chip one slot short when dragging forwards.

## Permissions & config

**None.** `entry/src/main/module.json5` declares no `requestPermissions`.
`deviceTypes` is `phone`, `tablet`, `2in1`, and the ability is the plain
`entity.system.home` entry point.

`EntryAbility` writes the top and bottom avoid-area heights (in px) into
`AppStorage` as `topRectHeight` / `bottomRectHeight`; `HeaderToolBar` and
`TabsWithBarIcon` read them with `@StorageProp` and convert at the point of use
with `px2vp`, feeding `safeAreaPadding` rather than `padding`.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- All four tab contents render the same static `NewsContent` - the channel a
  tab represents does not change what it shows. There is no per-channel feed.
- The subscription order is not persisted. Closing the app restores the four
  defaults (and, because `DEFAULT_CHANNEL` is mutated in place, the "defaults"
  within one process run are whatever the user last left).
- `TabsWithBarIcon`'s tab-strip `ForEach` types its key generator as
  `(item: string) => JSON.stringify(item)` over a `ChannelData[]`. It works
  because stringifying the object yields a unique key, but the declared type is
  wrong and two channels with identical id and name would collide.
- The indicator line width comes from
  `getMeasureUtils().measureTextSize(...)`, which returns **px**, so the result
  is passed through `px2vp` before use. Skipping that conversion is the usual
  way this indicator ends up three times too wide.
- Removing a channel does not move the tab selection, so deleting the currently
  focused channel leaves `focusIndex` pointing past the end of the strip.

## Pitfalls

- **`HW-11-0006`** (E/low, confirmed): the document's step-1 snippet binds the
  sheet to `$$this.canShowSheet` but opens it with `this.showEditSheet = true`,
  a third name that exists nowhere in the sample. Copied as printed the click
  handler writes an undeclared variable and the sheet never opens. Fix: the
  icon lives in `TabsWithBarIcon` and sets the `@Link` `canShowEditSheet`,
  which `HomePage` binds to `canShowSheet`.

## References

- `huawei_industry_tree/11_news_reading/docs/04_channel_selection.md` - the source document
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/channel_selection-0000002270325497
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `editMode`, `supportAnimation`, `onItemDragStart`, `onItemDrop`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-container-griditem.md` - `GridItem` and its drag behaviour
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-griditem
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet` and `SheetOptions`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-drag-sorting.md` - the drag-sort contract shared with `List`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-drag-sorting
- `documentation/harmonyos-references/02_application-framework/ts-container-tabs.md` - `TabsController`, `onAnimationStart`, `onContentWillChange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-tabs
- `documentation/harmonyos-guides/03_application-framework/arkts-two-way-sync.md` - `@Link` and the `$$` two-way binding used by `bindSheet`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-two-way-sync
- `NEWS-01` - the full news app whose tab bar this editor belongs to
