---
id: MEDIA-07
title: Floating mini-player - a WindowStage sub-window that drags, snaps to the edge and controls an AVPlayer
industry: 13_media_entertainment
doc: huawei_industry_tree/13_media_entertainment/docs/07_sub_window.md
doc_url: https://developer.huawei.com/consumer/cn/doc/architecture-guides/sub_window-0000002254941572
sample: huawei_industry_tree/13_media_entertainment/downloads/SubWindow.zip
kits: ["@kit.ArkUI", "@kit.AbilityKit", "@kit.MediaKit", "@kit.AudioKit", "@kit.BasicServicesKit", "@kit.ArkTS", "@kit.PerformanceAnalysisKit"]
apis: ["windowStage.createSubWindow", "window.findWindow", setUIContent, showWindow, moveWindowTo, resize, resizeAsync, setWindowBackgroundColor, getWindowProperties, "window.shiftAppWindowFocus", "display.getDefaultDisplaySync", "UIContext.vp2px", PanGesture, PanGestureOptions, onActionStart, onActionUpdate, onActionEnd, "media.createAVPlayer", fdSrc, "resourceManager.getRawFd", audioRendererInfo, prepare, play, pause, release, "@StorageLink", "@StorageProp", "context.eventHub"]
permissions: []
min_api: 20
modules: [entry]
findings: [HW-13-0022, HW-13-0023, HW-13-0024, HW-13-0025]
status: verified-with-fixes
---

## When to use

Load this card when playback has to **survive navigation inside the app** -
the little circle that hovers over every page while a track plays, that you can
drag to a corner and tap to play or pause. Music and podcast apps, video
mini-players, and anything else needing a persistent in-app control surface: a
call bar, a recording indicator, a live score chip.

The load-bearing choice is that this is **a real window, not an overlay
component**. `windowStage.createSubWindow` gives it its own `@Entry` page, its
own UI instance and its own z-order above the main window, so no page in the
host app has to know it exists and nothing has to be re-mounted on navigation.
The price is that everything crossing the boundary - the song, the play state,
the foreground event - has to go through `AppStorage` or the ability's
`eventHub`, because the two windows do not share component state.

Sub-window, drag, edge snap and player are four independent pieces here, and
three of them ship broken. **Read `HW-13-0022`, `HW-13-0023` and `HW-13-0024`
before adopting any of it**: the drag overshoots the finger, a double tap
creates a second `AVPlayer`, and selecting a second song never changes the
audio.

## Feature checklist

- A five-tab music app whose 我喜欢 (my favourites) tab lists nine tracks.
- Tapping a track opens a small round floating window that stays on top while
  the user moves between tabs.
- Dragging it moves it with the finger; releasing snaps it to the nearer screen
  edge, and it cannot be dragged under the status bar or the navigation
  indicator.
- Tapping it expands it sideways into a pill showing the title and artist;
  tapping the expanded pill toggles play/pause.
- The pill collapses back to a circle two seconds after the last interaction
  and re-snaps to the edge; dragging while expanded collapses it first.
- Focus returns to the main window after every interaction, so the app below
  stays usable.
- Bringing the app back to the foreground re-snaps the window to the edge.

## Architecture

One `entry` module. The sub-window's page lives in its own directory because it
is a second entry point, not a component of the first.

```
entry/src/main/ets
├── components/MyLike.ets          the favourites list + showSubWindow() (createSubWindow lives here)
├── components/TabContent.ets      BottomNavigationModel + the five tab descriptors
├── constants/Constants.ets        window name, sizes, delays, status codes
├── entryability/EntryAbility.ets  uiContext/windowStage/avoid areas -> AppStorage; onForeground -> eventHub
├── model/SongItem.ets             id/title/singer/src/index
├── model/SongListData.ets         SONG_LIST, nine static entries over three .wav files
├── pages/HomePage.ets             @Entry, the five bottom tabs
├── subwindow/MusicSubWindow.ets   @Entry of the SUB-window: the circle, the pill, the gestures
├── utils/MediaService.ets         singleton AVPlayer wrapper
└── utils/WindowUtils.ets          ChangeFocus, edgeDetermination, dragToMove, style switching
```

