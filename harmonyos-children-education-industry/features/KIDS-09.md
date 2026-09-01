---
id: KIDS-09
title: Rubber-band shape drawing - rectangle, circle and triangle sized by drag
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/09_draw_fixed_shape.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/draw_fixed_shape-0000002366469545
sample: huawei_industry_tree/08_children_education/downloads/DrawFixedShape.zip
kits: ["@kit.ArkUI"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, Path2D, onTouch, TouchType, rect, fillRect, arc, moveTo, lineTo, closePath, stroke, fill, clearRect, lineWidth, strokeStyle, fillStyle, bindSheet, Slider, "@Provide", "@Consume", "@StorageProp", NavDestination]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0068, HW-08-0069, HW-08-0070, HW-08-0071, HW-08-0072, HW-08-0073, HW-08-0074, HW-08-0075, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for **rubber-band drawing**: the user presses to anchor a
shape, drags to size it, and the shape follows the finger live. A drawing tool
here, and equally a crop box, a selection marquee, a map radius picker, a
measurement overlay.

This sample is `KIDS-07` with a shape mode bolted on. Take `KIDS-07` for the
toolbar and the sheets; take this card for the **shape geometry**, which is
the part that is genuinely reusable:

- **Anchor on `Down`, resize on `Move`.** `(x0, y0)` is the press point and
  `(x, y)` is the current finger; every shape is a function of the two.
- **One squared distance drives circle and triangle.**
  `length = dx² + dy²`, then `Math.sqrt(length)` is the radius - and for the
  triangle, the circumradius of an equilateral triangle inscribed in that same
  circle.

**Eight findings, and the shape mode's own behaviour is the problem.** Every
new shape wipes the canvas, and the live preview erases by painting over the
picture. The geometry is worth copying; the drawing strategy is not.

## Feature checklist

- A start page pushing the paint page.
- A four-tool toolbar: canvas colour, pen colour, brush, shape.
- A shape sheet offering rectangle, circle and triangle.
- Freehand drawing when no shape is selected.
- Pressing anchors the shape; dragging sizes it live.
- An eraser toggle, a thickness slider and a clear-all button.

## Architecture

Identical in shape to `KIDS-07`, with one extra tool and one extra constant
list.

```
entry/src/main/ets
├── components
│   ├── BottomButtonItem.ets
│   ├── BottomMenuBar.ets      four tools; the fourth picks a shape
│   └── TitleBar.ets
├── constants/StyleConstants.ets   CANVAS_COLOR, PEN_COLOR, SIMPLE_IMAGE, SHAPE
├── entryability/EntryAbility.ets
└── pages
    ├── StartPage.ets
    └── PaintPage.ets          one onTouch handler, two modes
```

The documented tree matches the zip.

**The mode switch is the top of the touch handler**, and everything hangs off
one number:

```
shapeNum === 3  ──> freehand: Path2D, accumulate, stroke
shapeNum <  3   ──> shape:    anchor on Down, redraw on every Move
                     0 = rectangle   1 = circle   2 = triangle
```

`3` meaning "no shape, draw freehand" is not obvious and is the default
(`HW-08-0075`).

**The triangle is inscribed in the circle the drag defines**, which is why one
`length` serves both tools:

```
r = √length                       the drag distance
top vertex     (x0,        y0 - r)
right vertex   (x0 + √(0.75·length), y0 + r/2)
left vertex    (x0 - √(0.75·length), y0 + r/2)
```

Each vertex is exactly `r` from the anchor - `√(0.75L + L/4) = √L` - so the
triangle is equilateral and the anchor is its centroid. That is the neat bit,
and `√0.75 ≈ cos(30°)` is where it comes from.

## Implementation steps

1. **Anchor `(x0, y0)` on `Down`** - and do *not* clear the canvas
   (`HW-08-0070`).
2. **Recompute the shape from `(x0, y0)` to `(x, y)` on every `Move`.**
3. **Preview on a second overlaid `Canvas`**, so nothing has to be erased from
   the picture (`HW-08-0071`).
4. **Sweep circles in radians** - `Math.PI * 2`, not `360` (`HW-08-0069`).
5. **Commit the shape to the main canvas on `Up`.**
6. **Stroke each shape once** (`HW-08-0075`).
7. **Centre the eraser on the touch and scale it with the brush**
   (`HW-08-0073`).

## Verified snippets

All snippets are from `DrawFixedShape.zip`. Corrected forms are marked.

