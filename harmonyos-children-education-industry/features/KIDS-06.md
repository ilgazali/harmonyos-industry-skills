---
id: KIDS-06
title: Tangram puzzle - drag and rotate pieces that snap to a target silhouette
industry: 08_children_education
doc: huawei_industry_tree/08_children_education/docs/06_jigsaw_puzzle.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/jigsaw_puzzle-0000002317714198
sample: huawei_industry_tree/08_children_education/downloads/JigsawPuzzle.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.PerformanceAnalysisKit", "@kit.BasicServicesKit"]
apis: [Path, commands, PanGesture, onActionStart, onActionUpdate, onActionEnd, GestureEvent, "UIContext.animateTo", rotate, position, zIndex, Stack, Grid, GridItem, "@CustomDialog", CustomDialogController, DismissReason, "window.getLastWindow", getWindowProperties, safeAreaPadding, "@StorageProp", "@Prop", "@Observed"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-08-0043, HW-08-0044, HW-08-0045, HW-08-0046, HW-08-0047, HW-08-0048, HW-08-0049, HW-08-0050, HW-08-0051, HW-08-0052, HW-08-0120]
status: verified-with-fixes
---

## When to use

Load this card for **a drag-and-drop puzzle whose pieces must land in the right
place at the right angle** - tangram, jigsaw, shape sorting, a seating plan, any
"arrange these into that" exercise.

Two ideas here are genuinely good and are not obvious:

- **Shapes as `Path` commands, not images.** The seven tangram pieces are seven
  SVG path strings assembled from named constants. No assets, no resolution
  ceiling, and a new piece is a new string.
- **Multiple solutions checked by set intersection.** Each piece carries *four*
  target positions - one per valid arrangement - and the puzzle is solved when
  every piece is placed **and** there is a target index common to all seven.
  That is what `findCommonElementInAllSubArrays` does, and it is the cleverest
  thing in the industry so far.

**Ten findings, and the two worst are structural.** Every piece wraps its
`Path` in a `Canvas` whose reference says it accepts no children, and the
draggable box is 20x100 while the shape drawn in it is up to 300x300 - so most
of a visible piece cannot be picked up. Take the model from this card; do not
take the layout.

## Feature checklist

- Seven coloured tangram pieces laid out in a tray.
- A grey silhouette of the target figure, and grey outlines at the start
  positions.
- Dragging moves a piece; tapping rotates it 45 degrees.
- Releasing near a correct target at the correct angle snaps the piece home.
- When all seven are placed in one consistent arrangement, a win dialog opens.

## Architecture

One `entry` module, one page, one piece component, one model, one constants
file.

```
entry/src/main/ets
├── constants/CalendarConstants.ets   the misnamed constants file (HW-08-0052)
│     CanvasConstants          path-command fragments, lengths x1.5
│     GeneratePiecesConstants  the seven shape names
│     PiecesConstants          TARGET_X/Y/ANGLE_ITEM, OFFSET_X/Y_ITEM
├── entryability/EntryAbility.ets
├── entrybackupability/
├── model/PuzzlePiece.ets             PuzzlePieceItem
└── pages
    ├── TanGramGame.ets               the board, the gestures, the win check
    └── TanGramPiece.ets              shape dispatch, five @Builder shapes
```

The documented tree matches the zip - including the calendar filename.

**Three stacked layers, drawn back to front by `zIndex`:**

| zIndex | Layer | Colour |
| --- | --- | --- |
| 0 | outlines at the pieces' start positions | `rgba(0, 0, 0, 0.2)` |
| 1 | the target silhouette to fill | `rgba(0, 0, 0, 0.2)` |
| 2 | the seven draggable pieces | the real colours |

All three iterate the same `PuzzlePiece` array and differ only in colour, in
which coordinate array they read, and in `zIndex`. That is a tidy way to build
a puzzle: one piece list, three views of it.

**The solution model is the part worth stealing.** Each piece has four
candidate targets:

