---
id: SPORT-13
title: Reorder dashboard cards by drag - a sequential GestureGroup over a Grid
industry: 03_sports_health
doc: huawei_industry_tree/03_sports_health/docs/13_custom_exercise_dashboard.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/custom_exercise_dashboard-0000002361577984
sample: huawei_industry_tree/03_sports_health/downloads/CustomExerciseDashboard.zip
kits: ["@kit.ArkUI", "@ohos.curves"]
apis: [Grid, GridItem, columnsTemplate, GestureGroup, GestureMode, LongPressGesture, PanGesture, onActionStart, onActionUpdate, onActionEnd, GestureEvent, "UIContext.animateTo", "curves.interpolatingSpring", scale, translate, zIndex, ForEach, "@Link", "@State", NavPathStack]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-03-0049, HW-03-0050, HW-03-0055]
status: verified-with-fixes
---

## When to use

Load this card for a **user-arrangeable grid of cards** - a dashboard the user
picks and orders, a home-screen widget tray, a favourites shelf, a settings
board.

The mechanism is the one ArkUI provides for exactly this and is not obvious:
**`GestureGroup(GestureMode.Sequence, LongPressGesture(), PanGesture())`**. A
sequential group requires the gestures to fire in order, so the card can only
be dragged after it has been held - which is what separates "reorder" from
"scroll" on a touch surface without any manual disambiguation.

Two supporting decisions make it work: a **deep copy on entry and commit on
confirm**, so a cancelled edit changes nothing; and a **spring animation
around the reorder**, so cards glide into their new places rather than jumping.

## Feature checklist

- A dashboard of selected cards, two per row, plus a tray of unselected ones.
- Cards can be added to and removed from the dashboard.
- Holding a card lifts it: it scales up and rises above its neighbours.
- Dragging then moves it, and passing another card swaps the two.
- Confirming writes the new arrangement back to the home page; leaving
  discards it.

## Architecture

One `entry` module, two pages, one model.

```
entry/src/main/ets
├── common/Constants.ets      SCALE_VALUE, DURATION, FIX_VP_X, FIX_VP_Y
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/ExerciseCard.ets    ExerciseCard, ExerciseData, exerciseCards
└── pages
    ├── MainPage.ets          the dashboard
    └── CustomPage.ets        the editor: two grids, the gestures, the reordering
```

The documented tree matches the zip.

**Edit on a copy, commit on confirm:**

```
MainPage            selectedCard / unselectedCard        (the live arrangement)
   │  @Link
CustomPage.aboutToAppear
   │  items1 = deep copy of selectedCard
   │  items2 = deep copy of unselectedCard
   │
   │  ... all dragging and add/remove happens on items1 / items2 ...
   │
confirm ──> selectedCard = deep copy of items1 ──> pageInfos.pop()
back    ──> nothing written
```

`JSON.parse(JSON.stringify(...))` is a blunt deep copy, and it is the right
shape for this: the editor must not touch the live arrangement until the user
commits, and a shallow copy would share the card objects.

**The drag state is three fields on the page**, not per card: `dragItem` (which
card is moving), `scaleItem` (which is lifted) and `offsetX`/`offsetY` (how
far). Every `GridItem` reads them and applies them only when its own id
matches, which is why one set serves the whole grid.

## Implementation steps

1. **Deep copy into local arrays** in `aboutToAppear`, and write back only on
   confirm.
2. **Wrap the two gestures in `GestureMode.Sequence`** so the pan only starts
   after the long press.
3. **Lift the card on the long press** by animating `scale`, and raise its
   `zIndex` so it passes over its neighbours.
4. **Track the drag with `translate`**, offset by a reference so the card
   follows the finger rather than jumping.
5. **Swap on threshold**, moving by one for a horizontal step and by the
   column count for a vertical one.
6. **Check the column, not just the array bounds, on horizontal moves**
   (`HW-03-0049`).
7. **Animate the swap with a spring** so the displaced card glides.

## Verified snippets

All snippets are from `CustomExerciseDashboard.zip`. Corrected forms are marked.

**The gesture group — `entry/src/main/ets/pages/CustomPage.ets`** (as shipped)

```typescript
GridItem() {
  this.customCard(item, index, true);
}
.scale({
  x: this.scaleItem === item.id ? Constants.SCALE_VALUE : 1,
  y: this.scaleItem === item.id ? Constants.SCALE_VALUE : 1
})
.zIndex(this.dragItem === item.id ? 1 : 0)          // the dragged card passes over the others
.translate(this.dragItem === item.id ? { x: this.offsetX, y: this.offsetY } : { x: 0, y: 0 })
.gesture(
  GestureGroup(GestureMode.Sequence,
    LongPressGesture({ repeat: true })
      .onAction(() => {
        this.getUIContext()?.animateTo({ curve: Curve.Friction, duration: Constants.DURATION }, () => {
          this.scaleItem = item.id;                 // lift
        });
      })
      .onActionEnd(() => {
        this.getUIContext()?.animateTo({ curve: Curve.Friction, duration: Constants.DURATION }, () => {
          this.scaleItem = -1;                      // drop
        });
      }),
    PanGesture({ fingers: 1, direction: null, distance: 0 })
      .onActionStart(() => {
        this.dragItem = item.id;
        this.dragRefOffsetX = 0;
        this.dragRefOffsetY = 0;
      })
      .onActionUpdate((event: GestureEvent) => {
        this.offsetY = event.offsetY - this.dragRefOffsetY;
        this.offsetX = event.offsetX - this.dragRefOffsetX;
        this.getUIContext()?.animateTo({ curve: curves.interpolatingSpring(0, 1, 400, 38) }, () => {
          // ... decide the direction and call up / down / left / right
        });
      })
  )
)
```