**The three shapes — `entry/src/main/ets/pages/PaintPage.ets`** (corrected, see `HW-08-0069` and `HW-08-0075`)

```typescript
this.context.beginPath();
this.x = event.touches[0].x;
this.y = event.touches[0].y;
this.context.strokeStyle = this.penColorList[this.penColor];
this.context.lineWidth = this.paintSize;

if (this.shapeNum === 0) {                       // rectangle
  this.context.rect(this.x0, this.y0, this.x - this.x0, this.y - this.y0);
  this.context.fillStyle = this.penColorList[this.penColor];
  this.context.fillRect(this.x0, this.y0, this.x - this.x0, this.y - this.y0);
} else if (this.shapeNum === 1) {                // circle
  const length = (this.x - this.x0) ** 2 + (this.y - this.y0) ** 2;
  // FIX: the sample passes 360 - arc takes RADIANS, so that is ~57 turns
  this.context.arc(this.x0, this.y0, Math.sqrt(length), 0, Math.PI * 2);
  this.context.fillStyle = this.penColorList[this.penColor];
  this.context.fill();
} else if (this.shapeNum === 2) {                // equilateral triangle
  const length = (this.x - this.x0) ** 2 + (this.y - this.y0) ** 2;
  this.context.moveTo(this.x0, this.y0 - Math.sqrt(length));
  this.context.lineTo(this.x0 + Math.sqrt(length * 0.75), this.y0 + Math.sqrt(length) / 2);
  this.context.lineTo(this.x0 - Math.sqrt(length * 0.75), this.y0 + Math.sqrt(length) / 2);
  this.context.closePath();
  this.context.fillStyle = this.penColorList[this.penColor];
  this.context.fill();
  // FIX: the sample also strokes here, then again below - the triangle's
  // outline is painted twice and comes out darker than the other two shapes.
}
this.context.stroke();
```

**`rect` adds to the path; `fillRect` paints immediately.** The rectangle
branch uses both because it wants a filled body *and* an outline in the trailing
`stroke()`. The circle and triangle get the same effect with `fill()` on the
path they just built - three tools, three idioms, one visual result.

**The rectangle needs no normalisation.** `rect(x0, y0, w, h)` accepts negative
width and height, so dragging up-and-left works without swapping the corners -
worth knowing, because the equivalent hand-rolled path would need it.

**`beginPath()` before each redraw is essential here** and the sample does it:
without it every `Move` would append another shape to one growing path and
`stroke()` would repaint all of them.

**Anchoring the drag — same file** (corrected, see `HW-08-0070`)

```typescript
} else if (this.shapeNum < 3) {
  if (event.type === TouchType.Down) {
    // FIX: the sample calls clearRect(0, 0, 1080, 1922) here, wiping the whole
    // canvas - so only one shape can exist and freehand work is destroyed.
    this.showThickness = false;
    this.isDrawing = true;
    this.x0 = event.touches[0].x;      // the anchor: only x0/y0 are set on Down
    this.y0 = event.touches[0].y;
  }
```

**Only the anchor is recorded on `Down`.** `x` and `y` stay at their previous
values until the first `Move`, which is why the two eraser calls that read
`this.x`/`this.y` in this branch operate on stale coordinates (`HW-08-0073`).

**Erasing the previous preview — same file** (corrected, see `HW-08-0071`)

```typescript
// FIX: the sample removes the last preview by painting over it -
//   rectangle: clearRect(...)                        punches a transparent hole
//   circle/triangle: arc(...) + fill(canvasColor)    stamps opaque background
// Both destroy anything already on the canvas in that region, and the erased
// area grows with the drag.
//
// Preview on a second Canvas instead:
Stack() {
  Canvas(this.context)        // committed artwork
  Canvas(this.preview)        // in-progress shape only
    .onTouch((event) => {
      this.preview.clearRect(0, 0, this.preview.width, this.preview.height);
      this.drawShape(this.preview);
      if (event.type === TouchType.Up) {
        this.drawShape(this.context);                 // commit
        this.preview.clearRect(0, 0, this.preview.width, this.preview.height);
      }
    })
}
```

**A bitmap canvas has no undo**, so "erase the last frame" can only mean
"destroy that region". The two-surface pattern is the standard answer and it
also removes the need for the `flag` bookkeeping the sample carries.

**Freehand mode — same file** (corrected, see `HW-08-0072`)

