---
id: UTIL-12
title: In-app floating tool ball - a draggable sub-window that snaps to an edge and fans out its tools
industry: 15_utilities
doc: huawei_industry_tree/15_utilities/docs/12_app_float_tool_ball.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_float_tool_ball-0000002322067001
sample: huawei_industry_tree/15_utilities/downloads/ToolBall.zip
kits: ["@kit.AbilityKit", "@kit.ArkUI", "@kit.BasicServicesKit", "@kit.CoreFileKit", "@kit.PerformanceAnalysisKit"]
apis: [common, display, hilog, window]
min_api: 20
modules: [entry (entry)]
findings: [HW-15-0002, HW-15-0034, HW-15-0035, HW-15-0101]
status: verified-with-fixes
---

## When to use

**Load this card when a control must float above every page of the app and
survive navigation** - an assistive-touch ball, a recording or call-in-progress
pill, a customer-service launcher, an accessibility shortcut, a debug overlay.

The technique is the load-bearing part: the ball is not a component in the
page tree and not an overlay on the `Navigation`; it is a **separate window**,
created with `windowStage.createSubWindow`, sized to 48 vp, loaded with its
own `@Entry` page, and moved with `moveWindowTo`. Because it is a window, no
page transition, `Navigation` push or `Tabs` switch can hide or clip it, and
it can be dragged past the main window's safe area.

Everything else follows from that choice, including the awkward parts. The
sub-window has its own component tree, so state is shared through `AppStorage`
(the main page publishes its `Scroller` there and the ball's tools call
`scrollToIndex` on it). And the sub-window takes focus when touched, so it has
to hand focus back explicitly with `window.shiftAppWindowFocus`.

**Read `HW-15-0034` before using the drag code at all** - as shipped the ball
races away from the finger on every drag.

## Feature checklist

- The main page is a waterfall gallery with a title; the tool ball appears
  over it on launch, near the bottom-right corner.
- The ball can be dragged anywhere on screen with one finger.
- On release it snaps to the nearer of the left or right edge.
- Dragging is clamped vertically to between the status bar and the navigation
  bar.
- Tapping the ball expands the window into a taller panel showing five tool
  icons fanned around the ball's centre.
- The fan opens to the right when the ball is on the left edge, and to the
  left when it is on the right edge; the panel's corner radii mirror the same
  way.
- The expanded panel collapses itself after 2 s of inactivity, and collapses
  immediately if a drag starts.
- Tool one scrolls the gallery to the top, tool five to the end, tool three
  exits the app; the rest raise a 自定义 (custom) toast.
- After every interaction, focus returns to the main window so the gallery
  stays scrollable.

## Architecture

One `entry` module, two `@Entry` pages - one per window.

```
entry/src/main/ets
├── common/CommonConstants.ets        sizes, the sub-window name, delays, initial position
├── entryability/EntryAbility.ets     publishes windowStage + avoid areas to AppStorage
├── entrybackupability/EntryBackupAbility.ets
├── pages/Index.ets                   @Entry (main window): gallery + createSubWindow
├── subwindow/ToolBallSubWindow.ets   @Entry (sub-window): the ball, the fan, the gestures
├── utils/WindowUtils.ets             ChangeFocus / edgeDetermination / dragToMove / resize
└── viewmodel
   ├── CustomToolModel.ets            the five tools: icon, offset from centre, click handler
   └── ImageDetailModel.ets           gallery data
```

The documented 工程目录 matches the zip.

**The design decision worth copying** is that the window is resized rather
than the content re-laid-out. Collapsed, the sub-window is 48x48 vp and holds
one image. Expanded, `switchFloatingWindowStyle` calls
`resizeAsync(84, 128)` (in px, via `vp2px`) and the same page renders the fan
instead. The alternative - one big transparent window that is mostly
click-through - would swallow touches over the whole screen; a window that is
exactly as large as its visible content never does.

