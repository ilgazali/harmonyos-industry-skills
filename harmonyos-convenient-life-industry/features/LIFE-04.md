---
id: LIFE-04
title: Cinema seat picker - a 14x10 seat grid painted on one Canvas, hit-tested by arithmetic
industry: 02_convenient_life
doc: huawei_industry_tree/02_convenient_life/docs/04_canvas_cinema.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_cinema-0000002272398929
sample: huawei_industry_tree/02_convenient_life/downloads/CanvasCinema.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, ImageBitmap, "context.fillStyle", "context.fillRect", "context.strokeStyle", "context.strokeRect", "context.lineWidth", "context.drawImage", "ImageBitmap.close", onReady, ClickEvent, "ClickEvent.x", "ClickEvent.y", "@CustomDialog", CustomDialogController, onWillDismiss, DismissReason, "@State", "@Prop", "@StorageProp", LengthMetrics, "util.format", "window.getWindowAvoidArea", "window.on('avoidAreaChange')", "ConfigurationConstant.ColorMode"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-02-0020, HW-02-0021, HW-02-0022, HW-02-0023, HW-02-0024, HW-02-0025, HW-02-0026, HW-02-0269]
status: verified-with-fixes
---

## When to use

Load this card when you need **many small, uniform, individually selectable
cells** and a component tree would be the wrong tool. A cinema auditorium is
140 seats; a `Grid` of 140 `GridItem`s means 140 component nodes, 140 layout
passes and 140 click handlers. One `Canvas` means one node, one `onClick`, and
a repaint of a single 20x20 rectangle per tap.

The trade is that you give up the layout engine: there is no component under
the finger, so the tap has to be converted to a cell index by arithmetic, and
"redraw one cell" has to be written by hand.

Take this card for seat maps, parking-space maps, warehouse bin grids, calendar
heatmaps - anything where the cells are a regular lattice and the count is in
the hundreds. Do not take it when cells need their own animations, focus, or
accessibility labels; those need real components.

## Feature checklist

- A 14-column by 10-row seat grid drawn on a single `Canvas`.
- Four seat states with a legend above the grid: 可售 (available, white with a
  blue outline), 已售 (sold, red), 已选座位 (selected, green with a check mark),
  不可售 (not selectable, grey).
- Tapping an available seat selects it; tapping an already selected or already
  confirmed seat shows a toast instead.
- A row-number strip floats over the left edge of the canvas.
- Selected seats appear below as removable chips reading "N排M座" (row N, seat M).
- A running "已选 N 个" (N selected) counter and a 完成 (complete) button that
  refuses an empty selection.
- Confirming opens a payment dialog listing the seats sorted by row then column,
  then a success dialog; confirmed seats turn red.

## Architecture

One `entry` module, four components, no model layer - the entire state is a
14x10 array of small integers.

```
entry/src/main/ets
├── entryability/EntryAbility.ets     light mode, full screen, avoid areas -> AppStorage
├── constants
│   ├── StyleConstants.ets            geometry, colours, and the derived canvas size
│   └── CanvasData.ets                ItemModel + the legend list + the row labels
├── components
│   ├── MovieInfo.ets                 Title: back arrow, legend row, screen graphic
│   ├── CinemaInfo.ets                THE CARD: canvas, hit-test, selection list, footer
│   └── CustomDialogExample.ets       confirm dialog + nested success dialog
└── pages/CanvasPage.ets              @Entry - Title() over CinemaInfo()
```

The documented tree matches the zip exactly.

**The geometry is derived, not typed twice.** `StyleConstants` computes the
canvas size from the cell size and gap:

```
RECT_SIZE   = 20            one seat
RECT_SPACE  = 5             gap
RECT_INDEX  = 25            pitch  = RECT_SIZE + RECT_SPACE
OFFSET_X/Y  = 2             inset so the 0.5 vp outline is not clipped
CANVAS_WIDTH  = 14 * 20 + 13 * 5 + 4 = 349
CANVAS_HEIGHT = 10 * 20 +  9 * 5 + 4 = 249
```