The documented tree matches the zip exactly, and the document's constraints
(API 20, HarmonyOS 6.0.0, DevEco 6.0.0) match `compatibleSdkVersion: "6.0.0(20)"` -
neither `HW-13-0003` (tree naming) nor `HW-13-0004` (overstated API level)
applies to this sample.

**The design decision worth copying** is the split between `WindowUtils` and
`MusicSubWindow`. Every window operation - focus shifting, edge snapping,
dragging, resizing - is a free function taking `(window, position)` and holding
no state; the component holds the state (`windowPosition`, `isExpand`) and
decides *when*. That is what makes the same three functions reusable from the
pan gesture, from the click handler and from the `onForeground` event
subscription without any of them knowing about each other.

**The part worth avoiding** is how the two windows communicate. Title, artist,
the raw-file name and the play status are four independent `AppStorage` keys
written by the list and read by the sub-window, with no object and no schema.
Nothing tells the sub-window that the *song* changed rather than just the
title - which is precisely the hole `HW-13-0024` falls through. One
`@StorageLink`ed track object, or a real playback service exposing a current
track, removes the whole class of bug.

Note also the module-scope `let uiContext = AppStorage.get('uiContext') as UIContext;`
at the top of `WindowUtils.ets`. It is evaluated once, at import time, and
works only because `EntryAbility` writes the key inside `loadContent`'s
callback while this file is first imported by the later-loading sub-window
page. That is a timing coincidence; read the context inside the functions.

## Implementation steps

1. **Publish what the sub-window needs from the ability**: the `UIContext`, the
   `WindowStage`, the main window id and the two avoid-area heights into
   `AppStorage` in `onWindowStageCreate`.
2. **Create the sub-window on demand** with `windowStage.createSubWindow(name, cb)`,
   then inside the callback `setUIContent` the sub-page, `moveWindowTo` the
   initial position, `resize` in **px** (`vp2px` the vp constants) and
   `showWindow`.
3. **Make it transparent** with `setWindowBackgroundColor('#00000000')` from
   the `setUIContent` completion callback, and **guard re-entry**:
   `createSubWindow` with an existing name fails, so check
   `window.findWindow(name)` first or destroy on close (`HW-13-0024`).
4. **Give the sub-window's page a `PanGesture`** and move the window from
   `onActionUpdate`. `event.offsetX/Y` are **cumulative since gesture start and
   in vp**, while `moveWindowTo` takes **px**: capture the start position in
   `onActionStart` and set `position = start + vp2px(offset)` (`HW-13-0022`).
5. **Clamp against the avoid areas** so the window cannot hide under the status
   bar or the navigation indicator.
6. **Snap on release** with a half-screen test, and resize with `resizeAsync`
   when the style changes so the following `moveWindowTo` sees the new width.
7. **Shift focus back to the main window** after every interaction, on a short
   delay, or the app below stops receiving input.
8. **Guard player creation.** `media.createAVPlayer()` is async; two quick taps
   both see `avPlayer === undefined` and create two players (`HW-13-0023`).
9. **Re-source the player when the selection changes**, and destroy the
   sub-window when playback ends (`HW-13-0024`).
10. **Initialise the debounce timer to a real sentinel** - a bare
    `let t: number;` is `undefined`, and `undefined !== null` is always true
    (`HW-13-0025`).

## Verified snippets

All snippets are from `SubWindow.zip`. Corrected forms are marked.

**Creating the floating window — `entry/src/main/ets/components/MyLike.ets`** (as shipped)

