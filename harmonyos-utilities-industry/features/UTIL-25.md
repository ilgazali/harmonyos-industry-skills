---
id: UTIL-25
title: Drag-to-rearrange note board - a Grid whose rowsTemplate is rewritten by the gesture
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/25_drag_and_change.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/drag_and_change-0000002364921921
sample: huawei_industry_tree/15_utilities/downloads/DragAndChange.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [Grid, GridItem, rowsTemplate, columnsTemplate, columnEnd, GestureGroup, GestureMode, LongPressGesture, PanGesture, "UIContext.animateTo", "curves.interpolatingSpring", "@StorageProp", "UIContext.px2vp", position, offset, hitTestBehavior]
min_api: 20
modules: [entry]
findings: [HW-15-0059, HW-15-0101]
status: verified-with-fixes
---

## When to use

Load this card when the user must be able to **rearrange and resize the tiles
of a board by dragging them**, and the tiles are not a uniform list - a notes
app, a widget wall, a dashboard, a home screen. The pattern here is not
drag-and-drop of list items; it is a `Grid` whose `rowsTemplate` and
`columnsTemplate` strings are recomputed on every gesture frame, so the layout
itself is the animated quantity.

Two gestures live on two different corner handles of the same tile: the
bottom-left handle drags the tile into another slot, the bottom-right handle
resizes the row or column boundary that the tile sits on. Both are a
`GestureGroup(GestureMode.Sequence, LongPressGesture, PanGesture)` - long-press
to arm, pan to act - which is the standard way to keep an in-place drag from
stealing scroll and text-selection events from the content underneath.

It generalises to any "the user owns the layout" screen. The transferable idea
is that a layout expressed as a **string template** (`'200fr 150fr 250fr'`) can
be regenerated from state as cheaply as any other property, so the drag handler
only ever has to maintain a handful of numbers, never a tree of measured
rectangles.

## Feature checklist

- The board opens with three note cards stacked in one column, each an editable
  title `TextArea` plus a body `TextArea`.
- Long-pressing the drag handle at a card's bottom-left lifts a follow-the-finger
  copy of that card; the original dims to 40% opacity.
- Dragging the copy into the top band, middle band or bottom band of the board
  moves the card to that position in the order.
- Dragging past the left or right edge of the board switches the board between
  one-column, two-cards-on-top and two-cards-on-bottom layouts.
- Releasing the finger drops the card and the layout settles under an
  interpolating spring.
- Long-pressing the resize handle at a card's bottom-right and panning changes
  the shared row height (and, in the two-across layouts, the shared column
  width) live.
- Every card stays at or above 120vp in each dimension and the board stays at or
  below 690vp tall; over-dragging is corrected on release rather than blocked
  during the drag.

## Architecture

One `entry` module, two pages, two files of constants and one model class.

```
entry/src/main/ets
├── common/Constants.ets       geometry (348vp board, 120vp minimum, 690vp cap) + the three seed note texts
├── common/PositionMode.ets    the three layout modes: COLUMNS / TOP2 / BOTTOM2
├── entryability/EntryAbility.ets
├── entrybackupability/EntryBackupAbility.ets
├── model/ItemModel.ets        @Observed: id, width, height, columnEnd, title, mainText
└── pages
    ├── MainPage.ets           @Entry - a 60vp 记事本 (notepad) title band above the board
    └── GridsPage.ets          the board: 492 lines, the whole feature
```

The documented tree matches the zip exactly.

**The design decision worth copying** is the paired `xxx` / `xxxTmp` state.
Every dimension exists twice: `columnsFirstHeight` is the value committed at the
last gesture end, `columnsFirstHeightTmp` is the value being previewed right
now. `handleChange()` only ever writes the `Tmp` copies, always as
`committed + offsetY` rather than `current + delta`, and `reset()` copies `Tmp`
back into the committed set when the gesture ends. That is what makes the pan
idempotent: `PanGesture.onActionUpdate` reports the offset from the *gesture
start*, not since the last frame, so a handler written against `current` would
compound the same movement dozens of times per second. Copy the pair, not just
the handler.