Seat `(col, row)` is drawn at `(RECT_INDEX * col + OFFSET_X, RECT_INDEX * row +
OFFSET_Y)`, and the inverse - the hit-test - is
`floor((local - OFFSET) / RECT_INDEX)`. Those two expressions are the whole
component. The `+ 4` on each dimension is `2 * OFFSET`, which is why the inset
is not optional (`HW-02-0021`).

**Three arrays hold the state, and only one of them needs to.**

- `seats: number[][]` - 0 available, 1 selected, 2 confirmed. This is the truth.
- `selectedSeats: number[][]` - the `[col, row]` pairs currently selected.
  Needed because the chips list and the dialog iterate it.
- `count: number` - a hand-maintained duplicate of `selectedSeats.length`
  (`HW-02-0025`).

Nothing renders from `seats`, so its `@State` decoration buys nothing: writes to
`seats[i][j]` are second-level and would not be observed anyway. The canvas is
repainted imperatively.

Flow of one selection:

```
Canvas.onClick(e)
  -> handleClick: screen coords -> (col, row)          <- HW-02-0020 lives here
  -> guard: last cell is 不可售, state 1 and 2 toast and return
  -> drawRect(col, row, 1): paints green + check mark
     AND mutates seats/count/selectedSeats             <- HW-02-0025 lives here
  -> @State change repaints the chip list and the counter
```

## Implementation steps

1. **Fix the lattice constants first** - `RECT_SIZE`, `RECT_SPACE`,
   `RECT_INDEX = RECT_SIZE + RECT_SPACE`, `OFFSET_X`, `OFFSET_Y` - and derive
   `CANVAS_WIDTH`/`CANVAS_HEIGHT` from them plus `2 * OFFSET`. Never type the
   canvas size as a literal.
2. **Seed the state array in `aboutToAppear`**: `WIDTH` columns of `HEIGHT`
   zeroes. Index it `[col][row]` and keep that order everywhere - the display
   labels flip it back (`row = item[1] + 1`, `seat = item[0] + 1`).
3. **Paint the whole grid in `Canvas.onReady`.** `onReady` fires when the canvas
   is ready to draw and after every resize, so it is the right place for the
   initial full paint.
4. **Write `drawRect(col, row, state)` as a pure painter.** Fill, then stroke
   for the available state - a white seat on a white canvas is invisible without
   the outline. Keep selection bookkeeping in the caller (`HW-02-0025`).
5. **Hit-test with `e.x` and `e.y`, not `e.displayX`/`e.displayY`.** They are
   already relative to the Canvas, so the only correction is the two-vp inset
   (`HW-02-0020`).
6. **Guard the three non-selectable cases before painting**: out of range, the
   fixed 不可售 cell, and states 1 and 2 (each with its own toast).
7. **Decode the check-mark image once**, as a component field, and `close()` it
   in `aboutToDisappear` - not once per tap (`HW-02-0022`).
8. **Give the confirm dialog `@Link selectedSeats`, not `@Prop`** - the
   `CustomDialogController` reference allows only `@Link` or `@Consume` for data
   a builder must track (`HW-02-0026`) - and sort a `slice()` of it inside
   `aboutToAppear` so the parent's chip order is untouched. Format the message
   with `util.format('%s排%s座', ...)`.
9. **Null the `CustomDialogController` in `aboutToDisappear`** - the sample does
   this and it is the documented pattern for avoiding a retain cycle between the
   component and its dialog.
10. **Scope 清空 (clear) to the current selection**, not to the whole grid, so
    confirmed seats survive it (`HW-02-0024`).

## Verified snippets

All snippets are from `CanvasCinema.zip`. Corrected forms are marked.

**Derived geometry - `CanvasCinema.zip#entry/src/main/ets/constants/StyleConstants.ets:53`** (as shipped)