The consequence to plan for is that **resizing moves the anchor**. A window
grows from its top-left corner, so a ball parked on the right edge would grow
off-screen; `switchFloatingWindowStyleMove` handles that by re-snapping
`x = displayWidth - windowRect.width` after the resize.

The second decision worth copying is `AppStorage` as the cross-window
channel. `Index.aboutToAppear` publishes its `Scroller` and the gallery
length; `ToolBallSubWindow` reads them back and the tool handlers drive the
main window's list directly:

```typescript
AppStorage.setOrCreate('scroller', this.scroller);
// ... in the sub-window:
this.scroller = AppStorage.get('scroller') as Scroller;
this.toolList[0].clickRes_ = () => { this.scroller.scrollToIndex(0); };
```

`AppStorage` is process-wide, so both windows see the same object. There is
no serialisation, which is what makes passing a live `Scroller` work.

## Implementation steps

1. **Publish the `WindowStage` in `onWindowStageCreate`** with
   `AppStorage.setOrCreate('windowStage', windowStage)` - the page cannot
   reach it otherwise.
2. **Create the sub-window from the page**:
   `windowStage.createSubWindow(name, (err, windowClass) => ...)`, check
   `err.code > 0` first.
3. **Set content, position, size, then show**, in that order.
   `setUIContent('subwindow/ToolBallSubWindow')` takes a page path from the
   module's `main_pages` profile.
4. **Derive the initial position from the display**, not from constants
   (`HW-15-0035`): `display.getDefaultDisplaySync()` minus the window size and
   the navigation-bar inset. The snapping code already does this arithmetic.
5. **`moveWindowTo` and `resize` take px**, so convert every vp constant with
   `getUIContext().vp2px(...)` at the call site.
6. **Attach a `PanGesture`** to the collapsed ball with
   `PanDirection.All`.
7. **Capture the window position in `onActionStart`** and treat
   `event.offsetX/Y` as a total since gesture start, not as a per-frame delta,
   converting vp to px (`HW-15-0034`). This is the single most important line
   in the sample.
8. **Snap on `onActionEnd`** with `edgeDetermination`, and use its return
   value to decide which way the fan opens.
9. **Shift focus back to the main window** after every interaction with
   `window.shiftAppWindowFocus(subWindowId, mainWindowId)`, on a short delay.
10. **Debounce the collapse** with a single module-level timer so a second tap
    restarts the 2 s countdown instead of stacking timers.

## Verified snippets

All snippets are from `ToolBall.zip`. Corrected forms are marked.

**Creating the sub-window — `entry/src/main/ets/pages/Index.ets`** (corrected, see `HW-15-0035`)

```typescript
private windowStage: window.WindowStage = AppStorage.get('windowStage') as window.WindowStage;

// 加载工具球子窗口
showToolBall() {
  const ctx = this.getUIContext();
  const ballPx = ctx.vp2px(CommonConstants.INIT_SUBWINDOW_SIZE);
  const disp = display.getDefaultDisplaySync();
  // FIX: sample uses the hardcoded CommonConstants.INIT_SUBWINDOW_POSITION_X / _Y (1100, 2300)
  const initX = disp.width - ballPx;
  const initY = disp.height - ballPx - (AppStorage.get('bottomRectHeight') as number);

  this.windowStage.createSubWindow(CommonConstants.ToolBall_SUBWINDOW, (err, windowClass) => {
    if (err.code > 0) {
      hilog.error(0x0000, 'testTag', '%{public}s', `failed to create subWindow Cause: ${err.message}`);
      return;
    }
    try {
      // 设置子窗口加载页
      windowClass.setUIContent('subwindow/ToolBallSubWindow', () => {
        windowClass.setWindowBackgroundColor(CommonConstants.WINDOW_BACKGROUND_COLOR);   // '#00000000'
      });
      // 设置子窗口坐标
      windowClass.moveWindowTo(initX, initY);
      // 设置子窗口大小
      windowClass.resize(ballPx, ballPx);
      // 展示子窗口
      windowClass.showWindow();
    } catch (err) {
      hilog.error(0x0000, 'testTag', '%{public}s', `failed to create subWindow Cause:${err}`);
    }
  });
}
```