```typescript
showSubWindow() {
  this.windowStage.createSubWindow(Constants.MUSIC_SUBWINDOW, (err, windowClass) => {
    if (err.code > 0) {
      hilog.error(0x0000, 'testTag', '%{public}s', `failed to create subWindow Cause: ${err.message}`);
      return;
    }
    try {
      // 设置子窗口加载页
      windowClass.setUIContent('subwindow/MusicSubWindow', () => {
        windowClass.setWindowBackgroundColor(Constants.WINDOW_BACKGROUND_COLOR);   // '#00000000'
      });
      // 设置子窗口左上角坐标
      windowClass.moveWindowTo(Constants.INIT_SUBWINDOW_POSITION_X, Constants.INIT_SUBWINDOW_POSITION_Y);
      // 设置子窗口大小
      windowClass.resize(this.getUIContext().vp2px(Constants.INIT_SUBWINDOW_SIZE),
        this.getUIContext().vp2px(Constants.INIT_SUBWINDOW_SIZE));
      // 展示子窗口
      windowClass.showWindow();
    } catch (err) {
      hilog.error(0x0000, 'testTag', '%{public}s', `failed to create subWindow Cause:${err}`);
    }
  });
}
```

**Three details make this a floating window rather than a second page.**
`setUIContent` takes a route into the same `pages` space but gives it a
separate UI instance - `MusicSubWindow` is `@Entry`-decorated for that reason.
The background colour is set from `setUIContent`'s completion callback, not
before it, because the content replaces the window's surface; setting it early
is a common way to end up with an opaque black square. And both `resize` and
`moveWindowTo` are **px** APIs, which is why every size constant is passed
through `vp2px` - the one conversion the drag code later forgets.

The window name is a fixed constant. That is what makes `window.findWindow(Constants.MUSIC_SUBWINDOW)`
work as a global handle from the sub-page - and also what makes the second call
to `showSubWindow()` fail, since a sub-window with that name already exists
(`HW-13-0024`).

**The drag — `entry/src/main/ets/utils/WindowUtils.ets`** (corrected, see `HW-13-0022`)

```typescript
export function dragToMove(event: GestureEvent, win: window.Window,
  start: Position, windowPosition: Position): void {
  const bottomRectHeight = AppStorage.get('bottomRectHeight') as number;
  const topRectHeight = AppStorage.get('topRectHeight') as number;
  const bottomY = display.getDefaultDisplaySync().height - bottomRectHeight -
    uiContext.vp2px(Constants.INIT_SUBWINDOW_SIZE);

  // FIX: shipped code does windowPosition.x += event.offsetX on every update.
  // offsetX/offsetY are cumulative since the gesture start, and are vp against a px position.
  const targetX = start.x + uiContext.vp2px(event.offsetX);
  const targetY = start.y + uiContext.vp2px(event.offsetY);

  windowPosition.x = targetX;
  windowPosition.y = Math.min(Math.max(targetY, topRectHeight), bottomY);
  win.moveWindowTo(windowPosition.x, windowPosition.y);
}
```

and at the call site, in `MusicSubWindow.floatingWindow()`:

```typescript
PanGesture(this.panOption)
  .onActionStart(() => {
    this.dragStart = { x: this.windowPosition.x, y: this.windowPosition.y };   // FIX: absent
  })
  .onActionUpdate((event: GestureEvent) => {
    dragToMove(event, this.subWindow, this.dragStart, this.windowPosition);
  })
  .onActionEnd(() => {
    edgeDetermination(this.subWindow, this.windowPosition);
    ChangeFocus(this.windowStage, Constants.MUSIC_SUBWINDOW);
  })
```

**`GestureEvent.offsetX` is cumulative, not incremental.** A drag that emits
updates at 5, 10, 15, 20 vp moves the shipped window by 5+10+15+20 = 50 vp: the
window accelerates away from the finger, quadratically with the number of
frames. The `+=` form would only be correct against a per-frame delta, which
this event does not provide.

The unit error compounds it. `windowPosition` feeds `moveWindowTo`, which is
documented in **px**, and is clamped against `display.getDefaultDisplaySync().height`,
also px - but `offsetX/offsetY` are vp, and this is the one calculation in the
file that skips `uiContext.vp2px`, so on a density-3 screen the movement is a
further 3x off. Treating the offset as an absolute displacement from a captured
origin also fixes the clamp: the shipped three-branch `if/else if/else if` on
`windowPosition.y` lets the value escape the boundary and then permits movement
in one direction only, where a `min`/`max` clamp simply stops at the edge.

**The player — `entry/src/main/ets/utils/MediaService.ets`** (corrected, see `HW-13-0023`, `HW-13-0024`)