```typescript
static readonly CANVAS_Y: number = 161;              // magic - see HW-02-0020
static readonly RECT_SIZE: number = 20;
static readonly RECT_SPACE: number = 5;
static readonly RECT_INDEX: number = this.RECT_SIZE + this.RECT_SPACE;
static readonly CANVAS_HEIGHT: number = 10 * this.RECT_SIZE + 9 * this.RECT_SPACE + 4;
static readonly CANVAS_WIDTH: number = 14 * this.RECT_SIZE + 13 * this.RECT_SPACE + 4;
static readonly CANVAS_LEFT_MARGIN: number = 15;
static readonly OFFSET_X: number = 2;
static readonly OFFSET_Y: number = 2;
```

**`this.` works in a static initializer** and is what lets `RECT_INDEX` and the
two canvas dimensions be derived rather than typed. The `+ 4` is `2 * OFFSET`:
the drawing is inset by two vp on each side so the 0.5 vp outline of the first
and last seat is inside the canvas.

`CANVAS_Y = 161` is the odd one out - it is the distance from the top of the
page to the top of the canvas, hand-summed from six resource floats
(`margin_ten` 10 + `title_height` 26 + `margin_thirty` 30 + `title2_height` 25 +
`margin_ten` 10 + `title3_height` 50) plus the page's `Column` space of 10. It
exists only to support the screen-coordinate hit-test and disappears with the
`HW-02-0020` fix.

**The painter - `CanvasCinema.zip#entry/src/main/ets/components/CinemaInfo.ets:75`** (corrected, see `HW-02-0022` and `HW-02-0025`)

```typescript
private selectedImg: ImageBitmap = new ImageBitmap('resources/base/media/selected.png');  // FIX: decode once

drawRect(x: number, y: number, id: number) {
  const px = StyleConstants.RECT_INDEX * x + StyleConstants.OFFSET_X;
  const py = StyleConstants.RECT_INDEX * y + StyleConstants.OFFSET_Y;
  if (id === 0) {                                   // 可售 - white fill plus a blue outline
    this.context.fillStyle = StyleConstants.DEFAULT_COLOR;
    this.context.fillRect(px, py, StyleConstants.RECT_SIZE, StyleConstants.RECT_SIZE);
    this.context.strokeStyle = StyleConstants.STROKE_COLOR;
    this.context.lineWidth = StyleConstants.LINE_WIDTH;
    this.context.strokeRect(px, py, StyleConstants.RECT_SIZE, StyleConstants.RECT_SIZE);
  } else if (id === 1) {                            // 已选座位 - green plus a check mark
    this.context.fillStyle = StyleConstants.SELECT_COLOR;
    this.context.fillRect(px, py, StyleConstants.RECT_SIZE, StyleConstants.RECT_SIZE);
    this.context.drawImage(this.selectedImg,        // FIX: the sample builds a new ImageBitmap here
      px + StyleConstants.DRAW_IMAGE_OFFSET_X, py + StyleConstants.DRAW_IMAGE_OFFSET_Y,
      StyleConstants.DRAW_IMAGE_WIDTH, StyleConstants.DRAW_IMAGE_HEIGHT);
  } else {                                          // 已售 - solid red
    this.context.fillStyle = StyleConstants.SELECTED_COLOR;
    this.context.fillRect(px, py, StyleConstants.RECT_SIZE, StyleConstants.RECT_SIZE);
  }
}
```

**The outline is load-bearing.** `DEFAULT_COLOR` is `#ffffffff` and the canvas
background is white; without `strokeRect` an available seat is literally
invisible. The document's snippet omits it (`HW-02-0021`).

The shipped `id === 1` branch also runs `this.seats[x][y] = 1; this.count++;
this.selectedSeats.push([x, y]);` - drawing and bookkeeping in one function,
which is `HW-02-0025`. Only one of the three branches does it, so every other
call site repeats the bookkeeping by hand.

Note the constant names: `SELECT_COLOR` (`#00965C`, green) is the **selected**
state and `SELECTED_COLOR` (`#E50019`, red) is the **sold** state. They read the
other way round.

**The hit-test - same file, line 111** (corrected, see `HW-02-0020`)