**Three calls carry the design.** `setWindowBackgroundColor('#00000000')`
inside the `setUIContent` callback is what makes the window transparent - the
rounded white ball is drawn by the page, and everything around it inside the
48x48 box must be see-through. `moveWindowTo` and `resize` are both in **px**,
which is why `vp2px` appears on the size but must also be applied to any
offset that comes from a gesture. And `showWindow()` last: a sub-window shown
before its content and geometry are set flashes at the wrong place.

The initial position is the finding. `INIT_SUBWINDOW_POSITION_X = 1100` and
`INIT_SUBWINDOW_POSITION_Y = 2300` are px coordinates chosen for one large
phone; the module declares `wearable` among its `deviceTypes`, where the ball
is created entirely off-screen and the user never sees it (`HW-15-0035`).
Deriving from `display.getDefaultDisplaySync()` costs two lines and the file
already imports `display` for exactly this arithmetic in `edgeDetermination`.

**The drag — `entry/src/main/ets/utils/WindowUtils.ets`** (corrected, see `HW-15-0034`)

```typescript
import { display, window } from '@kit.ArkUI';

export interface Position { x: number, y: number }

// FIX: the shipped signature has no dragStart; the caller now captures it in onActionStart
export function dragToMove(event: GestureEvent, window: window.Window, windowPosition: Position,
  context: UIContext, dragStart: Position): void {
  let bottomRectHeight = AppStorage.get('bottomRectHeight') as number;
  let topRectHeight = AppStorage.get('topRectHeight') as number;
  let bottomY =
    display.getDefaultDisplaySync().height - bottomRectHeight - context.vp2px(CommonConstants.INIT_SUBWINDOW_SIZE);

  // FIX: offsetX/offsetY are cumulative since gesture start and are in vp;
  //      the sample does `windowPosition.x += event.offsetX` against a px position.
  windowPosition.x = dragStart.x + context.vp2px(event.offsetX);
  const nextY = dragStart.y + context.vp2px(event.offsetY);
  windowPosition.y = Math.min(Math.max(nextY, topRectHeight), bottomY);

  window.moveWindowTo(windowPosition.x, windowPosition.y);
}
```

**Two unit errors compound in one statement.** `GestureEvent.offsetX` on a
`PanGesture` is the offset **since the gesture began**, re-reported on every
`onActionUpdate` frame, and it is expressed in **vp**. The shipped code adds
it to the position on every frame, so a straight 100 vp drag delivers roughly
`10 + 20 + 30 + ...` px of movement - the ball accelerates away from the
finger and lands far past it. The `vp2px` omission then multiplies the whole
error by the screen density, 3x on a typical phone. The same function
computes `bottomY` with a correct `context.vp2px(...)` three lines above,
which is what makes the omission diagnosable rather than ambiguous.

The clamp is the second change. The shipped code has three `if/else if`
branches that only permit movement when the position is already inside the
band or heading back into it; with a start-relative position a single
`Math.min`/`Math.max` expresses the same intent and cannot get stuck at a
boundary.

**Snapping and the fan direction — same file** (as shipped)

