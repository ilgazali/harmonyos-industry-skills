---
id: KIDS-07
title: Children's drawing board - Canvas strokes with colour, thickness and eraser pickers
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/07_draw_board.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_board-0000002353184357
sample: huawei_industry_tree/08_children_education/downloads/DrawBoard.zip
kits: ["@kit.ArkUI"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, Path2D, onTouch, TouchType, stroke, clearRect, lineCap, lineWidth, strokeStyle, bindSheet, Grid, GridItem, Slider, Circle, Rect, "UIContext.getPromptAction", showToast, NavPathStack, NavDestination, "@Provide", "@Consume", "@StorageProp", "UIContext.px2vp"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0053, HW-08-0054, HW-08-0055, HW-08-0056, HW-08-0057, HW-08-0058, HW-08-0059, HW-08-0060]
status: verified-with-fixes
---

## When to use

Load this card when a drawing surface needs **tools around it** - a colour
palette, a brush-thickness control, an eraser, a background or template picker.
`KIDS-03` is the bare stroke loop; this is the same loop with a toolbar, and
the toolbar is the reason to read it.

Three things are worth taking:

- **Half-modal sheets as tool panels.** Each toolbar button owns a
  `bindSheet` with `$$` two-way binding, so the panel state is a boolean per
  tool and the panels never fight over one flag.
- **Confirm-or-cancel inside a sheet.** The colour picker writes to a
  *temporary* index and commits it only on the confirm button, so backing out
  of the sheet leaves the pen unchanged.
- **Converting px to vp at the storage boundary.** The ability writes
  `px2vp(topRectHeight)` into `AppStorage`, so consumers bind a value that is
  already in the unit layout wants. Every other sample in this industry stores
  raw px and converts at each use site - this one is cleaner, and worth copying.

**Eight findings.** The one that matters most is a performance defect that
also shows visually: every touch-move re-strokes the whole accumulated path.

## Feature checklist

- A start page that pushes the paint page.
- A canvas with a selectable background colour or a faint tracing template.
- Drawing follows the finger in the selected colour and thickness.
- A colour sheet of 24 pen colours with a confirm step.
- A canvas sheet of 24 background colours.
- A template sheet of tracing images.
- A thickness slider, an eraser toggle, and a clear-all button.

## Architecture

One `entry` module, two pages, three components, one constants file.

```
entry/src/main/ets
├── components
│   ├── BottomButtonItem.ets   ButtonItem and CanvasButtonItem - icon + label
│   ├── BottomMenuBar.ets      four tool buttons and their three sheets
│   └── TitleBar.ets
├── constants/StyleConstants.ets   CANVAS_COLOR, PEN_COLOR, SIMPLE_IMAGE, button lists
├── entryability/EntryAbility.ets  full screen, px2vp into AppStorage
└── pages
    ├── StartPage.ets          the page; pushes paintPage
    └── PaintPage.ets          the NavDestination: canvas, touch loop, tools
```

The documented tree matches the zip.

**`StartPage` is the only page; `PaintPage` is a destination.** `main_pages.json`
lists `pages/StartPage`, and `route_map.json` registers `paintPage` against
`paintPageBuilder`. That is the layering `KIDS-03` got wrong by marking its
destination `@Entry` as well.

**Tool state is `@Provide` on the page and `@Consume` in the menu bar** -
`isEraserMode`, `showThickness`, `canvasColor`, `penColor`, `penColorList`,
`canvasBackgroundImage`. The menu bar is two levels down, so providing beats
threading parameters.

**The canvas is a `Stack` of four things**, back to front: the template image
at `opacity(0.3)`, the `Canvas` itself, the eraser/clear buttons pinned at
`85%/2%`, and the thickness slider pinned at `5%/92%`. The template sits
*behind* a transparent canvas rather than being drawn into it, which is what
lets it stay faint while the child's ink stays solid:

```typescript
.backgroundColor(this.canvasBackgroundImage === '' ? this.canvasColor : Color.Transparent)
```

## Implementation steps

1. **Create the context with anti-aliasing on** - this sample does.
2. **On `Down`**, record the point, reset the stroke state, set `lineCap`.
3. **On `Move`**, stroke only the new segment (`HW-08-0053`).
4. **Guard `Move` with an `isDrawing` flag** set on `Down` and cleared on `Up`.
5. **Centre the eraser on the touch and scale it with the brush**
   (`HW-08-0054`).
6. **Clear using the canvas's own size**, not a literal (`HW-08-0055`).
7. **Give each tool its own sheet flag** with `$$`, and commit colour choices
   through a temporary.

## Verified snippets

All snippets are from `DrawBoard.zip`. Corrected forms are marked.

**The stroke loop — `entry/src/main/ets/pages/PaintPage.ets`** (corrected, see `HW-08-0053`, `HW-08-0054`, `HW-08-0059`)