```typescript
if (this.shapeNum === 3) {
  if (event.type === TouchType.Move) {
    if (!this.isDrawing) { return; }        // guard: ignore drags begun elsewhere
    this.context.strokeStyle = this.penColorList[this.penColor];
    // FIX: the sample appends to a Path2D and calls stroke(this.tempPath),
    // repainting every segment of the stroke so far.
    this.context.beginPath();
    this.context.moveTo(this.x, this.y);
    this.context.lineTo(event.touches[0].x, event.touches[0].y);
    this.context.stroke();
    this.x = event.touches[0].x;
    this.y = event.touches[0].y;
  }
}
```

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`.

`main_pages.json` lists `pages/StartPage`; `route_map.json` registers
`paintPage` against `paintPageBuilder`, and `PaintPage` is a `NavDestination` -
the same correct layering as `KIDS-07`.

`StartPage` publishes its stack through `AppStorage.setOrCreate('pathStack', ...)`
and `TitleBar` reads it back with `AppStorage.get('pathStack')` - the only two
`AppStorage` uses in the project, which is what makes `HW-08-0068` visible:
`topRectHeight` and `bottomRectHeight` are bound by the page and **never
written by anyone**.

`resources` contains `base`, `en_US` and `zh_CN`, and the two locale
directories hold only the three scaffold strings (`HW-08-0074`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **One shape at a time.** Starting a shape clears the board, so shapes cannot
  be combined with each other or with freehand drawing (`HW-08-0070`).
- **Shapes are always filled** with the pen colour - there is no outline-only
  mode, so the thickness slider only affects the border of a solid shape.
- **The triangle is always equilateral and point-up**; its orientation and
  proportions cannot be changed by the drag, only its size.
- **The circle is anchored at its centre** while the rectangle is anchored at
  its corner, so the two tools respond differently to the same gesture.
- **Nothing is saved** - no export, no persistence, no undo.
- Multi-touch is unhandled: `touches[0]` with no length check and no finger id.
- Four of the toolbar's mock-up buttons (texture canvas, album, camera) raise a
  placeholder toast, as in `KIDS-07`.

## Pitfalls

- **`HW-08-0068` — the safe-area machinery is present and gutted.** The
  `avoidAreaChange` listener has two empty branches, `getWindowAvoidArea` is
  never called, the two `@StorageProp` fields are never read, and the inset is
  a hardcoded `padding({top: 24})` on a window laid out full-screen. The
  sibling `KIDS-07` does the whole chain properly.
- **`HW-08-0070` — `Down` in shape mode calls `clearRect(0, 0, 1080, 1922)`,**
  wiping the entire canvas, so a child who draws a picture and then reaches for
  the circle tool loses the picture.
- **`HW-08-0071` — the live preview erases by painting over the canvas**
  (transparent for rectangles, opaque `canvasColor` for circles and triangles),
  destroying anything already drawn in a region that grows with the drag.
- **`HW-08-0069` — `arc(..., 0, 360)` passes degrees where the API takes
  radians** - about 57 turns. It works only because over-sweeping still closes
  a circle. `KIDS-05` makes the opposite error with `6.28`.
- **`HW-08-0072` — freehand mode re-strokes the whole `Path2D` on every
  `Move`,** quadratic in stroke length, and with anti-aliasing on the start of a
  line comes out darker than its end.
- **`HW-08-0073` — the eraser is `clearRect(x, y, 20, 20)`,** anchored at the
  touch point's upper-left corner rather than centred, ignoring the 1-to-10
  brush size; two of its four call sites read stale `this.x`/`this.y`.
- **`HW-08-0074` — nine toolbar captions are Chinese literals** beside `$r`
  icons, and the `en_US`/`zh_CN` directories carry no application strings.
- **`HW-08-0075` — the triangle is stroked twice** (inside its branch and again
  after), so its outline is darker than the other two shapes; and the tool is a
  bare number where `3` means freehand.

## References

- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `rect`, `fillRect`, `arc` (radians), `moveTo`/`lineTo`, `closePath`, `clearRect`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-path2d.md` - `Path2D` and the `stroke(path)` overload
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-path2d
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `TouchType`, `TouchObject`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getWindowAvoidArea`, `on`/`off('avoidAreaChange')`, `setWindowLayoutFullScreen`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `KIDS-07` - the same toolbar and sheets, with the safe-area chain intact
- `KIDS-03` - the bare freehand loop
- `KIDS-05` - the other `arc` sample, with the opposite angle-unit mistake