```typescript
export function edgeDetermination(window: window.Window, windowPosition: Position): Direction {
  let halfWidth = display.getDefaultDisplaySync().width * CommonConstants.HALF;
  if (windowPosition.x < halfWidth) {
    windowPosition.x = 0;
  } else {
    windowPosition.x = display.getDefaultDisplaySync().width - window.getWindowProperties().windowRect.width;
  }
  window.moveWindowTo(windowPosition.x, windowPosition.y);
  return windowPosition.x === 0 ? Direction.RIGHT : Direction.LEFT;
}

export async function switchFloatingWindowStyle(isExpand: boolean, window: window.Window, context: UIContext) {
  let width = 0, height = 0;
  if (isExpand) {
    width = context.vp2px(CommonConstants.EXPANDED_SUBWINDOW_WIDTH);
    height = context.vp2px(CommonConstants.EXPANDED_SUBWINDOW_HEIGHT);
  } else {
    width = context.vp2px(CommonConstants.INIT_SUBWINDOW_SIZE);
    height = context.vp2px(CommonConstants.INIT_SUBWINDOW_SIZE);
  }
  await window.resizeAsync(width, height);
}
```

**The return value is the whole fan logic.** `edgeDetermination` snaps and
then answers "which way is there room" - `Direction.RIGHT` when the ball
landed on the left edge. The sub-window stores that in `this.dir` and uses it
three times: to mirror each tool's x offset
(`getFloatingWindowOffset` negates `x` when `dir === RIGHT`), to choose which
two corners of the panel get the 60 vp radius and which get 16 vp, and
implicitly to keep the expanded panel on screen.

`resizeAsync` rather than `resize` matters here because the caller awaits it
before snapping again; the synchronous `resize` used at creation time would
leave `getWindowProperties().windowRect.width` stale for the follow-up
`edgeDetermination`.

**Gesture wiring and the focus hand-back — `entry/src/main/ets/subwindow/ToolBallSubWindow.ets`** (corrected, see `HW-15-0034`)

```typescript
private dragStart: Position = { x: 0, y: 0 };   // FIX: added for the corrected dragToMove

.onClick(async () => {
  this.isExpand = true;
  await switchFloatingWindowStyle(this.isExpand, this.subWindow, this.getUIContext());
  edgeDetermination(this.subWindow, this.windowPosition);        // re-snap after growing
  this.debounce(() => {                                          // 倒计时恢复非扩展态
    this.isExpand = false;
    switchFloatingWindowStyleMove(this.isExpand, this.subWindow, this.windowPosition, this.getUIContext());
  }, CommonConstants.EXPAND_DELAY);                              // 2000 ms
  ChangeFocus(this.windowStage, CommonConstants.ToolBall_SUBWINDOW);
})
.gesture(
  PanGesture(this.panOption)
    .onActionStart(async () => {
      this.dragStart = { x: this.windowPosition.x, y: this.windowPosition.y };   // FIX
      if (debounceTimer !== null) {
        clearTimeout(debounceTimer);
        this.isExpand = false;
        await switchFloatingWindowStyle(this.isExpand, this.subWindow, this.getUIContext());
      }
    })
    .onActionUpdate((event: GestureEvent) => {
      dragToMove(event, this.subWindow, this.windowPosition, this.getUIContext(), this.dragStart);
    })
    .onActionEnd(() => {
      // 吸顶后，判断展开方向
      this.dir = edgeDetermination(this.subWindow, this.windowPosition);
      ChangeFocus(this.windowStage, CommonConstants.ToolBall_SUBWINDOW);
    })
);
```

**`ChangeFocus` is the piece that is easy to omit and impossible to work
around later.** Touching the sub-window gives it focus, and a focused
sub-window means the gallery underneath stops receiving scroll gestures. The
utility looks up both window ids and calls
`window.shiftAppWindowFocus(subWindowID, mainWindowID)` after a 500 ms delay -
long enough for the tap to be consumed by the ball, short enough that the user
does not notice.

`onActionStart` doubles as the collapse trigger: starting a drag on an
expanded panel clears the pending timer and shrinks the window first, so the
drag always moves a 48x48 box. Note `debounceTimer` is a module-level
`number` compared against `null` while `setTimeout` never returns `null` -
harmless here because `clearTimeout` on a stale id is a no-op, but the guard
does nothing.

## Permissions & config