**`GestureMode.Sequence` is the whole disambiguation.** In a sequential group
the pan is not recognised until the long press has fired, so an ordinary swipe
scrolls the grid and only a held card can be dragged. No thresholds, no manual
gesture arbitration.

**`PanGesture({ direction: null, distance: 0 })`** is deliberately
unconstrained: `null` accepts any direction and `0` starts tracking
immediately, because the long press has already established intent. That is
the opposite of `SPORT-03`'s pan, which uses a 100 vp threshold precisely
because nothing precedes it.

**`dragRefOffsetX/Y` is what makes the swap feel continuous.** `event.offsetX`
is measured from where the drag began, so after a swap the card would jump by
a whole cell; subtracting the accumulated reference cancels exactly that, and
each helper adjusts the reference by the same step it moved the card.

**The reorder helpers — same file** (corrected, see `HW-03-0049`)

```typescript
itemMove(index: number, newIndex: number): void {
  const tmp = this.items1.splice(index, 1);
  this.items1.splice(newIndex, 0, tmp[0]);
}

private static readonly COLS = 2;      // FIX: the sample hardcodes 2 and 1 in four places

down(index: number): void {
  if (index + CustomPage.COLS >= this.items1.length) {
    return;
  }
  this.offsetY -= this.FIX_VP_Y;
  this.dragRefOffsetY += this.FIX_VP_Y;
  this.itemMove(index, index + CustomPage.COLS);      // vertical: step by a whole row
}

left(index: number): void {
  if (index % CustomPage.COLS === 0) {
    return;                            // FIX: the sample checks only index - 1 < 0
  }
  this.offsetX += this.FIX_VP_X;
  this.dragRefOffsetX -= this.FIX_VP_X;
  this.itemMove(index, index - 1);
}

right(index: number): void {
  if (index % CustomPage.COLS === CustomPage.COLS - 1 || index + 1 >= this.items1.length) {
    return;                            // FIX: the sample checks only the array bound
  }
  this.offsetX -= this.FIX_VP_X;
  this.dragRefOffsetX += this.FIX_VP_X;
  this.itemMove(index, index + 1);
}
```

**Vertical moves step by the column count, horizontal by one** - and the
horizontal pair additionally has to respect the row boundary, which the
shipped code omits. The offset and the reference always move in opposite
directions: the card's visual position is pulled back by one cell while its
index moves forward by one, so it stays under the finger.

Four diagonal helpers (`lowerRight`, `lowerLeft`, `upperRight`, `upperLeft`)
combine both adjustments and step by `COLS ± 1`.

**Spring animation around the swap** - `curves.interpolatingSpring(0, 1, 400, 38)` -
is what makes a displaced card glide into the vacated cell instead of
teleporting. The four parameters are velocity, mass, stiffness and damping;
this configuration is stiff and well damped, so the motion settles quickly
without overshoot.

**Copy in, commit out — same file** (as shipped)

```typescript
aboutToAppear(): void {
  this.items1 = JSON.parse(JSON.stringify(this.selectedCard));
  this.items2 = JSON.parse(JSON.stringify(this.unselectedCard));
}

Image($r('app.media.confirm'))
  .onClick(() => {
    this.selectedCard = JSON.parse(JSON.stringify(this.items1));
    this.unselectedCard = JSON.parse(JSON.stringify(this.items2));
    this.pageInfos.pop();
  })
```

Copying on the way out as well as in is the belt-and-braces version: the page
being popped cannot then mutate what it handed back.

## Permissions & config

**None.** The sample declares no `requestPermissions`.

`Constants.ets` holds `SCALE_VALUE` (the lift factor), `DURATION` (the lift
animation) and `FIX_VP_X` / `FIX_VP_Y` - the cell pitch, which must match the
grid's actual cell size for the offset compensation to cancel exactly.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **The grid is two columns, and that is encoded in several places** - the
  `columnsTemplate`, the `± 2` in the vertical helpers, the `± 3` in the
  diagonals, and `FIX_VP_X` / `FIX_VP_Y`. Changing the column count means
  changing all four consistently.
- **`FIX_VP_X` and `FIX_VP_Y` are fixed vp values**, so the compensation is
  correct only at the cell size those constants were measured against - the
  grid does not adapt to screen width.
- Only the selected grid is reorderable; `itemMove` operates on `items1` alone.
- The dashboard's card list is static demo data in `ExerciseCard.ets`, and
  nothing is persisted - the arrangement lives in the pages' state.

## Pitfalls

- **`HW-03-0049` — `left` and `right` check only the array bounds, not the
  column.** In a two-column grid, dragging the left cell of a row leftwards
  swaps it with the right cell of the row above, so the card jumps up and
  across while the offset compensation pushes it the wrong way. The vertical
  helpers encode the column count; the horizontal ones do not.
- **`HW-03-0050` — `name` and `description` are typed `string` and filled with
  Chinese literals,** while the `image` on the same card is a `ResourceStr`.
  The card titles are the one part of the model that cannot be translated.

## References

- `documentation/harmonyos-references/02_application-framework/ts-combined-gestures.md` - `GestureGroup` and `GestureMode.Sequence`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-combined-gestures
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-longpressgesture.md` - `LongPressGesture` and `repeat`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-longpressgesture
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `fingers`, `direction`, `distance`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md`, `ts-container-griditem.md` - the grid and its cells
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-transformation.md` - `scale` and `translate`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-transformation
- `documentation/harmonyos-references/02_application-framework/ts-explicit-animation.md` - `animateTo`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-explicit-animation
- `documentation/harmonyos-references/02_application-framework/js-apis-curve.md` - `interpolatingSpring`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-curve
- `SPORT-03` - the industry's other `PanGesture`, unsequenced and therefore threshold-based
