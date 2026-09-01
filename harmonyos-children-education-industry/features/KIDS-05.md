---
id: KIDS-05
title: Go board and stones - a computed grid with snap-to-intersection placement
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/05_canvas_go_chess_pieces.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/canvas_go_chess_pieces-0000002289055858
sample: huawei_industry_tree/08_children_education/downloads/canvasCheckerboard.zip
kits: ["@kit.ArkUI"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, onReady, strokeRect, beginPath, moveTo, lineTo, stroke, arc, fill, fillText, font, textAlign, textBaseline, TapGesture, GestureEvent, fingerList, "display.getAllDisplays", "UIContext.px2vp", "MeasureUtils.measureTextSize", "@State"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0035, HW-08-0036, HW-08-0037, HW-08-0038, HW-08-0039, HW-08-0040, HW-08-0041, HW-08-0042, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for **a board game drawn on a canvas where pieces snap to a
computed grid** - Go, gomoku, Chinese chess, draughts, a battleship grid, any
lesson laid out on a lattice. The document names gomoku and Chinese chess as
the intended derivatives.

The three ideas worth taking are all in the same forty lines:

- **The grid is a spacing, not a picture.** One number, `spaceLen`, is derived
  from the board size and the line count; every line, every hit test and every
  stone position is that number times an index. Nothing is measured twice and
  nothing is stored per cell.
- **Snapping is `Math.round` on the same division.** Converting a touch to the
  nearest intersection is `Math.round((localX - margin) / spaceLen)` - the
  exact inverse of the drawing formula, which is why pieces land where the
  child aimed.
- **The label is fitted, not guessed.** `measureTextSize` is used in a search
  loop to find the largest font that fits inside a stone, so the move number
  scales with the board rather than being a hardcoded size.

**Eight findings.** The most important is structural: the board is painted
once, from `onReady`, using dimensions that arrive from a *separate* async
callback, with no ordering guarantee and no redraw path.

## Feature checklist

- A square 19x19 Go board with a border and a beige ground.
- Tapping anywhere places a stone on the nearest intersection.
- Stones alternate black and white.
- Each stone carries its move number, in the opposite colour.
- An occupied intersection and a tap outside the board are both ignored.

## Architecture

One `entry` module, one page, no components, no utils. 157 lines.

```
entry/src/main/ets
├── entryability/EntryAbility.ets   full screen, loads pages/Index
├── entrybackupability/
└── pages/Index.ets                 the whole feature
```

The documented tree matches the zip.

**The entire model is five numbers**, and they are worth listing because
everything else is derived:

| Field | Meaning |
| --- | --- |
| `flavor = 19` | lines per side |
| `borderMargin = 20` | vp between the canvas edge and the grid |
| `sideLen` | grid width in vp |
| `spaceLen` | `sideLen / (flavor - 1)` - one cell |
| `boardLen` | `spaceLen - 2` - stone diameter |

**`spaceLen` divides by `flavor - 1`, not by `flavor`.** Nineteen lines make
eighteen gaps, and a Go stone sits on the intersection rather than in the cell.
Getting this off by one puts the last line outside the board.

**Draw and hit-test are inverses of one another:**

```
draw   ──> x = borderMargin + spaceLen * i
hit    ──> i = Math.round((localX - borderMargin) / spaceLen)
```

`Math.round` rather than `Math.floor` is what makes the whole cell around an
intersection a target: floor would snap every touch to the intersection above
and left, so a stone would never land where the finger was.

**Occupancy is a plain boolean grid**, `hasBoardMark[i][j]`, filled in the same
pass that draws the board. There is no piece list and no undo - once a stone is
painted it is part of the canvas bitmap.

## Implementation steps

1. **Create the context with anti-aliasing on** - this sample does, and it is
   the reason the stones and diagonals look right.
2. **Take the board size from the canvas, not the screen** (`HW-08-0036`).
3. **Draw the grid from `onReady`**, and redraw whenever the size changes
   (`HW-08-0035`).
4. **Call `beginPath` per line**, or build one path and stroke it once
   (`HW-08-0038`).
5. **Snap the tap with `Math.round`** and reject out-of-range and occupied
   intersections before drawing.
6. **Fit the label with `measureTextSize`**, stepping back one size rather than
   two (`HW-08-0039`).
7. **Centre the number** with `textAlign = 'center'` and
   `textBaseline = 'middle'`.

## Verified snippets

All snippets are from `canvasCheckerboard.zip`. Corrected forms are marked.

**Setting up the context — `entry/src/main/ets/pages/Index.ets`** (as shipped)

```typescript
// 用来配置CanvasRenderingContext2D对象的参数，包括是否开启抗锯齿，true表明开启抗锯齿。
private settings: RenderingContextSettings = new RenderingContextSettings(true);
private context: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
```

**This is the line `KIDS-03` is missing.** Anti-aliasing is off by default, and
this sample draws 38 thin straight lines and 361 possible circles - the circles
are what make it matter. Copy this pair, not the bare constructor.

**Sizing the board — same file** (corrected, see `HW-08-0035` and `HW-08-0036`)

```typescript
// FIX: the sample computes these in an async display.getAllDisplays callback
// from the *screen* size, and draws from onReady with no ordering guarantee
// between the two and no redraw when the numbers arrive.
.onReady(() => {
  this.sideLen = this.context.width - this.borderMargin * 2;   // the canvas, not the display
  this.spaceLen = this.sideLen / (this.flavor - 1);
  this.boardLen = this.spaceLen - 2;
  this.drawCheckerboard();
})
.onAreaChange((_, newVal) => {
  const side = (newVal.width as number) - this.borderMargin * 2;
  if (side !== this.sideLen) {
    this.sideLen = side;
    this.spaceLen = this.sideLen / (this.flavor - 1);
    this.boardLen = this.spaceLen - 2;
    this.drawCheckerboard();                                    // repaint on resize
  }
});
```

**Canvas painting is imperative, so `@State` buys nothing here.** The sample
marks `sideLen`, `spaceLen`, `boardLen` and `count` as `@State`, which makes
them look like they drive the picture. They do not: a state change re-runs
`build`, and `build` issues no drawing commands. Anything that changes the
geometry has to call the draw function itself.

**The grid — same file** (corrected, see `HW-08-0038` and `HW-08-0040`)

```typescript
drawCheckerboard() {
  this.context.strokeRect(this.borderMargin, this.borderMargin, this.sideLen, this.sideLen);
  this.context.beginPath();                 // FIX: the sample never begins a path,
  for (let index = 0; index < this.flavor; index++) {   // so each stroke() redraws
    // 绘制竖线 - vertical: x fixed          // everything before it (741 segments
    this.context.moveTo(this.borderMargin + index * this.spaceLen, this.borderMargin);
    this.context.lineTo(this.borderMargin + index * this.spaceLen, this.sideLen + this.borderMargin);

    // 绘制横线 - horizontal: y fixed        // FIX: the sample's two comments are
    this.context.moveTo(this.borderMargin, this.borderMargin + index * this.spaceLen);
    this.context.lineTo(this.sideLen + this.borderMargin, this.borderMargin + index * this.spaceLen);
  }
  this.context.stroke();                    // one stroke for the whole lattice

  for (let i = 0; i < this.flavor; i++) {
    this.hasBoardMark[i] = [];
    for (let j = 0; j < this.flavor; j++) {
      this.hasBoardMark[i].push(false);
    }
  }
}
```

**`moveTo` + `lineTo` append to the current path; `stroke` paints all of it.**
That is the whole reason `beginPath` exists, and it is the single most common
Canvas mistake in this corpus. Building the lattice as one path and stroking
once is both correct and the fastest form.

**Initialising the occupancy grid inside the draw function** is deliberate:
redrawing the board is also how you reset the game, so the two are one
operation.

**Placing a stone — same file** (as shipped)

```typescript
TapGesture({ count: 1 })
  .onAction((event: GestureEvent | undefined) => {
    if (event) {
      let value = event.fingerList[0];

      let i = Math.round((value.localX - this.borderMargin) / this.spaceLen);
      let j = Math.round((value.localY - this.borderMargin) / this.spaceLen);

      if (i < 0 || j < 0 || i > (this.flavor - 1) || j > (this.flavor - 1) || this.hasBoardMark[i][j]) {
        return;                                    // off the board, or already taken
      }

      this.hasBoardMark[i][j] = true;
      let realX = this.borderMargin + this.spaceLen * i;
      let realY = this.borderMargin + this.spaceLen * j;
      this.drawBoard(realX, realY);
      this.count++;                                // parity flips the colour
    }
  })
```

**`localX`/`localY` are component-relative**, which is exactly what the grid
formula wants - no window or screen offset enters the calculation.

**The bounds test runs before the occupancy test**, so `hasBoardMark[i][j]` is
only indexed once `i` and `j` are known to be in range. That ordering is what
keeps the short-circuit safe; swapping the clauses would index out of bounds.

**`TapGesture` rather than `onTouch` is the right choice here.** A stone is
placed on a discrete tap, not dragged, so the gesture recogniser's own
press-and-release detection replaces the `Down`/`Move`/`Up` bookkeeping
`KIDS-03` and `KIDS-07` need.

**Drawing the stone — same file** (corrected, see `HW-08-0041`)

```typescript
drawBoard(realX: number, realY: number) {
  this.context.beginPath();
  this.context.arc(realX, realY, this.boardLen / 2, 0, Math.PI * 2);   // FIX: sample uses 6.28
  if (this.count % 2 === 0) {
    this.context.strokeStyle = '#000000';
    this.context.fillStyle = '#000000';
  } else {
    this.context.strokeStyle = '#FFFFFF';
    this.context.fillStyle = '#FFFFFF';
  }
  this.context.fill();
  this.context.stroke();

  let text = this.getShowNumber();
  let fontSize = this.getFontSize(text);
  this.context.font = fontSize + 'vp';
  this.context.fillStyle = this.count % 2 === 0 ? '#FFFFFF' : '#000000';   // inverse of the stone
  this.context.textAlign = 'center';
  this.context.textBaseline = 'middle';
  this.context.fillText(text, realX, realY);
}
```

**`textAlign = 'center'` with `textBaseline = 'middle'` is what lets the label
be drawn at the stone's centre point** rather than at a computed top-left. Two
attributes replace a text-measurement offset in both axes.

**The parity is read twice and incremented once, after the draw** - so the
first stone (`count === 0`) is black with white text and shows "1", because
`getShowNumber` returns `count + 1`.

**Fitting the label — same file** (corrected, see `HW-08-0039`)

```typescript
getFontSize(text: string): number {
  // FIX: the sample loops `for (let fontSize = 1;; fontSize++)` with no bound
  // and returns `fontSize - 2`, one step below the largest that fits.
  for (let fontSize = 1; fontSize <= 96; fontSize++) {
    const textSize = this.getUIContext().getMeasureUtils().measureTextSize({
      textContent: text,
      textAlign: TextAlign.JUSTIFY,
      constraintWidth: this.boardLen,        // vp
      fontSize: fontSize + 'vp'
    });
    if (this.getUIContext().px2vp(Number(textSize.width)) > this.boardLen ||
        this.getUIContext().px2vp(Number(textSize.height)) > this.boardLen) {
      return Math.max(1, fontSize - 1);
    }
  }
  return 96;
}
```

**The units here are easy to get wrong and this sample gets them right.**
`measureTextSize` returns width and height "both in px", `constraintWidth`
defaults to vp, and `boardLen` is vp - hence `px2vp` on the results and no
conversion on the constraint. Check this pairing before copying the loop
anywhere.

Note that `constraintWidth` caps the measured width, so in practice the height
comparison is what ends the search.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. No storage, no
network, no sensors - the sample is one page and a canvas.

`main_pages.json` lists `pages/Index` alone; there is no `routerMap` and no
navigation.

`EntryAbility` is the shortest in the industry: it pins light mode with
`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`, calls
`setWindowLayoutFullScreen(true)`, and loads the page. Both the discarded
promise on that call and the placeholder `EntryAbility_label` are
`HW-08-0042`.

`deviceTypes` is `["phone", "tablet", "2in1"]` with no orientation lock - which
is what makes the screen-derived geometry (`HW-08-0036`) a problem rather than
a detail.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **This is a drawing demo, not a game.** There are no Go rules: no capture,
  no ko, no liberties, no territory, no turn validation beyond "this
  intersection is free". `hasBoardMark` records occupancy and not colour.
- **Nothing can be undone, reset or saved.** Stones are painted into the canvas
  bitmap, so removing one means redrawing the board and every other stone, and
  no piece list is kept that would allow it. Closing the app loses the game.
- **The board size is fixed at 19 in a private field.** The document describes
  the grid as computed from `flavor * flavor`, and it is - but nothing exposes
  it, so gomoku's 15 or chess's 8 means editing the source.
- **The window is laid out full-screen and the page pads for nothing**, so the
  top of the grid sits under the status bar with only `borderMargin` of
  clearance.
- `getFontSize` runs a fresh measurement loop for every stone placed - roughly
  ten synchronous `measureTextSize` calls per tap. Acceptable at this rate;
  caching per digit-count would be the fix if it were not.

## Pitfalls

- **`HW-08-0035` — the board is drawn once from `onReady` using numbers set by
  an unrelated async callback.** Nothing orders the two, and nothing redraws
  when the numbers arrive, so the board can be painted at size zero and stay
  that way - a blank yellow square whose taps all compute `Infinity` and are
  silently rejected. The four `@State` fields make this look safe; imperative
  canvas drawing never re-runs on a state change.
- **`HW-08-0036` — the geometry comes from `display.getAllDisplays`, the
  physical screen,** not from the canvas. In split-screen, a resized 2in1
  window, or on a foldable, the grid is computed for the whole display and
  painted into a smaller square, and the hit test inherits the same wrong
  `spaceLen`.
- **`HW-08-0038` — `drawCheckerboard` never calls `beginPath`,** so the path
  only grows and each `stroke()` repaints everything before it: 741 segment
  strokes for a 38-line board, with the earliest lines painted 38 times over.
- **`HW-08-0039` — `getFontSize` returns `fontSize - 2`,** one step below the
  largest size measured to fit, and has no lower clamp, so a small board yields
  a negative size handed to `context.font`. The loop has no upper bound either.
- **`HW-08-0037` — the display callback binds `err`, never reads it, and
  dereferences `data[0]` on the next line,** where the reference's own example
  checks `err.code` and returns first.
- **`HW-08-0041` — stones are swept `0` to `6.28` instead of `Math.PI * 2`,**
  leaving the stroked outline open by a hairline at three o'clock. `fill`
  closes the region, `stroke` does not.
- **`HW-08-0040` — the comments labelling the horizontal and vertical grid
  lines are swapped,** in the sample and in the document that quotes it.
- **`HW-08-0042` — `setWindowLayoutFullScreen` is called without `then`/`catch`,**
  and the app ships with `EntryAbility_label` set to the literal `"label"`.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component and `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `RenderingContextSettings`, `beginPath`, `arc`, `fillText`, `textAlign`, `textBaseline`, `font`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-measureutils.md` - `measureTextSize` and its px return values
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-measureutils
- `documentation/harmonyos-references/02_application-framework/js-apis-measure.md` - `MeasureOptions`, and `constraintWidth` defaulting to vp
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-measure
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` - `getAllDisplays`, `Display.width`/`height` in px, and the error-checked example
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-tapgesture.md` - `TapGesture` and `GestureEvent.fingerList`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-tapgesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setWindowLayoutFullScreen`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `KIDS-03` - the freehand board, and the `onAreaChange` sizing this sample should have used
- `KIDS-09` - fixed shapes on a canvas, the third of the industry's six Canvas samples
- `KIDS-15` - the Schulte grid, the same lattice built from ArkUI components instead of drawn