```
piece 2 (square)   TARGET_X = [178.42, 178.42, 178.42, 178.42]
                   TARGET_Y = [371.06, 371.06, 371.06, 371.06]
                   ANGLE    = [     0,      0,      0,      0]

piece 0 (triangle) TARGET_X = [224.68, 104.20, 224.68, 104.20]
                   ANGLE    = [     0,      6,      0,      6]
```

Column *k* is arrangement *k*. A symmetric piece matches all four columns; an
asymmetric one matches only some. `times[pieceId]` collects the column indices
that matched, and the win test asks whether one column index appears in **every**
piece's list - i.e. whether the seven pieces are all solving the *same*
arrangement rather than each solving a different one.

## Implementation steps

1. **Describe each shape as a `Path`** built from shared command constants.
2. **Size the container to the path** so the touch target matches the shape
   (`HW-08-0044`) - and do not wrap it in a `Canvas` (`HW-08-0043`).
3. **Position the pieces with `Stack` + `position`**, not a `Grid` whose
   layout you then override (`HW-08-0048`).
4. **Rotate on tap inside `animateTo`**, in 45-degree steps modulo 8.
5. **Drag with `PanGesture`**, keeping the gesture delta and the stored
   coordinate in one space (`HW-08-0046`).
6. **On release, test distance and angle** against every candidate target and
   record which ones matched.
7. **Declare victory only on a common arrangement index**, not on seven
   independent placements.

## Verified snippets

All snippets are from `JigsawPuzzle.zip`. Corrected forms are marked.

**A shape — `entry/src/main/ets/pages/TanGramPiece.ets`** (corrected, see `HW-08-0043` and `HW-08-0044`)

```typescript
// FIX: the sample wraps this Path in `Canvas() { ... }.width(20).height(100)`.
// The Canvas reference states, under Child Components: "Not supported" - and
// the Canvas is given no context and no onReady, so it draws nothing.
@Builder
TriangleShape(color: string, angle: number, strokeWidth: number) {
  Path()
    .commands(`M0 0 L150 150 L0 150 Z`)   // PIECES_LENGTH_100 is String(100 * 1.5)
    .strokeWidth(strokeWidth)
    .stroke($r('app.color.black'))
    .fill(color)
    .rotate({ z: 1, angle: angle })
    .width(150)                           // FIX: sample says 20 x 100
    .height(150);
}
```

**`Path` renders itself.** It is a drawing *component* with `commands`,
`stroke` and `fill` attributes - the declarative counterpart to
`CanvasRenderingContext2D`. It needs no host, and `Canvas` explicitly takes no
children. Compare `KIDS-05`, which uses `Canvas` the intended way: one context,
`onReady`, imperative calls.

**Five builders cover seven pieces**, because two pairs differ only by
rotation - `TriangleShape(color, 0, w)` and `TriangleShape(color, 270, w)`,
`CanvasComponent(color, 0, w)` and `CanvasComponent(color, 180, w)`. Passing
the angle as a builder parameter rather than writing four path strings is the
right economy.

**The lengths carry a `* 1.5` scale factor in the constant itself** -
`static PIECES_LENGTH_100 = String(100 * 1.5);` - so the design proportions
stay readable while the drawn size changes in one place.

**Rotating on tap — `entry/src/main/ets/pages/TanGramGame.ets`** (as shipped)

```typescript
TanGramPiece({ color: item.color, shape: item.shape, pieceId: item.id, ... })
  .onClick(() => {
    this.uiContext.animateTo({
      duration: 100,
      curve: Curve.Linear,
      iterations: 1,
      playMode: PlayMode.Normal,
      expectedFrameRateRange: { min: 0, max: 3000, expected: 60 }
    }, () => {
      this.rotateAngleTimes[index] = this.rotateAngleTimes[index] + 1;
      this.rotateAngle[index] = 45 * this.rotateAngleTimes[index];
    });
  })
  .rotate({ angle: this.rotateAngle[index] })
```