The three layout modes are the second load-bearing choice. Rather than let a
card sit anywhere, the board has exactly three legal shapes, and `columnEnd` per
item plus a regenerated `rowsTemplate` describes all of them. `setRowsColumns()`
is one function with three branches and it is the only place that writes layout.

## Implementation steps

1. **Give each card two handles**, not one. A bottom-left `Image` for the move
   gesture and a bottom-right `Image` for the resize gesture, each positioned
   with `.offset({ x: ±item.width / 2 ∓ 10, y: item.height / 2 - 10 })` so they
   ride the card's corners as the card changes size.
2. **Set `.draggable(false)` on both handle images** - otherwise ArkUI's own
   image drag-and-drop fires and the custom gestures never see the pan.
3. **Wrap each pair in `GestureGroup(GestureMode.Sequence, …)`** with the
   `LongPressGesture` first. Sequence mode means the pan is only recognised
   after the long press has been accepted, so a plain swipe over the handle
   still scrolls the page.
4. **Record the grab point in the long press**, from
   `event.target.area.globalPosition`, offset by half the card's size so the
   ghost is centred under the finger.
5. **Convert the global grab point to board-local coordinates by measuring the
   board, not by subtracting the status bar** (`HW-15-0059`).
6. **Update `offsetX` / `offsetY` from the pan and re-derive everything else**
   inside `animateTo` on `curves.interpolatingSpring(0, 1, 400, 38)`.
7. **Classify the drop by band**: `fullHeightTmp / 3` and `fullHeightTmp / 3 * 2`
   split the board into three vertical bands; `LEFT_BORDER` (0) and
   `RIGHT_BORDER` (100) split each band into left / centre / right.
8. **Clamp on release, not during the drag.** `reset()` calls
   `handleOverBorder()` first, which pushes any dimension that went below
   `MIN_WIDTH_HEIGHT` back inside the legal range and re-runs `setRowsColumns()`.
9. **Mark the ghost `hitTestBehavior(HitTestMode.None)`** so it cannot swallow
   the touch stream that is driving it.

## Verified snippets

All snippets are from `DragAndChange.zip`. Corrected forms are marked.

**The move handle — `entry/src/main/ets/pages/GridsPage.ets`** (as shipped)

```typescript
Column() {
  Image($r('app.media.drag_button'))
    .width(20)
    .height(20)
    .draggable(false);
}
.width(20)
.height(20)
.offset({ x: -item.width / 2 + 10, y: item.height / 2 - 10 })
.gesture(
  GestureGroup(GestureMode.Sequence,
    LongPressGesture({ repeat: true })
      .onAction((event?: GestureEvent) => {
        if (this.selectedItem === undefined) {
          this.selectedItem = item;
          this.dragStartX = Number(event?.target.area.globalPosition.x) - item.width / 2;
          this.dragStartY = Number(event?.target.area.globalPosition.y) - item.height / 2;
        }
      })
      .onActionEnd(() => {
        this.reset();
      }),
    PanGesture({ fingers: 1 })
      .onActionStart(() => {
        this.selectedItem = item;
      })
      .onActionUpdate(event => {
        this.offsetY = event.offsetY;
        this.offsetX = event.offsetX;
        this.getUIContext().animateTo({ curve: curves.interpolatingSpring(0, 1, 400, 38) }, () => {
          this.setRowsColumns();
          this.handleDrag();
        });
      })
      .onActionEnd(() => {
        this.reset();
      })
  ).onCancel(() => {
    this.reset();
  })
);
```

**Three details carry this gesture.** `GestureMode.Sequence` is what makes the
handle safe to place on top of two editable `TextArea`s - the pan cannot start
until the long press has completed, so ordinary taps and text selection still
reach the content. `repeat: true` on the long press keeps `onAction` firing
while the finger is held, which matters because the grab point is only captured
there; the `if (this.selectedItem === undefined)` guard is what stops the repeat
from re-capturing a stale origin mid-drag. And `onCancel` on the group, not just
`onActionEnd` on each child, is the only path that runs when the system takes
the gesture away (an incoming call, a back swipe) - without it the board would
be left holding a `selectedItem` and a dimmed card forever.

