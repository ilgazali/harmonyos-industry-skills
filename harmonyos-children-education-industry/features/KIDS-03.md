---
id: KIDS-03
title: Handwriting practice board - freehand Canvas strokes over a generated guide grid
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/03_children_demo_canvas.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/children_demo_canvas-0000002237313638
sample: huawei_industry_tree/08_children_education/downloads/CanvasDemo.zip
kits: ["@kit.ArkUI", "@kit.CoreFileKit", "@kit.ArkTS", "@kit.AbilityKit"]
apis: [Canvas, CanvasRenderingContext2D, RenderingContextSettings, onReady, onTouch, onAreaChange, TouchObject, TouchType, beginPath, moveTo, lineTo, stroke, closePath, clearRect, toDataURL, "fileIo.openSync", "fileIo.writeSync", "fileIo.closeSync", "buffer.from", NavPathStack, NavDestination, pushPathByName, "@Provide", "@Consume", "window.setWindowLayoutFullScreen", "window.setSpecificSystemBarEnabled"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0014, HW-08-0015, HW-08-0016, HW-08-0017, HW-08-0018, HW-08-0019, HW-08-0020, HW-08-0021, HW-08-0022, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for **anything the user draws with a finger**: a handwriting or
calligraphy trainer, a signature pad, a sketch tool, a shape-tracing exercise.
It is the first of the industry's six Canvas samples and the one that
establishes the pattern the others reuse.

Three pieces are worth taking:

- **The four-line touch-to-stroke loop.** `beginPath` + `moveTo` on `Down`,
  `lineTo` + `stroke` on `Move`, `closePath` on `Up`. That is the whole
  drawing engine, and every other Canvas sample in this industry repeats it.
- **A guide grid generated rather than drawn as an image.** The Chinese
  practice-square lattice - two dashed diagonals, a dashed vertical and a
  dashed horizontal - is computed from the canvas width, so it fits whatever
  size the canvas ends up.
- **Exporting the canvas to a file.** `toDataURL` gives a base64 data URL,
  which is split, decoded through `buffer.from` and written into the app's
  temp directory, so the finished work can be shown back on a results page.

**Read the Pitfalls before copying.** Nine findings, and the two that matter
most are cheap to fix: the context is created with anti-aliasing off (the
documented default) on a board whose entire output is freehand strokes, and
the touch handler cannot tell one finger from another on a device a child
will rest a hand on.

## Feature checklist

- A large square canvas with a pale-yellow ground and a grey guide lattice.
- The character to trace is shown behind the canvas at large size.
- Drawing follows the finger in the selected colour and thickness.
- A clear button wipes the canvas and redraws the guide.
- A strip along the bottom shows all five characters, the current one raised.
- Forward and back arrows move through the five; forward also saves the
  current canvas.
- After the fifth, a completion screen shows all five saved images.

## Architecture

One `entry` module, one page, one destination, one small component.

```
entry/src/main/ets
├── components/BottomText.ets       one character tile in the bottom strip
├── entryability/EntryAbility.ets   full screen, hides the navigation indicator
├── entrybackupability/
└── pages
    ├── WriteBoard.ets              the board, the grid, the export, the arrows
    └── Finish.ets                  NavDestination showing the five results
```

The documented tree matches the zip exactly.

**Navigation is a single page with one destination.** `WriteBoard` is the
page; `Finish` is reached with `pushPathByName('Finish', this.imgUrl)` and
built by the page's own `@Builder` map. The five file URLs travel as the route
parameter and are read back in `onReady`, which the reference documents as
firing *before* the destination builds its children - so the plain field they
are stored in is populated in time for the first render.

**The saved page is a file, not a data URL in memory.** Each forward tap does
`toDataURL` then writes the decoded bytes into `context.tempDir`, keeping only
a `file://` path in state. That is the right shape: five full-canvas PNGs held
as base64 strings would be megabytes of component state.

```
draw ──> toDataURL() ──> split(';base64,') ──> buffer.from(_, 'base64')
                                                      │
                                          fileIo.writeSync into tempDir
                                                      │
                                   imgUrl[n] = 'file://' + path
                                                      │
                        n === 4 ──> pushPathByName('Finish', imgUrl)
```

**The guide grid is drawn as dashes, not as dashed lines.** `drawLine` walks
across the square emitting one short segment every two steps, four times - two
diagonals, a vertical, a horizontal. The step for the axis-aligned lines is
multiplied by `sqrt(2)` so their dashes look the same length as the diagonal
ones, which are drawn at 45 degrees.

## Implementation steps

1. **Create the context with anti-aliasing on** - the default is off
   (`HW-08-0014`).
2. **Record the canvas size in `onAreaChange`**, since `clearRect` needs it.
3. **Draw the guide in `onReady`** and again after every clear.
4. **Handle `Down`/`Move`/`Up` in `onTouch`**, checking `touches` is non-empty
   and tracking the finger by `id` (`HW-08-0017`).
5. **Export with `toDataURL`** and name the file for the format it actually
   returns (`HW-08-0016`).
6. **Write it with `openSync`/`writeSync`/`closeSync` in a `try`/`finally`**
   (`HW-08-0015`).
7. **Push the destination on the last character** and read the parameter in
   `onReady`.

## Verified snippets

All snippets are from `CanvasDemo.zip`. Corrected forms are marked.

**The drawing loop — `entry/src/main/ets/pages/WriteBoard.ets`** (corrected, see `HW-08-0014` and `HW-08-0017`)

```typescript
// FIX: the sample writes `new CanvasRenderingContext2D()`, which leaves
// antialias at its documented default of false.
private settings: RenderingContextSettings = new RenderingContextSettings(true);
private canvasContext: CanvasRenderingContext2D = new CanvasRenderingContext2D(this.settings);
private activeFinger: number = -1;          // FIX: the sample tracks no finger id

Canvas(this.canvasContext)
  .aspectRatio(1)
  .height('95%')
  .backgroundColor('#FEFFF1')
  .borderRadius(20)
  .opacity(0.7)
  .onReady(() => {
    this.drawLine(this.canvasContext, 20);   // the guide grid, once the size is known
  })
  .onAreaChange((_, newVal) => {
    this.canvasWidth = newVal.width as number;
    this.canvasHeight = newVal.height as number;
  })
  .onTouch((event) => {
    if (event.touches.length === 0) {        // FIX: the reference requires this check
      return;
    }
    const touch: TouchObject = event.touches[0];
    switch (event.type) {
      case TouchType.Down:
        this.activeFinger = touch.id;
        this.canvasContext.beginPath();
        this.canvasContext.moveTo(touch.x, touch.y);
        this.clearOpacity = 1;               // the clear button becomes fully opaque
        break;
      case TouchType.Move:
        if (touch.id !== this.activeFinger) { return; }
        this.canvasContext.lineTo(touch.x, touch.y);
        this.canvasContext.lineWidth = this.selectedWidth;
        this.canvasContext.strokeStyle = this.modeIndex === 0 ? this.selectedColor : 'white';
        this.canvasContext.stroke();
        break;
      case TouchType.Up:
      case TouchType.Cancel:                 // FIX: the sample handles only Up
        this.canvasContext.closePath();
        break;
      default:
    }
  });
```

**`onReady` is the only safe place to draw.** It fires when the canvas is
ready and its size is known; drawing from `aboutToAppear` would run before
layout, when `ctx.width` is not yet the laid-out width.

**`onAreaChange` exists because `clearRect` needs numbers.** `ctx.width` and
`ctx.height` are available inside drawing callbacks, but the clear button's
`onClick` is not one, so the size is cached into fields as the component is
laid out.

**The eraser is the background colour, not a real eraser.** `strokeStyle` is
`'white'` in erase mode - which works over a plain ground and would not over
the guide lattice, since erasing paints over the grid too. That is why the
clear button redraws the grid rather than only clearing.

**The guide grid — same file** (corrected, see `HW-08-0021`)

```typescript
getPoints = (r: number, l: number) => {
  let points: number[][] = [[], [], [], []];
  points[0] = [r - Math.sqrt(r * r / 2), r - Math.sqrt(r * r / 2)];   // inset by the corner radius
  points[1] = [l - points[0][0], points[0][1]];
  points[2] = [points[1][1], points[1][0]];
  points[3] = [l - points[0][0], l - points[0][1]];
  return points;
};

drawLine = (ctx: CanvasRenderingContext2D, r: number) => {
  const width = ctx.width;
  const height = ctx.height;
  let points = this.getPoints(r, width);
  ctx.lineWidth = 0.5;                    // FIX: the sample sets this on this.canvasContext
  ctx.strokeStyle = Color.Gray;

  let n = 100;
  let step = width / n;
  let start = points[0];
  for (let index = 0; start[0] <= points[3][0]; index++) {   // diagonal 1: dash, skip, dash
    ctx.beginPath();
    ctx.moveTo(start[0], start[1]);
    ctx.lineTo(start[0] + step, start[1] + step);
    start[0] += step * 2;
    start[1] += step * 2;
    ctx.stroke();
  }
  // ... diagonal 2 mirrored, then:
  step = step * Math.sqrt(2);             // axis-aligned dashes match the 45-degree ones
  // ... vertical down the middle, horizontal across it
};
```

**`Color.Gray` is a valid `strokeStyle`.** The attribute accepts
`string | number | CanvasGradient | CanvasPattern`, and the `Color` enum's
members *are* their hex values - `Gray` is `0x808080` - so passing the enum
sets a real grey rather than an ordinal.

**The corner inset uses `r - sqrt(r*r/2)`**, which is the distance from the
corner of a square to the point where a circle of radius `r` meets the
diagonal. It keeps the lattice from running under the canvas's 20-unit
`borderRadius`.

**Exporting a page — same file** (corrected, see `HW-08-0015` and `HW-08-0016`)

```typescript
import { fileIo } from '@kit.CoreFileKit';
import { buffer } from '@kit.ArkTS';

savePicture(img: string, n: number) {
  // FIX: the sample names the file .jpeg while toDataURL() defaults to image/png,
  // and has no try/finally, so a failed write leaks the descriptor.
  const imgPath = this.context.tempDir + '/' + Date.now() + '.png';
  const base64Image = img.split(';base64,').pop();
  if (base64Image === undefined) {
    return;
  }
  const file = fileIo.openSync(imgPath, fileIo.OpenMode.CREATE | fileIo.OpenMode.READ_WRITE);
  try {
    const imgBuffer = buffer.from(base64Image, 'base64');
    fileIo.writeSync(file.fd, imgBuffer.buffer);
    this.imgUrl[n] = 'file://' + imgPath;
  } finally {
    fileIo.closeSync(file);
  }
}
```

**`split(';base64,').pop()` is how a data URL is stripped.** `toDataURL`
returns `data:image/png;base64,<payload>`; only the payload is decodable, and
`buffer.from(_, 'base64')` turns it into bytes whose `.buffer` is what
`writeSync` wants.

**`tempDir` is the right directory for this.** The images exist to be shown on
the very next screen; they are not the user's documents and should not
outlive the session in `filesDir`.

**The results screen — `entry/src/main/ets/pages/Finish.ets`** (corrected, see `HW-08-0019`)

```typescript
// FIX: the sample decorates this @Entry and registers pages/Finish in
// main_pages.json, although it is only ever built as a destination.
@Component
export struct Finish {
  @Consume('pageInfos') pageInfos: NavPathStack;
  private imgUrl: Array<string> = [];

  build() {
    NavDestination() {
      Column() { /* the panda, five ImageItems, a next button */ }
    }
    .onReady((context: NavDestinationContext) => {
      this.pageInfos = context.pathStack;
      this.imgUrl = context.pathInfo.param as Array<string>;   // fires before children build
    })
    .hideTitleBar(true)
    .hideToolBar(true);
  }
}
```

**A plain field is enough here, and that is not an accident.** `onReady` is
documented as "triggered when the `NavDestination` component is about to build
a child component", so the assignment lands before the five `ImageItem`
builders run. A field assigned in `onAppear` instead would need `@State`.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`. Everything is
written inside the app's own sandbox, so no file permission is needed either.

The ability is **pinned to landscape** and to light mode:

```json
{
  "name": "EntryAbility",
  "srcEntry": "./ets/entryability/EntryAbility.ets",
  "orientation": "landscape",
  ...
}
```

and `EntryAbility.onCreate` calls
`setColorMode(ConfigurationConstant.ColorMode.COLOR_MODE_LIGHT)`, which is why
the hardcoded `#FEFFF1`, `#FCF5DB` and `#88B35A` throughout the pages hold up
and why `resources/dark/element/color.json` is dead.

`onWindowStageCreate` also hides the navigation indicator with
`setSpecificSystemBarEnabled('navigationIndicator', false)` - reasonable for a
drawing surface, where a swipe-up gesture bar sits exactly where a hand rests.
The avoid-area measurements it takes alongside are never used
(`HW-08-0018`).

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **Landscape only, and laid out in percentages of a landscape window.** The
  83%/15% split, the 240-point guide character and the panda at 83% height are
  tuned for a tablet; `deviceTypes` also lists `phone`, where the same
  proportions leave the board small.
- **The five characters are fixed** - `['白', '日', '依', '山', '尽']`, the
  opening line of a Tang poem, hardcoded in the page. There is no lesson
  model, no progression and no way to supply a different set.
- **Going back discards work.** The previous arrow clears the canvas and
  redraws the guide; it does not reload the image saved for that character, so
  the child re-writes it.
- **Nothing is persisted past the session.** The files live in `tempDir` and
  the URL array is component state, so closing the app loses the set.
- **The pen is fixed**: `selectedWidth`, `selectedColor` and `modeIndex` are
  plain fields set once and never changed - there is no colour or thickness UI
  in this sample. `KIDS-07` is the one that adds it.
- `toDataURL` is documented as involving a time-consuming memory copy, and it
  runs with the synchronous file write on the UI thread in the arrow's
  `onClick`. At one call per character that is acceptable; at one per stroke
  it would not be.

## Pitfalls

- **`HW-08-0014` — the context is created with no `RenderingContextSettings`,
  so anti-aliasing is off.** The documented default is `false`, and every
  stroke example in the reference passes `true`. On a calligraphy board, whose
  entire output is 20-unit diagonal freehand strokes plus a 0.5-unit hairline
  lattice, this is the single most visible defect.
- **`HW-08-0017` — the handler always takes `touches[0]`,** without the
  emptiness check the reference explicitly requires and without reading
  `TouchObject.id`. A child resting a palm on a writing board is the normal
  case, and the stroke jumps to it.
- **`HW-08-0015` — `savePicture` closes the file only on success.** No `try`,
  no `finally`, no `catch`; a failed write leaks the descriptor and throws out
  of the click handler silently.
- **`HW-08-0018` — the ability keeps a permanent `avoidAreaChange`
  subscription feeding two `AppStorage` keys nothing reads,** and never
  releases it. Both pages do draw under the system bars and pad for neither.
- **`HW-08-0019` — `Finish` is decorated `@Entry` and listed in
  `main_pages.json`** while being built as a `NavDestination`. Launched as a
  page its `@Consume('pageInfos')` would have no provider.
- **`HW-08-0016` — the file is named `.jpeg` and contains PNG,** because
  `toDataURL()` was called with no arguments.
- **`HW-08-0020` — seven of the eleven resource strings are `palette_*`
  leftovers from another sample,** referenced nowhere, while this sample's
  clear button is a hardcoded literal.
- **`HW-08-0021` — `drawLine` takes a `ctx` parameter but sets the pen on
  `this.canvasContext`.** Invisible only because every caller passes the same
  object.
- **`HW-08-0022` — the document's step 3 shows `.ontouch()`,** which is
  neither the correct name nor a call that takes the handler doing the work,
  and links `clearRect` into the atomic service documentation set.

## References

- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - the `Canvas` component and `onReady`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-canvasrenderingcontext2d.md` - `RenderingContextSettings`, `strokeStyle`, `clearRect`, `toDataURL`, `width`/`height`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-canvasrenderingcontext2d
- `documentation/harmonyos-references/02_application-framework/ts-universal-events-touch.md` - `TouchEvent`, `TouchObject.id`, and the emptiness check on `touches`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-universal-events-touch
- `documentation/harmonyos-references/02_application-framework/ts-appendix-enums.md` - the `Color` enum and its hex values
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-appendix-enums
- `documentation/harmonyos-references/02_application-framework/ts-basic-components-navdestination.md` - `onReady` and its ordering against the child build
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-components-navdestination
- `documentation/harmonyos-guides/03_application-framework/arkts-provide-and-consume.md` - the provider requirement behind `HW-08-0019`
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-provide-and-consume
- `documentation/harmonyos-references/02_application-framework/js-apis-file-fs.md` - `openSync`, `writeSync`, `closeSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-file-fs
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `on`/`off('avoidAreaChange')`, `setSpecificSystemBarEnabled`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `KIDS-07` - the drawing board that adds colour and thickness pickers to this same loop
- `KIDS-09` - fixed shapes drawn through the same `onTouch` handler
- `KIDS-05` - the Go board, the other generated-lattice Canvas sample