```typescript
export class MediaService {
  public avPlayer?: media.AVPlayer = undefined;
  private status: string = '';
  private creating: boolean = false;              // FIX: absent - no in-flight guard
  private currentResource: string = '';           // FIX: absent - the source is never re-read
  public static instance: MediaService = new MediaService();

  async avPlayerFdSrc(ctx: Context) {
    let avPlayer: media.AVPlayer = await media.createAVPlayer();
    this.setAVPlayerCallback(avPlayer);
    let context = ctx as common.UIAbilityContext;
    this.currentResource = AppStorage.get('resource') as string;
    let fileDescriptor = await context.resourceManager.getRawFd(this.currentResource);
    avPlayer.fdSrc = {
      fd: fileDescriptor.fd, offset: fileDescriptor.offset, length: fileDescriptor.length
    };                                            // assigning fdSrc triggers 'initialized'
    this.avPlayer = avPlayer;
  }

  async switchPlayOrPause(ctx: Context) {
    if (this.creating) {
      return;                                     // FIX: two quick taps each created a player
    }
    if (this.avPlayer === undefined) {
      this.creating = true;
      try {
        await this.avPlayerFdSrc(ctx);
      } finally {
        this.creating = false;
      }
    } else if (AppStorage.get('resource') as string !== this.currentResource) {
      await this.avPlayer.reset();                // FIX: a new selection never reached the player
      await this.avPlayerFdSrc(ctx);
    } else if (this.status === 'playing') {
      this.avPlayer.pause();
    } else {
      this.avPlayer.play();
    }
  }
}
```

**The whole player is driven by state-machine callbacks, not imperative calls.**
Assigning `fdSrc` moves the player to `initialized`; the `stateChange` handler
sets `audioRendererInfo` and calls `prepare()`; the `prepared` case calls
`play()`. That is the prescribed flow, and it is why `switchPlayOrPause` on a
cold player only creates and sources one - playback starts itself two callbacks
later.

That async gap is the defect. `media.createAVPlayer()` is awaited inside
`avPlayerFdSrc`, but the shipped `switchPlayOrPause` is not `async` and does not
await it, so a routine double tap runs the `undefined` branch twice: two
players, both playing the same file, the first leaked with no reference left to
release it. An in-flight flag is the minimum fix; a cached promise is tidier.

The second correction closes `HW-13-0024`: the shipped code reads
`AppStorage.get('resource')` exactly once, when the player is created, so every
later selection updates the pill's title while track one keeps playing.
Comparing the stored resource against the current one and calling `reset()`
before re-sourcing is what makes the list work.

**The debounce and the expand/collapse cycle — `entry/src/main/ets/subwindow/MusicSubWindow.ets`** (corrected, see `HW-13-0025`)

```typescript
let debounceTimer: number | null = null;      // FIX: was `let debounceTimer: number;`

async onClickFunction() {
  if (this.status === Constants.STATUS_PAUSE && this.isExpand) {
    this.status = Constants.STATUS_PLAY;
    MediaService.instance.switchPlayOrPause(this.ctx!);
  } else if (this.status === Constants.STATUS_PLAY && this.isExpand) {
    this.status = Constants.STATUS_PAUSE;
    MediaService.instance.switchPlayOrPause(this.ctx!);
  }
  await switchFloatingWindowStyle(true, this.subWindow);   // expand first, always
  this.isExpand = true;
  edgeDetermination(this.subWindow, this.windowPosition);
  this.debounce(() => {
    this.isExpand = false;
    switchFloatingWindowStyleMove(this.isExpand, this.subWindow, this.windowPosition);
  }, Constants.EXPAND_DELAY);                              // 2000 ms
  ChangeFocus(this.windowStage, Constants.MUSIC_SUBWINDOW);
}

debounce(func: () => void, wait: number) {
  if (debounceTimer !== null) {
    clearTimeout(debounceTimer);
  }
  debounceTimer = setTimeout(() => {
    func();
    debounceTimer = null;                     // FIX: shipped code clears but never nulls the id
  }, wait);
}
```