Note that `event.offsetX/offsetY` are cumulative from the gesture start. That is
why every consumer of them reads `committed + offset`, never `current + offset`.

**The follow-the-finger ghost — same file** (corrected, see `HW-15-0059`)

```typescript
// FIX: measure the board instead of assuming the header is zero
@State gridOriginY: number = 0;

@Builder
dragBuilder() {
  Column() {
    if (this.selectedItem) {
      Stack() {
        Column({ space: 10 }) {
          TextArea({ placeholder: '请输入标题', text: this.selectedItem.title })
            .height(40)
            .align(Alignment.Start);
          TextArea({ text: this.selectedItem.mainText })
            .height(this.selectedItem.height - 80);
        };
      }
      .width(this.selectedItem.width)
      .height(this.selectedItem.height)
      .backgroundColor('rgb(255, 255, 255)')
      .padding(15)
      .borderRadius(20);
    }
  }
  .hitTestBehavior(HitTestMode.None)
  .aspectRatio(1)
  .zIndex(1)
  .position({
    x: this.dragStartX + this.offsetX,
    y: this.dragStartY + this.offsetY - this.gridOriginY   // FIX: was px2vp(this.topRectHeight)
  });
}
```

`dragStartX/Y` come from `globalPosition`, i.e. window coordinates, while
`.position()` on a child of the board is measured from the board's own top-left.
The shipped code bridges the two by subtracting only
`px2vp(this.topRectHeight)`, the status-bar height - but `MainPage` also puts a
60vp 记事本 title band above the board. Everything the drag computes is
therefore a band too low: the ghost renders below the finger, and the same
expression drives the band thresholds in `handleDrag()`, so drops bias toward
the lower zone. Capture the board's own global origin instead (an
`onAreaChange` on the `Grid` writing `newValue.globalPosition.y` into
`gridOriginY`) and both the ghost and the classification become independent of
whatever chrome the host page adds above.

**Layout as a regenerated string — same file** (as shipped)

```typescript
setRowsColumns() {
  if (this.positionMode === PositionMode.COLUMNS) {
    this.gridItems[0].columnEnd = 0;
    this.gridItems[2].columnEnd = 0;
    let newGridItems = this.gridItems.map((item, index) => {
      if (index === 0) {
        return new ItemModel(item.id, this.columnsFirstHeightTmp, Constants.FULL_WIDTH, item.columnEnd,
          item.title, item.mainText);
      } else if (index === 1) {
        return new ItemModel(item.id, this.columnsSecondHeightTmp, Constants.FULL_WIDTH, item.columnEnd,
          item.title, item.mainText);
      } else {
        return new ItemModel(item.id, this.fullHeightTmp - this.columnsFirstHeightTmp - this.columnsSecondHeightTmp,
          Constants.FULL_WIDTH, item.columnEnd, item.title, item.mainText);
      }
    });
    this.gridItems = newGridItems;
    this.rowsT = this.columnsFirstHeightTmp + 'fr ' + this.columnsSecondHeightTmp + 'fr ' +
      (this.fullHeightTmp - this.columnsFirstHeightTmp - this.columnsSecondHeightTmp) + 'fr';
    this.columnsT = '1fr';
  }
  // TOP2 and BOTTOM2 branches follow the same shape
}
```

**`fr` used as an absolute unit is the trick worth understanding.** `Grid`
normalises the fractions, so `'200fr 150fr 250fr'` over a 600vp board resolves
to exactly 200 / 150 / 250 vp as long as the three numbers sum to the board
height - which is precisely the invariant `handleOverBorder()` maintains. The
sample gets pixel-accurate rows out of a proportional API by keeping the total
honest, and never has to convert units.

The third row's height is always computed as the remainder
(`fullHeightTmp - first - second`) rather than stored. One less number to keep
consistent, and rounding drift cannot open a gap at the bottom of the board.

**Band classification on drop — same file** (as shipped)