**Two parallel arrays, one for display and one for logic.** `rotateAngle`
accumulates without bound - 45, 90, ... 405, 450 - so the visual rotation keeps
turning the same way instead of snapping back to zero on the eighth tap.
`rotateAngleTimes` is the raw count, and the placement test takes it
`% ROTATE_ANGLE_TIMES` (8) to compare against the stored target angle. Keeping
the unbounded value for the animation and the modulo for the comparison is
exactly right; using the modulo for both would make every eighth tap rotate
backwards through 315 degrees.

**`expectedFrameRateRange` is a real hint, not decoration** - it tells the
render pipeline what this animation wants, which matters for a 100 ms step.

**Dragging — same file** (corrected, see `HW-08-0046`)

```typescript
PanGesture()
  .onActionStart(() => {
    // FIX: the sample stores the raw design-sheet value here, then multiplies
    // by scaleWidth at render time, so the piece moves scaleWidth x as far as
    // the finger. Convert the gesture delta instead, or store device units.
    this.currentX = this.offsetXItem[index];
    this.currentY = this.offsetYItem[index];
  })
  .onActionUpdate((info: GestureEvent) => {
    let offsetX = this.currentX + info.offsetX / this.scaleWidth;    // FIX
    let offsetY = this.currentY + info.offsetY / this.scaleHeight;   // FIX
    if (offsetY > -60 && offsetY < 640 && offsetX > -300 && offsetX < 300) {
      this.offsetXItem[index] = offsetX;
      this.offsetYItem[index] = offsetY;
    }
  })
  .onActionEnd(() => {
    this.checkPiecePlacement(item, this.offsetXItem[index], this.offsetYItem[index],
      this.rotateAngleTimes[index] % PiecesConstants.ROTATE_ANGLE_TIMES);
  })
```

**Capturing the start position in `onActionStart` is the correct pattern.**
`info.offsetX` is measured from where the drag began, so it must be added to a
snapshot taken at the start - not accumulated onto the live value, which would
square the movement.

**The bounds check runs before the assignment**, so a piece cannot be dragged
off the board. The four literals are in design-sheet units and are the reason
the drag region does not match the visible board on other screen sizes.

**Testing placement — same file** (as shipped)

```typescript
checkPiecePlacement(piece: PuzzlePieceItem, x: number, y: number, angle: number) {
  const TARGET_X = piece.targetX;
  const TARGET_Y = piece.targetY;
  this.times[piece.id] = [];                       // reset this piece's matches
  for (let times = 0; times < TARGET_X.length; times++) {
    const distance = Math.sqrt(Math.pow(x - TARGET_X[times], 2) + Math.pow(y - TARGET_Y[times], 2));
    if (distance < 30 && angle === piece.targetAngle[times]) {
      piece.isPlaced = true;
      this.times[piece.id].push(times);             // remember WHICH arrangement
      this.animatePieceToTarget(piece, TARGET_X[times], TARGET_Y[times]);
    } else {
      if (this.times[piece.id].length === 0) {
        piece.isPlaced = false;
      }
    }
  }
  this.checkAllPiecesPlaced();
}
```

**Distance *and* angle, both.** A tangram piece in the right place at the wrong
rotation is not placed, and the 30-unit radius is what makes the snap forgiving
enough for a child's aim.

**The loop deliberately does not `break`.** A symmetric piece matches several
arrangements at once, and all of those indices must be recorded - that is the
input to the win test below.

**`if (this.times[piece.id].length === 0)`** in the else branch is what stops a
later non-matching candidate from clearing a match found earlier in the same
loop. Reordering these two clauses would break the multi-arrangement logic.

**The win condition — same file** (as shipped)

```typescript
findCommonElementInAllSubArrays(arr: number[][]): boolean {
  if (arr.length === 0) { return false; }
  let commonElements = new Set(arr[0]);
  for (let i = 1; i < arr.length; i++) {
    const currentSet = new Set(arr[i]);
    commonElements = new Set((Array.from(commonElements)).filter(x => currentSet.has(x)));
    if (commonElements.size === 0) { return false; }   // early exit
  }
  return commonElements.size > 0;
}

checkAllPiecesPlaced() {
  let allPlaced = this.PuzzlePiece.every(p => p.isPlaced);
  if (allPlaced && this.findCommonElementInAllSubArrays(this.times)) {
    if (this.dialogController != null) {
      this.dialogController.open();
    }
  }
}
```