```typescript
handleClick(e: ClickEvent) {
  // FIX: e.x / e.y are already in the Canvas's coordinate system.
  // Shipped code used e.displayX / e.displayY and then subtracted
  // CANVAS_LEFT_MARGIN, CANVAS_Y (161) and px2vp(topRectHeight) by hand.
  const x: number = Math.floor((e.x - StyleConstants.OFFSET_X) / StyleConstants.RECT_INDEX);
  const y: number = Math.floor((e.y - StyleConstants.OFFSET_Y) / StyleConstants.RECT_INDEX);

  if (x === this.WIDTH - 1 && y === this.HEIGHT - 1) {
    return;                                                    // the fixed 不可售 cell
  }
  if (x >= 0 && x < this.WIDTH && y >= 0 && y < this.HEIGHT) {
    if (this.seats[x][y] === 1) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.seat_has_been_selected') });
      return;
    } else if (this.seats[x][y] === 2) {
      this.getUIContext().getPromptAction().showToast({ message: $r('app.string.seat_has_been_confirmed') });
      return;
    } else {
      this.drawRect(x, y, 1);
    }
  }
}
```

**`floor((local - inset) / pitch)` is the whole hit-test,** and it is exactly the
inverse of the drawing expression. That symmetry is the reason to keep `OFFSET`
in both: drop it from one side and every tap near a seat boundary picks the
wrong cell.

The range check after the floor is what handles taps in the gaps and outside the
grid - a negative coordinate floors to -1 and fails `x >= 0`.

**Initial paint - same file, line 150** (as shipped)

```typescript
Canvas(this.context)
  .width(StyleConstants.CANVAS_WIDTH)
  .height(StyleConstants.CANVAS_HEIGHT)
  .onReady(() => {
    for (let i = 0; i < this.WIDTH; i++) {
      for (let j = 0; j < this.HEIGHT; j++) {
        if (i === this.WIDTH - 1 && j === this.HEIGHT - 1) {
          this.context.fillStyle = StyleConstants.NOT_SELECTABLE_COLOR;
          this.context.fillRect(StyleConstants.RECT_INDEX * i + StyleConstants.OFFSET_X,
            StyleConstants.RECT_INDEX * j + StyleConstants.OFFSET_Y,
            StyleConstants.RECT_SIZE, StyleConstants.RECT_SIZE);
        } else {
          this.drawRect(i, j, 0);
        }
      }
    }
  })
  .onClick((e: ClickEvent) => {
    this.handleClick(e);
  })
  .margin({ left: StyleConstants.CANVAS_LEFT_MARGIN, right: StyleConstants.CANVAS_RIGHT_MARGIN });
```

**`onReady` is the only correct place for the first paint.** The rendering
context is not usable before it fires, and it fires again after a resize, so the
grid redraws itself when the window changes. Painting from `aboutToAppear`
instead would draw into a context that is not attached yet.

Note that the paint loop does **not** consult `seats` - it hard-codes the
available state and the one grey cell. That is why a rotation, which re-fires
`onReady`, silently discards the user's selection. Driving the loop from
`this.seats[i][j]` instead would make the repaint idempotent.

**Selection chips and per-seat removal - same file, line 239** (as shipped)

```typescript
ForEach(this.selectedSeats, (item: number[]) => {
  Row() {
    Text((item[1] + 1) + '排' + (item[0] + 1) + '座')     // row = [1], seat = [0]
      .fontSize($r('app.float.font_size_small'))
      .maxLines(StyleConstants.TEXT_MAXLINES);
    Divider().vertical(true).strokeWidth(StyleConstants.DIVIDER_STROKEWIDTH);
    Image($r('app.media.delete'))
      .onClick(() => {
        this.drawRect(item[0], item[1], 0);               // repaint that one cell
        this.seats[item[0]][item[1]] = 0;
        this.count--;
        for (let i = 0; i < this.selectedSeats.length; i++) {
          if (this.selectedSeats[i][0] === item[0] && this.selectedSeats[i][1] === item[1]) {
            this.selectedSeats.splice(i, 1);
            break;
          }
        }
      });
  }
});
```