```typescript
handleDrag() {
  let index = -1;
  this.gridItems.forEach((item: ItemModel, i: number) => {
    if (item.id === this?.selectedItem?.id) {
      index = i;
    }
  });
  this.selectedItem = this.gridItems[index];
  if (this.dragStartY + this.offsetY - this.getUIContext().px2vp(this.topRectHeight) <
    this.fullHeightTmp / Constants.ITEM_NUMS &&
    this.dragStartX + this.offsetX >= Constants.LEFT_BORDER &&
    this.dragStartX + this.offsetX <= Constants.RIGHT_BORDER) {
    let tmp = this.gridItems.splice(index, 1);
    this.gridItems.splice(0, 0, tmp[0]);
    this.positionMode = PositionMode.COLUMNS;
    this.setRowsColumns();
  }
  // five more bands: top-left / top-right -> TOP2, middle -> COLUMNS,
  // bottom-centre -> COLUMNS, bottom-left / bottom-right -> BOTTOM2
}
```

The reorder is `splice` out then `splice` in at the target index - the array
order *is* the visual order, because `ForEach` walks it and `Grid` fills
row-major. Note that `this.selectedItem` is re-read from the array after the
lookup: the item objects are replaced wholesale by `setRowsColumns()` on every
frame, so holding the old reference would leave the ghost rendering stale
dimensions.

The same `dragStartY + offsetY - px2vp(topRectHeight)` expression appears in all
six branches; fixing `HW-15-0059` means replacing it in one derived value, which
is a good reason to hoist it into a single `localY` local first.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`deviceTypes` is `phone`, `tablet`, `2in1` and `wearable` - which is optimistic
for a board whose width is the hardcoded constant `FULL_WIDTH = 348`. See
Constraints.

`MainPage` reads `topRectHeight` / `bottomRectHeight` from `AppStorage` via
`@StorageProp` and turns them into padding with `px2vp`, the same avoid-area
idiom used across this industry's samples.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The board is a fixed 348vp wide and its container a fixed 800vp tall.**
  `Constants.FULL_WIDTH`, `MAX_HEIGHT` and `RIGHT_BORDER` are all absolute vp
  literals, so on a tablet the board is a narrow strip in the middle of the
  screen and the left/right drop thresholds no longer align with its edges.
  Derive them from a measured width before shipping this on anything but a
  phone.
- **`ForEach` is keyed on `JSON.stringify(item)`.** Since `setRowsColumns()`
  rebuilds every `ItemModel` on every pan frame with a new height, every key
  changes on every frame, so the whole grid is torn down and re-created dozens
  of times per second during a drag. It works at three items and will not at
  thirty. Key on `item.id` and mutate the `@Observed` model instead of replacing
  it.
- Card contents are seeded from three static Chinese strings in `Constants` and
  edited in place; there is no persistence, so every layout and every edit is
  lost on exit.
- `handleOverBorder()` corrects violations *after* the fact with `+1` / `-10`
  nudges. The result is legal but not stable: repeated over-drags at the bottom
  cap will walk the two upper rows down 10vp at a time.
- The resize handle only ever adjusts a *shared* boundary, so there is no way to
  make one card taller without making its neighbour shorter - by design, since
  the rows must keep summing to `fullHeightTmp`.

## Pitfalls

- **`HW-15-0059` (B/low, probable) — the drag ghost and the drop bands ignore
  the 60vp page header.** The global-to-local conversion subtracts only the
  status-bar height, so the follow-the-finger card renders about a band below
  the finger and band thresholds bias drops toward the lower zone. Fix: measure
  the grid's own global origin (`onAreaChange` on the `Grid`) and subtract that
  instead of `px2vp(topRectHeight)`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-combined-gestures.md` - `GestureGroup`, `GestureMode.Sequence`, the recognition order
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-combined-gestures
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - `rowsTemplate`, `columnsTemplate` and the `fr` unit
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo` around the layout recomputation
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-location.md` - `position`, `offset` and their coordinate spaces
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-location
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `px2vp` and `animateTo` off the UIContext
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