**The two-stage tap is the interaction worth copying.** The first tap expands
the circle into a pill; only a tap *while expanded* toggles playback. That is
what keeps an accidental brush against the floating window from stopping the
music, and it is why the play/pause branches are guarded on `this.isExpand`.
The 2 s debounce then collapses it again, so the window spends most of its life
as a small circle.

The guard itself never worked. `let debounceTimer: number;` leaves the variable
`undefined`, and `undefined !== null` is `true`, so both `debounce()` and the
gesture's `onActionStart` always take the "there is a pending timer" branch:
`clearTimeout(undefined)` is a silent no-op, and the pending-collapse detection
the comments describe never actually branches. In `onActionStart` that means a
drag always runs the collapse-first path whether or not the pill is open. Typing
the variable `number | null`, initialising it to `null` and resetting it when
the timer fires makes the guard mean what it says.

Note `@StorageLink('status')` rather than `@StorageProp`: the sub-window needs
to *write* the status as well as read it, because `EntryAbility.onBackground`
also writes it when the app leaves the foreground.

**Snapping to the edge — `entry/src/main/ets/utils/WindowUtils.ets`** (as shipped)

```typescript
export function edgeDetermination(window: window.Window, windowPosition: Position): void {
  const HALF_WIDTH = display.getDefaultDisplaySync().width * Constants.HALF;
  if (windowPosition.x < HALF_WIDTH) {
    windowPosition.x = 0;
  } else {
    windowPosition.x = display.getDefaultDisplaySync().width - window.getWindowProperties().windowRect.width;
  }
  window.moveWindowTo(windowPosition.x, windowPosition.y);
}

export function ChangeFocus(windowStage: window.WindowStage, windowName: string) {
  setTimeout(() => {
    let subWindowID: number = window.findWindow(windowName).getWindowProperties().id;
    let mainWindowID: number = windowStage.getMainWindowSync().getWindowProperties().id;
    window.shiftAppWindowFocus(subWindowID, mainWindowID);
  }, Constants.CHANGE_FOCUS_DELAY);
}
```

The snap reads the window's **current** width from `getWindowProperties()`
rather than a constant, which is what lets the same function snap both the
50 vp circle and the 181 vp pill correctly - and why `switchFloatingWindowStyleMove`
awaits `resizeAsync` before snapping: `resize` returns before the resize is
applied, so a snap computed against the old width would leave a gap.

`ChangeFocus` is the piece most implementations miss: a sub-window that holds
focus makes the app underneath stop responding to input, so focus is shifted
back to the main window by id after every interaction. The 500 ms delay is a
magic number standing in for the window operation it must follow - a promise
chain on `resizeAsync`/`moveWindowToAsync` is the honest version.

## Permissions & config

**None.** The floating window is an **in-app** sub-window, not a system
overlay, so it is only visible while this app is in the foreground - a window
floating over *other* applications is a different mechanism with its own
permission. `deviceTypes` is `["phone", "tablet", "2in1"]`, and the audio is
three bundled `.wav` rawfiles reused across the nine list entries.

`EntryAbility` bridges the two windows in both directions:

```typescript
AppStorage.setOrCreate('uiContext', windowStage.getMainWindowSync().getUIContext());
AppStorage.setOrCreate('windowStage', windowStage);
AppStorage.setOrCreate('mainWindowId', windowStage.getMainWindowSync().getWindowProperties().id);

onForeground(): void {
  this.context.eventHub.emit('onForeground');     // the sub-window re-snaps to the edge
}
onBackground(): void {
  let status = AppStorage.get('status') as number;
  if (status === Constants.STATUS_PLAY) {
    AppStorage.set('status', Constants.STATUS_PAUSE);
  }
}
```

`MusicSubWindow` subscribes in `aboutToAppear` and unsubscribes in
`aboutToDisappear` - correct, and worth copying; note that
`eventHub.off('onForeground')` without the handler removes *all* subscribers,
fine here only because there is one. `onBackground` flips the *icon* state to
paused but never calls `pause()`, so the audio keeps playing while the button
shows the opposite.

## Constraints