**`allPlaced` alone would be wrong**, and this is the insight the sample gets
right: seven pieces can each sit on a legal target while belonging to four
*different* arrangements, which is not a solved puzzle. Intersecting the
per-piece index sets is the cheap test for "one consistent arrangement", and it
generalises to any number of solutions by widening the target arrays.

**Sizing to the window — same file** (corrected, see `HW-08-0045`)

```typescript
aboutToAppear(): void {
  this.PuzzlePiece = this.generatePieces();
  window.getLastWindow(this.context).then((windowClass: window.Window) => {
    let windowProperties = windowClass.getWindowProperties();
    this.screenWidth = this.uiContext.px2vp(windowProperties.windowRect.width);
    this.screenHeight = this.uiContext.px2vp(windowProperties.windowRect.height);
    const standardWidth = 376;          // 设计稿标准宽度
    const standardHeight = 809.1428;
    this.scaleWidth = this.screenWidth / standardWidth;
    // FIX: the sample writes
    //   this.screenHeight / standardHeight * this.screenHeight / standardHeight
    this.scaleHeight = this.screenHeight / standardHeight;
  }).catch((error: Error) => {
    hilog.error(0x0000, '', 'Failed to obtain the window size. Cause: ' + JSON.stringify(error));
  });
}
```

**`window.getLastWindow(...).getWindowProperties().windowRect` is the right
source** - the *window* rectangle, not the display. This is what `KIDS-05`
should have used instead of `display.getAllDisplays`, and it is the one place
this sample is ahead of that one. The `.catch` is present too.

**The win dialog — same file** (as shipped)

```typescript
dialogController: CustomDialogController | null = new CustomDialogController({
  builder: CustomDialogExample({}),
  autoCancel: true,
  onWillDismiss: (dismissDialogAction: DismissDialogAction) => {
    if (dismissDialogAction.reason === DismissReason.PRESS_BACK) { dismissDialogAction.dismiss(); }
    if (dismissDialogAction.reason === DismissReason.TOUCH_OUTSIDE) { dismissDialogAction.dismiss(); }
  },
  alignment: DialogAlignment.Bottom,
  width: '50%',
  height: 36,
});

aboutToDisappear() {
  this.dialogController = null;      // 在自定义组件即将析构销毁时将dialogController置空
}
```

**Nulling the controller in `aboutToDisappear` is the documented discipline**
for `CustomDialogController` and is worth copying - the controller holds a
reference back to the component that built it. Contrast `KIDS-02`, which never
releases its controller.

`onWillDismiss` distinguishing `PRESS_BACK` from `TOUCH_OUTSIDE` is the hook
for refusing a dismissal; here both are allowed, but the shape is the one to
copy when a dialog must not be dismissed by one of the two.

## Permissions & config

**None.** `module.json5` declares no `requestPermissions`.

`main_pages.json` lists `pages/TanGramGame` alone; there is no navigation.

`EntryAbility` pins light mode, goes full screen with
`setWindowLayoutFullScreen(true)` (with `then`/`catch`, unlike `KIDS-05`),
reads both avoid areas into `AppStorage` and subscribes to `avoidAreaChange` -
which it never releases (`HW-08-0049`). The page binds both keys with
`@StorageProp` and applies only the top one:

```typescript
}.safeAreaPadding({ top: this.uiContext.px2vp(this.topRectHeight) })
```

`safeAreaPadding` rather than plain `padding` is the correct attribute here -
it pads within the safe area rather than adding to the component's own padding.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- **One puzzle, one figure.** The target silhouette, the four arrangements and
  the seven start positions are all literal arrays in `PiecesConstants`. A
  second figure means a second set of arrays; there is no level model.
- **Everything is authored against a 376 x 809.1428 design sheet** and scaled
  by two factors at render time. Every coordinate in the sample - targets,
  start positions, drag bounds - is in those units, and the two coordinate
  spaces are the source of `HW-08-0045` and `HW-08-0046`.