**Deselecting repaints exactly one 20x20 rectangle.** That is the payoff of the
canvas approach - no layout pass, no diff, no component churn. The linear scan
to find the entry is fine at this size; a `Set` of `col * HEIGHT + row` keys
would scale better.

`splice` on a `@State` array is observed, so the chip disappears; `this.count--`
is the duplicate bookkeeping from `HW-02-0025`.

**The confirm dialog - `CanvasCinema.zip#entry/src/main/ets/components/CustomDialogExample.ets:37`** (as shipped)

```typescript
@Prop selectedSeats: Array<Array<number>> = [];
@State mess: string = '';

aboutToAppear(): void {
  this.selectedSeats.sort((a, b) => {
    if (a[1] === b[1]) {
      return a[0] - b[0];        // same row -> by seat
    } else {
      return a[1] - b[1];        // otherwise by row
    }
  });
  for (let x = 0; x < this.selectedSeats.length; x++) {
    if (this.mess === '') {
      this.mess = util.format('%s排%s座', (this.selectedSeats[x][1] + 1), (this.selectedSeats[x][0] + 1));
    } else {
      this.mess += util.format('、%s排%s座', (this.selectedSeats[x][1] + 1), (this.selectedSeats[x][0] + 1));
    }
  }
}
```

**`@Prop` here contradicts the reference (`HW-02-0026`).** The
`CustomDialogControllerOptions.builder` entry says plainly: "To listen for data
changes in the builder, use the @Link or @Consume decorator; other decorators,
such as @Prop and @ObjectLink, do not apply." `LIFE-05`'s equivalent dialog uses
`@Link`. Switch to `@Link` and sort a copy so the parent's chip order is not
disturbed:

```typescript
@Link selectedSeats: Array<Array<number>>;                       // FIX

aboutToAppear(): void {
  const ordered = this.selectedSeats.slice()
    .sort((a, b) => a[1] === b[1] ? a[0] - b[0] : a[1] - b[1]);  // FIX: sort a copy
  this.mess = ordered
    .map((s: number[]) => util.format('%s排%s座', s[1] + 1, s[0] + 1))
    .join('、');
}
```

**Dialog lifetime - `CanvasCinema.zip#entry/src/main/ets/components/CinemaInfo.ets:34`** (as shipped)

```typescript
dialogController: CustomDialogController | null = new CustomDialogController({
  builder: CustomDialogExample({
    cancel: () => {},
    confirm: () => { this.confirm(); },
    selectedSeats: this.selectedSeats,
  }),
  autoCancel: true,
  onWillDismiss: (dismissDialogAction: DismissDialogAction) => {
    if (dismissDialogAction.reason === DismissReason.PRESS_BACK) {
      dismissDialogAction.dismiss();
    }
    if (dismissDialogAction.reason === DismissReason.TOUCH_OUTSIDE) {
      dismissDialogAction.dismiss();
    }
  },
  gridCount: StyleConstants.GRID_COUNT,
  showInSubWindow: true,
  isModal: true,
  cornerRadius: $r('app.float.corner_radius'),
});

// 在自定义组件即将析构销毁时将dialogController置空
aboutToDisappear() {
  this.dialogController = null;
}
```

**Nulling the controller in `aboutToDisappear` is required, not tidy.** The
controller holds the builder closure, which captures `this`; leaving it alive
keeps the component alive after it leaves the tree.

`onWillDismiss` intercepting both `PRESS_BACK` and `TOUCH_OUTSIDE` and then
calling `dismiss()` is a no-op pass-through in the sample - the hook is wired so
that a real app can veto a dismissal while a payment is in flight.

`showInSubWindow: true` makes the dialog render in its own window, so it is not
clipped by the page.

## Permissions & config

None. `CanvasCinema.zip#entry/src/main/module.json5` declares no
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
    }]
  }
}
```

`main_pages.json` is `{ "src": ["pages/CanvasPage"] }`.

Root `build-profile.json5` targets `6.0.0(20)` and turns on both strict-mode
checks - worth copying:

```json5
"buildOption": {
  "strictMode": {
    "caseSensitiveCheck": true,
    "useNormalizedOHMUrl": true
  }
}
```

`EntryAbility.onCreate` pins the application to light mode:

```typescript
this.context.getApplicationContext()
  .setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT);