**None.** The sample declares no `requestPermissions`. This is an *in-app*
floating ball: a sub-window of the app's own `WindowStage`, so it disappears
when the app goes to the background. A ball that floats over *other* apps
would be a `WindowType.TYPE_FLOAT` system window and would need
`ohos.permission.SYSTEM_FLOAT_WINDOW` - a different, restricted feature.

`deviceTypes` is `phone`, `tablet`, `2in1`, `wearable`. The wearable entry is
what makes `HW-15-0035` a shipping bug rather than a tuning issue.
`main_pages` must list `subwindow/ToolBallSubWindow` or `setUIContent` fails
at runtime.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later.
- `moveWindowTo`, `resize` and `resizeAsync` are all in **px**; every vp
  constant crossing into them needs `vp2px`. `display.getDefaultDisplaySync()`
  also reports px.
- The sub-window is bound to the `WindowStage`. It vanishes with the app and
  cannot be shown over other applications.
- Nothing destroys the sub-window. `Index` creates it in `aboutToAppear` with
  no matching teardown, so re-entering the page after `terminateSelf` relies
  on the process having been destroyed.
- The ball intercepts touches over its 48x48 box; because it snaps to an edge
  it will sit on top of edge-anchored controls in the page beneath.
- The tool offsets in `CustomToolModel` are hand-tuned literals
  (`±20`, `MAXIMUM_DISTANCE/3`) fitted to the 84x128 vp expanded panel; the
  three sizes must be changed together.
- The main window's avoid areas are stored as raw px in `AppStorage` and read
  back with `px2vp` at the point of use - `dragToMove` compares them against
  px window positions, which is consistent, but mixing the convention with a
  sibling sample that stores vp will silently misplace the clamp.

## Pitfalls

- **`HW-15-0034`** (B/high, confirmed): `dragToMove` does
  `windowPosition.x += event.offsetX` on every `onActionUpdate`, but pan
  offsets are cumulative since the gesture started, giving roughly quadratic
  overshoot - and the offsets are vp added straight into a px position, a
  further 3x error on a density-3 screen. The ball races far ahead of the
  finger. Fix: capture the position in `onActionStart` and compute
  `position = start + vp2px(offset)`.
- **`HW-15-0035`** (B/medium, confirmed): the initial position is hardcoded at
  `(1100, 2300)` px in `CommonConstants`, so on any display smaller than about
  1148x2348 px - including the `wearable` device type the module declares -
  the ball is created fully off-screen. Fix: derive it from
  `display.getDefaultDisplaySync()` minus the window size, exactly as
  `edgeDetermination` already does.
- **`HW-15-0002`** (E/low, probable): the doc's 场景介绍 links its core
  mechanism to `harmonyos-guides/subwindow-guide`, a slug that appears
  nowhere in the crawled guides or references navigation trees (9428 slugs);
  live fetches return the SPA shell and are inconclusive. Fix: point it at the
  current sub-window development guide - locally that content lives in
  `harmonyos-guides/03_application-framework/application-window-stage.md`.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` - `createSubWindow`, `getMainWindowSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setUIContent`, `moveWindowTo`, `resize` / `resizeAsync`, `setWindowBackgroundColor`, `getWindowProperties`, `showWindow`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/application-window-stage.md` - creating and managing a sub-window (the guide `HW-15-0002`'s dead link should point to)
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/application-window-stage
- `documentation/harmonyos-guides/03_application-framework/window-overview.md` - window types and what an in-app sub-window can and cannot do
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/window-overview
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` - `getDefaultDisplaySync`, `width` / `height` in px
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `documentation/harmonyos-references/02_application-framework/ts-basic-gestures-pangesture.md` - `PanGesture`, `PanGestureOptions`, and the meaning of `GestureEvent.offsetX` / `offsetY`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/ts-basic-gestures-pangesture
- `huawei_industry_tree/15_utilities/docs/12_app_float_tool_ball.md` - the source page
  https://developer.huawei.com/consumer/cn/doc/architecture-guides/app_float_tool_ball-0000002322067001