- **Rotation is 45-degree steps in one direction only.** Eight taps returns a
  piece to its start angle; there is no counter-rotation and no free rotation.
- **The parallelogram cannot be flipped**, only rotated - so the one tangram
  piece that genuinely needs reflection has no way to be mirrored.
- **No reset and no persistence.** Winning opens a dialog; dismissing it leaves
  the board solved, with no way to start again short of relaunching.
- Two empty `Column()`s with fixed 328 x 197 and 328 x 328 sizes act as
  background panels behind the board, so the backdrop does not adapt.

## Pitfalls

- **`HW-08-0043` — every piece nests a `Path` inside a `Canvas`,** whose
  reference states under Child Components: "Not supported". The `Canvas` gets
  no context and no `onReady`, so twenty-one empty drawing surfaces are
  allocated to wrap components that render themselves.
- **`HW-08-0044` — the draggable box is 20x100 and the shape inside it is up to
  300x300.** Hit testing follows the layout box, so a piece can only be grabbed
  from a narrow strip at its left edge - in a game whose entire interaction is
  picking shapes up.
- **`HW-08-0045` — `scaleHeight` is the height ratio squared.** The line above
  it computes the width factor correctly; the vertical one multiplies the ratio
  by itself, so the error grows with distance from the 809.1428 design height.
- **`HW-08-0046` — the drag delta is in device units and the stored coordinate
  is in design units,** and the sum is scaled again at render time, so the piece
  moves `scaleWidth` times as far as the finger.
- **`HW-08-0047` — the document's drag handler calls `getWidthScaledVp` and
  `getHeightScaledVp`, which do not exist in the sample,** and its
  `onActionUpdate` computes two locals it never assigns - as printed, the drag
  does nothing.
- **`HW-08-0048` — three `Grid`s declare a `columnsTemplate` and then position
  every child absolutely,** so the grid layout is computed and discarded; the
  comment above says five columns and the template says four.
- **`HW-08-0049` — the `avoidAreaChange` listener is never released,** and
  `bottomRectHeight` is bound but never applied, so the navigation indicator
  overlaps the lower part of the board.
- **`HW-08-0050` — `animatePieceToTarget` contains no animation,** despite its
  name and the document's claim that `animateTo` drives the snap. The piece
  jumps.
- **`HW-08-0051` — `TanGramPiece` declares six `@State` fields nothing reads,**
  seeded with 372.148/817.777 where the parent uses 376/809.1428; `isPlaced` is
  an unused `@Prop`, and `@Observed` on `PuzzlePieceItem` has no `@ObjectLink`
  to pair with.
- **`HW-08-0052` — the constants file is called `CalendarConstants.ets`** in a
  tangram sample, and the document's project tree repeats the name.

## References

- `documentation/harmonyos-references/02_application-framework/ts-drawing-components-path.md` - `Path` and `commands`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-drawing-components-path
- `documentation/harmonyos-references/02_application-framework/ts-components-canvas-canvas.md` - Child Components: "Not supported"
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-components-canvas-canvas
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture` and `GestureEvent.offsetX`/`offsetY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `documentation/harmonyos-references/02_application-framework/arkts-apis-uicontext-uicontext.md` - `animateTo` and `expectedFrameRateRange`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-uicontext-uicontext
- `documentation/harmonyos-references/02_application-framework/ts-methods-custom-dialog-box.md` - `CustomDialogController`, `onWillDismiss`, `DismissReason`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-methods-custom-dialog-box
- `documentation/harmonyos-references/02_application-framework/ts-container-grid.md` - the container whose layout this sample overrides
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-container-grid
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `getLastWindow`, `getWindowProperties`, `off('avoidAreaChange')`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/arkts-observed-and-objectlink.md` - the pairing `@Observed` needs
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-observed-and-objectlink
- `KIDS-05` - the other board sample, which uses `Canvas` correctly but sizes from the display instead of the window
- `KIDS-10` - the sliding picture puzzle, the industry's other piece-arrangement game
- `SPORT-13` - dragging to reorder with a sequential `GestureGroup`, the pattern this sample would need for drag-versus-scroll