```

The seat colours are hard-coded hex literals, so this is a necessity rather than
a preference.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later; DevEco
  Studio 6.0.0 Release or later (document lines 53-55, matching
  `compatibleSdkVersion: "6.0.0(20)"`).
- The auditorium is fixed at 14 x 10 and the canvas is sized in vp, so the grid
  does not scale to the screen; on a narrow device the 349 vp canvas overflows.
- Exactly one seat is 不可售 - the last cell - and it is hard-coded in two
  places (`onReady` and `handleClick`). There is no data-driven notion of a
  broken or blocked seat.
- No seat starts as 已售 (sold): the red state only appears after the local user
  confirms. There is no server state in this sample.
- Light mode is forced application-wide.
- `onReady` repaints the grid from constants, not from `seats`, so a resize or
  rotation visually discards the selection while the chip list still shows it.
- The canvas has no accessibility surface: 140 seats are one node with one
  label. A production seat map needs a parallel accessible representation.

## Pitfalls

- **`HW-02-0020` - the hit-test uses `e.displayX`/`e.displayY`.** Those are
  screen coordinates, so the code subtracts the canvas margin, the two-vp inset,
  a hard-coded `CANVAS_Y = 161` and the status-bar height by hand. `ClickEvent.x`
  and `.y` are already relative to the clicked Canvas. Use them and all four
  corrections disappear.
- **`HW-02-0021` - both document snippets drop `OFFSET_X`/`OFFSET_Y`,** and the
  drawing snippet also drops `strokeRect`. Copy them as printed and available
  seats are invisible white-on-white, two vp off from where the hit-test looks.
- **`HW-02-0022` - a new `ImageBitmap` is decoded on every tap and never
  closed.** The reference's own example declares it once as a field and calls
  `close()` after `drawImage`.
- **`HW-02-0023` - `on('avoidAreaChange')` has no `off()`** in
  `onWindowStageDestroy`.
- **`HW-02-0024` - 清空 (clear) wipes confirmed seats too.** It loops the whole
  grid and writes 0 with no check on the current value, so seats the user has
  already paid for become available again - contradicting the toast the same
  component shows for state 2.
- **`HW-02-0026` - the payment dialog takes the seat list with `@Prop`.** The
  `CustomDialogController` reference allows only `@Link` or `@Consume` for data
  a dialog builder must track; with `@Prop` the confirmation can name no seats
  at all while the payment still goes through, because `confirm()` reads the
  parent's array directly.
- **`HW-02-0025` - `drawRect` paints *and* mutates selection state,** but only
  in its `id === 1` branch. The other three call sites each repeat the
  bookkeeping by hand, which is why `count` and `selectedSeats.length` exist as
  two separate counters.
- **Do not paint from `aboutToAppear`.** The rendering context is not attached
  until `onReady` fires; drawing earlier is silently discarded.
- **Do not drive `onReady` from constants.** It re-fires on resize; paint from
  `seats[i][j]` so the repaint is idempotent and the selection survives a
  rotation.
- **Do not decorate `seats` with `@State` and expect updates.** Second-level
  writes (`seats[i][j] = 1`) are not observed. Nothing renders from it here, so
  the decoration is misleading rather than broken - but do not add a view that
  depends on it.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - `Canvas`, `onReady` semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `fillStyle`, `fillRect`, `strokeStyle`, `strokeRect`, `lineWidth`, `drawImage`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-imagebitmap.md` - the `ImageBitmap` constructor, path resolution, and `close()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-imagebitmap
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-click.md` - `ClickEvent.x`/`.y` (component coordinates) versus `displayX`/`displayY` (screen coordinates)
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-click
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `CustomDialogController`, `onWillDismiss`, `DismissReason`, `showInSubWindow`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on`/`off('avoidAreaChange')`, `getWindowAvoidArea`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `LIFE-01` - the industry shell this page would sit in