```typescript
private settings: RenderingContextSettings = new RenderingContextSettings(true);
private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
private x: number = 0;
private y: number = 0;

Canvas(this.context)
  .onTouch((event) => {
    if (event.type === TouchType.Down) {
      this.showThickness = false;          // tapping the board dismisses the slider
      this.isDrawing = true;
      this.x = event.touches[0].x;
      this.y = event.touches[0].y;
      this.context.lineCap = 'round';      // round caps hide the segment joins
      if (this.isEraserMode) {
        this.eraseAt(this.x, this.y);
      }
      // FIX: the sample also does `this.context.beginPath()` and builds a
      // Path2D here; both are inert because stroking goes through the Path2D.
    }
    if (event.type === TouchType.Up || event.type === TouchType.Cancel) {
      this.isDrawing = false;
    }
    if (event.type === TouchType.Move) {
      if (!this.isDrawing) {
        return;                            // ignore a drag that began elsewhere
      }
      const nx = event.touches[0].x;
      const ny = event.touches[0].y;
      if (this.isEraserMode) {
        this.eraseAt(nx, ny);
      } else {
        this.isEmpty = false;
        this.context.strokeStyle = this.penColorList[this.penColor];
        this.context.lineWidth = this.paintSize;
        // FIX: the sample appends to a Path2D and calls
        //   this.context.stroke(this.tempPath)
        // which repaints every segment of the stroke so far.
        this.context.beginPath();
        this.context.moveTo(this.x, this.y);
        this.context.lineTo(nx, ny);
        this.context.stroke();
        this.x = nx;
        this.y = ny;
      }
    }
  });

eraseAt(x: number, y: number) {
  // FIX: the sample calls clearRect(x, y, 20, 20) - the touch point is used as
  // the rectangle's upper-left corner, and the brush size is ignored.
  const r = this.paintSize * 2;
  this.context.clearRect(x - r / 2, y - r / 2, r, r);
}
```

**The `isDrawing` guard is the piece `KIDS-03` is missing.** Without it a
`Move` arriving from a gesture that started outside the canvas draws a line
from a stale point. Setting it on `Down` and clearing it on `Up` costs one
boolean and removes the whole class.

**`lineCap = 'round'`** is why a freehand line looks continuous: segments are
drawn one per move event, and butt caps leave visible notches at every joint on
a curve.

**Stroking only the newest segment is the difference between linear and
quadratic work.** `stroke(path)` paints the *entire* path it is given, so
accumulating into a `Path2D` and re-stroking it means segment *n* is drawn *n*
times - and with anti-aliasing on, the repeated coverage makes the start of a
line visibly darker and thicker than its end.

**The tool sheets — `entry/src/main/ets/components/BottomMenuBar.ets`** (as shipped)

```typescript
private sheetHeight: number = 800;

ButtonItem({ data: this.buttonList[1] })
  .onClick(() => {
    this.penTempColor = this.penColor;     // seed the temporary from the live value
    this.isEraserMode = false;             // choosing a tool leaves eraser mode
    this.showPenColorBuilder = !this.showPenColorBuilder;
  })
  .bindSheet($$this.showPenColorBuilder, this.penColorBuilder(), {
    height: this.sheetHeight,
    backgroundColor: Color.White,
    onAppear: () => {
      this.penTempColor = this.penColor;   // and again on appear, for a swipe-open
    }
  });
```

**Every tool button clears `isEraserMode` first.** Picking up any tool should
put the eraser down, and doing it in all four handlers means there is no path
that leaves the board in eraser mode with a colour sheet open.

**Seeding the temporary in two places is deliberate**, not redundant: the
`onClick` covers the button press and `onAppear` covers the sheet being shown
any other way, so the checkmark always starts on the colour currently in use.

**The commit is one line and only in the confirm handler:**

```typescript
Button($r('app.string.confirm'))
  .onClick(() => {
    this.penColor = this.penTempColor;     // commit
    this.showPenColorBuilder = false;
  });
```

Tapping a swatch moves `penTempColor` and nothing else, so dismissing the sheet
by swipe leaves the pen as it was. That is the pattern to copy for any picker
that can be cancelled - `SPORT-13` uses the deep-copy version of the same idea.

**Choosing a background — same file** (as shipped)

```typescript
Rect()
  .width(110).height(58).radius(8)
  .fill(item)
  .onClick(() => {
    this.canvasBackgroundImage = '';       // a colour clears the template
    this.canvasColor = this.canvasColorList[index];
    this.showCanvasBuilder = false;
  });
```

**The two backgrounds are mutually exclusive and the code enforces it** by
clearing the other one on each selection, rather than by a mode flag. The
canvas then picks which to show with the ternary on `backgroundColor`.

**Clearing the board — `entry/src/main/ets/pages/PaintPage.ets`** (corrected, see `HW-08-0055`)

```typescript
Image(this.isEmpty ? $r('app.media.clear1') : $r('app.media.clear2'))
  .width(40)
  .onClick(() => {
    // FIX: the sample passes clearRect(0, 0, 1080, 1922) - a device pixel
    // resolution, in a coordinate space the rest of the file treats as vp.
    this.context.clearRect(0, 0, this.context.width, this.context.height);
    this.isEmpty = true;
  });
```