- API Version 20 Release or later; HarmonyOS 6.0.0 Release SDK or later;
  DevEco Studio 6.0.0 Release or later. `compatibleSdkVersion` and
  `targetSdkVersion` are both `6.0.0(20)`, and the document agrees.
- `moveWindowTo` is intended for floating-mode windows and moves relative to
  the parent window in non-freeform mode; it does not work on a main window in
  non-freeform mode. Coordinates are px and non-integers are rounded down.
- `moveWindowTo` and `resize` return before the effect is applied; use
  `moveWindowToAsync`/`resizeAsync` when the next step depends on the result.
- `Constants.STATUS_PAUSE_EXPAND` and `STATUS_PLAY_EXPAND` are declared and
  never used; the real state is the pair (`status`, `isExpand`).
- Nine list entries share three audio files and one title/artist pair, which is
  why the "second song does not play" defect is invisible in a casual demo.

## Pitfalls

- **`HW-13-0022`** (B/medium, confirmed): `dragToMove` does
  `windowPosition.x += event.offsetX` on every `onActionUpdate`, but the offsets
  are cumulative since the gesture start, so the movement is the sum of all
  intermediate offsets and the window flies past the finger; the offsets are
  also vp added to a px position with no `vp2px`, which every other calculation
  in the file performs. Fix: capture the start position in `onActionStart` and
  set `position = start + vp2px(offset)`.
- **`HW-13-0023`** (B/medium, confirmed): `switchPlayOrPause` has no in-flight
  guard while `avPlayerFdSrc` is async, so two quick taps both see
  `avPlayer === undefined` and each call `media.createAVPlayer()`; the first
  instance leaks and both play at once. Fix: an in-flight flag, or await/cache
  the creation promise.
- **`HW-13-0024`** (B/medium, probable): selecting a second track never changes
  the audio. `createSubWindow` with the same fixed name errors when the window
  already exists, and `MediaService` reads `AppStorage.get('resource')` only
  once, when the player is created; no `destroyWindow` exists anywhere. Fix:
  reset and re-source the player when the selection changes, and destroy or
  deliberately reuse the sub-window.
- **`HW-13-0025`** (B/low, confirmed): the `debounceTimer !== null` guard is
  always true - the variable is declared without an initialiser and is never
  set to `null` - so the pending-collapse detection never branches and
  `clearTimeout(undefined)` is a silent no-op. Fix: type it `number | null`,
  initialise to `null`, reset it when the timer fires.
- **Related systematic `HW-13-0005`**: five media samples create AVPlayer or
  SoundPool instances and never call `release()`. This sample does call
  `release()`, but only from the `idle` state that follows an error-triggered
  `reset()` - there is still no teardown path releasing the player (or
  destroying the sub-window) when the user is finished.

## References

- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-windowstage.md` - `createSubWindow`, `getMainWindowSync`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-windowstage
- `documentation/harmonyos-references/02_application-framework/arkts-apis-window-window.md` - `setUIContent`, `showWindow`, `moveWindowTo` (px), `resize`/`resizeAsync`, `getWindowProperties`, `shiftAppWindowFocus`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-window-window
- `documentation/harmonyos-guides/03_application-framework/window-overview.md` - main window vs sub-window, when a sub-window is the right tool
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/window-overview
- `documentation/harmonyos-guides/03_application-framework/application-window-stage.md` - the sub-window lifecycle, including destruction
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/application-window-stage
- `documentation/harmonyos-references/04_media/arkts-apis-media-avplayer.md` - `createAVPlayer`, `fdSrc`, the state machine, `release()`
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/arkts-apis-media-avplayer
- `documentation/harmonyos-guides/03_application-framework/arkts-gesture-events-single-gesture.md` - `PanGesture` and the cumulative `offsetX`/`offsetY` semantics
  https://developer.huawei.com/consumer/en/doc/harmonyos-guides/arkts-gesture-events-single-gesture
- `documentation/harmonyos-references/02_application-framework/js-apis-display.md` - `getDefaultDisplaySync`, display width/height in px
  https://developer.huawei.com/consumer/en/doc/harmonyos-references/js-apis-display
- `MEDIA-05` - the other window-property player control (brightness) in this industry