**`isEmpty` drives both tool icons** - `clear1`/`clear2` and
`eraser1`/`eraser2` - so the buttons look inactive on a blank board. A cheap,
readable way to give a child feedback about what is available.

**The px-to-vp boundary — `entry/src/main/ets/entryability/EntryAbility.ets`** (as shipped)

```typescript
let topRectHeight = avoidArea.topRect.height;              // px, per the window reference
AppStorage.setOrCreate('topRectHeight', uiContext.px2vp(topRectHeight));
```

**Convert once, at the boundary.** `getWindowAvoidArea` returns px and layout
attributes take vp; doing the conversion before the value enters `AppStorage`
means no consumer can forget it. `KIDS-02`, `KIDS-04` and `KIDS-06` all store
raw px and convert at each use, which is the same conversion written three
times.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. The album and camera
buttons in the canvas sheet are mock-ups that raise a toast, which is why no
media permission appears.

`main_pages.json` lists `pages/StartPage`; `route_map.json` registers:

```json
{
  "routerMap": [
    {
      "name": "paintPage",
      "pageSourceFile": "src/main/ets/pages/PaintPage.ets",
      "buildFunction": "paintPageBuilder",
      "data": { "description": "this is PaintPage" }
    }
  ]
}
```

`StartPage` publishes its stack with `AppStorage.setOrCreate('pathStack', ...)`
in `aboutToAppear`, though nothing in the sample reads it back.

`resources` contains `base`, `en_US` and `zh_CN` - but the two locale
directories hold only the three scaffold strings (`HW-08-0057`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Nothing is saved.** There is no export, no file write and no persistence;
  closing the page loses the drawing. `KIDS-03` is the sample that writes the
  canvas out with `toDataURL`.
- **No undo.** Strokes go straight into the canvas bitmap and the only reversal
  is the eraser or clear-all.
- **Four of the eight tool buttons are mock-ups**: texture canvas, choose from
  album, take a photo, and the custom-colour button all lead nowhere - the
  first three say so with a toast, the fourth does not (`HW-08-0056`).
- **The palettes are fixed** - 24 background colours and 24 pen colours as hex
  literals in `StyleConstants.ets`, with no custom-colour path.
- **The tool panels are a fixed 800 units tall** (`sheetHeight`), and the
  buttons inside them are positioned at percentages of that, so the sheets do
  not adapt to short windows.
- Multi-touch is not handled: `touches[0]` is taken without a length check or a
  finger id, so a resting palm can redirect the stroke - the same gap as
  `KIDS-03`.

## Pitfalls

- **`HW-08-0053` — every `Move` calls `stroke(tempPath)` on a `Path2D` that
  grows by one segment per event,** so drawing one line is quadratic in its
  length and, with anti-aliasing on, the repeated coverage darkens and thickens
  the earlier part of the stroke.
- **`HW-08-0054` — the eraser is `clearRect(x, y, 20, 20)`,** which takes the
  touch point as the rectangle's *upper-left corner*, so it erases below and to
  the right of the finger, and it ignores the 1-to-10 brush size entirely.
- **`HW-08-0055` — clear-all passes `clearRect(0, 0, 1080, 1922)`,** a device
  pixel resolution used in the vp space the rest of the file works in. It works
  only because it over-covers.
- **`HW-08-0057` — eight toolbar labels are Chinese literals in
  `ButtonType.txt`** while the icon beside each is a `Resource`, and the
  `en_US` and `zh_CN` directories the app ships contain no application strings
  at all.
- **`HW-08-0056` — the custom-colour button has no `onClick`,** while the four
  other unimplemented buttons in the same file raise an explicit toast.
- **`HW-08-0058` — the `avoidAreaChange` listener is never released,** and
  `bottomRectHeight` is bound but never applied, so the navigation indicator
  sits over the toolbar.
- **`HW-08-0059` — `beginPath()` and `closePath()` are called on the context**
  while all stroking goes through a `Path2D`, so both are inert.
- **`HW-08-0060` — `.rowsGap(12).rowsGap(10)` sets the same attribute twice,**
  and `StyleConstants.PEN_COLOR` (the number 19) shares its name with the
  exported `PEN_COLOR` colour array in the same file.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `stroke`, `clearRect`, `lineCap`, `lineWidth`, `RenderingContextSettings`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-path2d.md` - `Path2D` and the `stroke(path)` overload
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-path2d
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `TouchType` and `TouchObject`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/ts-universal-attributes-sheet-transition.md` - `bindSheet` and its options
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-attributes-sheet-transition
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-slider.md` - the thickness slider
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-slider
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getWindowAvoidArea` in px, `off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `KIDS-03` - the same stroke loop without a toolbar, and with a `toDataURL` export this sample lacks
- `KIDS-09` - fixed shapes drawn through the same `onTouch` handler
- `SPORT-14` - the other `bindSheet` tool-panel sample in this corpus
